# 시스템 콜

## kill() 과 bind()
kill() : 프로세스에 "시그널"을 보내서 프로세스에 어떤 행동을 하도록 요청하는 것   
→ *"시그널" 은 어떤 형태로 프로세스에 보내지며, 어떤 과정을 거쳐 프로세스에 전해질까?*


### 주제 선정 배경

### 문제 상황!
dash 앱을 4월 21일에 배포 후, 다음 날 출근했을 때, 리눅스 터미널 세션이 닫힘(정확히 닫힌건지, dash 앱이 비정상적으로 터지면서 linux 세션까지 멈춰버린건지 가물가물함..). 

그러면서 dash 앱도 같이 배포가 종료되어, 윈도우에서 하던 것처럼 똑같이 gunicorn app:server --bind ~로 배포를 했지만 *Address already in use* 에러가 발생함. 

찾아보니 이전에 띄웠던 앱의 프로세스가 아직 살아있거나, 좀비처럼 포트를 점유하는 상황.
→*어떤 과정으로 좀비 상태가 된 걸까?*

### 좀비 프로세스 발생 시점!
죽었는데도 PID와 프로세스 정보(task_struct)는 살아 있는 상태   
이유 : 부모 프로세스가 아직 자식의 종료 상태를 읽지 않았기 때문
### Address already in use (Bind Failed) 에러란?

서버 역할을 하는 프로그램(nginx, tomcat, java(spring), python(django), nodejs 등)이 리눅스 서버 내에서 특정 IP 주소와 Port 번호를 사용하려고 할 때, bind 시스템 콜을 사용하게 됨.

그런데 이미 다른 프로세스가 해당 Port 번호를 이미 사용하고 있을 때, 포트 충돌로 인한 Bind Failed 에러가 발생할 수 있음.

[참고 링크] https://reallinux.co.kr/blog/199

### 해결 과정

먼저, 점유 중인 PID를 확인했음

*PID : Process ID, 운영체제에서 프로세스를 식별하기 위해 사용하는 고유한 번호.*   

```
sudo netstat -tnlp | grep 9000
```

그 결과 
```
tcp    0    0    0.0.0.0:9000    0.0.0.0:*    LISTEN    29346/python3
```

이렇게 떴고, 여기서 **29346**이 포트를 점유한 PID였음.

*--workers 4 이렇게 멀티 프로세스를 사용한 경우, 총 다섯개의 프로세스 목록이 뜸. 거기서 가장 위의 프로세스가 마스터 프로세스로, 마스터를 kill 하면 한꺼번에 종료 됨*

검색 결과 많은 블로그에서 이 문제 해결 방식으로 kill -9을 통해 종료시켰지만, https://reallinux.co.kr/blog/199  이 블로그에서는 그런 방식의 문제점을 지적하고 있었음

kill 9 으로 강제종료를 시키게 되면, **시스템 자원해지를 정상적으로 하지 못하고 종료가 될 수 있고, 이는 다른 부작용 발생 여지가 있다는게 문제점이었음.**

9번은 강제 종료, 15번은 일반적인 종료의 시그널임.

그래서 
```
sudo kill -15 29346 
```
이런식으로 종료해야 한다는게 글쓴이의 의견이었음.

### kill 시스템 콜

[참고 링크] https://www.joinc.co.kr/w/man/2/kill

kill() 시스템 콜은 특정 프로세스나 프로세스 그룹에 시그널을 보내기 위해서 사용한다.

```
#include <sys/types.h>

#include <signal.h>

int kill(pid_t pid, int sig);
```
![alt text](image.png)

kill -15 종료 로직

1. 열려 있던 파일 닫기

2. 데이터베이스 연결 끊기

3. 메모리 해제

4. 임시 파일 정리

5. 로그 남기기 등

정상적인 종료과정은 위와 같고, 자원 해제가 제대로 일어남.

하지만 kill -9의 경우, **커널이 프로세스를 즉시 종료**시키는데, 자원 해제 할 기회를 아예 갖지 못함. 
→ *커널이 프로세스를 **즉시** 종료시킨다는 것의 의미?*

메모리, 파일, 네트워크 소켓 등이 닫히지 않을 수 있음. 그 결과, 소켓이 열려있는 상태로 남아 포트를 점유하고, 임시 파일이 디스크에 그대로 남는 등의 문제가 생김

그럼에도 불구하고 kill -9 을 권유하는 글이 많은 이유는, -15로 종료시킴에도 프로세스가 무한 루프, 데드락, 응답 없음 등으로 종료되지 않는 경우가 있기 때문.​ 

### kill -9 실행 과정
kill -9 시그널이 어떤 과정을 거쳐서 프로세스를 죽일까?

1. 먼저, kill -9는 SIGKILL 이라는 시그널을 보냄. 
이때 시그널을 프로세스에 보내는 주체는 **커널**이다.

이때, 프로세스는 시그널을 **받을 수는 있지만, 처리할 수는 없다**
→ 왜냐면 받자마자 죽을 준비를 할 시간도 없이 바로 사형 당하기 때문!

> ## 시그널   
>> 커널이 프로세스에게 보내는 메세지
>> 시그널은 프로세스를 중단시키고, 삭제하는 등의 작업에 사용된다. 
>> 시그널이라는 운영체제의 메터니즘은 외부 사건을 프로세스에게 전달하는 토대이다.
>> 이 기반 구조는 시그널을 보내거나 전달받는 방법을 모두 포함한다.

다른 시스템 콜들은 시그널을 안보내나?

2. 시그널을 받으면 프로세스는 signal handler를 등록해서 내가 어떻게 반응할지 정할 수 있다.

예를 들어

```
signal(SIGTERM, my_handler)
```
이렇게 하면 SIGTERM이 왔을 때, my_handler() 함수를 실행한다.

그런데 SIGKILL(kill -9)은 특수한 시그널이라서, handler를 등록할 수 없고, 무시할 수도 없고, 수정할 수도 없다.

즉, **SIGKILL은 프로세스가 시그널을 받을 수만 있고, 그 신호에 대해 아무런 행동을 취할 수 없다.**

커널이 바로 프로세스를 죽여버리기 때문에 정리할 기회조차 주지 않음

그냥 갑자기 길가다가 핵 폭탄 맞고 증발하는 것과 비슷한듯;죽을 준비도 못하고 죽어버림.

[사용자 공간] → kill(pid, SIGKILL) 호출 → [커널 공간으로 시스템 콜 진입] → 커널이 해당 PID의 프로세스 찾아서 → 프로세스에게 SIGKILL 플래그 설정 → 커널이 직접 프로세스를 강제 제거 (종료 상태로 만듦)

### 프로세스가 SIGKILL 처럼 강제 종료 되어 자원 정리(cleanup)를 못하면 실제로 어떤 시스템 위험이 발생 할 수 있는가?

1. 커널 레벨 리소스는 어느 정도 정리됨.    
열린 파일 디스크립터, 소켓, 메모리 같은 커널 리소스는 커널이 자동으로 회수한다.   
프로세스가 죽으면 task_struct 해제 과정에서 다 정리가 된다.

2. 사용자 레벨 자원은 정리되지 않는다.

예를 들면,   

* DB 트랜잭션 커밋/롤백
* 파일에 버퍼가 flush 되지 않음
* 임시 파일이 그대로 남음
* 락 파일이 삭제되지 않음
* 애플리케이션 레벨에서 관리하는 공유 메모리, 세마포어 남음
→ 위의 것들은 커널이 알 수 없어 자동으로 정리가 되지 않음

3. 결과적으로   

* 파일 데이터 유실 위험(flush 안됨)
* 락 걸린 채 죽어서 다른 프로세스 무한 대기
* 공유 메모리/세마포어가 남아있어 리소스 고갈
* 소켓이 close 되지 않아 TIME_WAIT 상태 쌓임 → 포트 부족
* DB 트랜잭션이 비정상 종료 → 데이터 손상 가능성

즉, 프로세스가 정상 종료되지 않으면, application layer에서 문제가 남는다.

1. kill -15 <pid>        # 정상 종료 요청
2. (조금 기다리기)
3. kill -9 <pid>         # 그래도 안 죽으면 강제 종료

실제 리눅스 표준 운영 가이드라인들도 위의 과정으로 종료시킬 것을 권장하고 있다.

> TimeoutStartFailureMode=, TimeoutStopFailureMode=
> These options configure the action that is taken in case a
> daemon service does not signal start-up within its configured
> TimeoutStartSec=, respectively if it does not stop within
> TimeoutStopSec=. Takes one of terminate, abort and kill. Both
> options default to terminate.
>
> If terminate is set the service will be gracefully terminated
> by sending the signal specified in KillSignal= (defaults to
> SIGTERM, see systemd.kill(5)). If the service does not
> terminate the FinalKillSignal= is sent after TimeoutStopSec=.
> If abort is set, WatchdogSignal= is sent instead and
> TimeoutAbortSec= applies before sending FinalKillSignal=. This
> setting may be used to analyze services that fail to start-up
> or shut-down intermittently. By using kill the service is
> immediately terminated by sending FinalKillSignal= without any
> further timeout. This setting can be used to expedite the
> shutdown of failing services.
>
> Added in version 246.

TimeoutStartFailureMode=, TimeoutStopFailureMode=는 데몬 서비스가 설정된 TimeoutStartSec= 내에 시작 신호를 보내지 못하거나, TimeoutStopSec= 내에 정상적으로 종료하지 못했을 경우 어떤 조치를 취할지를 설정하는 옵션입니다. 이 옵션은 terminate, abort, kill 중 하나를 선택할 수 있으며, **기본값은 terminate입니다.**

**terminate로 설정되어 있는 경우, KillSignal= (기본값은 SIGTERM, 자세한 내용은 systemd.kill(5) 참조)로 지정된 시그널을 보내 정상 종료를 시도합니다. 만약 서비스가 여전히 종료되지 않는다면 TimeoutStopSec= 이후 FinalKillSignal=을 보내 강제 종료합니다.**

abort로 설정된 경우에는 KillSignal 대신 WatchdogSignal=을 보내며, 이후 TimeoutAbortSec= 동안 대기한 후 FinalKillSignal=을 전송합니다. 이 설정은 서비스가 간헐적으로 시작이나 종료에 실패하는 문제를 분석하는 데 사용할 수 있습니다.

kill로 설정된 경우에는 추가 대기 없이 즉시 FinalKillSignal=을 전송하여 서비스를 강제 종료합니다. 이 설정은 장애가 발생한 서비스를 빠르게 종료할 때 사용할 수 있습니다.

이 기능은 systemd 버전 246에서 추가되었습니다.

[참고 링크] https://man7.org/linux/man-pages/man5/systemd.service.5.html

위에서 본 것처럼, 공식 문서에서도 기본 종료 값은 SIGTERM이며, 이렇게 해서도 종료되지 않는다면 강제 종료한다고 나와있다.

블로그에 나와있는 해결 방법에도 항상 의문을 가질 필요가 있는 것 같다. 
왜 kill -9 일까? 라는 의문을 던지지 않았다면 몰랐을 것 같다.

----
# 여기서부터는 추가 자료(발표 x)

### 부모 프로세스 vs 마스터 프로세스

gunicorn 으로 dash 앱을 배포하면서, workers를 4로 설정해두고 배포했다.
그러고 프로세스를 죽여야 할 일이 있어 PID를 조회하니, 분명 4로 설정해두었으니 네개의 프로세스가 떠야 맞는데, 다섯개의 PID가 조회되었다.

알아보니 gunicorn에서는 **마스터 프로세스** 라는 개념이 존재했다.

> gunicorn의 프로세스는 프로세스 기반의 처리 방식을 채택하고 있으며, 이는 내부적으로 크게 master process와 worker process로 나뉘어진다. 
> gunicorn이 실행되면, 그 프로세스 자체가 master process이며, fork를 사용해 설정에 부여된 worker 수대로 worker process가 생성된다. 
> master process는 worker process를 관리하는 역할을 하고, worker process는 웹 어플리케이션을 임포트하며, 요청을 받아 웹 어플리케이션 코드로 전달하여 처리하도록 하는 역할을 한다.

마스터 프로세스를 kill 하면 자식 프로세스까지 다같이 종료되는 것을 확인 할 수 있었다.

왜 그런건진 시간 관계상;;

### gunicorn? WSGI?

gunicorn은 WSGI(Web Server Gateway Interface) 서버이다.  
 
WSGI란 python으로 작성된 웹 어플리케이션과 python으로 작성된 서버 사이의 약속된 인터페이스 또는 규칙이다. 간단히 말하면 WSGI 서버와 웹 어플리케이션이 WSGI의 규칙에 따라 작성되면, 웹 어플리케이션 입장에서는 내부 구현과 상관 없이 자유롭게 WSGI 서버를 골라서 사용 할 수 있는 유연성을 제공한다.
