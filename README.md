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
  - 3 instances triển khai trên 3 Availability Zone (AZ).  
  - Sử dụng `PodAntiAffinity` để đảm bảo High Availability (HA).  
- **Tài nguyên**:  
  - CPU: 4 cores (EC2 instance type `m5.xlarge`).  
  - RAM: 16GB (xử lý ~1,000 requests/giây).  
- **Lưu trữ**:  
  - **EFS**: 50GB (cấu hình, logs, cache) - Chế độ `ReadWriteMany`.  
  - **S3**: 10TB+ (artifacts chính) + Lifecycle Policy (Glacier sau 90 ngày).  

---

### b. Database (RDS PostgreSQL)  
- **Loại instance**:  
  - `db.m6g.4xlarge` (16 vCPU, 64GB RAM).  
  - Multi-AZ deployment (failover <60 giây).  
- **Lưu trữ**:  
  - 1TB Provisioned Storage.  
  - IOPS: 10,000 (độ trễ <5ms).  

---

### c. Amazon S3  
- **Storage Class**:  
  - Standard (truy cập thường xuyên).  
  - Intelligent-Tiering (tối ưu chi phí).  
- **Tính năng**:  
  - Versioning + Cross-Region Replication.  
  - Mã hóa SSE-KMS.  

---

### d. Networking  
- **Thiết kế VPC**:  
  - Private Subnets (3 AZ) cho Nexus Pods và RDS.  
  - Public Subnets cho ALB.  
- **Security Groups**:  
  - Mở port 80/443 từ ALB → Nexus.  
  - Giới hạn quyền truy cập RDS từ EKS Security Group.  
- **Best Practices**:  
  - VPC Endpoints cho S3.  
  - Bật `enableDnsHostnames` và `enableDnsSupport`.  





