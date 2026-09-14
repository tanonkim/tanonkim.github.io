---
title: "2. Pub/Sub 패턴"
date: 2025-04-27 22:05:00 +0900
categories: [design-pattern, observer]
tags: [design-pattern, pub-sub]
---

`Pub/Sub 패턴`이란 발행자(Publisher)가 메세지를 주제(Topic)에 발행하고, 구독자는 특정 주제를 구독해서 관련 메세지만 받는 방식이다.

핵심은 중간에 `메시지 브로커` (또는 `이벤트 버스`)가 존재한다.

### 비유상황

피자집                    === Publisher (메세지 발행자)

확성기 (주제 : 피자) === Topic(메세지가 전송되는 중간 매개체)

손님들                     === Subscriber(메세지 수신자)

피자집 : 오늘 페퍼로니 나왔어요

손님 : 나는 피자 관련된 소식만 듣고 싶어라고 구독

확성기(중개자) : 손님에게 소식 전달

- 구조

Publisher → [Topic : ‘news’, ‘pizza’, ‘sports’] → Subscribers

## 예시코드

```typescript
class PubSub {
  private topics: Record<string, Function[]> = {};

  subscribe(topic: string, listener: Function) {
    if (!this.topics[topic]) {
      this.topics[topic] = [];
    }
    this.topics[topic].push(listener);
  }

  publish(topic: string, data: any) {
    if (!this.topics[topic]) return;
    this.topics[topic].forEach((listener) => listener(data));
  }

  unsubscribe(topic: string, listener: Function) {
    if (!this.topics[topic]) return;
    this.topics[topic] = this.topics[topic].filter(fn => fn !== listener);
  }
}

// 사용 예시
const pubsub = new PubSub();

const listener = (data: string) => {
  console.log(`🍕 pizza news: ${data}`);
};

pubsub.subscribe("pizza", listener);

pubsub.publish("pizza", "새 피자 나왔어요!"); // 구독자에게 알림
```

## Pub/Sub 과 Observer 패턴 차이

| **항목** | **Observer 패턴** | **Pub/Sub 패턴** |
| --- | --- | --- |
| 연결 방식 | Subject가 직접 Observer 관리 | 브로커를 통한 간접 연결<br>예시에선 PubSub 클래스 자체 |
| 알림 방식 | 직접 알림 : `subject.notify()` | 중간 topic을 통한 알림 |
| 사용 예시 | React 상태관리, 컴포넌트간 관리 | Kafka, RabbitMQ 등 메세지 시스템, <br>WebSocket 이벤트 등 |
| 유연성 | 구조 간단하지만 유연성이 낮음 | 구조 복잡하지만 유연성이 높음 |
