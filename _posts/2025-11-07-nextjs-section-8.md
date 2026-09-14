---
title: "Next.js 섹션8. 고급 라우팅 패턴"
date: 2025-11-07 21:46:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

### 8.1 페럴렐 라우트

페럴렐 라우트 : 여러개의 페이지를 병렬로 라우트 (페이지 : page.tsx에 작성된 것)

![](/assets/img/2025-11-07/nextjs-section-8-1.png)

보통 SNS, 관리자 페이지 등 복잡한 페이지에서 유용하게

`@sidebar` : 슬롯이라 함, **병렬로 렌더링**이 될 페이지 컴포넌트를 보관하는 폴더

@sidebar, @feed

![](/assets/img/2025-11-07/nextjs-section-8-2.png)

- src/app/parallel
- src/app/parallel/layout.tsx
- src/app/parallel/page.tsx
- src/app/parallel/@sidebar/page.tsx
  - src/app/parallel/layout.tsx에게 props로 자동 전달

\*복잡한 UI를 그려낼 때 유용할 것

1. Slot은 URL 경로에 아무런 영향도 없음(라우트 그룹 느낌)

   → /parallel/@sidebar X

2. Slot은 갯수 제한이 없음
   - src/app/parallel/@sidebar/feed/page.tsx

3. 갈 곳이 없는 Slot은 404 에러가 아닌, 그냥 **이전 페이지 유지함**

   예) /parallel → /parallel/setting으로 이동 시

   처음 접속시에만 이전 페이지 유지

   새로고침 시 404를 불러냄 (이전값을 찾을 수 없기 때문)

   ⇒ default.tsx를 추가하면 됨 (parallel/@sidebar/default.tsx, parallel/default.tsx)

### 8.2 인터셉팅 라우트

intercept : 가로채다

요청된 페이지 → 특정조건 만족 시 → 원하는 페이지로 대신 라우트

![](/assets/img/2025-11-07/nextjs-section-8-3.png)

`조건 : 초기접속이 아닐 때에만`

`ClientSide`에서만 Link, Router(Push) 등 조건이 발동됨

예) 인스타그램

접속 시 게시글 위에 모달 형태 상세페이지

상세페이지 새로고침 시(초기접속 요청 시) 디테일페이지

개발방법

`src/app/(.)book/[id]`

`(.)` 는 intercept해라

`.`는 상대경로, `./`는 동일한 경로

- Intercept Page

  ```typescript
  import BookPage from '@/app/book/[id]/page';
  import Modal from '@/components/modal';

  export default function Page(props: any) { // prop 내리기
      return (
          <div>
              가로채기 성공!
              <Modal>
                  <BookPage {...props} />
              </Modal>
          </div>
      );
  }
  ```

- Modal (ClientSide)

  ```typescript
  'use client';

  import { ReactNode, useEffect, useRef } from 'react';
  import style from './modal.module.css';
  import { createPortal } from 'react-dom';
  import { useRouter } from 'next/navigation';

  export default function Modal({ children }: { children: ReactNode }) {
      const dialogRef = useRef<HTMLDialogElement>(null);
      const router = useRouter();

      useEffect(() => {
          if (!dialogRef.current?.open) {
              dialogRef.current?.showModal();
              dialogRef.current?.scrollTo({
                  top: 0,
              });
          }
      }, []);

      return createPortal(
          <dialog
              className={style.modal}
              ref={dialogRef}
              onClose={() => {
                  // esc 눌렀을 때도 닫기
                  router.back();
              }}
              // 모달 바깥 클릭 시 
              onClick={(e) => { 
                  if ((e.target as any).nodeName === 'DIALOG') {
                      router.back();
                  }
              }}
          >
              {children}
          </dialog>,
          document.getElementById('modal-root') as HTMLElement
      );
  }

  ```

- RootLayout에 modal-root 생성

  ```typescript
  export default function RootLayout({
      children,
  }: Readonly<{
      children: React.ReactNode;
  }>) {
      return (
          <html lang="en">
              <body>
                  <div className={style.container}>
                      <header>
                          <Link href={'/'}>📚 ONEBITE BOOKS</Link>
                      </header>
                      <main>{children}</main>
                      <Footer />
                  </div>
                  <div id="modal-root"></div>
              </body>
          </html>
      );
  }
  ```

### 8.3패럴렐 & 인터셉트 라우트

→ 특정 item을 모달로 띄워주는게 가능해잠

1. default 페이지 생성(기본 페이지는 index 페이지기 때문)

   src/app/@modal/(.)book/[id]/page.tsx

   src/app/@modal/default.tsx

2. RootLayout에 modal 추가해주기
