# Go 문법 정리: iota와 RWMutex

## 1. iota - 연속 상수 자동 생성

### iota란?

`const` 블록 내에서 연속된 정수 상수를 자동으로 생성해주는 Go 키워드이다.

### 기본 사용법

```go
const (
    Idle Phase = iota  // 0
    PrePrepared        // 1
    Prepared           // 2
    Committed          // 3
)
```

위 코드는 아래와 동일하다:

```go
const (
    Idle        Phase = 0
    PrePrepared Phase = 1
    Prepared    Phase = 2
    Committed   Phase = 3
)
```

### iota 동작 규칙

1. `const` 블록이 시작되면 iota는 0으로 초기화된다
2. 같은 블록 내에서 줄이 바뀔 때마다 1씩 증가한다
3. 새로운 `const` 블록이 시작되면 다시 0부터 시작한다

```go
const (
    A = iota  // 0
    B         // 1
    C         // 2
)

const (
    X = iota  // 0 (새 블록이므로 리셋)
    Y         // 1
)
```

### 응용 패턴

#### 시작값 조정

```go
const (
    A = iota + 10  // 10
    B              // 11
    C              // 12
)
```

#### 값 건너뛰기

```go
const (
    A = iota  // 0
    _         // 1 (blank identifier로 버림)
    B         // 2
)
```

#### 비트 플래그

```go
const (
    Read    = 1 << iota  // 1  (0001)
    Write                // 2  (0010)
    Execute              // 4  (0100)
)
```

#### 곱셈 패턴

```go
const (
    KB = 1 << (10 * iota)  // 1
    MB                      // 1024
    GB                      // 1048576
)
```

### 주의사항

- iota는 Go 예약어가 아닌 미리 선언된 식별자(predeclared identifier)이다
- `const` 블록 외부에서는 사용할 수 없다
- 상수 이름(Idle, Prepared 등)은 개발자가 정의하는 것이며, Go 내장 타입이 아니다

---

## 2. RWMutex - 읽기/쓰기 분리 잠금

### sync.Mutex vs sync.RWMutex

| 타입 | 메서드 | 특징 |
|------|--------|------|
| `sync.Mutex` | `Lock()`, `Unlock()` | 읽기/쓰기 구분 없음 |
| `sync.RWMutex` | `Lock()`, `Unlock()`, `RLock()`, `RUnlock()` | 읽기/쓰기 구분 |

### RWMutex 메서드

| 메서드 | 용도 | 동시 접근 |
|--------|------|-----------|
| `Lock()` | 쓰기 잠금 획득 | 단독 접근만 허용 |
| `Unlock()` | 쓰기 잠금 해제 | - |
| `RLock()` | 읽기 잠금 획득 | 여러 읽기 동시 허용 |
| `RUnlock()` | 읽기 잠금 해제 | - |

### 동시성 규칙

1. 여러 고루틴이 동시에 `RLock()`을 획득할 수 있다
2. `Lock()`은 모든 `RLock()`이 해제될 때까지 대기한다
3. `Lock()`이 획득되면 다른 `Lock()`과 `RLock()` 모두 대기한다

### 동작 다이어그램

#### 읽기만 있는 경우

```
Goroutine A: RLock() ================ RUnlock()
Goroutine B:    RLock() ================ RUnlock()
Goroutine C:        RLock() ================ RUnlock()

--> 동시 실행 가능
```

#### 쓰기가 끼어드는 경우

```
Goroutine A (읽기): RLock() ======== RUnlock()
Goroutine B (읽기):    RLock() ========= RUnlock()
Goroutine C (쓰기):          [대기] Lock() ======== Unlock()
Goroutine D (읽기):          [대기]                  RLock() ===

--> 쓰기는 모든 읽기 완료 후 실행
--> 쓰기 중에는 읽기도 대기
```

### 사용 패턴

```go
type State struct {
    mu   sync.RWMutex
    data map[string]int
}

// 읽기 작업: RLock 사용
func (s *State) Get(key string) int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.data[key]
}

// 읽기 작업: RLock 사용
func (s *State) Count() int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return len(s.data)
}

// 쓰기 작업: Lock 사용
func (s *State) Set(key string, value int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.data[key] = value
}

// 쓰기 작업: Lock 사용
func (s *State) Delete(key string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.data, key)
}
```

### 언제 RWMutex를 사용하는가

RWMutex가 유리한 경우:
- 읽기 작업이 쓰기 작업보다 훨씬 많을 때
- 읽기 작업이 동시에 많이 발생할 때

Mutex가 나은 경우:
- 읽기/쓰기 비율이 비슷할 때
- 잠금 획득 시간이 매우 짧을 때 (RWMutex 오버헤드가 더 클 수 있음)

### 주의사항

1. `RLock()` 후에는 반드시 `RUnlock()`을 호출해야 한다
2. `Lock()` 후에는 반드시 `Unlock()`을 호출해야 한다
3. `RLock()`과 `Unlock()`을 섞어 쓰면 안 된다 (짝이 맞아야 함)
4. 같은 고루틴에서 `RLock()`을 중첩 호출하면 데드락 위험이 있다

```go
// 잘못된 예시 - 데드락 발생
func (s *State) Bad() {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    s.mu.Lock()  // 데드락! RLock 해제를 기다리지만 같은 고루틴
    defer s.mu.Unlock()
}
```

---

## 참고 자료

- Go 공식 문서: https://golang.org/pkg/sync/
- Effective Go: https://golang.org/doc/effective_go
