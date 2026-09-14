---
title: "element.getBoundingClientRect()"
date: 2025-04-20 17:44:00 +0900
categories: [browser, web-api]
tags: [browser, dom]
---

`element.getBoundingClientRect()` 메서드는 엘리먼트의 크기와 **뷰포트에 상대적인 위치 정보**를 제공하는 DOMRect 객체를 반환한다.

### 구문

`domRect = element.getBoundingClientRect();`

x, y, top, right, bottom, left, width, height 등의 속성을 가지고 있다.

width, height는 element의 가로 세로 값을 나타낸다.

y와 top은 같다. x와 left값은 같다.

bottom의 경우는 height를 포함한 값이다. right는 width값을 포함한 값이다.

![](/assets/img/2025-04-20/browser-get-bounding-client-rect-1.png)

### 예시 1)

```typescript
  const updateProgress = (clientX: number) => {
    if (!progressContainerRef.current || isVideoStarted) return;

    const clamped = getClamped(progressContainerRef, clientX);

    setProgress(clamped); // state
    getVideoTime(clamped);
    startVideo(clamped);
  };

  const getClamped = (progressContainerRef: RefObject<HTMLDivElement | null>, clientX: number) => {
    const rect = progressContainerRef.current!.getBoundingClientRect();
    const x = clientX - rect.left;
    const clamped = Math.max(0, Math.min(x, 850));

    return clamped;
  };

  const getVideoTime = (clamped: number) => {
    const previewTime = clamped / 850; // 0 ~ 1

    if (videoRef.current) {
      videoRef.current.currentTime = previewTime;
    }
  };

  // 100% 도달 시: 영상 1초부터 재생
  const startVideo = (clamped: number) => {
    if (clamped >= 600) {
      if (videoRef.current) {
        videoRef.current.currentTime = 1;
        videoRef.current.play();
        setIsProgressbarShowing(false);
      }
      setIsVideoStarted(true);
    }
  };
```

clientX : 마우스가 클릭된 브라우저 기준 X 좌표

rect.left : 브라우저에서 왼쪽으로 얼마나 떨어져있는가 (변경 X)

clientX - rect.left : 프로그레스 바 내부에서 몇 px 떨어져있는가

`progressContainerRef.current!.getBoundingClientRect` 리턴값

```json
{
  bottom: 1099.9921875;
  height: 19.9921875;
  left: 298.9921875;
  right: 1148.9765625;
  top: 1080;
  width: 849.984375;
  x: 298.9921875;
  y: 1080;
}
```

### 예시 2)

유저가 스크롤을 했을 때 element의 위치에 따라 element를 보여주거나 안보여주는 예시이다.

useRef를 이용해서 DOM을 기준으로 잡고, DOM에 getBoundingClientRect() 메서드를 실행해서 top위치를 감지한다. top위치에 따라서 hideElement를 true/false로 변경한다.

```typescript
'use client';

import { useEffect, useRef, useState } from 'react';

export default function Page() {
  const [hideElement, setHideElemnt] = useState<boolean>(false);
  const scrollRef = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    if (!scrollRef.current) return;
    window.addEventListener('scroll', yScrollEvent);
  }, [scrollRef.current]);

  const yScrollEvent = () => {
    const scroll = scrollRef.current.getBoundingClientRect();
    console.log(scroll);
    setHideElemnt(scroll.top <= -100);
  };

  return (
    <div className="h-[300vh] bg-cyan-100" ref={scrollRef}>
      {!hideElement && (
        <div className="fixed bg-slate-500">
          <span className="text-xl">스크롤을 일정 수치 내면 이 영역은 사라진다</span>
        </div>
      )}
    </div>
  );
}
```

![](/assets/img/2025-04-20/browser-get-bounding-client-rect-2.png)
