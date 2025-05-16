# 🚀 Amazon EKS Deployment with CI/CD (GitHub Actions + ECR + Kubernetes)

This project demonstrates how to set up a complete CI/CD pipeline to deploy a containerized application to **Amazon EKS**, using **GitHub Actions**, **Amazon ECR**, and **kubectl**.

### Inspired by:

* [Deploy a sample application on Linux](https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html)
* [Route application and HTTP traffic with Application Load Balancers](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)

---

## 🛠️ Project Overview

We build a Dockerized application (e.g., a simple Node.js or Flask app), push the image to **Amazon ECR**, and deploy it to an **EKS** cluster using **GitHub Actions**.

### 📦 Tools Used:

* **Amazon EKS** (Elastic Kubernetes Service)
* **Amazon ECR** (Elastic Container Registry)
* **GitHub Actions** (CI/CD Pipeline)
* **kubectl + eksctl**
* **Helm**

---

## ⚙️ Infrastructure Setup

### 1. **Install Required Tools**

Make sure you have the following installed locally:

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

![Tool Installation](https://github.com/user-attachments/assets/5714fee2-6d24-4fe5-8140-e1f76a1cd4b2)

---

### 2. **Create EKS Cluster (Fargate-based)**

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

![EKS Cluster](https://github.com/user-attachments/assets/9a542add-4180-4b5c-9274-a1bc5db52458)

---

### 3. **Configure kubectl to Access the Cluster**

```bash
aws eks --region us-east-1 update-kubeconfig --name demo-cluster
```

![kubectl Config](https://github.com/user-attachments/assets/fcf063b2-04b7-498f-aead-aaef8f515151)

---

## 🧱 Set Up ALB Ingress Controller

### 4. **Configure IAM OIDC Provider**

```bash
export cluster_name=demo-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If no OIDC provider is listed, run:

```bash
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

---

### 5. **Create IAM Policy for ALB Controller**

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

---

### 6. **Create IAM Role and Service Account**

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

### 7. **Install AWS Load Balancer Controller**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```

Verify:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 🎮 Deploy the Sample 2048 App

### 8. **Create Fargate Profile for Namespace**

```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

---

### 9. **Apply Kubernetes Manifests (Deployment + Service + Ingress)**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

![2048 App Deployed](https://github.com/user-attachments/assets/68ce305e-fcdc-44d6-b7de-3cace101e2d1)

---

## ✅ Verify the Deployment

Use the following to check if the LoadBalancer has been provisioned:

```bash
kubectl get ingress -n game-2048
```

Copy the **ADDRESS** and access the app in your browser.

---
