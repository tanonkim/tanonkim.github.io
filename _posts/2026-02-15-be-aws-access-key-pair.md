---
title: "AWS Access Key Pair"
date: 2026-02-15 18:54:00 +0900
categories: [aws, iam]
tags: [aws, iam, access-key]
---

### AWS 접근 방식 (보안관점)

1. 콘솔 엑세스 (AWS Console)
2. 프로그램 방식 액세스 (Command Line)

AccessKey Pair : IAM 사용자가 최대 두개까지 발급할 수 있는 영구적인 자격 증명

- 반영구적 : 삭제되지 않으면 유효기간 없이 사용
- 하드코딩되어 사용

⇒ 보안적으로 굉장히 위험 (한번 인증으로 끝)

![](/assets/img/2026-02-15/be-aws-access-key-pair-1.png)

### 임시자격증명을 활용

AWS 서비스 등에서 발급하는 유효기간이 있는 자격증명

특징 : 동적으로 생성됨(필요 시점에 생성되고, 필요에 따라 재발급됨)

- 지속기간이 정해져 있어서 로테이션이 필요없음
- IAM 유저 외에 다양한 Identity에게 자격증명 부여 가능
- 하나의 IAM 사용자에서 여러 자격 증명을 생성 가능

### AWS의 임시자격증명

AccessToken 포함 (ASIA로 시작)

생성 : IAM, WebIdentity(FaceBook 유저 등)이 특정 Role을 Assume(모자씌우기)하여 자격 생성

![](/assets/img/2026-02-15/be-aws-access-key-pair-2.png)

### 임시자격증명 활용

하드코딩 대신 필요할 때 인증을 거친 후 임시자격증명 발급

갱신이 필요하면 다시 인증 요청

활용예시

- EC2 IAM 역할 부여
- 다른 주체에게 잠시동안 권한을 빌려주는 패턴

유효기간 : 1~12시간

Assume하는 주체가 임시자격증명인 경우(Role Chaining) 무조건 최대 유효기간은 1시간
