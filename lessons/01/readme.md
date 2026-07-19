# AWS Load Balancer Controller Setup for Amazon EKS

This guide explains how to install and configure the **AWS Load Balancer Controller** to expose applications running on an Amazon EKS cluster using an AWS Application Load Balancer (ALB).

---

## Prerequisites

- Amazon EKS Cluster
- `kubectl` configured
- Helm installed
- AWS CLI configured with appropriate permissions

---

## Step 1: Configure OIDC Provider

Configure an **OIDC Identity Provider** for your EKS cluster.

> **Note:** If your cluster already has an OIDC provider configured, you can skip this step.

You can verify whether OIDC is already configured before proceeding.

---

## Step 2: Create IAM Policy and IAM Role

### 2.1 Create IAM Policy

Create an IAM policy named:

```text
AWSLoadBalancerControllerIAMPolicy
```

The required policy document is available in:

```text
aws_loadbalancer_controller_iam_policy.json
```

Create the policy using the AWS Console or AWS CLI.

---

### 2.2 Create IAM Role (IRSA)

Create an IAM Role using the following **OIDC Federated Trust Policy**.

Replace the placeholders:

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
          "<OIDC_PROVIDER_URL>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
```

After creating the IAM Role, attach the following policy:

```text
AWSLoadBalancerControllerIAMPolicy
```

---

## Step 3: Create Kubernetes Service Account

Create the Service Account by applying:

```bash
kubectl apply -f service_account.yaml
```

Ensure the Service Account contains the following annotation:

```yaml
annotations:
  eks.amazonaws.com/role-arn: <IAM_ROLE_ARN>
```

Replace `<IAM_ROLE_ARN>` with the IAM Role ARN created in the previous step.

---

## Step 4: Install AWS Load Balancer Controller

### Add the Helm Repository

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

### Install the Controller

Replace the placeholders below:

- `<CLUSTER_NAME>`
- `<AWS_REGION>`
- `<VPC_ID>`

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set region=<AWS_REGION> \
  --set vpcId=<VPC_ID> \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Verify Installation

```bash
kubectl get deployment aws-load-balancer-controller -n kube-system
```

Expected output:

```text
NAME                             READY   UP-TO-DATE   AVAILABLE
aws-load-balancer-controller     1/1     1            1
```

---

## Step 5: Tag the VPC Subnets

The AWS Load Balancer Controller discovers subnets based on subnet tags.

Add the following tag to every subnet that should be used for the Application Load Balancer.

| Key | Value |
|------|-------|
| kubernetes.io/role/elb | 1 |

> **Important**
>
> If these tags are not present, the AWS Load Balancer Controller cannot determine which subnets should be used to provision the Application Load Balancer.

---

## Step 6: Create the Ingress Resource

Deploy the Ingress manifest:

```bash
kubectl apply -f ingress_nginx.yaml
```

Verify that the Ingress resource is created:

```bash
kubectl get ingress
```

---

## Validation

After the Ingress is created successfully, the AWS Load Balancer Controller automatically:

- Creates an **Application Load Balancer (ALB)**.
- Creates the required **Security Groups**.
- Creates **Target Groups**.
- Registers the Kubernetes Service endpoints with the Target Group.
- Routes external traffic to the application pods.

Verify the following resources:

```bash
kubectl get ingress
```

```bash
kubectl get svc
```

```bash
kubectl get pods
```

> **Important**
>
> Ensure your application is exposed through a Kubernetes **Service** of type **ClusterIP** and that the Service correctly selects the application pods using labels.
>
> If the Service is not created correctly or is not associated with the application pods, the AWS Load Balancer Controller will **not** create or populate the Target Group, and traffic will not reach your application.

---

## Deployment Flow

```text
Application Pods
        │
        ▼
ClusterIP Service
        │
        ▼
Ingress
        │
        ▼
AWS Load Balancer Controller
        │
        ▼
Application Load Balancer (ALB)
        │
        ▼
Internet
```