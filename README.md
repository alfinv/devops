AWS regions  not guaranteed to work without any modification:

us-west-2
eu-west-1
### Cloudshell

```
wget -q https://raw.githubusercontent.com/aws-samples/
eks-workshop-v2/stable/lab/cfn/eks-workshop-ide-cfn.yaml -O eks-workshop-ide-cfn.yaml

aws cloudformation deploy --stack-name eks-workshop-ide \
    --template-file ./eks-workshop-ide-cfn.yaml \
    --parameter-overrides RepositoryRef=stable \
    --capabilities CAPABILITY_NAMED_IAM
```
The CloudFormation stack will take roughly 5 minutes to deploy, and once completed you can retrieve the URL for the Cloud9 IDE like so:
```

aws cloudformation describe-stacks --stack-name eks-workshop-ide \
    --query 'Stacks[0].Outputs[?OutputKey==`Cloud9Url`].OutputValue' --output text

```

The AWS CLI is already installed and will assume the credentials attached to the Cloud9 IDE:

```
aws sts get-caller-identity
```

### Cloud9
#### The eksctl utility has been pre-installed in your Amazon Cloud9 Environment, so we can immediately create the cluster. 

- Create a VPC across three availability zones
- Create an EKS cluster
- Create an IAM OIDC provider
- Add a managed node group named default
- Configure the VPC CNI to use prefix delegation
```
export EKS_CLUSTER_NAME=eks-workshop

curl -fsSL https://raw.githubusercontent.com/aws-samples/eks-workshop-v2/stable/cluster/eksctl/cluster.yaml | \
envsubst | eksctl create cluster -f -
```
This generally takes 20 minutes.
```
use-cluster $EKS_CLUSTER_NAME
```
#### Setup LoadBalancer:
1. Install the AWS Load Balancer Controller in the Amazon EKS cluster
     - This Service will create a Network Load Balancer that listens on port 80 and forwards connections to the ui Pods on port 8080. An NLB is a layer 4 load balancer that on our case operates at the TCP layer.


```
prepare-environment exposing/load-balancer

kubectl apply -k ~/environment/eks-workshop/modules/exposing/load-balancer/nlb

kubectl get service -n ui

alias kn='k config set-context --current --namespace '
kn ui
```
take a look at the load balancer itself:
``` 
 aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`]' 
 ```
 -  The NLB is accessible over the public internet
 - It uses the public subnets in our VPC

Inspect the targets in the target group that was created by the controller:
```sh
ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`].LoadBalancerArn' | jq -r '.[0]')
TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn')

aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
```
output:
```json
{
            "Target": {
                "Id": "i-05295721e406a90a6", <== ec2 instance name where kube-proxy 
                "Port": 30095
            },
            "HealthCheckPort": "30095",
            "TargetHealth": {
                "State": "healthy"
            }
        },
        {
            "Target": {
                "Id": "i-0402ca116eb4048af", <== ...
                "Port": 30095
```
Get the URL from the Service resource:
```sh
kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}"

```
Wait until the load balancer has finished provisioning:
```sh
wait-for-lb $(kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")

```

 Now that our application is exposed to the outside world!
 
 ### IP mode
 The NLB we have created is operating in _"instance mode"_. Instance target mode supports pods running on AWS EC2 instances. In this mode, AWS NLB sends traffic to the instances and the kube-proxy on the individual worker nodes forward it to the pods 

 The AWS Load Balancer Controller also supports creating NLBs operating in *"IP mode"* - sends traffic directly to the Kubernetes pods behind the service, eliminating the need for an extra network hop through the worker nodes in the Kubernetes cluster. _IP target mode_ supports pods running on both AWS EC2 instances and AWS Fargate. 
 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip <======
  namespace: ui
```
```sh
kubectl apply -k ~/environment/eks-workshop/modules/exposing/load-balancer/ip-mode

kubectl describe service/ui-nlb -n ui
    ...
     Annotations:   service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    ...
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
    ...
    "Target": {
                "Id": "10.42.118.118", <== ip adrres ec2 instnce
                "Port": 8080,
                "AvailabilityZone": "eu-west-1a"
            },
    ...            
```
### Ingress
```sh
prepare-environment exposing/ingress

kubectl apply -k ~/environment/eks-workshop/modules/exposing/ingress/creating-ingress

kubectl get ingress ui -n ui

aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-ui`) == `true`]'


```
```sh
ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-ui`) == `true`].LoadBalancerArn' | jq -r '.[0]')
TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn')
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
```
Get the URL from the Ingress resource:

```sh
kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}"
```
### Multiple Ingress pattern
It's common to leverage multiple Ingress objects in the same EKS cluster, for example to expose multiple different workloads. By default each Ingress will result in the creation of a separate ALB, but we can leverage the IngressGroup feature which enables you to group multiple Ingress resources together. The controller will automatically merge Ingress rules for all Ingresses within IngressGroup and support them with a single ALB. In addition, most annotations defined on an Ingress only apply to the paths defined by that Ingress.

In this example, we'll expose the catalog API out through the same ALB as the ui component, leveraging path-based routing to dispatch requests to the appropriate Kubernetes service. Let's check we can't already access the catalog API:
```sh 

ADDRESS=$(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
echo $ADDRESS
curl $ADDRESS/catalogue
```

The first thing we'll do is re-create the Ingress for ui component adding the annotation alb.ingress.kubernetes.io/group.name:
```yaml
~/environment/eks-workshop/modules/exposing/ingress/multiple-ingress/ingress-ui.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ui
  namespace: ui
  labels:
    app.kubernetes.io/created-by: eks-workshop
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
    alb.ingress.kubernetes.io/group.name: retail-app-group <===
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ui
                port:
                  number: 80
```

Now, let's create a separate Ingress for the catalog component that also leverages the same group.name:
```yaml
~/environment/eks-workshop/modules/exposing/ingress/multiple-ingress/ingress-catalog.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalog
  namespace: catalog
  labels:
    app.kubernetes.io/created-by: eks-workshop
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/group.name: retail-app-group <===
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /catalogue
            pathType: Prefix
            backend:
              service:
                name: catalog
                port:
                  number: 80
```
```sh

kubectl apply -k ~/environment/eks-workshop/modules/exposing/ingress/multiple-ingress

kubectl get ingress -l app.kubernetes.io/created-by=eks-workshop -A


kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}"


```


[![Resou8rce map application ](https://github.com/alfinv/devops/blob/dbab3a9dc12abf14febe17882a2635d863ad153c/img/Resource%20map.png)] 


---
## Cleaning Up

Cloud 9:
```
    delete-environment

    eksctl delete cluster $EKS_CLUSTER_NAME --wait
```
CloudShell:
```
    aws cloudformation delete-stack --stack-name eks-workshop-ide
```
