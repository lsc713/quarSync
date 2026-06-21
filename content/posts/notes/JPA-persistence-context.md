---
title: JPA 영속성 컨텍스트 — save 후 findById는 왜 SELECT를 안 하나
created: 2026-06-21
tags:
  - jpa
  - spring
  - persistence
  - transaction
  - backend
---

> [!summary] 핵심 한 줄
> 영속성 컨텍스트는 `EntityManager`가 트랜잭션 동안 들고 있는 **엔티티 작업 공간**이다. **1차 캐시 · 스냅샷 · 쓰기 지연 SQL 저장소**로 이뤄지고, **트랜잭션과 생명주기를 공유**한다. 그래서 같은 트랜잭션 안에서 `save()` 직후 `findById()`를 해도 DB로 SELECT가 안 나간다 — 답이 이미 1차 캐시에 있기 때문이다.

---

## 1. JpaRepository는 결국 EntityManager다

Spring Data JPA의 `JpaRepository`는 인터페이스고, 런타임 구현체는 `SimpleJpaRepository`다. 이 구현체는 내부에 `EntityManager`를 필드로 들고 있고, 우리가 부르는 메서드는 전부 거기로 위임된다.

- `save()` → `em.persist()` / `em.merge()`
- `findById()` → `em.find()`

그리고 그 `EntityManager`는 **트랜잭션 단위로 공유**된다. `@PersistenceContext`가 주입하는 건 실제 EM이 아니라, "지금 이 트랜잭션의 EM으로 위임하라"는 **공유 프록시**다. 결과적으로 **한 트랜잭션 안의 모든 리포지토리 호출은 같은 영속성 컨텍스트를 본다.**

> [!tip] 한 줄 직관
> 리포지토리는 EntityManager를 감싼 얇은 포장지다. 메서드 이름만 다를 뿐, 같은 EM·같은 영속성 컨텍스트를 공유한다.

---

## 2. save 후 findById가 SELECT를 안 날리는 이유

범인은 **1차 캐시(first-level cache)** 다.

```java
@Test
void save_and_findById() {
    User saved = userRepository.save(new User("juho"));

    assertThat(saved.getId()).isNotNull();

    User found = userRepository.findById(saved.getId()).orElseThrow();
    assertThat(found).isEqualTo(saved); // 같은 인스턴스
}
```

`persist` 시 엔티티는 `@Id`를 키로 **1차 캐시(Map)에 등록**된다. 이후 같은 트랜잭션에서 같은 id로 `find`하면, EntityManager는 **DB에 가기 전에 1차 캐시를 먼저 조회**한다. 키가 있으면 그 인스턴스를 그대로 반환한다 → DB SELECT 없음.

덤으로 **동일성(identity)** 도 보장된다. 같은 id를 두 번 조회하면 `==` 비교가 참인 **같은 인스턴스**가 나온다.

> [!note] SELECT 생략 ≠ INSERT 생략
> 여기서 생략되는 건 **조회 SELECT**다. 저장 자체의 INSERT가 언제 나가는지는 별개 문제이며, id 생성 전략에 따라 다르다([[#5. IDENTITY vs SEQUENCE — 쓰기 지연이 다른 이유]]).

> [!tip] 한 줄 직관
> `find`는 DB가 아니라 1차 캐시에 먼저 묻는다.

---

## 3. 트랜잭션 경계 = 1차 캐시 범위

왜 하필 "**같은** 트랜잭션"이 조건일까? 영속성 컨텍스트의 생명주기가 **트랜잭션 생명주기와 일치**하기 때문이다(Spring 기본 = 트랜잭션 범위 영속성 컨텍스트).

```
트랜잭션 시작 → EntityManager 바인딩 → (작업) → 커밋/롤백 시 flush 후 EM close → 1차 캐시 폐기
```

즉 **1차 캐시가 살아있는 구간 = 트랜잭션 구간**이다. 트랜잭션 밖(예: OSIV를 끈 상태의 컨트롤러)에서는 영속성 컨텍스트가 없으니 캐시 효과도, 지연 로딩도 기대할 수 없다.

> [!tip] 한 줄 직관
> 1차 캐시는 트랜잭션과 함께 태어나서 트랜잭션과 함께 죽는다.

---

## 4. REQUIRES_NEW로 트랜잭션을 분리하면

`REQUIRES_NEW`는 기존 트랜잭션을 **보류**하고 **새 트랜잭션**을 연다. 새 트랜잭션 = **새 EntityManager = 새 1차 캐시**다. 그래서:

- 바깥 트랜잭션에서 1차 캐시에 올려둔 엔티티는 **안쪽에서 보이지 않는다.** 안쪽에서 같은 id를 조회하면 **DB SELECT가 나간다.**
- 같은 row라도 **두 개의 인스턴스**가 생긴다 → 동일성이 깨진다.

> [!warning] 같은 데이터, 다른 인스턴스
> 바깥과 안쪽이 같은 row를 각자 들고 작업하면, 한쪽의 변경이 다른 쪽 스냅샷에 반영되지 않아 **dirty checking이 엇갈리거나 마지막 flush가 앞선 변경을 덮을 수 있다.** 트랜잭션을 굳이 분리할 때 항상 따져봐야 할 비용이다.

---

## 5. IDENTITY vs SEQUENCE — 쓰기 지연이 다른 이유

**쓰기 지연(transactional write-behind)** 이란, `persist` 시 바로 INSERT를 보내지 않고 **쓰기 지연 SQL 저장소에 모았다가 flush 시점에 한꺼번에** 내보내는 것이다. 여기서 JDBC 배치 INSERT도 가능해진다.

이 동작이 id 생성 전략에 따라 갈린다. 핵심은 **"INSERT를 하기 전에 id를 알 수 있느냐"** 다. Hibernate는 엔티티를 1차 캐시에 넣으려면 **키(id)가 먼저 있어야** 하기 때문이다.

- **SEQUENCE** — `persist` 시 DB 시퀀스를 호출해 **id만 미리 받아온다.** id를 손에 쥐었으니 엔티티를 1차 캐시에 등록하고 **INSERT는 flush로 미룰 수 있다** → 쓰기 지연 O, 배치 O.
- **IDENTITY** — id는 INSERT가 실행돼 `AUTO_INCREMENT`가 매겨져야 정해진다. id를 받으려면 어쩔 수 없이 **`persist` 즉시 INSERT를 날린다** → 그 엔티티에 대한 **쓰기 지연·JDBC 배치가 사실상 불가능**하다.

> [!warning] IDENTITY는 INSERT 배치가 안 된다
> `hibernate.jdbc.batch_size`를 켜도 IDENTITY 전략 엔티티는 INSERT가 배치되지 않는다. 대량 삽입 성능이 중요하면 SEQUENCE(또는 TABLE) 전략을 고려한다.

> [!tip] 한 줄 직관
> id를 미리 알 수 있으면(SEQUENCE) 쓰기를 미룰 수 있고, INSERT해야만 id를 알면(IDENTITY) 미룰 수 없다.

---

## 6. 영속성 컨텍스트 3요소 자료구조

영속성 컨텍스트 안은 크게 세 가지로 구성된다.

| 요소 | 자료구조 | 역할 |
|---|---|---|
| **1차 캐시** | `Map` (키 = 타입 + `@Id`, 값 = 엔티티) | 동일성 보장 · 조회 SELECT 생략 |
| **스냅샷** | 영속 시점 필드 값 **복사본** | flush 때 현재 값과 비교 → **dirty checking** |
| **쓰기 지연 SQL 저장소** | INSERT/UPDATE/DELETE **큐**(Hibernate `ActionQueue`) | flush 때 순서대로 실행 |

> [!tip] 한 줄 직관
> Map으로 "있나 없나", 스냅샷으로 "뭐가 바뀌었나", 큐로 "언제 내보낼까"를 관리한다.

---

## 7. 한눈에 정리

| 질문 | 답 |
|---|---|
| save 후 findById가 SELECT 안 하는 이유 | 1차 캐시(Map)에서 id로 히트 |
| 왜 "같은 트랜잭션"이어야 하나 | 영속성 컨텍스트 = 트랜잭션과 생명주기 공유 |
| REQUIRES_NEW로 분리하면 | 새 EM·새 1차 캐시 → SELECT 발생, 인스턴스 분리 |
| IDENTITY가 쓰기 지연 안 되는 이유 | id를 INSERT 후에야 알 수 있어 즉시 INSERT |
| SEQUENCE가 쓰기 지연 되는 이유 | id를 시퀀스로 미리 확보 → INSERT 지연·배치 가능 |
| 영속성 컨텍스트 3요소 | 1차 캐시(Map) · 스냅샷(복사본) · 쓰기 지연 큐 |

---

## 한계 / 확인 필요

- **flush 시점**: 커밋 직전 · JPQL 실행 전 auto-flush · 명시적 `flush()` — 정확한 트리거 정리
- **OSIV(Open Session In View)** on/off가 영속성 컨텍스트 범위에 미치는 영향
- **`merge` vs `persist`** 의 1차 캐시 동작 차이
- **2차 캐시(L2)** 와 1차 캐시의 관계

---

## 관련 노트

- [[DDD와 헥사고날 아키텍처]] — 애그리거트 = 트랜잭션 경계
- [[결제 취소 멱등성 설계]] — 정합성과 트랜잭션 설계
