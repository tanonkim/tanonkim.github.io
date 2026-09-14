---
title: "React useEffect 가이드"
date: 2025-11-05 08:46:00 +0900
categories: [react, react-study]
tags: [react, hooks]
---

### 주요 질문과 답변

1. useEffect에서 데이터 페칭을 올바르게 하는 방법은? []은 무엇인가?

   `[]`는 효과가 React 데이터 흐름의 어떤 값도 사용하지 않음을 의미한다.

   실제로 값을 사용하는데 []를 사용하면 버그의 원인이 된다

   useReducer와 useCallback을 사용해 의존성 필요를 제거하는게 올바른 해결책이다.

2. 함수를 effect의존성으로 지정해야 하나?

   컴포넌트 스코프를 사용하지 않는 함수는 컴포넌트 밖으로 옮기기

   effect에서만 사용하는 함수는 효과 내부로 옮기세요

   렌더 스코프의 함수를 사용해야 한다면 useCallback으로 감싸기

3. 왜 무한 refetching 루프가 발생하나?

   의존성 배열 없이 effect에서 데이터 페칭

   의존성 배열에서 항상 변하는 값을 지정했을 때

   함수는 문제를 일으킬 수 있기에 effect내부로 옮기거나 useCallback 사용하기

## 주요 개념

### 1. 각 렌더링은 고유한 props와 state를 가진다.

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

**핵심 이해:**

- `count`는 마법같은 데이터 바인딩이 아니라 단순한 숫자이다.
- 각 렌더링에서 `count`는 상수이다.
- React가 컴포넌트를 다시 호출할 때마다 새로운 `count` 값을 받는다.

### **2. 각 렌더링은 고유한 이벤트 핸들러를 가진다 (state와 클로저)**

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  function handleAlertClick() {
    setTimeout(() => {
      alert('You clicked on: ' + count);
    }, 3000);
  }
  
  // 3초 후 alert는 클릭 시점의 count를 표시합니다
}
```

**중요 포인트:**

- 이벤트 핸들러는 자신이 속한 렌더링의 `count` 값을 "기억"한다.
- **이는 JavaScript 클로저의 동작 방식이다.**
- 클래스 컴포넌트와 달리 각 렌더링의 값은 격리되어 있다.

### **3. 각 렌더링은 고유한 Effects를 가진다 (state와 props)**

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    document.title = `You clicked ${count} times`;
  });
}
```

**effect의 동작:**

- effect 함수 자체가 매 렌더링마다 다르다.
- 각 effect는 자신이 속한 렌더링의 props와 state를 본다.
- effect는 개념적으로 렌더 결과의 일부이다.

### **4. 동기화, 라이프사이클이 아니다 (Hooks의 의존성 배열)**

- 클래스의 라이프사이클 접근법

  ```javascript
  componentDidMount() {
    this.subscribe();
  }

  componentDidUpdate(prevProps) {
    if (prevProps.id !== this.props.id) {
      this.unsubscribe(prevProps.id);
      this.subscribe();
    }
  }

  componentWillUnmount() {
    this.unsubscribe();
  }
  ```

- Hooks의 동기화 접근법:

  ```javascript
  useEffect(() => {
    const subscription = subscribe(props.id);
    return () => {
      subscription.unsubscribe();
    };
  }, [props.id]);
  ```

### 5. 의존성에 대해 거짓말하지 않기 (올바른 사용 문법)

```javascript
// ❌ 잘못된 예시
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);  // count를 사용하지만 deps에 없음
  }, 1000);
  return () => clearInterval(id);
}, []);  // count가 빠짐!

// ✅ 올바른 예시 1: 의존성 포함
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);

// ✅ 올바른 예시 2: 함수형 업데이트 [다시보기]
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);  // 이전 state에 기반한 업데이트, 현재 값에 의존하지 않음
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### 6. effect를 자급자족하게 만들기 (useReducer, 함수형 업데이트)

- 함수형 업데이트 사용

  ```javascript
  // 이전 state에 기반한 업데이트
  setCount(c => c + 1);
  ```

- `useReducer`로 업데이트와 액션 분리

  ```javascript
  const [state, dispatch] = useReducer(reducer, initialState);

  useEffect(() => {
    const id = setInterval(() => {
      dispatch({ type: 'tick' });  // 액션만 전달
    }, 1000);
    return () => clearInterval(id);
  }, [dispatch]);  // dispatch는 안정적
  ```

### 7. 함수를 effect내부로 이동하기 (문법규칙)

```javascript
// ❌ 함수가 효과 외부에 있음
function SearchResults() {
  const [query, setQuery] = useState('react');
  
  function getFetchUrl() {
    return 'https://api.example.com?query=' + query;
  }
  
  async function fetchData() {
    const result = await fetch(getFetchUrl());
    // ...
  }
  
  useEffect(() => {
    fetchData();
  }, []);  // 의존성 누락!
}

// ✅ 함수를 효과 내부로 이동
function SearchResults() {
  const [query, setQuery] = useState('react');
  
  useEffect(() => {
    function getFetchUrl() {
      return 'https://api.example.com?query=' + query;
    }
    
    async function fetchData() {
      const result = await fetch(getFetchUrl());
      // ...
    }
    
    fetchData();
  }, [query]);  // 올바른 의존성
}
```

### **8. useCallback으로 함수 안정화 (useCallback)**

```javascript
function SearchResults() {
  const [query, setQuery] = useState('react');
  
  // query가 변경될 때만 함수가 변경됨
  const getFetchUrl = useCallback(() => {
    return 'https://api.example.com?query=' + query;
  }, [query]);
  
  useEffect(() => {
    const url = getFetchUrl();
    // 데이터 페칭...
  }, [getFetchUrl]);  // 함수도 의존성에 포함
}
```

### **9. 경쟁 조건(Race Conditions) 처리 (비동기에서 취소 로직 만들기)**

```javascript
function Article({ id }) {
  const [article, setArticle] = useState(null);
  
  useEffect(() => {
    let didCancel = false;
    
    async function fetchData() {
      const article = await API.fetchArticle(id);
      if (!didCancel) {
        setArticle(article);
      }
    }
    
    fetchData();
    
    return () => {
      didCancel = true;  // 클린업 시 취소 플래그 설정
    };
  }, [id]);
}
```

**경쟁 조건을 처리하라**: 비동기 작업에서는 취소 로직을 구현하세요
