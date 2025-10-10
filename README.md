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
- a. Nexus Pods trên EKS
  Replicas: 3 instances (trải đều 3 AZ).
  Resource Requests:
  CPU: 4 cores (e.g., m5.xlarge).
  RAM: 16GB → Đảm bảo xử lý ~1,000 RPS (requests/second).
  Storage:
  EFS: 50GB (config, logs, cache) → ReadWriteMany.
  S3: 10TB+ (artifacts) với lifecycle policy (chuyển sang Glacier sau 90 ngày).

- b. Database (RDS PostgreSQL)
  Instance Type: db.m6g.4xlarge (16 vCPU, 64GB RAM).
  Storage: 1TB (Provisioned IOPS: 10,000) → Độ trễ < 5ms.
  Multi-AZ: Auto-failover < 60s.

- c. Amazon S3
  Storage Class: Standard (truy cập thường xuyên) + Intelligent-Tiering.
  Versioning & Replication: Bật để disaster recovery.


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






