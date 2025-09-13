# EKS + Kubernetes RBAC + IAM Roles for Service Accounts (IRSA) Demo

## Overview

This project demonstrates the integration of Amazon EKS (Elastic Kubernetes Service) with Kubernetes Role-Based Access Control (RBAC) and IAM Roles for Service Accounts (IRSA). The demo showcases how to securely manage AWS resource access from Kubernetes pods by mapping Kubernetes service accounts to AWS IAM roles.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Account                            │
│  ┌─────────────────┐    ┌──────────────────────────────────┐  │
│  │   IAM Role      │    │            EKS Cluster          │  │
│  │ (IRSA Enabled)  │◄───┤                                  │  │
│  │                 │    │  ┌─────────────────────────────┐ │  │
│  └─────────────────┘    │  │      Kubernetes Pod         │ │  │
│                         │  │  ┌─────────────────────────┐ │ │  │
│  ┌─────────────────┐    │  │  │   Service Account       │ │ │  │
│  │   AWS Services  │◄───┼──┼──┤   (IRSA Annotated)     │ │ │  │
│  │ (S3, DynamoDB,  │    │  │  └─────────────────────────┘ │ │  │
│  │  etc.)          │    │  │                              │ │  │
│  └─────────────────┘    │  └─────────────────────────────┘ │  │
│                         │                                  │  │
│                         │  ┌─────────────────────────────┐ │  │
│                         │  │        RBAC Rules           │ │  │
│                         │  │ (Roles, RoleBindings, etc.) │ │  │
│                         │  └─────────────────────────────┘ │  │
│                         └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Technology Stack

- **Kubernetes**: Container orchestration platform
- **AWS EKS**: Managed Kubernetes service
- **RBAC**: Kubernetes Role-Based Access Control
- **IAM**: AWS Identity and Access Management
- **IRSA**: IAM Roles for Service Accounts
- **Terraform**: Infrastructure as Code tool

## Prerequisites

- AWS CLI configured with appropriate permissions
- kubectl installed and configured
- Terraform >= 1.0
- eksctl (optional, for cluster management)
- Docker (for building container images)
- Helm (optional, for package management)

## Step-by-Step Implementation

### 1. Deploy App to EKS

First, deploy a sample application to your EKS cluster:

```bash
# Apply the deployment
kubectl apply -f k8s/deployment.yaml

# Verify deployment
kubectl get pods -n demo-namespace
```

### 2. Configure RBAC for Access Control

Set up RBAC rules to control access within the cluster:

```bash
# Create namespace
kubectl create namespace demo-namespace

# Apply RBAC configurations
kubectl apply -f k8s/rbac.yaml

# Verify RBAC setup
kubectl get roles,rolebindings -n demo-namespace
```

### 3. Map K8s Service Accounts to AWS IAM Roles (IRSA)

Configure IRSA to allow pods to assume AWS IAM roles:

```bash
# Create IAM role with Terraform
terraform apply -target=aws_iam_role.irsa_role

# Annotate service account
kubectl annotate serviceaccount demo-service-account \
  -n demo-namespace \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT-ID:role/demo-irsa-role
```

### 4. Include YAML Configurations

All necessary Kubernetes manifests are provided in the `k8s/` directory:
- `deployment.yaml` - Application deployment
- `service-account.yaml` - Service account with IRSA annotation
- `rbac.yaml` - RBAC roles and bindings

### 5. Include Terraform for IAM/IAM Role Binding

Terraform configurations for AWS resources are in the `terraform/` directory:
- `main.tf` - Main configuration
- `iam.tf` - IAM roles and policies
- `variables.tf` - Input variables
- `outputs.tf` - Output values

### 6. Demo Pod Assuming IAM Role

Test the IRSA configuration with a demo pod:

```bash
# Deploy test pod
kubectl apply -f k8s/test-pod.yaml

# Exec into pod and test AWS access
kubectl exec -it test-pod -n demo-namespace -- aws sts get-caller-identity
```

## Code Examples

### RBAC Configuration (k8s/rbac.yaml)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: demo-namespace
  name: demo-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-role-binding
  namespace: demo-namespace
subjects:
- kind: ServiceAccount
  name: demo-service-account
  namespace: demo-namespace
roleRef:
  kind: Role
  name: demo-role
  apiGroup: rbac.authorization.k8s.io
```

### Service Account with IRSA (k8s/service-account.yaml)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-service-account
  namespace: demo-namespace
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT-ID:role/demo-irsa-role
automountServiceAccountToken: true
```

### Application Deployment (k8s/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      serviceAccountName: demo-service-account
      containers:
      - name: demo-app
        image: amazon/aws-cli:latest
        command: ["sleep", "3600"]
        env:
        - name: AWS_REGION
          value: "us-west-2"
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Terraform IAM Configuration (terraform/iam.tf)

```hcl
# IRSA IAM Role
resource "aws_iam_role" "irsa_role" {
  name = "demo-irsa-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRoleWithWebIdentity"
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks.arn
        }
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:demo-namespace:demo-service-account"
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = {
    Name = "demo-irsa-role"
  }
}

# IAM Policy for S3 access
resource "aws_iam_policy" "s3_access" {
  name        = "demo-s3-access"
  description = "Policy for S3 access from EKS pods"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::demo-bucket/*",
          "arn:aws:s3:::demo-bucket"
        ]
      }
    ]
  })
}

# Attach policy to role
resource "aws_iam_role_policy_attachment" "irsa_s3_policy" {
  role       = aws_iam_role.irsa_role.name
  policy_arn = aws_iam_policy.s3_access.arn
}

# OIDC Provider for EKS
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["9e99a48a9960b14926bb7f3b02e22da2b0ab7280"]
  url             = data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer

  tags = {
    Name = "eks-irsa-oidc-provider"
  }
}
```

## Result (Demo)

After completing the setup, you will have:

1. **EKS Cluster**: Running with IRSA enabled
2. **Application Pods**: Deployed with proper service account annotations
3. **RBAC Rules**: Controlling access within the Kubernetes cluster
4. **IAM Integration**: Pods can assume AWS IAM roles securely
5. **AWS Resource Access**: Pods can access AWS services (S3, DynamoDB, etc.) without storing credentials

### Testing the Setup

```bash
# Test RBAC permissions
kubectl auth can-i get pods --as=system:serviceaccount:demo-namespace:demo-service-account -n demo-namespace

# Test IRSA functionality
kubectl exec -it demo-app-pod -n demo-namespace -- aws sts get-caller-identity
kubectl exec -it demo-app-pod -n demo-namespace -- aws s3 ls s3://demo-bucket/
```

### Expected Output

```json
{
    "UserId": "AROAXXXXXXXXXXXXXXXX:botocore-session-1234567890",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/demo-irsa-role/botocore-session-1234567890"
}
```

## Directory Structure

```
k8s-rbac-iam-integration/
├── README.md
├── k8s/
│   ├── deployment.yaml
│   ├── service-account.yaml
│   ├── rbac.yaml
│   └── test-pod.yaml
├── terraform/
│   ├── main.tf
│   ├── iam.tf
│   ├── variables.tf
│   └── outputs.tf
└── scripts/
    ├── setup.sh
    ├── test.sh
    └── cleanup.sh
```

## Security Best Practices

- Use least privilege principle for IAM policies
- Regularly rotate service account tokens
- Monitor and audit RBAC permissions
- Implement network policies for additional security
- Use AWS CloudTrail for logging API calls

## Troubleshooting

### Common Issues

1. **IRSA not working**: Check OIDC provider configuration
2. **RBAC denying access**: Verify role bindings and subjects
3. **AWS API calls failing**: Check IAM policy permissions
4. **Pod startup issues**: Verify service account exists and is properly annotated

### Debug Commands

```bash
# Check service account annotations
kubectl describe sa demo-service-account -n demo-namespace

# Check pod environment variables
kubectl exec demo-app-pod -n demo-namespace -- env | grep AWS

# Check RBAC permissions
kubectl auth can-i --list --as=system:serviceaccount:demo-namespace:demo-service-account
```

## Contact

**Author**: [Your Name]
**Email**: [your.email@example.com]
**GitHub**: [@ramji3030](https://github.com/ramji3030)
**LinkedIn**: [Your LinkedIn Profile]

For questions, issues, or contributions, please:
- Open an issue in this repository
- Submit a pull request with improvements
- Contact me directly via the channels above

---

**License**: MIT License - see [LICENSE](LICENSE) file for details

**Contributing**: Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.
