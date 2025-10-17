# Nexus Repository Manager & IQ Server

Bản so sánh chi tiết **Nexus Repository Pro**, **Nexus Repository OSS** và **Nexus IQ Server** giúp bạn lựa chọn công cụ phù hợp cho quy trình DevOps.

---

## 📊 So sánh tính năng

| Tính năng                     | Nexus Repository OSS          | Nexus Repository Pro          | Nexus IQ Server               |
|-------------------------------|--------------------------------|--------------------------------|--------------------------------|
| **Giấy phép**                 | Mã nguồn mở (Apache 2.0)      | Thương mại                    | Thương mại                    |
| **Mục đích chính**           | Quản lý artifact               | Quản lý artifact nâng cao + Tuân thủ | Tự động hóa bảo mật & tuân thủ |
| **High Availability (HA)**     | ❌                            | ✅ (Cụm Active-Active)         | ✅ (Tích hợp với Pro)          |
| **Quét lỗ hổng**             | Cơ bản (Sonatype OSS Index)    | Cơ bản                        | Nâng cao (CVE, Zero-day, SBOM)|
| **Quản lý rủi ro giấy phép** | ❌                            | ✅                            | ✅                            |
| **Tích hợp CI/CD**           | Hạn chế                       | Jenkins, GitLab, GitHub       | Tự động hóa (Áp dụng chính sách) |
| **Bảo mật container**        | ❌                            | ✅ (Quét Docker)               | ✅ (Kubernetes + Phát hiện thông tin nhạy cảm) |
| **Kiểm toán & Tuân thủ**     | ❌                            | ✅ (Báo cáo tùy chỉnh)         | ✅ (SOC 2, GDPR, ISO 27001)  |
| **Hỗ trợ**                   | Cộng đồng                     | 24/7 SLA                      | 24/7 SLA                      |

---

## 🚀 Trường hợp sử dụng

### Nexus Repository OSS
- **Phù hợp**: Nhóm nhỏ, dự án mã nguồn mở, hoặc môi trường giới hạn ngân sách.
- **Tính năng**:
  - Lưu trữ và quản lý artifact (Docker, npm, Maven, v.v.).
  - Quét lỗ hổng cơ bản qua OSS Index.

### Nexus Repository Pro
- **Phù hợp**: Doanh nghiệp cần khả năng mở rộng, tuân thủ và HA.
- **Tính năng**:
  - Tường lửa chặn thành phần rủi ro.
  - High Availability với cụm active-active.
  - Quét image Docker nâng cao.

### Nexus IQ Server
- **Phù hợp**: Nhóm DevSecOps ưu tiên bảo mật tự động.
- **Tính năng**:
  - Tạo SBOM (SPDX, CycloneDX).
  - Tự động áp dụng chính sách trong pipeline CI/CD.
  - Bảo mật container và Kubernetes.

---

## Nexus Repository Manager OSS on Amazon EKS with S3 Backend Storage

### HA Architecture Overview
```plaintext
[Users]
  │
  ▼ HTTPS/SSL
[Application Load Balancer (ALB)] → Health Check, SSL Termination
  │
  ▼
[Amazon EKS Cluster]
  ├── [Nexus Pod (Single Instance)] → Deploy trên một AZ
  │    ├── [Persistent Volume (EBS)] → Local storage cho /nexus-data (config, cache, embedded database)
  │    └── [IAM Role (IRSA)] → Truy cập S3
  │
  ▼
[Amazon S3] → Blob Storage cho Artifacts (HA sẵn có)
  │
  ▼
[Backup & Monitoring]
  ├── AWS Backup (EBS, S3)
  └── Prometheus/Grafana → Giám sát hiệu năng


```
### 2. Giải Thích Từng Thành Phần

#### a. Nexus Pods trên EKS

- **Tài nguyên**:  
  - CPU: 4 cores (EC2 instance type `m5.xlarge`).  
  - RAM: 16GB (xử lý ~1,000 requests/giây).  
- **Lưu trữ**:  
  - **S3**: 10TB+ (artifacts chính) + Lifecycle Policy (Glacier sau 90 ngày).  
---

#### b. Amazon S3

- **Storage Class**:  
  - Standard (truy cập thường xuyên).  
  - Intelligent-Tiering (tối ưu chi phí).  
- **Tính năng**:  
  - Versioning + Cross-Region Replication.  
  - Mã hóa SSE-KMS.  

---

#### c. Networking  

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

## 4. Phân Biệt Cache vs Blob Storage (S3)

| **Tiêu Chí**       | **Cache**                                  | **Blob Storage (S3)**                |
|--------------------|--------------------------------------------|---------------------------------------|
| **Loại Dữ Liệu**   | Dữ liệu tạm, có thể tái tạo                | Artifacts chính (không thể mất)      |
| **Vị Trí**        | EBS storage                                 | S3 (durable storage)                 |
| **Kích Thước**    | Nhỏ (GBs)                                   | Lớn (TBs/PBs)                        |
| **Quản Lý**       | Tự động xóa khi hết hạn                     | Versioning + Lifecycle Policy        |
| **Truy Cập**      | Thường xuyên, tốc độ cao                    | Truy cập theo nhu cầu                |

---

## 5. Best Practices Quản Lý Cache

1. **Giới Hạn Dung Lượng Cache**:  
   ```plaintext
   Administration → Repository → <Repo> → Storage → Blob Store Quota

