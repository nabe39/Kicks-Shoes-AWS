# Tagging Strategy — Group G<N>

## Tag keys bắt buộc

| Key | Giá trị ví dụ | Quy tắc |
|-----|---------------|---------|
| Owner | vohongduc000@gmail.com | Một người chịu trách nhiệm. Luôn viết thường. |
| Environment | dev / staging / prod | Không trộn 'dev' và 'Dev'. |
| CostCenter | G13 | Group ID, không đổi. |
| Application | HealthBot | PascalCase. Không có 'healthbot' và 'HealthBot' cùng tồn tại. |

## Enforcement (production)
- IAM policy có condition `aws:RequestTag/Owner` để reject create không tag
- AWS Config rule `required-tags` để detect resource thiếu tag
- Lambda remediate (giống Self-Healing Security Guard) để stop resource không tag sau 1h

B.1 — Tạo S3 bucket có tag
# Set biến môi trường
export GROUP_ID="G13"                  
export OWNER="vohongduc000@gmail.com"   
export APP_NAME="LabCostV-duc"
export BUCKET_NAME="${APP_NAME,,}-${GROUP_ID,,}-$(date +%s)"
export REGION="us-east-1"

# Tạo bucket
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$REGION" \

# Áp tag NGAY (đừng để tới sau)
aws s3api put-bucket-tagging \
  --bucket "$BUCKET_NAME" \
  --tagging "TagSet=[
    {Key=Owner,Value=$OWNER},
    {Key=Environment,Value=dev},
    {Key=CostCenter,Value=$GROUP_ID},
    {Key=Application,Value=$APP_NAME}
  ]"

B.2 — Tạo Lambda function có tag
# Tạo IAM role basic cho Lambda (tối thiểu)
aws iam create-role \
  --role-name "${APP_NAME}-lambda-role" \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }' \
  --tags "Key=Owner,Value=$OWNER" "Key=Application,Value=$APP_NAME" "Key=CostCenter,Value=$GROUP_ID"

aws iam attach-role-policy \
  --role-name "${APP_NAME}-lambda-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Đợi 10 giây cho IAM propagate
sleep 10

# Tạo function code rỗng
echo 'def lambda_handler(event, context): return {"statusCode": 200}' > /tmp/handler.py
cd /tmp && zip handler.zip handler.py

aws lambda create-function \
  --function-name "${APP_NAME}-hello" \
  --runtime python3.12 \
  --handler handler.lambda_handler \
  --role "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/${APP_NAME}-lambda-role" \
  --zip-file fileb:///tmp/handler.zip \
  --tags "Owner=$OWNER,Environment=dev,CostCenter=$GROUP_ID,Application=$APP_NAME"


B.3 — Tạo EC2 instance có tag (rồi STOP ngay)
# Lấy AMI Amazon Linux 2023 mới nhất
AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64 \
  --query 'Parameters[0].Value' --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --tag-specifications "ResourceType=instance,Tags=[
    {Key=Name,Value=${APP_NAME}-test},
    {Key=Owner,Value=$OWNER},
    {Key=Environment,Value=dev},
    {Key=CostCenter,Value=$GROUP_ID},
    {Key=Application,Value=$APP_NAME}
  ]" \
  --query 'Instances[0].InstanceId' --output text)

echo "Instance ID: $INSTANCE_ID"

# STOP NGAY để tiết kiệm chi phí — chỉ cần resource tồn tại để có dòng billing
sleep 30
aws ec2 stop-instances --instance-ids "$INSTANCE_ID"