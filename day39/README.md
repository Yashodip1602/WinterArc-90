# Day 39: AWS EKS IAM, Cluster Access and IRSA

## Goal
Give people and apps only the permissions they need.

- Engineer: read-only Pod access in namespace `dev`
- App Pod: read-only access to one S3 bucket via IRSA

## Quick Concepts

| Term | Meaning |
|---|---|
| IAM | What an identity can do in AWS services |
| Kubernetes RBAC | What an identity can do inside the cluster |
| Access entry | Maps an IAM principal to EKS cluster access |
| IRSA | ServiceAccount assumes an IAM role via OIDC |

## Part 1: Inspect Cluster (Read-only)

| Command | What it does |
|---|---|
| `aws sts get-caller-identity` | Shows which IAM identity I am using |
| `aws eks list-clusters --region REGION` | Lists EKS clusters in the region |
| `aws eks describe-cluster --name CLUSTER_NAME --region REGION --query "cluster.accessConfig.authenticationMode" --output text` | Shows auth mode (`API`, `API_AND_CONFIG_MAP`, `CONFIG_MAP`) |
| `aws eks list-access-entries --cluster-name CLUSTER_NAME --region REGION` | Lists IAM principals with cluster access |
| `kubectl get serviceaccounts -A` | Lists ServiceAccounts in all namespaces |

## Part 2: RBAC Setup (Engineer Read-only in `dev`)

Create namespace:

```bash
kubectl create namespace dev
```

`pod-viewer-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-pod-viewer
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

`pod-viewer-binding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-viewer-binding
  namespace: dev
subjects:
  - kind: Group
    name: dev-pod-viewers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f pod-viewer-role.yaml
kubectl apply -f pod-viewer-binding.yaml
```

### Verify

```bash
kubectl auth can-i list pods -n dev --as=test --as-group=dev-pod-viewers
kubectl auth can-i delete pods -n dev --as=test --as-group=dev-pod-viewers
kubectl auth can-i list pods -n default --as=test --as-group=dev-pod-viewers
```

| Command | Result | Why |
|---|---|---|
| list pods in `dev` | yes | Role allows it |
| delete pods in `dev` | no | Verb not allowed |
| list pods in `default` | no | Role is limited to `dev` |

## Part 3: IRSA Setup (App Reads One S3 Bucket)

### Step 1: Get OIDC issuer

```bash
aws eks describe-cluster --name CLUSTER_NAME --region REGION \
  --query "cluster.identity.oidc.issuer" --output text
```

### Step 2: Check OIDC provider exists in IAM

```bash
aws iam list-open-id-connect-providers
```

If missing, create it:

```bash
eksctl utils associate-iam-oidc-provider --cluster CLUSTER_NAME --region REGION --approve
```

### Step 3: Create ServiceAccount

```bash
kubectl create serviceaccount s3-reader-sa -n dev
```

### Step 4: Trust policy (`trust-policy.json`)

Only namespace `dev` and ServiceAccount `s3-reader-sa` can assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/OIDC_ISSUER_WITHOUT_HTTPS"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "OIDC_ISSUER_WITHOUT_HTTPS:aud": "sts.amazonaws.com",
          "OIDC_ISSUER_WITHOUT_HTTPS:sub": "system:serviceaccount:dev:s3-reader-sa"
        }
      }
    }
  ]
}
```

```bash
aws iam create-role --role-name eks-dev-s3-reader \
  --assume-role-policy-document file://trust-policy.json
```

### Step 5: Permission policy (`s3-read-policy.json`)

One bucket only, read-only.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    }
  ]
}
```

```bash
aws iam put-role-policy --role-name eks-dev-s3-reader \
  --policy-name s3-read-one-bucket \
  --policy-document file://s3-read-policy.json
```

### Step 6: Annotate ServiceAccount

```bash
kubectl annotate serviceaccount s3-reader-sa -n dev \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT_ID:role/eks-dev-s3-reader
```

### Step 7: Test Pod

```bash
kubectl run s3-test -n dev --image=amazon/aws-cli \
  --overrides='{"spec":{"serviceAccountName":"s3-reader-sa"}}' \
  --command -- sleep 3600
```

### Step 8: Verify

```bash
kubectl exec -n dev s3-test -- aws sts get-caller-identity
kubectl exec -n dev s3-test -- aws s3 ls s3://BUCKET_NAME
kubectl exec -n dev s3-test -- aws s3 cp /etc/hostname s3://BUCKET_NAME/test.txt
```

| Command | Expected |
|---|---|
| `sts get-caller-identity` | Shows the assumed role `eks-dev-s3-reader` |
| `s3 ls` on allowed bucket | Success |
| `s3 cp` (write) | AccessDenied |

## IRSA Flow

Pod -> ServiceAccount -> projected OIDC token -> IAM role trust policy -> temporary credentials -> allowed S3 read

## IAM vs Kubernetes RBAC

| | AWS IAM | Kubernetes RBAC |
|---|---|---|
| Controls | AWS services | Kubernetes API objects |
| Scope | AWS account | Cluster and namespace |
| Example | `s3:GetObject` | `get pods` in `dev` |

## Why IRSA Is Safer Than Node Role

- Each app gets its own role, not a shared one.
- Trust policy locks the role to one namespace and one ServiceAccount.
- Credentials are temporary and auto-rotated.
- No hard-coded access keys in code or images.
- Node role stays small, only for node operations.
- A hacked Pod gets only one bucket read access, not the whole node permissions.

## Cleanup (only my lab resources)

```bash
kubectl delete pod s3-test -n dev
kubectl delete serviceaccount s3-reader-sa -n dev
kubectl delete rolebinding dev-pod-viewer-binding -n dev
kubectl delete role dev-pod-viewer -n dev
aws iam delete-role-policy --role-name eks-dev-s3-reader --policy-name s3-read-one-bucket
aws iam delete-role --role-name eks-dev-s3-reader
```

Do not delete the cluster or shared resources.

## Security Notes

- No AdministratorAccess used.
- Account ID, keys and tokens hidden in screenshots.
- Auth mode was not changed.
