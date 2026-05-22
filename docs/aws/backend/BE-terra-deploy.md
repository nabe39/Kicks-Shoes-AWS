# BE — Terraform Deployment Guide (Kicks-Shoes-AWS)

Tài liệu này hướng dẫn **deploy Backend (BE)** của repo `Kicks-Shoes-AWS` lên AWS bằng **Terraform** theo cấu trúc có sẵn trong repo.

> Phạm vi: môi trường `dev` (có thể áp dụng tương tự cho `production`).

---

## 0) Kiến trúc (tóm tắt)

Luồng request:

`User → (Route53/Domain) → (WAF optional) → ALB → ECS Fargate (Node.js BE) → (S3/DynamoDB/Redis/Secrets Manager/CloudWatch)`

Bộ Terraform chính ở repo:

- `infra/terraform/environments/dev/01-network`: VPC/Subnet/NAT/... (layer ổn định)
- `infra/terraform/environments/dev/02-app`: ALB/ECS/ECR integration/S3/DynamoDB/Redis/Lambda/WAF/Monitoring/... (layer app)

---

## 1) Những thứ cần chuẩn bị

### 1.1 Tooling (Windows/PowerShell)

- AWS CLI v2
- Docker Desktop
- Terraform >= 1.5

Kiểm tra nhanh:

```powershell
aws --version
docker version
terraform version
```

### 1.2 AWS credentials

Bạn cần có quyền đủ để tạo tài nguyên (DevOps) hoặc tối thiểu quyền deploy (Developer).

- Cách đơn giản nhất: cấu hình profile AWS CLI

```powershell
aws configure --profile default
# AWS Access Key ID
# AWS Secret Access Key
# Default region name: us-east-2
# Default output format: json
```

> IAM policy tham khảo cho backend developer: `docs/aws/backend/IAM-POLICY-FOR-BE-DEVELOPER.json` (chủ yếu phục vụ ECR/ECS/Logs/SecretsManager).

---

## 2) Hiểu nhanh về Terraform state/backend của repo

Trong `infra/terraform/environments/dev/01-network/versions.tf` và `02-app/versions.tf` có khai báo:

- `backend "s3" {}`

Backend **được cấu hình tại runtime** qua `-backend-config` (CI/CD). Khi chạy local, bạn cũng cần truyền `-backend-config` hoặc chấp nhận dùng local state (không khuyến nghị nếu team làm chung).

### 2.1 Remote state (đúng theo workflow của repo)

Workflow CI/CD của repo (xem `.github/workflows/deploy-dev-two-stack.yml`) đang dùng:

- S3 bucket state: `kicks-shoes-tf-state`
- Backend init: có `-backend-config="encrypt=true"`
- **Không cấu hình DynamoDB lock table** trong pipeline hiện tại

Vì vậy, khi chạy local bạn nên follow y hệt để tránh lệch hành vi.

Ví dụ backend-config (theo repo):

- cho `dev/01-network`
  - key: `dev/01-network/terraform.tfstate`
- cho `dev/02-app`
  - key: `dev/02-app/terraform.tfstate`

---

## 3) Deploy lần đầu (DevOps flow) — Terraform 2 stacks

### Contract (điều kiện thành công)

- Apply `01-network` thành công → tạo VPC/Subnets/NAT và tag đúng.
- Apply `02-app` thành công → tạo ALB + ECS service chạy được.
- `GET http://<ALB_DNS>/api/health` trả 200 (hoặc HTTPS nếu bạn bật custom domain).

### 3.1 Bước A — Deploy network stack (01-network)

1. Tạo file vars từ mẫu:

- copy `infra/terraform/environments/dev/01-network/terraform.tfvars.example` → `terraform.tfvars`

Ví dụ nội dung mẫu (đã có trong repo):

- `project_name = "kicks-shoes-dev"`
- `aws_region = "ap-southeast-1"`
- CIDR VPC/subnet

2. Init (remote backend):

```powershell
Set-Location infra/terraform/environments/dev/01-network

terraform init -reconfigure `
  -backend-config="bucket=kicks-shoes-tf-state-duc" `
  -backend-config="key=dev/01-network/terraform.tfstate" `
  -backend-config="region=us-east-2" `
  -backend-config="encrypt=true" `
  -upgrade
```

3. Plan + Apply:

```powershell
terraform validate
terraform plan -var-file="terraform.tfvars" -out tfplan
terraform apply tfplan
```

> Lưu ý quan trọng: `02-app` **không đọc remote_state** của `01-network`; nó tìm VPC/subnet qua **Tags** trong `infra/terraform/environments/dev/02-app/data.tf`.
> Vì vậy, sau khi apply `01-network`, hãy đảm bảo VPC/Subnet đã được tag đúng (theo code Terraform hiện tại).

### 3.2 Bước B — Deploy app stack (02-app)

1. Tạo `terraform.tfvars`

- copy `infra/terraform/environments/dev/02-app/terraform.tfvars.example` → `terraform.tfvars`

Các biến quan trọng:

- `aws_region = "ap-southeast-1"`
- `container_image = "<ACCOUNT>.dkr.ecr.ap-southeast-1.amazonaws.com/kicks-shoes-backend:dev-latest"`
- `app_config_secret_name = "kicks-shoes-dev/app-config"` (Secrets Manager)
- `enable_custom_domain = false` (bật lên nếu bạn có domain/Route53/ACM)

2. Đảm bảo Secret tồn tại trong Secrets Manager

`02-app/data.tf` sẽ đọc secret:

- `data.aws_secretsmanager_secret.app_config` dùng `var.app_config_secret_name`

Bạn cần tạo secret này trước (ít nhất chứa key/values app cần). Cách đặt keys tuỳ implementation backend.

3. Init backend + plan/apply:

```powershell
Set-Location infra/terraform/environments/dev/02-app

terraform init -reconfigure `
  -backend-config="bucket=kicks-shoes-tf-state" `
  -backend-config="key=dev/02-app/terraform.tfstate" `
  -backend-config="region=ap-southeast-1" `
  -backend-config="encrypt=true" `
  -upgrade

terraform validate
terraform plan -var-file="terraform.tfvars" -out tfplan
terraform apply tfplan
```

4. Lấy outputs để test nhanh:

```powershell
$AlbDns = terraform output -raw alb_dns_name
Write-Host "ALB: http://$AlbDns"
Invoke-WebRequest -Uri "http://$AlbDns/api/health" -UseBasicParsing
```

---

## 4) Deploy hằng ngày (Backend dev flow) — build/push image + rollout ECS

Với repo này, infra đã provision bằng Terraform rồi thì deployment thường ngày có 2 dạng:

### 4.1 Chỉ thay đổi code (không đổi infra)

1. Build + push image lên ECR.
2. Trigger ECS rollout (force new deployment).

Bạn có thể làm theo checklist trong:

- `docs/aws/backend/BE-DEPLOYMENT-CHECKLIST.md`

### 4.2 Code thay đổi và muốn Terraform “nhận” image mới

Nếu bạn đang dùng `container_image` trong Terraform để pin tag, hãy:

- update `container_image` trong `infra/terraform/environments/dev/02-app/terraform.tfvars`
- `terraform plan/apply`

---

## 5) Script có sẵn để demo end-to-end (khuyến nghị)

Repo có script PowerShell:

- `scripts/ecs-fargate-e2e.ps1`

Script này làm:

1. Check aws/docker/terraform
2. ensure ECR repo
3. login ECR
4. pull base image
5. build backend image từ `backend/Dockerfile`
6. push image
7. terraform init/plan/apply ở env demo (mặc định `infra/terraform/environments/demo`)
8. update-service + wait stable
9. in ALB URL + health URL

Ví dụ chạy:

```powershell
./scripts/ecs-fargate-e2e.ps1 `
  -AwsAccountId <ACCOUNT_ID> `
  -AwsProfile default `
  -AwsRegion ap-southeast-1 `
  -EcrRepository kicks-shoes-backend `
  -BaseTag latest `
  -NamePrefix kicks-fargate-demo
```

---

## 6) (Tuỳ chọn) Deploy Lambda Bedrock chat bằng Terraform

Nếu bạn cần phần W6 “Bedrock chat” (DynamoDB stream → Lambda gọi Bedrock KB):

1. Build lambda zip:

```powershell
Set-Location backend\lambda\bedrock-chat
.\build.ps1
```

2. Copy zip vào placeholder Terraform:

```powershell
Set-Location <REPO_ROOT>
Copy-Item backend\lambda\bedrock-chat.zip infra\terraform\lambda-placeholder.zip -Force
```

3. Apply `dev/02-app` lại:

```powershell
Set-Location infra/terraform/environments/dev/02-app
terraform plan -var-file="terraform.tfvars" -out tfplan
terraform apply tfplan
```

Chi tiết: `docs/aws/backend/LAMBDA-BEDROCK-DEPLOYMENT.md`.

---

## 7) Kiểm tra sau deploy & monitoring

### 7.1 ECS service stable

```powershell
aws ecs describe-services `
  --cluster (terraform output -raw ecs_cluster_name) `
  --services (terraform output -raw ecs_service_name) `
  --region ap-southeast-1 `
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}'
```

### 7.2 Tail logs

Log group thường gặp (tuỳ cấu hình task definition):

- `/ecs/kicks-backend`

```powershell
aws logs tail /ecs/kicks-backend --follow --region ap-southeast-1
```

### 7.3 Validate autoscaling (CPU target)

Trong vars mẫu của `02-app`:

- `autoscaling_cpu_target = 60`

Bạn có thể tạo load test nhanh theo:

- `docs/aws/backend/ecs-fargate-demo-checklist.md`

---

## 8) Troubleshooting nhanh

### 8.1 `terraform init` lỗi backend

- Kiểm tra S3 bucket state tồn tại và bạn có quyền.
- Kiểm tra DynamoDB lock table tồn tại và bạn có quyền.
- Nếu bucket policy chặn, DevOps cần cập nhật.

### 8.2 `02-app` không tìm thấy VPC/subnet

`infra/terraform/environments/dev/02-app/data.tf` locate VPC/subnets theo tags:

- VPC: `tag:Name` match `"*-vpc"`
- Subnet: `tag:Tier` = `public` / `private` / `firewall`

Nếu tags không đúng thì Terraform sẽ không tìm ra → plan/apply fail.

### 8.3 ECS task chạy nhưng health check fail

- Confirm backend expose đúng port `3000`.
- Check route `/api/health` tồn tại.
- Check security group inbound từ ALB → ECS target.
- Tail logs CloudWatch.

---

## 9) Cleanup / Teardown để tiết kiệm chi phí

Mục này dành cho lúc bạn **demo/test xong** và muốn xoá toàn bộ resource để tránh phát sinh chi phí.

> Cảnh báo:
>
> - `terraform destroy` sẽ xoá **hạ tầng** (ALB/ECS/NAT/VPC/Redis/DynamoDB/S3 uploads/...) và có thể xoá cả dữ liệu.
> - S3 bucket chứa Terraform state `kicks-shoes-tf-state` **KHÔNG nên xoá** nếu team còn dùng/hoặc còn muốn dùng lại state.
> - Secret `kicks-shoes-dev/app-config` thường được tạo thủ công (Terraform chỉ đọc), huỷ hay không tuỳ bạn.

### 9.1 Xoá App stack trước (02-app)

Xoá stack này trước để Terraform gỡ những resource phụ thuộc VPC (ECS service, ALB, endpoints, redis, dynamodb, uploads bucket...).

```powershell
Set-Location infra/terraform/environments/dev/02-app

terraform init -reconfigure `
  -backend-config="bucket=kicks-shoes-tf-state" `
  -backend-config="key=dev/02-app/terraform.tfstate" `
  -backend-config="region=ap-southeast-1" `
  -backend-config="encrypt=true" `
  -upgrade

terraform destroy -var-file="terraform.tfvars" -auto-approve
```

### 9.2 Xoá Network stack sau (01-network)

Stack network có NAT Gateway/VPC/Subnets/FlowLogs, đây là phần thường tốn chi phí (NAT GW) → nên destroy sau khi app stack đã gỡ xong.

```powershell
Set-Location infra/terraform/environments/dev/01-network

terraform init -reconfigure `
  -backend-config="bucket=kicks-shoes-tf-state" `
  -backend-config="key=dev/01-network/terraform.tfstate" `
  -backend-config="region=ap-southeast-1" `
  -backend-config="encrypt=true" `
  -upgrade

terraform destroy -var-file="terraform.tfvars" -auto-approve
```

### 9.3 Dọn ECR images (đỡ tốn storage, tránh rác)

Terraform stack **không tạo ECR repo** trong `dev/02-app` (ECR repo thường do CI/script tạo). Vì vậy bạn nên tự dọn image/tag đã push.

Ví dụ xoá tag `dev-latest` và một tag cụ thể (điền lại đúng region/repo):

```powershell
$Repo = "kicks-shoes-backend"
$Region = "ap-southeast-1"

# Xoá theo tag (lặp lại cho các tag bạn đã push)
aws ecr batch-delete-image --repository-name $Repo --region $Region --image-ids imageTag=dev-latest
aws ecr batch-delete-image --repository-name $Repo --region $Region --image-ids imageTag=<YOUR_TAG>
```

Nếu repo chỉ dùng cho demo và bạn muốn xoá luôn repository (cẩn thận: sẽ xoá toàn bộ image trong repo):

```powershell
aws ecr delete-repository --repository-name kicks-shoes-backend --region ap-southeast-1 --force
```

### 9.4 (Tuỳ chọn) Xoá CloudWatch Log Groups còn sót

Nhiều log group được tạo bởi Terraform hoặc ECS/Lambda. Nếu bạn destroy xong mà vẫn còn log group (do retention/hoặc tạo ngoài state), có thể xoá thủ công.

Ví dụ hay gặp:

- `/ecs/kicks-backend`
- `/vpc/kicks-shoes-dev/flow-logs`
- `/aws/lambda/<lambda-name>`

```powershell
aws logs delete-log-group --log-group-name "/ecs/kicks-backend" --region ap-southeast-1
```

### 9.5 (Tuỳ chọn) Xoá secret app-config (nếu chỉ demo)

`02-app` chỉ **đọc** secret theo `app_config_secret_name`, nên secret thường tồn tại độc lập. Nếu bạn tạo secret chỉ để demo:

```powershell
aws secretsmanager delete-secret --secret-id "kicks-shoes-dev/app-config" --region ap-southeast-1 --force-delete-without-recovery
```

---

## 10) Files liên quan trong repo

- Terraform:
  - `infra/terraform/environments/dev/01-network/*`
  - `infra/terraform/environments/dev/02-app/*`
- Docs nền tảng:
  - `docs/aws/backend/04-terraform-guide.md`
  - `docs/aws/backend/ecs-fargate-terraform-quickstart.md`
  - `docs/aws/backend/BE-DEPLOYMENT-CHECKLIST.md`
  - `docs/aws/backend/BE-DEVELOPER-GUIDE.md`
- Script:
  - `scripts/ecs-fargate-e2e.ps1`

---

## 11) Next steps (nếu bạn muốn mình bổ sung)

- Mình có thể bổ sung phần **cấu trúc secrets `app-config`** dựa trên code backend (đang đọc từ key nào), để bạn tạo secret đúng format ngay lần đầu.
- Mình có thể bổ sung phần **custom domain** (ACM us-east-1 + CloudFront + Route53) nếu bạn bật `enable_custom_domain = true`.
