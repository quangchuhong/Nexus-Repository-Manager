# Nexus Repository Manager on Amazon EKS with S3 Backend Storage, EFS share data

## HA Architecture Overview
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
## 2. Giải Thích Từng Thành Phần

### a. Nexus Pods trên EKS  
- **Replicas**:  
  ⭐ 3 instances (triển khai trên 3 AZ khác nhau)  
  ⭐ Sử dụng `PodAntiAffinity` để đảm bảo HA  
- **Resource Requests**:  
  ✅ CPU: 4 cores (VD: EC2 `m5.xlarge`)  
  ✅ RAM: 16GB (Xử lý ~1,000 RPS)  
- **Storage**:  
  🗂️ EFS: 50GB (lưu config, logs, cache) - `ReadWriteMany`  
  🗄️ S3: 10TB+ (artifacts) + Lifecycle Policy (Glacer after 90 days)  

---

### b. Database (RDS PostgreSQL)  
- **Instance Type**:  
  🚀 `db.m6g.4xlarge` (16 vCPU, 64GB RAM)  
  🚀 Multi-AZ (Auto-failover < 60s)  
- **Storage**:  
  💾 1TB Provisioned Storage  
  ⚡ IOPS: 10,000 (Độ trễ < 5ms)  

---

### c. Amazon S3  
- **Storage Class**:  
  📦 Standard (truy cập thường xuyên)  
  📦 Intelligent-Tiering (tối ưu cost)  
- **Tính năng**:  
  🔒 Versioning + Replication (Cross-Region)  
  🔐 SSE-KMS Encryption  

---

### d. Networking  
- **VPC Design**:  
  🌐 Private Subnets (3 AZ) → Nexus Pods + RDS  
  🌐 Public Subnets → ALB  
- **Security Groups**:  
  🔒 Chỉ mở port 80/443 từ ALB → Nexus  
  🔒 Restrict RDS access từ EKS Security Group  
- **Best Practices**:  
  ✅ VPC Endpoints cho S3 (giảm latency)  
  ✅ Enable DNS Hostnames & Support trong VPC  



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






