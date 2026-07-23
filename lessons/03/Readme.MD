# Amazon EKS Cluster Autoscaler using IAM Roles for Service Accounts (IRSA)

This guide demonstrates how to install and configure the **Kubernetes Cluster Autoscaler** on an **Amazon EKS** cluster using **IAM Roles for Service Accounts (IRSA)**.

The Cluster Autoscaler automatically adjusts the size of your Amazon EKS Managed Node Group by increasing the number of worker nodes when Pods cannot be scheduled due to insufficient resources and scaling down when nodes become underutilized.

---

# Architecture

```text
                    Kubernetes Scheduler
                             │
                             ▼
                     Pending Pods Detected
                             │
                             ▼
                    Cluster Autoscaler
                         (using IRSA)
                             │
                             ▼
                 Amazon EC2 Auto Scaling Group
                             │
                             ▼
              Amazon EKS Managed Node Group
                             │
                             ▼
                 New EC2 Worker Node Added
                             │
                             ▼
                  Pending Pods Scheduled
```

---

# Prerequisites

- Amazon EKS Cluster
- Amazon EKS Managed Node Group
- kubectl configured
- Helm installed
- AWS CLI configured
- OIDC Provider configured for the EKS Cluster

---

# Step 1: Configure OIDC Provider

Configure an OIDC Identity Provider for your EKS cluster.

> **Note:** If your cluster already has an OIDC provider configured, you can skip this step.

---

# Step 2: Tag the Auto Scaling Group

The Cluster Autoscaler discovers Auto Scaling Groups based on specific tags.

Navigate to:

```
AWS Console
    └── EC2
          └── Auto Scaling Groups
```

Select the Auto Scaling Group associated with your EKS Managed Node Group and add the following tags.

| Key | Value |
|------|-------|
| k8s.io/cluster-autoscaler/enabled | true |
| k8s.io/cluster-autoscaler/<CLUSTER_NAME> | owned |

Example:

| Key | Value |
|------|-------|
| k8s.io/cluster-autoscaler/enabled | true |
| k8s.io/cluster-autoscaler/my-eks-cluster | owned |

---

# Step 3: Create IAM Policy

Create an IAM Policy named:

```
AmazonEKSClusterAutoscalerPolicy
```

Policy Document:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:DescribeTags",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*"
        }
    ]
}
```

---

# Step 4: Create IAM Role (IRSA)

Create an IAM Role using the following trust relationship.

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
                    "<OIDC_PROVIDER_URL>:sub": "system:serviceaccount:kube-system:cluster-autoscaler"
                }
            }
        }
    ]
}
```

Attach the IAM Policy created in the previous step to this IAM Role.

---

# Step 5: Create Kubernetes Service Account

Create a Service Account named `cluster-autoscaler`.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: <IAM_ROLE_ARN>
```

Apply the Service Account.

```bash
kubectl apply -f service-account.yaml
```

Verify.

```bash
kubectl get serviceaccount cluster-autoscaler -n kube-system
```

---

# Step 6: Install Cluster Autoscaler

Add the Helm repository.

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm repo update
```

Install Cluster Autoscaler.

Replace:

- `<CLUSTER_NAME>`
- `<AWS_REGION>`

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
    --namespace kube-system \
    --set autoDiscovery.clusterName=<CLUSTER_NAME> \
    --set awsRegion=<AWS_REGION> \
    --set rbac.serviceAccount.create=false \
    --set rbac.serviceAccount.name=cluster-autoscaler
```

---

# Step 7: Verify Installation

Verify the deployment.

```bash
kubectl get deployment cluster-autoscaler -n kube-system
```

Verify the Pod.

```bash
kubectl get pods -n kube-system
```

View the logs.

```bash
kubectl logs deployment/cluster-autoscaler -n kube-system
```

The logs should indicate that the Auto Scaling Group has been discovered successfully.

---

# Step 8: Deploy a Sample Workload

Deploy an application that requests more CPU and memory than the current worker nodes can provide.

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress-app

spec:
  replicas: 10

  selector:
    matchLabels:
      app: stress-app

  template:
    metadata:
      labels:
        app: stress-app

    spec:
      containers:
      - name: nginx
        image: nginx

        resources:
          requests:
            cpu: "800m"
            memory: "1Gi"

          limits:
            cpu: "800m"
            memory: "1Gi"
```

Deploy the application.

```bash
kubectl apply -f stress-app.yaml
```

---

# Step 9: Verify Pending Pods

Initially, some Pods should remain in the **Pending** state because the cluster does not have enough available resources.

```bash
kubectl get pods
```

Describe a Pending Pod.

```bash
kubectl describe pod <POD_NAME>
```

You should see scheduling events similar to:

```
0/2 nodes are available: Insufficient cpu.
```

---

# Step 10: Verify Cluster Autoscaler

Watch the Cluster Autoscaler logs.

```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

You should observe messages indicating that the Cluster Autoscaler has detected unschedulable Pods and increased the desired capacity of the Auto Scaling Group.

---

# Step 11: Verify Worker Nodes

Watch the worker nodes joining the cluster.

```bash
kubectl get nodes --watch
```

After the new nodes become Ready, the Pending Pods should automatically transition to the Running state.

Verify.

```bash
kubectl get pods
```

---

# Step 12: Test Scale Down

Delete the sample deployment.

```bash
kubectl delete deployment stress-app
```

Wait for the Cluster Autoscaler scale-down delay (approximately 10 minutes by default).

The Cluster Autoscaler automatically terminates underutilized worker nodes.

Verify.

```bash
kubectl get nodes
```

---

# Validation Checklist

- ✅ OIDC Provider configured
- ✅ Auto Scaling Group tagged correctly
- ✅ IAM Policy created
- ✅ IAM Role (IRSA) created
- ✅ Kubernetes Service Account created
- ✅ Cluster Autoscaler installed successfully
- ✅ Cluster Autoscaler discovered the Managed Node Group
- ✅ Pending Pods triggered automatic scale-out
- ✅ New worker nodes joined the cluster
- ✅ Pending Pods became Running
- ✅ Worker nodes scaled down after workload removal

---

# Cleanup

Delete the sample workload.

```bash
kubectl delete deployment stress-app
```

Uninstall Cluster Autoscaler.

```bash
helm uninstall cluster-autoscaler -n kube-system
```

Optionally remove:

- IAM Role
- IAM Policy
- Kubernetes Service Account
- Auto Scaling Group tags (if no longer required)

---

# References

- Kubernetes Cluster Autoscaler
- Amazon EKS Best Practices Guide
- IAM Roles for Service Accounts (IRSA)