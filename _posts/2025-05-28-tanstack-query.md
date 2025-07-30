---
title: Tanstack-Query를 적용하면서
date: 2025-05-28 12:30:00 +0900
categories: [tanstack-query]
tags: [tanstack-query]
---

## 1. 문제 상황: 높은 DB CPU 사용률

- **문제의 핵심**: 클라이언트에서 **불필요한 API 요청을 반복적으로 발생**, 서버에 과부하
- Redis나 쿼리 캐시가 없고, read-only DB도 미구축 상태
- **특정 API는 페이지 진입 시마다 매번 호출됨 (유저 정보 관련)**
- 아래는 실제 모니터링 이미지 (쿼리 호출 과다):

![](/assets/img/2025-05-28/0528-1.png)

![](/assets/img/2025-05-28/0528-2.png)

read-only 디비/ 쿼리 캐싱/redis가 없기도 함

⇒ 근본적인 원인이 될수도 있는데, 클라이언트에서 계속해서 쿼리를 날리는 것도 문제

고민점 : staleTime, gcTime에 대한 의문

https://ttaerrim.tistory.com/53

```tsx
console.log(cartCountQuery);
// Proxy(Object) {status: 'success', fetchStatus: 'idle', isPending: false, isSuccess: true, isError: false, …}
```

```tsx
console.log(cartCountQuery.data);
// response : API response객체
```

## 2. 도입 방식 : 기존 useEffect → Tanstack Query로 전환

### Home 변환

- 변환 예시) Navigation

  ```tsx
  // original

  "use client";

  import { useEffect, useState } from "react";
  import { fetchAdmin } from "@/entities/user";
  import { fetchNotify, fetchCartCount } from "@/features/home/main/index";

  export function useGetNavigationData(token: string) {
    const [pushCount, setPushCount] = useState<number>(0);
    const [cartCount, setCartCount] = useState<number>(0);
    const [isAdmin, setIsAdmin] = useState<boolean>(false); // temp : showing timemoments if user is admin

    useEffect(() => {
      const getCount = async () => {
        const [notifyResponse, cartResponse, isAdminResponse] =
          await Promise.all([
            fetchNotify(token),
            fetchCartCount(token),
            fetchAdmin(token),
          ]);

        setPushCount(notifyResponse.data);
        setCartCount(cartResponse.data.count);
        setIsAdmin(isAdminResponse.unlock_admin ? true : false);
      };
      getCount();
    }, [token]);

    return { pushCount, cartCount, isAdmin };
  }
  ```

  ```tsx
  // tanstack applied

  "use client";

  import { fetchAdmin } from "@/entities/user";
  import { fetchNotify, fetchCartCount } from "@/features/home/main/index";
  import { useQueries } from "@tanstack/react-query";

  export function useGetNavigationData(token: string) {
    const results = useQueries({
      queries: [
        {
          queryKey: ["notify", token],
          queryFn: () => fetchNotify(token),
          staleTime: 1000 * 60, // 1 minutes
        },
        {
          queryKey: ["cartCount", token],
          queryFn: () => fetchCartCount(token),
          staleTime: 1000 * 60, // 1 minutes
        },
        {
          queryKey: ["isAdmin", token],
          queryFn: () => fetchAdmin(token),
          staleTime: Infinity,
          gcTime: Infinity, // INFINITY여야 DB 호출을 안함
        },
      ],
    });

    const [notifyQuery, cartCountQuery, adminQuery] = results;

    const pushCount = notifyQuery.data?.data ?? 0;
    const cartCount = cartCountQuery.data?.data?.count ?? 0;
    const isAdmin = adminQuery.isSuccess;

    return {
      pushCount,
      cartCount,
      isAdmin,
      isLoading: results.some((q) => q.isLoading),
      isError: results.some((q) => q.isError),
    };
  }
  ```

  `gcTime: Infinity` 설정 덕분에 `isAdmin` 쿼리는 **다시 호출되지 않음**, 캐싱 유지

  - isAdmin 캐싱X
    ![](/assets/img/2025-05-28/0528-5.png)

  - isAdmin이 캐싱됨 (GCTime : Infinite 적용 시)
    isAdmin 관련 API 호출이 안되고 있음을 확인할 수 있다.
    ![](/assets/img/2025-05-28/0528-6.png)

- 변환 예시) 홈화면 Board
  카테고리가 변경되어도 자동으로 캐싱됨 한번만 호출하고 staleTime내에는 캐싱함

  - 변경 전

  ```tsx
  "use client";

  import { useEffect, useRef, useState } from "react";
  import { fetchBoardByCategory } from "../api/client/fetch-board-by-category";

  export function useBoardByCategory(token: string) {
    const [boardData, setBoardData] = useState<bannerData>();
    const [selectedCategory, setSelectedCategory] = useState<
      | "all"
      | "main-banner"
      | "trending-moments"
      | "upcoming_moments"
      | "event-banner"
    >("all");

    const cache = useRef({}); // 캐싱 데이터 저장용

    useEffect(() => {
      const fetchData = async () => {
        const data = await fetchBoardByCategory(token, selectedCategory);
        cache.current[selectedCategory] = data;

        setBoardData(data);
      };

      fetchData();
    }, [selectedCategory]);

    return {
      boardData,
      selectedCategory,
      setSelectedCategory,
    };
  }
  ```

  - 변경 후

  ```tsx
  "use client";

  import { useState } from "react";
  import { fetchBoardByCategory } from "../api/client/fetch-board-by-category";
  import { useQuery } from "@tanstack/react-query";

  export function useBoardByCategory(token: string) {
    const [selectedCategory, setSelectedCategory] = useState<
      | "all"
      | "main-banner"
      | "trending-moments"
      | "upcoming_moments"
      | "event-banner"
    >("all");

    const { data: boardData } = useQuery({
      queryKey: ["boardData", selectedCategory],
      queryFn: () => fetchBoardByCategory(token, selectedCategory),
      staleTime: 1000 * 60 * 10, // 10분
    });

    return {
      boardData,
      selectedCategory,
      setSelectedCategory,
    };
  }
  ```

  코드가 간결해졌고, ref객체도 사용하지 않게 되었다.

### Store 변환

`queryKey` 변경만으로도 자동으로 refetch되므로, **이 `refetch()`는 제거해도 괜찮음.**

### Infinite Query

- `cursor` 기반 데이터 패칭
- Intersection Observer (`useInView`)와 조합
- 예시 1

  ```tsx
  "use client";

  import {
    fetchMessage,
    fetchReadAllMessage,
    fetchReadOneMessage,
  } from "@/entities/notification";
  import { fetchDeleteOneMessage } from "@/entities/notification/api/client/fetch-delete-one-message";
  import { messageHandler } from "@/shared/messageHandler";
  import { useRouter } from "next/navigation";
  import { useEffect, useState } from "react";
  import { useInView } from "react-intersection-observer";

  export function useNotificaionList(token: string) {
    const [cursor, setCursor] = useState<number | null>(0); // 초기값은 0
    const [isLoading, setLoading] = useState(false);
    const [hasMore, setHasMore] = useState(true); // 추가 데이터가 있는지 확인
    const [notifyData, setNotifyData] = useState<messageResultData>(null);
    const [read, setRead] = useState<{ id: number }[]>(); // 읽음 시 데이터 재호출 말고 객체 저장, 객체에 일치하는 데이터 존재 시 opacity 추가

    const [ref, inView] = useInView();

    const router = useRouter();

    const fetchData = async () => {
      if (isLoading || cursor === null || !hasMore) return; // 중복 요청 방지
      setLoading(true);

      try {
        const response = await fetchMessage(cursor, token);

        if (response.data.list.length === 0) {
          setHasMore(false);
        } else {
          setNotifyData((prevData) => ({
            ...response,
            data: {
              ...response.data,
              list:
                cursor === 0
                  ? response.data.list
                  : [...(prevData?.data.list || []), ...response.data.list], // 추가 로드 시 기존 데이터에 이어 붙이기
            },
          }));
          setCursor(response.data.paging.cursor); // 다음 요청을 위한 cursor 설정
        }
      } catch (error) {
        console.error("Error fetching data:", error);
      } finally {
        setLoading(false);
      }
    };

    useEffect(() => {
      fetchData();
    }, []);

    useEffect(() => {
      if (inView) {
        fetchData();
      }
    }, [inView]); // inView가 변경될 때 fetchData 호출

    // 전체 읽기
    const handleAllReadClick = async () => {
      const { success } = await fetchReadAllMessage(token);

      if (success) {
        setNotifyData(null);
        setCursor(0);
        setHasMore(true);

        // UI 깜빡임 방지를 위해 약간의 지연 후 fetchData 호출
        setTimeout(() => {
          fetchData();
        }, 100);
      }
    };

    // 푸시 읽음
    const handleReadPush = async (dataId: number) => {
      const { success } = await fetchReadOneMessage(dataId, token);

      if (success) {
        setRead((prevRead) => [...(prevRead || []), { id: dataId }]);
      }
    };

    // 푸시 삭제
    const handleDeletePush = async (dataId: number) => {
      const { success } = await fetchDeleteOneMessage(dataId, token);

      if (success) {
        setNotifyData((prevData) => {
          if (!prevData) return prevData;

          return {
            ...prevData,
            data: {
              ...prevData.data,
              list: prevData.data.list.filter((item) => item.id !== dataId),
            },
          };
        });
      }
    };

    const handleMessageHandler = (dataId: number, link: string) => {
      handleReadPush(dataId);
      // todo : 만약 news/${id}라면 router.push로 전달
      if (link?.startsWith("news"))
        return router.replace(`notice/${link.split("/")[1]}`);

      messageHandler(link);
    };

    return {
      handleAllReadClick,
      notifyData,
      read,
      ref,
      handleReadPush,
      handleDeletePush,
      handleMessageHandler,
    };
  }
  ```

- 예시 2

  - 기존

    ```tsx
    "use client";

    import { useEffect, useState } from "react";
    import { useInView } from "react-intersection-observer";
    import { ProductSkeleton } from "./ProductSkeleton";
    import Product from "@/app/(page)/(playzone)/nacho-store/components/NachoStore/Product";

    const PAGE_SIZE = 4;

    export function ItemList() {
      const [nachoProductList, setNachoProductList] = useState<NachoProduct[]>(
        [],
      );
      const [isLoading, setLoading] = useState(false);
      const [hasMore, setHasMore] = useState<boolean>(true); // infinity scroll
      const [page, setPage] = useState(1);

      const [ref, inView] = useInView();

      useEffect(() => {
        const fetchNachoProducts = async () => {
          if (!inView || !hasMore) return;
          setLoading(true);

          try {
            const take = 4; // 한 번에 가져올 상품 개수

            const productResponse = await fetch(
              `/api/moment/nacho-product?take=${take}&page=${page}`,
            );

            const productData = await productResponse.json();

            if (productData.length < take) {
              setHasMore(false);
            } else {
              setPage((prevPage: number) => prevPage + 1);
            }

            setNachoProductList((prev) => [...prev, ...productData]);

            setLoading(false);
          } catch (error) {
            console.error("Failed to fetch products:", error);
          }
        };
        fetchNachoProducts();
      }, [inView, hasMore]);

      return (
        <div>
          <div>
            {isLoading && <ProductSkeleton />}
            <Product nachoProductList={nachoProductList} />
          </div>
          <div ref={ref}></div> {/* Intersection Observer 대상 */}
        </div>
      );
    }
    ```

  - Tanstack-Query 적용

    ```tsx
    "use client";

    import { useEffect, useState } from "react";
    import { useInView } from "react-intersection-observer";
    import { ProductSkeleton } from "./ProductSkeleton";
    import Product from "@/app/(page)/(playzone)/nacho-store/components/NachoStore/Product";
    import { useInfiniteQuery } from "@tanstack/react-query";

    const PAGE_SIZE = 4;

    export function ItemList() {
      const [ref, inView] = useInView();

      const fetchNachoProducts = async ({
        pageParam = 1,
      }): Promise<NachoProduct[]> => {
        const res = await fetch(
          `/api/moment/nacho-product?take=${PAGE_SIZE}&page=${pageParam}`,
        );
        if (!res.ok) throw new Error("Failed to fetch products");
        return res.json();
      };

      const {
        data,
        fetchNextPage,
        hasNextPage,
        isFetchingNextPage,
        isLoading,
        error,
      } = useInfiniteQuery({
        queryKey: ["nacho-products"],
        queryFn: fetchNachoProducts,
        initialPageParam: 1,
        getNextPageParam: (lastPage, allPages) => {
          // 다음 페이지가 존재하면 페이지 번호 반환
          if (lastPage.length < PAGE_SIZE) return undefined; // 더 이상 없음
          return allPages.length + 1; // 다음 page 번호
        },
      });

      // inView일 때 다음 페이지 로드
      useEffect(() => {
        if (inView && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);

      const nachoProductList = data?.pages.flat() ?? [];

      return (
        <div>
          <div>
            {isLoading && <ProductSkeleton />}
            <Product nachoProductList={nachoProductList} />
          </div>
          <div ref={ref}></div> {/* Intersection Observer 대상 */}
        </div>
      );
    }
    ```

| 항목            | before                 | after (React Query)            |
| --------------- | ---------------------- | ------------------------------ |
| 중복 호출       | O (useEffect마다 호출) | X (staleTime/gcTime 기반 캐싱) |
| 메모리 관리     | 직접 구현              | 자동 GC 지원                   |
| 쿼리 캐싱       | useRef 등 수동 관리    | queryKey로 자동 관리           |
| 리팩토링 난이도 | 높음                   | queryKey 기반으로 간단         |
| Infinite Scroll | 수동 상태관리          | useInfiniteQuery 적용 가능     |

## 3. 유의점 (데이터가 존재할 때 사용할 것, undefined X)

https://github.com/tanonkim/unlock-event-next/pull/89

```tsx
if (user) {
  await issueMomentsAndSaveResult(
    random,
    editionId,
    user.id,
    token,
    gameResultArray,
  );
}

// useEffect 의존성 안에 user를 사용하는 경우 user를 넣고, user가 있는 경우를 체크할 것
```

- 궁금증
  푸시 같은건 쿼리 적용해야 할까?
  만약 푸시가 보내졌는데, 쿼리된다면 안될텐데 라는 고민

**결론**

- ✅ **캐싱이 필요한 데이터에만 React Query 적용**
  - 예: `isAdmin`, `cartCount`, `보드 목록`, `상품 리스트`
- ❌ 실시간성이 중요한 데이터는 신중하게 접근
  - 예: **푸시 알림은 서버 변경 직후 반영 필요 → 쿼리 캐싱 부적절**

## 4. 개선점

DB의 사용률이 확연히 줄어들었다.

사용중인 DB 스펙은 아래와 같다.
• PostgreSQL / Seoul
• Replica Nodes: 1
• 65536 MB RAM
• 16 vCPUs
• 1280 GB Disk
• Cluster Status: Running

기존)

![](/assets/img/2025-05-28/0528-3.png)

개선) Tanstack-Query 적용 후

![](/assets/img/2025-05-28/0528-4.png)
