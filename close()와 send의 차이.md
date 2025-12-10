Go로 고루틴 종료 패턴을 공부하다 보면 이런 코드를 보게 된다.

```go
close(stopCh)
```

처음엔 이게 좀 헷갈렸다. `ch <- value`랑 뭐가 다른 건지. 둘 다 채널로 뭔가 보내는 거 아닌가?

결론부터 말하면, `close()`는 메시지를 보내는 게 아니다. 채널의 상태를 바꾸는 거다.

---

## send와 close의 차이

```go
ch := make(chan struct{})

// 방법 1: send
ch <- struct{}{}

// 방법 2: close
close(ch)
```

둘 다 `<-ch`로 대기 중인 고루틴을 깨운다는 점에서는 비슷해 보인다. 근데 동작 방식이 완전히 다르다.

### send는 큐에 값을 넣는 것

```go
ch <- struct{}{}
```

채널 내부에는 큐가 있다. send를 하면 이 큐에 값이 하나 들어간다. 수신자가 `<-ch`로 값을 꺼내가면 큐에서 없어진다.

수신자가 3명이면? 3번 보내야 한다.

```go
ch <- struct{}{}  // 고루틴 1이 받음
ch <- struct{}{}  // 고루틴 2가 받음
ch <- struct{}{}  // 고루틴 3이 받음
```

### close는 상태를 바꾸는 것

```go
close(ch)
```

이건 큐에 뭔가를 넣는 게 아니다. 채널 내부의 `closed` 플래그를 `true`로 바꾸는 거다.

채널 내부를 단순화하면 이런 느낌이다.

```go
type channel struct {
    closed  bool        // 닫혔는지 여부
    queue   []value     // 값들이 담긴 큐
    waiters []goroutine // 수신 대기 중인 고루틴들
}
```

`close(ch)`를 호출하면 `closed`가 `true`로 바뀌고, 대기 중이던 모든 고루틴이 깨어난다.

---

## 수신할 때 무슨 일이 일어나나

`<-ch`를 실행하면 내부적으로 이런 흐름이다. 의사코드로 보면:

```
func receive(ch) {
    if ch.closed {
        return 즉시  // 블로킹 안 함
    }
    
    if ch.queue가 비었으면 {
        대기...
    }
    
    return queue에서 꺼낸 값
}
```

send의 경우 큐에 값이 있어야 수신이 된다. 값 하나당 수신 한 번.

close의 경우 `closed` 플래그만 확인한다. `true`면 즉시 반환. 큐를 확인하지 않는다. 그래서 몇 번이든 수신할 수 있다.

---

## 실험해보기

닫힌 채널에서 100번 수신해보자.

```go
func main() {
    ch := make(chan struct{})
    close(ch)
    
    for i := 0; i < 100; i++ {
        <-ch
        fmt.Println(i, "받음")
    }
}
```

출력:
```
0 받음
1 받음
2 받음
...
99 받음
```

다 된다. 닫힌 채널에서 수신하면 무조건 즉시 반환되니까.

send로 같은 걸 하려면?

```go
func main() {
    ch := make(chan struct{})
    
    go func() {
        for i := 0; i < 100; i++ {
            ch <- struct{}{}  // 100번 보내야 함
        }
    }()
    
    for i := 0; i < 100; i++ {
        <-ch
        fmt.Println(i, "받음")
    }
}
```

100번 보내야 100번 받을 수 있다.

---

## 비유: 가게 문

이 차이를 가게에 비유하면 이해하기 쉽다.

**send는 물건을 놓는 것**

```
가게 앞에 물건을 놓는다.
손님이 가져가면 없어진다.
손님 3명이면 물건 3개가 필요하다.
```

**close는 폐업 팻말을 거는 것**

```
가게 문에 "폐업" 팻말을 건다.
손님이 몇 명이든 팻말을 볼 수 있다.
팻말은 안 없어진다. 계속 거기 있다.
```

close는 "이 채널 끝났어"라는 상태 표시다. 메시지를 보내는 게 아니라.

---

## 왜 종료 신호에 close를 쓰나

여러 고루틴에게 동시에 종료 신호를 보내야 할 때 유용하다.

```go
func main() {
    stopCh := make(chan struct{})
    
    // 고루틴 3개 시작
    for i := 0; i < 3; i++ {
        go func(id int) {
            <-stopCh
            fmt.Println(id, "종료")
        }(i)
    }
    
    time.Sleep(time.Second)
    close(stopCh)  // 한 번에 모두 종료!
    time.Sleep(time.Second)
}
```

출력:
```
0 종료
1 종료
2 종료
```

send로 했다면 3번 보내야 했다. 고루틴이 몇 개인지 알아야 하고, 그만큼 정확히 보내야 한다. 실수하기 쉽다.

close는 고루틴이 몇 개든 상관없다. 한 번 닫으면 전부 깨어난다.

---

## 주의할 점

닫힌 채널에 send하면 패닉이 난다.

```go
ch := make(chan struct{})
close(ch)
ch <- struct{}{}  // panic: send on closed channel
```

채널을 두 번 닫아도 패닉이다.

```go
ch := make(chan struct{})
close(ch)
close(ch)  // panic: close of closed channel
```

그래서 close는 보통 채널을 만든 쪽에서만 호출한다. 여러 곳에서 close를 호출하면 두 번 닫는 실수가 생길 수 있다.

---

## 닫힌 채널 감지하기

수신할 때 채널이 닫혔는지 확인할 수 있다.

```go
val, ok := <-ch

if ok {
    // 정상적으로 값을 받음
} else {
    // 채널이 닫힘
}
```

`ok`가 `false`면 채널이 닫힌 거다. `val`은 해당 타입의 zero value가 된다.

---

## 정리

`ch <- value`와 `close(ch)`는 완전히 다른 동작이다.

| | send (`ch <- value`) | close (`close(ch)`) |
|---|---|---|
| 동작 | 큐에 값 추가 | closed 플래그를 true로 |
| 수신 | 값 1개당 수신 1번 | 무한히 수신 가능 |
| 대상 | 수신자 1명 | 모든 수신자 |
| 이후 | 계속 send 가능 | 더 이상 send 불가 |

종료 신호를 보낼 때는 close를 쓴다. 고루틴이 몇 개든 한 번에 종료시킬 수 있고, 채널의 "상태"를 바꾸는 것이기 때문에 나중에 확인하는 고루틴도 종료 상태를 알 수 있다.

처음엔 헷갈리는데 "close는 폐업 팻말"이라고 생각하면 이해하기 쉽다.
