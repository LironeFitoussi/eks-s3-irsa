# Lab: Access Amazon S3 from Kubernetes Using IAM Roles for Service Accounts (IRSA)

## Lab Objective

In this lab, you will configure a Kubernetes workload running on Amazon EKS to securely access files stored in an Amazon S3 bucket using **IAM Roles for Service Accounts (IRSA)**.

By the end of this lab, you will:

* Create an EKS cluster using `eksctl` and [eks.yaml](eks.yaml).
* Create an S3 bucket and upload a test file.
* Configure an IAM Identity Provider for EKS.
* Create an IAM Role (`S3TestAccessRole`) with S3 permissions using [iam-policy.json](iam-policy.json).
* Establish a trust relationship between the IAM Role and the Kubernetes Service Account `app`.
* Deploy an nginx pod that uses the Service Account and downloads the file from S3 via an initContainer.
* Verify that the file is read successfully from the S3 bucket.

---

## Prerequisites

Before starting, ensure you have:

* An AWS Account
* AWS CLI configured (`aws configure`)
* `eksctl` installed
* `kubectl` installed
* `make` installed (see below)
* Permissions to create: IAM Roles, IAM Policies, S3 Buckets, EKS Clusters

### Installing `make`

**Linux (Ubuntu/Debian):**

```bash
sudo apt update && sudo apt install -y make
```

**Linux (Amazon Linux / RHEL / CentOS):**

```bash
sudo yum install -y make
```

**Windows — Option 1: Chocolatey (recommended):**

```powershell
choco install make
```

**Windows — Option 2: Winget:**

```powershell
winget install GnuWin32.Make
```

**Windows — Option 3: WSL (Windows Subsystem for Linux):**

Run the lab inside a WSL terminal and use the Linux instructions above.

Verify installation:

```bash
make --version
```

---

## Project File Overview

```
eks-s3-irsa/
├── eks.yaml              # EKS cluster definition
├── iam-policy.json       # IAM policy for S3 access
├── Makefile              # Shortcuts for cluster create/delete
└── k8s/
    ├── sa.yaml           # Kubernetes ServiceAccount
    └── deployment.yaml   # nginx Deployment with aws-cli initContainer
```

---

## Step 1: Create the EKS Cluster

The cluster is defined in [eks.yaml](eks.yaml):

* **Cluster name:** `cluster`
* **Region:** `us-east-1`
* **Availability Zones:** `us-east-1a`, `us-east-1b`
* **Node group:** `on-demand` — 1x `t2.small`, 20GB volume

Create the cluster using the Makefile:

```bash
make create
```

This runs:

```bash
eksctl create cluster -f eks.yaml
```

Wait until the cluster is ready (approximately 15 minutes). Then verify:

```bash
kubectl get nodes
```

Expected output:

```text
NAME                          STATUS   ROLES    AGE   VERSION
ip-xxx-xxx-xxx-xxx.ec2...   Ready    <none>   2m    v1.xx.x
```

---

## Step 2: Create an S3 Bucket and Upload a Test File

Replace `<your-bucket-name>` with a globally unique name.

```bash
aws s3 mb s3://<your-bucket-name>
```

Create and upload a test file:

```bash
echo "Hello from S3 via IRSA" > README.md
aws s3 cp README.md s3://<your-bucket-name>/
```

Verify the upload:

```bash
aws s3 ls s3://<your-bucket-name>
```

Expected output:

```text
README.md
```

> **Note:** Update [k8s/deployment.yaml](k8s/deployment.yaml) line 21 to point to your bucket:
> ```yaml
> command: ['aws', 's3', 'cp', 's3://<your-bucket-name>/README.md', '-']
> ```

---

## Step 3: Associate an IAM OIDC Provider with EKS

IRSA requires an IAM OpenID Connect (OIDC) Provider linked to the cluster.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster cluster \
  --approve
```

Verify the OIDC issuer URL:

```bash
aws eks describe-cluster \
  --name cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Expected output (example):

```text
https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE123456789
```

---

## Step 4: Create an IAM Policy for S3 Access

The policy is already defined in [iam-policy.json](iam-policy.json):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::<your-bucket-name>/*"
      ]
    }
  ]
}
```

Before creating the policy, update the bucket name:

```bash
sed -i 's/<bucket-name>/<your-bucket-name>/g' iam-policy.json
```

Create the policy:

```bash
aws iam create-policy \
  --policy-name S3TestAccessPolicy \
  --policy-document file://iam-policy.json
```

Note the returned `Policy ARN` — you will need it in the next step.

---

## Step 5: Create an IAM Role for the Service Account

Create the IAM Role `S3TestAccessRole` that trusts the EKS OIDC Provider and allows only the `app` Service Account in the `default` namespace to assume it.

Get your AWS account ID and OIDC provider ID:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_ID=$(aws eks describe-cluster --name cluster \
  --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5)
```

Create the trust policy:

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
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
    }
  ]
}
EOF
```

Create the role:

```bash
aws iam create-role \
  --role-name S3TestAccessRole \
  --assume-role-policy-document file://trust-policy.json
```

Attach the S3 policy to the role:

```bash
aws iam attach-role-policy \
  --role-name S3TestAccessRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/S3TestAccessPolicy
```

---

## Step 6: Apply the Kubernetes Service Account

The Service Account is defined in [k8s/sa.yaml](k8s/sa.yaml):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::424432388155:role/S3TestAccessRole
```

Update the account ID if it differs from yours:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/424432388155/${ACCOUNT_ID}/g" k8s/sa.yaml
```

Apply the Service Account:

```bash
kubectl apply -f k8s/sa.yaml
```

Verify:

```bash
kubectl describe sa app
```

Confirm the annotation `eks.amazonaws.com/role-arn` is present.

---

## Step 7: Deploy the Application

The deployment is defined in [k8s/deployment.yaml](k8s/deployment.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: app
      initContainers:
      - name: aws-cli
        image: amazon/aws-cli
        command: ['aws', 's3', 'cp', 's3://<your-bucket-name>/README.md', '-']
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

The flow is:

1. The `aws-cli` **initContainer** assumes the IAM Role via IRSA and downloads `README.md` from S3.
2. If the download succeeds, the main **nginx** container starts.
3. If S3 access is denied, the initContainer fails and nginx never starts.

Apply the deployment:

```bash
kubectl apply -f k8s/deployment.yaml
```

---

## Step 8: Verify the Deployment

Check the pod status:

```bash
kubectl get pods
```

Expected output (after the initContainer completes):

```text
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-xxxxxxxxx-xxxxx   1/1     Running   0          1m
```

View the initContainer logs to confirm S3 access:

```bash
kubectl logs <pod-name> -c aws-cli
```

Expected output:

```text
Hello from S3 via IRSA
```

Describe the pod to confirm the Service Account is used:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Service Account:  app
```

---

## Validation Checklist

- [ ] EKS cluster `cluster` is running in `us-east-1`
- [ ] S3 bucket created and `README.md` uploaded
- [ ] OIDC Provider associated with the cluster
- [ ] IAM Policy `S3TestAccessPolicy` created from [iam-policy.json](iam-policy.json)
- [ ] IAM Role `S3TestAccessRole` created with correct trust policy
- [ ] Service Account `app` annotated with the IAM Role ARN ([k8s/sa.yaml](k8s/sa.yaml))
- [ ] Deployment `nginx-deployment` applied ([k8s/deployment.yaml](k8s/deployment.yaml))
- [ ] initContainer logs show the content of `README.md`
- [ ] nginx pod reaches `Running` status

---

## Common Issues

### initContainer fails with AccessDenied

Cause: IAM Role trust policy does not match the Service Account name/namespace, or the policy ARN is not attached.

Verify the role assumption:

```bash
kubectl exec -it <pod-name> -c aws-cli -- aws sts get-caller-identity
```

Confirm the role ARN in the output matches `S3TestAccessRole`.

---

### initContainer fails with NoSuchBucket

Cause: Bucket name in [k8s/deployment.yaml](k8s/deployment.yaml) does not match the bucket you created.

```bash
aws s3 ls
```

Update line 21 in [k8s/deployment.yaml](k8s/deployment.yaml) and re-apply.

---

### Pod stuck in `Init:0/1`

Cause: OIDC Provider not associated, or the Service Account annotation is missing/incorrect.

```bash
kubectl describe sa app
kubectl logs <pod-name> -c aws-cli
```

Re-run Step 3 and verify Step 6.

---

## Cleanup

Delete the Kubernetes resources:

```bash
kubectl delete -f k8s/deployment.yaml
kubectl delete -f k8s/sa.yaml
```

Delete the EKS cluster:

```bash
make delete
```

Delete IAM resources:

```bash
aws iam detach-role-policy \
  --role-name S3TestAccessRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/S3TestAccessPolicy

aws iam delete-role --role-name S3TestAccessRole
aws iam delete-policy --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/S3TestAccessPolicy
```

Delete the S3 bucket:

```bash
aws s3 rm s3://<your-bucket-name> --recursive
aws s3 rb s3://<your-bucket-name>
```
