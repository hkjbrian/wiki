# 기술 학습 기록

## 주제
> MySQL InnoDB 는 실제로 어떤 정보를 이용하여 읽기 일관성과 동시성을 만드는가

## 핵심 질문
- MVCC 가 왜 필요한가 ? 
- Undo Log 와 Read View 란 무엇인가 ? 
- Snapshot Read 와 Current Read 의 차이는 ?
- Isolation Level 에 따른 Read View 생성 시점의 차이
- MVCC 와 Lock

## 학습을 통해 정리한 내용

### MVCC 가 필요한 이유는 ?

데이터베이스로는 여러 사용자가 동시에 접근한다.

상품 재고가 10개일 때, 서로 다른 사용자가 동시에 10개씩 구매하는 요청이 온다면 ? 실제 재고는 10개 남았는데, 두 사용자가 모두 구매에 성공하는 문제가 발생할 수 있다.

이런 문제를 막기 위해서 데이터베이스에서는 **동시성 제어**가 필요하다.

가장 단순한 방법은 **Lock** 이다. 

```text
누군가 데이터를 수정하고 있다면, 다른 요청은 대기한다.
```

하지만 모든 읽기와 쓰기를 Lock 으로만 처리하게 된다면 성능이 매우 떨어지게 될 것이다.
그래서 InnoDB 에서는 MVCC 를 사용한다.

### MVCC 란?

MVCC 는 Multi-Version Concurrency Control(다중 버전 동시성 제어)의 약자이다.

핵심 아이디어는 다음과 같다
```text
하나의 데이터에 대해서 여러 시점의 버전을 관리하여, 트랜잭션별로 본인에게 맞는 버전을 읽도록 한다.
```
즉, 어떤 트랜잭션이 데이터를 수정하고 있더라도 다른 트랜잭션이 대기하는 것이 아니라 자신이 볼 수 있는 버전을 읽게 된다.

MVCC 의 장점은 다음과 같다.
- 읽기는 쓰기를 막지 않는다.
- 쓰기는 읽기를 막지 않는다.

단, 이것은 일반적인 SELECT 를 통한 읽기에 한해서 적용된다. UPDATE, DELETE, SELECT FOR UPDATE 와 같은 Current Read 에서는 Lock 이 개입하게 된다.

### MVCC 의 구성 요소

MVCC 를 이해하기 위해서는 4가지 요소를 알아야 한다.
```text
1. 현재 row
2. Undo Log
3. Transaction ID
4. Read View
```

**현재 row**  

테이블에는 기본적으로 최신 데이터가 저장되어 있다. 예를 들어 재고가 10개였다가 1개가 차감되면 테이블에는 9 라는 값이 존재한다.
이 때 이전 값인 10 은 그냥 덮어씌워지는 것이 아니라 Undo Log 에 저장된다.

**Undo Log**  

Undo Log 는 데이터 변경 전의 값을 저장하는 로그이다.

예를 들어 다음 SQL이 실행되었다고 하자.

```sql
UPDATE product
SET stock = 9
WHERE id = 1;
```
InnoDB는 현재 row를 9로 바꾸면서, 이전 값 10을 Undo Log에 남긴다.

```text
현재 row:
stock = 9

Undo Log:
이전 stock = 10
```

이걸 활용하여 트랜잭션별로 볼 수 있는 값의 차이를 만들어내게 된다.
중요한 점은 InnoDB 가 테이블 안에 row 복사본을 여러 개 쌓아두는 것이 아니라, row 와 Undo Log 를 이용해서 과거 버전을 찾아가는 것

**Transaction ID**  

InnoDB 는 각 트랜잭션에 ID 를 부여한다.
데이터를 변경하면 row 에는 이 row 를 마지막으로 변경한 Transaction ID 가 기록된다.

이 정보는 어떤 트랜잭션이 이 변경 사항을 볼 수 있는지 판단할 때 사용된다.

**Read View**  

현재 트랜잭션이 어떤 데이터를 볼 수 있는지 판단의 기준이 된다. 쉽게 말하면 스냅샷이다.

트랜잭션이 어떤 시점에 데이터를 읽을 때, InnoDB 는 Read View 를 기준으로 다음을 판단한다

```text
이 row 는 내가 볼 수 있는 row 인가 ? 
아직 커밋되지 않은 다른 트랜잭션이 만든 row 인가 ?
내가 직접 변경한 row 인가 ?
이미 커밋된 row 인가 ?
```

### MVCC 를 통한 읽기 동작 예시

초기 데이터 :
```text
product
id = 1
stock = 10
```

트랜잭션 A 의 시작

```sql
START TRANSACTION;

SELECT stock FROM product WHERE id = 1;

```

트랜잭션 A 는 stock = 10 을 읽는다.
이 때 트랜잭션 B 가 재고를 변경하고 커밋한다.

```sql
START TRANSACTION;

UPDATE product
SET stock = 9
WHERE id = 1;

COMMIT;
```

현재 row 는 다음과 같다
```text
현재 row:
stock = 9

Undo Log:
이전 stock = 10
```

여기서 트랜잭션 A 가 다시 조회한다면 ? 

결과는 Isolation Level 에 따라서 달라질 수 있다.
수준에 따라서 Read View 를 만드는 시점이 달라지기 때문이다.

**READ COMMITTED**

SELECT 를 실행할 때마다 새로운 Read View 를 만든다.

```text
트랜잭션 A 시작
SELECT stock → 10

트랜잭션 B가 stock을 9로 변경 후 COMMIT

트랜잭션 A에서 다시 SELECT
→ 새로운 Read View 생성
→ 이미 커밋된 B의 변경을 볼 수 있음
→ 9 조회
```

즉, 같은 트랜잭션 안에서도 다시 조회하게 되면 결과가 달라질 수 있다.
이것을 Non-repeatable Read 라고 한다.

**REPEATABLE READ**

트랜잭션 안에서 첫 Snapshot Read 시점의 Read View 를 계속 사용한다.
MySQL InnoDB 의 기본 격리 수준에 해당한다.

```text
트랜잭션 A 시작
SELECT stock → 10
이 시점의 Read View 생성

트랜잭션 B가 stock을 9로 변경 후 COMMIT

트랜잭션 A에서 다시 SELECT
→ 기존 Read View 재사용
→ 여전히 10 조회
```

즉, 같은 트랜잭션 안에서 같은 데이터를 반복 조회하면 같은 결과가 나온다.

### Snapshot Read 와 Current Read

MVCC 를 공부할 때 가장 많이 헷갈리는 부분이다.
MySQL 에서 읽기는 크게 2종류로 나눌 수 있다.
- Snapshot Read
- Current Read

**Snapshot Read**

일반적인 SELECT 는 Snapshot Read다

Snapshot Read 는 Read View 를 기준으로 자신에게 보여야 하는 버전을 읽게 된다.

특징은 다음과 같다.
```text
일반 SELECT 에서 사용
Undo Log 를 따라가서 과거 버전을 읽을 수 있다
읽기 때문에 Lock 을 걸지 않는다
트랜잭션 격리 수준에 따라서 보이는 데이터가 달라질 수 있다
```

**Current Read**

SELECT FOR UPDATE, UPDATE, DELETE 문은 Current Read 에 해당한다.

Current Read 는 Read View 스냅샷이 아니라 최신 데이터를 기준으로 동작한다.

왜냐하면 데이터를 수정하거나 잠글 목적이 있기 때문이다.

특징은 다음과 같다.
```text
최신 커밋 데이터를 기준으로 읽는다
변경을 위해 Lock 이 필요하다
UPDATE, DELETE, SELECT FOR UPDATE 에서 사용된다.
MVCC 만으로 처리하지 않는다
```

다음과 같은 상황이 있다고 하자.

초기 데이터 row = 10

트랜잭션 A 가 먼저 일반 SELECT 를 수행한다.

```sql
START TRANSACTION;

SELECT value
FROM sample
WHERE id = 1;
```

트랜잭션 A 는 Snapshot Read 를 통해 10 을 읽는다.

그 후 트랜잭션 B 가 값을 변경한다.

```sql
START TRANSACTION;

UPDATE sample
SET value = 5
WHERE id = 1;
```

아직 트랜잭션 B 가 COMMIT 은 하지 않은 상태에서 내부 상태는 다음과 같다
```text
row = 5
DB_TRX_ID = B
lock = B 가 X lock 을 보유

Undo Log :
이전 value = 10
```

이 상태에서 트랜잭션 A 가 Current Read 를 수행한다면 ?

트랜잭션 A 는 Undo log 를 따라 10 을 읽는 것이 아니라, 트랜잭션 B 의 종료를 기다린다.
Current Read 는 Lock 을 획득해야 하기 때문이다.

### MVCC, Lock, Commit, Rollback 의 관계

```text
MVCC
- 읽기 일관성 제공
- 일반 SELECT 가 과거 버전을 읽을 수 있게 해준다.

Undo Log
- Rollback 할 때 변경을 되돌리기 위해 사용한다
- Snapshot Read 에서 과거 버전을 재구성할 때도 사용한다

Lock
- 동시에 같은 row 를 수정하지 못하게 막는다.
- Current Read, UPDATE, DELETE 에서 활용된다.

Commit
- 현재 트랜잭션의 변경을 확정한다
- 보유하던 lock 을 해제한다

Rollback
- Undo Log 를 이용해 현재 트랜잭션의 변경을 취소한다
- 보유하던 lock 을 해제한다
```

### DB Isolation Level

DB 에는 다음 4단계의 문제가 발생할 수 있다.
1. Dirty Read : 다른 트랜잭션의 변경이 보이는 문제
2. Non-Repeatable Read : 같은 트랜잭션 내에서 여러 번 조회했을 때 값이 바뀌는 문제
3. Phantom Read : 같은 트랜잭션 내에서 여러 번 조회했을 때, row 가 삽입 / 삭제되는 문제

각 문제를 해결하는 단계대로 Read Uncommitted -> Read Committed -> Repeatable Read -> Serializable 수준이라고 말한다.

특히 Serializable 의 경우에는 MVCC 를 버리고 Lock 만 사용하는 방식이라기보다는 일반 SELECT 에서도 Current Read 를 사용하여 읽기 정합성을 더 강력하게 보장한다고 이해할 수 있다.

### Undo Log 의 체인 구조와 Read View 에 들어가는 정보

Undo Log 는 한 번의 변경에만 생기지 않는다. 하나의 트랜잭션에서 같은 row 를 여러 번 변경하면 변경마다 Undo log Record 가 만들어지고, 이것들이 체이닝된다.

클러스터 인덱스 ( 테이블 ) 에는 다음과 같은 부가정보가 함께 저장된다.

```text
DB_TRX_ID
→ 이 row 를 마지막으로 insert/update 한 트랜잭션 ID

DB_ROLL_PTR
→ 이전 버전을 만들 수 있는 Undo Log record 를 가리키는 포인터
```

Read View 에는 다음과 같은 정보들이 들어간다.
```text
creator_trx_id
→ 이 Read View 를 만든 트랜잭션 ID

m_ids
→ Read View 생성 시점에 아직 커밋되지 않고 실행 중이던 트랜잭션 ID 목록

up_limit_id
→ m_ids 중 가장 작은 트랜잭션 ID
→ 이 값보다 작은 trx_id 는 이미 커밋된 것으로 볼 수 있음

low_limit_id
→ Read View 생성 시점 이후에 만들어질 트랜잭션 ID 경계
→ 이 값 이상인 trx_id 는 나중에 시작된 트랜잭션이므로 볼 수 없음
```

이렇게 만들어진 Read View 를 기반으로 다음과 같이 현재 트랜잭션에서 row 값을 읽을 수 있는지 없는지 판단한다.
```text
1. DB_TRX_ID == TRANSACTION_ID
   → 내가 변경한 row 이므로 볼 수 있음

2. DB_TRX_ID < up_limit_id
   → Read View 생성 전에 이미 커밋된 변경이므로 볼 수 있음

3. DB_TRX_ID >= low_limit_id
   → Read View 생성 이후 시작된 트랜잭션의 변경이므로 볼 수 없음

4. DB_TRX_ID in m_ids
   → Read View 생성 당시 아직 실행 중이던 트랜잭션의 변경이므로 볼 수 없음

5. 그 외
   → Read View 생성 시점에는 이미 커밋된 변경으로 볼 수 있음
```

위 기준을 따라서 현재 row 를 볼 수 있는지 여부를 판단한다.

예를 들어 현재 row 가 다음과 같다.

```text
현재 row:
stock = 5
DB_TRX_ID = 100
DB_ROLL_PTR → undo_2
```

이 row 를 읽는 트랜잭션이 row 를 볼 수 없다면 DB_ROLL 을 따라 과거 버전으로 이동한다.

```text
undo_2:
이전 stock = 7
이전 DB_TRX_ID = 100
이전 roll pointer → undo_1

undo_1:
이전 stock = 10
이전 DB_TRX_ID = 70
```

이런 식으로 과거 버전을 찾아가다가 볼 수 있는 버전을 만나면 그 값을 반환한다.

### S Lock 과 X Lock

S Lock 은 공유 락이다.  
FOR SHARE 에서 활용된다. S lock 을 row 에 걸게 되면 다른 트랜잭션들이 읽을 수는 있지만, 수정할 수 없다.

X Lock 은 배타 락이다.  
변경 작업에서 사용되며 X lock 을 걸게 되면 다른 트랜잭션들이 해당 lock 이 풀릴 때 까지 대기하게 된다.

Lock 은 DB_TRX_ID 처럼 row record 에 저장되지 않고, lock Manager 가 메모리에서 관리한다.


## 프로젝트 적용

- 재고 차감 로직에서 `PESSIMISTIC_WRITE` 를 사용해 Current Read 와 X Lock 이 어떻게 동작하는지 확인할 수 있다.
- 일반 상품 조회 / 주문 조회 로직에서 Snapshot Read 가 Lock 없이 Read View 기준으로 동작하는 흐름을 떠올려볼 수 있다.
- 상품 수정, 상품 종료처럼 단순 수정 로직에서는 bulk query 보다 영속 상태 엔티티를 조회한 뒤 도메인 메서드로 상태를 변경하는 Dirty Checking 방식을 우선 고려한다.
- 상품 상태 만료 처리, 단순 플래그 변경처럼 엔티티별 도메인 로직이 필요 없는 대량 변경에서는 bulk query 를 사용할 수 있다.
- bulk query 사용 후에는 영속성 컨텍스트와 DB 상태 불일치가 발생할 수 있으므로 `clearAutomatically = true`, `flushAutomatically = true` 옵션을 상황에 따라 고려한다.
- Lazy Loading 이 필요한 연관관계는 서비스 계층의 `@Transactional(readOnly = true)` 범위 안에서 접근하거나, fetch join / EntityGraph / DTO 조회로 필요한 데이터를 명시적으로 가져온다.

## 오늘의 결론

MVCC 는 읽기 정합성을 위한 기술, Lock 은 쓰기 정합성을 위한 기술이라는 것을 이해하게 됨

특히 Undo Log 에 대해서 수정한 값을 별도 버전으로 저장하는 것이 아니라, 수정한 값은 테이블에서 즉시 수정하고, Undo Log 로 되돌아갈 값을 저장하고 있는 구조가 신기했음

X lock 을 바탕으로 쓰기 작업을 하게 되므로 쓰기 관련된 작업에서 확실히 병목이 발생할 가능성이 크겠다고 느꼈음


## 다음 액션

- [ ] 다양한 Lock 의 종류 학습하기

## 면접 질문
- InnoDB 에서 일반 SELECT 는 X Lock 이 걸린 row 를 왜 기다리지 않고 읽을 수 있는가? 이때 Read View, DB_TRX_ID, DB_ROLL_PTR, Undo Log 가 어떤 순서로 사용되는지 설명해주세요.
- REPEATABLE READ 트랜잭션 안에서 같은 row 를 일반 SELECT 로는 10으로 읽었는데, 이후 SELECT FOR UPDATE 로는 5로 읽을 수 있는 이유를 Snapshot Read 와 Current Read 관점에서 설명해주세요.
- Read View 의 `m_ids`, `up_limit_id`, `low_limit_id` 는 각각 어떤 역할을 하는가? 특히 Read View 생성 이후 시작된 트랜잭션의 변경을 `m_ids` 만으로 판별할 수 없는 이유를 설명해주세요.
- 한 트랜잭션에서 같은 row 를 10 → 7 → 5 로 여러 번 수정하면 Undo Log 는 어떻게 구성되는가? 이후 다른 트랜잭션의 Snapshot Read 와 현재 트랜잭션의 Rollback 에서 이 Undo Log chain 이 각각 어떻게 사용되는지 설명해주세요.
- JPA 에서 Dirty Checking 으로 재고를 차감하는 로직이 있다고 할 때, 이것만으로 동시성 정합성을 보장할 수 없는 이유는 무엇인가? InnoDB 의 Current Read, X Lock, Commit/Rollback 시점과 연결해서 설명해주세요.
