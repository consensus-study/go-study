Go 코드 보다 보면 함수 첫 번째 인자로 `ctx context.Context`가 붙어있는 걸 자주 본다. 처음엔 그냥 관례인가 싶은데, 이게 없으면 타임아웃 처리나 요청 취소가 굉장히 귀찮아진다.

## 왜 필요한가

HTTP 요청을 처리하는 상황을 생각해보자. 클라이언트가 요청을 보냈는데, 서버에서 DB 조회하고 외부 API 호출하고 이것저것 하는 동안 클라이언트가 연결을 끊어버렸다. 서버는 어떻게 해야 할까?

context 없이는 이걸 알 방법이 마땅치 않다. 그냥 끝까지 작업하고 응답 보내려다가 실패하는 수밖에. context는 이런 취소 신호를 전파하는 역할을 한다.

## context.WithTimeout 기본 형태

```go
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()
```

이 한 줄이 하는 일은 간단하다. 부모 context를 받아서, 5초 제한이 걸린 새 context를 만든다. 5초가 지나면 이 context는 자동으로 취소된다.

```go
func fetchData(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/data", nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err  // 5초 넘으면 여기서 context deadline exceeded 에러
    }
    defer resp.Body.Close()
    
    // ...
    return nil
}
```

## cancel 함수는 왜 defer로 호출하나

`WithTimeout`이 반환하는 cancel 함수는 반드시 호출해야 한다. 안 그러면 타이머가 만료될 때까지 리소스가 해제되지 않는다.

```go
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()  // 함수 끝나면 무조건 정리
```

작업이 5초 전에 끝나도 cancel을 호출해야 타이머 리소스가 바로 해제된다. defer로 걸어두면 함수가 어떻게 끝나든 호출이 보장되니까 이 패턴이 굳어진 거다.

## context 종류들

### context.Background()

최상위 context다. 취소되지 않고, 값도 없고, 데드라인도 없다. main 함수나 초기화 코드에서 시작점으로 쓴다.

```go
func main() {
    ctx := context.Background()
    doSomething(ctx)
}
```

### context.TODO()

아직 어떤 context를 써야 할지 모를 때 임시로 쓴다. 기능상으로는 Background와 같은데, 나중에 제대로 된 context로 바꿔야 한다는 표시 역할이다.

```go
// 나중에 적절한 context로 교체 예정
result := someFunction(context.TODO())
```

### context.WithCancel

타임아웃 없이 수동으로 취소할 수 있는 context를 만든다.

```go
ctx, cancel := context.WithCancel(parentCtx)

go func() {
    // 어떤 조건이 되면 취소
    if shouldStop {
        cancel()
    }
}()
```

### context.WithDeadline

특정 시각에 취소되는 context다. WithTimeout이 "지금부터 N초 후"라면, WithDeadline은 "몇 시 몇 분에" 취소다.

```go
deadline := time.Now().Add(30 * time.Second)
ctx, cancel := context.WithDeadline(parentCtx, deadline)
defer cancel()
```

사실 `WithTimeout(ctx, 5*time.Second)`은 내부적으로 `WithDeadline(ctx, time.Now().Add(5*time.Second))`과 같다.

## context가 취소됐는지 확인하기

### ctx.Done() 채널

context가 취소되면 Done() 채널이 닫힌다. select문으로 취소 여부를 확인할 수 있다.

```go
func doWork(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // 왜 취소됐는지 에러 반환
        default:
            // 작업 계속
        }
    }
}
```

### ctx.Err()

취소 원인을 알려준다.

| 반환값 | 의미 |
|--------|------|
| `nil` | 아직 취소 안 됨 |
| `context.Canceled` | cancel() 호출로 취소됨 |
| `context.DeadlineExceeded` | 타임아웃으로 취소됨 |

```go
if ctx.Err() == context.DeadlineExceeded {
    log.Println("타임아웃 발생")
}
```

## 실제 사용 패턴

### DB 쿼리에 타임아웃 걸기

```go
func getUser(ctx context.Context, id int) (*User, error) {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    
    row := db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = ?", id)
    
    var user User
    if err := row.Scan(&user.ID, &user.Name); err != nil {
        return nil, err
    }
    return &user, nil
}
```

### 여러 고루틴에 취소 전파

```go
func process(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()
    
    errCh := make(chan error, 2)
    
    go func() {
        errCh <- fetchFromAPI(ctx)
    }()
    
    go func() {
        errCh <- queryDatabase(ctx)
    }()
    
    // 둘 중 하나라도 에러나면 다른 것도 취소됨
    for i := 0; i < 2; i++ {
        if err := <-errCh; err != nil {
            return err
        }
    }
    return nil
}
```

ctx가 취소되면 이 ctx를 쓰는 모든 고루틴이 취소 신호를 받는다.

### HTTP 서버에서

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // r.Context()는 클라이언트 연결이 끊기면 취소됨
    ctx := r.Context()
    
    // 추가로 5초 타임아웃
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    result, err := slowOperation(ctx)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            http.Error(w, "timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(result)
}
```

## context에 값 넣기

context.WithValue로 요청 범위 값을 전달할 수 있다.

```go
type contextKey string

const userIDKey contextKey = "userID"

// 값 넣기
ctx = context.WithValue(ctx, userIDKey, "user-123")

// 값 꺼내기
userID, ok := ctx.Value(userIDKey).(string)
if !ok {
    // 값이 없거나 타입이 다름
}
```

주의할 점이 있다. 키 타입을 string으로 쓰면 다른 패키지와 충돌할 수 있어서, 위처럼 별도 타입을 정의하는 게 관례다.

그리고 WithValue는 설정값이나 의존성 주입용으로 쓰면 안 된다. 요청 ID, 인증 정보 같은 요청 범위 메타데이터에만 쓰는 게 맞다.

## 흔한 실수

### cancel 안 부르기

```go
// 이러면 리소스 누수
ctx, _ := context.WithTimeout(parentCtx, 5*time.Second)
```

반환값 무시하지 말고 defer로 호출하자.

### nil context 넘기기

```go
func doSomething(ctx context.Context) {
    // ctx가 nil이면 패닉 날 수 있음
}

doSomething(nil)  // 하지 마라
```

뭘 넘겨야 할지 모르겠으면 context.TODO() 넘겨라.

### 구조체에 context 저장하기

```go
type Service struct {
    ctx context.Context  // 이러지 마라
}
```

context는 요청 범위 값이라서 구조체 필드로 저장하면 안 된다. 함수 첫 번째 인자로 넘기는 게 맞다.

## 정리

context는 Go에서 취소 신호와 타임아웃을 전파하는 표준 방법이다. `WithTimeout`으로 시간 제한을 걸고, cancel은 defer로 반드시 호출하고, Done() 채널로 취소 여부를 확인한다. HTTP 핸들러에서는 `r.Context()`가 클라이언트 연결 상태를 반영하니까 이걸 부모로 쓰면 된다.

처음엔 번거로워 보이는데, 타임아웃 없이 외부 API 호출했다가 서버 전체가 먹통 되는 경험 한 번 하고 나면 왜 필요한지 뼈저리게 느끼게 된다.
