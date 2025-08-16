---
title: 이벤트 버블링, 캡쳐링
date: 2025-08-16 12:30:00 +0900
categories: [html, event-bubbling, event-capturing]
tags: [html]
---

## HTML 이벤트 흐름

HTML 문서의 엘리먼트 태그는 태그 안에 태그가 위치하는 식으로 계층적으로 이루어지고 있다. 이러한 계층 구조 때문에 HTML요소에 이벤트가 발생할 경우 연쇄적 이벤트 흐름이 일어난다.

예) 계층 구조

```jsx
<form onclick="alert('form')">
  FORM
  <div onclick="alert('div')">
    DIV
    <p onclick="alert('p')">P</p>
  </div>
</form>
```

이런 현상을 이벤트 전파라 부르며 전파 방향에 따라 버블링, 캡쳐링으로 구분한다.

- 버블링 : 자식요소 이벤트 → 바깥 부모요소로 이벤트 전파
- 캡쳐링 : 자식요소 이벤트가 부모 요소부터 시작하여 안쪽 자식 요소까지 도달

### 이벤트 전파 흐름

1. 캡쳐링 단계 : 이벤트가 하위 요소로 전파되는 단계
2. 타깃 단계 : 이벤트가 실제 타깃 요소에 전달되는 단계
3. 버블링 단계 : 이벤트가 상위 요소로 전파되는 단계

```jsx
<html>
  <head></head>
  <body>
    <table>
      <tbody>
        <tr>
          <td>Shady Grove</td>
          <td>Aeolian</td>
        </tr>
        <tr>
          <td>Over the River, Charlie</td>
          <td>Dorian</td>
        </tr>
      </tbody>
    </table>
  </body>
</html>
```

![](/assets/img/2025-08-16/1.webp)

1. `<td>` 를 클릭하면 이벤트가 최상위 조상에서 시작해 아래로 전파된다. (캡쳐링)
2. 이벤트가 타깃 요소에 도착해 리스너를 시작한다. (타깃)
3. 그리고 다시 상위 이벤트를 전파한다 (버블링)

### 이벤트 전파가 있는 이유

![](/assets/img/2025-08-16/2.webp)

1. 논리적인 이유

   자식 요소가 부모 요소 영역 안에 위치하고 있기에 자식 요소를 클릭했다 하더라도 부모 요소도 클릭하게 된 셈이다.

2. 성능적인 이유

   이벤트 등록 코드를 줄일 수 있다.

   ```jsx
   <ul id="post-list">
     <li id="post-1">Item 1</li>
     <li id="post-2">Item 2</li>
     <li id="post-3">Item 3</li>
     <li id="post-4">Item 4</li>
     <li id="post-5">Item 5</li>
     <li id="post-6">Item 6</li>
   </ul>
   ```

   이벤트 전파의 개념이 없다면 `<li>` 요소마다 하나씩 이벤트를 등록해야 한다. 하지만 버블링의 특성을 통해 부모 요소의 ul요소에 이벤트를 등록하면 몇번째의 li요소를 등록하든 부모 요소로 이벤트가 전파되기에 원하는 구현을 더 쉽게 할 수 있다. ⇒ `이벤트 위임`이라 부른다.

## 버블링 & 캡쳐링

### 이벤트 버블링

이벤트 버블링은 한 요소에 이벤트가 발생하면 그 부모 요소의 이벤트에도 이벤트가 같이 발생되는 이벤트 전파를 말한다.

**버블링 등록**

브라우저의 이벤트는 기본적으로 버블링 방식으로 이벤트가 전파된다.

예) 자바스크립트의 `addEventListener()`

```jsx
EventTarget.addEventListener(’eventType’, function, [, useCapture])
element.addEventListener('click', (e) => {...}, false)
```

`addEventListener()` 함수로 요소의 이벤트를 등록하면 기본적으로 버블링 전파 방식으로 이벤트가 흐른다.

참고 : 거의 모든 이벤트는 버블리오딘다. 다만 focus이벤트와 같이 버블링이 되지 않는 이벤트도 존재한다.

**버블링 흐름**

예) ancestor, parent, child

```jsx
<div id="ancestor">
  <div id="parent">
    <div id="child"></div>
  </div>
</div>;

const ancestor = document.querySelector("#ancestor");
const parent = document.querySelector("#parent");
const child = document.querySelector("#child");

let count = 1;

ancestor.addEventListener("click", (e) => {
  print("ancestor");
});

parent.addEventListener("click", (e) => {
  print("parent");
});

child.addEventListener("click", (e) => {
  // e.stopPropagation()
  print("child");
});

function print(name) {
  document
    .querySelector("section")
    .insertAdjacentHTML("beforeend", `<p>${count++}. ${name} clicked</p>`);
}

document.body.addEventListener(
  "click",
  (e) => {
    [...document.querySelector("section").children].forEach((e) => {
      e.remove();
    });

    count = 1;
  },
  true,
);
```

가장 안쪽의 자식 요소 클릭 시 결과는 아래와 같다

```jsx
1. child clicked
2. paranet clicked
3. ancestor clicked
```

### 이벤트 캡쳐링

이벤트 캡쳐링은 한 요소에 이벤트가 발생하면 그 요소의 자손 요소의 이벤트도 같이 같이 발생되는 전파를 말한다.

예) 자바스크립트의 `addEventListener()`

```jsx
EventTarget.addEventListener(’eventType’, function, [, useCapture])
element.addEventListener('click', (e) => {...}, true)
```

```jsx
<div id="ancestor">
  <div id="parent">
    <div id="child"></div>
  </div>
</div>;

ancestor.addEventListener(
  "click",
  (e) => {
    console.log("ancestor clicked");
  },
  true,
); // true : Allow event capturing

parent.addEventListener(
  "click",
  (e) => {
    console.log("parent clicked");
  },
  true,
);

child.addEventListener(
  "click",
  (e) => {
    console.log("child clicked");
  },
  true,
);
```

결과는 아래와 같다

```jsx
1. ancestor clicked
2. parent clicked
3. child clicked
```

- 참고 : 버블링 + 캡쳐링 동시 설정

  ```jsx
  // 버블링 호출
  child.addEventListener("click", (e) => {
    console.log("child clicked");
  });

  // 캡처링 호출
  child.addEventListener(
    "click",
    (e) => {
      console.log("child clicked");
    },
    true,
  );
  ```

  결과 : 마치 순회하는 형태가 된다.

  ```jsx
  ancestor clicked
  parent clicked
  child clicked
  child clicked
  parent clicked
  ancestor clicked
  ```

## 이벤트 전파 제어하기

### 버블링 & 캡쳐링 문제점

자식 요소만 클릭했을때만 이벤트를 발생하고 부모 요소는 이벤트를 발생하고 싶지 않은 상황이 있을 것이다. 이 때 브라우저는 기본적으로 캡쳐링-버블링으로 동작되기 때문에 이벤트 동작 자체를 바꿀 순 없다.

그래서 이벤트 전파 방지 처리를 하는 식으로 해결해야 한다.

### 이벤트 전파 방지

1. e.stopPropagation()

   버블링 또는 캡쳐링 설정에 따라 상위, 하위로 가는 이벤트 전파를 막을 수 있다.

   ```jsx
   <html>
     <body>
       <div id="ancestor">
         <div id="parent">
           <div id="child"></div>
         </div>
       </div>

       <section></section>
     </body>
   </html>;

   ancestor.addEventListener("click", (e) => {
     e.stopPropagation(); // 이벤트 전파 방지
     print("ancestor");
   });

   parent.addEventListener("click", (e) => {
     e.stopPropagation(); // 이벤트 전파 방지
     print("parent");
   });

   child.addEventListener("click", (e) => {
     e.stopPropagation(); // 이벤트 전파 방지
     print("child");
   });
   ```

2. e.stopImmediatePropagation()

   이벤트 전파와 더불어 형제 이벤트 실행을 중지한다.

3. e.preventDefault()

   이벤트 전파 뿐만 아니라 기본 이벤트 동작 자체를 취소한다.

### 주의점

e.stopPropagation()메서드는 추후에 문제가 될 수도 있다. window 내에서 발생하는 클릭 이벤트 전부를 감지하는 시스템을 사용할 때 stopPropagation으로 버블링을 막으면 해당 영역은 죽은 영역이 되어서 분석이 제대로 되지 않을 수 있다.
