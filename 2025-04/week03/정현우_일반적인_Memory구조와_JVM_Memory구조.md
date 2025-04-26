## Intro

Jungle CS Study의 첫 주제는 “Process 개념과 관련지어, 자신이 사용 중인 기술 스택의 동작 원리에 대해 설명하자”이다.

상세 주제를 정하기 전에, Process란 무엇인가 복습해보자. 

<aside>
💡Process는, 

- OS가 Disk에 있던 Program Code를 읽어 RAM(Main memory)에 Load하여, 실행시킨 프로그램이다 (e.g., class와 instance의 관계)
</aside>

프로세스가 thread, memory, system call 등을 모두 포함하는 워낙 포괄적인 개념이다 보니 구체적으로 어떤 주제를 잡을지 고민이 됐다… 그래도 요즘 포트폴리오로 만든 주변시위 Now라는 Spring App의 성능 테스트를 진행하면서 Prometheus와 Grafana로 JVM Memory metric을 접했기 때문에, “Memory”를 주제로 정했다! JVM도 하나의 Process니까, JVM의 메모리 구조를 까보는 것도 괜찮겠지?

![Image](https://github.com/user-attachments/assets/d03935fc-24f3-4692-bb4d-9e96d4547fad)

![Image](https://github.com/user-attachments/assets/dacb7e82-364e-4cef-ba93-365f454a2eb9)

## 글의 목표

- 일반적인 Virtual Memory 구조를 이해한다
- JVM Memory 구조(Runtime Data Area)에서, 세부 구성 요소를 이해한다
    - Heap Area
        - Eden, Survivor Space
    - Non-heap Area
        - Meta Space
        - …
- 일반적인 Virtual Memory 구조와 JVM Memory 구조(Runtime Data Area)의 차이를 이해한다

## 일반적인 Virtual Memory Segments

먼저, 흔히 배우는 가상 메모리 구조는 아래와 같다.

<img width="490" alt="Image" src="https://github.com/user-attachments/assets/89a7d9b9-6a54-45bf-a34c-22609ad7cb2b" />

### Code(Text) Segment

- 실행 가능한 프로그램 코드(이진수로 된 기계어)가 저장되는 영역
- 읽기 전용으로 설정되어 있어 프로그램이 자신의 코드를 수정할 수 없음
- CPU의 Program Counter Register가 해당 영역의 주소를 저장한다
- 프로그램이 메모리에 로드될 때 영역의 크기가 정해짐

### Data Segment

- Global 변수나, Static 변수 등 프로그램 전체에서 사용되는 데이터가 저장되는 공간
- Code 영역과 마찬가지로, 프로그램 시작 시 할당되고 종료 시 해제됨
- 크게 2가지의 Data 영역이 있다
    1. Initialized Data Segment
        1. Read-Only Data Segment
            - 상수와 문자열 리터럴이 저장됨
            - 실행 중에 수정이 불가능한 영역
            - 메모리 보호 기능으로 쓰기 시도 시 세그먼테이션 오류 발생
            - 예: `const int MAX_VALUE = 100;`, 문자열 상수 `"Hello, World"`
        2. GVAR(Global Variables) Segment
            - 초기값이 있는 전역 변수와 정적 변수 저장
            - Read/Write 모두 가능
            - 예: `int globalCounter = 0;`, `static int count = 10;`
    2. UnInitialized Data Segmnet
        1. BSS(Block Started by Symbol) Segment
            - 초기화되지 않은 전역 변수와 정적 변수가 저장
            - 프로그램 로드 시 0으로 초기화됨
            - Read/Write 모두 가능

### 힙(Heap) 영역

- 동적으로 할당된 메모리를 관리하는 영역
- 프로그램 실행 중 크기가 변할 수 있음
- 메모리 할당(malloc/new)과 해제(free/delete)가 명시적으로 이루어짐
- 낮은 주소에서 높은 주소 방향으로 증가

### 스택(Stack) 영역

- 함수 호출 정보, 지역 변수, 매개변수 등이 저장
- 함수가 호출될 때 스택 프레임이 생성되고 반환될 때 제거됨
- 높은 주소에서 낮은 주소 방향으로 증가
- 각 스레드는 독립적인 스택을 가짐

## JVM Memory Segments(Runtime Data Area)

이제 JVM의 메모리 구조를 살펴보자.

![Image](https://github.com/user-attachments/assets/d08c7dc1-7256-40ec-bd14-331e6a265a4c)

JVM의 메모리 구조는 크게 모든 JVM 쓰레드가 공유하는 영역(Shared Memory)와 쓰레드마다 개별적으로 가지는 영역으로 구분된다. 그리고 일반적인 memory 구조에는 없는 영역들도 있는데, 하나하나 뭐 하는 녀석들인지 살펴보자.

### Heap Memory

- 모든 JVM 스레드가 공유하는 메모리 영역
- new() 연산자로 생성한 객체와 배열이 저장되는 곳
- 전체 크기는 한정되어 있지만, 사용 크기는 앱 실행 중에 동적으로 증가하거나 감소함
- Garbage Collector(GC)에 의해 자동 관리됨
- 종류
    - **Young Generation**
        - **Eden Space**
            - 새로 생성된 대부분의 객체가 처음 위치하는 영역
        - **Survivor Spaces - S0, S1**
            - 가비지 컬렉션 후 살아남은 객체가 이동하는 두 개의 영역
            - ‘From’, ‘To’ 영역이라 부르기도 함
    - **Old Generation**
        - Young Generation에서 오랫동안 살아남은 객체들이 이동하는 영역
        - 보통 더 큰 메모리 공간을 차지하며, Full GC의 대상이 됨

### Metaspace

- JVM 내에서 실행될 수 있는 byte code인 class file format(`.class`)의 정보를 저장(class에 대한 meta data)
    - constant pool, static variables, member variables/methods에 대한 정보 등
- Java 8 이전과 이후로 포함 영역이 다름!
    
    ![Image](https://github.com/user-attachments/assets/ffc94f56-cf9e-425c-be52-e5daa536890a)
    
    - Java 8 이전: Permanent Generation로 구현
    - Java 8 이후: 네이티브 메모리에 있는 메타스페이스로 대체

### 코드 캐시(Code Cache)

- JIT(Just-In-Time) 컴파일러가 네이티브 코드를 저장하는 영역
- 자주 실행되는 메서드의 컴파일된 코드가 저장됨

### JVM 스택(JVM Stack)

- 각 스레드마다 독립적인 스택 생성. Stack 안에 “Frame”이란 것이 함께 만들어진다
- **Frame**
    - frame은 메소드가 호출될 때마다 만들어지먀, 메소드의 상태 정보를 저장함
    - frame의 3요소: 1) Local Variables 2) Operand Stack 3) Frame Data
    - Stack Frame의 크기는 Compile time에 정의됨

### Program Counter Register

- 각 스레드마다 생성되는 레지스터
- 현재 실행 중인 JVM 명령의 주소를 가리킴. Native 명령어의 주소값은 저장하지 않음
- PC Registers는 Register-base로 구동되는 방식이 아니라 **Stack-base**로 작동

### Native Method Stack

- java code와 상호작용하는 native methods(C/C++ …)의 실행을 관리하기 위한 스택
- C Stack이라고도 불림
- 고정 크기 또는 동적 확장/축소 가능

## 일반적인 Memory 구조와 JVM의 메모리 구조 비교

이제 JVM의 메모리 구조에 대해서도 살펴보았다. 위 구조를 한 눈에 비교해서 보자

<img width="490" alt="Image" src="https://github.com/user-attachments/assets/89a7d9b9-6a54-45bf-a34c-22609ad7cb2b" />

![Image](https://github.com/user-attachments/assets/d08c7dc1-7256-40ec-bd14-331e6a265a4c)

주요한 차이 요약은 아래와 같다.

| 특성 | 일반 프로세스 | JVM |
| --- | --- | --- |
| **코드 저장** | Code Segment(기계어) | Metaspace(바이트코드) |
| **전역 데이터** | Data Segment/BSS | Metaspace, Heap |
| **동적 메모리** | Heap(명시적 관리) | Heap(자동 GC 관리, 세대별 구조) |
| **지역 변수** | Stack | JVM Stack(with Frame) |
| **메모리 관리** | 프로그래머 책임 | 자동(GC) |
| **메모리 안전성** | 낮음(버퍼 오버플로우 등 가능) | 높음(범위 검사, 타입 안전성) |
| **메모리 모델** | 운영체제 관리 | JVM 관리 |
| **추가 구조** | 단순 | PC Register, C Stack 등 |
| **플랫폼 의존성** | 높음 | 낮음(WORA - Write Once Run Anywhere) |
| **성능 특성** | 직접 접근으로 빠름, 오버헤드 적음 | GC 일시 정지, 추가 추상화 레이어로 오버헤드 있음 |

이제 상세한 비교 내용을 보자.

## Code Segment 비교

- 실행될 프로세스의 명령어 집합이 저장되는 곳에 대한 비교를 해보자

### 일반: Code Segment

- *기계어*로 컴파일된 실행 파일의 코드가 직접 저장됨
- 읽기 전용으로 설정되어 수정 불가능(보안 강화)
- 예: C 프로그램의 함수 코드는 컴파일되어 기계어로 변환된 후 이 영역에 로드됨

### JVM: Metaspace, Code Cache

Metaspace

- 클래스 파일의 ***바이트코드***가 저장됨
- 클래스 구조, 메서드, 필드, 상수 풀 정보 포함
- 예: Java 클래스의 정적 메서드, 상수, 클래스 메타데이터가 이 영역에 저장됨
- 네이티브 메모리(OS가 관리하는 메모리)에 할당됨

Code Cache

- 자주 실행되는 메서드의 JIT 컴파일된 코드가 저장됨

**주요 차이점**:

- 일반 프로세스는 기계어 코드를 직접 실행하지만, JVM은 바이트코드를 해석하거나 JIT 컴파일을 통해 실행
- 일반 프로세스의 코드 세그먼트는 일반적으로 프로세스 수명 동안 고정되지만, JVM의 Metaspace 영역은 런타임에 클래스 로딩/언로딩에 따라 크기 변화가 가능

## 데이터 영역 비교

- 전역/정적 변수들이 저장/관리 되는 방식은 어떻게 다를까?

### 일반: Data Segment

- **초기화된 데이터 세그먼트**: 명시적으로 초기화된 전역/정적 변수 저장
- **BSS**: 초기화되지 않은 전역/정적 변수 저장(0으로 초기화됨)
- 프로그램의 전체 실행 기간 동안 존재
- 예: `int global_var = 100;` (초기화된 데이터), `static int uninitialized_var;` (BSS)

### JVM: Metaspace, Heap

- 클래스의 정적 변수는 Metaspace에 저장
- 참조 타입 정적 변수의 실제 객체는 JVM Heap에 저장됨
- 예: `static int counter = 0;`는 metaspace에 저장, `static Student student = new Student();`에서 참조는 metaspace에, 실제 Student 객체는 힙에 저장

**주요 차이점**:

- 일반 프로세스는 데이터 타입에 따라 직접 값을 저장하지만, JVM은 참조 타입의 경우 참조만 저장하고 실제 객체는 항상 힙에 저장
- JVM에서는 클래스 로딩 시 정적 필드가 초기화되며, 클래스 언로딩 시 제거될 수 있음

## Heap Segment 비교

- Reference type이 주로 저장되는 Heap 영역에는 어떠한 차이가 있는지 알아보자.

### 일반: Heap

- `malloc()`, `calloc()`, `realloc()` 등의 함수를 통해 명시적으로 메모리 할당
- `free()` 함수를 통해 명시적으로 메모리 해제 필요(메모리 관리는 프로그래머 책임)
- 할당과 해제가 반복되면서 메모리 단편화 발생 가능
- 범위를 벗어난 메모리 접근, 해제된 메모리 접근, 메모리 누수 등의 문제 발생 가능
- 예:
    
    ```c
    int* array = (int*)malloc(10 * sizeof(int));
    // 사용 후
    free(array);
    ```
    

### JVM: Heap

- 모든 객체 인스턴스와 배열이 저장됨
- 자동 메모리 관리(가비지 컬렉션)을 통해 더 이상 참조되지 않는 객체 자동 제거
- 세대별 가비지 컬렉션을 위해 Young/Old 영역으로 구분
- Young 영역은 다시 Eden과 두 개의 Survivor 공간으로 나뉨
- 예:
    
    ```java
    ArrayList<String> list = new ArrayList<>();// 힙에 할당
    list = null;// 참조 제거, GC 대상이 됨
    ```
    

**주요 차이점**:

- JVM은 자동 메모리 관리(GC)를 제공하여 메모리 누수, 댕글링 포인터 문제 감소
- JVM 힙은 세대별 구조로 최적화되어 단기 객체와 장기 객체를 효율적으로 관리
- 일반 프로세스는 메모리 할당 크기를 직접 지정하지만, JVM은 객체 크기를 자동으로 계산
- 일반 프로세스는 메모리 해제 시점을 프로그래머가 결정하지만, JVM은 가비지 컬렉터가 결정

## Stack Segment 비교

### 일반: Stack

- 함수 호출 정보(활성화 레코드/스택 프레임)와 지역 변수 저장
- LIFO(Last In First Out) 구조로 작동
- 컴파일 시간에 크기가 결정되는 지역 변수는 스택에 할당
- 함수 호출 시마다 새로운 스택 프레임 생성
- 재귀 호출이 너무 깊거나 지역 변수가 너무 많으면 스택 오버플로우 발생 가능
- 예:
    
    ```c
    void function() {
        int local_var = 10;// 스택에 할당// ...
    }// 함수 종료 시 스택 프레임 제거
    ```
    

### JVM: Stack(with Frame)

- 각 스레드마다 별도의 JVM 스택 생성
- 메서드 호출마다 Stack ***Frame*** 생성
- 스택 프레임은 로컬 변수 배열, 피연산자 스택, 프레임 데이터 포함
- 원시 타입 변수와 객체 참조값은 스택에 저장(객체 자체는 힙에 저장)
- 예:
    
    ```java
    void method() {
        int x = 10;// 스택에 저장
        Object obj = new Object();// 참조는 스택에, 객체는 힙에 저장
    }// 메서드 종료 시 스택 프레임 제거
    ```
    

**주요 차이점**:

- 일반 프로세스 스택은 주로 함수 실행 상태와 지역 변수 저장에만 집중한다
- 반면, JVM Stack은 기본적은 stack의 기능과 더불어 method 호출 단위의 Frame이란 개념이 추가되며, byte code 실행과 관련된 기능을 수행하기도 한다
    - JVM 스택은 스레드마다 독립적으로 생성됨
    - JVM 스택 프레임은 피연산자 스택을 포함하여 바이트코드 실행을 지원
    - JVM 스택은 참조 타입의 경우 참조만 저장하고 실제 객체는 항상 힙에 저장됨
    - JVM 스택은 현재 실행 중인 바이트코드에 대한 정보를 포함

## 추가 JVM 메모리 구조

- 일반적인 memory 구조에는 없는, JVM에만 있는 memory 영역 또한 존재한다

### PC 레지스터

- 각 스레드마다 별도의 PC 레지스터 존재
- 현재 실행 중인 JVM 명령어 주소(바이트코드 오프셋) 저장
- 네이티브 메서드 실행 시 undefined 값 가짐

### 네이티브 메서드 스택

- JNI(Java Native Interface)를 통해 호출된 네이티브 메서드를 위한 스택
- C 스택이라고도 불림
- 일반 프로세스의 스택과 유사하게 작동

주요 차이점:

- JVM은 ‘Write Once Run Anywhere’라는 철학으로 만들어졌다. 따라서 특정 OS나 Hardware에 종속되지 않고자 위와 같은 요소를 메모리 구조에 추가하였다

## 후기

오랜만에 CS 공부를 하니 시간 가는 줄 모르고 정말 재미있게 했다. 특히 요즘 prometheus로 JVM Metric을 수집하는데, JVM 관련 지식에 목말랐던 터라 이번 JVM의 메모리 구조에 대해 공부하는게 더욱 흥미로웠다. 이런 식으로 일반적인 CS 개념이 내가 요즘 궁금해하고 사용하는 기술과 비교하여 공부하니 학습 효과도 더욱 뛰어난 것 같다. 다만 아쉬운 점도 있다. **JVM의 Memory 구조에는 ‘Write Once Run Anywhere’라는 java의 철학을 반영하기 위해 존재하는 영역이나 방식**이 많아, Java가 이러한 철학을 지키기 위한 전체적인 흐름(JRE, Java Runtime Environment)에 대해 알고 있었다면 더욱 빠른 학습과 깊이 있는 이해가 가능했을 것 같다. 그래도 이번 주제를 열심히 준비한 덕분에 java의 실행 방식에 대해 전체적인 그림을 익힐 수 있었다.

또 이번 글을 준비하며 다음과 같은 주제에 흥미가 생겼다

- java의 전체적인 동작 흐름(compile부터 종료까지)
- java의 system call 동작 과정
- JVM Memory 최적화
- JIT Compile
- …

앞으로 스터디 할 날이 많으니, 차차 다뤄봐야겠다.

### 출처

https://hongsii.github.io/2018/12/20/jvm-memory-structure/

https://jaemunbro.medium.com/java-metaspace%EC%97%90-%EB%8C%80%ED%95%B4-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90-ac363816d35e

https://johngrib.github.io/wiki/java8-why-permgen-removed/

https://johngrib.github.io/wiki/jvm-stack/

https://loosie.tistory.com/846

![](https://image.samsungsds.com/kr/insights/javamemory_img01.jpg?queryString=20250214030334)


[https://help.sap.com](https://help.sap.com/)

https://kotlinworld.com/308

https://velog.io/@lucius__k/JIT-%EC%BB%B4%ED%8C%8C%EC%9D%BC%EB%9F%AC%EC%99%80-%EC%BD%94%EB%93%9C-%EC%BA%90%EC%8B%9C