---
title: "IAM"
date: 2026-07-11 10:12:00 +0900
categories: [aws, iam]
tags: [iam, iam-role, terraform]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## IAM 전환 (Access Key 제거 → 역할 기반 최소권한)

### 무엇을 / 왜

1. 코드 설정에 AWS Access Key를 하드코딩하는 방식 없애고, 역할 IAM Role 기반으로 전환한다.

   Access Key는 유출되면 그대로 탈취 → 역할은 AWS가 임시 자격증명을 자동 발급,순환하므로 키를 저장할 필요 없음

   최소 권한 : 앱은 딱 필요한 것만(S3 버킷, SQS 큐)

### 코드 준비

1. DefaultCredentialsProvider 폴백

```java
  private AwsCredentialsProvider credentialsProvider() {
      if (props.accessKey() == null || props.accessKey().isBlank()) {
          return DefaultCredentialsProvider.create();      // 운영: IAM Role 체인 (키 불필요)
      }
      return StaticCredentialsProvider.create(              // 로컬: LocalStack 더미 키
          AwsBasicCredentials.create(props.accessKey(), props.secretKey()));
  }
```

1. Terraform iam 모듈

- 신뢰정책 - 누가 이 역할을 assume할 수 있나

  ```java
  data "aws_iam_policy_document" "ecs_assume" {
    statement {
      actions = ["sts:AssumeRole"]
      principals {
        type        = "Service"
        identifiers = ["ecs-tasks.amazonaws.com"]   # ECS 태스크만 assume 가능
      }
    }
  }
  ```

- 실행 역할(`ECS 에이전트용` : 이미지 pull, 시크릿 주입)

  ```java
  resource "aws_iam_role" "execution" {
    name               = "${var.name}-ecs-execution-role"
    assume_role_policy = data.aws_iam_policy_document.ecs_assume.json
  }
  resource "aws_iam_role_policy_attachment" "execution_managed" {
    role       = aws_iam_role.execution.name
    policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"  # ECR pull +  로그
  }
  ```

- 태스크 역할 (앱 런타임 : S3 + SQS 최소 권한)

  ```java
  data "aws_iam_policy_document" "task" {
    statement {                                   # S3: 버킷 5종에만
      sid       = "ServiceAS3Objects"
      actions   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
      resources = [for b in var.s3_bucket_names : "arn:aws:s3:::${b}/*"]
    }
    statement {                                   # SQS: my-app-* 큐에만
      sid       = "ServiceASqs"
      actions   = ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage", ...]
      resources = ["arn:aws:sqs:ap-northeast-2:${account}:my-app-*"]
    }
  }
  ```
