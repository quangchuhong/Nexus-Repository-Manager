
# Nexus Repository Manager on Amazon EKS with S3 Backend Storage

This guide provides step-by-step instructions to deploy **Nexus Repository Manager** on **Amazon EKS** using **Amazon S3** as backend storage. The architecture ensures high availability, scalability, and cost efficiency.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Step-by-Step Deployment](#step-by-step-deployment)
4. [Cost Optimization](#cost-optimization)
5. [Monitoring and Backup](#monitoring-and-backup)
6. [Conclusion](#conclusion)

---

## Prerequisites

Before starting, ensure the following tools are installed and configured:
- **AWS CLI**: Configured with sufficient permissions.
- **kubectl**: Installed and configured to interact with EKS.
- **eksctl**: Installed for managing EKS clusters.

```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

---

## Architecture Overview
The architecture consists of the following components:
- **Amazon EKS Cluster**: Hosts Nexus Pods for high availability.
- **Persistent Volume (PV)**: Uses EBS or EFS for local storage of Nexus configuration and logs.
- **Amazon S3**: Acts as backend storage for artifacts and metadata.
- **AWS IAM & Networking**: Ensures secure access and network isolation.

```bash
+-----------------------------------------------------------+
|                     Amazon EKS Cluster                    |
|                                                           |
|  +-------------------+      +-------------------+         |
|  | Nexus Pod         |      | Nexus Pod         |         |
|  | (Deployment)      |      | (Deployment)      |         |
|  |                   |      |                   |         |
|  | +---------------+ |      | +---------------+ |         |
|  | | Nexus         | |      | | Nexus         | |         |
|  | | Container     | |      | | Container     | |         |
|  | +---------------+ |      | +---------------+ |         |
|  |                   |      |                   |         |
|  | +---------------+ |      | +---------------+ |         |
|  | | Persistent    | |      | | Persistent    | |         |
|  | | Volume (PV)   | |      | | Volume (PV)   | |         |
|  | +---------------+ |      | +---------------+ |         |
|  +-------------------+      +-------------------+         |
|                                                           |
|  +-----------------------------------------------------+  |
|  | Nexus Service (LoadBalancer)                        |  |
|  | Exposes Nexus UI and API on port 8081               |  |
|  +-----------------------------------------------------+  |
+-----------------------------------------------------------+

+-----------------------------------------------------------+
|                     Amazon S3                             |
|                                                           |
|  +-------------------+                                    |
|  | S3 Bucket         |                                    |
|  | (Blob Storage)    |                                    |
|  |                   |                                    |
|  | - Artifacts       |                                    |
|  | - Metadata        |                                    |
|  +-------------------+                                    |
+-----------------------------------------------------------+
```
---

## Step-by-Step Deployment
1. Create EKS Cluster
```
eksctl create cluster --name nexus-cluster --region <region> --node-type t3.medium --nodes 3
```

2. Deploy Nexus Repository Manager
```
1. Create a namespace for Nexus:
kubectl create namespace nexus
```
```
2. Create a Persistent Volume Claim (PVC):
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
```
Apply the PVC:
kubectl apply -f nexus-pvc.yaml
```
```
3. Deploy Nexus:
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
```
kubectl apply -f nexus-deployment.yaml
```
4. Expose Nexus via a LoadBalancer:
```
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
```
kubectl apply -f nexus-service.yaml

```
---

## 3. Configure S3 as Backend Storage

1. Install the S3 Blob Store plugin in Nexus.
2. Configure S3 Blob Store in the Nexus UI:
- Go to Repository > Blob Stores > Create Blob Store.
- Select S3 Blob Store and provide S3 bucket details.
  
## 4. Cost Optimization
To optimize storage costs:
- Use S3 Intelligent-Tiering for automatic cost savings.
- Transition data to S3 Standard-IA or S3 Glacier using S3 Lifecycle Policies.
- Compress data before uploading to S3.
- Delete unnecessary data regularly.
---
## 5. Monitoring and Backup
- Use AWS CloudWatch to monitor EKS and S3 performance.
- Set up Prometheus and Grafana for Nexus monitoring.
- Use AWS Backup for automated S3 bucket backups.
