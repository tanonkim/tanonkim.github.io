---
title: "Terraform"
date: 2026-06-21 18:24:00 +0900
categories: [infra, terraform]
tags: [terraform, iac, hcl]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## Terraform

IaC(Infrastructure as Code)란 : 인프라를 코드로

VPC, 서버, DB, 로드밸런서 같은 인프라를 콘솔 클릭이 아닌 코드로 선언, 그 코드로 적용해 실제 자원을 만드는 방식

### 사용 이유

1. 재현성(같은 코드 → 같은 인프라)
2. 버전관리(Git 이력, 리뷰, 롤백)
3. Plan으로 적용 전 미리 확인
4. 코드가 곧 문서
5. 자동화(CI/CD)

### 성질

1. 선언적
   1. 어떻게가 아닌 최종 상태만 적는다. - Terraform이 현재와 목표의 차이를 계산해 필요한 것만 바꾼다.
2. 멱등성
   1. 같은 코드를 여러번 적용해도 결과가 같다. (이미 같으면 아무것도 안함)

### **Terraform 핵심 4개념**

| 개념 | 한 줄 설명 |
|---|---|
| Provider | AWS 등과 통신하는 플러그인(hashicorp/aws) |
| Resource | 만들 자원 1개 (aws_vpc, aws_lb) |
| State | 실제 자원 ↔ 코드를 매핑한 현재 상태<br>(terraform.tfstate) |
| Module | 재사용 코드 묶음(modules/networks) |

State가 제일 중요하다. State엔 비밀번호가 들어갈 수 있어 절대 Git에 올리지 않는다.

### 작업 흐름

```javascript
terraform init       # provider, module 내려받기(디렉토리당 처음 1번)
teffarofm validate   # 문법, 정합석 검사(자격증명 불필요)
terraform plan       # 뭐가 바뀔지 미리보기
terraform apply      # 실제 적용
terraform destroy    # 전체 삭제
```

### **HCL 문법 빠르게 (코드 읽는 법)**

terraform 언어 = HCL. 5가지 블록만 알면 우리 코드가 읽힌다.

```hcl
resource "aws_vpc" "main" {           # ① resource: 자원 하나를 만든다
  cidr_block = "10.xx.xx.xx/16"
}
variable "region" { default = "ap-northeast-2" }   # ② variable: 입력값(손잡이)
output "vpc_id" { value = aws_vpc.main.id }         # ③ output: 결과로 내보낼 값
module "network" { source = "../modules/network" }  # ④ module: 재사용 코드 호출
# ⑤ locals / data : 계산값 / 외부 조회(예: 최신 AMI)
```

**참조(점 문법)**: `var.region`(변수), `module.network.vpc_id`(모듈 출력), `aws_vpc.main.id`(자원 속성).

코드에 나온 반복·조건 문법: `for_each`(맵을 돌며 여러 개 생성 — 서브넷 6개), `count`(0이면 안 만듦 — NAT 인스턴스), `조건 ? A : B`(삼항 — `nat_mode == "gateway" ? … : …`), `dynamic` 블록(블록 반복 — ALB 타깃그룹 여러 개).

### 프로젝트 구조

```hcl
terraform/
├─ modules/              # 재사용 공용 모듈 (3tier·4tier 공유)
│  ├─ network/           # VPC·서브넷·IGW·NAT·라우팅·S3 엔드포인트
│  ├─ alb/               # ALB + 타깃그룹 + 리스너
│  ├─ ecs-service/       # ECS Fargate 서비스 + 로그 + 오토스케일
│  ├─ rds/   cache/   s3/ # MySQL / Redis / 업로드 버킷
├─ 3tier/                # Public · App(Spring Boot) · DB (테스트용)
└─ 4tier/                # 퍼블릭 ALB · 내부 ALB · App · DB (테스트용)
```

### 리소스 별 만드는 것

| 계층 | 모듈 | 만드는 것 |
|---|---|---|
| 네트워크 | network | VPC, 서브넷(티어×AZ), IGW, **NAT(인스턴스 or 게이트웨이)**, 라우팅, S3 게이트웨이 엔드포인트 |
| 보안 | (루트) | **Security Group 체인** — SG를 source로 서로 참조(alb→app→data) |
| 컴퓨트 | ecs-service | **ECS Fargate**(서버리스 컨테이너) + 로그 + **CPU 오토스케일** + Spot |
| 로드밸런싱 | alb | External ALB(+HTTPS), 4tier는 Internal ALB 추가 |
| DB / 캐시 | rds / cache | **RDS MySQL**(마스터 암호 Secrets Manager) / **Redis** |
| 저장소 | s3 | 업로드 버킷(이미지·PDF) — 프라이빗·암호화·버저닝 |

**운영 VS 비용 토글**

| 토글 | 지금(비용 모드) | 운영급으로 올릴 때 | 효과 |
|---|---|---|---|
| `nat_mode` | `"instance"` (~$6) | `"gateway"` (~$43) | NAT 비용 vs HA·처리량 |
| `single_nat_gateway` | `true`(1개) | `false`(AZ별 2개) | 비용 vs AZ 이중화 |
| `db_instance_class` | `"db.t4g.small"` | 더 큰 클래스 | DB 성능 |
| `db_multi_az` | `false` | `true` | DB 자동 failover |
| `db_create_read_replica` | `false` | `true` | 읽기/쓰기 분리 |
| `redis_num_replicas` | `0` | `1+` | Redis 이중화 |
| `use_spot`(모듈) | `true` | `false` | 컴퓨트 ~70% 절감 vs 중단 안정성 |

### 사용법

```hcl
cd terraform/3tier        # 또는 4tier, 이건 프로젝트에서 실제 생성
cp terraform.tfvars.example terraform.tfvars   # app_image 등 채우기
terraform init && terraform validate && terraform plan
terraform apply           # 실제 생성
```

### Code

1. 단순한 자원 - VPC 하나

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = merge(var.tags, { Name = "${var.name}-vpc" })
}
```

- resource “aws_vpc” “this”

  타입은 aws_vpc, 이 코드 안에서 부를 이름은 this. 나중에 aws_vpc.this.id로 참조

- cidr_block = var.vpc_cidr

  값을 변수에서 받음 ⇒ 재사용 가능

1. 서브넷 6개를 한 번에 for_each

- 먼저 루트에서

```hcl
locals {
  subnets = {
    public_a = { cidr = "10.xx.xx.xx/24", az = var.azs[0], tier = "public", route = "igw" }
    public_c = { cidr = "10.xx.xx.xx/24", az = var.azs[1], tier = "public", route = "igw" }
    app_a    = { cidr = "10.xx.xx.xx/24", az = var.azs[0], tier = "app",    route = "nat" }
    app_c    = { cidr = "10.xx.xx.xx/24", az = var.azs[1], tier = "app",    route = "nat" }
    db_a     = { cidr = "10.xx.xx.xx/24", az = var.azs[0], tier = "db",     route = "none" }
    db_c     = { cidr = "10.xx.xx.xx/24", az = var.azs[1], tier = "db",     route = "none" }
  }
}
```

- 모듈에서 그 맵을 돌면서 서브넷 생성

```hcl
resource "aws_subnet" "this" {
  for_each                = var.subnets               # 맵을 돌며 6개 생성
  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value.cidr            # each.value = 맵의 값(객체)
  availability_zone       = each.value.az
  map_public_ip_on_launch = each.value.route == "igw" # 퍼블릭만 공인IP 자동
  tags = merge(var.tags, { Name = "${var.name}-${each.key}", Tier = each.value.tier })
}
```

핵심: "데이터(맵) + 반복(for_each)"로 자원을 찍어낸다. 이게 IaC의 힘.

1. 조건으로 자원 생성/생성X - NAT

```hcl
# gateway 모드일 때만 NAT Gateway 생성
resource "aws_nat_gateway" "this" {
  for_each      = var.nat_mode == "gateway" ? toset(local.nat_azs) : toset([])
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = aws_subnet.this[local.public_subnet_by_az[each.key]].id
}

# instance 모드일 때만 NAT 인스턴스(EC2) 생성
resource "aws_instance" "nat" {
  for_each          = var.nat_mode == "instance" ? toset(local.nat_azs) : toset([])
  ami               = data.aws_ami.fck_nat[0].id   # fck-nat 공개 AMI 자동 조회
  instance_type     = var.nat_instance_type        # t4g.nano
  source_dest_check = false                         # NAT는 이걸 꺼야 함(중요)
  subnet_id         = aws_subnet.this[local.public_subnet_by_az[each.key]].id
}
```

1. 라우팅도 조건 분기

```hcl
resource "aws_route" "private_nat_gw" {
  for_each       = { for az, rt in aws_route_table.private : az => rt if var.nat_mode == "gateway" }
  route_table_id = each.value.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id = aws_nat_gateway.this[...].id      # 게이트웨이로
}

resource "aws_route" "private_nat_instance" {
  for_each       = { for az, rt in aws_route_table.private : az => rt if var.nat_mode == "instance" }
  route_table_id = each.value.id
  destination_cidr_block = "0.0.0.0/0"
  network_interface_id = aws_instance.nat[...].primary_network_interface_id  # 인스턴스 ENI로
}
```

프라이빗 서브넷의 0.0.0.0/0 인터넷 전부를 NAT으로 보낸다 : 밖으로 나갈 땐 NAT을 거쳐라

1. Security Group 체인 - IP가 아니라, 앞 SG로 받기

```hcl
resource "aws_security_group" "app" {
  name   = "${var.project}-app-sg"
  vpc_id = module.network.vpc_id
}

# app은 8080을 "alb-sg에서 온 것만" 받는다 (IP가 아니라 SG로 지정!)
resource "aws_vpc_security_group_ingress_rule" "app_from_alb" {
  security_group_id            = aws_security_group.app.id
  ip_protocol                  = "tcp"
  from_port                    = 8080
  to_port                      = 8080
  referenced_security_group_id = aws_security_group.alb.id   # ← 핵심
}
```

- referenced_security_group_id = aws_security_group.alb.id : alb-sg단 자원에서 온 트래픽만 허용
  - IP를 안 쓰니 IP가 바뀌어도 규칙이 따라온다.
- SG 체인 : alb → app → data 순서로만 통과

1. 모듈 조립 - 한 모듈의 출력을 다음 모듈의 입력으로

3tier/main.tf

```hcl
module "network" {
  source   = "../modules/network"
  vpc_cidr = var.vpc_cidr
  subnets  = local.subnets
  nat_mode = var.nat_mode
}

module "alb_external" {
  source     = "../modules/alb"
  subnet_ids = module.network.subnet_ids_by_tier["public"]   # ← network의 출력을 입력으로
  ...
}
```

이 출력이 network 모듈이 이렇게 계산해서 보냄 (modules/network/main.tf)

```hcl
output "subnet_ids_by_tier" {
  value = {
    for t in distinct([for s in var.subnets : s.tier]) :
    t => [for k, s in var.subnets : aws_subnet.this[k].id if s.tier == t]
  }
}
```

- 토글이 코드에서 동작하는 방법 - RDS/Replica

```hcl
variable "multi_az"            { type = bool, default = true }
variable "create_read_replica" { type = bool, default = true }

resource "aws_db_instance" "primary" {
  multi_az = var.multi_az        # 변수값 그대로
}
resource "aws_db_instance" "replica" {
  count = var.create_read_replica ? 1 : 0   # false면 0개 = 안 만듦
  replicate_source_db = aws_db_instance.primary.identifier
}
```

1. Fargate Spot 전략

```hcl
resource "aws_ecs_service" "this" {
  launch_type = var.use_spot ? null : "FARGATE"   # Spot 쓰면 launch_type 비우고 아래로

  dynamic "capacity_provider_strategy" {
    for_each = var.use_spot ? [
      { cp = "FARGATE_SPOT", weight = var.spot_weight,     base = 0 },  # 주로 Spot
      { cp = "FARGATE",      weight = var.ondemand_weight, base = 0 },  # 일부 온디맨드(fallback)
    ] : []
    content {
      capacity_provider = capacity_provider_strategy.value.cp
      weight            = capacity_provider_strategy.value.weight
      base              = capacity_provider_strategy.value.base
    }
  }
}
```
