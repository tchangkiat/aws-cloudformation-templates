# Create CloudFormation Stacks

1. Create a stack with VPC.json in Network folder.
2. Create a stack with EKS.json, with parameters referencing subnets created by VPC.json. This template takes about 20 minutes or so to complete.

# Configure Bastion Host

## 1. Set up environment variables

1. Execute the following commands and re-connect to the bastion host:

```bash
# Replace <> with the respective values
echo 'export AWS_ACCOUNT_ID=409989946510' >> ~/.bashrc
echo 'export AWS_REGION=ap-southeast-1' >> ~/.bashrc
echo 'export AWS_EKS_CLUSTER=MyProject2' >> ~/.bashrc
```

## 2. Configure AWS CLI

1. Execute the following commands and enter the access key ID and secret access key, along with other information like default region and output format:

```bash
aws configure
```

2. Execute the following command and replace the respective values in arrow brackets:

```bash
aws eks --region $AWS_REGION update-kubeconfig --name $AWS_EKS_CLUSTER
```

## 3. Test the connectivity to the EKS cluster

1. Execute this command:

```bash
kubectl get svc
```

2. Expected Output (similar to the one provided below):

```
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
svc/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1m
```

## 4. Set up AWS App Mesh (optional)

1. Execute the following commands:

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

curl https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/proxy-auth-cf.json -o proxy-auth-cf.json

aws cloudformation create-stack --stack-name AppMeshProxyAuthPolicy --template-body file://proxy-auth-cf.json --parameters ParameterKey=MeshName,ParameterValue=default-mesh --capabilities CAPABILITY_NAMED_IAM

eksctl create iamserviceaccount --cluster $AWS_EKS_CLUSTER --namespace default --name sample-express-api-service --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AppMeshProxyAuth-default-mesh --override-existing-serviceaccounts --approve

# Scale up an additional node to fit in 2 sets of sample-express-api and 1 set of AWS App Mesh Pods
eksctl scale nodegroup --cluster=$AWS_EKS_CLUSTER --nodes=3 --name `eksctl get nodegroup --cluster $AWS_EKS_CLUSTER | grep 'EKSNodeGroup' | awk '{print $2}'`
```

2. Deploy sample-express-api.

```bash
curl https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/deployment.yaml -o deployment.yaml
```

3. Remember to change [URL] in deployment.yaml to the container image URL and execute ```kubectl apply -f deployment.yaml```

4. The Envoy containers should be injected in your application Pods automatically.

## 5. Set up AWS X-Ray Integration (optional)

1. Execute the following commands:

```bash
eksctl create iamserviceaccount --name sample-express-api-service-account --namespace default --cluster $AWS_EKS_CLUSTER --attach-policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --approve

helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system --set region=$AWS_REGION --set serviceAccount.create=false --set serviceAccount.name=appmesh-controller --set tracing.enabled=true --set tracing.provider=x-ray

kubectl rollout restart deployment sample-express-api
```

2. Modify your source code to include and use AWS X-Ray SDK.

## 6. Tear Down

1. Execute the following commands in the Bastion Host if AWS App Mesh was set up:

```bash
kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/deployment.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualservice.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualrouter.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualnode.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh.yaml"
```

2. Delete all related CloudFormation templates.
