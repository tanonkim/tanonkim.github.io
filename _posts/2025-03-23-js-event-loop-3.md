---
title: "Javascript 이벤트 루프3"
date: 2025-03-23 20:12:00 +0900
categories: [javascript, async]
tags: [javascript, event-loop]
---

### MicroTask Queue의 무한루프

MacroTask Queue와 MicroTask Queue의 차이점은 우선순위이지만, 브라우저 동작 관점에서 보면 굉장히 중요한 요소이다.

MicroTask Queue의 콜백은 가장 높은 우선순위로 처리되지만, **브라우저가 화면을 렌더링하는 과정보다 더 먼저** 처리된다. 이 때 만약 MicroTask Queue에 잘못된 무한루프 동작의 버그의 코드가 들어가면 웹페이지는 먹통이 되어버리는 현상이 발생할 수 있다.

예1) 버튼을 클릭하면 콘솔에 Clicked를 출력하는 이벤트 핸들러 등록

```javascript
<button id="btn">Click me</button>

<script>
  document.getElementById("btn").addEventListener("click", () => {
    document.body.insertAdjacentHTML('beforeend', "<p>Clicked</p>");
  });
</script>
```

예2) setTimeout 무한루프 적용

```javascript
<button id="btn">Click me</button>
<button id="btn2">무한루프 시작</button>

<div id="loopArea" style="border: 1px solid black; max-height: 150px; overflow-y: auto;"></div>

<script>
  document.getElementById("btn").addEventListener("click", () => {
    document.body.insertAdjacentHTML('beforeend', "<p>Clicked</p>");
  });

  document.getElementById("btn2").addEventListener("click", () => {
    // setTimeout 함수는 Task Queue에 콜백을 추가
    function loop() {
      setTimeout(() => {
        document.getElementById("loopArea").append("Loop\n");
        loop(); // loop 함수가 setTimeout 객체의 콜백으로 자신을 계속해서 추가
      }, 0);
    }

    // loop 함수를 호출
    loop();
  });
</script>
```

이 코드에서 setTimeout 무한루프를 적용해, 버튼을 누르면 타이머가 계속 돌아가 ‘Loop’콘솔을 출력할 것이다. 

예3) Promise 객체를 이용한 재귀함수 호출

```html
<button id="btn">Click me</button>
<button id="btn2">무한루프 시작</button>

<div id="loopArea" style="border: 1px solid black; max-height: 150px; overflow-y: auto"></div>

<script>
    document.getElementById('btn').addEventListener('click', () => {
        document.body.insertAdjacentHTML('beforeend', '<p>Clicked</p>');
    });

    document.getElementById('btn2').addEventListener('click', () => {
        // Promise 함수는 MicroTask Queue에 콜백을 추가
        function loop() {
            Promise.resolve().then(() => {
                document.getElementById('loopArea').append('Loop\n');
                loop(); // 해당 함수가 Promise의 객체로 자신을 계속 추가
            });
        }
        // loop 함수를 호출
        loop();
    });
</script>
```

해당 예시는 Promise 객체를 이용해 재귀함수 호출로 무한 루프를 구현한 것이다. 코드는 setTimeout예시와 별다른 차이점이 없지만, MacroTask Queue가 아닌 MicroTask Queue로 들어가는 차이점이 있다. 

버튼을 실행하면 MicroTask Queue가 무한루프에 빠지고, 웹사이트가 먹통이 된다. 똑같은 무한루프 동작인데, setTimeout은 정상 작동하고 Promise객체는 동작하지 않는다. 왜냐하면 MicroTask Queue가 비어있지 않으므로, 다른 MacroTask나 이벤트 콜백이 실행될 수 없기 때문이다. 무한루프 코드로 끊임없이 MicroTask Queue에 콜백이 적재되고, 이벤트루프가 브라우저 화면 렌더링을 하기 전에 우선으로 끊임없이 처리하기 때문에, 결국 사용자는 이벤트에 반응하지 못하고 페이지가 멈추거나 응답하지 않는다.

결론 : MicroTask Queue를 사용하는 Promise 코드나 Mutation Observer의 콜백에서 지나치게 MicroTask Queue에 콜백을 적재하는 작업이 있다면 이를 조심히 다루어야 한다.
