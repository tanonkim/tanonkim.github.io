---
title: "Next.js 섹션1. Next.js 소개"
date: 2025-11-07 21:39:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

Next.js : 리액트 전용의 웹개발 **프레임워크**

- 라이브러리 : 개발자가 주도권 가짐
- 프레임워크 : 자체 제공 기능 이용, 허용하는 범위 내 도구 사용 가능

리액트 : UI 개발을 위한 자바스크립트 라이브러리

[Next.js 프레임워크](/posts/nextjs-framework/)

**사전 렌더링**

- 브라우저 요청에 `사전에 렌더링이 완료된 HTML을 응답`하는 렌더링 방식
- Client Side Render의 단점을 해결

리액트 : Client Side Render

- 클라이언트(브라우저)에서 직접 화면을 렌더링
- (초기 접속 요청 이후에) **페이지 이동이 빠름**
  - JS 번들에는 모든 페이지 컴포넌트가 있기 때문
- **FCP : 초기 접속 요청 느림**
  - FCP : First Contentful Paint
    - 요청 시작부터 컨텐츠가 화면에 처음 나오는데 걸리는 시간

**구조**

![](/assets/img/2025-11-07/nextjs-section-1-1.png)

![](/assets/img/2025-11-07/nextjs-section-1-2.png)

Next.js 구조

**구조**

![](/assets/img/2025-11-07/nextjs-section-1-3.png)

FCP 순간에는 사용자와 상호작용은 불가(HTML만이니까)

![](/assets/img/2025-11-07/nextjs-section-1-4.png)

TTI : Time To Interactive

접속요청부터 상호작용까지

유저 - 브라우저 - 서버

Page Router - 초창기부터 제공

App Router - 13버전부터, Page Router의 단점 보완
