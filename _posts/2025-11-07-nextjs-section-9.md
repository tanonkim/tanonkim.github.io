---
title: "Next.js 섹션9. 최적화와 배포"
date: 2025-11-07 21:47:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

### 9.1 이미지 최적화

평균적으로 가장 용량이 큰건 이미지임 ⇒ 이미지 최적화는 필수

다양한 이미지 최적화 기법

- webp, AVIF 차세대 형식 변환
- 디바이스 사이즈에 맞게 이미지 가져오기
- Lazy Loading
- 블러 이미지…

Next.js에선 `Image` 내장 컴포넌트 사용하기

#### Image 적용 전

![](/assets/img/2025-11-07/nextjs-section-9-1.png)

문제점

1. JPEG 사용 → webp, avif
2. 한꺼번에 이미지를 불러오고 있음
   1. 불필요한 이미지를 동시에
3. 필요한 이미지 크기보다 훨씬 크다(용량도 크다)

#### Image 적용 후

외부 서버 URL이라면 보안 때문에 차단됨

→ next.config.js 설정 필요

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
    // Url Fetch 시 log 발생
    logging: {
        fetches: {
            fullUrl: true,
        },
    },
    images: {
        domains: ['shopping-phinf.pstatic.net'],
    },
};
```

JPEG → webp

이미지 크기도 줄어듬

### 9.2 검색 엔진 최적화(SEO)

구글/네이버 등 검색엔진에서 우리 서비스가 잘 노출되도록 하는 기술

SEO 방법

- SiteMap
- RSS
- 시멘틱 태그
- 메타 데이터 설정

#### 정적인 메타 변수

```typescript
// page.tsx

export const metadata: Metadata = {
    title: '한입 북스',
    description: '한입 북스 도서를 만나보세요',
    openGraph: {
        title: '한입 북스',
        description: '한입 북스 도서를 만나보세요',
        images: ['/thumbnail.png'],
    },
};
```

#### 동적인 메타 변수

```typescript
// src/app/book/[id]/page.tsx

export async function generateMetadata({ params }: { params: { id: string } }): Promise<Metadata> {
    const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book/${params.id}`);

    if (!response.ok) {
        throw new Error(response.statusText);
    }

    const book: BookData = await response.json();

    return {
        title: `${book.title} : 한입북스 검색`,
        description: `${book.description} : 한입북스 검색결과`,
        openGraph: {
            title: `${book.title} : 한입북스 검색`,
            description: `${book.description} : 한입북스 검색결과`,
            images: [book.coverImgUrl],
        },
    };
}
```

해당 API는 중복 호출됨, 비효율적이지 않냐 :

`Request Memoization`이 사용됨

하나의 페이지를 렌더링하는 과정에서 중복된 API 방지

### 9.3 배포하기

vercel 배포

### 9.4 배포 후 최적화

![](/assets/img/2025-11-07/nextjs-section-9-2.png)
