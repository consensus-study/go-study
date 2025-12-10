Go로 동시성 프로그래밍을 하다 보면 이런 코드를 자주 마주친다.

```go
stopCh := make(chan struct{})
```

처음 봤을 때 솔직히 좀 당황했다. `struct{}`가 뭔데? 빈 구조체를 왜 채널로 만들지? 그냥 `chan bool` 쓰면 안 되나?

오늘은 이 패턴이 뭔지, 왜 쓰는지 정리해보려고 한다.

---

## 문제 상황: 고루틴을 어떻게 종료시키지?

백그라운드에서 뭔가를 계속 돌리는 고루틴이 있다고 해보자.

```go
func worker() {
    for {
        // 뭔가 열심히 일하는 중...
        doWork()
        time.Sleep(time.Second)
    }
}

func main() {
    go worker()
    
    time.Sleep(5 * time.Second)
    // worker를 종료시키고 싶은데... 어떻게?
}
```

`for` 무한 루프 안에서 돌고 있는 고루틴을 밖에서 멈추게 할 방법이 없다. Go에는 고루틴을 강제로 죽이는 API가 없다. 고루틴 스스로 종료하게 만들어야 한다.

---

## 해결책: 채널로 신호 보내기

채널을 하나 만들어서 "이제 그만해"라는 신호를 보내면 된다.

```go
func worker(stopCh chan struct{}) {
    for {
        select {
        case <-stopCh:
            // 신호 받음, 종료!
            fmt.Println("종료 신호 받음, 끝낸다")
            return
        default:
            doWork()
            time.Sleep(time.Second)
        }
    }
}

func main() {
    stopCh := make(chan struct{})
    
    go worker(stopCh)
    
    time.Sleep(5 * time.Second)
    
    close(stopCh)  // 종료 신호!
    
    time.Sleep(time.Second)  // 정리할 시간 좀 주고
}
```

`close(stopCh)`를 호출하면 `<-stopCh`에서 대기하던 모든 고루틴이 깨어난다. 채널이 닫히면 수신 연산이 즉시 반환되기 때문이다.

---

## 근데 왜 struct{}인가?

여기서 핵심 질문. 왜 하필 `chan struct{}`일까?

`chan bool`이나 `chan int`도 되긴 한다.

```go
stopCh := make(chan bool)
close(stopCh)  // 이것도 동작함
```

근데 `struct{}`를 쓰는 이유가 있다.

### 1. 메모리를 안 먹는다

```go
fmt.Println(unsafe.Sizeof(struct{}{}))  // 0
fmt.Println(unsafe.Sizeof(true))        // 1
fmt.Println(unsafe.Sizeof(0))           // 8
```

`struct{}`는 크기가 0바이트다. 빈 구조체니까 담을 게 없어서 메모리를 안 쓴다.

물론 종료 채널 하나에서 이게 큰 차이를 만들진 않는다. 근데 관례적으로 이렇게 쓴다.

### 2. 의도가 명확하다

```go
stopCh := make(chan struct{})  // "신호만 보낼 거야, 데이터는 없어"
stopCh := make(chan bool)      // "true/false 값을 보낼 건가?"
```

`struct{}`를 보면 "아 이건 데이터 전달이 아니라 신호용이구나"라는 게 바로 보인다. 코드의 의도를 드러내는 셈이다.

`chan bool`을 쓰면 `true`를 보내야 하나 `false`를 보내야 하나 고민하게 된다. 사실 그 값은 아무 의미 없는데도.

### 3. Go 커뮤니티의 관례다

표준 라이브러리, 유명한 오픈소스 프로젝트들이 전부 이 패턴을 쓴다. `context` 패키지 내부 구현을 보면 `done chan struct{}`가 있다.

남들이 다 이렇게 쓰니까 따라 쓰는 게 좋다. 코드 읽기가 편해진다.

---

## 실전 패턴: 종료 + 대기

실제로는 종료 신호만으로 부족한 경우가 많다. 고루틴이 진짜 끝났는지 확인하고 싶을 때가 있다.

```go
type Worker struct {
    stopCh chan struct{}
    wg     sync.WaitGroup
}

func (w *Worker) Start() {
    w.stopCh = make(chan struct{})
    w.wg.Add(1)
    go w.run()
}

func (w *Worker) run() {
    defer w.wg.Done()  // 끝나면 "나 끝났어" 알림
    
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-w.stopCh:
            fmt.Println("종료 중...")
            return
        case <-ticker.C:
            fmt.Println("일하는 중...")
        }
    }
}

func (w *Worker) Stop() {
    close(w.stopCh)  // 1. 종료 신호
    w.wg.Wait()      // 2. 진짜 끝날 때까지 대기
    fmt.Println("완전히 종료됨")
}
```

이 조합을 많이 쓴다.

- `stopCh`: "그만해" 명령
- `WaitGroup`: "진짜 끝났나" 확인

`close(stopCh)`만 하고 바로 다음 코드로 넘어가면, 고루틴이 아직 정리 작업 중일 수 있다. DB 연결 닫기, 파일 플러시 같은 게 안 끝났을 수도 있다는 거다. `wg.Wait()`으로 확실히 기다려줘야 안전하다.

---

## select와 함께 쓰기

`select`문 안에서 여러 채널을 동시에 기다릴 수 있다. 이게 핵심이다.

```go
for {
    select {
    case <-stopCh:
        // 종료 신호
        return
    case msg := <-msgCh:
        // 메시지 처리
        handleMessage(msg)
    case <-ticker.C:
        // 주기적 작업
        doPeriodicWork()
    }
}
```

`select`는 여러 case 중 하나가 준비되면 그걸 실행한다. 아무것도 준비 안 되면 대기한다.

`stopCh`가 닫히면 `case <-stopCh`가 바로 선택되고 `return`으로 고루틴이 종료된다. 다른 작업 중이더라도 다음 `select` 루프에서 종료 신호를 받을 수 있다.

---

## 주의할 점

### close는 한 번만

```go
close(stopCh)
close(stopCh)  // panic: close of closed channel
```

이미 닫힌 채널을 또 닫으면 패닉이 난다. 보통 `sync.Once`를 쓰거나 상태 플래그로 관리한다.

```go
type Worker struct {
    stopCh   chan struct{}
    stopOnce sync.Once
}

func (w *Worker) Stop() {
    w.stopOnce.Do(func() {
        close(w.stopCh)
    })
}
```

### 보내기 vs 닫기

```go
// 보내기 - 하나의 수신자만 받음
stopCh <- struct{}{}

// 닫기 - 모든 수신자가 받음
close(stopCh)
```

종료 신호는 보통 `close`를 쓴다. 여러 고루틴이 같은 채널을 보고 있을 수 있으니까, 한 번에 다 깨워야 한다.

---

## 정리

`chan struct{}`는 Go에서 고루틴 종료 신호를 보내는 표준 패턴이다.

- `struct{}`는 크기가 0이고 "신호만 보낸다"는 의도가 명확함
- `close(stopCh)`로 신호를 보내면 대기 중인 모든 고루틴이 깨어남
- `sync.WaitGroup`과 함께 써서 "신호 + 대기" 패턴으로 안전하게 종료

처음엔 이상해 보이는데 익숙해지면 꽤 깔끔한 패턴이다. Go 코드 읽다 보면 계속 나오니까 알아두면 좋다.
