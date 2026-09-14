---
title: "AWS 네트워크 보안 점검"
date: 2026-09-07 17:24:00 +0900
categories: [aws, security]
tags: [aws, waf, security-group]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 요약

트래픽 급증 : 인터넷 전역 스캔이었고 WAF가 잘 막고 있었다. 문제는 공격이 아닌, WAF를 거치지 않고 서버로 바로 들어가는 문이 있었고, 이를 해결

### 구조

```yaml
[인터넷]
   │
Route53 (api / worker .example.com)
   │
   ▼
ALB (my-admin-alb, my-main-alb)  ←── AWS WAF 가 여기 붙는다
   │  :443 HTTPS, :3000 평문 HTTP
   ▼
EC2 (ASG, 100% Spot) :3000  ←── 앱(Node/Fastify)이 듣는 포트
```

| **장치** | **계층** | **못하는 일** |
|---|---|---|
| ASG(보안그룹) | L3/L4 | HTTP 내용을 못 본다 |
| ALB | L7 | 그 자체로 보안 검사를 하지 않는다. |
| WAF | L7 | ALB를 거치지 않으면 아무것도 못한다. |

### 무엇이 들어왔나?

64시간 WAF 로그 약 81만건 분석

- 총 810,199건 / 차단 52,234건(6.4%)
- **트래픽 양은 평소보다 오히려 적었다** (main 744k vs 지난주 1.09M)
- **급증한 건 4xx뿐** — main 10배, admin 57~97배

### 정체

**(A) IP·ec2 도메인 직격 스캐너 — 29,496건 / 697개 IP**

Host헤더가 도메인이 아닌, 내부 ALB 고정 EIP거나 역방향 DNS

```yaml
ec2-x-x-x-x.ap-northeast-2.compute.amazonaws.com  →  x.x.x.x (= admin ALB EIP)
```

**ec2 도메인의 정체**: AWS는 모든 공인 IP에 `ec2-<IP>.<리전>.compute.amazonaws.com` 형태의 PTR(역방향 DNS)을 자동으로 붙인다. 공격자가 우리 도메인을 몰라도 IP 대역만 훑으면 이 이름이 나온다. **표적 공격이 아니라 인터넷 전역 스캔**

노린 것: /.env*(12종 변형), /.git/config, /.aws/credentials, /actuator/env, /wp-config.php.bak, POST /fetch·/proxy(SSRF 탐침). 최대 단일 공격자는 15,212건 / 서로 다른 URI 15,078개(전량 차단).

**(B) 스크래퍼 — 25,156건, 전량 차단** UA `node`, Vercel(AWS us-east-1) IP 로테이션. 기존 룰이 100% 잡고 있었다. → 사고없이 오류로.

⇒ WAF가 잘 대응하고 있다.

### 진단

1. RequestCount는 전체 요청수가 아니다

```yaml
RequestCount        2,065
HTTPCode_ELB_4XX   21,770   ← 요청보다 4xx가 10배 많다?
```

모순처럼 보이지만 정상

AWS 문서 정의 : RequestCount: **로드밸런서가 타깃을 고를 수 있었던** 요청 수

1. 로그가 없으면 흔적 없음이 아닌 볼 수 없음

### WAF 우회 - 진짜 문제

보안그룹의 기존 상태 (main, admin ASG 공유)

```yaml
tcp/22    0.0.0.0/0     tcp/3000  0.0.0.0/0   ← 앱 포트!
tcp/80    0.0.0.0/0     tcp/6600  0.0.0.0/0
tcp/443   0.0.0.0/0
```

실제로 밖에서 이렇게 응답했다.

```yaml
$ curl http://ec2-x-x-x-x.ap-northeast-2.compute.amazonaws.com:3000/health
200 {"service":"my-admin-server","dependencies":{"mongodb":"connected"}}
```

### 왜 위험한가

WAF는 ALB에 붙는 부가 기능이다. 요청이 ALB에 도착해야 검사가 시작. **인스턴스 IP로 바로 들어오면 ALB를 거치지 않으므로 WAF는 그 요청의 존재를 모른다.**

WAF를 달았는데, 왜 공격이 보이냐의 답.

### 고친 방법

`:3000`의 출처를 **ALB 보안그룹으로만** 제한

```yaml
# ① ALB 출처 허용을 "먼저" 추가
aws ec2 authorize-security-group-ingress --group-id sg-xxx \
  --ip-permissions '[{"IpProtocol":"tcp","FromPort":3000,"ToPort":3000,
                      "UserIdGroupPairs":[{"GroupId":"sg-xxx"}]}]'
# ② 타깃 healthy 확인
# ③ 그 다음에 0.0.0.0/0 회수
aws ec2 revoke-security-group-ingress --group-id sg-xxx \
  --security-group-rule-ids sgr-xxx
```

이어서 아무것도 안 듣던 80·443·6600을 회수하고, 공인 22도 닫았다. 22는 닫기 전에 **대체 경로를 먼저 확인**했다 — `user-data`가 `tailscale up --ssh`를 쓰고 SSM 에이전트도 세 인스턴스 모두 Online이었다.

```yaml
before:  http://x.x.x.x:3000/health   → 200
after :  http://x.x.x.x:3000/health   → 000 (차단)
         https://worker.example.com → 200 (정상)
```

### 또 다른 문제

main-server가 admin-server를 이렇게 부르고 있었다. 64시간에 856번.

```
POST http://my-admin-alb-1234567890.ap-northeast-2.elb.amazonaws.com:3000/api/v1/items/sync
headers: { "X-API-KEY": <비밀키> }
```

이걸 받기 위해 ALB에 평문 `:3000` 리스너가 열려 있었다.

**왜 위험한가**

http라 API키가 암호화 없이 나간다. + VPC 내부 통신이 아님. WAF 로그의 발신 IP가 사설 IP가 아니라 공인 IP로 찍혔다. 패킷이 VPC를 나갔다가 ALB 공인 IP로 되돌아온 것이다.

왜 평문이었나(AWS 설계/구조적 이유)

```
https://my-admin-alb-12...   검증 켬 → 실패 (이름이 인증서와 불일치) 
http://my-admin-alb-12...:3000 → 200
```

axios 기본값이 **인증서 검증 확인**이므로, **ALB DNS 이름을 쓰는 한 평문 말고는 길이 없었다.** 평문 `:3000` 리스너가 존재한 이유가 이것 하나였다.

**고친 방법**

환경변수 변경

```
- ADMIN_SERVER_BASE_URL=http://my-admin-alb-12...elb.amazonaws.com:3000
+ ADMIN_SERVER_BASE_URL=https://api.example.com
```

이 한 줄로 세 가지가 동시에 풀린다 — TLS 정상화, Host 화이트리스트에서 ALB 이름 예외 불필요, 평문 `:3000` 리스너 제거 가능.

### Host 화이트리스트

**기존상태**

WAF 룰이 7개 존재하나 Host 헤더를 보는 룰이 없었다. → 차단이 전부 누가 왔나(IP 평판) 기준이었고, 어디로 왔나 기준은 X

**왜 위험한가**<br>IP 평한 목록은 후행 지표. (예시 IP)

```
x.x.x.x     → ip-reputation 에 걸림 → 차단 ❌
y.y.y.y     → 목록에 없음        → 통과 ✅ 앱까지 도달
```

Rate-Limit도 막지 못함. `rate-main` `<N>`/분·`rate-admin` `<N>`/분인데 키가 IP라, 분산 스캐너는 IP당 51~640건이라 **절대 안 걸린다**(이 창에서 rate 룰 발동 0회).

⇒ 스캐너 2574건이 앱까지 도달.

**어떻게 고쳤냐**

Host가 WhiteList에 없으면 차단하는 룰 신설. 앱까지 도달한 2574건이 **전부** IP/ec2 Host였으므로 이 룰 하나로 0이 된다.

(Count로 50분 관찰 → 매칭 251건 전부 스캐너, 정상 도메인 0건 → Block 승격.)

```
ALB DNS 이름 Host → 403     api.example.com    → 200
raw IP Host       → 403     worker.example.com → 200
```
