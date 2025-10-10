# Nexus Repository Manager on Amazon EKS with S3 Backend Storage, EFS share data

## Architecture Overview
```plaintext
[Users]
  │
  ▼ HTTPS/SSL
[Application Load Balancer (ALB)] → Health Check, SSL Termination
  │
  ▼
[Amazon EKS Cluster]
  ├── [Nexus Pods (3 Replicas)] → Deploy trên nhiều AZ
  │    ├── [Persistent Volume (EFS)] → Shared storage cho /nexus-data (config, cache)
  │    └── [IAM Role (IRSA)] → Truy cập S3
  │
  ▼
[Amazon S3] → Blob Storage cho Artifacts (HA sẵn có)
  │
  ▼
[Amazon RDS (PostgreSQL Multi-AZ)] → Metadata (users, permissions, repo config)
  │
  ▼
[Backup & Monitoring]
  ├── AWS Backup (EFS, RDS, S3)
  └── Prometheus/Grafana → Giám sát hiệu năng

```

---

## Step-by-Step Deployment

### 1. Create EKS Cluster
```bash
eksctl create cluster --name nexus-cluster --region <region> --node-type t3.medium --nodes 3
```

### 2. Deploy Nexus Repository Manager
1. Create a namespace for Nexus:
   ```bash
   kubectl create namespace nexus
   ```

2. Create a Persistent Volume Claim (PVC):
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nexus-pvc
     namespace: nexus
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 50Gi
   ```
   Apply the PVC:
   ```bash
   kubectl apply -f nexus-pvc.yaml
   ```

3. Deploy Nexus:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nexus
     namespace: nexus
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nexus
     template:
       metadata:
         labels:
           app: nexus
       spec:
         containers:
         - name: nexus
           image: sonatype/nexus3:latest
           ports:
           - containerPort: 8081
           volumeMounts:
           - name: nexus-storage
             mountPath: /nexus-data
         volumes:
         - name: nexus-storage
           persistentVolumeClaim:
             claimName: nexus-pvc
   ```
   Apply the Deployment:
   ```bash
   kubectl apply -f nexus-deployment.yaml
   ```

4. Expose Nexus via a LoadBalancer:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nexus-service
     namespace: nexus
   spec:
     selector:
       app: nexus
     ports:
       - protocol: TCP
         port: 8081
         targetPort: 8081
     type: LoadBalancer
   ```
   Apply the Service:
   ```bash
   kubectl apply -f nexus-service.yaml
   ```

### 3. Configure S3 as Backend Storage
1. Install the S3 Blob Store plugin in Nexus.
2. Configure S3 Blob Store in the Nexus UI:
   - Go to **Repository > Blob Stores > Create Blob Store**.
   - Select **S3 Blob Store** and provide S3 bucket details.

---

## Cost Optimization
To optimize storage costs:
- Use **S3 Intelligent-Tiering** for automatic cost savings.
- Transition data to **S3 Standard-IA** or **S3 Glacier** using S3 Lifecycle Policies.
- Compress data before uploading to S3.
- Delete unnecessary data regularly.

---

## Monitoring and Backup
- Use **AWS CloudWatch** to monitor EKS and S3 performance.
- Set up **Prometheus** and **Grafana** for Nexus monitoring.
- Use **AWS Backup** for automated S3 bucket backups.

---

## Conclusion
This guide provides a scalable and cost-effective solution for deploying Nexus Repository Manager on Amazon EKS with S3 backend storage. Follow the steps to ensure a robust and efficient artifact management system.

---

Here is the complete **Terraform code** to deploy the **Nexus Repository Manager** on **Amazon EKS** using **Amazon S3** as backend storage. You can copy and use it directly.

---

### **Folder Structure**
```
nexus-eks-s3/
├── main.tf
├── variables.tf
├── outputs.tf
├── eks/
│   ├── main.tf
│   ├── variables.tf
├── s3/
│   ├── main.tf
│   ├── variables.tf
├── iam/
│   ├── main.tf
│   ├── variables.tf
└── kubernetes/
    ├── nexus-deployment.yaml
    ├── nexus-service.yaml
    ├── nexus-pvc.yaml
```

---

### **File: main.tf**
```hcl
provider "aws" {
  region = var.aws_region
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  token                  = module.eks.cluster_token
}

module "eks" {
  source = "./eks"
}

module "s3" {
  source = "./s3"
}

module "iam" {
  source = "./iam"
}

resource "kubernetes_namespace" "nexus" {
  metadata {
    name = "nexus"
  }
}

resource "kubernetes_manifest" "nexus_pvc" {
  manifest = yamldecode(file("${path.module}/kubernetes/nexus-pvc.yaml"))
}

resource "kubernetes_manifest" "nexus_deployment" {
  manifest = yamldecode(file("${path.module}/kubernetes/nexus-deployment.yaml"))
}

resource "kubernetes_manifest" "nexus_service" {
  manifest = yamldecode(file("${path.module}/kubernetes/nexus-service.yaml"))
}
```

---

### **File: variables.tf**
```hcl
variable "aws_region" {
  default = "us-east-1"
}

variable "cluster_name" {
  default = "nexus-cluster"
}

variable "bucket_name" {
  default = "nexus-artifacts-bucket"
}
```

---

### **File: outputs.tf**
```hcl
output "eks_cluster_name" {
  value = module.eks.cluster_name
}

output "s3_bucket_name" {
  value = module.s3.bucket_name
}

output "eks_node_role_arn" {
  value = module.iam.eks_node_role_arn
}
```

---

### **Folder: eks/**

#### **File: eks/main.tf**
```hcl
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = var.cluster_name
  cluster_version = "1.27"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id

  worker_groups = [
    {
      instance_type = "t3.medium"
      asg_max_size  = 3
    }
  ]
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name   = "nexus-vpc"
  cidr   = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

#### **File: eks/variables.tf**
```hcl
variable "cluster_name" {
  type = string
}
```

---

### **Folder: s3/**

#### **File: s3/main.tf**
```hcl
resource "aws_s3_bucket" "nexus_bucket" {
  bucket = var.bucket_name
  acl    = "private"

  lifecycle_rule {
    enabled = true

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}

output "bucket_name" {
  value = aws_s3_bucket.nexus_bucket.bucket
}
```

#### **File: s3/variables.tf**
```hcl
variable "bucket_name" {
  type = string
}
```

---

### **Folder: iam/**

#### **File: iam/main.tf**
```hcl
resource "aws_iam_role" "eks_node_role" {
  name = "eks-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_node_s3_access" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

output "eks_node_role_arn" {
  value = aws_iam_role.eks_node_role.arn
}
```

#### **File: iam/variables.tf**
```hcl
# No variables needed for this module
```

---

### **Folder: kubernetes/**

#### **File: kubernetes/nexus-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nexus
  namespace: nexus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus
  template:
    metadata:
      labels:
        app: nexus
    spec:
      containers:
      - name: nexus
        image: sonatype/nexus3:latest
        ports:
        - containerPort: 8081
        volumeMounts:
        - name: nexus-storage
          mountPath: /nexus-data
      volumes:
      - name: nexus-storage
        persistentVolumeClaim:
          claimName: nexus-pvc
```

#### **File: kubernetes/nexus-service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nexus-service
  namespace: nexus
spec:
  selector:
    app: nexus
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
  type: LoadBalancer
```

#### **File: kubernetes/nexus-pvc.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus-pvc
  namespace: nexus
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

---

### **How to Use**

1. **Initialize Terraform**:
   ```bash
   terraform init
   ```

2. **Review the Deployment Plan**:
   ```bash
   terraform plan
   ```

3. **Deploy the System**:
   ```bash
   terraform apply
   ```

4. **Destroy the System** (when no longer needed):
   ```bash
   terraform destroy
   ```

---

### **Results**
- An EKS cluster will be created and configured to run Nexus Repository Manager.
- An S3 bucket will be created to store artifacts.
- Nexus will be deployed on EKS and exposed via a LoadBalancer.

---
