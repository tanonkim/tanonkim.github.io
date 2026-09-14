---
title: "ECS + Fargate"
date: 2026-07-11 09:37:00 +0900
categories: [aws, ecs]
tags: [ecs, fargate, terraform]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## ECS + Fargate 구성

### 무엇을 / 왜

컨테이너를 ECS Fargate로 운영한다.

ECS = 컨테이너 오케스트레이션, Fargate = 어디서 돌릴지를 AWS가 관리(서버리스)

### Fargate VS EC2

| 관점 | ECS + EC2 | **ECS + Fargate** |
|---|---|---|
| 서버 관리 | 인스턴스, ASG/AMI/OS 패치 직접 | AWS가 관리 |
| 비용(저트래픽) | 인스턴스 통째 과금 | 태스크 당 과금, 유휴 없음 |
| 비용(고트래픽) | vCPU당 저렴(고밀도 시 이득) | 다소 비쌈 |
| Auto Scaling | 직접 구성 | 간단 |

### Fargate로 결정 근거

- 저트래픽 + 솔로 개발 → EC2의 인스턴스/ASG(오토스케일그룹)/패치 관리 부담이 큼.
- 비용은 저밀도(태스크 1~2개, HA 2AZ)에선 사실상 무승부 — 오히려 Fargate가 유리(24시간 유휴 인스턴스 불필요, HA도 태스크만 2AZ 배치).
- EC2가 확실히 싸지는 건 인스턴스를 태스크로 꽉 채우는 고밀도일 때. 그때는 트래픽 데이터가 있으니 그때 옮기면 됨.
- "24시간 서비스면 EC2가 경제적"은 고밀도 전제. 저밀도에선 성립 안 함 (인스턴스 유휴 용량도 과금되므로).

### 구현

1. 클러스터 + 용량 공급자

```hcl
  resource "aws_ecs_cluster" "this" {
    name = "${var.name}-cluster"
  }
  resource "aws_ecs_cluster_capacity_providers" "this" {
    cluster_name       = aws_ecs_cluster.this.name
    capacity_providers = ["FARGATE", "FARGATE_SPOT"]   # 둘 다 등록, 서비스가 전략 선택
  }
```

1. 태스크 정의(Fargate) - TF가 소유

```hcl
  resource "aws_ecs_task_definition" "this" {
    family                   = "${var.name}-api"
    requires_compatibilities = ["FARGATE"]
    network_mode             = "awsvpc"        # Fargate 필수: 태스크마다 ENI+사설IP
    cpu                      = "1024"          # 1 vCPU
    memory                   = "2048"          # 2GB → JVM 75%면 heap ~1.5GB
    execution_role_arn       = var.execution_role_arn   # ECS가 이미지 pull·시크릿 주입
    task_role_arn            = var.task_role_arn         # 앱 런타임(S3·SQS)
    runtime_platform {
      operating_system_family = "LINUX"
      cpu_architecture        = "X86_64"
    }
    container_definitions = local.container_definitions  # 컨테이너 정의(JSON)
  }
```

1. env 자동 배선(IaC_코드형 인프라의 힘)

```hcl
  app_environment = {
    SPRING_PROFILES_ACTIVE = "production"
    SERVICE_A_DB_URL       = "jdbc:mysql://${module.rds.endpoint}:3306/appdb"  # ← RDS 모듈 출력
    SERVICE_A_REDIS_HOST   = var.redis_host
    SERVICE_A_CORS_ALLOWED_ORIGINS = join(",", var.frontend_origins)
    ...
  }
```

1. 서비스 (**Spot** 전략 + ALB 연결)

```hcl
  resource "aws_ecs_service" "this" {
    cluster         = aws_ecs_cluster.this.id
    task_definition = aws_ecs_task_definition.this.arn
    desired_count   = var.autoscale_min

    dynamic "capacity_provider_strategy" {   # 순수 FARGATE_SPOT (아래 Spot 참고)
      for_each = local.cp_strategy
      content {
        capacity_provider = capacity_provider_strategy.value.capacity_provider
        weight            = capacity_provider_strategy.value.weight
        base              = capacity_provider_strategy.value.base
      }
    }
    network_configuration {
      subnets          = var.app_subnet_ids   # 프라이빗 서브넷
      security_groups  = [var.security_group_id]  # app-sg (ALB에서 8080만)
      assign_public_ip = false                # 아웃바운드는 NAT
    }
    load_balancer {                           # ALB 타깃그룹 연결
      target_group_arn = var.target_group_arn
      container_name   = "my-api"
      container_port   = 8080
    }
    health_check_grace_period_seconds  = 60
    deployment_minimum_healthy_percent = 100  # 배포 중 항상 100% 유지(무중단)
    deployment_maximum_percent         = 200

    lifecycle { ignore_changes = [desired_count] }  # ← desired_count는 오토스케일 소유
  }
```

1. 오토스케일 (CPU 타깃 트래킹)

```hcl
  resource "aws_appautoscaling_target" "this" {
    min_capacity       = 2   # HA (2 AZ에 1개씩)
    max_capacity       = 6
    resource_id        = "service/${cluster}/${service}"
    scalable_dimension = "ecs:service:DesiredCount"
    service_namespace  = "ecs"
  }
  resource "aws_appautoscaling_policy" "cpu" {
    policy_type = "TargetTrackingScaling"
    target_tracking_scaling_policy_configuration {
      predefined_metric_specification { predefined_metric_type = "ECSServiceAverageCPUUtilization" }
      target_value = 60   # 평균 CPU 60% 목표 → 넘으면 태스크 자동 증설
    }
  }
```

- 배포 모델

TF가 태스크 정의를 소유 → 배포 = terraform apply -var container_image=ECR:sha. 새 이미지 → 새 태스크 정의 리비전 → 서비스 롤링. (CI가 이걸 자동 실행 — OIDC 항목에서)

- 검증

terraform plan → ECS 클러스터·서비스·태스크정의·오토스케일 포함 79 리소스, 에러 0 (실 AWS 대상, 생성 0)
