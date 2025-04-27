# ✅ 시스템콜

시스템 콜은 운영 체제의 핵심 자원(메모리, 디스크, CPU 등)에 안전하게 접근하기 위한 인터페이스이다.
사용자는 시스템 콜을 통해 커널을 거쳐 자원을 관리하고 접근한다.

## 시스템콜과 자원 접근

| 자원        | 시스템콜 예시                         | 설명                          |
|:------------|:---------------------------------------|:------------------------------|
| **메모리**   | `mmap()`, `brk()`                      | 메모리 공간 할당/해제 요청     |
| **디스크**   | `open()`, `read()`, `write()`           | 파일 시스템 접근               |
| **CPU**      | `fork()`, `exec()`, `nanosleep()`       | 프로세스 생성, CPU 점유 제어    |
| **네트워크** | `socket()`, `connect()`, `send()`, `recv()` | 네트워크 통신               |

---

> 시스템콜은 결국 "하드웨어 자원 접근 권한"을 커널에 집중시키고,  
> 사용자 프로그램은 안전하게 간접 접근하도록 만드는 **권한 분리 메커니즘**이다.

---
# ✅ 권한의 분리
`시스템 콜은 근본적으로 권한의 분리이다.`
보안성, 안정성, 유지보수성 때문에 운영체제(OS)뿐만 아니라, 복잡한 시스템에서는 거의 필수적으로 쓰이는 설계 방식이다.


## 1. 데이터베이스(DB) - 사용자 권한 관리

**예시**
- 사용자를 만들 때 `SELECT`, `INSERT`, `UPDATE`, `DELETE` 등의 권한을 개별 부여.
- 읽기 전용 계정은 `SELECT`만 가능하고, `INSERT`나 `DELETE`는 제한할 수 있음.

**이유**
- 중요한 데이터(결제 기록, 고객 정보) 보호.
- 실수나 해킹에 의한 피해 최소화.
- **최소 권한 원칙(Principle of Least Privilege)** 적용.

---

## 2. 네트워크 - 포트 접근 권한

**예시**
- 1024번 이하 포트(예: 80, 443)는 root 권한이 있어야 열 수 있음.
- 일반 사용자는 8080 같은 높은 포트 번호를 사용.

**이유**
- HTTP, HTTPS 등 시스템 핵심 서비스 보호.
- 가짜 서비스(피싱 서버 등) 생성 방지.

---

## 3. JVM(Java Virtual Machine) - Security Manager

**예시**
- 코드가 파일, 네트워크 등 민감한 자원에 접근할 때 사전에 권한 검사.
- 서드파티 라이브러리나 외부 코드의 위험 행동 제한.

**이유**
- 애플리케이션 내 코드 단위에서도 신뢰 경계를 설정.
- 보안 취약점을 코드 레벨에서 방지.

> ※ 최근 JVM에서는 Security Manager 기능은 deprecated 되었지만, 설계 철학은 여전히 중요하다.

---

## 4. 웹 애플리케이션 - 인증(Authentication)과 권한(Authorization)

**예시**
- Authentication: 로그인하여 "누구인지" 확인.
- Authorization: "무엇을 할 수 있는지"를 구분.
    - 일반 유저: 자신의 글 수정 가능
    - 관리자: 모든 글 삭제 가능

**이유**
- 사용자별로 기능을 제한하여 데이터 보호.
- 보안 강화 및 사고 예방.

---

# ✅ 권한 분리와 MSA구조

> 확장해서 생각하면 권한의 분리는 MSA구조와 밀접해있다. 
> MSA 구조에서는 각 서비스가 독립적인 권한과 책임을 가지고 있고 구조가 복잡해 짐에 따라 발생할 수 있는 문제들이 있다.
> 
> 이렇게 복잡한 MSA구조로 발생할 수 있는 문제들을 OS시스템에서는 어떻게 해결하고 있을까?


## MSA 구조에서 발생할 수 있는 문제

| 문제 | 설명 |
|:---|:---|
| 데이터 일관성 문제 | 여러 서비스가 서로 다른 시점의 데이터를 바라볼 수 있음. (ex. 커밋/플러시 타이밍 불일치) |
| 네트워크 지연 및 오류 | 서비스 간 통신 실패로 인해 데이터 불일치 발생 가능. |
| 트랜잭션 복잡성 증가 | 분산 트랜잭션(2PC, SAGA 패턴 등)이 필요해짐. |
| 락과 병목 현상 | 서로 다른 서비스가 같은 자원(DB row 등)을 두고 경쟁할 수 있음. |

---

### DB 문제 발생 시나리오

- 버퍼에는 반영됐지만 commit 전에 장애가 발생하면 → 데이터 유실
- 여러 트랜잭션이 동시에 같은 데이터를 접근하면 → 락 경합 및 병목
- 락이 오래 유지되면 → 다른 트랜잭션 대기, 타임아웃 발생

> 결과적으로 어플리케이션은 **"갱신 실패"**, **"트랜잭션 충돌"**, **"읽기 일관성 오류"** 를 겪을 수 있음.

---

## 시스템콜(User - Kernel 분리)에서는 왜 이런 문제가 덜 발생할까?

### 시스템콜 설계 특징

| 항목                     | 설명 |
|:-----------------------|:---|
| **Atomicity (원자성 보장)** | 대부분의 시스템콜(fork, write, read 등)은 커널 내부에서 원자적으로 수행됨. |
| 명확한 권한 구분              | 커널은 자원(메모리, 파일, 네트워크 등)에 대한 최종 관리 권한을 독점함. |
| **컨텍스트 스위칭 보장**            | 유저 모드 → 커널 모드 전환 시 전체 상태를 정확히 전환함. |
| 에러 핸들링 체계화             | 시스템콜은 호출 결과를 리턴 코드(0, -1)와 errno 설정으로 명확히 전달함. |
| **락, 동기화 메커니즘 내장**         | 커널은 내부적으로 세마포어, 스핀락 등을 통해 동시성 문제를 엄격히 관리함. |

> 커널은 유저 공간에 비해 훨씬 더 강력한 제어권과 동시성 보호 장치를 갖추고 있어  
> **"권한 분리 + 안정성"** 을 동시에 이룰 수 있도록 설계되어 있다.

---

# 원자성
DB connection 마다 트랜잭션을 관리할 수 있다. 
트랜잭션 rollback, commit, flush 등을 활용해서 원자성을 보장할 수 있다. (예시 JAVA/Spring)

```Java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.sql.Statement;

public class TransactionalExample {

    public void performBusinessLogic() {
        Connection connection = null;
        Statement statement = null;

        try {
            // DB 연결
            connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb", "user", "password");
            connection.setAutoCommit(false);  // 트랜잭션 시작

            statement = connection.createStatement();
            
            // 비즈니스 로직 수행
            statement.executeUpdate("UPDATE account SET balance = balance - 100 WHERE id = 1");
            statement.executeUpdate("UPDATE account SET balance = balance + 100 WHERE id = 2");
            
            // 트랜잭션 커밋
            connection.commit();
        } catch (SQLException e) {
            if (connection != null) {
                try {
                    // 트랜잭션 롤백
                    connection.rollback();
                } catch (SQLException ex) {
                    ex.printStackTrace();
                }
            }
        } finally {
            try {
                if (statement != null) statement.close();
                if (connection != null) connection.close();
            } catch (SQLException ex) {
                ex.printStackTrace();
            }
        }
    }
}

```
하지만 이렇게 하나의 비즈니스 로직이 하나의 트랜잭션을 활용하는 경우는 거의 없다.


## 여러 트랜잭션을 사용하는 비즈니스 로직의 원자성

비즈니스 로직에서 여러 트랜잭션을 사용하게 되면, 중간에 오류가 발생하면 이전에 실행했던 트랜잭션은 모두 rollback이 필요하다.

1. 컨넥션 단이 아닌, 레포지토리 단위에서 상속받아 사용한다. transaction로직을 구현해서 사용할 수 있다. (예시 nestjs)
2. @Transactional(Java/Spring), @Transaction(nodejs/nestjs) 사용
```ts
export class CreateItemUseCase {
  constructor(
    @Inject(ITEM_REPOSITORY)
    private readonly ItemRepository: itemRepository,
    @Inject(POLICY_REPOSITORY)
    private readonly PolicyRepository: policyRepository){
  }

  async execute(request: Request): Promise<Response> {
    try {
      const { item } = request;
      await this.itemRepository.startTransaction();
      await this.policyRepository.startTransaction();

      const createdItemId = await this.itemRepository.create(item);
      const createdItemPolicy = await this.policyRepository.create(item);

      await this.itemRepository.update(createdItemId, createdItemPolicy);

      await this.itemRepository.commitTransaction();
      await this.policyRepository.commitTransaction();
      return {
        success: true,
      };
    }
    catch (error) {
      await this.itemRepository.rollbackTransaction();
      await this.policyRepository.rollbackTransaction();
      throw new CustomError({
        ...ERROR_CODE,
        message: error.message,
      });
    }
    finally {
      await this.itemRepository.release();
      await this.policyRepository.release();
    }
  }
}

```

비즈니스 로직에 비동기 처리가 추가되어있다면 추가적인 문제가 발생할 수 있다.

## 비동기 처리 로직에서 원자성
외부 시스템으로 요청을 보내고 웹 훅을 통해 결과값을 확인하는 비동기 로직이 중간에 있을 수 있다.
이 때, 발생할 수 있는 두 가지 문제가 있다.

1. 비동기 요청을 보낸 후 트랜잭션 실패
2. 너무 빠른 비동기 처리로 트랜잭션 성공/실패 전 응답

### 해결방법 : delayedQueue
RabbitMQ 4.0.x 이상에서 지연 큐를 지원한다. 이를 활용하면 비동기 웹 훅으로 요청이 들어왔을 때, 트랜잭션 종료 확인이 안된다면 지연 큐에 넣은 후 재확인하는 과정을 추가할 수 있다.

# ✅ 핵심 요약

> **"권한의 분리"** 는 시스템 안정성과 보안을 위해 필수적이지만,  
> 그로 인해 **데이터 일관성, 락 경합, 트랜잭션 실패** 같은 복잡한 문제가 발생할 수 있다.
>
> OS 커널은 강력한 제어와 동시성 보호로 문제를 예방하지만,  
> **분산 시스템(MSA, 클라우드, DB)** 에서는 별도의 복구 전략과 설계 패턴을 통해 신뢰성을 확보해야 한다.

