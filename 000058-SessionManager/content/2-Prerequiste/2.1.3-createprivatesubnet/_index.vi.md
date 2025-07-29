---
title : "Triển Khai Dự Án AWS CDK Kết Nối ECS, EC2, IAM"
date :  "`r Sys.Date()`" 
weight : 4
chapter : false
pre : " <b> 2.1.3 </b> "
---


# 🚀 Hướng Dẫn Triển Khai Ứng Dụng Trên AWS Bằng AWS CDK

Tài liệu này hướng dẫn cách thiết lập và triển khai một ứng dụng Spring Boot sử dụng AWS CDK với các dịch vụ liên quan: ECS, EC2 (VPC), IAM và CloudWatch Logs. Dự án sử dụng AWS Fargate để chạy container mà không cần quản lý server.

---

## ✅ 1. Cài Đặt Môi Trường AWS CDK Trên Ubuntu

### 🔹 Gỡ bỏ Node.js/NPM cũ (nếu có)

```bash
sudo yum remove -y nodejs npm
sudo yum autoremove -y

```

### 🔹 Cài đặt NVM và Node.js (Phiên bản ổn định)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"
nvm install 22.9.0
nvm use 22.9.0
```

### 🔹 Cài đặt AWS CDK

```bash
npm install -g aws-cdk
```

Kiểm tra phiên bản:

```bash
cdk --version
```

---

## ✅ 2. Khởi Tạo Dự Án CDK TypeScript

```bash
mkdir test-cdk && cd test-cdk
cdk init app --language typescript
```

### 🔹 Cài thêm các thư viện AWS cần thiết

```bash
npm install @aws-cdk/aws-ecs @aws-cdk/aws-ec2 @aws-cdk/aws-ecs-patterns @aws-cdk/aws-rds
```

---

## ✅ 3. Tạo Stack CDK (lib/webenglish-cdk-stack.ts)

Tạo file `lib/webenglish-cdk-stack.ts` với nội dung:

```ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as logs from 'aws-cdk-lib/aws-logs';
import { Construct } from 'constructs';

export class WebenglishCdkStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'MyVpc', {
      maxAzs: 2,
      subnetConfiguration: [
        { subnetType: ec2.SubnetType.PUBLIC, name: 'Public' },
        { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, name: 'Private' },
      ],
    });

    const cluster = new ecs.Cluster(this, 'MyCluster', { vpc });

    const executionRole = iam.Role.fromRoleArn(
      this,
      'EcsExecutionRole',
      'arn:aws:iam::466322313916:role/WebenglishEcsExecutionRole'
    );

    const logGroup = new logs.LogGroup(this, 'WebenglishLogGroup', {
      retention: logs.RetentionDays.ONE_WEEK,
    });

    const taskDefinition = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      memoryLimitMiB: 1024,
      cpu: 512,
      executionRole,
    });

    taskDefinition.addContainer('WebenglishApp', {
      image: ecs.ContainerImage.fromRegistry(
        '466322313916.dkr.ecr.ap-northeast-1.amazonaws.com/webenglish:webenglish-app'
      ),
      logging: ecs.LogDriver.awsLogs({
        logGroup,
        streamPrefix: 'webenglish-app',
      }),
      portMappings: [{ containerPort: 8080 }],
      environment: {
        SPRING_DATASOURCE_URL: 'jdbc:mysql://localhost:3306/webenglish',
        SPRING_DATASOURCE_USERNAME: 'root',
        SPRING_DATASOURCE_PASSWORD: '123456',
      },
    });

    new ecs.FargateService(this, 'WebenglishService', {
      cluster,
      taskDefinition,
      desiredCount: 1,
      assignPublicIp: true,
    });
  }
}
```

---

## ✅ 4. Cấu Hình AWS CLI

### 🔹 Đăng nhập AWS

```bash
aws configure
```

Nhập:
- `AWS Access Key ID`
- `AWS Secret Access Key`
- `Region`: `ap-northeast-1`

---

## ✅ 5. Tạo IAM Role Cho ECS Task

### Trên AWS Console:

1. Vào **IAM > Roles**
2. Tạo role `WebenglishEcsExecutionRole`
3. Gán chính sách:
   - `AmazonECSTaskExecutionRolePolicy`
4. Ghi lại ARN của role:  
   Ví dụ:  
   `arn:aws:iam::466322313916:role/WebenglishEcsExecutionRole`

---

## ✅ 6. Bootstrap CDK & Deploy

```bash
cdk bootstrap aws://466322313916/ap-northeast-1
cdk synth
cdk deploy
```

---

## 📌 Ghi Chú Bổ Sung

- Bạn cần phải tạo ECR và đẩy image `webenglish:webenglish-app` lên trước đó.
- Nếu dùng database RDS thực sự, cần tạo một RDS instance và cập nhật `SPRING_DATASOURCE_URL`.

---

## 🧠 Mở Rộng Sau Này

- Tích hợp Load Balancer (ALB) bằng `ApplicationLoadBalancedFargateService`
- Kết nối RDS thông qua Security Group và Secrets Manager
- Tích hợp CI/CD: GitHub Actions hoặc AWS CodePipeline

---

## 🧾 Tài Liệu Tham Khảo

- https://docs.aws.amazon.com/cdk/
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
