---
title: "[Spring Security] Spring Security Filter Chain"
date: 2026-02-15 19:21:00 +0900
categories: [spring, security]
tags: [spring-security, filter-chain]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 요청의 흐름

> HTTP요청 → Servlet Container → DeligatingFilterProxy → FilterChainProxy → SecurityFilterChain(Filter1 → Filter2 → … → FilterN) → DispatcherServlet → Controller

Spring Security는 `서블릿 필터` 기반으로 동작한다. 사용자의 요청이 Controller에 도달하기 전에 여러 보안 필터를 거치면서 인증과 인가가 처리된다.

### 핵심 구조

DelegatingFilterProxy → FilterChainProxy → SecutiryFilterChain

**DelegatingFilterProxy**

DelegatingFilterProxy는 Servlet Container 영역에서 동작하는 서블릿 필터이다. 필터 자체는 보안처리를 하지 않고, SpringSecurityFilterChain이라는 이름의 SpringBean을 찾아 **위임**한다. Servlet Container와 Spring Container를 연결해주는 역할을 한다.

**FilterChainProxy**

FilterChainProxy가 위임을 받는 SpringBean이다. 초기화 시점에 설정 클래스로부터 필터 목록을 전달받아 보관하다가 요청이 들어오면 각 필터를 순서대로 실행한다.

### 초기화과정

애플리케이션이 시작되면 다음의 과정을 거친다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .formLogin().disable()
            .authorizeRequests()
                .antMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            .and()
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

1. SecurityFilterChain 빈을 반환하는 설정 메서드에서 HttpSecurity를 통해 어떤 필터를 사용할지 정의
2. HttpSecurity가 각 설정에 맞는 필터를 생성하여 DefaultSecurityFilterChain을 구성
3. WebSecutiry가 각 설정 클래스의 필터 목록을 모아 FilterChainProxy에 전달
4. FilterChainProxy가 Bean으로 등록되어 요청 처리 준비

### 주요 Security Filter 목록

| 필터 | 역할 |
| --- | --- |
| SecurityContextPersistenceFilter | SecurityContext를 로드/저장. 인증 상태를 세션에서 꺼내와 SecurityContextHolder에 저장 |
| LogoutFilter | 로그아웃 요청인지 확인. 맞으면 세션 무효화, 인증토큰 삭제, 쿠키 삭제 후 리다이렉트 |
| UsernamePasswordAuthenticationFilter | “form 로그인 시” username/password를 파싱하여 `UsernamePasswordAuthenticationToken` 생성 후 `AuthenticationManager`에 인증 위임 |
| ConcurrentSessionFilter | 동시 세션 제어. 세션 만료 여부를 체크 |
| BearerTokenAuthenticationFilter | Authorization 헤더의 Bearer 토큰 인증 처리 |
| BasicAuthenticationFilter | Authorization 헤더의 Basic 토큰 인증 처리 |
| RequestCacheAwareFilter | 이전 요청 정보를 캐시에서 꺼내 복원 |
| RememberMeAuthenticationFilter | 인증이 안 된 경우 RememberMe 쿠키를 검사하여 자동 인증 |
| AnonymousAuthenticationFilter | 여전히 인증이 안 됐으면 익명 사용자로 Authentication 객체 설정 (null 방지) |
| SessionManagementFilter | 세션 정책(고정 보호, 동시 세션 등)에 맞는지 검사 |
| ExceptionTranslationFilter | 이후 발생하는 인증/인가 예외를 캐치하여 적절한 응답 처리 |
| FilterSecurityInterceptor | 최종 인가 처리. 요청에 대한 접근 권한(Authorization)이 있는지 확인 |

### 주요 필터 상세

- **UsernamePasswordAuthenticationFilter**

  form 로그인의 핵심 필터. http.formLogin() 설정을 활성화하면 된다.

  request에서 username과 password를 꺼내 UsernamePasswordAuthenticationToken를 만든 뒤, AuthenticationManager.authenticate()에 넘겨 실제 인증을 수행한다. 인증 성공 시 SecurityContextHolder에 Authentication 객체가 저장된다.

- **LogoutFilter**

  requiresLogout()으로 로그아웃 요청인지 판별, CompositeLogoutHandler를 통해 LogoutHandler를 순차적으로 실행한다. LogoutSuccessHandler가 성공 후 처리를 담당한다.

- **ExceptionTranslationFilter + FilterSecurityInterceptor**

  FilterSecurityInterceptor에서 권한이 없다고 판단하면 예외를 던지고, ExceptionTranslationFilter가 이를 받아 401, 403를 구분하여 처리

### 커스텀 필터 추가하기

JWT 인증 등 커스텀 필터가 필요할 때는 `addFilterBefore()`, `addFilterAfter()`, `addFilterAt()`을 사용한다.

```java
http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
```

UsernamePasswordAuthenticationFilter 이전에 jwtAuthenticationFilter를 실행 → 토큰 기반 인증을 먼저 처리

### 디버깅

`@EnableWebSecurity(debug = true)`를 설정 클래스에 붙이면, 매 요청마다 어떤 Security Filter들이 실행되는지 콘솔에 출력되어 디버깅에 유용

---

### Security Filter Chain 사용 (실무 적용 예시)

파일: AuthorizationServerSecurityConfig.java

Order: @Order(1)

Filter Chain(SecurityFilterChain): authorizationServerSecurityFilterChain

역할: OAuth2 토큰 발급 엔드포인트

Order: @Order(2)

Filter Chain: defaultSecurityFilterChain

역할: 나머지 요청 (전체 허용)

핵심 구성:

- Custom Grant Type: PasswordWithEntryAuthenticationConverter → 패스워드 기반<br>커스텀 인증
- Provider 체인:
  - PasswordWithEntryAuthenticationProvider - 로그인 처리
  - CustomRefreshTokenAuthenticationProvider - 토큰 갱신
- OIDC 메타데이터 활성화

**전체 흐름**

[Client Request]

1. module-authentication

   /oauth2/token 토큰 발급, CustomPassword Grant, RefreshToken Grant

2. module-core

   JWT 검증, 경로별 인가처리

### 질문

1. **@Async 메서드 안에서 SecurityContextHolder.getContext()를 호출하면 어떻게 되나요? 값이 나올까요? 왜 그런 결과가 나오나요?**

   값 : 기본 설정에서는 null이 나온다 (빈 SecurityContext)

   이유 : SecurityContextHolder의 기본 전략이 `MODE_THREADLOCAL`이기 때문이다. → ThreadLocal은 현재 스레드에서만 바인딩된 저장소이다. `@Async` 가 만들어 낸 새로운 스레드에서 원래 요청 스레드의 SecurityContext에 접근할 수 없다.

   해결방법 : 애플리케이션 시작 시점에 전략을 `MODE_INHERITABLETHREADLOCAL` 로 변경하면 자식 스레드에서 SecurityContext가 전파된다.

   ```java
   SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
   ```

   주의점 : 다만 주의할 점이 있다. `스레드풀` 을 사용하는 환경에서는 InheritableThreadLocal도 문제가 될 수 있다. 스레드가 재사용되면 이전 요청의 SecurityContext가 남아있을 수 있기 때문

   이 경우 Spring이 제공하는 DelegatingSecurityContextExecutor나 DelegatingSecyrityContextAsyncTaskExecutor를 사용하여 SecurityContext를 명시적으로 전달하는 방식이 더 안전하다.

2. **Spring Security Filter Chain은 요청당 매번 실행되는데, 성능 이슈는 없나요?**

   일반적인 상황에서는 문제가 없다.

   1. 애플리케이션 초기화 시점에 한번 생성되어 **`싱글톤`**으로 재사용된다. 요청마다 새로운 필커 객체가 만들어지는게 아니다.
   2. 각 필터는 자신이 처리할 조건에 해당하지 않으면 다음 필터로 넘어간다.
      1. 예) LogoutFilter는 로그아웃 URL이 아니면 바로 통과하고, UsernamePasswordAuthenticationFilter는 로그인 요청이 아니면 바로 통과한다. 실제 무거운 로직이 실행되는 필터는 소수이다.
   3. 인증된 사용자의 경우 SecurityContextPersistenceFilter가 세션에서 SecurityContext를 꺼내온다.
   4. Filter Chain은 Servlet Filter자체의 동작 방식과 동일하다. Spring MVC DispathcerServlet도 서블릿 필터를 거치며, Security Filter만 특별히 오버헤드가 큰 것은 아니다.

   실제로 성능에 영향을 줄 수 있는 경우

   → 커스텀 필터 안에서 DB 조회나 외부 API호출 등 무거운 작업을 매 요청마다 수행하는 경우, 이 경우 캐싱 등 최적화 방법 사용하기

3. 필터는 트랜잭션 안에서 실행되나요?

   요청의 흐름

   ```java
   HTTP 요청 
   → Servlet Container → Servlet **Filter Chain** → DispatcherServlet 
   → Controller → Service(@Transactional)
   ```

   Spring의 트랜잭션은 일반적으로 @Transactional이 선언된 Service 계층에서 시작된다.

   Spring Filter Chain은 DispatcherServlet보다도 앞에서 실행되므로 트랜잭션이 아직 시작되기 전이다.

   (Servlet Filter Chain 안에 하나의 필터로 Security Filter Chain + 일반 Filter Chain이 존재)
