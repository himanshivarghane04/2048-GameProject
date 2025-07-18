# 🕹️ 2048 Game Deployment on AWS EKS with Ingress

Deploy the classic **2048 game** using **AWS EKS (Fargate)** with **Ingress** and **AWS Load Balancer Controller**.

![image](https://github.com/krunalbhandekar/2048-game-aws-kubernetes-ingress/blob/main/assets/2048_live.png)

---

## 📦 Prerequisites

Before starting, install the following tools:

- [kubectl](https://kubernetes.io/docs/tasks/tools/) – Kubernetes CLI
- [eksctl](https://github.com/eksctl-io/eksctl) – EKS Cluster Manager
- [aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) – AWS Command Line Tool

---

## ☁️ Step 1: Configure AWS CLI

```bash
aws configure
```

Provide your **AWS Access Key**, **Secret Key**, **region**, and **output format**.

---

## ☸️ Step 2: Create EKS Cluster with Fargate

```bash
eksctl create cluster --name <cluster-name> --region <region-name> --fargate
```

⏱️ Takes around 10–15 minutes.

**✅ Why Fargate?**

AWS Fargate provides serverless compute for containers, so you don't need to manage EC2 nodes. It simplifies cluster management and auto-scales based on workload needs.

It creates:

- ✅ A **dedicated VPC**
- ✅ **Subnets** (public and/or private)
- ✅ **Route Tables**
- ✅ **Internet Gateway** (IGW)
- ✅ **NAT Gateway** (optional, depends on config)
- ✅ **Security Groups**
- ✅ **Elastic IPs** (for NAT)
- ✅ **IAM Roles and Policies**
- ✅ **CloudFormation stack** (manages all of the above)

---

## 🔧 Step 3: Configure kubectl for EKS

```bash
aws eks update-kubeconfig --name <cluster-name> --region <region-name>
```

---

## 🧩 Step 4: Create Fargate Profile

```bash
eksctl create fargateprofile \
    --cluster <cluster-name> \
    --region <region-name> \
    --name alb-sample-app \
    --namespace game-2048
```

⏱️ Takes around 5 minutes.

Check the profile in the **AWS EKS Console** under **`Cluster → Compute → Fargate Profiles`**.

---

## 🚀 Step 5: Deploy 2048 Game + Service + Ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Verify:

```bash
kubectl get pods -n game-2048
```

Expected output:

| NAME                            | READY | STATUS  | RESTARTS | AGE |
| ------------------------------- | ----- | ------- | -------- | --- |
| deployment-2048-bdbddc878-7zdgg | 1/1   | Running | 0        | 66s |
| deployment-2048-bdbddc878-b7gtt | 1/1   | Running | 0        | 66s |
| deployment-2048-bdbddc878-kmjzc | 1/1   | Running | 0        | 66s |
| deployment-2048-bdbddc878-tkk7z | 1/1   | Running | 0        | 66s |
| deployment-2048-bdbddc878-v95rp | 1/1   | Running | 0        | 66s |

```bash
kubectl get svc -n game-2048
```

Expected output:

| NAME         | TYPE     | CLUSTER-IP     | EXTERNAL-IP | PORT(S)      | AGE  |
| ------------ | -------- | -------------- | ----------- | ------------ | ---- |
| service-2048 | NodePort | 10.100.166.209 | <none>      | 80:30258/TCP | 3m9s |

```bash
kubectl get ingress -n game-2048
```

Expected output:

| NAME         | CLASS | HOSTS | ADDRESS | PORTS | AGE   |
| ------------ | ----- | ----- | ------- | ----- | ----- |
| ingress-2048 | alb   | \*    |         | 80    | 3m36s |

ℹ️ **Note:** Ingress will be created, but without an ALB address unless the **AWS Load Balancer Controller** is installed.

---

## 🔐 Step 6: Configure IAM OIDC Provider

**✅ Why IAM OIDC?**

This allows EKS to securely assume IAM roles for Kubernetes service accounts.

```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

---

## 🛡️ Step 7: Set Up ALB Add-On

**Download IAM Policy**

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

**Create IAM Policy**

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

**Create IAM Role & Service Account**

```bash
eksctl create iamserviceaccount \
  --cluster=<cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

## 📦 Step 8: Install AWS Load Balancer Controller (via Helm)

**Add Helm Repo**

```bash
helm repo add eks https://aws.github.io/eks-charts
```

**Update Helm Repo**

```bash
helm repo update eks
```

**Install Controller**

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region-name> \
  --set vpcId=<vpc-id>
```

🔍 Find VPC ID from AWS Console → EKS → Cluster → Networking

---

## ✅ Step 9: Verify Deployment

**Check Controller Deployment**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected output:

| NAME                         | READY | UP-TO-DATE | AVAILABLE | AGE |
| ---------------------------- | ----- | ---------- | --------- | --- |
| aws-load-balancer-controller | 2/2   | 2          | 2         | 93s |

**Check Running Pods**

```bash
kubectl get pods -n kube-system
```

Expected output:

| NAME                                          | READY | STATUS  | RESTARTS | AGE   |
| --------------------------------------------- | ----- | ------- | -------- | ----- |
| aws-load-balancer-controller-5db9cb9959-7lskn | 1/1   | Running | 0        | 3m23s |
| aws-load-balancer-controller-5db9cb9959-tpkbz | 1/1   | Running | 0        | 3m23s |
| coredns-8448d8f9b5-gt6tn                      | 1/1   | Running | 0        | 60m   |
| coredns-8448d8f9b5-pzsx6                      | 1/1   | Running | 0        | 60m   |
| metrics-server-68cd648d99-ppz67               | 0/1   | Pending | 0        | 62m   |
| metrics-server-68cd648d99-tjfpd               | 0/1   | Pending | 0        | 62m   |

- Metrics server might remain pending if not configured for Fargate.

---

## 🌐 Step 10: Verify Ingress and Access the Game

```bash
kubectl get ingress -n game-2048
```

Expected output:

| NAME         | CLASS | HOSTS | ADDRESS                                       | PORTS | AGE  |
| ------------ | ----- | ----- | --------------------------------------------- | ----- | ---- |
| ingress-2048 | alb   | \*    | k8s-game2048-ingress2-xxxxx.elb.amazonaws.com | 80    | 106m |

🟢 Check the ALB DNS in browser once it Active:

Open `http://<alb-dns>` to play the 2048 game.

---

## 🧹 Step-by-Step: Delete All Resources

### ✅ 1. Delete Kubernetes Resources (Ingress, Service, Deployments)

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

### ✅ 2. Delete Fargate Profile

```bash
eksctl delete fargateprofile \
  --cluster <cluster-name> \
  --name alb-sample-app \
  --region <region-name>
```

### ✅ 3. Delete Load Balancer Controller

If installed with Helm:

```bash
helm uninstall aws-load-balancer-controller -n kube-system
```

Then, delete the service account:

```bash
eksctl delete iamserviceaccount \
  --name aws-load-balancer-controller \
  --namespace kube-system \
  --cluster <cluster-name> \
  --region <region-name>
```

### ✅ 4. Delete IAM Policy

```bash
aws iam delete-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy
```

Replace `<your-aws-account-id>` accordingly.

### ✅ 5. Delete EKS Cluster

This will delete the cluster and all associated Fargate profiles automatically:

```bash
eksctl delete cluster --name <cluster-name> --region <region-name>
```

⏱️ Takes a few minutes.

### ✅ 6. Delete ALB (If Leftover)

In rare cases, if the ALB created by the ingress is still active:

- Go to the **AWS Console → EC2 → Load Balancers**
- Manually find and delete the ALB associated with your cluster

### ✅ 7. Clean Up Remaining Resources (VPC, Subnets, etc.)

- Go to the **AWS CloudFormation Console →** https://console.aws.amazon.com/cloudformation/
- Look for a stack named:

```bash
eksctl-<cluster-name>-vpc
```

- Select it → Click **Delete**

⏳ This will automatically delete:

    - The VPC
    - Subnets
    - Internet Gateway
    - NAT Gateway
    - Route Tables
    - Elastic IPs
    - Security Groups

### 🗑️ Optional Cleanup

**🧾 Remove kubeconfig entry (locally)**

If you want to remove the cluster from your local `kubectl` config:

```bash
kubectl config delete-cluster <cluster-name>
kubectl config delete-context <cluster-name>
```

---

## ✅ Project Summary

- Deployed 2048 game on AWS EKS using **Fargate**.
- Configured **Ingress** and **AWS Load Balancer Controller**.
- Used **Helm**, **IAM roles**, and **OIDC provider** for secure deployments.

---

### 👨‍💻 Author

Maintained by **[Himanshi Varghane] (https://www.linkedin.com/in/himanshi-varghane-230203262/)**

---
