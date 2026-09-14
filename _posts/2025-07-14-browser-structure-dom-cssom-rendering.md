---
title: "브라우저 구조 부터 DOM과 CSSOM, 렌더링 과정"
date: 2025-07-14 21:05:00 +0900
categories: [browser, rendering]
tags: [browser, dom]
---

## 1. 브라우저 구조

![](/assets/img/2025-07-14/browser-structure-dom-cssom-rendering-1.png)

사용자 인터페이스 : 뒤로가기 버튼

브라우저 엔진 : 사용자 인터페이스 - 렌더링 엔진의 가교 역할 (뒤로가기 버튼 눌렀으니 페이지 보여줘)

렌더링 엔진 : 웹사이트를 보여주는 엔진

통신 : 네트워크 담당하는 엔진

자바스크립트 해석기 : 크롬 V8엔진

UI 백엔드 : 클릭 시 반응 등

## 2. DNS

naver.com입력하면 IP주소를 찾아야 함. Domain Name System

{이름 : IP주소}를 찾아옴 → DNS Cache에 저장함

## 3. 리소스 요청

IP로 찾아서 리소스 요청

**Byte String**(0101010)형태로 전달 ⇒ 인코딩 (UTF-8)

`<meta charset=”utf-8”>`

## 4. 토큰화

![](/assets/img/2025-07-14/browser-structure-dom-cssom-rendering-2.png)

- 토큰화

  919191 → `<html>`

  123123 → `<span>` `</span>` `</html>`

- Node
  - `<HTML>`

### DOM 개념

## DOM

`DOM`(Document Object Model)은 웹 문서의 구조화된 표현이다. 프로그래밍 언어가 웹 페이지 내의 객체에 접근하고 조작가능할 수 있게 하는 인터페이스로 동작한다. 주로 자바스크립트에서 웹페이지의 요소를 선택/수정/추가/제거할 때 사용된다.

### DOM의 주요 특징

- 트리 구조 : DOM은 트리 구조로 표현되며, 각 노드는 웹 페이지의 다양한 부분을 나타낸다. (요소, 속성, 텍스트 등)
- Live 특성 : DOM에 대한 변경은 즉시 웹페이지에 반영된다.
- 조작 가능 : 자바스크립트를 사용하여 DOM요소를 선택, 수정, 추가, 제거할 수 있다.

### 주요 DOM 관련 메서드 및 속성

- 요소 선택
  - document.getElementById(id) : 주어진 ID를 가진 요소를 반환한다.
  - document.getElementByTagName(name) : 주어진 태그 이름을 가진 요소를 반환한다.
  - document.getElementByClassName(name) : 주어진 클래스 이름을 가진 요소들의 목록을 반환한다.
  - document.querySelector(selector) : 주어진 CSS 선택자와 일치하는 첫 요소를 반환한다.
  - document.querySelectorAll(selector) : 주어진 CSS 선택자와 일치하는 모든 요소의 목록을 반환한다.
- 요소 조작
  - element.innerHTML : 요소의 내부 HTML을 가져오거나 설정한다.
  - element.setAttribute(name, value) : 요소의 속성 값을 설정한다.
  - element.getAttribute(name) : 요소의 속성 값을 가져온다.
  - element.appendChild(element) : 요소에 자식 요소를 추가한다.
  - element.removeChild(elemtn) : 요소에 자식 요소를 제거한다.
  - [element.getBoundingClientRect()](/posts/browser-get-bounding-client-rect/) : 요소의 크기와 뷰포트에 상대적인 위치 정보를 제공하는 DOMRect 객체를 반환한다.
- 이벤트 리스너
  - element.addEventListener(event, function) : 요소에 이벤트 리스너를 추가한다.
  - element.removeEventListener(event, function) : 요소에서 이벤트 리스너를 제거한다.

### HTML을 이용한 화면에 UI 표현하기

예) 코드

```html
<body>
	<h1>안녕하세요.</h1>
	<ul>
		<li>
      리액트
    </li>
    <li>
    가상돔
    </li>
    <li>
    엘리먼트
    </li>
  </ul>
</body>
```

예) 트리 구조

![](/assets/img/2025-07-14/browser-structure-dom-cssom-rendering-3.png)

## DOM 조작

DOM 트리를 DOM에서 제공하는 API를 통해 조작이 가능하다. API를 이용해서 DOM 구조에 접근하거나 원하는 요소를 수정/삭제가 가능하다.

```html
<-- HTML -->
<button class="button">Button</button>

// JS
var button = document.querySelector('.button');
button.onclick = function () {
	this.style.backgroundColor = "red";
}
```

### 웹페이지 렌더링 (CRP : Critical Rendering Path _ 중요한 렌더링 경로)

![](/assets/img/2025-07-14/browser-structure-dom-cssom-rendering-4.png)

CRP(중요한 렌더링 경로)는 웹브라우저가 HTML, CSS, JavaScript와 같은 웹페이지의 리소스를 받아서 실제로 화면에 페이지를 표시하는 과정이다. 이 과정은 웹 페이지의 로딩 시간과 성능에 중요한 영향을 미쳐 최적화가 중요하다.

CRP(중요한 렌더링 경로)의 목적은 가능한 웹페이지를 빠르게 렌더링하는 것이다. 아래는 전략적인 최적화 기법이다.

- CSS와 JavaScript의 최소화 및 압축
- 인라인 CSS와 JavaScript의 사용 최소화
  - 중요하지 않은 스타일이나, 스크립트를 외부 파일로 분리하여 초기 렌더링을 가속화
- 비동기 스크립트 로딩
  - async, defer 속성을 사용해서 JavaScript의 실행을 지연하면서 블로킹을 방지한다.
- 중요한 CSS를 위쪽에 위치시키기
  - 페이지 윗 부분을 빠르게 렌더링하려면 중요한 스타일을 HTML 문서의 상단에 위치시킨다.
- 웹폰트 최적화
  - 웹폰트의 로딩 지연을 방지하기 위해 필요한 스타일만 불러오고 font-display 옵션을 사용한다.

---

### 4.1 DOM tree

Document Object Model을 생성

### CSSOM 개념

## CSSOM

DOM이 HTML 문서의 구조화된 표현을 제공하는 것 처럼, CSSOM은 CSS 스타일 시트의 구조화된 표현을 제공한다.

![](/assets/img/2025-07-14/browser-structure-dom-cssom-rendering-5.png)

### CSSOM의 주요 특징

- 트리구조 : 이 트리의 각 노드는 CSS 규칙을 나타내며, 스타일 시트 내의 선택자, 속성 및 값에 해당한다.
- 동적 : DOM과 마찬가지로 CSSOM도 동적이다. JavaScript를 사용해서 실시간으로 CSSOM을 수정할 수 있고, 즉시 화면에 반영된다.
- 렌더링 블록 : 웹뷰라우저가 페이지를 렌더링할 때 CSSOM은 필수적이다.
  - 브라우저는 DOM + CSSOM을 합쳐서 렌더 트리를 생성하고, 이 트리를 기반으로 페이지 레이아웃, 페인트를 수행한다.
  - CSSOM이 완전히 생성되지 않으면  페이지 렌더링이 차단된다. → CSS 자원의 로딩과 처리는 페이지 성능에 큰 영향을 미칠 수 있다.
- JavaScript와 상호작용 : 자바스크립트를 사용해서 CSSOM에 접근하고 변경할 수 있다.

**CSSOM의 최적화**

css 최소화 및 압축 : CSS 파일의 크기를 줄여 로딩 시간을 최소화

비중요한 CSS 지연 로드 : 페이지 로딩의 초기 단계에서 필요하지 않은 CSS는 지연로드, 비동기 로드

미디어 쿼리 활용 : 미디어 쿼리를 활용하면 특정 화면/기기에 적용되는 스타일을 정의할 수 있다.

---

### 4.2 CSSOM Tree

`<link rel=”stylesheet” href=”style.css”>`

css 파일 요청

바이트 단위 → 캐릭터화 → 토큰화 → 노드 → CSSOM Tree

## 5. JS가 바닥에 있는 이유

`<script src=”main.js”></script>`

JS가 실행되는 동안은 **DOM을 멈춤**

왜냐하면 요청사항에 따라 자주 바뀌지 X

\*CSS는 위에 있는 이유 : CSS는 DOM의 생성을 막지 않음

DOM이 완성되었다고 웹사이트에 보이지 않고, DOM은 CSSOM을 기다림

CSSOM도 웹사이트의 성능에 중요함

### 5.1 defer, async

defer : JS가 DOM의 생성을 막지 않음

`<script src=”main.js” defer></script>`

`<script src=”main.js” async></script>`

## 6. Render Tree 구축

![](/assets/img/2025-07-14/browser-structure-dom-cssom-rendering-6.png)

DOM + CSSOM + JS → RenderTree에는

head, meta태그 등등 화면에 필요없는 것들은 다 제외

display : none 등

## 7. Layout

반응형 웹사이트 (50%) ⇒ 기기별 정확한 **px을** 계산하는 단계

### `Reflow` 발생

- 페이지 초기 렌더링 (초기 Layout 그릴 시)
- 윈도우 리사이즈 (Viewport 크기 변경 시)
- DOM API를 통한 노드 생성/추가
- 요소위치, 크기 변경(left, top, margin, padding, border, width, height)
- 폰트 변경
- [**스크롤**](https://github.com/tanonkim/Infinite-Scroll/pull/1)

## 8. Paint

실제로 그리는 단계

한 페이지에 다 그리지 않고, 여러개의 Layer로 그린다.

## 9. Composite

합쳐서 보여줌

**→ 7,8,9는 반복됨**

---

## 웹사이트 성능 향상

[렌더링 성능](/posts/browser-rendering-performance/)

**JS/CSS > 스타일 > 레이아웃 > 페인트 > 합성**

1. 소스 사이즈 줄이기
2. 외부에서 가져오는 리소스 줄이기 (media 쿼리 사용하기, 불필요한 파일은 나중에)
3. 외부에서 가져오는 횟수 줄이기
4. Reflow, Repaint 줄이기
   1. JS/CSS > 스타일 > 레이아웃 > 페인트 > 합성

      ![](/assets/img/2025-07-14/browser-structure-dom-cssom-rendering-7.png)

   left, top 속성, 즉 너비/높이/위치와 같은 도형을 변경하는 레이아웃 속성을 변경하면 브라우저는 다른 모든 요소를 확인하고 페이지를 다시 처리해야 한다. 영향을 받는 영역은 다시 페인트를 해야 하며, 최종적으로 페인트된 요소는 다시 합성해야 한다

5. JS/CSS > 스타일 > ~~레이아웃~~ > 페인트 > 합성

   페인트 전용 속성 : background-image, color, box-shadow와 같은 속성을 변경한 경우 레아아웃 단계가 필요없다. 레이아웃단계를 생략하면 비용이 많이 드는 레이아웃 작업을 방지할 수 있다.

6. JS/CSS > 스타일 > ~~레이아웃~~ > ~~페인트~~ > 합성

   레이아웃이나 페인트가 필요하지 않은 속성을 변경하면 바로 합성(컴포지션) 단계로 이동가능하다.
