# 04-terraform — Comprehensive Terraform Guide for Kicks-Shoes Dev Environment

**Scope:** Complete Terraform infrastructure setup, module usage, state management, CI/CD integration, and backend integration for Kicks-Shoes dev/prod environments.

**Document version:** 1.0 | Last updated: April 2026

---

## 1) Terraform là gì và tại sao dùng cho dự án này

### 1.1 Terraform overview

Terraform là Infrastructure as Code (IaC) tool cho phép mô tả hạ tầng AWS (hoặc cloud khác) bằng HCL (HashiCorp Configuration Language). Thay vì click button AWS Console, bạn viết code `.tf` để define resource, commit vào git, review qua PR, và deploy bằng pipeline.

### 1.2 Tại sao Kicks-Shoes dùng Terraform

1. **Version control hạ tầng**: Lịch sử thay đổi, branch strategy, PR review giống như code.
2. **Reproducibility**: Tạo lại hạ tầng dev/prod từ code bất cứ lúc nào.
3. **Team collaboration**: Nhiều dev có thể làm việc trên cùng hạ tầng mà không conflict.
4. **Cost tracking & optimization**: Dễ thấy resource nào consume chi phí.
5. **Separation of concerns**: Network layer riêng (01-network), app layer riêng (02-app).

### 1.3 Tại sao chia thành 01-network và 02-app

```
01-network (state: dev/01-network/terraform.tfstate)
  └─ VPC, subnets, NAT Gateway, route tables
     (Ít thay đổi, stable, reuse giữa dev/prod)

02-app (state: dev/02-app/terraform.tfstate)
  └─ ALB, ECS, RDS, DynamoDB, S3, CloudWatch, etc.
     (Thay đổi thường xuyên theo feature, deploy CI/CD)

01-network outputs → 02-app inputs (via terraform_remote_state)
```

Lợi ích:
- Network layer ổn định, không cần thay đổi khi deploy app code.
- 02-app có thể apply độc lập mà không ảnh hưởng network.
- Lock file (DynamoDB) chỉ lock riêng stack cần apply, không lock toàn bộ.

---

## 2) Cấu trúc Terraform thư mục dự án

```
infra/
├── terraform/
│   ├── modules/                  # Custom modules (optional)
│   │   └── ...
│   │
│   └── environments/
│       ├── dev/
│       │   ├── 01-network/
│       │   │   ├── main.tf
│       │   │   ├── variables.tf
│       │   │   ├── outputs.tf
│       │   │   ├── providers.tf
│       │   │   ├── versions.tf
│       │   │   └── terraform.tfvars.example
│       │   │
│       │   └── 02-app/
│       │       ├── main.tf
│       │       ├── data.tf                  # data sources + remote state
│       │       ├── variables.tf
│       │       ├── outputs.tf
│       │       ├── providers.tf             # alias for us-east-1 (WAF/ACM)
│       │       ├── versions.tf
│       │       └── terraform.tfvars.example
│       │
│       ├── prod/
│       │   ├── 01-network/
│       │   └── 02-app/
│       │
│       └── shared/                         # Optional: shared modules
│           └── ...
```

**State files location (S3 remote backend):**
```
s3://kicks-shoes-tf-state/
├── dev/
│   ├── 01-network/terraform.tfstate
│   └── 02-app/terraform.tfstate
├── prod/
│   ├── 01-network/terraform.tfstate
│   └── 02-app/terraform.tfstate
└── terraform.lock.hcl              # State lock table in DynamoDB
```

---

## 3) Luồng cấu hình Terraform: Từ code đến AWS

### 3.1 Stage 1: Local Development

**Cách làm:**

```bash
# 1. Edit .tf files (main.tf, variables.tf, outputs.tf, providers.tf, versions.tf)
# 2. Create terraform.tfvars từ .tfvars.example
# 3. Initialize Terraform

cd infra/terraform/environments/dev/01-network

terraform init                        # Download providers + modules, setup backend

# 4. Validate + Plan
terraform validate                    # Check syntax
terraform plan -var-file="terraform.tfvars" -out=tfplan
  → Review output, ensure changes match intent

# 5. Apply (local only, for testing)
terraform apply tfplan
  → State file automatically synced to S3
```

**Điểm quan trọng:**
- `terraform.tfvars` chứa sensitive data, không commit vào git, lưu local.
- State lock file (DynamoDB) tự động khóa state khi apply, giải phóng sau.
- Nếu apply thất bại, state vẫn được lưu (partial state), cần manual fix hoặc rollback.

### 3.2 Stage 2: Push to Git & PR

**Workflow:**

```bash
# 1. Commit Terraform code changes
git add infra/terraform/environments/dev/01-network/*.tf
git commit -m "feat: add NAT Gateway auto-scaling"

# 2. Push to feature branch
git push origin feature/nat-gateway

# 3. Create PR on GitHub
# PR description includes:
# - What Terraform resources change
# - Why (business justification)
# - terraform plan output (pasted or summarized)

# 4. GitHub Actions trigger (automatic)
# - terraform validate
# - terraform plan → comment on PR with full plan
# - ESLint, tests, security scan
# - Status: Ready for review
```

**Ý nghĩa:**
- Team lead xem terraform plan trước khi approve.
- Bộc lộ resource thay đổi, giúp catch lỗi sớm (e.g., accidental delete).

### 3.3 Stage 3: Merge & Auto Deploy

**Workflow:**

```bash
# 1. PR approved + merged vào kicks-production branch
# 2. GitHub Actions auto-trigger deploy workflow

# Workflow steps:
# a) Setup: Check out code, setup AWS credentials, Terraform
# b) terraform init
# c) terraform plan -out=tfplan
# d) terraform apply tfplan
#    → State auto-synced to S3
# e) Outputs exported for 02-app consumption
# f) SNS notification (success/failure)
```

**Điểm khác biệt từ local:**
- Credentials lấy từ GitHub Secrets (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN nếu STS).
- State lock tự động release sau apply.
- Nếu fail → alert gửi Slack/SNS.

### 3.4 Cách Terraform nói chuyện với nhau giữa stacks

**Data source: Remote state**

```hcl
# In 02-app/data.tf:

data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "kicks-shoes-tf-state"
    key    = "dev/01-network/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# Usage in 02-app/main.tf:
resource "aws_ecs_service" "backend" {
  vpc_id = data.terraform_remote_state.network.outputs.vpc_id
  subnets = data.terraform_remote_state.network.outputs.private_subnet_ids
  # ...
}
```

**Output dependency:**

```
01-network outputs (terraform.tfstate):
  - vpc_id
  - private_subnet_ids
  - db_subnet_group_name
  - nat_public_ips

02-app reads these outputs via remote_state.
```

---

## 4) Terraform modules dùng sẵn từ terraform-aws-modules (public registry)

### 4.1 Module list & version

| Module | Version | Purpose | Stack |
|--------|---------|---------|-------|
| `terraform-aws-modules/vpc/aws` | ~> 5.0 | VPC, subnets, NAT GW, route tables | 01-network |
| `terraform-aws-modules/security-group/aws` | ~> 5.0 | SG ALB, ECS, RDS, Redis | 02-app |
| `terraform-aws-modules/alb/aws` | ~> 9.0 | ALB, listeners, target groups | 02-app |
| `terraform-aws-modules/ecs/aws//modules/cluster` | ~> 5.0 | ECS cluster | 02-app |
| `terraform-aws-modules/ecs/aws//modules/service` | ~> 5.0 | ECS service + auto scaling | 02-app |
| `terraform-aws-modules/acm/aws` | ~> 5.0 | ACM certificates | 02-app |
| `terraform-aws-modules/elasticache/aws` | ~> 1.0 | ElastiCache Redis cluster | 02-app |
| `terraform-aws-modules/dynamodb-table/aws` | ~> 4.0 | DynamoDB tables | 02-app |
| `terraform-aws-modules/s3-bucket/aws` | ~> 4.0 | S3 buckets + versioning | 02-app |
| `terraform-aws-modules/route53/aws` | ~> 3.0 | Route53 records | 02-app |

### 4.2 Tại sao dùng public modules

1. **Battle-tested**: Được cộng đồng dùng lâu, lỗi đã được fix.
2. **Best practices**: Input validation, security defaults, tagging.
3. **Reduce maintenance**: Không cần maintain custom module code.
4. **Versioning**: Explicit version control, pin module version trong code.
5. **Source of truth**: Tập trung vào business logic, không reinvent wheel.

### 4.3 Cách sử dụng module (example: VPC)

```hcl
# In 01-network/main.tf:

data "aws_availability_zones" "available" {
  state = "available"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-vpc"
  cidr = var.vpc_cidr

  azs              = slice(data.aws_availability_zones.available.names, 0, 2)
  public_subnets   = var.public_subnet_cidrs
  private_subnets  = var.private_subnet_cidrs
  database_subnets = var.db_subnet_cidrs

  enable_nat_gateway     = true
  single_nat_gateway     = true         # dev cost saving
  one_nat_gateway_per_az = false

  enable_dns_hostnames = true
  enable_dns_support   = true

  create_database_subnet_group       = true
  create_database_subnet_route_table = true

  tags = local.common_tags
}
```

**Module interface:**
- **Input**: `source`, `version`, variable arguments.
- **Output**: Module outputs (vpc_id, subnet_ids, etc.).
- Attributes accessed via `module.vpc.vpc_id`.

---

## 5) Terraform State Management: Lưu trữ, khóa, và bảo mật

### 5.1 State file là gì

State file (`.tfstate`) là JSON file chứa snapshot trạng thái AWS resource hiện tại:

```json
{
  "resources": [
    {
      "type": "aws_vpc",
      "name": "main",
      "instances": [
        {
          "attributes": {
            "id": "vpc-12345678",
            "cidr_block": "10.0.0.0/16",
            "tags": { "Name": "kicks-shoes-vpc" }
          }
        }
      ]
    }
  ]
}
```

**Tại sao cần state:**
- Terraform dùng state để biết resource nào đã tạo, cần update hay delete.
- Mapping từ tên resource trong code sang ID thực tế AWS.
- Nếu không có state, Terraform sẽ create duplicate resources mỗi lần apply.

### 5.2 State lưu ở đâu: S3 remote backend

```hcl
# In versions.tf:

terraform {
  backend "s3" {
    bucket         = "kicks-shoes-tf-state"
    key            = "dev/01-network/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**S3 bucket lưu state:**
- **Encryption**: KMS (key management service) mã hóa state at-rest.
- **Versioning**: S3 versioning enabled, có thể rollback state cũ.
- **Access control**: IAM policy chỉ cho authorized role/user access.
- **Backup**: Cross-region replication (optional).

### 5.3 State locking với DynamoDB

```
DynamoDB table: terraform-locks

Khi terraform apply:
  1. Acquire lock: Write entry to terraform-locks table
  2. Apply resources
  3. Release lock: Delete entry from table

Nếu 2 người/pipeline apply cùng lúc:
  → Lock conflict, second apply fail (safe)
```

**Điểm quan trọng:**
- Lock TTL 30 phút (auto-release nếu process crash).
- Manual unlock: `terraform force-unlock <LOCK_ID>` (cẩn thận!).

### 5.4 State sensitive data

State file chứa:
- Database password (RDS, DynamoDB, MongoDB)
- API keys (AWS credentials in environment)
- Secret token
- Encryption key

**Bảo mật state:**
1. S3 bucket encrypted (KMS).
2. Bucket versioning + MFA delete.
3. Bucket policy: Restrict read/write to authorized IAM role chỉ.
4. VPC endpoint (optional): Terraform apply từ EC2 không qua internet.
5. **Không commit state vào git** (add `*.tfstate` vào `.gitignore`).

---

## 6) Terraform init, plan, apply, destroy: Cách hoạt động chi tiết

### 6.1 `terraform init`

**Mục đích**: Khởi tạo working directory, download providers/modules, setup backend.

```bash
terraform init
```

**Nội bộ:**
1. Create `.terraform/` thư mục (local cache).
2. Download provider plugin (hashicorp/aws version 5.0).
3. Download modules từ Terraform Registry (terraform-aws-modules/*).
4. Create `.terraform.lock.hcl` (lock file cho version).
5. Configure backend: Check S3 bucket, DynamoDB lock table, acquire lock.

**Output:**
```
Initializing the backend...
Downloading Terraform modules...
Initializing provider plugins...
Terraform has been successfully initialized!
```

### 6.2 `terraform plan`

**Mục đích**: Tính toán diff giữa code mong muốn vs. state hiện tại, không modify AWS.

```bash
terraform plan -var-file="terraform.tfvars" -out=tfplan
```

**Nội bộ:**
1. Load code từ `.tf` files.
2. Evaluate variables (từ `-var-file` hoặc environment).
3. Read state từ S3 (via backend).
4. Compare desired state (code) vs. actual state (S3).
5. Generate diff:
   - `+` resource: create
   - `~` attribute: modify
   - `-` resource: delete

**Output (example):**
```
Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + cidr_block           = "10.0.0.0/16"
      + enable_dns_hostnames = true
      + tags                 = { "Name" = "kicks-shoes-vpc" }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Điểm quan trọng:**
- `-out=tfplan` lưu plan vào file, đảm bảo apply thực hiện chính xác plan đã xem.
- Plan được comment vào PR, team lead review trước approve.

### 6.3 `terraform apply`

**Mục đích**: Thực thi plan, modify AWS resources, update state.

```bash
terraform apply tfplan
```

**Nội bộ:**
1. Acquire lock từ DynamoDB (chờ nếu locked).
2. For each resource in plan:
   a. Call AWS API (create/update/delete).
   b. Update state file in memory.
   c. On success: state persisted to S3.
   d. On failure: roll back in-memory state, release lock.
3. Release lock từ DynamoDB.
4. Output: Outputs value (e.g., vpc_id).

**Output:**
```
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 2s [id=vpc-12345678]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

vpc_id = "vpc-12345678"
```

### 6.4 `terraform destroy`

**Mục đích**: Xóa tất cả resource được manage bởi Terraform stack này.

```bash
terraform destroy -var-file="terraform.tfvars"
```

**Cẩn thận:**
- Xóa mọi resource: VPC, subnets, NAT GW, ALB, ECS, RDS, S3, etc.
- **KHÔNG dùng destroy cho production**, chỉ dev/test.
- Nếu S3 bucket chứa data quan trọng, `terraform destroy` sẽ fail (safe).

---

## 7) CI/CD Pipeline & GitHub Actions Integration

### 7.1 GitHub Actions Workflow trigger

```yaml
# .github/workflows/deploy.yml

name: Deploy Terraform

on:
  pull_request:
    paths:
      - 'infra/terraform/**'
      - 'backend/**'
      - '.github/workflows/deploy.yml'
  push:
    branches: [kicks-production]
    paths:
      - 'infra/terraform/**'
      - 'backend/**'

jobs:
  terraform-plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Init (dev/01-network)
        run: |
          cd infra/terraform/environments/dev/01-network
          terraform init
      
      - name: Terraform Plan (dev/01-network)
        run: |
          cd infra/terraform/environments/dev/01-network
          terraform plan -var-file="terraform.tfvars" -out=tfplan
      
      - name: Comment PR with Plan
        uses: actions/github-script@v6
        with:
          script: |
            // Post plan output to PR comment
            github.rest.issues.createComment({...})

  terraform-apply:
    if: github.ref == 'refs/heads/kicks-production' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ap-southeast-1
      
      - name: Terraform Apply (dev/01-network)
        run: |
          cd infra/terraform/environments/dev/01-network
          terraform init
          terraform plan -out=tfplan
          terraform apply tfplan
      
      - name: Terraform Apply (dev/02-app)
        run: |
          cd infra/terraform/environments/dev/02-app
          terraform init
          terraform plan -out=tfplan
          terraform apply tfplan
      
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "Terraform deploy: ${{ job.status }}"}
```

### 7.2 Deploy order: 01-network trước, 02-app sau

**Tại sao:**

```
01-network output: vpc_id, subnet_ids, db_subnet_group_name
                   ↓
02-app input: Sử dụng VPC, subnets từ 01-network
                   ↓
02-app: Không thể apply nếu 01-network chưa xong
```

**Trong CI/CD pipeline:**

```yaml
jobs:
  stage-1-network:
    runs-on: ubuntu-latest
    steps:
      - name: Apply dev/01-network
  
  stage-2-app:
    needs: stage-1-network    # Chờ stage-1 success
    runs-on: ubuntu-latest
    steps:
      - name: Apply dev/02-app
```

**Nếu không đúng thứ tự:**
- 02-app apply trước 01-network → Lỗi: `data source not found` (remote state empty).
- Terraform fail, không auto-retry.

---

## 8) Terraform x Backend ECS integration: Cách Terraform deploy backend

### 8.1 Terraform define ECS task definition

```hcl
# In 02-app/main.tf:

resource "aws_ecs_task_definition" "backend" {
  family                   = "kicks-shoes-backend"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 256
  memory                   = 512

  container_definitions = jsonencode([
    {
      name      = "backend"
      image     = var.container_image  # Passed from CI/CD
      portMappings = [
        {
          containerPort = 3000
          hostPort      = 3000
          protocol      = "tcp"
        }
      ]
      environment = [
        { name = "NODE_ENV", value = "production" },
        { name = "PORT", value = "3000" },
        { name = "HOST", value = "0.0.0.0" },
        { name = "AWS_REGION", value = var.aws_region },
        { name = "DYNAMODB_TABLE_NAME", value = aws_dynamodb_table.main.name }
      ]
      secrets = [
        {
          name      = "MONGODB_URI"
          valueFrom = aws_secretsmanager_secret.mongodb_uri.arn
        }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.ecs.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "backend"
        }
      }
    }
  ])

  execution_role_arn = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn      = aws_iam_role.ecs_task_role.arn
}
```

**Điểm chính:**
- `var.container_image` = ECR image URI từ CI/CD pipeline.
- Environment variables: PORT, HOST, AWS_REGION, table name.
- Secrets: MONGODB_URI inject từ Secrets Manager (không hardcode).
- IAM role: Quyền để container access DynamoDB, S3, Secrets Manager.

### 8.2 Terraform define ECS service

```hcl
module "ecs_service" {
  source  = "terraform-aws-modules/ecs/aws//modules/service"
  version = "~> 5.0"

  name        = "kicks-shoes-backend-service"
  cluster_arn = module.ecs_cluster.arn

  desired_count = 2
  launch_type   = "FARGATE"
  cpu           = 256
  memory        = 512

  subnet_ids         = data.terraform_remote_state.network.outputs.private_subnet_ids
  security_group_ids = [aws_security_group.ecs.id]
  assign_public_ip   = false

  task_definition_arn = aws_ecs_task_definition.backend.arn

  load_balancer = {
    service = {
      target_group_arn = module.alb.target_groups["backend"].arn
      container_name   = "backend"
      container_port   = 3000
    }
  }

  autoscaling_min_capacity = 2
  autoscaling_max_capacity = 10
  autoscaling_target_cpu   = 60

  enable_execute_command = false  # Disable ECS exec for security

  tags = local.common_tags
}
```

**Điểm chính:**
- Task đẩy vào private subnets (không public IP).
- ALB target group định tuyến traffic vào port 3000.
- Auto scaling: Min 2, max 10, scale out khi CPU > 60%.
- ECS exec disabled (security best practice).

### 8.3 Backend code lấy config từ Terraform output

```javascript
// backend/src/app.js

import dotenv from 'dotenv';
dotenv.config();

const PORT = process.env.PORT || 3000;
const HOST = process.env.HOST || '0.0.0.0';
const AWS_REGION = process.env.AWS_REGION || 'ap-southeast-1';
const DYNAMODB_TABLE_NAME = process.env.DYNAMODB_TABLE_NAME;
const MONGODB_URI = process.env.MONGODB_URI;

// Connect to DynamoDB with region từ Terraform
const dynamoClient = new DynamoDBClient({ region: AWS_REGION });

// Bind server tới HOST:PORT
const server = http.createServer(app);
server.listen(PORT, HOST, () => {
  console.log(`Server running on ${HOST}:${PORT}`);
});

export { server };
```

**Luồng:**
1. CI/CD build image → push ECR.
2. Terraform update task definition với image URI mới.
3. Terraform apply → ECS service launch task mới.
4. Container startup → `dotenv.config()` load env từ task definition.
5. Backend connect tới DynamoDB region từ Terraform.

---

## 9) Import state: Khi nào và cách làm

### 9.1 Khi nào import state

**Scenario:**
- Resource AWS tạo bằng AWS Console hoặc script cũ.
- Giờ muốn quản lý bằng Terraform.
- Cần import state cũ vào Terraform state file, để Terraform không tạo duplicate.

**Example:** S3 bucket `kicks-shoes-uploads` tạo bằng Console, giờ muốn Terraform quản lý.

### 9.2 Cách import

```bash
# 1. Define resource block (empty) trong 02-app/main.tf:
resource "aws_s3_bucket" "uploads" {
  # Empty, sẽ import attributes từ AWS
}

# 2. Import AWS resource vào Terraform state:
cd infra/terraform/environments/dev/02-app

terraform import aws_s3_bucket.uploads kicks-shoes-uploads
  → Terraform queries AWS API, fetch bucket attributes
  → Merge vào state file

# 3. Verify import:
terraform state show aws_s3_bucket.uploads
terraform plan    # Should show no changes

# 4. Add config từ imported attributes:
resource "aws_s3_bucket" "uploads" {
  bucket = "kicks-shoes-uploads"
  tags = {
    Name = "kicks-shoes-uploads"
  }
}

# 5. Re-plan + apply:
terraform plan
terraform apply
```

**Cẩn thận:**
- Import chỉ copy state, không copy `.tf` config → phải viết config tay.
- Sau import, plan sẽ hiện thay đổi (cấu hình tay vs. actual AWS) → sync config.
- Nếu config sai, apply sẽ modify AWS resource → có thể break app.

---

## 10) Terraform workflow từ dev lokkal đến production

### 10.1 Workflow step-by-step

```
Developer local machine:
  1. Edit infra/terraform/environments/dev/02-app/main.tf
     (Example: Thêm DynamoDB GSI)
  
  2. terraform plan (local)
     → Review output
  
  3. git commit + push feature branch
     → Create PR
  
  4. GitHub Actions trigger (on PR):
     - terraform validate ✓
     - terraform plan → comment to PR
     - Code review + approval

PR Merge:
  5. Merge to kicks-production
     → GitHub Actions trigger deploy job
  
  6. Deploy job:
     a) terraform init
     b) terraform plan -out=tfplan
        → Acquire lock (DynamoDB)
     c) terraform apply tfplan
        → DynamoDB GSI created, state updated to S3
        → Release lock
     d) terraform output → export values
     e) ECS task definition updated
     f) SNS notification (success)

Result:
  7. Backend container running with new DynamoDB GSI configured
     Terraform state synced to S3
     Team notified
```

### 10.2 Rollback strategy

**Nếu apply gây lỗi:**

```bash
# Option 1: Rollback via git + Terraform

# 1. Identify previous commit
git log --oneline infra/terraform/

# 2. Revert commit
git revert <commit-hash>
git push origin kicks-production

# 3. GitHub Actions auto-deploy với revert code
#    → terraform apply removes bad resource

# Option 2: Manual fix + fix commit
# 1. Edit .tf file sửa lỗi
# 2. git commit + push
# 3. GitHub Actions auto-deploy
```

**Terraform-level rollback:**

```bash
# Use S3 version history (nếu enabled)

# 1. List previous versions
aws s3api list-object-versions --bucket kicks-shoes-tf-state --prefix dev/02-app/

# 2. Get previous state version ID
version_id=XXXXXXXXXXX

# 3. Download old state
aws s3api get-object \
  --bucket kicks-shoes-tf-state \
  --key dev/02-app/terraform.tfstate \
  --version-id $version_id \
  terraform.tfstate.old

# 4. Manual restore (cẩn thận!)
# Replace current state với old state
cp terraform.tfstate.old terraform.tfstate

# 5. terraform plan to verify
terraform plan
# Should show resources being deleted to rollback
```

**Best practice:** Rollback via git revert + auto-deploy (safest).

---

## 11) Troubleshooting Terraform issues

| Vấn đề | Nguyên nhân | Cách fix |
|---|---|---|
| `Error acquiring the state lock` | Khóa từ apply trước còn active | Chờ 30 phút auto-release, hoặc `terraform force-unlock <LOCK_ID>` |
| `data source not found` (in 02-app) | 01-network chưa apply | Apply 01-network trước |
| `InvalidParameterException: Invalid image` | Container image URI sai | Verify ECR image exist, tag correct |
| `AccessDenied` to S3/DynamoDB | IAM role chưa có quyền | Add quyền vào IAM policy |
| `terraform plan` khác giữa local vs. CI/CD | Terraform version khác | Pin version trong .github/workflows |
| State file bị corrupt | S3 version deleted | Restore từ backup hoặc rebuild |

---

## 12) Best practices & checklist

### 12.1 Checklist trước mỗi apply

- [ ] `terraform plan` reviewed, no unexpected changes
- [ ] Variables correct (region, environment, image)
- [ ] State file location (S3 bucket) accessible
- [ ] AWS credentials valid (không expired token)
- [ ] Terraform version match CI/CD version (1.5.0)
- [ ] Modules updated (`terraform init -upgrade`)
- [ ] No hardcoded secret/password trong .tf code

### 12.2 Best practices

1. **Small commits**: Thay đổi logic -> logic commit, infrastructure -> infra commit. Dễ review, dễ rollback.
2. **Plan before apply**: Luôn chạy plan, review output trước apply.
3. **Pin module versions**: Không dùng `>= 5.0`, dùng `~> 5.0` (fixed minor).
4. **Tag resources**: Mọi resource tagged với `Environment`, `Project`, `ManagedBy = terraform`.
5. **Separate concerns**: Network layer riêng, app layer riêng.
6. **Backup state**: S3 versioning + cross-region replication.
7. **Lock state**: Dùng DynamoDB lock, tránh concurrent apply.
8. **Document variables**: Comment rõ input variable, tại sao cần, giá trị default.

---

## 13) FAQ - Frequently Asked Questions

### Q1: Terraform state file bị xóa hoặc mất, làm sao khôi phục?

**A:** 
1. Check S3 versioning: `aws s3api list-object-versions --bucket kicks-shoes-tf-state`
2. Restore từ version cũ: `aws s3api get-object --bucket kicks-shoes-tf-state --key dev/02-app/terraform.tfstate --version-id <VERSION_ID> terraform.tfstate`
3. Reupload state: Copy file về local, verify content, push lại S3.
4. **Best practice**: Bật S3 versioning + MFA delete để tránh vô tình xóa state.

---

### Q2: Pipeline fail khi apply Terraform, làm sao rollback nhanh?

**A:**
1. **Quick rollback**: Revert PR (sử dụng GitHub `revert` button), auto-trigger deploy với code cũ.
2. **Manual fix**: Edit `.tf` file, fix lỗi, push lại, auto-deploy.
3. **Force unlock** (nếu stuck lock): `terraform force-unlock <LOCK_ID>` nhưng **cẩn thận**, chỉ dùng khi chắc lock là stale.
4. **Verify after rollback**: `terraform plan` trước apply để chắc chắn.

---

### Q3: Làm sao biết infrastructure hiện tại đang ở trạng thái nào?

**A:**
```bash
# Cách 1: Xem Terraform state
cd infra/terraform/environments/dev/02-app
terraform state list            # List tất cả resource
terraform state show aws_ecs_service.kicks  # Chi tiết một resource
terraform output               # Outputs (ALB DNS, ECS cluster name)

# Cách 2: AWS Console
# EC2 → Load Balancers → Check ALB health
# ECS → Cluster → Check service running tasks

# Cách 3: AWS CLI
aws ecs describe-services --cluster kicks-shoes-dev --services kicks-shoes-dev
aws elbv2 describe-target-health --target-group-arn arn:...
aws logs tail /ecs/kicks-shoes-dev --follow
```

---

### Q4: Backend code thay đổi nhưng Terraform `.tf` không thay đổi, cần apply lại không?

**A:** **Không cần**. Nếu chỉ backend code thay đổi (Java/Node.js/Python):
1. Build image mới (CI/CD).
2. Push ECR.
3. **Không cần `terraform apply`** (infrastructure không đổi).
4. ECS auto-deploy nếu task definition updated (hoặc manual force new deployment).

**Ngoại lệ**: Nếu Terraform code có thay đổi (ví dụ update container_port, add DynamoDB GSI), **phải apply**.

---

### Q5: Cách update ECS task definition khi backend code thay đổi?

**A:**
```hcl
# In Terraform 02-app/main.tf:

resource "aws_ecs_task_definition" "backend" {
  # ...
  container_definitions = jsonencode([
    {
      image = var.container_image  # <- CI/CD pass new image URI
      # ...
    }
  ])
}

# CI/CD pipeline:
# 1. Build image, push ECR: kicks-shoes-backend:abc123
# 2. terraform apply -var=container_image=<ECR_URI>
#    → Task definition updated with new image
# 3. ECS service force new deployment
#    → Old tasks terminated, new tasks launched with new image
```

---

### Q6: Terraform plan hiện quá nhiều thay đổi, không biết cái nào quan trọng?

**A:**
1. **Filter by resource type**: `terraform plan | grep -E "^(aws_|module\.)"` xem resource nào thay đổi.
2. **Check critical resource**: Focus on ALB, ECS service, DynamoDB table, security group.
3. **Identify unexpected changes**: Nếu plan hiện xóa resource không dự tính → **STOP**, review trước apply.
4. **Save plan**: `terraform plan -out=tfplan` rồi review file: `terraform show tfplan`.

---

### Q7: Làm sao pin module version để tránh breaking change?

**A:**
```hcl
# ❌ KHÔNG TỐT (có thể break)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0"
}

# ✅ TỐT (pin minor version)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.3"  # Cho phép 5.3, 5.4, ... nhưng không 6.0
}

# ✅ CHÍNH XÁC NHẤT (pin exact version)
module "ecs" {
  source  = "terraform-aws-modules/ecs/aws//modules/service"
  version = "5.9.2"
}
```

---

### Q8: State lock bị stuck (terraform-locks DynamoDB), làm sao mở?

**A:**
```bash
# 1. Check lock status
aws dynamodb scan --table-name terraform-locks --region ap-southeast-1

# 2. Identify lock ID
lock_id="..." # Lấy từ output scan

# 3. Force unlock (CẨN THẬN: chỉ dùng khi chắc lock là stale)
cd infra/terraform/environments/dev/02-app
terraform force-unlock $lock_id

# 4. Verify unlock
terraform plan  # Nếu chạy được → lock mở thành công

# LƯỚI: Nếu force-unlock sai cách, có thể corrupt state!
# Luôn backup state trước: aws s3 cp s3://kicks-shoes-tf-state/... local/
```

---

### Q9: Cách chạy Terraform local không thông qua CI/CD?

**A:**
```bash
# 1. Setup AWS credentials (local machine)
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="ap-southeast-1"

# 2. Create terraform.tfvars (từ .tfvars.example)
cd infra/terraform/environments/dev/01-network
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars, đặt giá trị đúng

# 3. Init + Plan
terraform init
terraform plan -var-file="terraform.tfvars" -out=tfplan

# 4. Review plan (IMPORTANT: xem kỹ trước apply)
terraform show tfplan

# 5. Apply
terraform apply tfplan
# → State tự động synced to S3 backend

# ⚠️ LƯU Ý:
# - terraform.tfvars chứa sensitive data, NOT commit to git
# - Lock được acquire từ DynamoDB, cẩn thận khi local apply
# - Nếu fail, state có thể partial update, cần manual fix hoặc rollback
```

---

### Q10: Làm sao tìm ra resource nào consume chi phí nhiều nhất?

**A:**
```bash
# 1. List tất cả resource trong state
cd infra/terraform/environments/dev/02-app
terraform state list

# 2. Chi tiết từng resource
terraform state show aws_ec2_nat_gateway.kicks
terraform state show aws_dynamodb_table.main
terraform state show aws_elasticache_cluster.redis

# 3. AWS Console: Cost Explorer
#    - Filter by resource tag (ManagedBy=terraform)
#    - View by service (ECS, DynamoDB, NAT Gateway, etc.)

# 4. Common cost drivers:
#    - NAT Gateway ($32/month/gateway)
#    - Data transfer (inter-AZ, egress)
#    - DynamoDB on-demand (pay per request)
#    - RDS (nếu dùng provisioned capacity)
#    - CloudFront (nếu high traffic)

# 5. Optimization:
#    - Single NAT Gateway (dev), 1 per AZ (prod) → save cost dev
#    - DynamoDB on-demand → provisioned capacity nếu stable traffic
#    - ElastiCache → optional, chỉ dùng nếu cần cache
#    - S3: Add lifecycle policy để auto-delete old data
```

---

### Q11: Team member A apply Terraform, Team member B try apply cùng lúc, điều gì xảy ra?

**A:**
```
Team A: terraform apply tfplan
  → DynamoDB lock acquired
  → Applying resources...
  
Team B: terraform apply tfplan (cùng lúc)
  → Waiting for lock (stuck)...
  → After A release lock → B acquire lock → B apply
  
Result: Sequential apply, safe, no conflict
```

**Nếu không có lock:**
- A + B apply đồng thời → conflict → state corrupt → disaster.

**Lock timeout:**
```bash
# Lock TTL: 30 minutes (auto-release nếu process crash)
# Nếu vượt quá 30 min (hung process), lock tự release

# Check lock
aws dynamodb scan --table-name terraform-locks

# Manually release (cẩn thận!)
terraform force-unlock <LOCK_ID>
```

---

### Q12: Cách migrate resource từ 01-network sang 02-app (hoặc ngược lại)?

**A:**
```bash
# Scenario: Thay đổi cấu trúc state (chia tách resource)

# 1. Copy resource definition từ 01-network vào 02-app

# 2. Remove từ 01-network (để tránh conflict)
cd infra/terraform/environments/dev/01-network
terraform state rm aws_security_group.backend
  → Remove khỏi state 01-network

# 3. Import vào 02-app
cd ../02-app
terraform import aws_security_group.backend sg-12345678
  → Import vào state 02-app

# 4. Verify
terraform state show aws_security_group.backend
terraform plan  # Should show no changes

# 5. Apply both stacks
cd ../01-network && terraform apply
cd ../02-app && terraform apply

# ⚠️ CẨN THẬN: Resource tạm thời không có state → dễ bị tạo duplicate
# Luôn backup state trước: aws s3 sync s3://kicks-shoes-tf-state/ backup/
```

---

### Q13: Terraform plan output quá dài, làm sao refine?

**A:**
```bash
# Lọc theo resource type
terraform plan | grep "aws_ecs"        # Chỉ ECS changes
terraform plan | grep "aws_dynamodb"   # Chỉ DynamoDB changes

# Lọc destroy operations
terraform plan | grep "destroy"

# Lọc thay đổi cụ thể
terraform plan | grep "will be updated"

# Save + view từng phần
terraform plan -out=tfplan
terraform show tfplan | head -100     # 100 dòng đầu
terraform show tfplan | grep "name =" # Chỉ xem name attribute

# Format JSON (dễ parse)
terraform plan -out=tfplan -json | jq '.resource_changes[] | select(.type == "aws_ecs_service")'
```

---

### Q14: Làm sao trigger Terraform deploy thủ công mà không đợi git push?

**A:**
```bash
# Cách 1: GitHub Actions dispatch (manual trigger)
# Setup trong .github/workflows/deploy.yml:

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment (dev/prod)'
        required: true
        default: 'dev'
      apply:
        description: 'Apply Terraform? (yes/no)'
        required: true
        default: 'no'

# Trigger via GitHub UI:
# Actions tab → Deploy workflow → Run workflow → Select input → Run

# Cách 2: GitHub CLI (local)
gh workflow run deploy.yml -f environment=dev -f apply=yes

# Cách 3: Manual AWS (không recommended)
cd infra/terraform/environments/dev/02-app
aws sts assume-role --role-arn arn:aws:iam::...:role/TerraformRole
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
terraform init && terraform apply -auto-approve  # CẨN THẬN!
```

---

### Q15: Cách viết Terraform test để verify infrastructure?

**A:**
```bash
# Terraform built-in: terraform validate + terraform plan

# Advanced: Terratest (Go framework)
# Example: infra/tests/vpc_test.go

package tests

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestVpc(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../terraform/environments/dev/01-network",
    }
    
    terraform.InitAndApply(t, opts)
    defer terraform.Destroy(t, opts)
    
    vpcId := terraform.Output(t, opts, "vpc_id")
    assert.NotNil(t, vpcId)
}

# Run test
go test -v terratest

# Tool thay thế: Checkov (static security scan)
checkov -d infra/terraform/environments/dev/ --framework terraform
```

---

## 14) Reference & External Resources

- **Terraform AWS Provider**: https://registry.terraform.io/providers/hashicorp/aws/latest
- **terraform-aws-modules**: https://github.com/terraform-aws-modules
- **AWS S3 backend config**: https://www.terraform.io/language/backends/s3
- **GitHub Actions setup-terraform**: https://github.com/hashicorp/setup-terraform

---

## 15) Liên hệ nhanh các file Terraform

- **Dev network**: `infra/terraform/environments/dev/01-network/`
- **Dev app**: `infra/terraform/environments/dev/02-app/`
- **Prod network**: `infra/terraform/environments/prod/01-network/`
- **Prod app**: `infra/terraform/environments/prod/02-app/`
- **CI/CD workflow**: `.github/workflows/deploy.yml`
- **State backend**: S3 bucket `kicks-shoes-tf-state`
- **State lock table**: DynamoDB table `terraform-locks`
