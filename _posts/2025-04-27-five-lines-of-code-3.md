---
title: "Five Lines of Code 3장. 긴 코드 조각내기"
date: 2025-04-27 19:07:00 +0900
categories: [refactoring, five-lines-of-code]
tags: [refactoring, five-lines-of-code]
---

3장 주제

- 다섯줄 제한
- 세부 사항을 보지 않고 코드 작업하기
- 메서드 추출로 긴 메서드 분해
- 호출 또는 전달, 한가지만 할 것으로 추상화 수준 맞추기
- if 문은 함수의 시작에만 배치로 if문 분리하기

DRY, KISS 지침을 따른 경우라도 코드는 쉽게 지저분해지고, 혼란스러울 수 있다. 원인은 아래와 같다.

- 메서드가 여러가지 다른 일을 수행한다.
- 낮은 수준의 원시 연산을 사용한다. (배열 조작, 산술 연산)
- 조석과 적절한 메서드와 변수명 같이 사람이 읽을 수 있는 텍스트가 부족하다.

### 3.1 왜 다섯 줄인가?

**3.1.1 규칙 : 다섯 줄 제한**

정의 : 메서드는 `{}`를 제외하고는 5줄 이상이 되어서는 안된다.

설명 : 문장이라고 하는 코드는 `if`, `for`, `while`또는 `세미콜론`으로 끝나는 모든 것을 말한다.

즉 할당, 메서드 호출, return등이 해당된다.

공백, 중괄호는 제외한다.

**스멜**

메서드가 길다는 것 자체가 스멜이다.

20줄인 하나의 메서드보다 각 5줄의 코드인 4개의 메서드가 더 이해하기 쉽다.

### 3.2 함수 분해를 위한 리팩터링 패턴 소개

다섯 줄 제한 규칙은 이해하기 쉽지만 항상 지킬 수 있는 것은 아니다.

**3.2.1 메서드 추출**

예) 초기 draw 함수

```javascript
function draw() {
		// 
		code
		
		// 맵 그리기
		code
		
		// 플레이어 그리기
		code
}
```

예) 메서드 분해

```javascript
function draw() {
	// code
	
	drawMap(g);
	drawPlayer(g);
}

function drawMap(g : type) {
	code
}

function drawPlayer(g : type) {
	code
}
```

절차

1. 추출할 줄의 주변을 빈 줄로 표시
2. 원하는 이름으로 새 메서드 생성
3. 새 메서드 호출 (그룹의 상위에)
4. 분리하고자 하는 코드를 잘라내어 메서드에 넣기 + 컴파일
5. 매개변수 오류 해결
6. 컴파일 + 오류 해결
7. 주석 있으면 제거, 사용하지 않는 줄 제

**변경 스텝**

예) 변경전

```javascript
function minimun(arr : number[][]) {
    let result = Number.POSITIVE_INFINITY;
    for (let x = 0; x < arr.length; x++) {
        for (let y = 0; y < arr[x].length; y++) {
            
            if (result > arr[x][y]) {
                result = arr[x][y];
            }
        }
    }
    return result;
}
```

예) 변경 (1/3)

```javascript
function minimun(arr : number[][]) {
    let result = Number.POSITIVE_INFINITY;
    for (let x = 0; x < arr.length; x++) {
        for (let y = 0; y < arr[x].length; y++) {
				    min();
        }
    }
    return result;
}

function min() {
	  if (result > arr[x][y]) {
	      result = arr[x][y];
	  }
}
```

예) 변경 (2/3)

```javascript
function minimun(arr : number[][]) {
    let result = Number.POSITIVE_INFINITY;
    for (let x = 0; x < arr.length; x++) {
        for (let y = 0; y < arr[x].length; y++) {
				    result = min();
        }
    }
    return result;
}

function min() {
	  if (result > arr[x][y]) {
	      result = arr[x][y];
	  }
    return result;
}
```

예) 변경 (3/3)

```javascript
function minimun(arr : number[][]) {
    let result = Number.POSITIVE_INFINITY;
    for (let x = 0; x < arr.length; x++) {
        for (let y = 0; y < arr[x].length; y++) {
				    result = min(result, arr, x, y);
        }
    }
    return result;
}

function min(result:number, arr:number[][], x:number,y:number) {
	  if (result > arr[x][y]) {
	      result = arr[x][y];
	  }
    return result;
}

```

### 3.3 추상화 수준을 맞추기 위한 함수 분해

**3.3.1 호출 또는 전달, 한 가지만 할 것**

함수 내에서 객체에 있는 `메서드를 호출`하거나, `객체를 인자로 전달`할 수 있지만 둘을 섞어 사용하지 말기

⇒ 책임이 고르지 않게 됨

예) 좋지 않은 코드

```javascript
function draw() {
  let canvas = document.getElementById("GameCanvas") as HTMLCanvasElement;
  let g = canvas.getContext("2d");

  g.clearRect(0, 0, canvas.width, canvas.height); // g의 메서드 호출
	
	drawMap(g); // 인자 전달
	drawPlayer(g); // 인자 전달
}

function drawMap(g : type) {
	code
}

function drawPlayer(g : type) {
	code
}
```

⇒ 메서드 추출을 해야 함

### 3.4 좋은 함수 이름의 속성

- 함수의 의도를 설명해야 한다.
- 함수가 하는 것을 모두 담아야 한다.
- 도메인에서 일하는 사람이 이해해야 한다.

예) 수정본

```javascript
function createGraphics() {
  let canvas = document.getElementById("GameCanvas") as HTMLCanvasElement;
  let g = canvas.getContext("2d");

  g.clearRect(0, 0, canvas.width, canvas.height); // g의 메서드 호출
  
  return g;
	
}

function draw() {
	let g = createGraphics();
	drawMap(g);
	drawPlayer(g);
}

function drawMap(g : type) {
	code
}

function drawPlayer(g : type) {
	code
}
```

**update문 메서드 추출하기**

예) 기존 코드

```typescript
function update() {
  while (inputs.length > 0) { // 메서드 분리
    let current = inputs.pop();
    if (current === Input.LEFT)
      moveHorizontal(-1);
    else if (current === Input.RIGHT)
      moveHorizontal(1);
    else if (current === Input.UP)
      moveVertical(-1);
    else if (current === Input.DOWN)
      moveVertical(1);
  }

  for (let y = map.length - 1; y >= 0; y--) { // 메서드 분리
    for (let x = 0; x < map[y].length; x++) {
      if ((map[y][x] === Tile.STONE || map[y][x] === Tile.FALLING_STONE)
        && map[y + 1][x] === Tile.AIR) {
        map[y + 1][x] = Tile.FALLING_STONE;
        map[y][x] = Tile.AIR;
      } else if ((map[y][x] === Tile.BOX || map[y][x] === Tile.FALLING_BOX)
        && map[y + 1][x] === Tile.AIR) {
        map[y + 1][x] = Tile.FALLING_BOX;
        map[y][x] = Tile.AIR;
      } else if (map[y][x] === Tile.FALLING_STONE) {
        map[y][x] = Tile.STONE;
      } else if (map[y][x] === Tile.FALLING_BOX) {
        map[y][x] = Tile.BOX;
      }
    }
  }
}
```

예) 수정 코드

```javascript
function update() {
	handleInput();
	updateMap();
}

function handleInput() {
  while (inputs.length > 0) { // 메서드 분리
    let current = inputs.pop();
    if (current === Input.LEFT)
      moveHorizontal(-1);
    else if (current === Input.RIGHT)
      moveHorizontal(1);
    else if (current === Input.UP)
      moveVertical(-1);
    else if (current === Input.DOWN)
      moveVertical(1);
  }
}

function updateMap() {
  for (let y = map.length - 1; y >= 0; y--) { // 메서드 분리
    for (let x = 0; x < map[y].length; x++) {
      if ((map[y][x] === Tile.STONE || map[y][x] === Tile.FALLING_STONE)
        && map[y + 1][x] === Tile.AIR) {
        map[y + 1][x] = Tile.FALLING_STONE;
        map[y][x] = Tile.AIR;
      } else if ((map[y][x] === Tile.BOX || map[y][x] === Tile.FALLING_BOX)
        && map[y + 1][x] === Tile.AIR) {
        map[y + 1][x] = Tile.FALLING_BOX;
        map[y][x] = Tile.AIR;
      } else if (map[y][x] === Tile.FALLING_STONE) {
        map[y][x] = Tile.STONE;
      } else if (map[y][x] === Tile.FALLING_BOX) {
        map[y][x] = Tile.BOX;
      }
    }
  }
}
```

### 3.5 너무 많은 일을 하는 함수 분리하기

**3.5.1 if문은 함수의 시작에만 배치**

**if문이 있는 경우 해당 if문은 함수의 첫 번째 항목이여야 한다.** (+ 그 이후에 아무것도 안해야 한다, 즉 유일해야 한다) //

예) 코드 스멜이 나는 기존 코드

```javascript
function updateMap() {
  for (let y = map.length - 1; y >= 0; y--) { // 메서드 분리
    for (let x = 0; x < map[y].length; x++) {
      if () {
      } else if () {
      } else if () {
      } else if () {
      }
    }
  }
}
```

예) 수정 코드

```javascript
function updateMap() {
  for (let y = map.length - 1; y >= 0; y--) { // 메서드 분리
    for (let x = 0; x < map[y].length; x++) {
				updateTile(x,y);     
    }
  }
}

function updateTile(x:number, y:number) {
 if () {} 
 else if () {} 
 else if () {} 
 else if () {}
}
```
