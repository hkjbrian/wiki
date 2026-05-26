# 기술 학습 기록

## 주제
> MySQL InnoDB 는 실제로 어떤 정보를 기준으로 데이터를 잠그고 동시성을 제어하는가

## 핵심 질문
- InnoDB 의 Lock 은 row 에 걸리는가, index record 에 걸리는가?
- Record Lock 을 이해하기 위해 Index Record 를 어떻게 이해해야 하는가?
- 인덱스가 없거나 조건에 맞는 인덱스를 사용하지 못하면 왜 Lock 범위가 커지는가?
- Gap Lock, Next-Key Lock 은 왜 필요한가?
- S Lock, X Lock 과 Record Lock, Gap Lock 은 어떤 관계인가?
- INSERT 할 때는 어떤 Lock 이 걸리는가?
- Intention Lock, AUTO-INC Lock, Predicate Lock 은 왜 필요한가?
- Isolation Level 에 따라 Lock 동작은 어떻게 달라지는가?

## 학습을 통해 정리한 내용

### InnoDB Lock 을 이해하기 위한 출발점

InnoDB 의 Lock 을 이해할 때 가장 중요한 문장은 다음과 같다.

```text
InnoDB 의 row-level lock 은 실제로 index record 에 걸린다.
```

즉, 개발자가 보기에는 `id = 1` 인 row 를 잠근 것처럼 보이지만, InnoDB 내부적으로는 해당 row 를 찾기 위해 사용한 **index record** 를 잠그는 방식으로 동작한다.

그래서 Lock 을 분석할 때는 단순히 다음처럼 질문하면 부족하다.

```text
어떤 row 를 잠그는가?
```

더 정확한 질문은 다음과 같다.

```text
어떤 인덱스를 사용하는가?
그 인덱스에서 어떤 index record 를 스캔하는가?
그 과정에서 record 만 잠그는가, gap 도 잠그는가?
```

### Index Record 란?

테이블에 다음과 같은 데이터가 있다고 하자.

```sql
CREATE TABLE product (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100),
  price INT
);
```

```text
id | name     | price
---|----------|------
1  | Keyboard | 30000
2  | Mouse    | 15000
3  | Monitor  | 200000
```

`id` 가 Primary Key 이므로 InnoDB 는 `id` 기준의 clustered index 를 가진다.

```text
PRIMARY INDEX

[id=1]
[id=2]
[id=3]
```

여기서 `[id=1]`, `[id=2]`, `[id=3]` 각각이 index record 이다.

Secondary Index 가 있다면 index record 는 또 생긴다.

```sql
CREATE INDEX idx_product_price ON product(price);
```

```text
PRICE_INDEX

[price=15000, id=2]
[price=30000, id=1]
[price=200000, id=3]
```

즉 하나의 row 는 여러 인덱스에 여러 index record 로 표현될 수 있다.

### Record Lock 이란?

Record Lock 은 index record 에 거는 Lock 이다.

예를 들어 다음 SQL 이 실행된다고 하자.

```sql
START TRANSACTION;

SELECT *
FROM product
WHERE id = 1
FOR UPDATE;
```

`id` 는 Primary Key 이므로 InnoDB 는 Primary Index 에서 `id = 1` 인 index record 를 찾는다.

```text
PRIMARY INDEX

[id=1]  <- X Record Lock
[id=2]
[id=3]
```

이 상태에서 다른 트랜잭션이 같은 row 를 수정하려고 하면 대기한다.

```sql
UPDATE product
SET price = 40000
WHERE id = 1;
```

개발자 관점에서는 `id = 1 row 가 잠겼다` 라고 말할 수 있다.

하지만 InnoDB 내부 관점에서는 `PRIMARY INDEX 의 id = 1 index record 에 lock 이 걸렸다` 라고 이해하는 것이 더 정확하다.

### 인덱스가 없는 경우에는 Lock 을 걸지 않는가?

그렇지 않다. Lock 은 걸린다.

InnoDB 테이블은 clustered index 기반으로 데이터를 저장한다.

```text
1. Primary Key 가 있으면 Primary Key 로 clustered index 구성
2. Primary Key 가 없으면 NOT NULL Unique Index 를 clustered index 로 선택
3. 그것도 없으면 InnoDB 가 숨겨진 clustered index 를 내부적으로 생성
```

따라서 명시적인 인덱스가 전혀 없어도 InnoDB 는 내부적으로 row 를 식별할 수 있는 구조를 가진다.

문제는 **검색 조건에 맞는 인덱스가 없을 때**이다.

```sql
UPDATE product
SET price = price + 1000
WHERE status = 'ON_SALE';
```

`status` 인덱스가 없다면 InnoDB 는 `status = 'ON_SALE'` 인 row 가 어디 있는지 바로 알 수 없다. 결국 Primary Index 를 따라 많은 row 를 스캔해야 한다.

```text
PRIMARY INDEX

[id=1] 확인 -> status = ON_SALE -> update 대상
[id=2] 확인 -> status = SOLD_OUT -> 대상 아님
[id=3] 확인 -> status = ON_SALE -> update 대상
[id=4] 확인 -> status = ON_SALE -> update 대상
[id=5] 확인 -> status = SOLD_OUT -> 대상 아님
```

`UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` 는 단순 조회가 아니라 변경 가능성이 있는 Current Read 이다.

따라서 조건을 판단하기 위해 스캔한 record 들에도 Lock 영향이 생길 수 있다.

즉 인덱스가 없다는 것은 다음을 의미한다.

```text
Lock 을 못 건다
```

가 아니라,

```text
조건에 맞는 row 를 찾기 위해 더 많이 스캔하고,
그만큼 더 넓은 범위에 Lock 영향이 생길 수 있다.
```

인덱스가 있으면 상황이 달라진다.

```sql
CREATE INDEX idx_product_status ON product(status);
```

```text
STATUS_INDEX

[ON_SALE, id=1]
[ON_SALE, id=3]
[ON_SALE, id=4]
[SOLD_OUT, id=2]
[SOLD_OUT, id=5]
```

이제 InnoDB 는 `ON_SALE` 범위만 찾아가면 된다.

```text
[ON_SALE, id=1] 확인
[ON_SALE, id=3] 확인
[ON_SALE, id=4] 확인
```

스캔 범위가 작아지면 Lock 영향도 작아진다.

### 여러 인덱스가 있을 때 어떤 index record 에 Lock 이 걸리는가?

예를 들어 다음 인덱스들이 있다고 하자.

```text
PRIMARY_KEY_INDEX(id)
PRICE_INDEX(price)
NAME_INDEX(name)
```

그리고 다음 SQL 을 실행한다.

```sql
UPDATE product
SET name = 'New Name'
WHERE price = 10000;
```

옵티마이저가 `PRICE_INDEX` 를 사용한다면, InnoDB 는 먼저 `PRICE_INDEX` 에서 `price = 10000` 인 index record 를 찾는다.

```text
PRICE_INDEX

[price=5000, id=2]
[price=10000, id=1]  <- 검색 과정에서 lock 대상
[price=20000, id=3]
```

하지만 실제 row 데이터는 clustered index, 즉 Primary Index 에 있다.

따라서 실제 row 를 변경하기 위해 Primary Index 의 record 에도 접근하고 Lock 을 잡는다.

```text
PRIMARY_INDEX

[id=1]  <- 실제 row 변경을 위한 lock 대상
```

또한 `name` 이 변경되고, `NAME_INDEX` 가 존재한다면 기존 name index entry 를 삭제하고 새로운 name index entry 를 삽입해야 한다.

이 과정에서도 추가적인 index 변경과 Lock 이 발생할 수 있다.

정리하면 다음과 같다.

```text
UPDATE 시에는
1. 검색에 사용한 index record/range 를 잠글 수 있고
2. 실제 row 가 있는 clustered index record 도 잠그며
3. 변경되는 컬럼이 secondary index 에 포함되어 있다면 해당 index 변경 과정에서도 Lock 이 발생할 수 있다.
```

### row 자체에는 Lock 을 잡지 않는가?

InnoDB 관점에서는 row 와 index record lock 이 완전히 별개라고 보기 어렵다.

InnoDB 의 clustered index leaf record 는 실제 row 데이터를 담고 있다.

```text
PRIMARY INDEX

[id=1, name='Keyboard', price=30000]
```

따라서 Primary Index 의 record 에 Lock 을 거는 것이 실질적으로 row 를 잠그는 역할을 한다.

즉 다음 두 표현은 관점의 차이다.

```text
개발자 관점:
id = 1 row 에 lock 이 걸렸다.

InnoDB 내부 관점:
PRIMARY INDEX 의 id = 1 index record 에 lock 이 걸렸다.
```

### S Lock 과 X Lock

S Lock 은 Shared Lock, 즉 공유 락이다.

```sql
SELECT *
FROM product
WHERE id = 1
FOR SHARE;
```

이 경우 다른 트랜잭션도 읽기 목적의 S Lock 은 함께 잡을 수 있다.

하지만 해당 row 를 수정하려는 X Lock 과는 충돌한다.

X Lock 은 Exclusive Lock, 즉 배타 락이다.

```sql
SELECT *
FROM product
WHERE id = 1
FOR UPDATE;
```

또는

```sql
UPDATE product
SET price = 40000
WHERE id = 1;
```

이 경우 해당 record 를 변경하거나 변경할 가능성이 있으므로 X Lock 을 잡는다.

다른 트랜잭션의 X Lock 또는 S Lock 과 충돌할 수 있다.

중요한 점은 S Lock, X Lock 과 Record Lock, Gap Lock 은 서로 다른 분류라는 것이다.

```text
Record Lock, Gap Lock, Next-Key Lock
-> 어디를 잠그는가?

S Lock, X Lock
-> 어떤 모드로 잠그는가?
```

따라서 다음처럼 조합해서 이해해야 한다.

```text
S Record Lock
X Record Lock
X Next-Key Lock
Gap Lock
Insert Intention Lock
```

### Gap Lock 이란?

Gap Lock 은 index record 사이의 공간을 잠그는 Lock 이다.

예를 들어 Primary Index 가 다음과 같다고 하자.

```text
[id=5]     [id=10]
```

이 사이에는 index 정렬 순서상 gap 이 있다.

```text
(5, 10)
```

다음 SQL 이 실행된다.

```sql
SELECT *
FROM product
WHERE id BETWEEN 5 AND 10
FOR UPDATE;
```

이때 InnoDB 는 `id = 5`, `id = 10` record 만 잠그는 것이 아니라, 그 사이의 gap 도 잠글 수 있다.

```text
[id=5]  gap  [id=10]
 lock   lock   lock
```

이유는 중간에 새로운 row 가 삽입되는 것을 막기 위해서이다.

```sql
INSERT INTO product(id, name, price)
VALUES (7, 'Mouse', 10000);
```

만약 이 insert 가 허용되면, 같은 트랜잭션 안에서 동일 범위를 다시 조회했을 때 처음에는 없던 `id = 7` row 가 나타날 수 있다.

이것이 Phantom Read 문제이다.

Gap Lock 은 기존 row 를 수정하지 못하게 막는 Lock 이라기보다는, **해당 index 범위 안에 새 record 가 삽입되는 것을 막는 Lock** 이다.

### 문자열이나 날짜 타입에서 Gap Lock 은 어떻게 동작하는가?

Gap 은 정수 사이의 수학적 간격이 아니다.

Gap 은 **인덱스 정렬 순서상 인접한 두 index record 사이의 공간**이다.

문자열 index 가 다음과 같다고 하자.

```text
NAME_INDEX

['Alice']
['Bob']
['Chris']
['Tom']
```

여기에도 gap 이 존재한다.

```text
(-infinity, 'Alice')
('Alice', 'Bob')
('Bob', 'Chris')
('Chris', 'Tom')
('Tom', +infinity)
```

다음 SQL 은 문자열 정렬 순서 기준으로 범위를 잠글 수 있다.

```sql
SELECT *
FROM user
WHERE name BETWEEN 'Bob' AND 'Tom'
FOR UPDATE;
```

문자열의 정렬 순서는 collation 의 영향을 받는다.

따라서 대소문자 구분 여부, 언어별 정렬 규칙에 따라 gap 의 기준도 달라질 수 있다.

날짜 타입도 마찬가지이다.

```text
[2026-01-01] gap [2026-02-01] gap [2026-03-01]
```

즉 Gap Lock 을 이해할 때 중요한 것은 값의 타입이 아니라 **인덱스의 정렬 순서**이다.

### Next-Key Lock 이란?

Next-Key Lock 은 Record Lock 과 Gap Lock 이 합쳐진 개념이다.

```text
Next-Key Lock = Gap Lock + Record Lock
```

예를 들어 index 가 다음과 같다고 하자.

```text
[id=10] [id=20] [id=30]
```

`id = 20` 에 대한 Next-Key Lock 은 보통 다음 범위를 의미한다.

```text
(10, 20]
```

즉 다음 두 가지를 함께 잠근다.

```text
10 과 20 사이의 gap
+
id = 20 index record
```

Next-Key Lock 은 InnoDB 의 기본 격리 수준인 `REPEATABLE READ` 에서 Phantom Read 를 방지하기 위해 중요하게 사용된다.

### INSERT 할 때는 어떤 Lock 이 걸리는가?

현재 Primary Index 가 다음과 같다고 하자.

```text
PRIMARY INDEX

[id=5]      [id=10]
```

여기에 `id = 7` 을 삽입한다.

```sql
INSERT INTO product(id, name, price)
VALUES (7, 'Mouse', 10000);
```

`id = 7` 은 아직 존재하지 않으므로 처음부터 `[id=7]` record 에 Record Lock 을 걸 수는 없다.

먼저 InnoDB 는 `id = 7` 이 들어갈 위치를 찾는다.

```text
[id=5]   여기에 삽입   [id=10]
```

즉 gap `(5, 10)` 에 들어간다.

이때 InnoDB 는 해당 gap 에 Insert Intention Lock 을 잡는다.

```text
[id=5]  (insert intention)  [id=10]
```

그리고 실제 record 가 삽입되면 새로 생긴 index record 에 X Record Lock 을 잡는다.

```text
[id=5] [id=7] [id=10]
          ^
      X Record Lock
```

정리하면 INSERT 는 대략 다음 흐름으로 이해할 수 있다.

```text
1. 삽입할 위치의 gap 에 Insert Intention Lock
2. 새로 삽입된 index record 에 X Record Lock
```

같은 gap 에 insert 하더라도 위치가 다르면 서로 막지 않을 수 있다.

```text
[id=5]   7   8   [id=10]
```

하지만 다른 트랜잭션이 이미 해당 gap 을 Gap Lock 또는 Next-Key Lock 으로 잠그고 있다면 insert 는 대기한다.

### Intention Lock 이란?

Intention Lock 은 table-level lock 이다.

하지만 실제 데이터를 보호하기 위한 주된 Lock 이라기보다는, **테이블 안의 row 에 Lock 을 잡을 예정이라는 표시**에 가깝다.

예를 들어 다음 SQL 이 실행된다.

```sql
UPDATE product
SET price = 10000
WHERE id = 1;
```

InnoDB 는 row 에 X Record Lock 을 잡기 전에 테이블에 IX Lock 을 잡는다.

```text
product table  <- IX Lock

[id=1] <- X Record Lock
[id=2]
[id=3]
```

Intention Lock 의 종류는 대표적으로 다음과 같다.

```text
IS Lock
-> 이 테이블 안의 어떤 row 에 S Lock 을 잡을 예정이다.

IX Lock
-> 이 테이블 안의 어떤 row 에 X Lock 을 잡을 예정이다.
```

Intention Lock 이 필요한 이유는 row-level lock 과 table-level lock 을 조율하기 위해서이다.

만약 Intention Lock 이 없다면, 어떤 트랜잭션이 테이블 전체 Lock 을 잡으려고 할 때 테이블 안의 모든 row lock 을 검사해야 할 수 있다.

```text
[id=1] lock 있나?
[id=2] lock 있나?
[id=3] lock 있나?
...
```

비효율적이다.

Intention Lock 이 있으면 테이블 레벨에서 빠르게 알 수 있다.

```text
이 테이블 안에는 이미 row lock 을 잡은 트랜잭션이 있다.
```

서로 다른 row 를 수정하는 트랜잭션들은 둘 다 IX Lock 을 잡을 수 있다.

```text
Tx A: table IX + id=1 X Record Lock
Tx B: table IX + id=2 X Record Lock
```

이 둘은 가능하다.

하지만 table 전체에 강한 Lock 을 잡으려는 요청과는 충돌할 수 있다.

### AUTO-INC Lock 이란?

AUTO-INC Lock 은 `AUTO_INCREMENT` 값을 생성할 때 사용하는 특수한 table-level Lock 이다.

예를 들어 다음 테이블이 있다.

```sql
CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL
);
```

여러 트랜잭션이 동시에 insert 한다고 하자.

```sql
INSERT INTO orders(user_id) VALUES (10), (11), (12);
```

```sql
INSERT INTO orders(user_id) VALUES (20), (21);
```

이때 InnoDB 는 각 row 에 auto increment 값을 안전하게 배정해야 한다.

```text
id = 1, 2, 3, 4, 5 ...
```

AUTO_INCREMENT 값은 아직 존재하지 않는 row 의 key 를 생성하는 문제이다.

따라서 일반적인 Record Lock 만으로는 충분하지 않다.

AUTO-INC Lock 은 auto increment counter 를 여러 트랜잭션이 동시에 건드릴 때 충돌하지 않도록 조율한다.

중요한 점은 auto increment 값이 반드시 연속된다는 보장이 없다는 것이다.

```text
Rollback
실패한 insert
동시 insert
innodb_autoinc_lock_mode 설정
```

등에 의해 중간에 gap 이 생길 수 있다.

따라서 auto increment PK 를 “비즈니스적으로 빈틈없는 순번”으로 사용하면 안 된다.

주문번호, 영수증 번호처럼 연속성이 중요한 값은 별도의 번호 발급 정책이 필요하다.

### Predicate Lock 이란?

Predicate Lock 은 주로 Spatial Index 에서 사용되는 특수한 Lock 이다.

일반 B-Tree Index 는 정렬 순서가 있다.

```text
1, 2, 3, 4, 5
Alice, Bob, Chris, Tom
```

그래서 Record Lock, Gap Lock, Next-Key Lock 을 적용할 수 있다.

하지만 Spatial Index 는 공간 데이터를 다룬다.

```sql
CREATE TABLE place (
  id BIGINT PRIMARY KEY,
  area GEOMETRY NOT NULL,
  SPATIAL INDEX(area)
);
```

공간 데이터에서는 단순히 “다음 key” 라는 개념이 잘 맞지 않는다.

특정 좌표, 사각형, 다각형이 서로 겹치는지 여부를 기준으로 검색하기 때문이다.

```sql
SELECT *
FROM place
WHERE ST_Intersects(area, ST_GeomFromText('POLYGON(...)'))
FOR UPDATE;
```

이런 경우에는 특정 record 사이의 gap 을 잠그는 것만으로는 phantom 을 막기 어렵다.

그래서 InnoDB 는 Spatial Index 에 대해 Predicate Lock 을 사용한다.

Predicate Lock 은 쉽게 말하면 다음과 같다.

```text
특정 row 나 gap 이 아니라,
특정 조건에 매칭될 수 있는 공간 범위를 잠근다.
```

일반적인 CRUD 에서는 자주 만나지 않지만, 위치 기반 서비스, 지도, 배송 반경, GIS 도메인에서는 중요하다.

### Isolation Level 과 Lock 의 관계

Isolation Level 은 트랜잭션들이 서로를 얼마나 강하게 격리할지 정하는 설정이다.

InnoDB 에서는 Isolation Level 에 따라 같은 SQL 이라도 Lock 전략과 범위가 달라질 수 있다.

#### READ COMMITTED

`READ COMMITTED` 는 커밋된 데이터만 읽는다.

```text
각 SELECT 마다 새로운 Read View 를 생성한다.
```

Lock 관점에서는 gap locking 이 줄어든다.

그래서 동시성이 좋아질 수 있다.

하지만 같은 트랜잭션 안에서 같은 범위를 다시 조회했을 때 새로운 row 가 보일 수 있다.

```text
Phantom Read 가능
```

단, `READ COMMITTED` 라고 해서 Gap Lock 이 완전히 사라지는 것은 아니다.

Foreign Key 검사나 Duplicate Key 검사 같은 경우에는 여전히 Gap Lock 이 사용될 수 있다.

#### REPEATABLE READ

MySQL InnoDB 의 기본 Isolation Level 이다.

```text
트랜잭션 안에서 첫 Snapshot Read 시점의 Read View 를 재사용한다.
```

Locking Read 나 UPDATE, DELETE 에서는 Phantom 을 막기 위해 Next-Key Lock 이 중요하게 사용된다.

```sql
SELECT *
FROM product
WHERE price BETWEEN 10000 AND 20000
FOR UPDATE;
```

이 경우 단순히 현재 존재하는 record 만 잠그는 것이 아니라, 해당 범위의 gap 도 잠글 수 있다.

```text
[10000] gap [15000] gap [20000]
   lock      lock      lock
```

따라서 다른 트랜잭션이 `price = 15000` 인 row 를 insert 하려고 하면 대기할 수 있다.

`REPEATABLE READ` 는 읽기 일관성과 phantom 방지에 더 강하지만, 그만큼 insert blocking 이나 deadlock 가능성이 커질 수 있다.

#### SERIALIZABLE

가장 강한 격리 수준이다.

일반 SELECT 도 더 강한 locking read 처럼 동작할 수 있다.

따라서 읽기와 쓰기의 충돌이 늘어나고 동시성은 낮아진다.

```text
정합성은 강해짐
동시성은 낮아짐
```

### Snapshot Read 와 Current Read 관점에서 Lock 이해하기

일반 SELECT 는 보통 Snapshot Read 이다.

```sql
SELECT *
FROM product
WHERE id = 1;
```

Snapshot Read 는 Read View 와 Undo Log 를 이용해 자신에게 보여야 하는 버전을 읽는다.

일반적으로 Lock 을 기다리지 않는다.

반면 다음 쿼리들은 Current Read 이다.

```sql
SELECT *
FROM product
WHERE id = 1
FOR UPDATE;
```

```sql
UPDATE product
SET price = 10000
WHERE id = 1;
```

```sql
DELETE FROM product
WHERE id = 1;
```

Current Read 는 최신 데이터를 기준으로 동작하며, 변경 가능성이 있기 때문에 Lock 을 획득해야 한다.

정리하면 다음과 같다.

```text
Snapshot Read
- 일반 SELECT
- Read View 기준으로 읽음
- Undo Log 를 따라 과거 버전을 읽을 수 있음
- 보통 Lock 을 기다리지 않음

Current Read
- SELECT FOR UPDATE
- SELECT FOR SHARE
- UPDATE
- DELETE
- 최신 데이터를 기준으로 읽음
- Lock 을 획득해야 함
```

## 프로젝트 적용

- 재고 차감 로직에서 `PESSIMISTIC_WRITE` 를 사용하면 DB 에서는 `SELECT ... FOR UPDATE` 와 유사한 Current Read 가 발생하고, 대상 record 에 X Lock 이 걸린다고 이해할 수 있다.
- 재고처럼 동시에 수정될 수 있는 값은 단순 Dirty Checking 만으로는 동시성 정합성을 보장하기 어렵다. 조회 시점과 수정 시점 사이에 다른 트랜잭션이 개입할 수 있기 때문이다.
- `WHERE id = ?` 처럼 Primary Key 기반 단건 수정은 Lock 범위가 비교적 좁다.
- `WHERE status = ?`, `WHERE expired_at < ?` 같은 조건으로 bulk update/delete 를 할 때는 해당 조건에 적절한 인덱스가 없으면 많은 record 를 스캔하고 더 넓은 Lock 영향을 만들 수 있다.
- 상품 상태 만료 처리, 대량 플래그 변경, 통계성 업데이트는 실행 계획을 확인하고 인덱스가 Lock 범위를 줄이는 데 충분한지 검토해야 한다.
- 범위 조건으로 `FOR UPDATE` 를 사용하는 경우 Gap Lock, Next-Key Lock 때문에 현재 존재하지 않는 row 의 insert 까지 막을 수 있음을 고려해야 한다.
- AUTO_INCREMENT PK 는 식별자로만 보고, 비즈니스적으로 연속된 주문번호나 정산번호로 사용하지 않는 것이 좋다.
- Deadlock 은 완전히 없애는 대상이라기보다 재시도 가능한 실패로 보고 애플리케이션 레벨에서 retry 전략을 고려해야 한다.

## 오늘의 결론

InnoDB Lock 은 단순히 “row 를 잠근다”라고 이해하면 부족하다.

핵심은 다음과 같다.

```text
InnoDB 는 index record 를 기준으로 Lock 을 건다.
```

그래서 Lock 문제를 분석할 때는 SQL 만 보면 안 되고, 반드시 인덱스와 실행 계획을 함께 봐야 한다.

특히 인덱스가 없거나 적절하지 않으면, InnoDB 는 조건에 맞는 row 를 찾기 위해 더 많은 index record 를 스캔하고, 그 결과 Lock 범위도 넓어질 수 있다.

MVCC 는 일반 SELECT 의 읽기 일관성을 제공하고, Lock 은 Current Read 와 쓰기 정합성을 보장한다.

결국 InnoDB 의 동시성 제어는 다음 두 축이 함께 동작하는 구조라고 이해할 수 있다.

```text
MVCC
-> 읽기를 최대한 막지 않기 위한 장치

Lock
-> 쓰기 충돌과 phantom 을 막기 위한 장치
```

## 다음 액션

- [ ] 실제 MySQL 환경에서 `SELECT ... FOR UPDATE` 로 Record Lock 확인하기
- [ ] 범위 조건에서 Gap Lock, Next-Key Lock 으로 INSERT 가 막히는 예제 실습하기
- [ ] 인덱스 유무에 따라 UPDATE 실행 계획과 Lock 범위가 어떻게 달라지는지 비교하기
- [ ] `READ COMMITTED` 와 `REPEATABLE READ` 에서 Gap Lock 동작 차이 실습하기
- [ ] Deadlock 발생 예제를 만들고 `SHOW ENGINE INNODB STATUS` 로 분석하기

## 면접 질문

- InnoDB 의 Record Lock 은 row 에 걸리는가, index record 에 걸리는가? Primary Key 기반 조회와 Secondary Index 기반 조회를 나누어 설명해주세요.
- 조건에 맞는 인덱스가 없는 UPDATE 문이 의도보다 많은 row 에 Lock 영향을 줄 수 있는 이유는 무엇인가요?
- `UPDATE product SET name = ? WHERE price = ?` 실행 시 `price` 에 Secondary Index 가 있다면 어떤 index record 들에 Lock 이 걸릴 수 있는지 설명해주세요.
- Gap Lock 은 왜 필요한가요? `id = 5`, `id = 10` 이 존재할 때 `id = 7` insert 와 연결해서 설명해주세요.
- Next-Key Lock 이 Record Lock 과 Gap Lock 의 조합이라는 말은 무슨 뜻인가요?
- 문자열 컬럼에 대한 Gap Lock 은 어떤 기준으로 적용되나요? collation 과 연결해서 설명해주세요.
- S Lock, X Lock 과 Record Lock, Gap Lock 은 서로 어떤 관계인가요?
- INSERT 시 아직 존재하지 않는 PK 값에 대해서는 어떻게 Lock 을 잡나요? Insert Intention Lock 과 X Record Lock 의 흐름으로 설명해주세요.
- Intention Lock 은 왜 필요한가요? Row Lock 과 Table Lock 의 조율 관점에서 설명해주세요.
- AUTO-INC Lock 은 왜 필요한가요? 일반 Record Lock 만으로 AUTO_INCREMENT 값을 안전하게 발급하기 어려운 이유를 설명해주세요.
- Predicate Lock 은 어떤 상황에서 사용되나요? Spatial Index 에서 Next-Key Lock 이 잘 맞지 않는 이유와 함께 설명해주세요.
- `READ COMMITTED` 와 `REPEATABLE READ` 에서 Gap Lock 사용 방식은 어떻게 달라지나요?
- 일반 SELECT 는 X Lock 이 걸린 row 를 왜 기다리지 않을 수 있는데, `SELECT FOR UPDATE` 는 왜 기다려야 하나요?
- JPA 의 `PESSIMISTIC_WRITE` 는 InnoDB 의 Current Read, X Lock 과 어떤 관계가 있나요?
