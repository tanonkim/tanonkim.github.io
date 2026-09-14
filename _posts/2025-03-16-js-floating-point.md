---
title: "소수점 처리 방식"
date: 2025-03-16 17:01:00 +0900
categories: [javascript, basics]
tags: [javascript, number]
---

### 처리 방식

IEEE 754 표준의 64비트 부동소수점을 사용

### 소수점 더하기 오류

![](/assets/img/2025-03-16/js-floating-point-1.png)

오류 : 10자리 밑의 저 잉여값은 어디에서 오는걸까?

### 원인

1. 컴퓨터는 **10진수를 2진수**로 계산
2. 소수 중 일부는 10진수를 2진수로 변환할 때 무한소수가 됨
3. 무한소스를 유한한 범위(**메모리 범위**)내에 표현하려면 근사치를 사용해야 함
4. 근사값으로 표시하게 되므로 미세하게 증감이 발생함

### 해결방안

**toFixed()**

`toFixed()` 메서드 사용

입력한 자리수에 맞춰 반올림

```javascript
var a = 0.1;
var b = 0.2;

var sum = (a + b).toFixed(1); 
// console.log(typeof sum) // string 
```

주의점 : toFixed()의 리턴 타입은 string

```javascript
var a = [0.1,0.2,0.3];
var sum = 0;

for(var i=0; i < a.length ; i++ ){
	sum = +(((sum + a[i])).toFixed(1));
}
```

**Math 객체 사용**

Math.ceil()

Math.floor()

Math.round()

Math.trunc()

**Number.EPSILON**

Number.EPSILON : JS에서 가장 작은 오차값을 의미
