# Lab 2: Convert IRSA to EKS Pod Identity

## Lab Objective

In this lab, you will migrate the S3 access setup from **IRSA (IAM Roles for Service Accounts)** to **EKS Pod Identity** — a newer, simpler approach introduced by AWS in 2023.

By the end of this lab, you will:

* Understand the differences between IRSA and Pod Identity.
* Install the EKS Pod Identity Agent addon.
* Create a new IAM Role with a Pod Identity trust policy.
* Create a Pod Identity Association linking the role to the Kubernetes Service Account.
* Remove the IAM role annotation from the Service Account.
* Verify that the pod still accesses S3 — now via Pod Identity.

---

## IRSA vs Pod Identity

| | IRSA | Pod Identity |
|---|---|---|
| Setup per cluster | OIDC provider required | Pod Identity Agent addon required |
| IAM Role trust | Trusts OIDC provider (per cluster) | Trusts `pods.eks.amazonaws.com` (reusable across clusters) |
| SA annotation | Required (`eks.amazonaws.com/role-arn`) | Not required |
| Association managed in | Kubernetes SA YAML | AWS EKS API |
| IAM Role reuse across clusters | Requires new trust policy per cluster | Works as-is |

---

## Prerequisites

* Lab 1 completed — EKS cluster `cluster` is running in `us-east-1`
* S3 bucket `irsa-lab-050752632489` exists with `README.md`
* IAM Policy `S3TestAccessPolicy` already created
* `make` installed (see Lab 1 prerequisites for installation instructions)

---

## Project Files

```
eks-s3-irsa/
├── pod-identity-trust-policy.json   # Trust policy for Pod Identity role
└── k8s-pod-identity/
    ├── sa.yaml                       # ServiceAccount WITHOUT role annotation
    └── deployment.yaml               # Same nginx + aws-cli deployment
```

---

## Step 1: Install the EKS Pod Identity Agent Addon

Pod Identity requires a DaemonSet agent running on each node.

```bash
aws eks create-addon \
  --cluster-name cluster \
  --addon-name eks-pod-identity-agent
```

Wait for it to become active:

```bash
aws eks describe-addon \
  --cluster-name cluster \
  --addon-name eks-pod-identity-agent \
  --query "addon.status"
```

Expected output:

```text
"ACTIVE"
```

Verify the agent pods are running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

Expected output:

```text
NAME                           READY   STATUS    RESTARTS   AGE
eks-pod-identity-agent-xxxxx   1/1     Running   0          1m
```

---

## Step 2: Create an IAM Role with Pod Identity Trust Policy

The trust policy is in [pod-identity-trust-policy.json](pod-identity-trust-policy.json):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "pods.eks.amazonaws.com"
    },
    "Action": [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }]
}
```

Key difference from IRSA: the principal is the AWS service `pods.eks.amazonaws.com` — not the OIDC provider. This means the same role can be reused across multiple clusters without updating the trust policy.

Create the role:

```bash
aws iam create-role \
  --role-name S3PodIdentityRole \
  --assume-role-policy-document file://pod-identity-trust-policy.json
```

Attach the existing S3 policy:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam attach-role-policy \
  --role-name S3PodIdentityRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/S3TestAccessPolicy
```

---

## Step 3: Create the Pod Identity Association

Link the IAM Role to the Kubernetes Service Account `app` in the `default` namespace:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws eks create-pod-identity-association \
  --cluster-name cluster \
  --namespace default \
  --service-account app \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/S3PodIdentityRole
```

Verify the association:

```bash
aws eks list-pod-identity-associations --cluster-name cluster
```

Expected output includes an entry for `namespace: default`, `serviceAccount: app`.

---

## Step 4: Apply the Updated Service Account

The new Service Account in [k8s-pod-identity/sa.yaml](k8s-pod-identity/sa.yaml) has **no annotation** — the role binding is managed entirely by AWS:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: default
```

Apply it (this overwrites the annotated IRSA version):

```bash
kubectl apply -f k8s-pod-identity/sa.yaml
```

Confirm the annotation is gone:

```bash
kubectl describe sa app
```

The `eks.amazonaws.com/role-arn` annotation should no longer appear.

---

## Step 5: Redeploy

Apply the deployment from [k8s-pod-identity/deployment.yaml](k8s-pod-identity/deployment.yaml):

```bash
kubectl apply -f k8s-pod-identity/deployment.yaml
```

Restart the pods to pick up the updated Service Account:

```bash
kubectl rollout restart deployment/nginx-deployment
```

---

## Step 6: Verify

Wait for the pod to be ready:

```bash
kubectl get pods -w
```

Expected:

```text
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-xxxxxxxxx-xxxxx   1/1     Running   0          30s
```

Check the initContainer logs:

```bash
POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -c aws-cli
```

Expected output:

```text
Hello from S3 via IRSA
```

S3 access works — now via Pod Identity instead of IRSA.

---

## Validation Checklist

- [ ] Pod Identity Agent addon is `ACTIVE`
- [ ] IAM Role `S3PodIdentityRole` created with Pod Identity trust policy
- [ ] `S3TestAccessPolicy` attached to `S3PodIdentityRole`
- [ ] Pod Identity Association created for `default/app`
- [ ] ServiceAccount `app` has no `eks.amazonaws.com/role-arn` annotation
- [ ] Pod reaches `Running` status
- [ ] initContainer logs show `Hello from S3 via IRSA`

---

## How Pod Identity Works Internally

```
Pod starts
  └── kubelet contacts Pod Identity Agent (DaemonSet on node)
        └── Agent calls EKS API: "what role is associated with this SA?"
              └── EKS returns S3PodIdentityRole ARN
                    └── Agent calls sts:AssumeRole + sts:TagSession
                          └── Injects temporary credentials into pod via
                              AWS_CONTAINER_CREDENTIALS_FULL_URI env var
```

The credentials are injected the same way as IRSA — pods use the standard AWS SDK credential chain and don't need any code changes.

---

## Cleanup

```bash
# Remove Pod Identity Association
aws eks list-pod-identity-associations --cluster-name cluster \
  --query 'associations[].associationId' --output text | \
  xargs -I{} aws eks delete-pod-identity-association \
    --cluster-name cluster --association-id {}

# Remove IAM Role
aws iam detach-role-policy \
  --role-name S3PodIdentityRole \
  --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/S3TestAccessPolicy

aws iam delete-role --role-name S3PodIdentityRole

# Remove addon
aws eks delete-addon --cluster-name cluster --addon-name eks-pod-identity-agent

# Remove k8s resources
kubectl delete -f k8s-pod-identity/
```
