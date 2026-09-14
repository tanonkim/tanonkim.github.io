---
title: "Frontend Test 6. 커버리지 리포트 읽기"
date: 2025-05-05 15:58:00 +0900
categories: [test, frontend-test]
tags: [test, frontend-test]
---

## 커버리지 리포트 개요

커버리지 리포트 : 테스트 프레임워크에는 구현 코드가 얼마나 테스트 되었는지 측정하는 리포트의 기능이 존재

`npx jest —coverage`

예) jest의 커버리지

![](/assets/img/2025-05-05/frontend-test-6-1.png)

jest의 커버리지 리포트는 아래와 같다. 백분율로 나타낸다.

Stmts_구문 커버리지 : 구현 파일에 있는 구문이 적어도 한 번은 실행되었는지 나타낸다.

Branch_분기 커버리지 : 구현 파일에 있는 모든 조건 분기가 한 번은 실행되었는지 나타낸다. (if, case, 삼항연산자)

Funcs_함수 커버리지 : 모든 함수가 적어도 한 번은 호출됐는지 나타낸다. (export)

Lines_라인 커버리지 :

Uncovered Line_커버되지 않은 라인

---

- jest.config.ts

```typescript
  collectCoverage: true,
```

해당 옵션을 true로 한 후 open coverage/lcov-report/index.html을 실행하면 아래처럼 보인다.

![](/assets/img/2025-05-05/frontend-test-6-2.png)
