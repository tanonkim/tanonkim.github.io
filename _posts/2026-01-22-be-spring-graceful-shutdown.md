---
title: "Graceful shutdown"
date: 2026-01-22 00:24:00 +0900
categories: [spring, core]
tags: [spring, graceful-shutdown]
---

### 기본 개념

- Graceful shutdown이 진행되면 더이상 요청은 거부 한다.
- 처리 중인 요청이 있다면 마무리하고 server를 종료시킨다.

### 등장 배경

리눅스 OS에 띄워진 어플리케이션을 종료할 때 kill명령어를 사용한다.

SIGKILL -9 : OS에 바로 Kill 요청

SIGTERM -15 : 소프트웨어 종료 시그널 프로세스에 -15를 보내는 역할, 동작 방법은 프로세스에 달림

### 설정

먼저, 아래와 같이 설정 파일에 서버를 shutdown하는 방식을 graceful로 설정한다.

```yaml
server:
  shutdown: graceful // 기본 값은 immediate
```

### 동작 과정

먼저, SpringBoot가 구동될 때, shutdown 설정 값이 graceful이라면 아래와 같이 GracefulShutdown 객체를 생성해 TomcatWebServer에 할당한다.

```java
	public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
		Assert.notNull(tomcat, "Tomcat Server must not be null");
		this.tomcat = tomcat;
		this.autoStart = autoStart;
		this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL) ? new GracefulShutdown(tomcat) : null;
		initialize();
	}

```

- 종료시의 과정

생성된 Tomcat이 동작하다가, 종료된다면 shutDownGracefully이라는 메서드가 호출된다.

```java
	@Override
	public void shutDownGracefully(GracefulShutdownCallback callback) {
		if (this.gracefulShutdown == null) {
			callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
			return;
		}
		this.gracefulShutdown.shutDownGracefully(callback);
	}
```

- 완료되지 않은 요청들에 대해 대기하는 방식 과정(내부 구현 코드)

```java
	private void doShutdown(GracefulShutdownCallback callback) {
		List<Connector> connectors = getConnectors();
		connectors.forEach(this::close); // 1. connector를 close
		try {
			for (Container host : this.tomcat.getEngine().findChildren()) {
				for (Container context : host.findChildren()) {
					while (!this.aborted && isActive(context)) {
						Thread.sleep(50); // 2. 현재 처리중인 요청이 있다면 50ms씩 기다리면서 완료되지 않은 요청에 대한 확인을 한다.
					}
					if (this.aborted) {
						logger.info("Graceful shutdown aborted with one or more requests still active");
						callback.shutdownComplete(GracefulShutdownResult.REQUESTS_ACTIVE);
						return;
					}
				}
			}

		}
		catch (InterruptedException ex) {
			Thread.currentThread().interrupt();
		}
		logger.info("Graceful shutdown complete");
		callback.shutdownComplete(GracefulShutdownResult.IDLE);
	}
```

먼저, doShutdown 메서드에서 connector들을 닫음으로써 새로운 요청들을 받지 않도록 한다.

doShutdown 메서드 내부 while문에서 `isActive`라는 메서드를 통해 현재 처리중인 요청이 있는지 확인하고 있다면 루프에서 50ms씩 기다리면서 지속적으로 완료되지 않은 요청에 대한 확인을 한다.

isActive 함수의 ((StandardWrapper) wrapper).getCountAllocated()의 값을 확인해보시면 실제로 처리중인 요청의 개수가 반환된다.

### Timeout

무한루프를 돌거나 데드락으로 인해 영영 응답하지 못하는 요청들에 대해서는 어떻게 해야할까?

```bash
spring:
  lifecycle:
    timeout-per-shutdown-phase: 5s
```

위와 같이 timeout-per-shutdown-phase 옵션을 주면 shutdown 최대 대기 시간을 지정할 수 있다

```java
final class GracefulShutdown {

	// ...

	void abort() {
		this.aborted = true;
	}
}
```

Graceful shutdown이 5초가 넘어간다면 GracefulShutdown 객체의 abort 메서드를 통해 Graceful shutdown aborted값을 true로 변경하고, if (this.aborted) {} 조건문이 true가 되어 종료가 된다.

- 궁금한 점

  Graceful ShutDown이 설정되어 있지 않다면, 무중단 배포 때 문제가 없을까?
