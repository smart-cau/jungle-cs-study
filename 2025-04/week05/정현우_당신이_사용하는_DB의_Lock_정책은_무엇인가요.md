## Intro

  오늘 주제는 바로 “Lock” 입니다! 이번에는 Lock과 관련된 개념 주제를 선정하는데 정말로 애를 먹었습니다… Java에서 Lock을 사용하는 다양한 방식과 장단점에 대해서 설명할까, 아니면 Transaction의 ACID 속성에서 Isolation(격리성)의 다양한 수준과 해당 수준이 보장되는 DB별 방법에 대해서 설명할까 등… 이런 주제들에 대해 고민했지만 이 녀석들에 대해 쓰려니 내용도 넘 어렵고 글도 길어져 대신 뭘 할까 참 많은 고민을 했습니다…

  고민해보니 역시 주제는 내가 직접 경험해본게 좋겠다 싶어 제가 개발한 [주변시위 Now] 서비스의 ‘시위 응원하기’을 떠올렸습니다.
  <p align="center">
  <img src="https://github.com/user-attachments/assets/2cfd320f-67ee-4e20-8f40-b67894e95dd8" width="320">
  </p>

  시위 응원하기 기능은 아주 단순합니다. 응원하고 싶은 시위 id에 따라, 해당 id의 cheerCount라는 값을 1씩 올려줍니다.

```java
@Modifying
@Query(
	"UPDATE ProtestCheerCount pc SET pc.cheerCount = pc.cheerCount + 1 WHERE pc.protestId = :protestId"
)
Integer incrementCheerCount(@Param("protestId") Long protestId);
```

JPQL로 작성한 위 쿼리는 여러 트랜잭션이 동시에 `incrementCheerCount`를 요청해도, 동시성 문제 없이 정상적으로 잘 작동합니다. 어떻게 이런 일이 가능할까요? 운이 좋아서일까요? JPA에서 동시성 관리까지 해줘서일까요? 현재 사용하고 있는 PostgreSQL DB 덕분일까요? 정답은 바로 PostgreSQL에서 `update`, `delete` 문 등은 자체적으로 Row Lock을 사용하기 때문입니다. 지금부터 Lock의 개념과 PSQL에서 update, delete 문 사용 시 주의점 등에 대해 알아봅시다.

## Lock이란?

Lock이란, 여러 실행흐름(threads, transactions)이 하나의 공유자원을 사용할 때 생길 수 있는 문제를 방지하기 위한 방법 중 하나입니다. Lock을 설명할 때 가장 많이 쓰이는 비유는 좀 비위 상하는 거긴 하지만 ‘화장실’ 예시입니다. 배가 아픈 여러 사람(thread, transaction)이 하나의 변기칸(공유자원)을 공유해야 한다고 생각해봅시다. 만약 화장실 문의 잠금장치(Lock)가 작동하지 않는다면, 한 사람이 큰 일을 치루고 있는데 더 급한 사람이 볼 일 보고 있는 사람의 위에 배출을 하는 대참사가 발생할 수 있을 것입니다… 잠금장치(Lock)이라는 안전장치가 있기에 한 번에 한명 씩, 안전함을 느끼며 편히 볼 일을 볼 수 있는거죠.
<p align="center">
  <img src="eye-on-cheer.gif" width="320">
  </p>

  ## PostgreSQL의 UPDATE 문

### PostgreSQL에서의 UPDATE문 사용 시 Lock 기본 정책

PostgreSQL에서 UPDATE 문을 실행하면 기본적으로 **Row-level의 Exclusive Lock**을 획득합니다. 우리의 화장실 비유를 이어가자면, 수정하려는 각 화장실 칸(Row)에 "청소 중" 팻말(Exclusive Lock)을 걸어둔 것과 같습니다. 이 팻말이 있는 동안에는 다른 사람이 그 칸을 사용할 수 없죠.

즉, UPDATE 문이 처리되는 동안 해당 행은 다른 트랜잭션에서 수정하거나 삭제할 수 없으며, 트랜잭션 격리 수준에 따라 읽기도 제한될 수 있습니다.

### 예시

```sql
-- 트랜잭션 1
BEGIN;
UPDATE protest_cheer_count SET cheer_count = cheer_count + 1 WHERE protest_id = 1;
-- 이 시점에서 protest_id가 1인 row에 Exclusive Lock이 걸립니다-- 트랜잭션 2 (동시에 실행)
BEGIN;
UPDATE protest_cheer_count SET cheer_count = cheer_count + 1 WHERE protest_id = 1;
-- 이 쿼리는 트랜잭션 1이 COMMIT 또는 ROLLBACK될 때까지 대기합니다
```

트랜잭션 1이 행에 대한 독점 잠금을 유지하는 동안 트랜잭션 2는 해당 행을 수정하기 위해 대기열에 들어갑니다. 트랜잭션 1이 완료되면 트랜잭션 2가 실행되어 값이 올바르게 증가합니다.

### 의도치 않은 Table Lock 사용하게 되는 예시와 설명

하지만 잘못된 쿼리 작성으로 "화장실 칸 하나"가 아닌 "화장실 전체"에 청소 중 팻말을 걸어버리는(Table Lock) 경우가 있습니다. 가장 흔한 예는 인덱스가 없는 컬럼으로 조건을 지정하는 경우입니다.

### 예시

```sql
-- protest_name에 인덱스가 없는 경우
UPDATE protest_cheer_count SET cheer_count = cheer_count + 1
WHERE protest_name = '대학로 집회';
```

위 쿼리에서 `protest_name` 컬럼에 인덱스가 없다면, PostgreSQL은 모든 행을 검사해야 합니다(Full Table Scan). 이 과정에서 PostgreSQL은 테이블 전체에 Lock을 걸거나, 최소한 모든 행에 순차적으로 Lock을 걸게 됩니다.

또 다른 예로는 WHERE 절이 없는 UPDATE 문입니다:

```sql
-- 모든 행을 업데이트
UPDATE protest_cheer_count SET status = 'VERIFIED';
```

이 경우 PostgreSQL은 Table Lock을 걸게 됩니다.

### Table Lock 사용의 문제점

Table Lock은 마치 "화장실 전체를 폐쇄했다"는 팻말을 걸어놓는 것과 같습니다. 다른 모든 트랜잭션들은 멀쩡한 칸도 사용하지 못하고 밖에서 기다려야 합니다.

구체적인 문제점은:

1. **처리량 감소**: 동시에 여러 트랜잭션을 처리할 수 없어 애플리케이션 전체 성능이 저하됩니다.
2. **확장성 제한**: 사용자가 늘어나면 병목 현상이 더 심해집니다.
3. **데드락 위험 증가**: 여러 트랜잭션이 서로 다른 테이블에 Lock을 걸고 상대방의 Lock이 해제되기를 기다리는 상황이 발생할 수 있습니다.

### 해결 방법

1. **인덱스 활용**: 자주 조회/수정 조건으로 사용되는 컬럼에는 인덱스를 생성합니다.
    
    ```sql
    CREATE INDEX idx_protest_name ON protest_cheer_count(protest_name);
    ```
    
2. **작은 단위로 나누기**: 대량 업데이트가 필요하다면, 작은 배치로 나누어 실행합니다.
    
    ```sql
    -- 한 번에 1000건씩 처리
    UPDATE protest_cheer_count SET status = 'VERIFIED'
    WHERE id IN (SELECT id FROM protest_cheer_count WHERE status != 'VERIFIED' LIMIT 1000);
    ```

## MySQL에서의 UPDATE 문

MySQL(InnoDB)과 PostgreSQL의 UPDATE 문 Lock 정책에는 몇 가지 중요한 차이점이 있습니다.

### PostgreSQL과의 차이점 설명

1. **인덱스 사용에 따른 Lock 범위**:
    - PostgreSQL: 인덱스 유무와 관계없이 일반적으로 영향받는 행에만 Lock을 겁니다. 다만 Full Table Scan이 필요한 경우 모든 행을 순회하며 Lock을 검.
    - MySQL(InnoDB): 인덱스를 사용하는 경우 해당 행만 Lock, 인덱스를 사용할 수 없는 경우 테이블 전체 또는 넓은 범위에 "Gap Lock"을 포함한 다양한 Lock을 겁니다.
2. **Gap Lock의 존재**:
    - PostgreSQL: Gap Lock 개념이 없습니다.
    - MySQL: REPEATABLE READ(기본 격리 수준)에서는 Gap Lock을 사용합니다. 이는 조건에 해당하는 행들 사이의 "간격"에도 Lock을 걸어 다른 트랜잭션이 그 사이에 데이터를 삽입하지 못하게 합니다.

MySQL에서 Gap Lock을 화장실 비유로 설명하자면, 사용 중인 칸뿐만 아니라 "빈 칸에도 미리 예약"을 걸어두는 것과 같습니다. 누군가 새로운 칸을 만들어 끼워넣을 수 없게 하는 거죠.

예를 들어, MySQL에서 다음 쿼리를 실행하면:

```sql
-- protest_id에 인덱스가 있는 경우, MySQL의 REPEATABLE READ 격리 수준에서
UPDATE protest_cheer_count SET cheer_count = cheer_count + 1
WHERE protest_id BETWEEN 10 AND 20;
```

protest_id가 10~20 사이인 행들에 Lock을 걸고, 추가로 10 미만의 가장 가까운 값과 20 초과의 가장 가까운 값 사이의 간격(gap)에도 Lock을 겁니다. 이렇게 하면 다른 트랜잭션이 protest_id가 11, 12, ... 등의 새 행을 INSERT하지 못합니다.

## 결론

자신이 사용하는 DB의 Lock 정책을 아는 것은 매우 중요합니다. 마치 여러 사람이 동시에 사용하는 화장실의 규칙을 모르고 들어갔다가는 민망한 상황이 발생할 수 있는 것처럼, DB의 Lock 정책을 모르고 쿼리를 작성했다가는 예상치 못한 성능 문제나 교착 상태를 만날 수 있습니다.

특히 PostgreSQL과 MySQL은 같은 SQL 문법을 사용하더라도 내부적으로 다른 Lock 정책을 적용하기 때문에, DB를 변경하는 과정에서 예상치 못한 동시성 문제가 발생할 수 있습니다.

결국, 성능을 고려하는 개발자가 되기 위해서는:

1. 자신이 사용하는 DB의 Lock 정책 기본 원리를 이해하세요.
2. 성능 중요 쿼리의 실행 계획(EXPLAIN)을 확인하는 습관을 들이세요.
3. 대량의 데이터를 다루는 경우, Lock 전략을 명시적으로 고려한 쿼리를 작성하세요.

위 사항에 대해 잘 파악하고 있다면, DB 병목으로 인한 디버깅 시간을 줄이는데 도움이 될 수 있을겁니다.

## 후기

사실 위 내용을 제대로 설명하기 위해 Transaction과 MVCC(Multi version Concurrency Control) 등에 대해서도 설명을 하고 싶었으나 주제 선정에 참 많은 시간을 쏟아버리는 바람에 그렇지 못해 아쉽습니다. 기회가 된다면 주제 선정을 위해 방황하며 찾은 정보들에 대해서도 나누는 글을 써봐야겠습니다.

## 출처

https://www.postgresql.org/docs/current/mvcc.html

https://www.postgresql.org/docs/current/sql-update.html