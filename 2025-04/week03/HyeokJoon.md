# 주제 : Process 

---
## ✅ 현재 내 컴퓨터에서 동작하고 있는 프로세스는 무엇일까?

### 실행중인 프로세스 확인 방법
- **작업 관리자(Task Manager), ps/top 명령어**: 실행 중인 PID, CPU 사용률, 메모리 사용량 등 확인
- **Mac 활성상태보기(Activity Monitor)** : 실행중인 모든 프로세스를 계층별로 확인할 수 있다
- **lsof -i**, **netstat**: 모든 프로세스 확인

### 자주 발생했던 에러
`Web server failed to start. Port 8080 was already in use.` 와 같은 에러가 발생하는 경우 8080번 포트가 **할당된 프로세스**를 찾아서 종료해야 한다.
- **포트번호를 사용중인 프로세스 확인**: 
  - `lsof -i :8080`(mac) 
  - `netstat -ano | findstr :8080`(window)
- **프로세스 종료**: 
  - `kill -9 {pid}`(mac) 
  - `taskkill /f /pid {pid}`(window)

---

## ✅ 하나의 프로세스가 여러 포트를 사용할 수 있을까?

포트는 커널 레벨에서 소켓과 바인딩되며, 하나의 프로세스는 여러 소켓을 사용할 수 있다. 따라서 하나의 프로세스가 여러 포트를 동시에 사용할 수 있다.

예시로 Docker 컨테이너와 SSH 포트 포워딩을 사용하면 모두 여러 포트를 할당할 수 있다.

>처음에는 도커에서 여러 컨테이너를 실행시킬 때, 각각 포트번호를 바인딩하는것을 하나의 프로세스에 여러 포트를 할당한다고 생각했지만 이 생각은 잘못되었다는걸 깨달았다.
>
>-> 잘못된 이유 : 도커로 실행시킬 때, 각각의 컨테이너는 하나의 프로세스로 동작한다. 따라서 각각의 컨테이너에 포트번호를 바인딩하는 것은 하나의 프로세스에 하나의 포트번호를 할당하는 것과 같다.

### Docker와 ssh

우선 도커와 ssh 를 통해 여러 포트번호를 할당할 수 있는것은 맞지만, 이 둘은 서로 다른 방식으로 동작한다.
도커는 프로세스가 아닌, 격리된 환경이고 컨테이너에서 메인 프로세스를 실행시킨다. 포트는 컨테이너의 프로세스에 할당된다. 
그리고 ssh는 하나의 프로세스로 여러 포트를 포워딩한다.

### ssh
```angular2html
[ssh 프로세스] PID=1234 ───────────┐
                                 │
                    ┌────────────┴────────────┐
                    │                         │
             [스레드-1]                  [스레드-2]
                │                            │
         (파일 열기, 계산 등)         (포트 열기 → 8080 listen)
```
### docker
```angular2html
Host OS
 └── Docker Daemon
     ├── Container A (Spring Boot) - PID 1 (java)
     └── Container B (NestJS)      - PID 1 (node)
```

### 확인 방법
- `ssh 로 8080, 8081 포트를 로컬포워딩`
- `프로세스 확인` : `lsof -i`

### 결과
```angular2html
lsof -i
COMMAND   PID         USER      FD  TYPE             DEVICE SIZE/OFF   NODE NAME
ssh       19932 kimhyeokjoon    3u  IPv4 0x6029ab22674a988c      0t0    TCP 192.168.10.49:61050->ec2-13-209-141-253.ap-northeast-2.compute.amazonaws.com:ssh (ESTABLISHED)
ssh       19932 kimhyeokjoon    5u  IPv6 0xe365aab044850de0      0t0    TCP localhost:http-alt (LISTEN)
ssh       19932 kimhyeokjoon    6u  IPv4 0xf2350382bac09b70      0t0    TCP localhost:http-alt (LISTEN)
ssh       19932 kimhyeokjoon    7u  IPv6 0x5c7bf3671daa4f28      0t0    TCP localhost:sunproxyadmin (LISTEN)
ssh       19932 kimhyeokjoon    8u  IPv4  0xd66c41779f05095      0t0    TCP localhost:sunproxyadmin (LISTEN)
```
```angular2html
lsof -i :8080
COMMAND   PID         USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ssh     19932 kimhyeokjoon    5u  IPv6 0xe365aab044850de0      0t0  TCP localhost:http-alt (LISTEN)
ssh     19932 kimhyeokjoon    6u  IPv4 0xf2350382bac09b70      0t0  TCP localhost:http-alt (LISTEN)
lsof -i :8081
COMMAND   PID         USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ssh     19932 kimhyeokjoon    7u  IPv6 0x5c7bf3671daa4f28      0t0  TCP localhost:sunproxyadmin (LISTEN)
ssh     19932 kimhyeokjoon    8u  IPv4  0xd66c41779f05095      0t0  TCP localhost:sunproxyadmin (LISTEN)
```
동일한 PID(19932) 프로세스에 8080, 8081 포트가 모두 바인딩 되어있다.
- 하나의 SSH 프로세스(PID 1234)가 여러 스레드를 생성하여 각각의 로컬 포트를 열어 외부 포트와 연결한다.
- 각각 다른 FD로 관리된다.

---

## ✅ NestJS, Java application에 cron 로직을 추가하면 백그라운드 프로세스가 추가로 생기게 되나?

@Scheduled나 @Cron은 시간이라는 조건에 따라 자동 실행되기 때문에 전형적인 트리거 기반 방식으로, 
시간을 주기적으로 확인해서 실행시키기 때문에 다른 프로세스(백그라운드 프로세스)로 실행되는 개념이 아니다.

-> ~~사실 프로세스를 종료하면 수행되지 않는것만 봐도 알 수 있다.~~

### Spring의 @Scheduled, Nest의 @Cron
- Cron 표현식을 통해 실행되는 태스크는 별도의 프로세스를 생성하지 않고, 내부의 스레드 풀에서 실행되기 때문에 OS 레벨에서 확인되는 프로세스 수는 그대로 유지된다.

> Nestjs는 단일 스레드이기 때문에 기본적으로 병렬수행이 불가능하다.
---


## ✅ Java application에서 자식 프로세스 생성 방법
### 생성 (ProcessBuilder)
```java
ProcessBuilder processBuilder = new ProcessBuilder("python3", "data_analysis.py");
Process process = processBuilder.start();
```
ProcessBuilder를 통해 Java 어플리케이션에서 새로운 프로세스 생성을 운영체제에 요청한다.

1. Java 애플리케이션 실행 (JVM 프로세스)

2. 외부 프로세스 요청: 
   - ProcessBuilder나 Runtime.exec()를 통해 외부 프로세스를 실행하려고 할 때, Java는 운영 체제(OS)에 해당 프로세스 실행 요청을 보낸다.

3. 운영 체제에서 프로세스 생성:
   - 운영 체제는 요청을 받으면 새로운 프로세스를 생성하고, 해당 프로세스는 Java 애플리케이션과는 별개로 독립적인 실행 흐름을 갖는다.
   - **자체적인 PID(프로세스 ID)**를 가지고, OS의 프로세스 관리 시스템에 의해 관리된다.

4. Java에서 프로세스와 상호작용:
   - Process 객체를 통해 생성된 프로세스를 관리할 수 있는 핸들 역할을 할 수 있다.(프로세스의 출력 결과를 읽거나, 오류를 처리하는 등의 작업이 가능)

### 실행 확인
`docker ps` 에서는 컨테이너만 확인되고, `docker exec -it <container> ps -ef` 를 수행하면 내부에서 동작하는 프로세스들을 확인할 수 있다.
```angular2html
$ docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS     NAMES
abc123         myimage   "java -jar app.jar ..."  10 minutes ago  Up 10 minutes  8080/tcp  my-container

$ docker exec -it my-container ps -ef
PID   USER     TIME  COMMAND
1     root     0:05  java -jar app.jar
123   root     0:02  python3 data_analysis.py
```

---
## 🔍 추가로 다루면 좋은 주제

- **프로세스 vs 스레드 차이**
- **멀티 프로세스 vs 멀티 스레드 구조 비교**
- **Context Switching의 비용**
- **프로세스 관리 도구 소개 (PM2, uWSGI 등)**
- **Kubernetes에서의 프로세스 구조**

---

