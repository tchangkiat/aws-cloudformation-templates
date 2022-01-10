# Create Resources

1. Create a stack with VPC.json in Network folder.
2. Create an EC2 key pair separately, which will be used to SSH into the bastion host.
3. Create a stack with EKS.json, with parameters referencing subnets created by VPC.json and the key pair which you have created in step 2. This template takes about 25 minutes or so to complete.

# Set Up Connection to the EKS Cluster

Connect to the bastion host via SSH or EC2 Instance Connect.

1. Execute the following commands to set up the environment variables and re-connect to the bastion host.

```bash
# Replace <> with the respective values
echo 'export AWS_ACCOUNT_ID=<AWS Account Id>' >> ~/.bashrc
echo 'export AWS_REGION=<AWS Region>' >> ~/.bashrc
echo 'export AWS_EKS_CLUSTER=<EKS Cluster Name>' >> ~/.bashrc
echo 'export CONTAINER_IMAGE_URL=<Container Image URL>' >> ~/.bashrc
```

2. Execute the following commands to configure the AWS CLI and update kubeconfig in the bastion host. Replace <> with the access key ID and secret access key respectively.

```bash
aws configure set aws_access_key_id <Your Access Key>
aws configure set aws_secret_access_key <Your Access Key Secret>
aws configure set region $AWS_REGION
aws configure set output json

aws eks --region $AWS_REGION update-kubeconfig --name $AWS_EKS_CLUSTER
```

3. Execute ```kubectl get svc``` to test the connectivity to the EKS cluster.

4. The output should be similar to the one provided below.

```
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
svc/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1m
```

# Set up AWS App Mesh (optional)

Note: These instructions are using github.com/tchangkiat/sample-express-api as the application. You may download the configuration (e.g. yaml, json) files and modify them to cater to your application.

1. Execute the following commands:

```bash
eksctl utils associate-iam-oidc-provider --region=$AWS_REGION --cluster=$AWS_EKS_CLUSTER --approve

kubectl create namespace appmesh-system

kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts

eksctl create iamserviceaccount --namespace appmesh-system --name appmesh-controller --attach-policy-arn arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess,arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --cluster $AWS_EKS_CLUSTER --approve

helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system --set region=$AWS_REGION --set serviceAccount.create=false --set serviceAccount.name=appmesh-controller

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-namespace.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualnode.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualrouter.yaml"

kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualservice.yaml"

curl https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/proxy-auth-cf.json -o proxy-auth-cf.json

aws cloudformation create-stack --stack-name AppMeshProxyAuthPolicy --template-body file://proxy-auth-cf.json --parameters ParameterKey=MeshName,ParameterValue=default-mesh --capabilities CAPABILITY_NAMED_IAM

eksctl create iamserviceaccount --cluster $AWS_EKS_CLUSTER --namespace default --name sample-express-api-service-account --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AppMeshProxyAuth-default-mesh --attach-policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --override-existing-serviceaccounts --approve

# Scale up an additional node to fit in 2 sets of sample-express-api and 1 set of AWS App Mesh Pods
eksctl scale nodegroup --cluster=$AWS_EKS_CLUSTER --nodes=3 --name `eksctl get nodegroup --cluster $AWS_EKS_CLUSTER | grep 'EKSNodeGroup' | awk '{print $2}'`
```

2. Deploy the sample application and verify if the Envoy containers are injected:

```bash
curl https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/deployment.yaml -o deployment.yaml

sed -i "s|\[URL\]|${CONTAINER_IMAGE_URL}|g" deployment.yaml

kubectl apply -f deployment.yaml
```

3. The Envoy containers should be injected in your application Pods automatically.

# Set up AWS X-Ray Integration (optional)

Note: You should set up App Mesh first before setting up AWS X-Ray Integration.

1. Execute the following commands:

```bash
helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system --set region=$AWS_REGION --set serviceAccount.create=false --set serviceAccount.name=appmesh-controller --set tracing.enabled=true --set tracing.provider=x-ray

kubectl rollout restart deployment sample-express-api
```

2. The X-Ray Daemon containers should be injected in your application Pods automatically.

2. Modify your source code to include and use the AWS X-Ray SDK (this was already done for the sample application).

# Clean Up

1. Execute the following command in the Bastion Host if the sample application was set up:

```bash
kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/deployment.yaml"
```

2. Execute the following commands in the Bastion Host if AWS App Mesh was set up:

```bash
kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualservice.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualrouter.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualnode.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh.yaml"

helm uninstall appmesh-controller --namespace appmesh-system
```

3. Delete all related CloudFormation stacks.
