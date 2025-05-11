# Lock (락)

여러 스레드나 프로세스가 동시에 공유 자원(임계 영역)에 접근할 경우, 데이터의 무결성과 일관성을 보장하기 위해 **하나의 실행 단위만 해당 자원에 접근하도록 제한하는 동기화 메커니즘**이다.

즉, 동시성 또는 병렬성을 갖는 실행 환경에서 **경쟁 조건(Race Condition)**을 방지하는 역할을 수행한다.

---

## 락이 필요한 이유: Producer-Consumer Problem

- 다수의 프로세스/스레드가 동일한 자원에 동시에 접근하게 되면, 예기치 못한 데이터 손상이 발생해서 데이터의 일관성이 깨질 수 있다.
- 이런 상황을 방지하고자 자원 접근을 순차화하기 위해 락(Lock)을 사용한다.

---

## 실무에서 락이 필요한 예시

- 주문 처리 시 재고 차감 로직
- 결제 API 중복 호출 방지
- 쿠폰 선착순 지급 로직
- 포인트 잔액 차감

---

## 자주 사용하는 Lock

| 종류             | 방식                      | 장점            | 단점                     |
| -------------- | ----------------------- | ------------- | ---------------------- |
| Database Lock  | Row-level / Table-level | 일관성 보장, 내장 기능 | 느릴 수 있음, 확장 어려움        |
| Redis Lock     | 단일 분산 락 / Redlock       | 빠름, 간단함       | 장애 대응 어려움, 데이터 불일치 가능성 |
| In-Memory Lock | 서버 단일 프로세스 내            | 빠르고 간단        | 분산 환경에서 불가             |

---

# Database Lock

## Row-Level Lock

트랜잭션 단위로 락을 걸 수 있다. `UPDATE`, `DELETE`는 자동으로 row-level 락이 걸리지만, `SELECT`는 그렇지 않기 때문에 조건 판단 후 갱신할 경우 수동으로 락을 걸어주어야 한다.

### `SELECT ... FOR UPDATE`

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- application logic: if balance >= 100
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

**필요한 이유**:
- select1 -> (검증1)
- select2 -> (검증2)
- 업데이트1 -> 업데이트2
- \=> 조건이 충족되지 않아도 동시에 업데이트될 수 있음 (예: 잔액 음수 발생 가능)

| 상황                 | 필요 여부                      |
| ------------------ | -------------------------- |
| 단순 UPDATE / DELETE | ❌ 자동 락                     |
| 읽고 → 조건 판단 → 갱신    | ✅ 필요 (`SELECT FOR UPDATE`) |
| 여러 row 조회 후 일부 갱신  | ✅ 필요                       |
| 읽기만 할 때 (`SELECT`) | ❌ 필요 없음                    |

> 💡 인덱스를 타지 않으면 전체 row를 탐색하기 때문에 row-level이 아닌 table-level lock과 동일하게 lock이 걸려서 성능 저하가 발생
> \
> 💡 격리 수준이 `READ COMMITTED` 이상일 때 `SELECT FOR UPDATE`가 의도대로 작동합니다.

---

## Table-Level Lock

전체 테이블 단위로 락을 걸어 마이그레이션, 배치 작업 등에 사용한다.

```sql
LOCK TABLE users WRITE;  -- 읽기/쓰기 모두 막음
LOCK TABLE users READ;   -- 쓰기만 막고 여러 읽기는 허용

UNLOCK TABLES; -- lock 해제
```

---

# Redis Lock (분산 락)

## 개요

Redis는 메모리 기반 Key-Value 저장소이지만, 명령의 단일 스레드 처리와 Lua 스크립트를 통해 **원자적 연산**을 보장할 수 있다.\
이 방법을 활용하여 분산 락을 구현할 수 있다.


## 단순 분산 락 구현

```bash
SET resource_name unique_value NX PX 3000
```

- `NX`: 존재하지 않을 때만 설정
- `PX`: 만료 시간(ms)
- 락 해제는 unique\_value를 검증 후 삭제 (루아 스크립트 사용)

## Redlock 알고리즘

멀티 Redis 인스턴스를 이용한 고가용성 락 알고리즘

- N개의 Redis 노드 중 과반수에서 성공 시 락 획득
- 각 노드에 동일한 key와 value를 제한 시간 내 저장
- 장애 내성 확보 (단일 Redis 장애 시에도 안전)

---

## 단일 서버 환경에서는?

- Redis를 사용하지 않아도, 내부에 `HashMap` 등으로 락 구현 가능
- 단, **멀티 스레드 환경에서는 싱글톤으로 관리되어야 함**

---

# 정리: 언제 어떤 락을 써야 하나?

| 환경             | 사용 권장 락                               |
| -------------- |---------------------------------------|
| 단일 서버, 단일 프로세스 | In-memory Lock (e.g. HashMap)         |
| 단일 서버, 멀티 스레드  | 싱글톤 형태의 In-memory Lock                |
| 다중 서버 (분산 환경)  | Redis Lock (Redlock)                  |
| 트랜잭션 기반 시스템    | DB Row/Table Lock (SELECT FOR UPDATE) |

---

# 추가: 네트워크 수준 동시성 제어란?

- API Gateway, Load Balancer, Proxy 서버에서 **Request ID 기반 라우팅/차단/순서 제어**를 통해 특정 요청만 통과시키는 로직도 넓은 의미의 동시성 제어로 볼 수 있음
- 이 경우 어플리케이션 레벨의 락보다 **오버헤드가 적고**, 미리 필터링 가능

---
