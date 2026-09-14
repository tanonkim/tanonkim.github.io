---
title: "Next.js 섹션7. 서버 액션"
date: 2025-11-07 21:45:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

### 7.1 서버 액션 소개

`서버액션` : 브라우저에서 호출할 수 있는 서버에서 실행되는 비동기 함수 ⇒ 별도의 API 없이 요청 가능

![](/assets/img/2025-11-07/nextjs-section-7-1.png)

`form 양식` 제출 시 FormData 형식으로 NextServer에서 비동기 실행이 가능

![](/assets/img/2025-11-07/nextjs-section-7-2.png)

- 브라우저에 Next-Action이라는 해시 값까지 명시가 됨
  - api가 생성이 되는 것!
  - Headers, Payload 참고

![](/assets/img/2025-11-07/nextjs-section-7-3.png)

![](/assets/img/2025-11-07/nextjs-section-7-4.png)

#### 서버 액션 사용 이유

1. 코드가 매우 간결해짐
   1. 간단한 경우 api 역할 가능
2. 보안상으로 민감, 중요한 데이터 사용할 때 유용
   1. 브라우저에서 작동하지 않음(호출만 가능)

### 7.2 리뷰 추가 기능 구현하기

server-action 코드 분리 : src/actions/create-review.actions.ts

bookId는 input으로 부터 받도록 하기 (input에 hidden을 사용)

```typescript
function ReviewEditor({ bookId }: { bookId: string }) {
    return (
        <section>
            <form action={≈}>
                <input name="content" placeholder="리뷰내용" required />
                <input name="author" placeholder="작성자" required />
                <input name="bookId" value={bookId} hidden readOnly />
                <button type="submit">작성하기</button>
            </form>
        </section>
    );
}
```

```typescript
'use server';

export async function createReviewAction(formData: FormData) {
    // console.log('server action called'); // api가 생성됨, 브라우저에서 api 자동호출
    // console.log(formData);

    const bookId = formData.get('bookId')?.toString();
    const content = formData.get('content')?.toString();
    const author = formData.get('author')?.toString();

    if (!content || !author || !bookId) {
        // 서버 예외처리 + 클라이언트 예외처리 (required)
        return;
    }

    try {
		    // API 접근 or Prisma 접근하기! (DB 직접 접근이 좀 더 의미에 맞는 듯)
        const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/review`, {
            method: 'POST',
            body: JSON.stringify({ // 직렬화
                bookId,
                content,
                author,
            }),
        });

        console.log(response.status);
    } catch (error) {
        console.error(error);
        return;
    }
}

```

### 7.3 리뷰 조회 기능 구현하기

코드 컴포넌트 분리

- 문법 (Spread, …review 참고)

```javascript
async function ReviewList({ bookId }: { bookId: string }) {
    const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/review/book/${bookId}`);
    if (!response.ok) {
        throw new Error(`Review fetch Failed : ${response.statusText}`); // error.tsx로 넘어감
    }
    const reviews: ReviewData[] = await response.json();
    return (
        <section>
            {reviews.map((review) => (
                <ReviewItem key={`review-items-${review.id}`} {...review} />
            ))}
        </section>
    );
}

import { ReviewData } from '@/types';
import style from './review-item.module.css';
export default function ReviewItem({ id, content, author, createdAt, bookId }: ReviewData) {
    return (
        <div className={style.container}>
            <div className={style.author}>{author}</div>
            <div className={style.content}>{content}</div>
            <div className={style.bottom_container}>
                <div className={style.date}>{new Date(createdAt).toLocaleString()}</div>
                <div className={style.delete_btn}>삭제하기</div>
            </div>
        </div>
    );
}
```

### 7.4 리뷰 재검증 구현하기

server action이 실행 → 리뷰 리스트가 업데이트 되어야 함

`revalidatePath` :

1. **서버에서만 호출이 가능함**
2. 경로를 재검증함(데이터 캐시도 무효화함, 데이터 캐시 삭제, force-cache라도 삭제됨)

3. 페이지 캐싱인 풀라우트 캐시도 같이 무효화됨, 다시 페이지 생성해서 캐시로 저장하지 않음
   1. Dynamic 페이지처럼 생성됨
      1. .next/server 에서 확인
   2. book/1로 들어가면 느려질 수도 있음(다이나믹 페이지처럼 행동됨)

   ![](/assets/img/2025-11-07/nextjs-section-7-5.png)

```typescript
export async function createReviewAction(formData: FormData) {
    // console.log('server action called'); // api가 생성됨, 브라우저에서 api 자동호출
    // console.log(formData);

    const bookId = formData.get('bookId')?.toString();
    const content = formData.get('content')?.toString();
    const author = formData.get('author')?.toString();

    if (!content || !author || !bookId) {
        // 서버 예외처리 + 클라이언트 예외처리 (required)
        return;
    }

    try {
        const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/review`, {
            method: 'POST',
            body: JSON.stringify({
                bookId,
                content,
                author,
            }),
        });

        console.log(response.status);
				revalidatePath(`/book/${bookId}`); // 재검증 (다시 생성)
    } catch (error) {
        console.error(error);
        return;
    }
}

// 렌더링되는 컴포넌트
export default function Page({ params }: { params: { id: string } }) {
    return (
        <div className={style.container}>
            <BookDetail bookId={params.id} />    // 1
            <ReviewEditor bookId={params.id} />  // 2
            <ReviewList bookId={params.id} />    // 3
        </div>
    );
}
```

### 7.5 다양한 재검증 방식 살펴보기

`revalidateTag`

- 타 API 캐시는 삭제하지 않음 ⇒ 경제적임!

#### 재검증방법 5가지

```typescript

// 1. 특정 주소에 해당하는 페이지만 재검증 (다시 생성)
revalidatePath(`/book/${bookId}`);

// 2. 특정 경로의 모든 동적 페이지 재검증, 예) book/1, book/2, book/3
revalidatePath('/book/[id]', 'page');

// 3. 특정 레이아웃을 갖는 모든 페이지 검증
revalidatePath('/(with-searchbar)', 'layout');

// 4. 모든 데이터 재검증
revalidatePath('/', 'layout'); // rootLayout 가지는 페이지 재검증 : 모든 페이지 재검증

```

```typescript
const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/review/book/${bookId}`, {
		next: { tags: [`review-${bookId}`] },
});
    
  // 5. 태그 기준, 데이터 캐시 재검증
  revalidateTag(`review-${bookId}`); // tag : 데이터 fetch 시 직접 설정
```

### 7.6 클라이언트 컴포넌트에서의 서버 액션

useActionState

```typescript
'use client';

import { useActionState, useEffect } from 'react'; // React 19
import style from './review-editor.module.css';
import { createReviewAction } from '@/actions/create-review.action';

export default function ReviewEditor({ bookId }: { bookId: string }) {
    const [state, formAction, isPending] = useActionState(createReviewAction, null); // useActionState(action 함수, 상테 초기값)

    useEffect(() => {
        if (state && !state.status) {
            alert(state.error);
        }
    }, [state]);

    return (
        <section>
            <form className={style.form_container} action={formAction}>
                <input disabled={isPending} name="bookId" value={bookId} hidden />
                <textarea disabled={isPending} required name="content" placeholder="리뷰 내용" />
                <div className={style.submit_container}>
                    <input required name="author" placeholder="작성자" />
                    <button disabled={isPending} type="submit">
                        {isPending ? '제출 중 ...' : '작성하기'}
                    </button>
                </div>
            </form>
        </section>
    );
}

```

```typescript

// useActionState를 사용하면 첫번째 인자로 state 값을 받아야 함 (status, formData)
export async function createReviewAction(_: any, formData: FormData) {
    // console.log('server action called'); // api가 생성됨, 브라우저에서 api 자동호출
    // console.log(formData);

    const bookId = formData.get('bookId')?.toString();
    const content = formData.get('content')?.toString();
    const author = formData.get('author')?.toString();

    if (!content || !author || !bookId) {
        // 서버 예외처리 + 클라이언트 예외처리 (required)
        return {
            status: false,
            error: '리뷰내용과 작성자를 입력해주세요',
        };
    }

```

사용방법

1. useActionState 작성

   const [state, formAction, isPending]  // 상태, action되는 값, pending중인지

   = useActionState(createReviewAction, null); // action 함수, 초기값

2. action함수에 state 생성하기
   1. return 값 생성하기
   2. state 타입 붙이기

   ```javascript
   export async function createReviewAction(_: any, formData: FormData) {
       // console.log('server action called'); // api가 생성됨, 브라우저에서 api 자동호출
       // console.log(formData);

       const bookId = formData.get('bookId')?.toString();
       const content = formData.get('content')?.toString();
       const author = formData.get('author')?.toString();

       if (!content || !author || !bookId) {
           // 서버 예외처리 + 클라이언트 예외처리 (required)
           return {
               status: false,
               error: '리뷰내용과 작성자를 입력해주세요',
           };
       }
   ```

### 7.7 리뷰 삭제 기능 구현하기

`submit` : 타입 검사 X 등 강제로 폼 제출

`requestSubmit` : 사용자가 submit 버튼 누른거랑 동일하게 동작

```typescript
'use server';

import { delay } from '@/util/delay';
import { revalidateTag } from 'next/cache';

export async function deleteReviewAction(_: any, formData: FormData) {
    const reviewId = formData.get('reviewId')?.toString();
    const bookId = formData.get('bookId')?.toString();

    if (!reviewId || !bookId) {
        // 서버 예외처리 + 클라이언트 예외처리 (required)
        return {
            status: false,
            error: '삭제할 리뷰가 없습니다.',
        };
    }

    try {
        await delay(1000);
        const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/review/${reviewId}`, {
            method: 'DELETE',
        });
        if (!response.ok) {
            throw new Error(response.statusText);
        }

        console.log(response.status);

        // 5. 태그 기준, 데이터 캐시 재검증
        revalidateTag(`review-${bookId}`); // tag : 데이터 fetch 시 직접 설정
        return {
            status: true,
            error: '',
        };
    } catch (error) {
        // console.error(error);
        return {
            status: false,
            error: `리뷰 삭제에 실패했습니다. : ${error}`,
        };
    }
}

```

```typescript
'use client';

import { deleteReviewAction } from '@/actions/delete-review.action';
import { useActionState, useEffect, useRef } from 'react';

export default function ReviweItemDelete({
    reviewId,
    bookId,
}: {
    reviewId: number;
    bookId: number;
}) {
    const formRef = useRef<HTMLFormElement>(null);
    const [state, formAction, isPending] = useActionState(deleteReviewAction, null);

    useEffect(() => {
        if (state && !state.status) {
            alert(state.error);
        }
    }, [state]);

    return (
        <form ref={formRef} action={formAction}>
            <input name="reviewId" value={reviewId} hidden />
            <input name="bookId" value={bookId} hidden />
            {isPending ? (
                <div>Loading...</div>
            ) : (
                <div onClick={() => formRef.current?.requestSubmit()}>삭제하기</div>
            )}
        </form>
    );
}
```

삭제 페이지 전체가 ClientSide로 전환할 필요가 없기 때문에, 컴포넌트 분리함
