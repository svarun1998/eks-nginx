# Using AWS Systems Manager Parameter Store with Amazon EKS using External Secrets Operator (ESO)

This guide demonstrates how to securely fetch secrets from **AWS Systems Manager Parameter Store** and inject them into Kubernetes applications running on **Amazon EKS** using **External Secrets Operator (ESO)** and **IAM Roles for Service Accounts (IRSA)**.

The application in this example consumes the secrets as **environment variables**.

---

# Architecture

```text
                    AWS Systems Manager
                      Parameter Store
                             │
                             │
                             ▼
                   IAM Role (IRSA)
                             │
                             ▼
                External Secrets Operator
                             │
                             ▼
                      SecretStore
                             │
                             ▼
                    ExternalSecret
                             │
                             ▼
                  Kubernetes Secret
                             │
                             ▼
                  Nginx Deployment
                  (Environment Variables)
```

---

# Prerequisites

- Amazon EKS Cluster
- kubectl configured
- Helm installed
- AWS CLI configured
- OIDC Provider configured for the EKS cluster

---

# Step 1: Configure OIDC Provider

Configure an OIDC Identity Provider for your EKS cluster.

> **Note:** If your cluster already has an OIDC provider configured, you can skip this step.

---

# Step 2: Create Parameters in AWS Systems Manager Parameter Store

Create the required parameters in **AWS Systems Manager Parameter Store** before proceeding.

Example:

| Parameter Name | Type | Value |
|---------------|------|-------|
| `/myapp/username` | String | admin |
| `/myapp/password` | SecureString | MyPassword123 |

> **Note:** You can organize your parameters using any naming convention that suits your application (for example, `/myapp/*`, `/dev/myapp/*`, or `/prod/myapp/*`). Ensure the parameter names match those referenced in your `ExternalSecret` resource.

# Step 3: Create IAM Policy

Create an IAM Policy named:

```
ExternalSecretsSSMPolicy
```

Policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": [
                "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/myapp/*"
            ]
        }
    ]
}
```

Replace:

- `<ACCOUNT_ID>`
- `<REGION>`

---

# Step 4: Create IAM Role (IRSA)

Create an IAM Role.

Use the following trust relationship.

Replace:

- `<ACCOUNT_ID>`
- `<OIDC_PROVIDER_URL>`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER_URL>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "<OIDC_PROVIDER_URL>:aud": "sts.amazonaws.com",
                    "<OIDC_PROVIDER_URL>:sub": "system:serviceaccount:external-secrets:external-secrets"
                }
            }
        }
    ]
}
```

Attach the IAM Policy created in the previous step.

---

# Step 5: Install External Secrets Operator

Add the Helm repository.

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm repo update
```

Install External Secrets Operator.

```bash
helm install external-secrets external-secrets/external-secrets \
    --namespace external-secrets \
    --create-namespace
```

Verify installation.

```bash
kubectl get pods -n external-secrets
```

Expected output:

```
external-secrets
external-secrets-webhook
external-secrets-cert-controller
```

---

# Step 6: Configure IAM Role for Service Account (IRSA)

The External Secrets Operator Helm chart automatically creates a ServiceAccount named:

```text
external-secrets
```

Verify the ServiceAccount.

```bash
kubectl get sa -n external-secrets
```

Edit the ServiceAccount.

```bash
kubectl edit serviceaccount external-secrets -n external-secrets
```

Add the following annotation under `metadata`.

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<IAM_ROLE_NAME>
```

Example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets
  namespace: external-secrets
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eks-ssm-parameter-store-role
```

Save and exit the editor.

Verify that the annotation has been added.

```bash
kubectl describe serviceaccount external-secrets -n external-secrets
```

You should see:

```text
Annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eks-ssm-parameter-store-role
```

Restart the External Secrets Operator deployment so that the pods assume the IAM role.

```bash
kubectl rollout restart deployment external-secrets -n external-secrets
```

Wait for the new pods to become ready.

```bash
kubectl get pods -n external-secrets
```

# Step 7: Create SecretStore

Create `secret-store.yaml`.

> **Note:** Current versions of External Secrets Operator use **external-secrets.io/v1**.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-parameter-store
spec:
  provider:
    aws:
      service: ParameterStore
      region: us-east-1

      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
```

Apply.

```bash
kubectl apply -f secret-store.yaml
```

Verify.

```bash
kubectl get secretstore
```

Expected output.

```
NAME                  AGE    STATUS
aws-parameter-store   15s    Valid
```

---

# Step 8: Create ExternalSecret

Create `external-secret.yaml`.

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: myapp-secret
spec:

  refreshInterval: 1m

  secretStoreRef:
    name: aws-parameter-store
    kind: SecretStore

  target:
    name: myapp-secret

  data:
    - secretKey: USERNAME
      remoteRef:
        key: /myapp/username

    - secretKey: PASSWORD
      remoteRef:
        key: /myapp/password
```

Apply.

```bash
kubectl apply -f external-secret.yaml
```

Verify.

```bash
kubectl get externalsecret
```

---

# Step 9: Verify Kubernetes Secret

External Secrets Operator automatically creates a Kubernetes Secret.

```bash
kubectl get secrets
```

Expected.

```
myapp-secret
```

Describe the secret.

```bash
kubectl describe secret myapp-secret
```

---

# Step 10: Deploy Nginx Application

Create `nginx-deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 1

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:

      containers:

      - name: nginx

        image: nginx

        env:

        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: USERNAME

        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: PASSWORD
```

Deploy.

```bash
kubectl apply -f nginx-deployment.yaml
```

---

# Step 11: Verify Environment Variables

Since the official Nginx image doesn't include a shell, create a temporary BusyBox pod to verify the synced Kubernetes Secret or modify the deployment to use an image with a shell (e.g., Ubuntu or BusyBox) if you need to inspect environment variables interactively.

To verify the secret itself:

```bash
kubectl get secret myapp-secret -o yaml
```

Or decode a value:

```bash
kubectl get secret myapp-secret \
-o jsonpath="{.data.USERNAME}" | base64 --decode
```

Expected output:

```
admin
```

---

# Step 12: Test Automatic Secret Synchronization

Update one or more parameter values in **AWS Systems Manager Parameter Store**.

Wait for the configured `refreshInterval` (1 minute in this example).

Verify that the Kubernetes Secret has been synchronized with the latest values.

```bash
kubectl get secret myapp-secret -o yaml
```

If your application consumes secrets as environment variables, restart the deployment to load the updated values.

```bash
kubectl rollout restart deployment nginx
```

---

# Validation Checklist

- ✅ OIDC Provider configured
- ✅ IAM Policy created
- ✅ IAM Role (IRSA) created
- ✅ External Secrets Operator installed
- ✅ ServiceAccount annotated with IAM Role
- ✅ SecretStore created successfully
- ✅ ExternalSecret created successfully
- ✅ Kubernetes Secret synchronized
- ✅ Nginx deployment consumes secrets as environment variables
- ✅ Parameter Store updates synchronize automatically to Kubernetes Secret

---

# Cleanup

Delete the created resources.

```bash
kubectl delete deployment nginx
kubectl delete externalsecret myapp-secret
kubectl delete secretstore aws-parameter-store
helm uninstall external-secrets -n external-secrets
```

Optionally delete the IAM Role, IAM Policy, and Parameter Store entries from AWS.