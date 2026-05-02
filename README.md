# AWS EKS Live Deployment

This project provisions an Amazon EKS cluster and deploys a sample web application behind an AWS Application Load Balancer (ALB) using the AWS Load Balancer Controller.

The repository is based on:

- Amazon EKS quickstart: https://docs.aws.amazon.com/eks/latest/userguide/quickstart.html
- AWS Load Balancer Controller installation guide: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/

## Project Overview

The deployment includes:

- An `eksctl` cluster configuration for an EKS cluster in `ap-south-1`
- A managed node group with 2 `t3.small` worker nodes
- A Kubernetes `Deployment` running the public `docker-2048` sample app
- A `ClusterIP` `Service` exposing the app inside the cluster
- A Kubernetes `Ingress` that provisions an internet-facing ALB with TLS
- An IAM policy for the AWS Load Balancer Controller

Current app flow:

`Internet -> ALB -> Ingress -> ClusterIP Service -> Deployment Pods`

## Repository Structure

- [`config.yaml`](./config.yaml) - `eksctl` cluster definition
- [`iam_policy.json`](./iam_policy.json) - IAM policy used by the AWS Load Balancer Controller
- [`deploy.yaml`](./deploy.yaml) - application deployment
- [`svc.yaml`](./svc.yaml) - internal Kubernetes service
- [`ingress.yaml`](./ingress.yaml) - ALB ingress and TLS configuration

## Architecture Notes

This repo follows the EKS quickstart idea of deploying a web application on EKS, but it uses a more explicit setup:

- Cluster creation is managed with `eksctl`
- Traffic is exposed through the AWS Load Balancer Controller instead of EKS Auto Mode
- TLS is terminated at the ALB using an ACM certificate
- The application is reachable through the custom host `app.sujitramdhakal.com.np`

## Prerequisites

Before deployment, make sure you have:

- An AWS account with permissions for EKS, EC2, IAM, VPC, and ELB
- AWS CLI configured
- `kubectl` installed
- `eksctl` installed
- `helm` installed
- A registered domain name
- An ACM certificate in the same AWS region as the ALB

## Deployment Steps

### 1. Create the EKS cluster

```bash
eksctl create cluster -f config.yaml
```

This creates:

- EKS cluster name: `live-app`
- Region: `ap-south-1`
- Managed node group with 2 nodes

### 2. Associate the IAM OIDC provider

The AWS Load Balancer Controller documentation recommends IAM Roles for Service Accounts (IRSA) for EKS.

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster live-app \
  --approve
```

### 3. Create the IAM policy for the AWS Load Balancer Controller

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Save the returned policy ARN.

### 4. Create the controller service account with IRSA

Replace `<AWS_ACCOUNT_ID>` with your AWS account ID.

```bash
eksctl create iamserviceaccount \
  --cluster=live-app \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region ap-south-1 \
  --approve
```

### 5. Install the AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=live-app \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 6. Deploy the application

```bash
kubectl apply -f deploy.yaml
kubectl apply -f svc.yaml
kubectl apply -f ingress.yaml
```

### 7. Verify the deployment

```bash
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get ingress
```

You can also inspect the ALB that was created in the AWS console under EC2 -> Load Balancers.

## Important Configuration in This Repo

### Cluster config

The cluster definition currently uses:

- Kubernetes version `1.34`
- Public cluster endpoint access enabled
- Public worker nodes (`privateNetworking: false`)
- Single NAT gateway
- OIDC disabled in the file by default

### Ingress config

The ingress is configured with:

- `alb.ingress.kubernetes.io/scheme: internet-facing`
- `alb.ingress.kubernetes.io/target-type: ip`
- HTTPS listener on port `443`
- HTTP to HTTPS redirect
- A region-specific ACM certificate ARN

## Cleanup

To remove the Kubernetes resources:

```bash
kubectl delete -f ingress.yaml
kubectl delete -f svc.yaml
kubectl delete -f deploy.yaml
```

To remove the EKS cluster:

```bash
eksctl delete cluster --name live-app --region ap-south-1
```

## References

- Amazon EKS quickstart: https://docs.aws.amazon.com/eks/latest/userguide/quickstart.html
- AWS Load Balancer Controller installation guide: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/
- Amazon EKS Kubernetes version lifecycle: https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
