# EKS S3 IRSA — IAM Roles for Service Accounts

Demonstrates how to grant a Kubernetes workload running on Amazon EKS secure access to an S3 bucket using **IAM Roles for Service Accounts (IRSA)** — no static credentials required.

## How It Works

```
Pod (initContainer: aws-cli)
  └── uses ServiceAccount: app
        └── annotated with IAM Role ARN
              └── Role trusts EKS OIDC Provider
                    └── allows s3:GetObject on S3 bucket
```

The pod runs an `aws-cli` initContainer that downloads a file from S3. If the IAM trust chain is correctly configured, the file is fetched and the main nginx container starts. If not, the initContainer fails.

## Project Structure

```
eks-s3-irsa/
├── eks.yaml              # EKS cluster definition (eksctl)
├── iam-policy.json       # IAM policy — s3:GetObject on the bucket
├── Makefile              # make create / make delete
└── k8s/
    ├── sa.yaml           # ServiceAccount annotated with IAM Role ARN
    └── deployment.yaml   # nginx + aws-cli initContainer
```

## Quick Start

### 1. Create the EKS cluster

```bash
make create
```

### 2. Create the S3 bucket and upload a test file

```bash
aws s3 mb s3://<your-bucket-name> --region us-east-1
echo "Hello from S3 via IRSA" > test-file.txt
aws s3 cp test-file.txt s3://<your-bucket-name>/README.md
```

### 3. Associate the IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster cluster \
  --approve
```

### 4. Create the IAM Policy

```bash
aws iam create-policy \
  --policy-name S3TestAccessPolicy \
  --policy-document file://iam-policy.json
```

### 5. Create the IAM Role with trust policy

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_ID=$(aws eks describe-cluster --name cluster \
  --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5)

cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}:sub": "system:serviceaccount:default:app"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name S3TestAccessRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name S3TestAccessRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/S3TestAccessPolicy
```

### 6. Deploy to Kubernetes

```bash
kubectl apply -f k8s/sa.yaml
kubectl apply -f k8s/deployment.yaml
```

### 7. Verify

```bash
kubectl get pods
kubectl logs <pod-name> -c aws-cli
# Expected: Hello from S3 via IRSA
```

## Cleanup

```bash
kubectl delete -f k8s/
aws iam detach-role-policy --role-name S3TestAccessRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/S3TestAccessPolicy
aws iam delete-role --role-name S3TestAccessRole
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/S3TestAccessPolicy
aws s3 rm s3://<your-bucket-name> --recursive
aws s3 rb s3://<your-bucket-name>
make delete
```

## Cluster Config

| Setting | Value |
|---------|-------|
| Name | `cluster` |
| Region | `us-east-1` |
| Node type | `t2.small` |
| Nodes | 1 |
| Kubernetes | 1.34 |
