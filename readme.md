# Opinionated basic setup EKS + Karpenter + Spot instance + NGINX Ingress

## NOT SUITABLE FOR PRODUCTION, LEARNING PURPOSE ONLY

*This project is licensed under the terms of the MIT license.*

Example of running basic Kubernetes cluster setup. Features:

1. Pod autoscaling with HPA + node autoscaling with Karpenter
2. Cost savings with spot instance
3. Load balancing with Network Load Balancer (NLB)

Tested with:
1. Kubernetes 1.21
2. Karpenter 0.11
3. Nginx ingress 1.20

## Steps

1. create cluster
    ```
    eksctl create cluster -f cluster.yml 
    ```

2. install metrics server
    ```
    kubectl apply -f metrics-server-v0.6.1.yml 
    kubectl get apiservice v1beta1.metrics.k8s.io -o json | jq '.status'

    export KARPENTER_VERSION=v0.11.1
    export CLUSTER_NAME=eks-jkt-121
    export AWS_DEFAULT_REGION="ap-southeast-3"
    export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
    ```

3. generate instance profile with necessary permissions to run containers and networking
    ```
    aws cloudformation deploy \
      --stack-name "Karpenter-${CLUSTER_NAME}" \
      --template-file "karpenterv-v0.11.1-cf-template.yml" \
      --capabilities CAPABILITY_NAMED_IAM \
      --parameter-overrides "ClusterName=${CLUSTER_NAME}"
    ```

4. grant access to instances with the profile to connect to the cluster
    ```
    eksctl create iamidentitymapping \
      --username system:node:{{EC2PrivateDNSName}} \
      --cluster  ${CLUSTER_NAME} \
      --arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME} \
      --group system:bootstrappers \
      --group system:nodes

    kubectl describe configmap -n kube-system aws-auth
    ```

5. prerequsite for IRSA
    ```
    eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} --approve
    ```

6. create SA for karpenter controller
    ```
    eksctl create iamserviceaccount \
      --cluster "${CLUSTER_NAME}" --name karpenter --namespace karpenter \
      --role-name "${CLUSTER_NAME}-karpenter" \
      --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}" \
      --role-only \
      --approve

    export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
    export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
    ```

7. install karpenter
    ```
    helm repo add karpenter https://charts.karpenter.sh
    helm repo update

    helm upgrade --install --namespace karpenter --create-namespace \
      karpenter karpenter/karpenter \
      --version ${KARPENTER_VERSION} \
      --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
      --set clusterName=${CLUSTER_NAME} \
      --set clusterEndpoint=${CLUSTER_ENDPOINT} \
      --set nodeSelector.intent=control-apps \
      --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
      --wait

    kubectl get pods -n karpenter
    kubectl get deployment -n karpenter
    ```

8. configure karpenter default provisioner
    ```
    kubectl apply -f default-provisioner.yml
    ```

    ❗❗ **Caution**

    Karpenter will create control plane security group and shared node security group. To avoid error `Multiple tagged security groups found for instance i-xxxxxxxxxx; ensure only the k8s security group is tagged` when working with service type LoadBalancer or with nginx ingress controller, configure the `securityGroupSelector`, or use Launch Template.

    Read more at [Karpenter provisioner docs](https://karpenter.sh/v0.11.1/aws/provisioning/#securitygroupselector-required-when-not-using-launchtemplate)

9. install nginx ingress controller
    ```
    kubectl apply -f nginx-ingress-v1.2.0.yml
    ```

10. deploy test service
    ```
    kubectl apply -f service.yml 

    kubectl get pods -n apps -o wide
    ```

11. hpa
    ```
    kubectl autoscale deployment svc-a --cpu-percent=50 --min=1 --max=10 -n apps
    kubectl autoscale deployment svc-b --cpu-percent=50 --min=1 --max=10 -n apps
    ```

12. install node termination handler
    ```
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update
    helm install aws-node-termination-handler \
      --namespace kube-system \
      --version 0.18.5 \
      --set nodeSelector."karpenter\\.sh/capacity-type"=spot \
      eks/aws-node-termination-handler

    kubectl get ds -n kube-system
    ```

13. deploy nginx ingress
    ```
    kubectl apply -f ingress.yml
    ```

    Note: 
    1. host parameter can be obtained from NLB DNS
    2. set either `kubernetes.io/ingress.class: "nginx"` annotation or `ingressClassName: nginx`

    Look at following blog post: [Using a Network Load Balancer with the NGINX Ingress Controller on Amazon EKS](https://aws.amazon.com/blogs/opensource/network-load-balancer-nginx-ingress-controller-eks/) for guide and [Nginx ingress docs](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/) for configuration reference 

14. verify 
    ```
    curl http://a85f867e9a10b4bb38e37f96047f0b48-9c2009c017d2525a.elb.ap-southeast-3.amazonaws.com/svc-a
    curl http://a85f867e9a10b4bb38e37f96047f0b48-9c2009c017d2525a.elb.ap-southeast-3.amazonaws.com/svc-b
    ```