---
title: "Terraform (적용 기록)"
date: 2026-07-11 11:09:00 +0900
categories: [infra, terraform]
tags: [terraform, iac, s3-backend]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 무엇을 / 왜

VPC·RDS·ALB·ECS 같은 인프라를 콘솔 클릭이 아니라 코드로 선언하고 그 코드로 실제 자원을 만든다.

이점

- 재현성 : 같은 코드 → 같은 인프라
- 버전관리 : Git/리뷰/롤백
- plan으로 적용 전 미리보기
- 코드가 곧 문서
- 자동화(CI/CD)

성질

- 선언적(최종 상태만 작성 → Terraform이 알아서 차이 반영)
- 멱등성(여러번 적용해도 결과가 같음)

### 개념

| 개념 | 설명 |
|---|---|
| Provider | AWS와 통신하는 플러그인(hashicorp, aws) |
| Resource | 만들 자원 1개(aws_vpc, aws_ecs_service) |
| State | 실제 자원 ↔ 코드 매핑, password는 git 금지 |
| Module | 재사용 코드 묶음(modules, network) |

### 구조

```hcl
terraform/
├─ modules/                    # 재사용 모듈
│  ├─ network/   VPC·서브넷·NAT·라우팅·S3 엔드포인트
│  ├─ iam/       ECS 실행/태스크 역할
│  ├─ s3/        업로드 버킷 5종
│  ├─ rds/       MySQL single-AZ
│  ├─ alb/       ALB·타깃그룹·리스너
│  ├─ ecs-service/ Fargate 클러스터·서비스·오토스케일
│  └─ cicd/      OIDC·배포 Role·ECR
└─ prod/                       # 컴포지션 루트(여기서 terraform 실행)
├─ backend.tf variables.tf locals.tf main.tf outputs.tf
└─ security.tf              # SG 체인(alb→app→rds)
```

### State 백엔드 (S3 네이티브 락 — DynamoDB 안 씀)

```hcl
  terraform {
    backend "s3" {
      bucket       = "<state-bucket>"
      key          = "<env>/terraform.tfstate"
      use_lockfile = true    # ← Terraform 1.10+ S3 자체 잠금 (DynamoDB 불필요)
      encrypt      = true
    }
  }
```

lock : 여러명/CI가 동시에 apply하면 state가 깨짐 → 누가 apply중인지 표시, 예전에는 DynamoDB이었으나 지금은 S3 락파일로 대체(리소스 감축 목적)

### 핵심 패턴

① 데이터(맵) + 반복(for_each) — 서브넷 6개를 한 번에

```hcl
resource "aws_subnet" "this" {
	for_each          = var.subnets          # 맵을 돌며 6개 생성
	cidr_block        = each.value.cidr
	availability_zone = each.value.az
}
```

② SG 체인 — IP가 아니라 "앞 SG"를 참조

```hcl
resource "aws_vpc_security_group_ingress_rule" "app_from_alb" {
	security_group_id            = aws_security_group.app.id
	from_port                    = 8080
	referenced_security_group_id = aws_security_group.alb.id   # ← alb-sg에서 온 것만
}
# alb → app(8080) → rds(3306). IP가 바뀌어도 규칙이 따라옴
```

③ 모듈 조립 — 한 모듈의 출력을 다음 모듈의 입력으로

```hcl
module "alb" {
	public_subnet_ids = module.network.subnet_ids_by_tier["public"]  # network 출력→입력
}
module "ecs" {
	target_group_arn = module.alb.target_group_arn                   # alb 출력→입력
	environment = {
	SERVICE_A_DB_URL = "jdbc:mysql://${module.rds.endpoint}:3306/appdb" # rds 출력 자동 배선
	}
}
```

### 작업 흐름 & 테스트 결과

terraform init       # provider·module 다운로드<br>
terraform validate   # 문법·정합성 (자격증명 불필요) → Success ✅<br>
terraform plan       # 무엇이 바뀔지 미리보기 → Plan: 79 to add, 0 change, 0 destroy ✅<br>
terraform apply      # 실제 적용 (미실행 — 코드 단계)

plan은 실계정 대상 read-only검증, 리소스는 생성 0
