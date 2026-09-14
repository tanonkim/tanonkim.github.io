---
title: "웹 최적화 - Bundle 사이즈 줄이기/성능최적화 (WIP)"
date: 2025-06-15 19:47:00 +0900
categories: [javascript, bundler]
tags: [javascript, bundle-size]
---

## next/bundle-analyzer를 실행하면 존재하는 파일들

### 1. client.html

✅ **클라이언트 번들 분석**

- 브라우저로 전송되는 JavaScript 번들 (ex. `main.js`, `_app.js`, 각 페이지 chunk, dynamic import chunk)
- 초기 페이지 로딩에 영향을 미치는 번들 크기를 주로 확인
- 보통 `main.js`, `_app.js`, `_document.js`, `vendor.js` 등의 이름으로 chunk가 뜸

🔍 **주요 체크 포인트**

- `main.js`: 초기 페이지 로딩에 필수로 포함되는 코드
- 동적 import chunk: dynamic import된 모듈이 여기서 분리됨
- third-party 라이브러리가 어느 chunk에 포함됐는지

---

### 2. nodejs.html

✅ **서버 사이드 렌더링 번들 분석 (Node.js 런타임)**

- Next.js가 Node.js 환경에서 페이지를 SSR할 때 사용하는 코드
- 서버 사이드 렌더링 시 필요한 Prisma Client, DB 연결, API 핸들러, middleware 등이 포함됨
- 크기가 크면 서버 cold start 시간(특히 serverless 환경)이나 빌드 시간이 길어질 수 있음

🔍 **주요 체크 포인트**

- Prisma, database client, 외부 라이브러리가 포함되는지
- lodash, moment.js, date-fns처럼 무거운 라이브러리가 포함되면 tree-shaking 여부 확인

---

### 3. edge.html

✅ **Edge Runtime 번들 분석 (Vercel Edge Functions 등)**

- Next.js 13 이상에서 Edge Runtime(Edge API Routes, Middleware, Edge Functions 등) 코드 번들을 분석
- 특히 `middleware.ts`, 일부 API Routes가 Edge Runtime으로 설정된 경우 여기로 번들링됨
- Edge Function은 크기가 클수록 cold start에 영향

🔍 **주요 체크 포인트**

- Edge middleware에서 외부 라이브러리 의존성이 많은지
- Prisma 같은 Node.js 전용 라이브러리는 Edge 번들에 포함되면 안되므로 반드시 확인

## 1. Prisma

### 1.1 문제점

![](/assets/img/2025-06-15/js-bundler-bundle-size-optimization-1.png)

**원인**

prisma generated의 번들 사이즈가 너무 지나치게 큼

import를 쉽게 하기 위해서 generated를 node_modules에 넣지 않았으나, → node_modules로 옮김

**결론**

prisma의 generate의 위치를 node_modules로 옮기기 (build 시 generate할 수 있도록 package.json에 설정 변경함)

### 1.2 node_modules로 옮긴 이후(그대로)

![](/assets/img/2025-06-15/js-bundler-bundle-size-optimization-2.png)

### 1.3 SSR Chunk로 분리

![](/assets/img/2025-06-15/js-bundler-bundle-size-optimization-3.png)

### 장점

1. 초기 번들 크기 감소

   prisma는 런타임에 사용되는 ORM이고 가볍지 않기 때문에, CSR 클라이언트 번들에 포함되면 전체 페이지 로딩 성능에 악영향을 준다.

   SSR 전용 chunk로 분리하면 클라이언트 번들에 제외되어 브라우저에서 적은 JS만 다운로드 한다.

2. 코드 분할을 통한 캐싱 최적화

   `library.js`가 여러 chunk로 분리되면, 변경이 없는 부분은 HTTP 캐시를 활용할 수 있어 재요청이 줄어든다.

   일부 쿼리 관련 모듈이 변경되어도 전체를 다시 다운받지 않아도 된다.

3. 서버사이드 전용 라이브러리와의 분리

   Prisma는 Node.js환경에서 동작하므로 클라이언트에서 실행하면 에러가 발생한다.

   이를 SSR chunk로 분리하여 클라이언트 번들에 의도치 않게 포함되는 것을 방지할 수 있다.

### 단점

1. Chunk간 의존성 증가

   library.js가 여러개로 나위면 실행 시 여러 모듈을 동적으로 로딩한다.

   SSR환경에서 해석/로드 비용을 증가할 수 있고, TTFB에 영향을 줄 수 있다.

2. 빌드 시간 증가 가능성

   Next.js는 chunk를 나누는 과정에서 각 dependency graph를 분석해야 하므로 빌드 시간이 소폭 증가할 수 있다.

   Prisma 자체가 많은 파일을 생성하므로, 이를 여러 SSR chunk로 쪼갤 경우 webpack 처리 시간이 늘어날 수 있다.

3. 트레이싱 및 디버깅 복잡도 증가

   오류가 발생했을 때 어떤 Chunk에서 문제가 생겼는지 파악이 어려워질 수 있다.

**분리 방법 (코드)**

```typescript
const getAuthPrismaClient = async () => {
  const { PrismaClient } = await require('../../../node_modules/@prisma/auth');
  return new PrismaClient();
};

const getMomentPrismaClient = async () => {
  const { PrismaClient } = await import('../../../node_modules/@prisma/moment');
  return new PrismaClient();
};

const getBoardPrismaClient = async () => {
  const { PrismaClient } = await import('../../../node_modules/@prisma/board');
  return new PrismaClient();
};

const getPaymentPrismaClient = async () => {
  const { PrismaClient } = await import('../../../node_modules/@prisma/payment');
  return new PrismaClient();
};

const getEventPrismaClient = async () => {
  const { PrismaClient } = await import('../../../node_modules/@prisma/event');
  return new PrismaClient();
};

const getVotePrismaClient = async () => {
  const { PrismaClient } = await import('../../../node_modules/@prisma/vote');
  return new PrismaClient();
};

const getScanPrismaClient = async () => {
  const { PrismaClient } = await import('../../../node_modules/@prisma/scan');
  return new PrismaClient();
};

const authPrismaClient = await getAuthPrismaClient();
const momentPrismaClient = await getMomentPrismaClient();
const boardPrismaClient = await getBoardPrismaClient();
const paymentPrismaClient = await getPaymentPrismaClient();
const eventPrismaClient = await getEventPrismaClient();
const votePrismaClient = await getVotePrismaClient();
const scanPrismaClient = await getScanPrismaClient();

export {
  authPrismaClient,
  momentPrismaClient,
  boardPrismaClient,
  paymentPrismaClient,
  eventPrismaClient,
  votePrismaClient,
  scanPrismaClient,
};
```

## 2. jsQR 등의 번들 사이즈 → dynamic import로 지연하기

⇒ 클라이언트 구성요소 및 라이브러리 지연 로딩

[Guides: Lazy Loading](https://nextjs.org/docs/app/guides/lazy-loading)

```typescript
'use client';

import { fetchQRScan } from '../api/post-qr-scan';

export const scanCamera = (scanResult, setIsResponseSuccess, setErrorMessage, webcamRef, router) => {
  const video = webcamRef.current;
  if (!video) {
    return;
  }

  try {
    // 사진 촬영
    const photoTaken = video.takePhoto();

    // 이미지 로드
    const img = new Image();
    img.src = photoTaken;

    img.onload = async () => {
      // Canvas 설정
      const canvas = document.createElement('canvas');
      const context = canvas.getContext('2d');
      canvas.width = img.width;
      canvas.height = img.height;

      // 이미지 캔버스에 그리기
      context.drawImage(img, 0, 0, canvas.width, canvas.height);

      // ImageData 가져오기
      // const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
      const imageData = context.getImageData(0, 0, canvas.width, canvas.height);

      // QR 코드 스캔
      const jsQR = (await import('jsQR')).default;
      const qrCodeResult = jsQR(imageData.data, imageData.width, imageData.height);
      if (qrCodeResult) {
        scanResult.setQRLink(qrCodeResult.data);

        const qrKey = qrCodeResult.data.split('key=')[1];

        const response = await fetchQRScan(qrKey, true);

        if (response.success) {
          setIsResponseSuccess(true);
          setTimeout(() => {
            router.replace('/member/scan/result');
          }, 1200);
        } else if (!response.success) {
          setIsResponseSuccess(false);
          setErrorMessage(response.message);
          setTimeout(() => {
            router.back();
          }, 2000);
        }
      }

      // OCR 처리
      const Tesseract = (await import('tesseract.js')).default;

      Tesseract.recognize(canvas, 'eng')
        .then(({ data }) => {
          const regex = /(AAAA.{10})|(BBBB.{10})|(CCCC.{10})|(DDDD.{10})/;
          const match = data.text.match(regex);

          if (match) {
            const extractedText = match[1] || match[2] || match[3] || match[4];

            setIsResponseSuccess(true);
            scanResult.setOCR(extractedText);

            setTimeout(() => {
              router.replace('/member/scan/ocr');
            }, 1200);
          }
        })
        .catch((err) => console.error('OCR Error:', err));
    };
  } catch (error) {
    console.error('Error capturing photo or processing image:', error);
  }
};

```
