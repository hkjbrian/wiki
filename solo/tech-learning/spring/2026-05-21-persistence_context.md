# 기술 학습 기록

## 주제
> Persist Context 와 JPA 에서 사용하는 실제 구현체

## 핵심 질문
- bulk query vs Dirty Checking 언제 사용해야 하나?
- 영속성 컨텍스트의 생명주기에 대한 이해
- Persistable 인터페이스란?
- JPQL 과 영속성 컨텍스트
- Lazy Loading 으로 설정한 엔티티는 어떻게 처리되는가 ? 

## 학습을 통해 정리한 내용

**bulk query vs Dirty Checking**

먼저 bulk 쿼리란 JPQL 문법으로 영속성 컨텍스트를 거치지 않고 DB 에서 직접 쿼리를 실행하는 것을 말하는 것에 가깝다.

Dirty Checking 기반 :
- 특정 엔티티 1개 또는 소수 엔티티를 수정할 때
- 도메인 메서드, 검증, 상태 전이 규칙을 따르고 싶을 때
- 코드의 의도가 "객체 상태 변경" 인 경우

bulk : 
- 많은 row 를 한 번에 수정할 때
- 단순 상태값 일괄 변경, 만료 처리와 같이 DB 수준에서 바로 처리하는게 나은 경우

**엔티티 생명주기의 이해**

크게 4가지 상태를 갖는다.

```text
비영속 transient
영속 managed
준영속 detached
삭제 removed
```

- 비영속
  - 그냥 new 로 만들어진 자바 객체를 말하며 JPA 가 모른다.
  - id 값이 없는 경우, isNew() = true 인 객체

- 영속
  - EntityManager 또는 Repository 를 통해 영속성 컨텍스트에서 관리되고 있는 객체이다.

- 준영속
  - 영속성 컨텍스트가 관리했었지만 현재는 관리하지 않는 객체
  - 대표적으로 다음과 같은 경우 detached 엔티티이다.

```text
em.clear();
em.detach(entity);
em.close();
transaction 종료 후 외부에서 엔티티 사용
```

  - 다시 영속 상태로 돌리고 싶다면 merge() 를 사용할 수 있다. 
  - 다만, 이 때 merge() 메서드의 반환 객체는 파라미터로 전달한 객체가 아닌 영속성 컨텍스트가 관리하는 별도 영속 객체이다.


- 삭제
  - 영속화 엔티티 중 삭제 예정인 것들, flush 시점에 delete SQL 이 나간다.

**Persistable 인터페이스는 왜 필요할까?**
SimpleJpaRepository.save() 구현을 보면 다음과 같다.
```java
@Transactional
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        entityManager.persist(entity);
        return entity;
    } else {
        return entityManager.merge(entity);
    }
}
```

즉,
```text
새 엔티티라고 판단됨
→ persist()
→ managed 상태로 등록
→ flush 시 insert

기존 엔티티라고 판단됨
→ merge()
→ 상태를 managed 객체에 복사
→ flush 시 update 가능
```

Spring Data JPA 가 새 엔티티인지 판단하는 기준은 다음과 같다.
- @Version 필드가 있으면 version 이 null 인지 확인
- 없으면 id 가 null 인지 확인
- Persistable 을 구현하면 isNew() 결과로 판단

그래서 직접 할당 id 를 쓰는 경우에는 Persistable 을 구현하여 isNew() 판단을 명시하기도 한다.

**JPQL 과 영속성 컨텍스트**
JpaRepository 를 사용하거나 EntityManager 를 활용하는 경우 조회 대상을 영속성 컨텍스트에 관리해준다.

JPQL 의 경우에는 SQL 을 실행하는 것처럼 보이는데, 이 경우에는 영속성 컨텍스트가 관여하지 않을까 ? 

결론적으로 JPQL 로 조회해도 1차 캐시를 사용한다. 다만 em.find() 와는 동작 방식이 다르다.

em.find(Member.class, 1L)는 먼저 1차 캐시를 확인한다.

```text
1차 캐시에 Member#1 있음
→ DB 조회 없이 바로 반환
```

반면 JPQL은 보통 SQL을 실행한다.
```text
em.createQuery(
"select m from Member m where m.id = :id",
Member.class
).setParameter("id", 1L).getSingleResult();
```

이 경우 흐름은 대략:
```text
JPQL을 SQL로 변환
→ DB에 select 실행
→ 결과 row의 id 확인
→ 영속성 컨텍스트에 같은 EntityKey가 있는지 확인
→ 있으면 기존 managed 인스턴스 반환
→ 없으면 새 엔티티 생성 후 managed 등록
```
즉, JPQL 은 SQL 실행 자체를 생략하진 못하지만, 결과를 엔티티로 만드는 과정에서 1차 캐시 / 동일성 보장 규칙을 따른다.

**Lazy Loading 되는 엔티티의 이해**

FetchType.LAZY 로 설정해두는 경우, 연관 엔티티를 처음부터 조회하지 않고, 실제로 값에 접근할 때 조회하게 된다.

```java
@Entity
class Member {
    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}

Member member = memberRepository.findById(id).orElseThrow();
```
이 때 team 은 프록시 객체이다.
- id 는 알고 있다.
- Session / 영속성 컨텍스트와 연결되어 있다.
- 초기화 여부 : false

```java
member.getTeam().getName();
```
처럼 실제 값에 접근하는 경우에 프록시가 초기화된다.

흐름은 대략 다음과 같다.
```text
teamProxy.getName() 호출
→ 프록시의 interceptor가 호출을 가로챔
→ 아직 초기화되지 않았는지 확인
→ 연결된 Hibernate Session이 열려 있는지 확인
→ 영속성 컨텍스트 1차 캐시에 Team#id가 있는지 확인
→ 있으면 그 상태를 사용
→ 없으면 select team ... where id = ? 실행
→ 조회 결과로 엔티티 상태 로딩
→ 프록시 내부 target/상태 초기화
→ teamProxy.getName() 결과 반환
```

여기서 중요한 포인트는 **프록시 자체**가 영속성 컨텍스트에 등록된다는 점이다.

## 프로젝트 적용

- 단순 수정 로직에서는 bulk query 보다 영속 상태 엔티티를 조회한 뒤 도메인 메서드로 상태를 변경하는 Dirty Checking 방식을 우선 고려한다.
- 대량 상태 변경, 만료 처리, 단순 플래그 변경처럼 엔티티별 도메인 로직이 필요 없는 경우에는 bulk query 를 사용할 수 있다.
- bulk query 사용 후에는 영속성 컨텍스트와 DB 상태 불일치가 발생할 수 있으므로 `clearAutomatically = true`, `flushAutomatically = true` 옵션을 상황에 따라 고려한다.
- Lazy Loading 이 필요한 연관관계는 서비스 계층의 `@Transactional(readOnly = true)` 범위 안에서 접근하거나, fetch join / EntityGraph / DTO 조회로 필요한 데이터를 명시적으로 가져온다.

## 오늘의 결론

JPA 에서 중요한 것은 엔티티 객체 자체가 아니라, 해당 객체가 현재 영속성 컨텍스트에 의해 관리되고 있는지 여부이다.

영속 상태 엔티티는 1차 캐시, 동일성 보장, Dirty Checking, Lazy Loading 의 대상이 된다. 반면 bulk query 는 영속성 컨텍스트를 우회해 DB 에 직접 SQL 을 실행하므로, 이미 로딩된 엔티티와 DB 사이에 불일치가 발생할 수 있다.

Spring Data JPA 의 `save()` 는 단순 저장 메서드가 아니라, 새 엔티티 여부에 따라 `persist()` 또는 `merge()` 를 선택하는 메서드이다. 따라서 엔티티 식별자 생성 전략과 `isNew()` 판단 기준을 이해하는 것이 중요하다.

## 다음 액션

- [ ] `SimpleJpaRepository.save()` 의 `isNew()` 판단 흐름을 실제 코드로 다시 확인하기
- [ ] `persist()` 와 `merge()` 의 차이를 테스트 코드로 비교하기
- [ ] bulk update 이후 `clearAutomatically = true` 적용 전후 차이 확인하기
- [ ] Lazy Loading 프록시 초기화 시점과 `LazyInitializationException` 발생 조건 실험하기
- [ ] `@Transactional(readOnly = true)` 범위 안팎에서 Lazy Loading 동작 비교하기

## 면접 질문

- JPA 엔티티의 생명주기 4가지를 설명해주세요.
- `persist()` 와 `merge()` 의 차이는 무엇인가요?
- Spring Data JPA 의 `save()` 는 내부적으로 어떻게 동작하나요?
- `Persistable` 인터페이스는 언제 필요한가요?
- JPQL 로 조회한 엔티티도 영속성 컨텍스트에서 관리되나요?
- `em.find()` 와 JPQL 조회는 1차 캐시를 사용하는 방식이 어떻게 다른가요?
- bulk update 가 영속성 컨텍스트를 우회한다는 말은 무슨 뜻인가요?
- bulk update 후 `clearAutomatically = true` 가 필요한 이유는 무엇인가요?
- Lazy Loading 프록시는 언제 초기화되나요?
- `LazyInitializationException` 은 왜 발생하나요?
