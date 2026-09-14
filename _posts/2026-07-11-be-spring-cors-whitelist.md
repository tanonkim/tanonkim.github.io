---
title: "CORS"
date: 2026-07-11 10:53:00 +0900
categories: [spring, cors]
tags: [spring, cors, terraform]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 무엇을 / 왜

프론트(Vercel)와 백엔드가 다른 도메인이라, 브라우저가 CORS(Cross-Origin) 정책으로 API 호출을 막는다. 허용할 프론트 origin 화이트리스트를 환경별로 준다.

앱 코드 — 화이트리스트 (와일드카드 금지)

```java
// CorsProperties.java — 콤마 구분 문자열을 List로 자동 바인딩
@ConfigurationProperties(prefix = "app.cors")
public record CorsProperties(List<String> allowedOrigins) {}
// SecurityConfig.java
config.setAllowedOrigins(corsProperties.allowedOrigins());  // 패턴 아님, 정확 매칭
config.setAllowCredentials(true);
```

env 주입

```java
  # application.yml — 콤마 구분 다중 origin
  allowed-origins:
  ${CORS_ALLOWED_ORIGINS:http://localhost:6500,http://localhost:6501,http://localhost:6502}
  # 운영: 실제 Vercel 도메인 (콤마 구분, 와일드카드/끝슬래시 금지)
  CORS_ALLOWED_ORIGINS=https://app.example.com,https://admin.example.com
```

Terraform 배선

```hcl
  # 리스트 변수를 콤마 문자열로 합쳐 env로 주입
  CORS_ALLOWED_ORIGINS = join(",", var.frontend_origins)
  # 예: frontend_origins = ["https://app.example.com", "https://admin.example.com"]
```

### allowCredentials=true의 제약

쿠키·인증정보를 주고받으려면 allowCredentials=true가 필요한데, 이때 CORS 스펙상 와일드카드(\*)를 못 쓴다.

→ 정확한 origin을 명시해야 함. (그래서 화이트리스트 방식)

- \* 금지
- 끝 슬래시(`https://app.example.com/`) — origin은 scheme+host+port까지만
- origin 사이 공백
