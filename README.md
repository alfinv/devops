AWS regions should work without any modification:

us-west-2
eu-west-1

### Cloudshell [(link)(https://eu-west-1.console.aws.amazon.com/cloudshell/home?region=eu-west-1#)]

```
wget -q https://raw.githubusercontent.com/aws-samples/eks-workshop-v2/stable/lab/cfn/eks-workshop-ide-cfn.yaml -O eks-workshop-ide-cfn.yaml

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

###  => Cloud9
#### The eksctl utility has been pre-installed in your Amazon Cloud9 Environment, so we can immediately create the cluster. 

- Create a VPC across three availability zones
- Create an EKS cluster
- Create an IAM OIDC provider
- Add a managed node group named default
- Configure the VPC CNI to use prefix delegation
```sh
export EKS_CLUSTER_NAME=eks-workshop

curl -fsSL https://raw.githubusercontent.com/aws-samples/eks-workshop-v2/stable/cluster/eksctl/cluster.yaml | \
envsubst | eksctl create cluster -f -
```
or edit type of instances befarre apply it
```sh
curl -fsSL https://raw.githubusercontent.com/aws-samples/eks-workshop-v2/stable/cluster/eksctl/cluster.yaml --output cluster.yaml && vim cluster.yaml
cat cluster.yaml | envsubst | eksctl create cluster -f -
```
This generally takes 20 minutes.
```
use-cluster $EKS_CLUSTER_NAME
```
### => or using  Teraform
```
git clone https://github.com/alfinv/devops.git
cd devops/terraform

export EKS_CLUSTER_NAME=eks-workshop
terraform init
terraform apply -var="cluster_name=$EKS_CLUSTER_NAME" -auto-approve

use-cluster $EKS_CLUSTER_NAME

```
## Start app
```

prepare-environment introduction/getting-started

kubectl apply -k ~/ironment/eks-workshop/base-application

kubectl -n catalog exec -it \
  deployment/catalog -- curl catalog.catalog.svc/catalogue | jq .
```



#### Setup LoadBalancer:


expose an application running in the EKS cluster to the Internet using a **layer 4** !!! Network Load Balancer. (NLB)
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
Application Load Balancer (ALB) is a popular AWS service that load balances incoming traffic at the application layer **(layer 7 !!!)** across multiple targets, such as Amazon EC2 instances, in a region.
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
EC2 >>
Load balancers >>
k8s-retailappgroup-085b071a94 :

[![Resou8rce map application ](https://github.com/alfinv/devops/blob/dbab3a9dc12abf14febe17882a2635d863ad153c/img/Resource%20map.png)] 

```sh

ADDRESS=$(kubectl get ingress -n ui ui -o jsonpath="{.status.loadBalancer.ingress[*].hostname}{'\n'}")
curl $ADDRESS/catalogue | jq .
```

## EBS
Elastic Block Store is an easy-to-use, scalable, high-performance block-storage service. It provides persistent volume (non-volatile storage) to users. Persistent storage enables users to store their data until they decide to delete the data.

###  1.  StatefulSets
 - maintain a sticky identity for each of its Pods
 - Stable, unique network identifiers
  - Stable, persistent storage
  - Ordered, graceful deployment and scaling
  - Ordered, automated rolling updates

StatefulSet already deployed as part of the Catalog microservice - utilizes a MySQL database
```
kubectl describe statefulset -n catalog catalog-mysql
```
```yaml
 Mounts:
      /var/lib/mysql from data (rw)
  Volumes:
   data:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)*
 - * When a Pod is removed from a node for any reason, the data in the emptyDir is deleted permanently.
```
### 2. EBS CSI Driver
The Kubernetes Container Storage Interface (CSI) helps you run stateful containerized applications. CSI drivers provide a CSI interface that allows Kubernetes clusters to manage the lifecycle of persistent volumes.

Manage the Amazon EBS CSI driver as an Amazon EKS add-on. The IAM role needed by the addon was created for us so we can go ahead and install the addon:
```sh
#Create the IAM role needed for the EBS CSI driver addon
prepare-environment fundamentals/storage/ebs
#or use terraform ro role creation: cd ~/environment/devops/terraform-csi-irole ; terraform init; terraform plan
# terraform apply -auto-approve

aws eks create-addon --cluster-name $EKS_CLUSTER_NAME --addon-name aws-ebs-csi-driver \
  --service-account-role-arn $EBS_CSI_ADDON_ROLE
aws eks wait addon-active --cluster-name $EKS_CLUSTER_NAME --addon-name aws-ebs-csi-driver


kubectl get daemonset ebs-csi-node -n kube-system
```
It cold be perform manuaauly: EKS>Clusters>eks-workshop on tab Add-ons button: "Get more add-ons"
[![addons](https://github.com/alfinv/devops/blob/74e6dced61dec1a32d5eba0a88868fb0d9c87f0b/img/AddonsEKS.png)]
instead 

We also already have our StorageClass object configured using Amazon EBS GP2 volume type. Run the following command to confirm:
```sh
kubectl get storageclass
```

#### 3.  modifying the MySQL DB StatefulSet of the catalog microservice to utilize a EBS block store volume as the persistent storage for the database files using k8s pv provisioning
Create a **new** StatefulSet for the MySQL  - in existing StatefulSet fields we need to update are immutable and cannot be changed.
```sh

kubectl apply -k ~/environment/eks-workshop/modules/fundamentals/storage/ebs/
kubectl rollout status --timeout=100s statefulset/catalog-mysql-ebs -n catalog

kubectl get statefulset -n catalog catalog-mysql-ebs


kubectl get pv | grep -i catalog
```
use aws cli to check the Amazon EBS volume that got created automatically for us:
```sh 
aws ec2 describe-volumes \
    --filters Name=tag:kubernetes.io/created-for/pvc/name,Values=data-catalog-mysql-ebs-0 \
    --query "Volumes[*].{ID:VolumeId,Tag:Tags}" \
    --no-cli-pager
```
EC2 Volumes: [link (https://eu-west-1.console.aws.amazon.com/ec2/home?region=eu-west-1#Volumes:)]

Check mounting in container -  disk that is currently being mounted on the */var/lib/mysql*. This is the EBS Volume for the stateful MySQL database files that being stored in a persistent way.
```sh
 kubectl exec catalog-mysql-ebs-0  -n catalog -- bash -c "df -h"
```
```js
Filesystem      Size  Used Avail Use% Mounted on
overlay         100G  7.6G   93G   8% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.8G     0  3.8G   0% /sys/fs/cgroup
/dev/nvme0n1p1  100G  7.6G   93G   8% /etc/hosts
shm              64M     0   64M   0% /dev/shm
/dev/nvme1n1     30G  211M   30G   1% /var/lib/mysql
tmpfs           7.0G   12K  7.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           3.8G     0  3.8G   0% /proc/acpi
tmpfs           3.8G     0  3.8G   0% /sys/firmware
```

---
## Cleaning Up

Cloud 9:
```
    delete-environment

    eksctl delete cluster $EKS_CLUSTER_NAME --wait
```
Terraform :
```
    cd ~/devops/terraform
    terraform destroy -var="cluster_name=$EKS_CLUSTER_NAME" -auto-approve
```

CloudShell:
```
    aws cloudformation delete-stack --stack-name eks-workshop-ide
```
 ### Other
 ```
 ssh-keygen -t ed25519 -C "your_email@example.com"
 ssh-add ~/.ssh/id_ed25519
 ```
