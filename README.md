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
  - 3 instances (triển khai trên 3 Availability Zone khác nhau)  
  - Sử dụng `PodAntiAffinity` để đảm bảo High Availability  
- **Resource Requests**:  
  - CPU: 4 cores (Ví dụ: EC2 instance type `m5.xlarge`)  
  - RAM: 16GB (Xử lý ~1,000 requests/second)  
- **Storage**:  
  - EFS: 50GB (lưu trữ cấu hình, logs, cache) với chế độ `ReadWriteMany`  
  - S3: 10TB+ (artifacts chính) + Lifecycle Policy (tự động chuyển sang Glacier sau 90 ngày)  

---

### b. Database (RDS PostgreSQL)  
- **Instance Type**:  
  - `db.m6g.4xlarge` (16 vCPU, 64GB RAM)  
  - Multi-AZ deployment (tự động failover trong vòng <60 giây)  
- **Storage**:  
  - 1TB Provisioned Storage  
  - IOPS: 10,000 (độ trễ <5ms)  

---

### c. Amazon S3  
- **Storage Class**:  
  - Standard (cho dữ liệu truy cập thường xuyên)  
  - Intelligent-Tiering (tối ưu chi phí tự động)  
- **Tính năng bảo mật**:  
  - Versioning + Cross-Region Replication  
  - Mã hóa dữ liệu bằng SSE-KMS  

---

### d. Networking  
- **Thiết kế VPC**:  
  - Private Subnets (3 AZ) cho Nexus Pods và RDS  
  - Public Subnets cho Application Load Balancer (ALB)  
- **Security Groups**:  
  - Chỉ mở port 80/443 từ ALB tới Nexus  
  - Giới hạn quyền truy cập RDS từ Security Group của EKS  
- **Best Practices**:  
  - Sử dụng VPC Endpoints cho S3 để giảm độ trễ  
  - Kích hoạt `enableDnsHostnames` và `enableDnsSupport` trong VPC  


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






