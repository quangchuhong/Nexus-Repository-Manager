# Nexus Repository Manager & IQ Server Comparison

A comprehensive comparison of **Sonatype Nexus Repository Pro**, **Nexus Repository OSS**, and **Nexus IQ Server** to help you choose the right tool for your DevOps workflow.

---

## 📊 Feature Comparison

| Feature                        | Nexus Repository OSS          | Nexus Repository Pro          | Nexus IQ Server               |
|--------------------------------|--------------------------------|--------------------------------|--------------------------------|
| **License**                    | Open Source (Apache 2.0)      | Commercial                    | Commercial                    |
| **Primary Use Case**           | Artifact Management           | Advanced Artifact Management + Compliance | Security & Compliance Automation |
| **High Availability (HA)**     | ❌                            | ✅ (Active-Active Clustering) | ✅ (Integrated with Pro)       |
| **Vulnerability Scanning**     | Basic (Sonatype OSS Index)    | Basic                         | Advanced (CVE, Zero-day, SBOM)|
| **License Risk Management**    | ❌                            | ✅                            | ✅                            |
| **CI/CD Integration**          | Limited                       | Jenkins, GitLab, GitHub      | Full Automation (Policy Enforcement) |
| **Container Security**         | ❌                            | ✅ (Docker Scanning)          | ✅ (Kubernetes + Secrets Detection) |
| **Audit & Compliance**         | ❌                            | ✅ (Custom Reports)           | ✅ (SOC 2, GDPR, ISO 27001)  |
| **Support**                    | Community                     | 24/7 SLA                      | 24/7 SLA                      |

---

## 🚀 Use Cases

### Nexus Repository OSS
- **Ideal for**: Small teams, open-source projects, or budget-limited environments.
- **Features**:
  - Store and manage artifacts (Docker, npm, Maven, etc.).
  - Basic vulnerability scanning via OSS Index.

### Nexus Repository Pro
- **Ideal for**: Enterprises needing scalability, compliance, and HA.
- **Features**:
  - Firewall for blocking risky components.
  - High Availability with active-active clustering.
  - Advanced Docker image scanning.

### Nexus IQ Server
- **Ideal for**: DevSecOps teams prioritizing automated security.
- **Features**:
  - SBOM generation (SPDX, CycloneDX).
  - Real-time policy enforcement in CI/CD pipelines.
  - Container and Kubernetes security scanning.

---

## 🛠️ Getting Started

### Nexus Repository OSS
```yaml
# docker-compose.yml
version: "3"
services:
  nexus:
    image: sonatype/nexus3:latest
    ports:
      - "8081:8081"
    volumes:
      - nexus-data:/nexus-data
volumes:
  nexus-data:




## Nexus Repository Manager on Amazon EKS with S3 Backend Storage, EFS share data

### HA Architecture Overview
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
### 2. Giải Thích Từng Thành Phần

#### a. Nexus Pods trên EKS

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

#### b. Database (RDS PostgreSQL)

- **Loại instance**:  
  - `db.m6g.4xlarge` (16 vCPU, 64GB RAM).  
  - Multi-AZ deployment (failover <60 giây).  
- **Lưu trữ**:  
  - 1TB Provisioned Storage.  
  - IOPS: 10,000 (độ trễ <5ms).  
  
---

#### c. Amazon S3

- **Storage Class**:  
  - Standard (truy cập thường xuyên).  
  - Intelligent-Tiering (tối ưu chi phí).  
- **Tính năng**:  
  - Versioning + Cross-Region Replication.  
  - Mã hóa SSE-KMS.  

---

#### d. Networking  

- **Thiết kế VPC**:  
  - Private Subnets (3 AZ) cho Nexus Pods và RDS.  
  - Public Subnets cho ALB.  
- **Security Groups**:  
  - Mở port 80/443 từ ALB → Nexus.  
  - Giới hạn quyền truy cập RDS từ EKS Security Group.  
- **Best Practices**:  
  - VPC Endpoints cho S3.  
  - Bật `enableDnsHostnames` và `enableDnsSupport`.  


### 3. Cache trong Nexus


| **Loại Cache**             | **Mục Đích**                                      | **Vị Trí Lưu Trữ**                     |
|----------------------------|--------------------------------------------------|-----------------------------------------|
| **Metadata Cache**         | Lưu thông tin version, dependencies của artifacts | `/nexus-data/cache`                     |
| **Proxy Repository Cache** | Lưu artifacts tải từ remote repositories (Maven Central, npmjs) | `/nexus-data/blobs/<repo>/content` |
| **Index Cache**            | Tăng tốc tìm kiẩm artifacts                       | `/nexus-data/index`                     |
| **Database Cache**         | Tối ưu truy vấn database (nếu dùng embedded DB)   | `/nexus-data/db`                        |


---

## 4. Tại Sao Cần Lưu Cache Trên EFS?

- **High Availability (HA)**:  
  - Nhiều Pods truy cập cùng một cache → Đảm bảo consistency khi Pods restart/migrate.  
  - Tránh rebuild cache từ đầu khi Pod mới khởi động.  

- **Hiệu Suất**:  
  - Giảm số lần gọi API đến S3 → Tiết kiệm chi phí và giảm độ trễ.  
  - Parallel read/write từ nhiều Pods.  

- **Độ Bền Dữ Liệu**:  
  - Backup tự động qua AWS Backup → Phục hồi nhanh khi sự cố.  

---

## 5. Phân Biệt Cache vs Blob Storage (S3)

| **Tiêu Chí**       | **Cache**                                  | **Blob Storage (S3)**                |
|--------------------|--------------------------------------------|---------------------------------------|
| **Loại Dữ Liệu**   | Dữ liệu tạm, có thể tái tạo                | Artifacts chính (không thể mất)      |
| **Vị Trí**        | EFS (shared storage)                       | S3 (durable storage)                 |
| **Kích Thước**    | Nhỏ (GBs)                                 | Lớn (TBs/PBs)                        |
| **Quản Lý**       | Tự động xóa khi hết hạn                    | Versioning + Lifecycle Policy        |
| **Truy Cập**      | Thường xuyên, tốc độ cao                   | Truy cập theo nhu cầu                |

---

## 6. Best Practices Quản Lý Cache

1. **Giới Hạn Dung Lượng Cache**:  
   ```plaintext
   Administration → Repository → <Repo> → Storage → Blob Store Quota

