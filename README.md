# Complete DevSecOps CI/CD Pipeline with GitOps Deployment on AWS EKS

A production-ready Spring Boot banking application deployed on Amazon EKS with a complete DevSecOps CI/CD pipeline featuring Jenkins, ArgoCD, SonarQube, OWASP dependency scanning, Trivy container scanning, and comprehensive monitoring with Prometheus and Grafana.

## Project Overview

This project demonstrates an enterprise-grade DevSecOps implementation for a Spring Boot banking application with:

- **Application**: Spring Boot 3.3.3 banking application with MySQL database
- **Container Orchestration**: Amazon EKS (Elastic Kubernetes Service)
- **CI/CD Pipeline**: Jenkins with 12-stage automated pipeline
- **GitOps**: ArgoCD for automated Kubernetes deployments with auto-sync and self-heal
- **Code Quality**: SonarQube for static code analysis
- **Security Scanning**:
  - OWASP Dependency Check for vulnerability scanning
  - Trivy for container image scanning
- **Infrastructure as Code**: Terraform for AWS infrastructure provisioning
- **Package Management**: Helm charts for Kubernetes deployments
- **Monitoring**: Prometheus + Grafana for comprehensive observability
- **Auto-Scaling**: Horizontal Pod Autoscaler (HPA) for dynamic scaling

## Architecture

![DevSecOps Architecture](images/arc.png)

### Technology Stack

| Category | Technology |
|----------|-----------|
| **Application** | Spring Boot 3.3.3, Java 17, Maven |
| **Database** | MySQL 8.0 |
| **Container** | Docker |
| **Container Registry** | DockerHub |
| **Orchestration** | Kubernetes (Amazon EKS) |
| **CI/CD** | Jenkins |
| **GitOps** | ArgoCD |
| **Code Quality** | SonarQube |
| **Security Scanning** | OWASP Dependency-Check, Trivy |
| **Package Management** | Helm 3 |
| **Monitoring** | Prometheus, Grafana |
| **Infrastructure** | Terraform, AWS (EKS, EC2, EBS) |
| **Metrics** | Kubernetes Metrics Server, HPA |

## Infrastructure Components

### AWS Resources
- **Region**: us-west-1 (N. California)
- **EKS Cluster**: bankapp-cluster-v2
- **Node Group**: 2 x t2.medium instances
- **EC2 Master**: Jenkins server (t2.large)
- **EBS CSI Driver**: For persistent storage
- **VPC**: Default VPC configuration
- **LoadBalancers**:
  - ArgoCD UI
  - Grafana Dashboard
  - BankApp Service

### Kubernetes Resources
- **Namespace**: bankapp-namespace
- **Deployments**:
  - BankApp (2 replicas with auto-scaling)
  - MySQL (1 replica with persistent storage)
- **Services**:
  - BankApp: LoadBalancer (external access)
  - MySQL: ClusterIP (internal only)
- **ConfigMaps**: Application configuration
- **Secrets**: Database credentials
- **PersistentVolumeClaim**: 10Gi EBS volume for MySQL
- **HPA**: Auto-scaling from 2 to 5 replicas (40% CPU threshold)
- **Ingress**: NGINX ingress for routing (optional)

### Monitoring Stack
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **Node Exporter**: System metrics
- **Kube State Metrics**: Kubernetes object metrics

## CI/CD Pipeline

### Jenkins Pipeline Stages

1. **Git Checkout**: Clone repository from GitHub
2. **Compile**: Maven clean compile
3. **Test**: Run unit tests
4. **Build Application**: Maven package (creates JAR)
5. **SonarQube Analysis**: Code quality and security analysis
6. **Quality Gate**: Validate code quality standards
7. **OWASP Dependency Check**: Scan dependencies for vulnerabilities
8. **Build Docker Image**: Multi-stage Docker build
9. **Trivy Image Scan**: Container security vulnerability scanning
10. **Push to DockerHub**: Push image with version tag and latest
11. **Update Kubernetes Manifest**: Update deployment YAML with new image tag
12. **Commit & Push Changes**: Push updated manifest to trigger GitOps

### Pipeline Features
- Automated versioning using Jenkins build number
- Code quality analysis with SonarQube
- Security scanning with OWASP Dependency-Check
- Container vulnerability scanning with Trivy
- Docker multi-stage builds for optimized image size
- Automated manifest updates for GitOps workflow
- Automatic cleanup and docker logout
- Quality gate validation
- Comprehensive error handling

## Prerequisites

### Tools Required
- AWS CLI (configured with appropriate credentials)
- kubectl (v1.28+)
- eksctl (v0.167+)
- Terraform (v1.0+)
- Docker (v20.10+)
- Helm (v3.0+)
- Git

### AWS Permissions
- EKS cluster creation and management
- EC2 instance management
- EBS volume creation
- IAM role and policy management
- VPC networking
- LoadBalancer creation

### Accounts Needed
- AWS Account with appropriate permissions
- DockerHub account (for pushing images)
- GitHub account (for repository)

## Complete Setup Instructions

### 1. Infrastructure Setup

#### Create EC2 Instance (via Terraform)
```bash
cd terraform
terraform init
terraform plan
terraform apply -auto-approve
```

#### SSH into EC2 Instance
```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

#### Install Required Tools on EC2
```bash
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Docker
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Trivy
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy

# Maven
sudo apt-get install -y maven
```

### 2. Create EKS Cluster

```bash
eksctl create cluster \
  --name bankapp-cluster-v2 \
  --region us-west-1 \
  --nodegroup-name bankapp-nodes \
  --node-type t2.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --managed
```

Wait approximately 15-20 minutes for cluster creation.

#### Enable IAM OIDC Provider
```bash
eksctl utils associate-iam-oidc-provider \
  --region=us-west-1 \
  --cluster=bankapp-cluster-v2 \
  --approve
```

#### Install EBS CSI Driver
```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster bankapp-cluster-v2 \
  --region us-west-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole

eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster bankapp-cluster-v2 \
  --region us-west-1 \
  --service-account-role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

### 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Expose ArgoCD server
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

#### Create ArgoCD Application
```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/ArsalanAnwer0/Springboot-BankingApp.git'
    targetRevision: main
    path: kubernetes
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: bankapp-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### 4. Install Jenkins

```bash
# Install Java
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre openjdk-17-jdk

# Install Jenkins
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins at `http://<EC2_PUBLIC_IP>:8080`

#### Install Jenkins Plugins
- Maven Integration
- Docker Pipeline
- Docker
- Git
- SonarQube Scanner
- OWASP Dependency-Check

#### Configure Jenkins Tools

1. **JDK** (Manage Jenkins → Tools → JDK installations):
   - Name: JDK17
   - JAVA_HOME: /usr/lib/jvm/java-17-openjdk-amd64
   - Uncheck "Install automatically"

2. **Maven** (Manage Jenkins → Tools → Maven installations):
   - Name: Maven
   - MAVEN_HOME: /usr/share/maven
   - Uncheck "Install automatically"

3. **Dependency-Check** (Manage Jenkins → Tools → Dependency-Check installations):
   - Name: DP-Check
   - Check "Install automatically"
   - Install from github.com

#### Configure Docker Permissions
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 5. Install and Configure SonarQube

```bash
# Run SonarQube in Docker
docker run -d --name sonarqube \
  -p 9000:9000 \
  sonarqube:latest

# Wait for SonarQube to start
sleep 60
```

Access SonarQube at `http://<EC2_PUBLIC_IP>:9000`
- Default credentials: admin / admin
- Change password on first login

#### Configure SonarQube in Jenkins

1. **Install SonarQube Scanner Plugin** in Jenkins
2. **Configure SonarQube Server** (Manage Jenkins → System → SonarQube servers):
   - Name: sonarqube-server
   - Server URL: http://<EC2_PRIVATE_IP>:9000
   - Server authentication token: (Generate in SonarQube → My Account → Security → Tokens)

3. **Add SonarQube Token to Jenkins Credentials**:
   - Kind: Secret text
   - Secret: <your-sonarqube-token>
   - ID: sonarqube

4. **Configure SonarQube Scanner** (Manage Jenkins → Tools):
   - Name: SonarQube Scanner
   - Install automatically from Maven Central

### 6. Configure Jenkins Credentials

1. **GitHub** (ID: github):
   - Kind: Username with password
   - Username: Your GitHub username
   - Password: GitHub Personal Access Token
   - ID: github

2. **DockerHub** (ID: dockerhub):
   - Kind: Username with password
   - Username: Your DockerHub username
   - Password: DockerHub password/token
   - ID: dockerhub

### 7. Create Jenkins Pipeline Job

1. Open Jenkins → New Item
2. Enter name: bankapp-pipeline
3. Select "Pipeline"
4. Under "Pipeline" section:
   - Definition: "Pipeline script from SCM"
   - SCM: Git
   - Repository URL: https://github.com/YOUR_USERNAME/Springboot-BankingApp.git
   - Credentials: Select your GitHub credentials
   - Branch: */main
   - Script Path: Jenkinsfile
5. Click "Save"

### 8. Install Prometheus and Grafana

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus + Grafana stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# Expose Grafana via LoadBalancer
kubectl patch svc prometheus-grafana -n monitoring -p '{"spec":{"type":"LoadBalancer"}}'

# Get Grafana admin password
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

Access Grafana at the LoadBalancer URL with username `admin`.

### 9. Configure SonarQube Webhook (Optional)

To enable Quality Gate feedback in Jenkins:

1. Go to SonarQube → Administration → Configuration → Webhooks
2. Click "Create"
3. Name: Jenkins
4. URL: http://<EC2_PRIVATE_IP>:8080/sonarqube-webhook/
5. Click "Create"

### 10. Expose Application via LoadBalancer

```bash
# Change service type to LoadBalancer
kubectl patch svc bankapp-service -n bankapp-namespace -p '{"spec":{"type":"LoadBalancer"}}'

# Get application URL
kubectl get svc bankapp-service -n bankapp-namespace
```

## Security Features

### Code Quality & Security
- **SonarQube Analysis**: Static code analysis for bugs, vulnerabilities, and code smells
- **Quality Gates**: Enforced quality standards before deployment
- **OWASP Dependency Check**: Scans all dependencies for known vulnerabilities
  - Checks against National Vulnerability Database (NVD)
  - Reports Critical, High, Medium, and Low severity issues

### Container Security
- **Trivy Scanning**: Comprehensive container vulnerability scanning
- **Severity Filtering**: Focuses on HIGH and CRITICAL vulnerabilities
- **Automated Scanning**: Every build is scanned before deployment
- **Multi-stage Builds**: Reduces attack surface by excluding build dependencies

### Kubernetes Security
- **Secrets Management**: Database credentials stored in Kubernetes Secrets
- **Resource Limits**: CPU and memory limits defined for all containers
- **Health Checks**: Readiness and liveness probes configured
- **Network Policies**: ClusterIP services for internal communication
- **RBAC**: Role-Based Access Control for service accounts

## Monitoring and Observability

### Prometheus Metrics
- **Cluster Metrics**: CPU, memory, disk, network usage
- **Node Metrics**: Individual node performance
- **Pod Metrics**: Application-specific metrics
- **Custom Metrics**: Application performance indicators

### Grafana Dashboards
Pre-configured dashboards available:
- **Kubernetes / Compute Resources / Cluster**: Overall cluster health
- **Kubernetes / Compute Resources / Namespace (Pods)**: Pod-level metrics
- **Kubernetes / Compute Resources / Pod**: Individual pod performance
- **Node Exporter / Nodes**: System-level metrics

### Horizontal Pod Autoscaler (HPA)
```yaml
minReplicas: 2
maxReplicas: 5
targetCPUUtilizationPercentage: 40
```

HPA automatically scales the application between 2-5 replicas based on CPU utilization.

## GitOps Workflow

### How It Works

1. Developer pushes code to GitHub
2. Jenkins pipeline automatically triggers
3. Pipeline executes all stages (build, test, scan, build image)
4. Pipeline updates Kubernetes manifest with new image tag
5. Pipeline commits and pushes updated manifest to GitHub
6. ArgoCD detects the change (within 3 minutes)
7. ArgoCD automatically syncs and deploys to EKS
8. Kubernetes performs rolling update
9. Old pods are gracefully terminated
10. New pods become ready and start serving traffic

### ArgoCD Features
- **Auto-Sync**: Automatically deploys changes from Git
- **Self-Heal**: Automatically fixes drift from desired state
- **Rollback**: Easy rollback to previous versions
- **Health Status**: Real-time application health monitoring
- **Sync Waves**: Ordered resource deployment

## Project Structure

```
.
├── kubernetes/                     # Kubernetes manifests
│   ├── bankapp-deployment.yml      # BankApp deployment
│   ├── bankapp-service.yml         # BankApp service
│   ├── bankapp-hpa.yml             # Horizontal Pod Autoscaler
│   ├── bankapp-ingress.yml         # Ingress configuration
│   ├── bankapp-namespace.yml       # Namespace definition
│   ├── configmap.yml               # Application configuration
│   ├── mysql-secret.yml            # MySQL credentials
│   ├── mysql-deployment.yml        # MySQL deployment
│   ├── mysql-service.yml           # MySQL service
│   └── persistent-volume-claim.yml # MySQL PVC
├── helm/                           # Helm charts
│   └── bankapp/                    # BankApp Helm chart
│       ├── Chart.yaml              # Chart metadata
│       ├── values.yaml             # Default values
│       └── templates/              # Kubernetes templates
├── terraform/                      # Terraform IaC
│   ├── main.tf                     # EC2 instance configuration
│   └── variables.tf                # Variable definitions
├── src/                            # Spring Boot application
│   ├── main/java/                  # Application code
│   └── main/resources/             # Application resources
├── Dockerfile                      # Multi-stage Docker build
├── Jenkinsfile                     # Jenkins pipeline definition
├── pom.xml                         # Maven project configuration
└── README.md                       # This file
```

## Troubleshooting

### Pipeline Fails at Docker Build
```bash
# Verify Jenkins user is in docker group
groups jenkins

# If not, add jenkins to docker group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### ArgoCD Not Syncing
```bash
# Check application status
kubectl get application bankapp -n argocd

# Force sync
kubectl patch app bankapp -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Refresh application
kubectl -n argocd patch app bankapp --type merge -p '{"spec":{"source":{"targetRevision":"main"}}}'
```

### SonarQube Analysis Fails
```bash
# Check SonarQube is running
docker ps | grep sonarqube

# View SonarQube logs
docker logs sonarqube

# Restart SonarQube
docker restart sonarqube
```

### Pods Not Starting
```bash
# Check pod status
kubectl get pods -n bankapp-namespace

# View pod logs
kubectl logs <pod-name> -n bankapp-namespace

# Describe pod for events
kubectl describe pod <pod-name> -n bankapp-namespace

# Check PVC status
kubectl get pvc -n bankapp-namespace
```

### MySQL Connection Issues
```bash
# Verify MySQL is running
kubectl get pods -l app=mysql -n bankapp-namespace

# Check MySQL logs
kubectl logs -l app=mysql -n bankapp-namespace

# Test connectivity from app pod
kubectl exec -it <bankapp-pod> -n bankapp-namespace -- nc -zv mysql-svc 3306
```

### Grafana Not Accessible
```bash
# Check Grafana service
kubectl get svc prometheus-grafana -n monitoring

# Get LoadBalancer URL
kubectl get svc prometheus-grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Check Grafana pods
kubectl get pods -l app.kubernetes.io/name=grafana -n monitoring
```

## Access URLs

After setup, you can access:

- **Jenkins**: http://<EC2_PUBLIC_IP>:8080
- **SonarQube**: http://<EC2_PUBLIC_IP>:9000
- **ArgoCD**: http://<ARGOCD_LOADBALANCER>
- **Grafana**: http://<GRAFANA_LOADBALANCER>
- **BankApp**: http://<BANKAPP_LOADBALANCER>:8080

## Security Note

This is a demonstration project for educational purposes. The credentials and secrets included are for development and testing only. For production deployments:

- Use AWS Secrets Manager or HashiCorp Vault for secret management
- Never commit credentials or private keys to version control
- Implement proper RBAC and network policies
- Enable pod security policies and admission controllers
- Use private container registries
- Enable audit logging
- Implement SSL/TLS for all services
- Use managed databases (RDS) instead of self-hosted MySQL
- Enable encryption at rest and in transit
- Implement proper backup and disaster recovery

## Key Achievements

- Complete DevSecOps pipeline with 12 automated stages
- GitOps deployment with ArgoCD auto-sync and self-heal
- Code quality analysis with SonarQube
- Security scanning with OWASP and Trivy
- Infrastructure as Code with Terraform
- Container orchestration with Kubernetes on AWS EKS
- Comprehensive monitoring with Prometheus and Grafana
- Auto-scaling with Horizontal Pod Autoscaler
- Helm charts for package management
- Production-ready architecture with high availability

## License

This project is for educational and demonstration purposes.

## Author

**Arsalan Anwer**
- GitHub: [@ArsalanAnwer0](https://github.com/ArsalanAnwer0)
- LinkedIn: [Arsalan Anwer](https://www.linkedin.com/in/arsalan-anwer/)

## Acknowledgments

- Spring Boot Team for the excellent framework
- Kubernetes community for comprehensive documentation
- Jenkins community for CI/CD best practices
- ArgoCD team for GitOps implementation
- SonarQube for code quality tools
- OWASP for security scanning tools
