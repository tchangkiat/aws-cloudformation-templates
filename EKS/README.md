# Create Resources

1. Create a stack with VPC.json in Network folder.
2. Create an EC2 key pair separately, which will be used to SSH into the bastion host.
3. Create a stack with EKS.json, with parameters referencing subnets created by VPC.json and the key pair which you have created in step 2. CloudFormation will take about 25 minutes or so to provision the resources in this template.

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

3. Execute `kubectl get svc` to test the connectivity to the EKS cluster.

4. The output should be similar to the one provided below.

```
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
svc/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1m
```

# Set up AWS App Mesh (optional)

Note: These instructions are using github.com/tchangkiat/sample-express-api as the application. You may download the configuration (e.g. yaml, json) files and modify them for your application.

1. Execute the following commands:

```bash
# Create an IAM OIDC provider for your cluster. This is required to use IAM roles for service accounts
eksctl utils associate-iam-oidc-provider --region=$AWS_REGION --cluster=$AWS_EKS_CLUSTER --approve

# Create a namespace for App Mesh
kubectl create namespace appmesh-system

# Add the EKS repository to Helm
helm repo add eks https://aws.github.io/eks-charts

# Install the App Mesh CRDs
kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"

# Create IAM role and service account pair for App Mesh
eksctl create iamserviceaccount --namespace appmesh-system --name appmesh-controller --attach-policy-arn arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess,arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --cluster $AWS_EKS_CLUSTER --approve

# Install App Mesh Controller
helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system --set region=$AWS_REGION --set serviceAccount.create=false --set serviceAccount.name=appmesh-controller

# Configure default namespace for mesh
kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-namespace.yaml"

# Create the mesh
kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh.yaml"

# Create the virtual node
kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualnode.yaml"

# Create the virtual router
kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualrouter.yaml"

# Create the virtual service
kubectl apply -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualservice.yaml"

# Download CloudFormation template to create a policy for proxy authorization for App Mesh injector to add the sidecar containers to any pod deployed with a label specified
curl https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/proxy-auth-cf.json -o proxy-auth-cf.json

# Create a stack with the template above
aws cloudformation create-stack --stack-name AppMeshProxyAuthPolicy --template-body file://proxy-auth-cf.json --parameters ParameterKey=MeshName,ParameterValue=default-mesh --capabilities CAPABILITY_NAMED_IAM

# Create IAM role and service account pair for the application
eksctl create iamserviceaccount --cluster $AWS_EKS_CLUSTER --namespace default --name sample-express-api-service-account --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AppMeshProxyAuth-default-mesh --attach-policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess --override-existing-serviceaccounts --approve
```

2. Deploy the sample application:

```bash
# Download the YAML file for deploying the application in the Kubernetes cluster
curl https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/deployment.yaml -o deployment.yaml

# Replace [URL] with the URL of the container image
sed -i "s|\[URL\]|${CONTAINER_IMAGE_URL}|g" deployment.yaml

# Create the deployment
kubectl apply -f deployment.yaml
```

3. The Envoy containers should be injected in your application Pods automatically.

# Set up AWS X-Ray Integration (optional)

Note: You should set up App Mesh first before setting up AWS X-Ray Integration.

1. Execute the following commands:

```bash
# Update the App Mesh Controller to enable X-Ray so that the X-Ray Daemon will be added automatically in the Pods
helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system --set region=$AWS_REGION --set serviceAccount.create=false --set serviceAccount.name=appmesh-controller --set tracing.enabled=true --set tracing.provider=x-ray

# Restart the deployment for the X-Ray Daemon to be injected in the Pods
kubectl rollout restart deployment sample-express-api
```

2. The X-Ray Daemon containers should be injected in your application Pods automatically.

3. Modify your source code to include and use the AWS X-Ray SDK (this was already done for the sample application).

# Set up logging with Amazon OpenSearch and Fluent Bit (optional)

Credit: [EKS Workshop](https://www.eksworkshop.com/intermediate/230_logging/).

1. Create an OpenSearch Service cluster using OpenSearch.json in the root folder.

2. Execute the following commands to set up the cluster and the necessary IAM roles and Kubernetes service account for logging:

```bash
# Create an IAM OIDC provider for your cluster. This is required to use IAM roles for service accounts
eksctl utils associate-iam-oidc-provider --region=$AWS_REGION --cluster=$AWS_EKS_CLUSTER --approve

# Create a new namespace in the EKS cluster
kubectl create namespace logging

# Replace the value with the domain name of the OpenSearch service cluster which you have created in Step 1
export AWS_OSS_DOMAIN_NAME=oss-cluster
echo 'export AWS_OSS_DOMAIN_NAME=oss-cluster' >> ~/.bashrc

# Create a policy for the Fluent Bit service account and IAM role pair
cat <<EoF > fluent-bit-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "es:ESHttp*"
            ],
            "Resource": "arn:aws:es:${AWS_REGION}:${AWS_ACCOUNT_ID}:domain/${AWS_OSS_DOMAIN_NAME}/*",
            "Effect": "Allow"
        }
    ]
}
EoF

aws iam create-policy --policy-name fluent-bit-policy --policy-document file://~/fluent-bit-policy.json

eksctl create iamserviceaccount \
    --name fluent-bit \
    --namespace logging \
    --cluster ${AWS_EKS_CLUSTER} \
    --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/fluent-bit-policy" \
    --approve \
    --override-existing-serviceaccounts
```

3. Execute the following commands to map roles to users for OpenSearch Service cluster

```bash
# Replace the value with the master account username of the OpenSearch service cluster
export AWS_OSS_MASTER_ACCOUNT=oss-user

# Replace the value with the master account password of the OpenSearch service cluster
export AWS_OSS_MASTER_ACCOUNT_PASSWORD=<Password>

# Get the Fluent Bit Role ARN
export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster $AWS_EKS_CLUSTER --namespace logging -o json | jq '.[].status.roleARN' -r)
echo "export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster ${AWS_EKS_CLUSTER} --namespace logging -o json | jq '.[].status.roleARN' -r)" >> ~/.bashrc

# Get the Amazon OpenSearch Endpoint
export AWS_OSS_ENDPOINT=$(aws opensearch describe-domain --domain-name $AWS_OSS_DOMAIN_NAME --output text --query "DomainStatus.Endpoint")
echo "export AWS_OSS_ENDPOINT=$(aws opensearch describe-domain --domain-name ${AWS_OSS_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")" >> ~/.bashrc

# Update the OpenSearch internal database
curl -sS -u "${AWS_OSS_MASTER_ACCOUNT}:${AWS_OSS_MASTER_ACCOUNT_PASSWORD}" \
    -X PATCH \
    https://${AWS_OSS_ENDPOINT}/_opendistro/_security/api/rolesmapping/all_access?pretty \
    -H 'Content-Type: application/json' \
    -d'
[
  {
    "op": "add", "path": "/backend_roles", "value": ["'${FLUENTBIT_ROLE}'"]
  }
]
'
```

4. Execute the following commands to deploy Fluent Bit:

```bash
curl -Ss https://raw.githubusercontent.com/tchangkiat/aws-cloudformation-templates/main/EKS/logging/fluentbit.yaml \
    | envsubst > ~/fluentbit.yaml

kubectl apply -f ~/fluentbit.yaml
```

5. Run the following command to get the URL to access the OpenSearch Dashboards. Use the master account username and password. Follow [this guide](https://www.eksworkshop.com/intermediate/230_logging/kibana/) to complete the set up in the dashboards.

```bash
echo "OpenSearch Dashboards URL: https://${AWS_OSS_ENDPOINT}/_dashboards/"
```

# Clean Up

1. Execute the following command in the Bastion Host if logging (Fluent Bit + OpenSearch) was set up:

```bash
kubectl delete -f ~/environment/logging/fluentbit.yaml

eksctl delete iamserviceaccount \
    --name fluent-bit \
    --namespace logging \
    --cluster ${AWS_EKS_CLUSTER} \
    --wait

aws iam delete-policy   \
  --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/fluent-bit-policy"

kubectl delete namespace logging
```

2. Execute the following command in the Bastion Host if the sample application was set up:

```bash
kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/deployment.yaml"
```

3. Execute the following commands in the Bastion Host if AWS App Mesh was set up:

```bash
kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualservice.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualrouter.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh-virtualnode.yaml"

kubectl delete -f "https://raw.githubusercontent.com/tchangkiat/sample-express-api/master/k8s/eks/appmesh.yaml"

helm uninstall appmesh-controller --namespace appmesh-system
```

4. Delete all related CloudFormation stacks.
