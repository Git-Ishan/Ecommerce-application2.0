# End-to-End DevOps Project on AWS with CI/CD Pipeline

This project demonstrates a comprehensive end-to-end CI/CD infrastructure for a containerized e-commerce application built on microservices.  
It uses **Terraform** for AWS infrastructure provisioning, **Jenkins** for continuous integration, and **FluxCD** for GitOps-based continuous deployment on a Kubernetes (EKS) cluster.

---

## Project Architecture

```
Developer Push
     │
     ▼
  GitHub Repo
     │
     ├──► Jenkins (CI)
     │         ├── Build & Unit Tests
     │         ├── Code Quality (golangci-lint)
     │         ├── Docker Build & Push → Docker Hub
     │         └── Update Kubernetes Manifests
     │
     └──► FluxCD watches repo (CD)
               └── Auto-syncs manifests → EKS Cluster
```

---

## Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| Infrastructure | Terraform | Provision VPC, EKS, S3, DynamoDB |
| Local Testing | Docker Compose | Run all services locally |
| Container Orchestration | Kubernetes (EKS) | Deploy and manage workloads |
| CI Pipeline | Jenkins | Build, test, lint, push Docker images |
| CD / GitOps | FluxCD | Auto-deploy from Git to EKS |
| DNS | AWS Route53 | Custom domain routing |
| Ingress | AWS ALB (via Helm) | Expose app publicly |
| Monitoring | Prometheus + Grafana | Metrics and dashboards |
| Tracing | Jaeger + OpenTelemetry | Distributed tracing |

---

## Project Structure

```
.
├── .github/                  # Repo configs
├── Terraform/                # IaC for VPC, EKS, S3, DynamoDB
│   ├── backend/              # Remote state (S3 + DynamoDB)
│   └── modules/
│       ├── eks/
│       └── vpc/
├── jenkins/                  # Jenkinsfile for CI pipeline
├── flux/                     # FluxCD manifests (GitRepository, Kustomization)
├── kubernetes/               # K8s manifests for all services
├── src/                      # Microservice source code
├── docker-compose.yml        # Local multi-service setup
└── README.md
```

---

## 1. AWS Infrastructure (Terraform)

Terraform provisions all AWS resources:

- **S3 + DynamoDB** — Remote backend for Terraform state and state locking
- **VPC** — Public/private subnets, route tables, IGW, NAT gateway
- **EKS** — Managed Kubernetes cluster with auto-scaling node groups
- **Route53** — DNS for custom domain routing

### Deploy Infrastructure

```bash
# Step 1: Initialize and apply the backend first
cd Terraform/backend
terraform init
terraform apply

# Step 2: Provision VPC and EKS
cd ..
terraform init
terraform apply

# Destroy when done
terraform destroy
```

---

## 2. Run Locally with Docker Compose

Test the full application stack locally before deploying.

**Prerequisites:** Docker, `t2.large` EC2 (or equivalent local machine with 8+ GB RAM)

```bash
git clone https://github.com/<your-username>/ultimate-devops-project-demo.git
cd ultimate-devops-project-demo

docker compose up -d
```

Access at: `http://localhost:8080`

If you hit storage issues on EC2, expand the EBS volume to 30 GB and run:

```bash
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

---

## 3. Jenkins CI Pipeline

Jenkins replaces GitHub Actions for the CI pipeline. The `Jenkinsfile` defines these stages:

**Triggers:** On every push to `main` branch (via GitHub webhook)

**Pipeline Stages:**

```
Checkout → Build & Test → Code Quality → Docker Build & Push → Update K8s Manifest
```

| Stage | What it does |
|---|---|
| Build & Test | Sets up Go 1.22, builds `main.go`, runs unit tests |
| Code Quality | Runs `golangci-lint` |
| Docker Build & Push | Builds image, tags with `BUILD_NUMBER`, pushes to Docker Hub |
| Update Manifest | Replaces image tag in `kubernetes/productcatalog/deploy.yaml`, commits back |

### Jenkins Setup

1. Install Jenkins on your EC2 instance (or use a dedicated Jenkins server)
2. Install required plugins: Git, Docker Pipeline, Go, Credentials Binding
3. Add credentials in Jenkins:
   - `DOCKER_USERNAME` — Docker Hub username
   - `DOCKER_TOKEN` — Docker Hub access token
   - `GIT_CREDENTIALS` — GitHub token for manifest push
4. Create a Pipeline job pointing to your repo's `Jenkinsfile`
5. Configure a GitHub webhook: `http://<jenkins-url>/github-webhook/`

### Sample Jenkinsfile (Product Catalog Service)

```groovy
pipeline {
  agent any
  environment {
    DOCKER_IMAGE = "your-dockerhub-username/product-catalog"
    IMAGE_TAG    = "${BUILD_NUMBER}"
  }
  stages {
    stage('Build & Test') {
      steps {
        sh 'cd src/product-catalog && go build ./... && go test ./...'
      }
    }
    stage('Code Quality') {
      steps {
        sh 'golangci-lint run src/product-catalog/...'
      }
    }
    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
          sh '''
            docker build -t $DOCKER_IMAGE:$IMAGE_TAG src/product-catalog/
            echo $TOKEN | docker login -u $USER --password-stdin
            docker push $DOCKER_IMAGE:$IMAGE_TAG
          '''
        }
      }
    }
    stage('Update K8s Manifest') {
      steps {
        sh "sed -i 's|image: .*|image: $DOCKER_IMAGE:$IMAGE_TAG|' kubernetes/productcatalog/deploy.yaml"
        sh 'git add kubernetes/productcatalog/deploy.yaml && git commit -m "CI: update image to $IMAGE_TAG" && git push'
      }
    }
  }
}
```

---

## 4. Kubernetes on EKS

Connect your machine to the EKS cluster and deploy all services.

```bash
# Configure kubectl
aws eks --region <region> update-kubeconfig --name <cluster-name>

# Verify
kubectl get nodes

# Deploy all services
kubectl apply -f kubernetes/serviceaccount.yaml
kubectl apply -f kubernetes/complete-deploy.yaml

# Check status
kubectl get pods
kubectl get svc
kubectl get deployments
```

### Ingress Setup (AWS ALB)

```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve

# Download and create IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# Create IAM service account
eksctl create iamserviceaccount \
  --cluster=<cluster-name> --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install ALB controller via Helm
helm repo add eks https://aws.github.io/eks-charts && helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<vpc-id>

# Apply ingress
kubectl apply -f kubernetes/ingress.yaml
```

Then configure **Route53** to point your domain to the ALB DNS.

---

## 5. FluxCD (GitOps CD)

FluxCD replaces ArgoCD. It watches the Git repository and automatically applies any changes to the EKS cluster — same GitOps principle, different tooling.

### Install FluxCD

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux onto your cluster (links it to your GitHub repo)
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=<your-repo-name> \
  --branch=main \
  --path=./flux \
  --personal
```

### How it works

1. Jenkins updates the image tag in `kubernetes/productcatalog/deploy.yaml` and pushes to `main`
2. FluxCD detects the change (polls every 1 minute by default)
3. FluxCD automatically applies the updated manifest to the EKS cluster
4. New pods roll out with zero downtime

### Verify FluxCD

```bash
flux get sources git
flux get kustomizations
```

---

## Prerequisites

- AWS account with IAM permissions for EC2, EKS, VPC, S3, DynamoDB, Route53, IAM
- Tools installed: Terraform, AWS CLI, kubectl, Helm, Flux CLI, Docker, Git
- Jenkins server (EC2 or self-hosted)
- Docker Hub account

---

## Cleanup

```bash
# Remove K8s resources
kubectl delete -f kubernetes/

# Uninstall Flux
flux uninstall

# Destroy AWS infrastructure
cd Terraform && terraform destroy
cd backend && terraform destroy
```
