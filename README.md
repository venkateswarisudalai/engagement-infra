Markdown
# Engagement Infrastructure (GitOps)

This repository serves as the single source of truth for the cloud infrastructure and application deployments for the **Upvote App** and related engagement services. 

It utilizes a **GitOps** methodology powered by Argo CD, running on a managed AWS Elastic Kubernetes Service (EKS) cluster.

---

## 🏗️ Architecture Overview

* **Cloud Provider:** AWS (ap-south-1)
* **Kubernetes Engine:** Amazon EKS (v1.30)
* **Compute:** Managed Node Group (t3.medium instances)
* **Traffic Routing:** AWS Load Balancer Controller (Application Load Balancers)
* **Continuous Delivery:** Argo CD

---

## 🛠️ Prerequisites

Before interacting with this cluster or deploying infrastructure, ensure your local terminal is authenticated with AWS and has the following tools installed:

* `aws-cli` (Configured with appropriate IAM admin credentials)
* `eksctl` (For cluster provisioning)
* `kubectl` (For interacting with the cluster)
* `helm` (For installing Kubernetes controllers)

---

## 🚀 Environment Bootstrap Guide

If the cluster is entirely destroyed, follow these exact steps to rebuild the environment and restore the automated GitOps pipeline.

### 1. Provision the EKS Cluster
Run the following `eksctl` command to spin up the VPC, subnets, control plane, and worker nodes.

```bash
eksctl create cluster \
  --name=upvote-cluster \
  --region=ap-south-1 \
  --version=1.30 \
  --nodegroup-name=workers \
  --node-type=t3.medium \
  --nodes=3 \
  --nodes-min=2 \
  --nodes-max=4 \
  --managed \
  --with-oidc
2. Configure Load Balancer IAM Permissions
Create the required IAM service account to allow the cluster to manage AWS Application Load Balancers. Replace YOUR_AWS_ACCOUNT_ID with your actual 12-digit AWS Account ID.

Bash
curl -O [https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json)

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=upvote-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole-Upvote \
  --attach-policy-arn=arn:aws:iam::YOUR_AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
3. Install the AWS Load Balancer Controller
Deploy the traffic controller via Helm.

Bash
helm repo add eks [https://aws.github.io/eks-charts](https://aws.github.io/eks-charts)
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=upvote-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
4. Install Argo CD (The GitOps Engine)
Install Argo CD to handle automated deployments from this repository.

Bash
kubectl create namespace argocd
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml) --server-side
5. Connect the Repository
Apply the root application manifest to tell Argo CD to start tracking this repository and deploying the resources located in the k8s/ directory.

Bash
kubectl apply -f argo-application-root.yaml
📂 Repository Structure
k8s/: Contains all Kubernetes manifests (Deployments, Services, Ingress) for the applications.

argo-application-root.yaml: The master configuration file that links Argo CD to this specific repository.

README.md: Infrastructure documentation.
