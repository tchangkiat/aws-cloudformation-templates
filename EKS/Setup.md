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
aws eks --region <region-code> update-kubeconfig --name <cluster_name>
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