# Opinionated basic setup EKS + Karpenter + Spot instance + Application Load Balancer

## NOT SUITABLE FOR PRODUCTION, LEARNING PURPOSE ONLY

*This project is licensed under the terms of the MIT license.*

Example of running basic Kubernetes cluster setup. Features:

1. Pod autoscaling with HPA + node autoscaling with Karpenter
2. Cost savings with spot instance
3. Load balancing with Application Load Balancer (ALB)
4. Observability for logging and monitoring with CloudWatch Container Insights

Tested with:
1. Kubernetes 1.28
2. Karpenter 0.31

## Cluster setup

1. Export parameters
    ```
    export KARPENTER_VERSION=v0.31.1
    export CLUSTER_NAME=eks-jkt-128
    export AWS_DEFAULT_REGION="ap-southeast-3"
    export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
    ```

2. Create instance profile with necessary permissions to run containers and networking
    ```
    curl -fsSL https://raw.githubusercontent.com/aws/karpenter/"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > karpenter-${KARPENTER_VERSION}-cf-template.yml \
    && aws cloudformation deploy \
    --stack-name "Karpenter-${CLUSTER_NAME}" \
    --template-file "karpenter-${KARPENTER_VERSION}-cf-template.yml" \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameter-overrides "ClusterName=${CLUSTER_NAME}"
    ```

3. Create cluster with necessary instance profile and service account mappings
    ```
    cat <<EOF > cluster.yml
    ---
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig
    metadata:
      name: ${CLUSTER_NAME}
      region: ${AWS_DEFAULT_REGION}
      version: "1.28"
      tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}

    iam:
      withOIDC: true
      serviceAccounts:
      - metadata:
          name: karpenter
          namespace: karpenter
        roleName: ${CLUSTER_NAME}-karpenter
        attachPolicyARNs:
        - arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}
        roleOnly: true

    iamIdentityMappings:
    - arn: "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
      username: system:node:{{EC2PrivateDNSName}}
      groups:
      - system:bootstrappers
      - system:nodes

    managedNodeGroups:
    - instanceType: m5.xlarge
      amiFamily: AmazonLinux2
      name: ${CLUSTER_NAME}-ng
      desiredCapacity: 2
      minSize: 2
      maxSize: 10
      iam:
        withAddonPolicies:
          cloudWatch: true
      labels:
        purpose: control-apps
    EOF
        
    eksctl create cluster -f cluster.yml 

    export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
    export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

    echo $CLUSTER_ENDPOINT $KARPENTER_IAM_ROLE_ARN
    ```

    Create service role for spot instance (only execute if you never use spot instance before)
    ```
    aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
    # If the role has already been successfully created, you will see:
    # An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.

    ```

4. Install karpenter
    ```
    # Logout of helm registry to perform an unauthenticated pull against the public ECR
    helm registry logout public.ecr.aws

    helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
    --set settings.aws.clusterName=${CLUSTER_NAME} \
    --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
    --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
    --set controller.resources.requests.cpu=1 \
    --set controller.resources.requests.memory=1Gi \
    --set controller.resources.limits.cpu=1 \
    --set controller.resources.limits.memory=1Gi \
    --wait

    kubectl get deployment -n karpenter
    ```

5. Configure Karpenter default provisioner
    ```
    cat <<EOF > default-provisioner.yml
    apiVersion: karpenter.sh/v1alpha5
    kind: Provisioner
    metadata:
      name: default
    spec:
      labels:
        purpose: apps
      requirements:
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["c", "m", "r"]
        - key: "karpenter.k8s.aws/instance-cpu"
          operator: In
          values: ["2", "4"]
        - key: "karpenter.sh/capacity-type" # If not included, the webhook for the AWS cloud provider will default to on-demand
          operator: In
          values: ["spot", "on-demand"]
      limits:
        resources:
          cpu: "1000"
      providerRef:
        name: default
      consolidation:
        enabled: true
    ---
    apiVersion: karpenter.k8s.aws/v1alpha1
    kind: AWSNodeTemplate
    metadata:
      name: default
    spec:
      subnetSelector:
        karpenter.sh/discovery: ${CLUSTER_NAME}
      securityGroupSelector:
        kubernetes.io/cluster/$CLUSTER_NAME: owned
    EOF

    kubectl apply -f default-provisioner.yml
    ```

    ❗❗ Note
    `securityGroupSelector` will attach security group to provision nodes. AWS Load Balancer controller currently requries the node to have exactly one security group with tag `kubernetes.io/cluster/$CLUSTER_NAME` attached. See [this issue](https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/2367) and [Karpenter docs](https://karpenter.sh/docs/concepts/node-templates/#specsecuritygroupselector) for more details.

## Workload deployment

1. Deploy test service
    ```
    kubectl apply -f service.yml 

    kubectl get pods -n apps -o wide

    # Verify Karpenter provision new nodes because we set service labels 'purpose=apps', 
    # and the node is a spot instance (if capacity is available in the AZ)
    kubectl get nodes --show-labels=true | grep spot 
    ```

2. Configure HPA
    ```
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    kubectl get deployment metrics-server -n kube-system

    kubectl autoscale deployment svc-a --cpu-percent=50 --min=1 --max=10 -n apps
    kubectl autoscale deployment svc-b --cpu-percent=50 --min=1 --max=10 -n apps
    kubectl get hpa -n apps
    ```

3. Setup ALB
    ```
    # Install AWS Load Balancer Controller
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

    aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

    eksctl create iamserviceaccount \
    --cluster=${CLUSTER_NAME} \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve

    helm repo add eks https://aws.github.io/eks-charts
    helm repo update eks
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=${CLUSTER_NAME} \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller 

    kubectl get deployment -n kube-system aws-load-balancer-controller

    # Create ALB
    kubectl apply -f ingress.yml
    kubectl get ingress/svc-ingress -n apps
    ```

4. verify 
    ```
    curl http://k8s-apps-svcingre-f2fbee10c9-940739950.ap-southeast-3.elb.amazonaws.com/svc-a
    curl http://k8s-apps-svcingre-f2fbee10c9-940739950.ap-southeast-3.elb.amazonaws.com/svc-b
    ```

## Observability
    ```
    FluentBitHttpPort='2020'
    FluentBitReadFromHead='Off'
    [[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
    [[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
    curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${CLUSTER_NAME}'/;s/{{region_name}}/'${AWS_DEFAULT_REGION}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f - 

    kubectl get ds -n amazon-cloudwatch
    ```

## Update history
Oct 2023:
- Upgrade versions
- Change Nginx NLB to ALB
- Setting container insight