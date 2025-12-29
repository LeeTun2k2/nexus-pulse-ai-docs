
### Infrastructure Design (/docs/design/infrastructure-design.md)
File này tài liệu về deployment: Docker cho containerization, Compose cho local/dev, Kubernetes cho prod, và config S3.

```markdown
# Infrastructure Design

## 1. Overview
Infrastructure low-cost, cloud-agnostic (AWS/GCP), serverless where possible. Sử dụng Docker cho dev/prod consistency, Kubernetes cho scaling.

- **Environments**:
  - Dev: Local Docker Compose.
  - Prod: Kubernetes on EKS/GKE, với auto-scaling.

## 2. Docker Compose (for Dev/Local)
Sử dụng docker-compose.yml để run all services locally. Ví dụ config:

S3 Configuration (AWS or MinIO for local)
Bucket Setup: Create bucket ai-content-gen với public-read cho downloads (CDN optional).
Config in Code: Use AWS SDK in Go
Security: IAM role cho EC2/K8s pods; Lifecycle policy xóa old files sau 30 ngày để save cost.
Cost Optimization: Use S3 Intelligent-Tiering; Estimate: $0.023/GB/month cho 1000 users (assume 1GB/user/month).

Last Updated: December 28, 2025