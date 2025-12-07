이전 글에서 sync.Pool이 GC 때 "강제로 비워진다"고 썼다. 참조가 있는데 왜 비워지는지 의문이 들 수 있다. 이 글에서는 그 이유를 Go GC 런타임 내부까지 파고들어 설명한다.

---

## Go GC의 기본 특성

Go의 GC는 다음 세 가지 특성을 가진다.

**Non-generational (비세대)**

Java나 .NET처럼 객체를 Young/Old 세대로 나누지 않는다. 모든 객체를 동등하게 취급한다. 세대 구분의 복잡성을 피하고, Go 프로그램의 특성상 세대별 GC가 큰 이점을 보이지 않았기 때문이다.

**Non-compacting (비압축)**

GC 후 메모리를 재배치하지 않는다. 객체가 힙에서 이동하지 않으므로 포인터 업데이트 비용이 없다. 대신 메모리 단편화가 발생할 수 있는데, Go는 tcmalloc 기반의 메모리 할당기로 이를 완화한다.

**Concurrent (동시성)**

대부분의 GC 작업이 애플리케이션과 동시에 실행된다. Stop-The-World(STW) 구간을 최소화해서 지연 시간을 줄인다.

---

## Tricolor Mark and Sweep 알고리즘

Go GC의 핵심은 삼색 마킹 알고리즘이다.

### 세 가지 색의 의미

```
White (흰색): 아직 방문하지 않은 객체. GC 사이클 시작 시 모든 객체가 흰색.
             마킹이 끝났을 때 여전히 흰색이면 수거 대상.

Gray (회색):  방문했지만 참조하는 객체들을 아직 검사하지 않은 객체.
             "처리 대기 큐"에 있는 상태.

Black (검은색): 방문했고 참조하는 객체들도 모두 검사한 객체.
               확실히 살아있는 객체.
```

### 마킹 과정

```
1. 초기 상태
   모든 객체: 흰색
   
2. 루트 스캔
   전역 변수, 고루틴 스택의 포인터들 → 회색으로 마킹
   
3. 마킹 루프
   while (회색 객체가 남아있음) {
       회색 객체 하나를 꺼냄
       이 객체가 참조하는 모든 흰색 객체를 회색으로 변경
       이 객체를 검은색으로 변경
   }
   
4. 스윕
   여전히 흰색인 객체들의 메모리를 회수
```

시각적으로 표현하면 다음과 같다.

```
[시작]
Root → A(white) → B(white) → C(white)
              ↘ D(white)

[루트 스캔 후]
Root → A(gray) → B(white) → C(white)
             ↘ D(white)

[A 처리 후]
Root → A(black) → B(gray) → C(white)
              ↘ D(gray)

[B, D 처리 후]
Root → A(black) → B(black) → C(gray)
              ↘ D(black)

[C 처리 후]
Root → A(black) → B(black) → C(black)
              ↘ D(black)

모든 객체가 검은색 → 흰색 객체 없음 → 수거할 것 없음
```

만약 A → D 참조가 중간에 끊어졌다면?

```
[A가 D 참조를 끊은 상태에서 마킹 완료]
Root → A(black) → B(black) → C(black)

D(white) ← 아무도 참조 안 함

D는 흰색으로 남음 → 스윕 단계에서 회수
```

---

## 동시성 GC의 문제: 참조 변경

GC가 마킹하는 동안 애플리케이션도 실행된다. 이때 문제가 생길 수 있다.

### 객체 손실 문제

```
상황:
1. GC가 A를 검은색으로 마킹 완료
2. GC가 B를 처리하려고 함
3. 이 순간 애플리케이션이:
   - B → C 참조를 끊음
   - A → C 참조를 추가함

결과:
A(black) → C(white)  ← 검은색이 흰색을 참조!
B(gray)              ← C 참조가 없어졌으므로 C를 회색으로 안 만듦

C는 살아있어야 하는데 흰색으로 남음 → 잘못 수거됨 → 버그!
```

이 문제를 해결하기 위해 Write Barrier가 필요하다.

---

## Write Barrier

Write Barrier는 포인터가 변경될 때 GC에게 알려주는 메커니즘이다.

### 삼색 불변성 (Tricolor Invariant)

GC의 정확성을 보장하기 위해 다음 조건 중 하나를 유지해야 한다.

**Strong Tricolor Invariant (강한 불변성)**
```
검은색 객체는 흰색 객체를 직접 참조할 수 없다.
```

**Weak Tricolor Invariant (약한 불변성)**
```
검은색 객체가 흰색 객체를 참조해도 되지만,
그 흰색 객체는 회색 객체로부터 도달 가능해야 한다.
(회색 객체가 보호해주는 상태)
```

### Dijkstra Write Barrier (삽입 장벽)

Go 1.7 이전에 사용했다.

```
writePointer(slot, ptr):
    shade(ptr)      // ptr이 가리키는 객체를 회색으로
    *slot = ptr
```

새로 참조되는 객체를 회색으로 마킹한다. 강한 불변성을 보장한다.

문제점: 스택에서의 포인터 쓰기에도 장벽을 적용해야 하는데, 이 비용이 크다. 대안으로 스택을 "영구 회색"으로 취급하면 GC 마지막에 스택을 다시 스캔해야 한다.

### Yuasa Write Barrier (삭제 장벽)

```
writePointer(slot, ptr):
    shade(*slot)    // 기존에 참조하던 객체를 회색으로
    *slot = ptr
```

참조가 끊어지는 객체를 보호한다. 삭제되는 참조의 대상을 회색으로 마킹해서 이번 GC에서 수거되지 않게 한다.

문제점: 정밀도가 낮다. 이미 죽은 객체도 이번 GC에서 살아남을 수 있다.

### Hybrid Write Barrier (혼합 장벽)

Go 1.8부터 사용한다. 두 장벽의 장점을 결합했다.

```
writePointer(slot, ptr):
    shade(*slot)              // 기존 참조 대상을 회색으로
    if any stack is grey:
        shade(ptr)            // 새 참조 대상도 회색으로
    *slot = ptr
```

핵심 아이디어는 다음과 같다.

- 힙 객체에 대한 쓰기: 기존 값과 새 값 모두 보호
- 스택: 한 번 스캔하면 다시 스캔 불필요
- 새로 할당되는 객체: 검은색으로 시작

이 방식으로 Mark Termination 단계의 STW 시간을 크게 줄였다.

---

## Write Barrier 실제 코드

Go 컴파일러는 힙 포인터 쓰기에 자동으로 Write Barrier를 삽입한다.

```go
var sink *int

func foo() {
    x := new(int)
    sink = x  // 여기에 Write Barrier 삽입됨
}
```

컴파일된 어셈블리를 보면 다음과 같다.

```asm
CMPL runtime.writeBarrier(SB), $0    // Write Barrier가 활성화되어 있나?
JNE  barrier_enabled                 // 활성화됐으면 점프
MOVQ CX, "".sink(SB)                 // 아니면 그냥 쓰기
JMP  done
barrier_enabled:
    LEAQ "".sink(SB), DI
    CALL runtime.gcWriteBarrierCX(SB) // GC에게 알림
done:
```

`runtime.writeBarrier`는 전역 변수다. GC의 Mark 단계에서만 1로 설정된다. 대부분의 시간에는 0이므로 추가 비용 없이 바로 쓰기가 실행된다.

---

## GC 단계별 동작

Go GC는 네 단계로 나뉜다.

### 1. Sweep Termination

이전 GC 사이클의 스윕이 완료되지 않았다면 완료한다. 새 사이클 시작 전 정리 작업이다.

### 2. Mark (STW 시작점 포함)

```
[STW 시작]
- 모든 P를 safe-point로 이동
- Write Barrier 활성화
- GC 상태를 _GCmark로 변경
- 루트 마킹 준비
[STW 종료]

[동시 실행]
- 백그라운드에서 마킹 워커 실행
- 애플리케이션도 동시에 실행
- Write Barrier가 참조 변경 추적
- 필요시 mutator assist (애플리케이션이 마킹 작업 도움)
```

### 3. Mark Termination (STW)

```
[STW 시작]
- 마킹 작업 완료 확인
- Write Barrier 비활성화
- 다음 GC 트리거 조건 계산
[STW 종료]
```

### 4. Sweep (동시 실행)

흰색 객체들의 메모리를 회수한다. 애플리케이션과 동시에 실행되며, 새 할당이 필요할 때 lazy하게 스윕한다.

---

## STW 시간 측정

실제 STW 시간을 확인해보자.

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // GC 통계 활성화
    runtime.SetGCPercent(100)
    
    // 메모리 할당
    var data [][]byte
    for i := 0; i < 1000000; i++ {
        data = append(data, make([]byte, 100))
    }
    
    // GC 실행
    start := time.Now()
    runtime.GC()
    elapsed := time.Since(start)
    
    var stats runtime.MemStats
    runtime.ReadMemStats(&stats)
    
    fmt.Printf("GC 횟수: %d\n", stats.NumGC)
    fmt.Printf("총 STW 시간: %v\n", time.Duration(stats.PauseTotalNs))
    fmt.Printf("마지막 GC 시간: %v\n", elapsed)
    
    _ = data
}
```

일반적으로 STW 시간은 수십 마이크로초에서 수 밀리초 사이다. Go 런타임은 이 시간을 최소화하기 위해 계속 개선되고 있다.

---

## sync.Pool과 GC의 관계

이제 본론이다. sync.Pool이 GC 때 비워지는 이유를 runtime 소스 코드 수준에서 살펴보자.

### poolCleanup 함수

sync/pool.go에 정의된 함수다.

```go
func poolCleanup() {
    // This function is called with the world stopped,
    // at the beginning of a garbage collection.
    // It must not allocate and probably should not call
    // any runtime functions.

    // 1. 이전 victim 캐시 완전 삭제
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }

    // 2. 현재 local 캐시를 victim으로 이동
    for _, p := range allPools {
        p.victim = p.local
        p.victimSize = p.localSize
        p.local = nil
        p.localSize = 0
    }

    // 3. allPools를 oldPools로 교체
    oldPools, allPools = allPools, nil
}
```

### 호출 시점

poolCleanup은 init 함수에서 런타임에 등록된다.

```go
func init() {
    runtime_registerPoolCleanup(poolCleanup)
}
```

runtime/mgc.go에서 GC 시작 시 clearpools를 호출한다.

```go
// gcStart 내부
func gcStart(trigger gcTrigger) {
    // ...
    
    // clearpools before we start the GC
    clearpools()
    
    // ...
}
```

clearpools는 등록된 poolCleanup을 호출한다. 이 호출은 STW 상태에서 이루어진다.

### Victim Cache 메커니즘

Go 1.13부터 도입됐다. GC 한 번에 전부 비우지 않고 2단계로 진행한다.

```
[GC 사이클 1]
local:  [buf1, buf2, buf3]
victim: []

poolCleanup 실행 후:
local:  []
victim: [buf1, buf2, buf3]  ← local이 victim으로 이동

[GC 사이클 2]
local:  [buf4]  ← 새로 Put된 것
victim: [buf1, buf2, buf3]

poolCleanup 실행 후:
local:  []
victim: [buf4]
(buf1, buf2, buf3은 삭제됨)  ← 이전 victim이 비워짐
```

Get 호출 시 local이 비어있으면 victim에서 가져온다. 이 방식으로 GC 직후 성능 저하를 완화한다.

### 왜 참조가 있는데 비워지나?

일반적인 GC 원리와 다르다. 참조 체인을 보면 다음과 같다.

```
pool (전역 변수)
  → sync.Pool 구조체 (참조됨)
      → local (참조됨)
          → poolLocal 배열 (참조됨)
              → private, shared의 버퍼들 (참조됨)
```

참조가 연결되어 있으므로 일반적인 GC라면 버퍼들이 수거되지 않아야 한다.

하지만 poolCleanup은 STW 상태에서 **명시적으로 참조를 끊는다.**

```go
p.local = nil      // 참조 끊기
p.localSize = 0
```

참조가 끊어진 후 Mark 단계가 시작되므로, 이전 local의 버퍼들은 도달 불가능한 객체가 되어 수거된다.

정리하면 다음과 같다.

```
sync.Pool 내용물이 사라지는 이유:
1. GC가 시작되면 STW 상태에서 poolCleanup 호출
2. poolCleanup이 p.local = nil로 참조를 끊음
3. Mark 단계에서 이 버퍼들은 도달 불가능
4. Sweep 단계에서 수거됨

일반 변수가 살아남는 이유:
1. 참조가 유지됨
2. Mark 단계에서 루트부터 도달 가능
3. 검은색으로 마킹되어 수거 안 됨
```

---

## GC 튜닝 옵션

### GOGC

힙이 이전 GC 후 라이브 힙 크기 대비 몇 % 증가했을 때 GC를 트리거할지 결정한다.

```
GOGC=100 (기본값): 힙이 2배가 되면 GC
GOGC=200: 힙이 3배가 되면 GC (GC 빈도 감소, 메모리 사용 증가)
GOGC=50: 힙이 1.5배가 되면 GC (GC 빈도 증가, 메모리 사용 감소)
GOGC=off: GC 비활성화
```

```go
import "runtime/debug"

// 런타임에 변경
debug.SetGCPercent(200)
```

### GOMEMLIMIT (Go 1.19+)

소프트 메모리 제한을 설정한다.

```bash
GOMEMLIMIT=512MiB ./myapp
```

컨테이너 환경에서 OOM 방지에 유용하다. 힙이 제한에 가까워지면 GC가 더 공격적으로 동작한다.

```go
debug.SetMemoryLimit(512 * 1024 * 1024)
```

주의: 제한을 너무 낮게 설정하면 GC가 과도하게 실행되어 CPU 사용량이 증가한다.

---

## GC Pacer

GC가 언제 실행될지 결정하는 알고리즘이다.

### 목표

- 힙 크기를 목표치 근처로 유지
- GC CPU 사용률을 25% 이하로 유지
- STW 시간 최소화

### 동작 원리

```
Trigger Ratio = (목표 힙 크기 - 현재 힙 크기) / 현재 힙 크기

GC 시작 조건:
- 힙 크기가 이전 GC 후 크기의 (1 + GOGC/100) 배에 도달
- 또는 2분 이상 GC가 없었을 때 (강제 GC)
```

### Mutator Assist

할당 속도가 마킹 속도를 초과하면, 할당을 요청한 고루틴이 마킹 작업을 일부 수행한다.

```go
// runtime/malloc.go의 mallocgc 내부 (개념적)
func mallocgc(size uintptr, ...) {
    // ...
    if gcBlackenEnabled {
        // 할당량에 비례해서 마킹 작업 수행
        gcAssistAlloc()
    }
    // ...
}
```

이 메커니즘으로 GC가 뒤처지는 것을 방지한다.

---

## Green Tea GC (Go 1.25 실험적)

2025년 10월 발표된 새로운 GC 알고리즘이다.

### 기존 GC의 문제

```
기존 Tricolor 마킹:
- 객체 단위로 그래프 순회
- 메모리 접근 패턴이 불규칙 (poor spatial locality)
- CPU 캐시 효율이 낮음
- 마킹 시간의 35% 이상이 메모리 대기
```

### Green Tea의 핵심 아이디어

```
기존: 객체 단위로 스캔
     Object A → Object B → Object C (메모리상 흩어져 있음)

Green Tea: 메모리 블록 단위로 스캔
     [Page 1: A, B, C, D] → 한 번에 스캔
     [Page 2: E, F, G] → 한 번에 스캔
```

같은 메모리 페이지에 있는 객체들을 모아서 처리한다. 캐시 효율이 올라간다.

### 성능 개선

```
벤치마크 결과:
- GC CPU 비용 10~40% 감소
- SIMD 가속 적용 시 추가 10% 감소
- 대부분의 워크로드에서 효과 있음
```

### 사용 방법

```bash
GOEXPERIMENT=greenteagc go build
```

Go 1.26에서 기본값으로 적용될 예정이다.

---

## 실제 GC 동작 관찰

### GODEBUG로 GC 로그 출력

```bash
GODEBUG=gctrace=1 ./myapp
```

출력 예시는 다음과 같다.

```
gc 1 @0.012s 2%: 0.026+0.67+0.021 ms clock, 0.10+0.26/0.47/0.15+0.085 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
```

```
gc 1:           1번째 GC
@0.012s:        프로그램 시작 후 12ms에 발생
2%:             이 GC가 전체 CPU의 2% 사용

0.026+0.67+0.021 ms clock:
  0.026ms:      STW (Mark 시작)
  0.67ms:       동시 마킹
  0.021ms:      STW (Mark 종료)

4->4->0 MB:
  4MB:          GC 시작 시 힙 크기
  4MB:          GC 종료 시 힙 크기
  0MB:          라이브 힙 크기

5 MB goal:      다음 GC 트리거 힙 크기
4 P:            사용 중인 P 개수
```

### runtime/trace로 상세 분석

```go
import "runtime/trace"

func main() {
    f, _ := os.Create("trace.out")
    trace.Start(f)
    defer trace.Stop()
    
    // 애플리케이션 코드
}
```

```bash
go tool trace trace.out
```

브라우저에서 GC 타임라인을 시각적으로 볼 수 있다.

---

## 정리

Go GC의 핵심 메커니즘을 요약하면 다음과 같다.

```
1. Tricolor Mark and Sweep
   - 흰색/회색/검은색으로 객체 분류
   - 루트부터 시작해서 도달 가능한 객체 마킹
   - 흰색으로 남은 객체 수거

2. Write Barrier
   - 동시성 GC에서 참조 변경 추적
   - Hybrid Write Barrier로 STW 최소화
   - 컴파일러가 자동 삽입

3. sync.Pool 특별 처리
   - GC 시작 시 STW 상태에서 poolCleanup 호출
   - 참조를 명시적으로 끊어서 수거되게 함
   - Victim Cache로 2단계 삭제

4. 튜닝 옵션
   - GOGC: GC 빈도 조절
   - GOMEMLIMIT: 메모리 상한선
   - GC Pacer가 자동으로 최적화
```

sync.Pool이 GC 때 비워지는 이유는 "참조가 없어서"가 아니라 "런타임이 STW 상태에서 의도적으로 참조를 끊기 때문"이다. 이런 내부 동작을 이해하면 Go의 메모리 관리 전략이 더 명확해진다.

---

## 참고 자료

- Go GC Guide: https://go.dev/doc/gc-guide
- Green Tea GC 발표: https://go.dev/blog/greenteagc
- Hybrid Write Barrier 제안: https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md
- sync/pool.go 소스 코드: https://go.dev/src/sync/pool.go
- runtime/mgc.go 소스 코드: https://go.dev/src/runtime/mgc.go
