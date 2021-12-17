# Setup Guide

## IAM User

Create an IAM user with the necessary permissions (refer to policy below) which has at least the CLI access (access key ID and secret access key required).

Policy Definition:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:*",
                "cloudwatch:*",
                "s3:*",
                "cloudformation:*",
                "ec2:*",
                "eks:*",
                "ec2-instance-connect:*"
            ],
            "Resource": "*"
        }
    ]
}
```

## CloudFormation

Use the IAM user created above to:
1. Create a stack with VPC.json in Network folder
2. Create a stack with EKS.json, with parameters referencing subnets and security groups created by VPC.json

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