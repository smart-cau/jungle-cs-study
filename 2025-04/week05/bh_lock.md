국내 락 추천 : 터치드 - Hi Bully
# 락(Lock)에 대해서 알아보자.

<<<<<<< HEAD
## 1. 락(Lock)이 왜 필요할까?

프로그램에서 프로세스는 단 하나지만, 스레드는 여러개가 존재할 수 있다. 2개의 스레드 A, B가 있다고 가정하고 둘 다 `Coins` 변수를 공유하고 있다고 생각해보자.

스레드 A와 B가 서로에게 영향을 주지 않고 독립적으로 수행된다면, 훌륭한 일이지만 안타깝게도 스레드 A가 `Coins` 변수를 사용하고 있을 때, 스레드 B도 사용하고 싶을 수 있다. 이렇게 될 경우 스레드 A와 스레드 B가 동시에 `Coins` 변수를 사용하고 있기에, 값이 어떻게 변화될 지 모르게 된다.

양자역학과 같이 유추할 수 없는 형태가 된다면, 변수를 사용하는 방법은 굉장히 어려워진다. 이 때, 락을 이용해서 문제를 해결할 수 있다.

### 락의 효능

![alt text](image.png)
- **경쟁 상태 방지** : 여러 스레드가 동시에 공유 데이터를 수정할 때, 예측할 수 없는 결과를 방지함!
- **데이터 일관성** : 공유 데이터가 모든 스레드에서 일관되게 보일 수 있도록 함!

## 2. 락에 대해서 알아보자.

락은 멀티 스레드 환경에서 공유 자원에 대한 접근을 관리하는 개념을 의미한다. 앞서서 말한 것과 같이 멀티스레드가 서로 공유하고 있는 자원을 사용할 때, 필연적으로 경쟁 상태(Race Condition)이 발생할 수 있다.

이러한 문제를 방지하고 데이터의 일관성을 유지하기 위해서 사용한다. 말 그대로 잠금(Lock)을 걸어서 공유 자원이 사용 중일 때, 다른 스레드가 접근하지 못하게 하는 방식이다.

락에서 사용되는 주요한 단어들을 알아보고, 락을 구현하는 여러 방식들에 대해서 살펴보자.

### 임계 영역(Critical Section)

임계 영역은 실질적으로 문제가 있을 것이라고 판단되는 코드의 영역을 의미한다. CS 이론들은 우리가 확인할 수 없는 H/W 영역에 존재하는 경우가 대다수인데, 임계 영역은 S/W에서 확인할 수 있다.

```
public void CalculateCoin(int count){
    coin = 1000;
    coin -= count; // 스레드가 동시 접근하면 문제가 발생할 수 있다!
}
```

우리가 작성하는 코드에서 공유 자원으로 사용되는 곳이 있으면 그게 임계 영역이 될 수 있다. 예를 들어, 여러 스레드가 동시에 같은 변수 `coin`을 수정하려고 할 때, 그 코드 부분이 임계 영역이 된다. 

이러한 임계 영역들은 락으로 보호되어야 한다. 임계 영역을 적절히 보호하지 않으면 **경쟁 상태(Race Condition)** 가 발생하여 예상치 못한 결과가 나올 수 있다.

### 원자적 연산(Atomic Operation)

원자적 연산은 스레드가 실행하는 일의 단위처럼 사용된다. 

원자적 연산은 다른 스레드에 의해서 중간에 중단되지 않으며, 완전히 실행되거나 전혀 실행되지 않는 연산을 의미한다. 연산이 **분할할 수 없는(`atomic`)** 단위로 취급된다.

```
int money = 0;  // 일반적인 연산
money++;        // (의외로) 복합 연산
```

대부분 프로그래밍 언어는 단일로 이루어진 연산을 사용하지만, 복합 연산을 사용하는 경우가 있다. `money++`와 같은 증감연산자가 대표적인데, money 값 읽기, 더하기, 쓰기로 구성되어 있다. 원자적 연산이 보장되지 않으면 이 과정에서 다른 스레드가 간섭할 때, 우리는 원하는 결과를 얻을 수 없다.

원자적 연산을 보장하여, 우리가 원하는 결과를 출력할 수 있도록 해야한다.

## 3. 락의 종류

### 3-1. 낙관적 락(Optimistic Lock)

낙관적 락은은 충돌이 드물게 발생한다고 가정하고 설계한다. 리소스에 접근할 때는 락을 걸지 않는데, 데이터에 대한 변경 사항을 적용하기 전에 데이터가 변경되었는지 확인하는 과정을 거친다.

만약에 데이터가 변경 되었다면 충돌 해결 방법을 적용하고 데이터를 저장한다. 동시 수정되는 경우가 없을 때, 자주 사용되는 방식이다.

### 3-2. 비관적 락(Pessimistic Lock)

낙관적 락과 반대로, 충돌이 자주 발생한다고 가정하고 설계한다. 리소스에 접근할 때 락을 흭득해야 하며, 락을 누가 사용중이라고 하면, 락이 없는 다른 스레드는 락을 흭득하기 전까지 대기하고 있어야 한다.

데이터의 일관성을 보장하는 좋은 방법. 비관적 락을 구현하는 방법에는 공유 락(Shared Lock)과 배타 락(Exclusive Lock)이 있다.

### 3-3. 공유 락(Shared Lock)

공유 락은 읽기 락(Read Lock)으로도 불린다. 공유 락이 걸린 데어티에 대해서는 읽기 연산만 가능하고 쓰기 연산이 불가능하다.

그렇기 때문에 데이터는 일관성을 유지하고 있으며, 공유 락이 걸린 데이터는 다른 스레드도 똑같이 공유 락을 흭득해서 읽기가 가능하다.

### 3-4. 배타 락(Exclusive Lock)

배타 락은 쓰기 락(Write Lock)으로도 불린다. 배타 락을 흭득한 스레드는 읽기 연산과 쓰기 연산을 모두 수행할 수 있는데, 스레드가 데이터에 대한 배타 락을 흭득하면, 다른 스레드는 배타 락이 걸린 데이터에 대해서 읽기/쓰기 작업을 수행할 수 없다.

이 경우엔 **블로킹(Blocking)** 상태가 되었다고 한다.

## 4. 비관적 락 구현 시 발생할 수 있는 데드 락(Dead Lock)

낙관적 락은 모든 스레드가 동시에 접근할 수 있기 때문에 데드 락 문제가 발생하지 않는다. 하지만, 비관적 락을 구현하면 데드 락 문제가 발생하게 된다.

데드 락은 스레드가 공유 자원의 락을 기다리느라, 무한정 대기하게 되는 상황을 의미한다.

2개의 스레드 A와 B가 있다. 

A 스레드는 `유저의 코인을 추가하는 일`을 수행하고 있기 때문에 `Coins` 데이터가 필요하다. B 스레드는 `코인을 유저에게 전송하는 일`을 수행하고 있어 `Coins`와 `User` 데이터가 모두 필요하다. 

그러면 이런 상황이 발생할 수 있다.

- A, B 스레드가 동시에 실행되었다.
- A 스레드가 `Coins` 데이터에 대한 락을 획득한다.
- B 스레드가 `User` 데이터에 대한 락을 획득한다.
- A 스레드가 작업을 계속하기 위해 `User` 데이터에 대한 락이 필요하지만, 이미 B 스레드가 점유하고 있어 대기한다.
- B 스레드도 작업을 진행하기 위해 `Coins` 데이터에 대한 락이 필요하지만, A 스레드가 점유하고 있어 대기한다.
- 유저는 게임이 느려진 것을 깨닫고 하염없이 프레임이 올라오기를 대기한다...🥲

이렇게 A, B 스레드는 서로가 보유한 자원을 기다리며 영원히 진행되지 못하는 데드락 상태에 빠지게 된다.

### 4-1. 데드락을 해결하는 방법들

- **락 흭득 순서 통일**
  - 모든 스레드가 데이터의 락을 흭득할 때, 항상 동일한 순서로 자원을 흭득하게 한다.
- **타임아웃 도입**
  - 락을 흭득하고 N초의 시간 뒤에 락 권한을 해제하게 한다.
- **락 관리자 도입**
  - 모든 락 권한을 관리하는 관리자를 도입한다. 데드 락이 발생할 가능성이 있는 락 요청을 방지한다.

## 5. 게임에서 언제 락을 사용할까 ?

금융 정보는 동서양을 막론하고 아주 중요하고 민감한 정보다. 락을 설명하는 예시 시나리오 중 계좌에 대한 예시가 많은데, 다음과 같이 흘러간다.

1. 2,000원이 들어 있는 계좌 하나를 공유하는 독고남수 씨와 제임스 씨가 있다.
2. 제임스 씨가 맥북 프로를 사고 싶어서 계좌에서 2,000원을 출금했다.
3. 근데 동시에 독고남수 씨가 로또에 당첨되어서 당첨금 2,147,483,647원을 계좌에 입금되었다.
4. 아뿔사! 은행 DB는 제임스 씨가 출금할 때, 계좌는 2,000원이라고 판단했다.
5. 독고남수 씨의 당첨금 입금은 제임스 씨의 출금 과정을 처리하면서 발생했다.
6. 결과적으로 계좌에는 0원이 남아있게 되었다.

상상도 할 수 없는 아주 끔찍한 일이다. 이와 같이 게임에서도 동서양을 막론하고 아주 중요하고 민감한 정보가 있다. 바로... **플레이어 데이터(Player Data)** 다.

실의에 빠진, 독고남수 씨가 메이플스토리를 하면서 위안을 찾기로 했다. 오랜만에 게임에 접속한 그는 전 서버에서 아무도 깨지 못한 위업을 클리어 하기로 했다.

1. 독고남수 씨는 2,147,483,647초 뒤, 엄청난 위업을 클리어 했다.
2. A 서버에서 독고남수 씨에게 위업 클리어 보상을 제공했다.
3. 근데 동시에 다른 B 서버에서 플레이어 데이터 동기화 작업이 발생했다.
4. 아뿔사! B 서버는 A 서버가 클리어 보상을 받기 전 데이터를 저장했다.
5. 독고남수 씨가 다시 게임에 접속했을 때, 엄청난 위업의 보상들은 사라져 있었다.

이런 일들이 발생할 수 있어서, 플레이어 데이터는 Lock을 사용해서 사전에 방지할 수 있어야 한다. 내가 사용하는 Unity 엔진에서는 C# 언어를 사용하고 있고 다음과 같이 플레이어 데이터에 Lock을 지정할 수 있다.

```
private static readonly object savedLock = new object();

lock(savedLock)
{
    Debug.Log("업적 데이터 락 흭득!");

    플레이어_데이터.Save();

    Debug.LogWarning("업적 데이터 락 해제!");
}
```

이외로도 보스 아이템 흭득, 캐시 샵에서 한정된 아이템 구매할 때, 길드 공동 계좌에서 돈을 입출금할 때 등 다양한 부분에서 Lock이 사용되고 있다.
=======
## 1. 락(Lock)의 기본 개념

락(Lock)은 멀티스레드 환경에서 공유 자원에 대한 접근을 제어하는 동기화 메커니즘입니다. 여러 스레드가 동시에 같은 자원에 접근할 때 발생할 수 있는 경쟁 상태(Race Condition)를 방지하고, 데이터의 일관성을 유지하기 위해 사용됩니다. 락이 설정된 자원에는 한 번에 하나의 스레드만 접근할 수 있으며, 다른 스레드는 해당 자원에 대한 락이 해제될 때까지 대기해야 합니다.

## 2. 락의 필요성

락이 필요한 주요 이유는 다음과 같습니다:

1. **경쟁 상태 방지**: 여러 스레드가 동시에 공유 데이터를 수정할 때 예측할 수 없는 결과가 발생하는 것을 방지합니다.
2. **원자성 보장**: 여러 단계로 구성된 연산이 중간에 중단되지 않고 완전히 실행되도록 보장합니다.
3. **데이터 일관성 유지**: 공유 데이터에 대해 모든 스레드가 일관된 뷰를 가지도록 합니다.
4. **가시성 보장**: 한 스레드가 변경한 데이터가 다른 스레드에 올바르게 보이도록 보장합니다.

### 락이 없을 경우 발생하는 문제

락이 없으면 다음과 같은 문제가 발생할 수 있습니다:

- **데이터 손상**: 여러 스레드가 동시에 같은 데이터를 수정하면 데이터가 손상될 수 있습니다.
- **일관성 없는 상태**: 일부 연산이 절반만 적용된 상태에서 다른 스레드가 데이터를 읽으면 불완전한 상태를 관찰할 수 있습니다.
- **예측 불가능한 동작**: 프로그램이 실행할 때마다 다른 결과를 보일 수 있어 버그 재현과 디버깅이 어려워집니다.
- **데이터 경쟁(Data Race)**: 메모리 접근 순서가 정의되지 않아 프로그램이 미정의 동작을 보일 수 있습니다.

## 3. 락의 주요 유형

### 3.1 공유 락(Shared Lock)과 배타 락(Exclusive Lock)

#### 공유 락(Shared Lock, 읽기 락)
- 여러 스레드가 동시에 데이터를 읽을 수 있도록 허용합니다.
- 다른 스레드도 공유 락을 획득할 수 있습니다.
- 데이터 수정은 허용하지 않습니다.
- 주로 읽기 작업에 사용됩니다.

#### 배타 락(Exclusive Lock, 쓰기 락)
- 한 스레드만 데이터에 접근할 수 있습니다.
- 다른 스레드는 공유 락이나 배타 락을 획득할 수 없습니다.
- 데이터 읽기와 수정 모두 허용합니다.
- 주로 쓰기 작업에 사용됩니다.

```cpp
// 읽기-쓰기 락 예제 (C++)
std::shared_mutex rwMutex;

// 읽기 작업 (공유 락 사용)
void readData() {
    std::shared_lock<std::shared_mutex> lock(rwMutex);
    // 데이터 읽기 작업
}

// 쓰기 작업 (배타 락 사용)
void writeData() {
    std::unique_lock<std::shared_mutex> lock(rwMutex);
    // 데이터 쓰기 작업
}
```

### 3.2 낙관적 잠금(Optimistic Locking)과 비관적 잠금(Pessimistic Locking)

#### 낙관적 잠금(Optimistic Locking)
- 충돌이 드물게 발생한다고 가정하고 설계됩니다.
- 리소스에 접근할 때 락을 걸지 않습니다.
- 대신 변경 사항을 적용하기 전에 데이터가 변경되었는지 확인합니다.
- 변경되었다면 충돌 해결 메커니즘(재시도, 병합 등)을 적용합니다.
- 일반적으로 버전 번호나 타임스탬프를 이용하여 구현합니다.
- 충돌이 적고 읽기 작업이 많은 환경에서 성능이 좋습니다.

```cpp
// 낙관적 잠금 예제
struct Data {
    int value;
    int version;
};

bool updateValueOptimistically(Data& data, int newValue, int expectedVersion) {
    // 버전 확인
    if (data.version != expectedVersion) {
        return false; // 버전 불일치, 다른 스레드가 이미 수정함
    }
    
    // 버전 일치, 업데이트 수행
    data.value = newValue;
    data.version++; // 버전 증가
    return true;
}
```

#### 비관적 잠금(Pessimistic Locking)
- 충돌이 자주 발생한다고 가정하고 설계됩니다.
- 리소스에 접근하기 전에 명시적으로 락을 획득합니다.
- 락을 획득할 때까지 다른 스레드는 대기합니다.
- 데이터 일관성을 보장하지만 동시성이 제한됩니다.
- 충돌이 많거나 쓰기 작업이 많은 환경에서 유용합니다.

```cpp
// 비관적 잠금 예제
std::mutex dataMutex;

void updateValuePessimistically(Data& data, int newValue) {
    std::lock_guard<std::mutex> lock(dataMutex); // 락 획득
    
    // 락을 획득한 후 업데이트 수행
    data.value = newValue;
    data.version++;
}
```

## 4. 게임 개발에서의 락 활용 사례: 게임 아이템 캐시 관리

멀티플레이어 온라인 게임에서 아이템 캐시 시스템은 대량의 아이템 데이터를 효율적으로 관리하면서 여러 플레이어의 동시 접근을 처리해야 하는 복잡한 시스템입니다. 이런 환경에서 아이템 데이터의 일관성을 유지하기 위해서는 적절한 락 메커니즘이 필수적입니다. 플레이어가 거래, 장착, 강화, 분해 등 다양한 아이템 관련 작업을 동시에 수행할 때 발생할 수 있는 데이터 손상이나 중복 처리 문제를 방지하기 위해 락이 사용됩니다. 아이템 캐시 시스템은 성능 최적화를 위해 여러 계층의 캐시(메모리 캐시, 분산 캐시, 데이터베이스)를 사용하는 경우가 많으며, 각 계층에서 적절한 락 전략을 구현하는 것이 중요합니다.

```cpp
// 게임 아이템 캐시 관리 시스템에서의 락 활용 예제
class ItemCacheManager {
private:
    struct ItemCacheEntry {
        Item item;
        bool isDirty;
        std::chrono::system_clock::time_point lastAccessed;
        int accessCount;
    };
    
    // 아이템 ID를 기준으로 캐시된 아이템 관리
    std::unordered_map<ItemId, ItemCacheEntry> itemCache;
    
    // 아이템별 락 (세분화된 락 구현)
    std::unordered_map<ItemId, std::shared_mutex> itemMutexes;
    
    // 캐시 전체에 대한 락 (캐시 구조 변경 시 사용)
    std::shared_mutex cacheMutex;
    
    // 주기적인 캐시 동기화를 위한 백그라운드 스레드
    std::thread syncThread;
    std::atomic<bool> isRunning;
    
    // 데이터베이스 연결
    DatabaseConnection dbConn;
    
public:
    ItemCacheManager() : isRunning(true) {
        // 백그라운드 동기화 스레드 시작
        syncThread = std::thread(&ItemCacheManager::syncCacheWithDatabase, this);
    }
    
    ~ItemCacheManager() {
        isRunning = false;
        if (syncThread.joinable()) {
            syncThread.join();
        }
    }
    
    // 아이템 조회 (읽기 작업)
    Item getItem(PlayerId playerId, ItemId itemId) {
        // 먼저 캐시에 아이템이 있는지 확인
        std::shared_lock<std::shared_mutex> cacheLock(cacheMutex);
        auto it = itemCache.find(itemId);
        
        if (it != itemCache.end()) {
            // 캐시에 있는 경우, 아이템별 공유 락 획득
            std::shared_lock<std::shared_mutex> itemLock(getOrCreateItemMutex(itemId));
            cacheLock.unlock(); // 캐시 전체 락 해제
            
            // 캐시 항목 업데이트 (마지막 접근 시간, 접근 횟수)
            it->second.lastAccessed = std::chrono::system_clock::now();
            it->second.accessCount++;
            
            // 권한 검사
            if (!hasItemAccess(playerId, it->second.item)) {
                throw PermissionDeniedException("Player does not have access to this item");
            }
            
            return it->second.item;
        }
        
        // 캐시에 없는 경우, DB에서 로드
        cacheLock.unlock();
        return loadItemFromDatabase(playerId, itemId);
    }
    
    // 아이템 수정 (쓰기 작업)
    void updateItem(PlayerId playerId, ItemId itemId, const ItemUpdateFunction& updateFunc) {
        // 낙관적 락을 위한 재시도 로직
        const int MAX_RETRIES = 3;
        int retries = 0;
        
        while (retries < MAX_RETRIES) {
            // 캐시 확인 및 아이템 로드
            std::shared_lock<std::shared_mutex> cacheLock(cacheMutex);
            auto it = itemCache.find(itemId);
            
            if (it == itemCache.end()) {
                // 아이템이 캐시에 없는 경우 로드
                cacheLock.unlock();
                loadItemFromDatabase(playerId, itemId);
                cacheLock.acquire();
                it = itemCache.find(itemId);
                
                if (it == itemCache.end()) {
                    throw ItemNotFoundException("Item not found after DB load");
                }
            }
            
            // 아이템별 배타적 락 획득
            std::unique_lock<std::shared_mutex> itemLock(getOrCreateItemMutex(itemId));
            cacheLock.unlock();
            
            // 권한 검사
            if (!hasItemAccess(playerId, it->second.item)) {
                throw PermissionDeniedException("Player does not have permission to modify this item");
            }
            
            // 아이템 버전 정보 확인 (낙관적 락)
            int currentVersion = it->second.item.getVersion();
            
            try {
                // 아이템 수정
                Item updatedItem = updateFunc(it->second.item);
                
                // 버전 확인 (낙관적 락)
                if (updatedItem.getVersion() != currentVersion) {
                    retries++;
                    continue; // 다른 스레드가 이미 수정함, 재시도
                }
                
                // 버전 증가 및 아이템 업데이트
                updatedItem.incrementVersion();
                it->second.item = updatedItem;
                it->second.isDirty = true;
                it->second.lastAccessed = std::chrono::system_clock::now();
                
                // 트랜잭션 로그에 기록
                logItemUpdate(playerId, itemId, currentVersion, updatedItem.getVersion());
                
                return; // 성공적으로 업데이트 완료
            } catch (const ConcurrentModificationException& e) {
                // 동시 수정 감지, 재시도
                retries++;
            }
        }
        
        throw TooManyRetriesException("Failed to update item after maximum retries");
    }
    
    // 아이템 거래 (두 플레이어 간 아이템 이동)
    bool tradeItem(PlayerId fromPlayer, PlayerId toPlayer, ItemId itemId) {
        // 데드락 방지를 위해 항상 낮은 ID의 플레이어부터 락 획득
        if (fromPlayer > toPlayer) {
            std::swap(fromPlayer, toPlayer);
        }
        
        // 아이템 배타적 락 획득
        std::unique_lock<std::shared_mutex> itemLock(getOrCreateItemMutex(itemId));
        
        // 아이템 존재 확인 및 권한 검사
        auto it = itemCache.find(itemId);
        if (it == itemCache.end()) {
            // 아이템을 DB에서 로드
            itemLock.unlock();
            loadItemFromDatabase(fromPlayer, itemId);
            itemLock.lock();
            it = itemCache.find(itemId);
            
            if (it == itemCache.end()) {
                return false; // 아이템 없음
            }
        }
        
        // 거래 가능 여부 확인
        if (!it->second.item.isTradable() || it->second.item.getOwnerId() != fromPlayer) {
            return false; // 거래 불가능 또는 소유자 불일치
        }
        
        // 실제 거래 처리 (DB 트랜잭션 포함)
        bool success = executeItemTrade(it->second.item, fromPlayer, toPlayer);
        
        if (success) {
            // 아이템 소유자 변경 및 상태 업데이트
            it->second.item.setOwnerId(toPlayer);
            it->second.item.incrementVersion();
            it->second.isDirty = true;
            it->second.lastAccessed = std::chrono::system_clock::now();
            
            // 거래 로그 기록
            logItemTrade(itemId, fromPlayer, toPlayer);
        }
        
        return success;
    }
    
private:
    // 아이템별 뮤텍스 관리 (필요에 따라 생성)
    std::shared_mutex& getOrCreateItemMutex(ItemId itemId) {
        std::unique_lock<std::shared_mutex> lock(cacheMutex);
        return itemMutexes[itemId]; // 없으면 자동 생성
    }
    
    // DB에서 아이템 로드 및 캐시에 추가
    Item loadItemFromDatabase(PlayerId playerId, ItemId itemId) {
        // 아이템별 락 획득 (다른 스레드가 동시에 같은 아이템을 로드하지 않도록)
        std::unique_lock<std::shared_mutex> itemLock(getOrCreateItemMutex(itemId));
        
        // 다시 한번 캐시 확인 (다른 스레드가 이미 로드했을 수 있음)
        std::shared_lock<std::shared_mutex> cacheLock(cacheMutex);
        auto it = itemCache.find(itemId);
        if (it != itemCache.end()) {
            return it->second.item;
        }
        cacheLock.unlock();
        
        // DB에서 아이템 로드
        Item item = dbConn.loadItem(itemId);
        
        // 권한 검사
        if (!hasItemAccess(playerId, item)) {
            throw PermissionDeniedException("Player does not have access to this item");
        }
        
        // 캐시에 아이템 추가
        std::unique_lock<std::shared_mutex> writeCacheLock(cacheMutex);
        itemCache[itemId] = {
            item,
            false, // dirty 플래그 초기화
            std::chrono::system_clock::now(),
            1 // 접근 횟수 초기화
        };
        
        return item;
    }
    
    // 백그라운드 동기화 스레드 - 주기적으로 dirty 캐시 항목을 DB에 기록
    void syncCacheWithDatabase() {
        while (isRunning) {
            // 5초마다 동기화 수행
            std::this_thread::sleep_for(std::chrono::seconds(5));
            
            std::vector<ItemId> dirtyItems;
            
            // 캐시에서 dirty 아이템 목록 수집
            {
                std::shared_lock<std::shared_mutex> cacheLock(cacheMutex);
                for (const auto& pair : itemCache) {
                    if (pair.second.isDirty) {
                        dirtyItems.push_back(pair.first);
                    }
                }
            }
            
            // 각 dirty 아이템을 DB에 저장
            for (ItemId itemId : dirtyItems) {
                try {
                    // 아이템별 락 획득
                    std::unique_lock<std::shared_mutex> itemLock(getOrCreateItemMutex(itemId));
                    
                    // 캐시에서 아이템 확인
                    auto it = itemCache.find(itemId);
                    if (it != itemCache.end() && it->second.isDirty) {
                        // DB에 아이템 저장
                        dbConn.saveItem(it->second.item);
                        it->second.isDirty = false;
                    }
                } catch (const std::exception& e) {
                    // 동기화 오류 로깅
                    logSyncError(itemId, e.what());
                }
            }
            
            // 캐시 정리 (오래된 항목 제거)
            pruneCacheItems();
        }
    }
    
    // 캐시 정리 - 오래된 항목 또는 접근 빈도가 낮은 항목 제거
    void pruneCacheItems() {
        std::unique_lock<std::shared_mutex> cacheLock(cacheMutex);
        
        auto now = std::chrono::system_clock::now();
        std::vector<ItemId> itemsToRemove;
        
        // 30분 이상 접근이 없고, dirty가 아닌 항목 제거
        for (const auto& pair : itemCache) {
            auto elapsed = std::chrono::duration_cast<std::chrono::minutes>(
                now - pair.second.lastAccessed).count();
                
            if (!pair.second.isDirty && elapsed > 30) {
                itemsToRemove.push_back(pair.first);
            }
        }
        
        // 실제 캐시에서 제거
        for (ItemId itemId : itemsToRemove) {
            itemCache.erase(itemId);
            itemMutexes.erase(itemId);
        }
    }
};
```

이 게임 아이템 캐시 관리 시스템은 다양한 락 관련 개념을 적절히 조합하여 구현되었습니다. 전체 캐시 구조에 대한 공유 락과 각 아이템별 세분화된 락을 사용함으로써 동시성을 최대화하면서도 데이터 일관성을 보장합니다. 특히 자주 읽히는 아이템 데이터에 대해서는 공유 락을 통해 여러 스레드가 동시에 읽을 수 있도록 하고, 아이템 수정 작업에는 배타 락을 사용하여 데이터 무결성을 유지합니다. 또한 낙관적 잠금 기법을 적용하여 버전 정보를 이용한 동시 수정 감지 및 충돌 해결을 구현했습니다. 데드락 방지를 위해 플레이어 ID 기반의 일관된 락 획득 순서를 정하고, 백그라운드 동기화 스레드를 통해 변경된 아이템 데이터를 주기적으로 데이터베이스에 반영함으로써 성능과 안정성의 균형을 맞추고 있습니다. 이 시스템은 수많은 플레이어가 동시에 아이템을 조회하고 수정하는 대규모 온라인 게임에서 발생할 수 있는 데이터 경쟁 문제를 효과적으로 해결하면서도, 불필요한 락으로 인한 병목 현상을 최소화하는 효율적인 구조를 제공합니다.
>>>>>>> ca8516e (류병현 4월 5주차 락)
