---
title: "X-Forwarded-For 헤더"
date: 2026-01-31 20:30:00 +0900
categories: [spring, security]
tags: [http, x-forwarded-for, proxy]
---

### XFF 헤더

X-Forwarded-For는 클라이언트 IP 주소를 전달하는데 사용되는 HTTP 요청의 헤더이다. 주로 Load Balancer나 Proxy Server가 클라이언트의 HTTP 요청을 처리할 때 **클라이언트의 실제 IP 주소**를 뒤에 있는 서버로 전달하기 위해 사용된다.

![](/assets/img/2026-01-31/be-security-x-forwarded-for-1.png)

### XFF 흐름

1. client가 웹사이트에 접속한다.
2. HTTP 요청이 프록시 서버나 로드밸런서를 통해 전달되어, 해당서버는 클라이언트의 실제 IP주소를 XFF 헤더에 추가한다.
3. HTTP 요청이 뒤에있는 서버로 전달될 때 서버는 XFF헤더를 읽어 사용자의 IP 주소를 파악한다.

### XFF 헤더 형식

X-Forwarded-For header는 IP주소 목록을 가진다. 이 목록은 각 프록시나 로드밸런서가 요청을 처리할 때마다 추가된다.

```java
X-Forwarded-For: <client-ip>, <proxy-ip1>, <proxy-ip2>, .....
```

– `<client-ip>`: 원래 클라이언트의 IP 주소

– `<proxy-ip1>`: 첫 번째 프록시

```java
public static String getClientIp(HttpServletRequest req) {
    String ip = req.getHeader("X-Forwarded-For");
    if (ip == null) ip = req.getRemoteAddr();
    return ip;
}
```

`X-Forwarded-For (XFF) 헤더`는 HTTP 프록시나 로드 밸런서를 통해 웹 서버에 접속하는 클라이언트의 원 IP 주소를 식별하는 표준 헤더로 쓰이고 있다. 그래서 프록시나 로드밸런스 등을 사용할 경우 Apache/Nginx에서 설정이 되어있다는 가정하에 클라이언트의 실제 접속 IP를 가져올 수 있다.

- Util 코드

```java
public static String getClientIP(HttpServletRequest request) {
    String ip = request.getHeader("X-Forwarded-For");
    logger.info("> X-FORWARDED-FOR : " + ip);

    if (ip == null) {
        ip = request.getHeader("Proxy-Client-IP");
        logger.info("> Proxy-Client-IP : " + ip);
    }
    if (ip == null) {
        ip = request.getHeader("WL-Proxy-Client-IP");
        logger.info(">  WL-Proxy-Client-IP : " + ip);
    }
    if (ip == null) {
        ip = request.getHeader("HTTP_CLIENT_IP");
        logger.info("> HTTP_CLIENT_IP : " + ip);
    }
    if (ip == null) {
        ip = request.getHeader("HTTP_X_FORWARDED_FOR");
        logger.info("> HTTP_X_FORWARDED_FOR : " + ip);
    }
    if (ip == null) {
        ip = request.getRemoteAddr();
        logger.info("> getRemoteAddr : "+ip);
    }
    logger.info("> Result : IP Address : "+ip);

    return ip;
}
```
