---
title: "DB Lock 종류 + 해결방법"
date: 2026-03-01 22:11:00 +0900
categories: [database, mysql]
tags: [mysql, lock, jpa]
---

## 1. Lock의 기본 개념

Lock은 **동시성 제어**를 위한 매커니즘이다. 여러 트랜잭션이 동시에 같은 데이터에 접근할 때 데이터 `정합성`을 보장하기 위해 사용된다

## 2. Lock 종류(DB레벨)

### Shared Lock

**읽기 작업에 사용된다.**

여러 트랜잭션이 동시에 SLock을 획득할 수 있지만, SLock이 걸린 데이터에 대해 쓰기는 불가능하다. SLock이 걸린 데이터에 대해 쓰기(XLock)는 불가하다.

```java
SELECT * FROM member WHERE id = 1 FOR SHARE;
```

### Exclusive Lock

쓰기 작업에 사용된다.

XLock이 걸리면 다른 트랜잭션은 SLock/XLock 모두 획득할 수 없다.

```sql
SELECT * FROM member WHERE id = 1 FOR UPDATE;
UPDATE member SET name = 'kim' WHERE id = 1;  -- 자동으로 X Lock
```

### Lock 범위에 따른 분류(MySQL InnoDB 기준)

| 종류 | 설명 |
| --- | --- |
| **Row Lock** | 특정 행에만 Lock. 가장 세밀하고 동시성이 높음 |
| **Gap Lock** | 인덱스 레코드 사이의 "간격"에 Lock. 팬텀 리드 방지 |
| **Next-Key Lock** | Row Lock + Gap Lock 결합. InnoDB의 기본 잠금 방식 |
| **Table Lock** | 테이블 전체에 Lock. 동시성 최저 |
| **Auto-Increment Lock** | AUTO_INCREMENT 컬럼 삽입 시 테이블 레벨 Lock |

**Gap Lock 예시**: `WHERE age BETWEEN 20 AND 30`으로 조회하면, 20~30 사이 "간격"에도 Lock이 걸려 그 범위에 새로운 INSERT가 차단됩니다. 이것이 Phantom Read를 방지하는 원리

**Phantom Read**: 한 개의 Transaction 에서 같은 조건으로 2번 읽었는데 2 번의 결과가 다른 현상. **없던 데이터가 생기는 현상**

## 3. Application 레벨 Lock(JPA 관점)

### 낙관전 Lock (Optimistic Lock)

**가정 : 충돌이 거의 없을 것**

충돌이 발생하면 그 때 발생하는 방식, DB Lock을 사용하지 않고 버전 관리로 동시성을 제어한다.

```java
@Entity
public class Product {
    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private int stock;

    @Version  // 이 필드로 충돌 감지
    private Long version;
}

// TX-A: SELECT product (version=1)
// TX-B: SELECT product (version=1)
// TX-A: UPDATE product SET stock=9, version=2 WHERE id=1 AND version=1  → 성공
// TX-B: UPDATE product SET stock=8, version=2 WHERE id=1 AND version=1  → 실패! (version 불일치)
//        → OptimisticLockException 발생
```

**JPA에서 사용**

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.OPTIMISTIC)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithOptimisticLock(@Param("id") Long id);
}
```

**LockModeType 종류**

`OPTIMISTIC` : 엔티티 수정 시 version 체크

`OPTIMISTIC_FORCE_INCREMENT` : 엔티티를 읽기만 해도 Version 증가, 연관 엔티티 변경 시 부모의 정합성 보장에 유용

**충돌 시 재시도 처리**

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    
    
    @Retryable(
	    value = OptimisticLockException.class,
	    maxAttempts = 3,
	    backoff = @Backoff(delay=100)
    )
    @Transactional
    public void decreaseStock(Long id) {
	    Product product = productRepository.findByIdWithOptimisticLock(id)
												   .orElseThrow();
		   product.decreaseStock();
    }
    
}
```

적합한 상황 : 충돌 빈도가 낮은 경우, 읽기가 많고 쓰기가 적은 경우

### 비관적 Lock (Pessimistic Lock)

**가정 : 충돌이 자주 발생할 것**

데이터를 읽을 때부터 Lock을 거는 방식

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

		@Lock(LockModeType.PESSIMISTIC_WRITE)
		@Query("SELECT p FROM product WHERE p.id = :id")
		Optional<Product> findByIdWithPessimisticLock(@Param("id") Long id);

}
```

**LockModeType 종류:**

| 모드 | 생성 SQL | 설명 |
| --- | --- | --- |
| `PESSIMISTIC_READ` | `FOR SHARE` | S Lock. 다른 트랜잭션 읽기 허용, 쓰기 차단 |
| `PESSIMISTIC_WRITE` | `FOR UPDATE` | X Lock. 다른 트랜잭션 읽기/쓰기 모두 차단 |
| `PESSIMISTIC_FORCE_INCREMENT` | `FOR UPDATE` + version++ | X Lock + 버전 증가 |

**타임아웃 설정**

DeadLock 방지에 중요하다.

```java

@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
		@QueryHint(name = "javax.persistence.lock.timeout", value = "3000") // 3 seconds
})
@Query("SELECT p from Product WHERE p.id = :id")
Optional<Product> findByIdWithPessimisticLock(@Param("id") Long id);
```

적합한 상황 : 금융 등 데이터 정합성이 절대적으로 필요한 경우, 충돌 빈도가 높은 경우

### NamedLock (User-Level Lock)

MySQL의 GET_LOCK()을 활용하는 방식

특정 문자열 키에 대한 Lock이다. 테이블이나 행이 아닌 이름에 대한 Lock이라는 점이 특징이다.

```java
public interface LockRepository extends JpaRepository<Product, Long> {

		@Query(value = "SELECT GET_LOCI(:key, 3000)", nativeQuery = true)
		void getLock(@Param("key") String key);

		@Query(value = "SELECT GET_LOCI(:key)", nativeQuery = true)		
		void releaseLock(@Param("key") String key);
	
}
```

NamedLock은 별도의 커넥션 풀(DataSource)를 사용하느게 좋다. Lock 전용 커넥션과 비즈니스 로직 커넥션이 같은 풀을 쓰면 커넥션 고갈이 발생할 수 있다.

## 4. 분산환경에서 Lock

단일 서버가 아닌 다중 서버(Scale-out) 환경에서는 DB Lock만으로는 부족할 수 있어 분산 락이 필요하다.

### Redis 기반 분산 락(Redisson)

```java
@Component
@RequiredArgsConstructor
public class RedissonLockService {

		private final RedissonClient redissionClient;
		
		public void executeWithLock(String key, Runnable task) {
				Rlock lock = redissionClient.getLock(key);
				try {
						boolean acquired = lock.tryLock(5, 3, TimeUnit.SECONDS); // 최대 대기시간, Lock 자동 해제 시간
						if (!acquired) {
							throw new RuntimeException("Lock 획득 실패")
						}
						
						task.run();
       } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }

}
```

|  | Redisson | Lettuce |
| --- | --- | --- |
| 방식 | Pub/Sub 기반 (이벤트 구동) | Spin Lock (폴링) |
| Redis 부하 | 낮음 | 높음 (반복 요청) |
| 재시도 | 내장 | 직접 구현 |
| 추천 | ✅ 분산 락에 적합 | 간단한 경우 |

## 5. Deadlock (교착 상태)

### 발생조건 (4가지 모두 충족 시)

1. 상호배제 : 자원을 하나의 트랜잭션만 사용 가능
2. 점유와 대기 : Lock을 가진 채 다른 Lock을 기다림
3. 비선점 : 다른 트랜잭션의 Lock을 강제로 빼앗을 수 없음
4. 순환대기 : TX-A → TX-B → TX-A 형태의 원형 대기

```java
TX-A: UPDATE account SET balance = 900 WHERE id = 1;  -- id=1 X Lock 획득
TX-B: UPDATE account SET balance = 900 WHERE id = 2;  -- id=2 X Lock 획득
TX-A: UPDATE account SET balance = 1100 WHERE id = 2; -- id=2 대기 (TX-B가 점유)
TX-B: UPDATE account SET balance = 1100 WHERE id = 1; -- id=1 대기 (TX-A가 점유) → DEADLOCK!
```

### 해결방법

1. 접근 순서 통일

   id가 작은 것부터 Lock → 순환대기 방지

2. Lock의 타임아웃 설정

   QueryHint 예

3. 트랜잭션 범위 최소화

   트랜잭션 안에서 외부 API 호출 X

   ```java
   // BAD: 트랜잭션 안에서 외부 API 호출
   @Transactional
   public void process() {
       Product p = productRepo.findByIdWithLock(1L);
       externalApiCall();  // 오래 걸림 → Lock 장기 점유
       p.decrease();
   }

   // GOOD: Lock 구간 최소화
   public void process() {
       ExternalResult result = externalApiCall();  // 트랜잭션 밖
       decreaseWithLock(1L, result);               // 짧은 트랜잭션
   }
   ```

4. 인덱스 최적화

   인덱스가 없으면 테이블 풀 스캔 → 의도치 않은 넓은 범위 Lock → DeadLock 확률 증가

## 6. 어떤 Lock을 선택할 것인가

충돌빈도 낮음 + 단일서버 → 낙관적 락(@Version)

충돌빈도 높음 + 단일서버 → 비관적 락(PESSIMISTIC_WRITE)

충돌빈도 높음 + 다중서버 → Redis 분산 락 (Redission)

초고정합성 + 분산락 → NamedLock or ZooKeeper

일반적으로 낙관적 락 → 충돌이 빈번해지면 비관적 락이나 분산 락으로 전환하는 접근이 일반적이다.
