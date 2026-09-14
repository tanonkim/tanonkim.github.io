---
title: "for await…of"
date: 2025-04-05 22:36:00 +0900
categories: [javascript, async]
tags: [javascript, async-await]
---

`for await …of` 문은 동기 이터러블 뿐만 아니라, 비동기 이터러블 객체를 순환하는 루프를 생성한다. 이 문은 비동기 함수 본문 내부와 모듈 내부를 포함하여 await를 사용할 수 있는 맥락에서만 사용할 수 있다.

> 반복문 내에서 일어나는 모든 비동기 구문을 기다려주는 구문이다.

예) 반복해서 api를 통신하는 코드

```javascript
const username = process.env.NEXT_PUBLIC_USERNAME;
const password = process.env.NEXT_PUBLIC_PASSWORD;

const names = ['user1', 'user2', 'user3', 'user4', 'user5'];

const base64 = btoa(`${username}:${password}`);

names.forEach(async (name) => {
    const response = await fetch(`http://localhost:3100/net/users/nickName?nickName=${name}`, {
        method: 'GET',
        headers: {
            'Content-Type': 'application/json',
            Authorization: `Basic ${base64}`,
        },
    });
    const data = await response.json();

    console.log(data);
});

console.log('모든 API 통신 끝!');

```

실행 결과

```shell
$ 모든 API 통신 끝!
$ <json result 'user1'...>
$ <json result 'user2'...>
$ <json result 'user3'...>
$ <json result 'user4'...>
$ <json result 'user5'...>
```

해당 예시 코드를 실행하면 다음과 같은 결과가 나온다. 

중요한 점은 **forEach에서의 모든 비동기 작업이 끝나는 것을 기다리지 않는다**는 것이다.

### for await…of

이 상황에서 문제를 `for await...of`문을 사용해서 해결할 수 있다.

```javascript
const username = process.env.NEXT_PUBLIC_USERNAME;
const password = process.env.NEXT_PUBLIC_PASSWORD;

const names = ['user1', 'user2', 'user3', 'user4', 'user5'];

const base64 = btoa(`${username}:${password}`);

// 즉시 실행 async함수 IIFE로 감쌈
(async function () {
    for await (let name of names) {
        const response = await fetch(`http://localhost:3100/net/users/nickName?nickName=${name}`, {
            method: 'GET',
            headers: {
                'Content-Type': 'application/json',
                Authorization: `Basic ${base64}`,
            },
        });
        const data = await response.json();

        console.log(data);
    }

    console.log('모든 API 통신 끝!');
})();

```

실행 결과

```javascript
$ <json result 'user1'...>
$ <json result 'user2'...>
$ <json result 'user3'...>
$ <json result 'user4'...>
$ <json result 'user5'...>
$ 모든 API 통신 끝!
```

### Promise.all()과의 차이

Promise.all()은 인자의 프로미스 배열을 **동시에** 실행한다

↔ for await…of 내의 비동기 작업은 루프를 돌며 **순차적으로** 실행된다

예) Promise.all()

```javascript
const timer = (time) => {
    return new Promise((resolve, reject) => {
        console.log(`${time} 타이머 시작`);
        setTimeout(() => {
            console.log(`${time} 타이머 끝`);
            resolve();
        }, time);
    });
};

async function runPromiseAll() {
    const times = [6000, 5000, 4000, 3000, 2000, 1000];

    await Promise.all(times.map((time) => timer(time)));

    console.log('모든 타이머 끝');
}

runPromiseAll();
```

```javascript
// 결과
6000 타이머 시작
5000 타이머 시작
4000 타이머 시작
3000 타이머 시작
2000 타이머 시작
1000 타이머 시작
1000 타이머 끝
2000 타이머 끝
3000 타이머 끝
4000 타이머 끝
5000 타이머 끝
6000 타이머 끝
모든 타이머 끝
```

예) `for await…of`

```javascript
async function runPromiseAll() {
    const times = [6000, 5000, 4000, 3000, 2000, 1000];

    // await Promise.all(times.map((time) => timer(time)));
    for await (let time of times) {
        await timer(time);
    }

    console.log('모든 타이머 끝');
}

runPromiseAll();
```

```javascript
6000 타이머 시작
6000 타이머 끝
5000 타이머 시작
5000 타이머 끝
4000 타이머 시작
4000 타이머 끝
3000 타이머 시작
3000 타이머 끝
2000 타이머 시작
2000 타이머 끝
1000 타이머 시작
1000 타이머 끝
모든 타이머 끝
```

- forEach()

```javascript
async function runPromiseAll() {
    const times = [6000, 5000, 4000, 3000, 2000, 1000];

    // await Promise.all(times.map((time) => timer(time)));
    // for await (let time of times) {
    //     await timer(time);
    // }
    times.forEach(async (time) => await timer(time));

    console.log('모든 타이머 끝');
}
```

```
6000 타이머 시작
5000 타이머 시작
4000 타이머 시작
3000 타이머 시작
2000 타이머 시작
1000 타이머 시작
모든 타이머 끝
1000 타이머 끝
2000 타이머 끝
3000 타이머 끝
4000 타이머 끝
5000 타이머 끝
6000 타이머 끝
```
