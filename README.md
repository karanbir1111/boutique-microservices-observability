# Cloud-Native Boutique Microservices: Cloud Infrastructure, GitOps CI/CD & Observability

A production-ready, cloud-native e-commerce application deployed on Amazon Elastic Kubernetes Service (EKS) using modern DevOps practices. This project demonstrates end-to-end infrastructure provisioning, continuous integration, GitOps continuous delivery, and full-stack observability.

---

## 🏗️ System Architecture & Workflow

```
                                    ┌─────────────┐
                                    │   Frontend  │
                                    │ (Port 3000) │
                                    └──────┬──────┘
                                           │
                                    ┌──────▼──────┐
                                    │   Gateway   │
                                    │ (Port 3001) │
                                    └──────┬──────┘
                                           │
            ┌──────────────────────────────┼──────────────────────────────┐
            │                              │                              │
     ┌──────▼──────┐              ┌────────▼──────┐             ┌────────▼──────┐
     │    Auth     │              │Product Service│             │  User Service │
     │ (Port 3002) │              │  (Port 3003)  │             │  (Port 3006)  │
     └──────┬──────┘              └───────┬───────┘             └───────┬───────┘
            │                             │
     ┌──────▼──────┐              ┌───────▼───────┐
     │Order Service│              │    Orders     │
     │ (Port 3004) │              │  (Port 3005)  │
     └──────┬──────┘              └───────────────┘
            │
     ┌──────▼──────┐
     │  PostgreSQL │ (auth_db, products_db, orders_db, users_db)
     │ (Port 5432) │
     └─────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           Observability & Monitoring                        │
│  [Prometheus] ◄──── [Grafana Dashboard]                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

The system comprises 7 independent microservices built with **Node.js/Express** and **React**, communicating over REST APIs and backed by a multi-database **PostgreSQL** instance.

---

## 🛠️ Technology Stack & Core Pillars

### 1. Infrastructure as Code (IaC) — Terraform
All cloud resources are provisioned modularly via Terraform under the `projects/Infrastructure/` directory:
*   **Networking:** A custom VPC featuring 3 public subnets distributed across availability zones (`us-east-1a/b/c`), equipped with Route Tables and Internet Gateways. Subnets are automatically tagged for AWS Elastic Load Balancer (ELB) auto-discovery.
*   **Compute:** An Amazon EKS Cluster (`v1.34`) utilizing a Managed Node Group (`m7i-flex.large` instances) for high pod capacity and compute performance.
*   **Security:** Fully configured OpenID Connect (OIDC) provider mapping Kubernetes Service Accounts to AWS IAM Roles (**IRSA**), enforcing the principle of least privilege.
*   **Registries:** Elastic Container Registry (ECR) repositories for each microservice with automated scan-on-push policies.

### 2. CI/CD Pipeline — GitHub Actions
An automated build-and-deploy pipeline defined in `.github/workflows/ci.yml`:
*   **Parallel Builds:** Matrix builds trigger concurrently to build Docker images from service subdirectories on code updates.
*   **ECR Pushing:** Securely authenticates and pushes tagged images (using Git commit SHAs) to AWS ECR.
*   **Manifest Sync:** Automatically patches image tags inside GitOps Kubernetes manifests and commits changes back to the repository.

### 3. GitOps Continuous Delivery — ArgoCD
Declares and enforces the target state of the EKS cluster:
*   Bootstrap configurations install ArgoCD via Helm.
*   The `gitops/argo-cd.yml` resource syncs the EKS cluster with the declarative Kubernetes manifests in `gitops/k8s/`.
*   Features automatic self-healing and pruning of out-of-sync resources.

### 4. Full-Stack Observability — Prometheus & Grafana
*   **Metrics Scraping:** Installs `kube-prometheus-stack` via Helm. A custom `ServiceMonitor` targets the Gateway microservice to pull Prometheus metrics `/metrics`.
*   **Visual Dashboards:** Configures a Grafana dashboard through a Kubernetes ConfigMap containing key panels (request rate, p95/p99 response latency, pod CPU/memory utilization, and JS event loop lag).
*   **Logs Forwarding:** Uses AWS Fluent Bit daemonsets to stream container logs to AWS CloudWatch Logs.

---

## 🔧 DevOps Case Studies: Troubleshooting Log

The deployment codebase includes several solved real-world troubleshooting scenarios. Below are the key engineering solutions implemented:

### Case 1: Node Group Pod Capacity Exhaustion
*   **Problem:** Early testing used `t3.medium` instances. Once ArgoCD and the Prometheus Monitoring stack were added, newly scheduled microservices pods remained stuck in `Pending` with the error `Too many pods; no new claims to deallocate`.
*   **Root Cause:** Under the AWS VPC CNI, pod IP allocation depends on the EC2 instance size. A `t3.medium` is restricted to 17 pods. With 7 microservices (running multiple replicas), system controllers, and monitoring agents, the cluster exhausted the pod limit.
*   **Solution:** Upgraded the node group configuration to `m7i-flex.large` (2 vCPU, 8 GiB RAM), which supports up to 29 pods per node.

### Case 2: AWS EBS CSI Storage Driver Permissions (EKS 1.32+)
*   **Problem:** PostgreSQL pods failed to start, remaining in `ContainerCreating` with volume mount failures.
*   **Root Cause:** Kubernetes version 1.32+ in AWS EKS decouples volume controller actions. IAM permissions attached to EKS nodes can no longer manage EBS attachments directly; instead, the EBS CSI Driver requires IAM Roles for Service Accounts (IRSA).
*   **Solution:** Integrated an OIDC Provider, configured a trust policy matching the `system:serviceaccount:kube-system:ebs-csi-controller-sa` service account, and linked it to the `AmazonEBSCSIDriverPolicy` IAM role. The EKS CSI driver add-on was then deployed using this service account ARN.

### Case 3: PostgreSQL Database Initialization Conflict
*   **Problem:** The database statefulset completed starting, but backend microservices failed to load data, reporting that DB tables/schemas did not exist.
*   **Root Cause:** AWS EBS volumes default to creating a `lost+found` directory on creation. PostgreSQL checks the mount path; since the directory was not empty, Postgres skipped its default initialization/seeding scripts.
*   **Solution:** Abstracted the SQL schema seed scripts into a standalone Kubernetes Job (`restore-job.yml`). The deployment procedure was updated to run this Job as a sequencer once the PostgreSQL pod enters the `Running` state.


---

## 🚀 How to Deploy

### 1. Provision Infrastructure
Configure your AWS CLI (`aws configure`) and initialize Terraform:
```bash
cd projects/Infrastructure
terraform init
terraform plan
terraform apply --auto-approve
```

### 2. Configure kubectl
Update your local kubeconfig context to connect to your EKS cluster:
```bash
aws eks update-kubeconfig --region us-east-1 --name eks-cluster
```

### 3. Deploy Kubernetes Manifests via GitOps
Apply the root Kustomize manifest to bootstrap the application:
```bash
kubectl apply -k gitops/
```

Wait for the PostgreSQL pod to be ready:
```bash
kubectl get pods -n boutique -l app=boutique-postgres
```

Seed the database:
```bash
kubectl apply -f gitops/k8s/database/restore-job.yml
```


