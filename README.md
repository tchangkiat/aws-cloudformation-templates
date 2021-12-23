# AWS CloudFormation Templates
This repository contains CloudFormation templates and guides to simplify the processes for provisioning resources (e.g. creating an EKS cluster which can be managed by a bastion host).

## Disclaimer

Please review the templates and be aware of the resources created. I am not responsible for any cost incurred by the resources. You may modify the templates to remove the resources which you do not require.

## IAM User

Create an IAM user with the necessary permissions (refer to policy below) to create a stack with any templates in this repository. Please ensure that the user has the CLI access and create an access key ID and secret access key for the user. These permissions may not follow the principle of least priviledge - consider this user as an admin who has the priviledges to create resources using any provided CloudFormation templates in this repository.

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
                "ecr:*",
                "cloudformation:*",
                "ec2:*",
                "elasticloadbalancing:*",
                "xray:*",
                "eks:*",
                "ec2-instance-connect:*",
                "appmesh:*",
                "codebuild:*",
                "ssm:GetParameters"
            ],
            "Resource": "*"
        }
    ]
}
```
