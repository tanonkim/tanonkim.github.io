---
title: "React Hooks 가이드 (패턴, 함정, 실용적인 활용법)"
date: 2025-11-10 16:08:00 +0900
categories: [react, react-study]
tags: [react, hooks]
---

### 1. 서론

React Hook은 React의 가장 중요한 기능 중 하나로, 단순히 API가 아니라 **컴포넌트를 바라보는 새로운 사고방식**입니다.

**Hook이 해결하는 문제들:**

- 상태와 로직을 깔끔하게 정리
- Prop drilling 악몽 방지
- 복사-붙여넣기 없이 코드 재사용

### 2. **Hook이란 정말 무엇인가?**

Hook은 **컴포넌트 코드에서 직접** React 기능을 활용할 수 있게 해주는 특별한 함수이다.

**Hook 사용 시 React와의 대화**

- "상태가 필요해" → `useState`
- "렌더링 후 작업이 필요해" → `useEffect`
- "재렌더링 없이 값을 기억하고 싶어" → `useRef`

### 함정 #1: 무한 useEffect 루프

**문제:**

```javascript
useEffect(() => {
  setCount(count + 1); *// 무한 루프!*
}, [count]); *// count가 의존성이면서 업데이트됨*`
```

**해결책**: Effect 내부에서 state 업데이트 시 조건 로직 추가 또는 구조 재검토

### 함정 #4: 과도한 메모이제이션

모든 것을 "혹시 몰라서" 메모이제이션하면:

- 코드 가독성 저하
- 메모이제이션 자체의 비용으로 성능 악화 가능

**메모이제이션이 필요한 경우:**

- 계산이 비싼 경우
- 메모이제이션된 자식 컴포넌트에 함수/prop 전달 시

### 함정 #5: 커스텀 Hook에서 Hook 순서 깨기

커스텀 Hook 내부에서도 조건부 Hook 호출 금지!

### 함정 #6: State Setter가 비동기임을 잊음

**문제:**

```javascript
setCount(count + 1);
setCount(count + 1); // 여전히 +1만 증가
```

**해결책:**

```javascript
setCount(c => c + 1);
setCount(c => c + 1); *// 이제 +2 증가*`
```
