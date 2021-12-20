# Setup Guide

## CloudFormation

Use the IAM user created above to:
1. Create a stack with VPC.json in Network folder
2. Create a stack with EKS.json, with parameters referencing subnets created by VPC.json. This template takes about 20 minutes or so to complete

## Bastion Host

### 1. Configure AWS CLI
Execute the following commands and enter the access key ID and secret access key, along with other information like default region and output format:
```bash
aws configure
```

Execute the following command and replace the respective values in arrow brackets:
```bash
aws eks --region $AWS_REGION update-kubeconfig --name $AWS_EKS_CLUSTER
```

### 2. Test
Execute this command:
```bash
kubectl get svc
```
Expected Output (similar to the one provided):
```
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
svc/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1m
```

### 3. Set up AWS App Mesh (optional)
Execute the following commands:
```bash
kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts

kubectl create namespace appmesh-system

eksctl utils associate-iam-oidc-provider --region=$AWS_REGION --cluster=$AWS_EKS_CLUSTER --approve

eksctl create iamserviceaccount --namespace appmesh-system --name appmesh-controller --attach-policy-arn arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess,arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --cluster $AWS_EKS_CLUSTER --approve

helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system --set region=$AWS_REGION --set serviceAccount.create=false --set serviceAccount.name=appmesh-controller

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-namespace.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualnode.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualrouter.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualservice.yaml"

curl https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/proxy-auth-policy.json -o proxy-auth-policy.json

aws iam create-policy --policy-name AppMeshProxyAuth --policy-document file://proxy-auth-policy.json

eksctl create iamserviceaccount --cluster $AWS_EKS_CLUSTER --namespace default --name sample-express-api-service --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AppMeshProxyAuth --override-existing-serviceaccounts --approve
```