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
```
take a look at the load balancer itself:
``` 
 aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`]' 
 ```
 -  The NLB is accessible over the public internet
 - It uses the public subnets in our VPC

Inspect the targets in the target group that was created by the controller:
```
ALB_ARN=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-ui-uinlb`) == `true`].LoadBalancerArn' | jq -r '.[0]')
TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN | jq -r '.TargetGroups[0].TargetGroupArn')

aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
```
Get the URL from the Service resource:
```
kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}"

```
Wait until the load balancer has finished provisioning:
```
wait-for-lb $(kubectl get service -n ui ui-nlb -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")

```

 Now that our application is exposed to the outside world!

___
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
