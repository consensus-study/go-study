Lock-free 자료구조를 구현하려면 `atomic`과 `unsafe.Pointer`를 이해해야 한다. 기초부터 차근차근 정리한다.

---

## 1. 왜 atomic이 필요한가

### 동시성 문제: 은행 계좌 예시

```go
var balance int = 1000  // 잔액 1000원
```

두 사람이 동시에 500원씩 출금한다.

```go
// 사람 A (고루틴 A)
if balance >= 500 {
    balance = balance - 500
}

// 사람 B (고루틴 B)  
if balance >= 500 {
    balance = balance - 500
}
```

기대하는 결과: 1000 - 500 - 500 = 0원

실제로 일어날 수 있는 일:

```
시간  | 사람 A              | 사람 B              | balance
─────┼────────────────────┼────────────────────┼─────────
  1  | balance 읽음 (1000) |                    | 1000
  2  |                    | balance 읽음 (1000) | 1000
  3  | 1000 >= 500? Yes   |                    | 1000
  4  |                    | 1000 >= 500? Yes   | 1000
  5  | 1000-500 = 500     |                    | 1000
  6  |                    | 1000-500 = 500     | 1000
  7  | balance에 500 씀   |                    | 500
  8  |                    | balance에 500 씀   | 500  ← 문제!
```

결과: 500원 (0원이어야 하는데!)

둘 다 "아직 1000원 있네, 출금해도 되겠다"라고 판단해버렸다. `balance = balance - 500`이 사실은 세 단계이기 때문이다.

```
1. balance 읽기
2. 500 빼기
3. balance에 쓰기
```

이 중간에 다른 고루틴이 끼어들 수 있다.

---

## 2. 해결책 1: mutex (자물쇠)

```go
var balance int = 1000
var mu sync.Mutex

// 사람 A
mu.Lock()  // 자물쇠 잠금 (다른 사람 접근 차단)
if balance >= 500 {
    balance = balance - 500
}
mu.Unlock()  // 자물쇠 해제

// 사람 B
mu.Lock()  // A가 Unlock할 때까지 여기서 대기
if balance >= 500 {
    balance = balance - 500
}
mu.Unlock()
```

```
시간  | 사람 A              | 사람 B              | balance
─────┼────────────────────┼────────────────────┼─────────
  1  | Lock() 성공         |                    | 1000
  2  | balance 읽음 (1000) | Lock() 대기중...   | 1000
  3  | 1000 >= 500? Yes   | 대기중...          | 1000
  4  | balance = 500      | 대기중...          | 500
  5  | Unlock()           |                    | 500
  6  |                    | Lock() 성공!       | 500
  7  |                    | balance 읽음 (500) | 500
  8  |                    | 500 >= 500? Yes   | 500
  9  |                    | balance = 0       | 0  ← 정상!
```

정상 동작한다. 근데 문제가 있다.

```
단점:
- B가 기다려야 함 (대기 시간 발생)
- Lock/Unlock 자체가 비용이 있음
- 고루틴이 많으면 병목이 됨
```

---

## 3. 해결책 2: atomic (하드웨어 레벨 동기화)

CPU는 특정 연산을 "쪼개지지 않게" 실행할 수 있다. 이걸 atomic 연산이라고 한다.

```go
var counter int64 = 0

// 고루틴 A
atomic.AddInt64(&counter, 1)  // 읽기+더하기+쓰기를 한 번에

// 고루틴 B
atomic.AddInt64(&counter, 1)  // 얘도 한 번에
```

CPU한테 "이 연산하는 동안 다른 코어가 끼어들지 마"라고 지시하는 거다.

```
비유:
- mutex = 소프트웨어 자물쇠 (느림)
- atomic = 하드웨어 자물쇠 (빠름)
```

### atomic 연산 종류

```go
// Load: 안전하게 읽기
val := atomic.LoadInt64(&counter)

// Store: 안전하게 쓰기  
atomic.StoreInt64(&counter, 100)

// Add: 안전하게 더하기
atomic.AddInt64(&counter, 1)

// Swap: 값 교체하고 이전 값 반환
old := atomic.SwapInt64(&counter, 200)

// CompareAndSwap: 조건부 교체 (가장 중요!)
swapped := atomic.CompareAndSwapInt64(&counter, old, new)
```

---

## 4. CAS (Compare-And-Swap) - 가장 중요한 연산

```go
atomic.CompareAndSwapInt64(&value, old, new)
```

한국어로: **"비교하고-같으면-바꿔라"**

```
CAS(주소, 기대값, 새값)

만약 주소에 있는 값이 기대값과 같으면:
    → 새값으로 바꾸고 true 반환
아니면:
    → 아무것도 안 하고 false 반환
    
이 전체가 CPU 명령어 하나로 실행됨 (쪼개지지 않음)
```

### CAS로 출금 구현

```go
var balance int64 = 1000

// 500원 출금 시도
func withdraw(amount int64) bool {
    for {
        // 1. 현재 잔액 읽기
        current := atomic.LoadInt64(&balance)
        
        // 2. 잔액 체크
        if current < amount {
            return false  // 잔액 부족
        }
        
        // 3. "아직 current원이면, current-amount로 바꿔라"
        if atomic.CompareAndSwapInt64(&balance, current, current-amount) {
            return true  // 성공!
        }
        
        // 4. 실패하면 다시 시도 (누가 중간에 바꿔버린 것)
    }
}
```

두 사람이 동시에 실행하면:

```
시간  | 사람 A                    | 사람 B                    | balance
─────┼──────────────────────────┼──────────────────────────┼─────────
  1  | current = 1000           |                          | 1000
  2  |                          | current = 1000           | 1000
  3  | CAS(1000, 500) 시도      | CAS(1000, 500) 시도      | 1000
  4  | 성공! balance=500        | 실패! (이미 500임)       | 500
  5  |                          | 다시 시도...             | 500
  6  |                          | current = 500            | 500
  7  |                          | CAS(500, 0) 시도         | 500
  8  |                          | 성공! balance=0          | 0
```

둘 다 출금 성공, 결과도 정상이다.

핵심: CAS는 **"내가 읽은 값이 아직 그대로면 바꿔라"**다. 누가 중간에 바꿨으면 실패하고 다시 시도한다.

---

## 5. unsafe.Pointer가 필요한 이유

### Go의 타입 안전성

Go는 타입이 다른 포인터끼리 변환이 안 된다.

```go
var a int = 10
var b float64 = 3.14

var p *int = &a
p = &b  // 컴파일 에러! *int에 *float64를 못 넣음
```

이건 보통 좋은 거다. 실수를 막아주니까.

### 문제: atomic 함수는 특정 타입만 받는다

```go
// atomic 패키지의 포인터 관련 함수들
atomic.LoadPointer(addr *unsafe.Pointer) unsafe.Pointer
atomic.StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
atomic.CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) bool
```

전부 `unsafe.Pointer` 타입만 받는다. `*Node`를 안 받는다.

왜? Go에는 수천 가지 포인터 타입이 있을 수 있다. `*int`, `*string`, `*Node`, `*MyStruct`... 이걸 다 지원하려면 함수가 수천 개 필요하다.

그래서 "아무 포인터나 다 되는 범용 타입"을 만들었다. 그게 `unsafe.Pointer`.

### unsafe.Pointer 사용법

```go
// 아무 포인터나 unsafe.Pointer로 변환 가능
var node *Node = &Node{value: 10}
var ptr unsafe.Pointer = unsafe.Pointer(node)

// 다시 원래 타입으로 변환 가능
var node2 *Node = (*Node)(ptr)
```

```
*Node  →  unsafe.Pointer  →  *Node
         (중간 다리 역할)
```

### 왜 "unsafe"인가?

타입 검사를 안 해서 위험하다.

```go
var node *Node = &Node{value: 10}
var ptr unsafe.Pointer = unsafe.Pointer(node)

// 이것도 컴파일됨 (근데 완전 잘못됨!)
var str *string = (*string)(ptr)  // Node를 string으로 해석하려고 함
fmt.Println(*str)  // 쓰레기 값 또는 크래시
```

컴파일러가 안 막아준다. 프로그래머가 알아서 조심해야 한다. 그래서 이름이 "unsafe".

---

## 6. Lock-free Linked List 구현

이제 배운 걸 조합해서 lock-free linked list를 만들어보자.

### 기본 구조

```go
type Node struct {
    value int
    next  unsafe.Pointer  // *Node 대신 unsafe.Pointer 사용
}

type List struct {
    head unsafe.Pointer  // 첫 번째 노드를 가리킴
}

func NewList() *List {
    return &List{}
}
```

왜 `next`가 `unsafe.Pointer`인가? `atomic.CompareAndSwapPointer`를 쓰려고.

### 헬퍼 함수

unsafe.Pointer 변환이 번거로우니까 헬퍼를 만든다.

```go
// unsafe.Pointer → *Node
func loadNode(p *unsafe.Pointer) *Node {
    return (*Node)(atomic.LoadPointer(p))
}

// *Node → unsafe.Pointer로 저장
func storeNode(p *unsafe.Pointer, node *Node) {
    atomic.StorePointer(p, unsafe.Pointer(node))
}
```

### 삽입 (맨 앞에)

```go
func (list *List) InsertFront(value int) {
    newNode := &Node{value: value}
    
    for {
        // 1. 현재 head 읽기
        oldHead := atomic.LoadPointer(&list.head)
        
        // 2. 새 노드의 next를 현재 head로 설정
        newNode.next = oldHead
        
        // 3. CAS로 head를 새 노드로 변경 시도
        if atomic.CompareAndSwapPointer(
            &list.head,               // 이 주소의 값이
            oldHead,                  // 아직 이거면
            unsafe.Pointer(newNode),  // 이걸로 바꿔라
        ) {
            return  // 성공!
        }
        // 실패하면 루프 돌아서 다시 시도
    }
}
```

### 삽입 과정 그림

```
[초기 상태]
head → [10] → [20] → nil

[고루틴 A: 5 삽입, 고루틴 B: 7 삽입 동시 시도]

고루틴 A:
  1. oldHead = 주소([10])
  2. newNode = [5], newNode.next = 주소([10])
  
     head → [10] → [20] → nil
             ↑
     [5] ────┘
     
고루틴 B:
  1. oldHead = 주소([10])  
  2. newNode = [7], newNode.next = 주소([10])
  
     head → [10] → [20] → nil
             ↑
     [7] ────┘
```

둘 다 준비 완료. 이제 CAS 경쟁:

```
[CAS 경쟁 - 둘 중 하나만 성공]

고루틴 A: CAS(head, 주소([10]), 주소([5])) → 성공!
고루틴 B: CAS(head, 주소([10]), 주소([7])) → 실패! (head가 이미 [5]를 가리킴)

[A 성공 후 상태]
head → [5] → [10] → [20] → nil

[고루틴 B 재시도]
  1. oldHead = 주소([5])  ← 새로 읽음
  2. newNode.next = 주소([5])
  
     head → [5] → [10] → [20] → nil
             ↑
     [7] ────┘

  3. CAS(head, 주소([5]), 주소([7])) → 성공!

[최종 결과]
head → [7] → [5] → [10] → [20] → nil
```

mutex 없이 두 고루틴이 안전하게 삽입했다.

### 탐색

탐색은 읽기만 하니까 간단하다.

```go
func (list *List) Contains(value int) bool {
    current := loadNode(&list.head)
    
    for current != nil {
        if current.value == value {
            return true
        }
        current = loadNode(&current.next)
    }
    
    return false
}

func (list *List) Print() {
    current := loadNode(&list.head)
    
    for current != nil {
        fmt.Printf("%d → ", current.value)
        current = loadNode(&current.next)
    }
    fmt.Println("nil")
}
```

---

## 7. 전체 코드

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "unsafe"
)

type Node struct {
    value int
    next  unsafe.Pointer
}

type List struct {
    head unsafe.Pointer
}

func NewList() *List {
    return &List{}
}

func loadNode(p *unsafe.Pointer) *Node {
    return (*Node)(atomic.LoadPointer(p))
}

func (list *List) InsertFront(value int) {
    newNode := &Node{value: value}
    
    for {
        oldHead := atomic.LoadPointer(&list.head)
        newNode.next = oldHead
        
        if atomic.CompareAndSwapPointer(&list.head, oldHead, unsafe.Pointer(newNode)) {
            return
        }
    }
}

func (list *List) Contains(value int) bool {
    current := loadNode(&list.head)
    
    for current != nil {
        if current.value == value {
            return true
        }
        current = loadNode(&current.next)
    }
    
    return false
}

func (list *List) Print() {
    current := loadNode(&list.head)
    
    for current != nil {
        fmt.Printf("%d → ", current.value)
        current = loadNode(&current.next)
    }
    fmt.Println("nil")
}

func main() {
    list := NewList()
    
    // 동시에 삽입 테스트
    var wg sync.WaitGroup
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(val int) {
            defer wg.Done()
            list.InsertFront(val)
        }(i)
    }
    
    wg.Wait()
    
    list.Print()
    fmt.Println("Contains 5:", list.Contains(5))
    fmt.Println("Contains 100:", list.Contains(100))
}
```

---

## 8. Lock-free vs Mutex 비교

```
Mutex:
  - 구현 쉬움
  - 대기 발생 (blocking)
  - 데드락 가능성
  - 고루틴 많으면 병목

Lock-free (atomic + CAS):
  - 구현 어려움
  - 대기 없음 (non-blocking)
  - 데드락 없음
  - 고루틴 많아도 확장성 좋음
```

---

## 9. 주의사항

### ABA 문제

```
1. 고루틴 A: 값 읽음 (A)
2. 고루틴 B: A → B → A 로 바꿈
3. 고루틴 A: CAS(기대값=A, 새값=X) → 성공!

문제: A가 "같은 A"인 줄 알았는데 사실 "다른 A"였음
```

삽입에서는 큰 문제 없지만, 삭제에서는 문제가 된다. 해결책:
- 버전 번호 붙이기 (tagged pointer)
- Hazard Pointer
- Epoch-based reclamation

### 메모리 순서 (Memory Ordering)

Go의 atomic은 sequentially consistent라서 비교적 안전하다. 하지만 복잡한 lock-free 구조에서는 메모리 순서 문제가 생길 수 있다.

### GC와의 상호작용

Go는 GC가 있어서 삭제된 노드의 메모리 관리가 상대적으로 쉽다. C/C++에서는 삭제가 훨씬 복잡하다.

---

## 10. 정리

```
포인터 기초:
  - &변수 = 주소 얻기
  - *포인터 = 값 얻기

unsafe.Pointer:
  - "아무 포인터나 담을 수 있는 타입"
  - atomic 포인터 함수가 이 타입만 받음
  - 타입 검사 안 해서 위험함 (그래서 unsafe)
  
  사용법:
    *Node → unsafe.Pointer → *Node
           (변환)         (변환)

atomic:
  - CPU가 "한 번에" 처리하는 연산
  - 중간에 다른 코어가 끼어들지 못함
  - mutex보다 빠름
  
  주요 함수:
    Load    - 안전하게 읽기
    Store   - 안전하게 쓰기
    Add     - 안전하게 더하기
    Swap    - 교체하고 이전 값 반환
    CAS     - 비교하고 같으면 교체

CAS (Compare-And-Swap):
  - "기대값이면 바꾸고 true, 아니면 false"
  - lock-free의 핵심 연산
  
  패턴:
    for {
        old := atomic.Load(&ptr)
        new := 새로운값
        if atomic.CAS(&ptr, old, new) {
            break  // 성공
        }
        // 실패하면 재시도
    }

Lock-free Linked List:
  - next 포인터를 unsafe.Pointer로 선언
  - 삽입: CAS로 head 또는 prev.next 변경
  - 실패하면 다시 읽고 재시도
```

삭제 연산은 더 복잡하다. 노드를 삭제하면서 동시에 다른 고루틴이 그 노드를 탐색 중일 수 있기 때문이다. 그건 다음에 다루자.
