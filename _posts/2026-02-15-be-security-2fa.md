---
title: "2FA(2단계 인증)"
date: 2026-02-15 18:16:00 +0900
categories: [spring, security]
tags: [authentication, 2fa, otp]
---

### 2FA란

사용자가 로그인할 때 두 가지 서로 다른 인증 요소를 요구하는 보안 방식이다.

인증의 3가지 요소:

- 지식(Something you know) — 비밀번호, PIN
- 소유(Something you have) — 휴대폰, 하드웨어 키
- 생체(Something you are) — 지문, 얼굴 인식

2FA는 이 중 2가지를 조합한다. 가장 흔한 조합은 비밀번호(지식) + OTP(소유)이다.

### 일반적인 2FA 흐름

```java
1. 사용자가 아이디/비밀번호 입력
2. 서버가 1차 인증 성공 확인
3. 서버가 OTP(일회용 코드)를 생성
4. OTP를 SMS/이메일/앱으로 전송
5. 사용자가 OTP 입력
6. 서버가 OTP 검증 → 최종 로그인 완료
```

여기서 핵심은 1차 인증과 2차 인증 사이의 "중간 상태"를 어떻게 관리하느냐이다.

### Spring JPA 관점에서의 설계

**1. 엔티티 설계**

User 엔티티에 2FA 관련 필드 추가:

| 필드 | 역할 |
| --- | --- |
| `twoFactorEnabled` | 2FA 활성화 여부 (boolean) |
| `twoFactorSecret` | TOTP 시크릿 키 (TOTP 방식 사용 시) |
| `phoneNumber` | SMS 발송 대상 (Twilio 사용 시) |

OTP 관리용 엔티티 (SMS/이메일 방식):

| 필드 | 역할 |
| --- | --- |
| `code` | 생성된 OTP 코드 |
| `userId` | 대상 사용자 |
| `expiresAt` | 만료 시간 (보통 3~5분) |
| `verified` | 사용 여부 (재사용 방지) |

1. **두 가지 주요 방식**

**방식 A: SMS/이메일 OTP**

- 서버가 랜덤 6자리 코드 생성 → DB에 저장 → Twilio 등 SMS 발송
- 사용자가 입력한 코드와 DB의 코드를 비교
- **장점:** 구현이 직관적
- **단점:** SMS 비용, 전송 지연, SIM 스와핑 공격에 취약

**방식 B: TOTP (Time-based One-Time Password)**

- Google Authenticator, Authy 등의 앱 사용
- 서버와 클라이언트가 **같은 시크릿 키**를 공유
- 30초마다 시간 기반으로 동일한 코드가 생성됨 (RFC 6238)
- DB에는 **시크릿 키만 저장**, 코드 자체는 저장하지 않음
- **장점:** 네트워크 불필요, 비용 없음, 더 안전
- **단점:** 사용자가 앱을 설치해야 함

구현 전략 옵션:

- 세션 기반: 세션에 `TWO_FA_PENDING` 플래그 저장 → OTP 검증 후 제거
- JWT 기반: 1차 인증 후 제한된 권한의 임시 토큰 발급 → OTP 검증 후 전체 권한 토큰 재발급
- DB 기반: 별도의 `AuthenticationSession` 엔티티로 상태 추적
