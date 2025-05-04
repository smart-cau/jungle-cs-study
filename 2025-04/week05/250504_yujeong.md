# Django DB Transaction 과 lock의 관계
lock을 설명하는 예시 중에 가장 많이 언급되는 것이 은행 계좌 상태 관리이다. 
은행 계좌의 상태도 DB를 통해 관리가 되며, A가 B에게 돈을 송금하는 과정 하나를 트랜잭션이라 한다. 하나의 트랜잭션이 완료되기 전까지 다른 트랜잭션은 해당 데이터를 참조하거나 수정하지 못한다. 이것을 트랜잭션이 가지고 있는 ACID의 특성 중 하나에 해당된다(독립성).

트랜잭션이 락과 관련성이 있을 것 같아, 실제 개발 사례 중 트랜잭션, 락에 대해 검색을 하다가, JPA Transactioning이라는 카카오페이 개발 블로그를 보게 되었다.   

그 글을 보고, 내가 주로 개발하는 Django의 ORM에서의 lock은 어떻게 관리되고 있을까?라는 궁금증이 생겼다.

그래서 해당 주제를 공부해보기로 결정했다.

Django 공식 문서와, 회사 DB인 Oracle의 공식 문서를 참고하여 작성되었다.

그리고 그냥 생각의 흐름따라 꼬리물기?식으로 작성해보았다.

[참고 자료] 
1. https://tech.kakaopay.com/post/jpa-transactional-bri/
2. https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/data-concurrency-and-consistency.html#GUID-4BD4DFD6-DAEA-41B2-BB56-7135568F0548
3. https://docs.djangoproject.com/en/5.2/topics/db/transactions/

# Django ORM → DB 락 동작 흐름

1. Django ORM 코드 실행

```
with transaction.atomic():
    obj = MyModel.objects.select_for_update().get(id=1)
```

* atomic() : 트랜잭션을 정의하는 코드 블록으로, 성공하면 commit,  오류 시 rollback 되는 영역. ACID의 원자성(Atomicity)과 관련이 있다. 
atomicity : 전체가 한 번에 성공하거나, 전부 실패하고 이전 상태로 돌아간다.

* select_for_update() → row-level exclusive lock 요청 플래그를 SQL에 포함

```
exclusive lock : 배타적 잠금, 쓰기 잠금(write lock)이라고도 불린다.

어떤 트랜잭션에서 데이터를 변경하고자 할 때, 해당 트랜잭션이 완료될때까지 해당 테이블 혹은 레코드(row)를 다른 트랜잭션에서 읽거나 쓰지 못하게 하기 위해 exclusive lock을 걸고 트랜잭션을 진행시키는 것이다.

exclusive lock에 걸린 테이블, 레코드 등의 자원에 대해 다른 트랜잭션이 exclusive lock을 걸 수 없다.
```

2. SQL 쿼리 생성 및 실행
Django가 실제로 DB에 날리는 SQL 쿼리 

```
SELECT * FROM my_model WHERE id=1 FOR UPDATE;
```

FOR UPDATE → DB에게 "이 row에 락 걸어줘" 라고 요청

3. DB 엔진 처리(Oracle 예)
* Oracle은 쿼리를 분석하고, 해당 row에 exclusive lock 설정
* 같은 row에 대해 다른 트랜잭션이 update, delete, select_for_update 등을 요청하면 대기(blocking) 시킴
* 읽기(select)는 멀티버전 일관성(MVCC) 덕분에 락 없이 과거 버전 데이터 제공

```
멀티버전 일관성(MVCC, Multi-Version Concurrency Control) : 동시 접근을 허용하는 데이터베이스에서 동시성을 제어하기 위해 사용하는 방법 중 하나.
원본의 데이터와 변경 중인 데이터를 동시에 유지하는 방식으로, 원본 데이터에 대한 snapshot을 백업하여 보관한다. 만약 두 가지 버전의 데이터가 존재하는 상황에서 새로운 사용자가 데이터에 접근하면, 데이터베이스의 snapshot을 읽는다. 그러다가 변경이 취소되면 원본 snapshot을 바탕으로 데이터를 복구하고, 만약 변경이 완료되면 최종적으로 디스크에 반영하는 방식으로 동작한다.
```

4. 트랜잭션 종료 시(commit 또는 rollback)
* Django atomic 블록이 끝나면 → commit 또는 rollback 실행
* Oracle 은 row-level lock 해제(unlock)

commit : 트랜잭션 제어 명령어, 보류 중인 모든 데이터 변경 사항을 영구적으로 적용. 현재 트랜잭션 종료   
rollback : 트랜잭션 제어 명령어, 보류 중인 모든 데이터 변경 사항을 폐기. 현재 트랜잭션 종료, 직전 커밋 직후의 단계로 회귀   
savepoint : 트랜잭션 제어 명령어, rollback할 표인트 지정

위 과정을 공부하면서, 내가 지정해두었던 시나리오가 동시성 제어 중, 비관적 동시성 제어에 해당한다는 것도 깨달았다. 

* 비관적 동시성 제어(Pessimistic Concurrency Control)
사용자들이 같은 데이터를 동시에 수정할 것이라고 가정
데이터를 읽는 시점에 lock을 걸고, 트랜잭션이 완료될때까지 이를 유지한다.
SELECT 시점에 Lock을 거는 비관적 동시성 제어는 시스템의 동시성을 심각하게 떨어뜨릴 수 있어서 wait 또는 nowait 옵션과 함께 사용해야 한다.

### 왜 SELECT 시점 락이 동시성을 심각하게 떨어뜨릴까?

비관적 동시성 제어에서는 데이터를 읽는 순간(lock)부터, 트랜잭션이 끝날 때까지(lock 유지) 계속 락을 잡고 있다.

사용자 A가 어떤 행(row)을 SELECT FOR UPDATE 로 읽고 락을 건 상태로 트랜잭션을 오래 유지하고 있다. 이 동안 다른 사용자(트랜잭션)들은 그 행에 접근(수정 또는 SELECT FOR UPDATE)이 불가하다. 사용자가 많으면 많을수록 락 경합(lock contention)이 심해지게 된다.

즉, 읽기만 해도 다른 트랜잭션들을 대기시켜버리는 구조는 시스템 전체의 동시성을 급락시키게 된다.

그리고 wait, nowait 옵션은 무엇일까?

Oracle(또는 PostgreSQL, MySQL 일부 엔진)에서 FOR UPDATE 같이 락을 걸 때, wait, nowait 옵션을 선택할 수 있다.

wait : 데이터베이스가 지정한 초 수만큼 기다리는 동안 락이 해제되지 않으면 에러 발생.
락이 풀리면 제어권을 반환한다.   
nowait : 락이 걸려있으면 데이터베이스가 즉시 제어권을 반환한다. 기다리지 않고 곧바로 에러를 반환하는 것.

이를 사용하게 되면 불필요하게 긴 시간 동안 시스템 응답을 기다리지 않을 수 있고, 서버 과부하도 방지할 수 있다.




