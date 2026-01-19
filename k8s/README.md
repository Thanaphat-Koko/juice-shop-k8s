# OWASP Juice Shop - AWS EKS Deployment Guide

This guide provides step-by-step instructions to deploy OWASP Juice Shop on AWS EKS (Elastic Kubernetes Service).

## Prerequisites

Before you begin, ensure you have the following installed and configured:

- **AWS CLI** (v2.x or later) - [Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- **kubectl** - [Installation Guide](https://kubernetes.io/docs/tasks/tools/)
- **eksctl** - [Installation Guide](https://eksctl.io/installation/)
- **Helm** (v3.x) - [Installation Guide](https://helm.sh/docs/intro/install/)
- AWS account with appropriate IAM permissions

## Step 1: Configure AWS CLI

Configure your AWS credentials:

```bash
aws configure
```

Enter your AWS Access Key ID, Secret Access Key, default region, and output format.

## Step 2: Create EKS Cluster

Create a new EKS cluster using eksctl:

```bash
eksctl create cluster \
  --name juice-shop-cluster \
  --region ap-southeast-1 \
  --version 1.28 \
  --nodegroup-name juice-shop-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed
```

> **Note**: Replace `ap-southeast-1` with your preferred AWS region.

This process takes approximately 15-20 minutes.

## Step 3: Update kubeconfig

After the cluster is created, update your kubeconfig:

```bash
aws eks update-kubeconfig --name juice-shop-cluster --region ap-southeast-1
```

Verify the connection:

```bash
kubectl get nodes
```

## Step 4: Install AWS Load Balancer Controller

The AWS Load Balancer Controller is required for the ALB Ingress to work.

### 4.1 Create IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster juice-shop-cluster \
  --region ap-southeast-1 \
  --approve
```

### 4.2 Create IAM Policy

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### 4.3 Create Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=juice-shop-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region ap-southeast-1 \
  --approve
```

> **Important**: Replace `<YOUR_AWS_ACCOUNT_ID>` with your actual AWS account ID.

### 4.4 Install the Controller using Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=juice-shop-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Verify the installation:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## Step 5: Deploy Juice Shop

### Option A: Using Kustomize (Recommended)

Deploy all resources at once:

```bash
kubectl apply -k k8s/
```

### Option B: Deploy Individually

Deploy resources one by one:

```bash
# Create namespace
kubectl apply -f k8s/namespace.yaml

# Deploy the application
kubectl apply -f k8s/deployment.yaml

# Create service
kubectl apply -f k8s/service.yaml

# Create ingress
kubectl apply -f k8s/ingress.yaml

# (Optional) Enable autoscaling
kubectl apply -f k8s/hpa.yaml

# (Optional) Apply network policy
kubectl apply -f k8s/networkpolicy.yaml
```

## Step 6: Verify Deployment

Check if all resources are running:

```bash
# Check pods
kubectl get pods -n juice-shop

# Check services
kubectl get svc -n juice-shop

# Check ingress
kubectl get ingress -n juice-shop
```

Wait for the pods to be in `Running` state and the ingress to have an ADDRESS assigned.

## Step 7: Access the Application

Get the ALB DNS name:

```bash
kubectl get ingress juice-shop -n juice-shop -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Open the URL in your browser. It may take a few minutes for the ALB to become fully available.

## Configuration Options

### Enable HTTPS

To enable HTTPS, you need an ACM certificate. Edit `ingress.yaml` and uncomment the SSL annotations:

```yaml
annotations:
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
  alb.ingress.kubernetes.io/ssl-redirect: "443"
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:REGION:ACCOUNT_ID:certificate/CERTIFICATE_ID
```

### Scale the Deployment

Manual scaling:

```bash
kubectl scale deployment juice-shop -n juice-shop --replicas=3
```

The HPA (Horizontal Pod Autoscaler) will automatically scale based on CPU/memory usage.

### Update Image Version

To update to a specific version:

```bash
kubectl set image deployment/juice-shop juice-shop=bkimminich/juice-shop:v15.0.0 -n juice-shop
```

## Monitoring

View pod logs:

```bash
kubectl logs -f deployment/juice-shop -n juice-shop
```

Describe resources for troubleshooting:

```bash
kubectl describe pod -n juice-shop
kubectl describe ingress juice-shop -n juice-shop
```

## Cleanup

To delete all Juice Shop resources:

```bash
kubectl delete -k k8s/
```

To delete the entire EKS cluster:

```bash
eksctl delete cluster --name juice-shop-cluster --region ap-southeast-1
```

## Troubleshooting

### Pods not starting
- Check pod events: `kubectl describe pod <pod-name> -n juice-shop`
- Check logs: `kubectl logs <pod-name> -n juice-shop`

### Ingress not getting an address
- Verify AWS Load Balancer Controller is running
- Check controller logs: `kubectl logs -n kube-system deployment/aws-load-balancer-controller`

### Cannot access the application
- Ensure security groups allow inbound traffic on port 80/443
- Verify the ALB is in "active" state in AWS Console

## File Structure

```
k8s/
├── namespace.yaml      # Dedicated namespace for Juice Shop
├── deployment.yaml     # Main deployment with 2 replicas
├── service.yaml        # ClusterIP service
├── ingress.yaml        # AWS ALB Ingress
├── hpa.yaml           # Horizontal Pod Autoscaler
├── networkpolicy.yaml  # Network security policies
├── kustomization.yaml  # Kustomize configuration
└── README.md          # This file
```

## Security Considerations

> ⚠️ **Warning**: OWASP Juice Shop is an intentionally vulnerable application designed for security training. **Do NOT expose it to the public internet in production environments.**

Recommended security measures:
1. Use VPN or IP whitelisting to restrict access
2. Deploy in a separate, isolated VPC
3. Enable AWS WAF for additional protection
4. Regularly rotate and audit access credentials
5. Monitor application and infrastructure logs

## Support

- [OWASP Juice Shop Documentation](https://pwning.owasp-juice.shop/)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [GitHub Issues](https://github.com/juice-shop/juice-shop/issues)
