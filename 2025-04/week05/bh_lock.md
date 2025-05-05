국내 락 추천 : 터치드 - Hi Bully
# 락(Lock)에 대해서 알아보자.

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