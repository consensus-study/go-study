`conn.Read(buf)`

이 한 줄이 실행될 때 내부에서 무슨 일이 일어나는가? Go의 net 패키지 소스 코드를 직접 까보면서 추적해본다.

---

## 1. 시작점: net.Conn 인터페이스

net 패키지를 열면 가장 먼저 `Conn` 인터페이스가 보인다.

```go
// net/net.go

type Conn interface {
    Read(b []byte) (n int, err error)
    Write(b []byte) (n int, err error)
    Close() error
    LocalAddr() Addr
    RemoteAddr() Addr
    SetDeadline(t time.Time) error
    SetReadDeadline(t time.Time) error
    SetWriteDeadline(t time.Time) error
}
```

주석을 보면 중요한 내용이 있다.

```go
// Multiple goroutines may invoke methods on a Conn simultaneously.
```

여러 고루틴이 동시에 하나의 Conn 메서드를 호출해도 된다는 거다. 이건 내부적으로 동기화가 되어 있다는 뜻이다. 나중에 이게 어떻게 구현되어 있는지 보자.

---

## 2. 실제 구현체: conn 구조체

인터페이스는 껍데기고, 실제 구현은 `conn` 구조체다.

```go
// net/net.go

type conn struct {
    fd *netFD
}
```

단 하나의 필드. `netFD` 포인터다. 이름에서 알 수 있듯이 Network File Descriptor다. Unix에서는 모든 게 파일이고, 소켓도 파일 디스크립터로 표현된다.

`conn`이 제대로 된 상태인지 체크하는 헬퍼 메서드도 있다.

```go
func (c *conn) ok() bool { 
    return c != nil && c.fd != nil 
}
```

nil 체크다. conn 자체가 nil이거나 내부 fd가 nil이면 사용할 수 없다.

---

## 3. Read 메서드 구현 분석

이제 핵심인 `Read` 메서드를 보자.

```go
// net/net.go

func (c *conn) Read(b []byte) (int, error) {
    if !c.ok() {
        return 0, syscall.EINVAL
    }
    n, err := c.fd.Read(b)
    if err != nil && err != io.EOF {
        err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return n, err
}
```

한 줄씩 뜯어보자.

**1단계: 유효성 검사**
```go
if !c.ok() {
    return 0, syscall.EINVAL
}
```

conn이 유효하지 않으면 `syscall.EINVAL`을 반환한다. EINVAL은 "Invalid argument"라는 표준 Unix 에러다. 여기서 이미 syscall 패키지가 등장한다. net 패키지가 OS 시스템 콜과 직접 연결되어 있다는 증거다.

**2단계: 실제 읽기**
```go
n, err := c.fd.Read(b)
```

여기서 `c.fd.Read(b)`를 호출한다. conn의 Read는 결국 netFD의 Read로 위임된다. 이 netFD.Read가 실제로 시스템 콜을 수행하는 부분이다.

**3단계: 에러 래핑**
```go
if err != nil && err != io.EOF {
    err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
}
```

에러가 발생하면 (EOF 제외) `OpError`로 래핑한다. 왜 EOF는 제외할까? EOF는 에러가 아니라 정상적인 스트림 종료 신호이기 때문이다. 상대방이 연결을 정상적으로 닫으면 EOF가 온다.

---

## 4. OpError: 네트워크 에러의 표준 구조

`OpError`를 자세히 보자. net 패키지의 모든 에러는 이 구조체로 래핑된다.

```go
// net/net.go

type OpError struct {
    // Op is the operation which caused the error, such as
    // "read" or "write".
    Op string

    // Net is the network type on which this error occurred,
    // such as "tcp" or "udp6".
    Net string

    // For operations involving a remote network connection, like
    // Dial, Read, or Write, Source is the corresponding local
    // network address.
    Source Addr

    // Addr is the network address for which this error occurred.
    // For local operations, like Listen or SetDeadline, Addr is
    // the address of the local endpoint being manipulated.
    // For operations involving a remote network connection, like
    // Dial, Read, or Write, Addr is the remote address of that
    // connection.
    Addr Addr

    // Err is the error that occurred during the operation.
    Err error
}
```

에러가 발생하면 어떤 작업(Op)에서, 어떤 네트워크 타입(Net)으로, 어떤 주소(Source, Addr)에서 문제가 생겼는지 다 담긴다.

Error 메서드 구현을 보면 이 정보들이 어떻게 조합되는지 알 수 있다.

```go
func (e *OpError) Error() string {
    if e == nil {
        return "<nil>"
    }
    s := e.Op
    if e.Net != "" {
        s += " " + e.Net
    }
    if e.Source != nil {
        s += " " + e.Source.String()
    }
    if e.Addr != nil {
        if e.Source != nil {
            s += "->"
        } else {
            s += " "
        }
        s += e.Addr.String()
    }
    s += ": " + e.Err.Error()
    return s
}
```

결과적으로 이런 에러 메시지가 만들어진다.

```
read tcp 192.168.1.100:54321->93.184.216.34:80: connection reset by peer
```

"read" + "tcp" + 로컬주소 + "->" + 원격주소 + ":" + 실제에러. 디버깅할 때 아주 유용하다.

---

## 5. Write 메서드: Read와의 대칭 구조

Write도 같은 패턴이다.

```go
// net/net.go

func (c *conn) Write(b []byte) (int, error) {
    if !c.ok() {
        return 0, syscall.EINVAL
    }
    n, err := c.fd.Write(b)
    if err != nil {
        err = &OpError{Op: "write", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return n, err
}
```

Read와 거의 동일하다. 차이점은 EOF 체크가 없다는 것. Write에서는 EOF가 의미가 없기 때문이다.

---

## 6. Close: 연결 종료의 내부

```go
// net/net.go

func (c *conn) Close() error {
    if !c.ok() {
        return syscall.EINVAL
    }
    err := c.fd.Close()
    if err != nil {
        err = &OpError{Op: "close", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return err
}
```

같은 패턴이다. netFD.Close()를 호출하고 에러를 래핑한다.

주석을 보면 중요한 내용이 있다.

```go
// Close closes the connection.
// Any blocked Read or Write operations will be unblocked and return errors.
```

Close를 호출하면 블로킹 중인 Read나 Write가 즉시 에러와 함께 반환된다. 이건 고루틴 정리할 때 중요한 특성이다. 고루틴이 Read에서 블로킹 중일 때, 다른 고루틴에서 Close를 호출하면 Read가 에러와 함께 리턴된다.

---

## 7. 주소 정보: LocalAddr와 RemoteAddr

```go
// net/net.go

func (c *conn) LocalAddr() Addr {
    if !c.ok() {
        return nil
    }
    return c.fd.laddr
}

func (c *conn) RemoteAddr() Addr {
    if !c.ok() {
        return nil
    }
    return c.fd.raddr
}
```

여기서 `laddr`과 `raddr`이 나온다. local address와 remote address다. 이 정보가 netFD에 저장되어 있다.

주석이 흥미롭다.

```go
// The Addr returned is shared by all invocations of LocalAddr, so
// do not modify it.
```

반환되는 Addr은 공유 객체다. 호출할 때마다 새로 만들지 않고 같은 걸 반환한다. 성능을 위한 선택인데, 이걸 수정하면 안 된다는 경고다.

---

## 8. 커널 버퍼 크기 조절: SetReadBuffer, SetWriteBuffer

여기서 커널 버퍼 얘기가 나온다.

```go
// net/net.go

// SetReadBuffer sets the size of the operating system's
// receive buffer associated with the connection.
func (c *conn) SetReadBuffer(bytes int) error {
    if !c.ok() {
        return syscall.EINVAL
    }
    if err := setReadBuffer(c.fd, bytes); err != nil {
        return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
    }
    return nil
}

// SetWriteBuffer sets the size of the operating system's
// transmit buffer associated with the connection.
func (c *conn) SetWriteBuffer(bytes int) error {
    if !c.ok() {
        return syscall.EINVAL
    }
    if err := setWriteBuffer(c.fd, bytes); err != nil {
        return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
    }
    return nil
}
```

주석을 보자. "operating system's receive buffer", "operating system's transmit buffer". 이건 OS 커널의 버퍼 크기를 조절하는 거다.

내부적으로 이건 `SO_RCVBUF`와 `SO_SNDBUF` 소켓 옵션을 설정하는 시스템 콜로 이어진다. 커널이 이 연결을 위해 얼마나 많은 메모리를 할당할지 결정한다.

이게 앞서 말한 커널 버퍼다.

```
[TCP 데이터 흐름]

상대방 -> 네트워크 -> NIC -> 커널 TCP 스택 -> 커널 수신 버퍼 -> read() -> 유저 공간
                                              ^^^^^^^^^^^^^^^^
                                              SetReadBuffer로 조절
```

---

## 9. Deadline 설정: 타임아웃 구현

```go
// net/net.go

func (c *conn) SetDeadline(t time.Time) error {
    if !c.ok() {
        return syscall.EINVAL
    }
    if err := c.fd.SetDeadline(t); err != nil {
        return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
    }
    return nil
}

func (c *conn) SetReadDeadline(t time.Time) error {
    if !c.ok() {
        return syscall.EINVAL
    }
    if err := c.fd.SetReadDeadline(t); err != nil {
        return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
    }
    return nil
}

func (c *conn) SetWriteDeadline(t time.Time) error {
    if !c.ok() {
        return syscall.EINVAL
    }
    if err := c.fd.SetWriteDeadline(t); err != nil {
        return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
    }
    return nil
}
```

Deadline 관련 주석이 아주 상세하다.

```go
// A deadline is an absolute time after which I/O operations
// fail instead of blocking. The deadline applies to all future
// and pending I/O, not just the immediately following call to
// Read or Write. After a deadline has been exceeded, the
// connection can be refreshed by setting a deadline in the future.
```

중요한 포인트들:

1. Deadline은 절대 시간이다. "5초 후"가 아니라 "2024년 1월 1일 12:00:05"
2. 설정하면 현재 블로킹 중인 I/O에도 적용된다
3. Deadline이 지나면 I/O가 실패한다
4. 새 Deadline을 설정하면 연결을 다시 쓸 수 있다

그리고 에러 처리에 대한 설명:

```go
// If the deadline is exceeded a call to Read or Write or to other
// I/O methods will return an error that wraps os.ErrDeadlineExceeded.
// This can be tested using errors.Is(err, os.ErrDeadlineExceeded).
```

Deadline 초과 에러는 `os.ErrDeadlineExceeded`를 래핑한다. `errors.Is`로 체크할 수 있다.

Idle timeout 구현 방법도 나와 있다.

```go
// An idle timeout can be implemented by repeatedly extending
// the deadline after successful Read or Write calls.
```

Read/Write 성공할 때마다 Deadline을 미래로 밀어주면 idle timeout이 된다.

---

## 10. Listener 인터페이스: 서버 소켓

서버를 만들 때 쓰는 Listener를 보자.

```go
// net/net.go

type Listener interface {
    // Accept waits for and returns the next connection to the listener.
    Accept() (Conn, error)

    // Close closes the listener.
    // Any blocked Accept operations will be unblocked and return errors.
    Close() error

    // Addr returns the listener's network address.
    Addr() Addr
}
```

Accept의 주석이 중요하다. "waits for and returns the next connection". 새 연결이 올 때까지 블로킹한다.

그리고 Close 주석: "Any blocked Accept operations will be unblocked and return errors." conn.Close()와 같은 특성이다. 다른 고루틴에서 Close를 호출하면 블로킹 중인 Accept가 에러와 함께 리턴된다.

---

## 11. 에러 타입들: 세밀한 에러 처리

net 패키지는 다양한 에러 타입을 정의한다.

```go
// net/net.go

// Various errors contained in OpError.
var (
    errNoSuitableAddress = errors.New("no suitable address found")
    errMissingAddress    = errors.New("missing address")
    errCanceled          = canceledError{}
    ErrWriteToConnected  = errors.New("use of WriteTo with pre-connected connection")
)
```

그리고 timeout 관련 에러:

```go
var errTimeout error = &timeoutError{}

type timeoutError struct{}

func (e *timeoutError) Error() string   { return "i/o timeout" }
func (e *timeoutError) Timeout() bool   { return true }
func (e *timeoutError) Temporary() bool { return true }

func (e *timeoutError) Is(err error) bool {
    return err == context.DeadlineExceeded
}
```

`timeoutError`는 `Error` 인터페이스를 구현한다.

```go
type Error interface {
    error
    Timeout() bool
    Temporary() bool
}
```

네트워크 에러는 `Timeout()`과 `Temporary()` 메서드를 가진다. 이걸로 에러 유형을 판단할 수 있다.

```go
if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
    // 타임아웃 처리
}
```

주석을 보면 Temporary는 deprecated다.

```go
// Deprecated: Temporary errors are not well-defined.
// Most "temporary" errors are timeouts, and the few exceptions are surprising.
// Do not use this method.
Temporary() bool
```

"temporary" 에러의 정의가 모호해서 쓰지 말라고 한다. 대부분의 temporary 에러는 그냥 타임아웃이다.

---

## 12. 연결 에러의 특수 처리

OpError의 Temporary 메서드를 보면 흥미로운 부분이 있다.

```go
func (e *OpError) Temporary() bool {
    // Treat ECONNRESET and ECONNABORTED as temporary errors when
    // they come from calling accept. See issue 6163.
    if e.Op == "accept" && isConnError(e.Err) {
        return true
    }

    if ne, ok := e.Err.(*os.SyscallError); ok {
        t, ok := ne.Err.(temporary)
        return ok && t.Temporary()
    }
    t, ok := e.Err.(temporary)
    return ok && t.Temporary()
}
```

Accept에서 ECONNRESET이나 ECONNABORTED가 발생하면 temporary로 취급한다. 왜? 클라이언트가 연결했다가 바로 끊는 경우다. 서버 입장에서는 이건 "재시도하면 될 수도 있는" 에러다.

---

## 13. context 취소와 매핑

```go
// net/net.go

type canceledError struct{}

func (canceledError) Error() string { return "operation was canceled" }

func (canceledError) Is(err error) bool { return err == context.Canceled }
```

```go
func mapErr(err error) error {
    switch err {
    case context.Canceled:
        return errCanceled
    case context.DeadlineExceeded:
        return errTimeout
    default:
        return err
    }
}
```

context 에러를 net 패키지의 에러로 매핑한다. `context.Canceled`는 `errCanceled`로, `context.DeadlineExceeded`는 `errTimeout`으로. 근데 `errors.Is`로 체크하면 원래 context 에러로도 매칭된다. `Is` 메서드 덕분이다.

---

## 14. Buffers: 벡터 I/O 지원

소스 코드에 `Buffers` 타입이 있다.

```go
// net/net.go

// Buffers contains zero or more runs of bytes to write.
//
// On certain machines, for certain types of connections, this is
// optimized into an OS-specific batch write operation (such as
// "writev").
type Buffers [][]byte

var (
    _ io.WriterTo = (*Buffers)(nil)
    _ io.Reader   = (*Buffers)(nil)
)
```

이건 여러 바이트 슬라이스를 한 번에 쓰는 기능이다. 내부적으로 `writev` 시스템 콜을 사용한다.

왜 필요할까? 예를 들어 HTTP 응답을 보낸다고 하자.

```go
// 일반적인 방식
conn.Write(headers)  // 시스템 콜 1
conn.Write(body)     // 시스템 콜 2

// Buffers 사용
bufs := net.Buffers{headers, body}
bufs.WriteTo(conn)   // 시스템 콜 1 (writev)
```

시스템 콜 횟수가 줄어든다.

WriteTo 구현을 보자.

```go
func (v *Buffers) WriteTo(w io.Writer) (n int64, err error) {
    if wv, ok := w.(buffersWriter); ok {
        return wv.writeBuffers(v)
    }
    for _, b := range *v {
        nb, err := w.Write(b)
        n += int64(nb)
        if err != nil {
            v.consume(n)
            return n, err
        }
    }
    v.consume(n)
    return n, nil
}
```

먼저 `buffersWriter` 인터페이스를 체크한다.

```go
type buffersWriter interface {
    writeBuffers(*Buffers) (int64, error)
}
```

Writer가 이 인터페이스를 구현하면 최적화된 경로를 탄다. TCPConn 같은 게 이걸 구현해서 writev를 사용한다. 구현하지 않으면 폴백으로 하나씩 Write한다.

consume 메서드도 보자.

```go
func (v *Buffers) consume(n int64) {
    for len(*v) > 0 {
        ln0 := int64(len((*v)[0]))
        if ln0 > n {
            (*v)[0] = (*v)[0][n:]
            return
        }
        n -= ln0
        (*v)[0] = nil
        *v = (*v)[1:]
    }
}
```

이미 쓴 바이트를 버퍼에서 제거한다. 부분 쓰기가 일어났을 때 남은 부분만 유지하기 위한 로직이다.

---

## 15. 스레드 제한: cgo lookup 보호

DNS lookup 관련 코드를 보면 스레드 제한이 있다.

```go
// net/net.go

// Limit the number of concurrent cgo-using goroutines, because
// each will block an entire operating system thread.

var threadLimit chan struct{}

var threadOnce sync.Once

func acquireThread(ctx context.Context) error {
    threadOnce.Do(func() {
        threadLimit = make(chan struct{}, concurrentThreadsLimit())
    })
    select {
    case threadLimit <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func releaseThread() {
    <-threadLimit
}
```

cgo를 사용하는 DNS lookup은 OS 스레드를 블로킹한다. 고루틴은 가볍지만 OS 스레드는 무겁다. 너무 많은 동시 lookup이 발생하면 시스템 스레드가 고갈될 수 있다.

그래서 채널로 세마포어를 구현해서 동시 lookup 수를 제한한다. 패키지 문서에도 나와 있다.

```go
// On all systems (except Plan 9), when the cgo resolver is being used
// this package applies a concurrent cgo lookup limit to prevent the system
// from running out of system threads. Currently, it is limited to 500 concurrent lookups.
```

500개 제한이다.

---

## 16. ErrClosed: 닫힌 연결 에러

```go
// net/net.go

// errClosed exists just so that the docs for ErrClosed don't mention
// the internal package poll.
var errClosed = poll.ErrNetClosing

// ErrClosed is the error returned by an I/O call on a network
// connection that has already been closed, or that is closed by
// another goroutine before the I/O is completed.
var ErrClosed error = errClosed
```

닫힌 연결에서 I/O를 하면 `ErrClosed`가 반환된다. 주석이 재밌다. 내부 패키지 poll을 문서에 노출시키기 싫어서 이렇게 래핑했다.

에러 체크는 이렇게 한다.

```go
if errors.Is(err, net.ErrClosed) {
    // 연결이 이미 닫혔다
}
```

---

## 17. ReadFrom 최적화: sendfile 지원

TCP 연결에서 파일을 전송할 때 최적화 경로가 있다.

```go
// net/net.go

type tcpConnWithoutReadFrom struct {
    noReadFrom
    *TCPConn
}

// Fallback implementation of io.ReaderFrom's ReadFrom, when sendfile isn't
// applicable.
func genericReadFrom(c *TCPConn, r io.Reader) (n int64, err error) {
    // Use wrapper to hide existing r.ReadFrom from io.Copy.
    return io.Copy(tcpConnWithoutReadFrom{TCPConn: c}, r)
}
```

TCPConn은 `io.ReaderFrom`을 구현한다. 소스가 파일이면 내부적으로 `sendfile` 시스템 콜을 사용할 수 있다. sendfile은 커널 공간에서 파일 데이터를 직접 소켓으로 보낸다. 유저 공간을 거치지 않아서 빠르다.

```
[일반 방식]
파일 -> read() -> 유저버퍼 -> write() -> 커널버퍼 -> 네트워크

[sendfile]
파일 -> sendfile() -> 네트워크
         (커널 내에서 직접 전송)
```

근데 sendfile을 못 쓰는 경우도 있다. 그때 genericReadFrom이 폴백으로 동작한다.

`noReadFrom` 구조체가 왜 필요한지 주석에 나와 있다.

```go
// noReadFrom can be embedded alongside another type to
// hide the ReadFrom method of that other type.
type noReadFrom struct{}

// ReadFrom hides another ReadFrom method.
// It should never be called.
func (noReadFrom) ReadFrom(io.Reader) (int64, error) {
    panic("can't happen")
}
```

`io.Copy`가 내부적으로 ReaderFrom을 체크한다. 무한 재귀를 방지하려고 ReadFrom을 숨기는 트릭이다.

WriteTo도 같은 패턴이다.

```go
type tcpConnWithoutWriteTo struct {
    noWriteTo
    *TCPConn
}

func genericWriteTo(c *TCPConn, w io.Writer) (n int64, err error) {
    return io.Copy(w, tcpConnWithoutWriteTo{TCPConn: c})
}
```

---

## 18. DNS Resolver 선택: Pure Go vs cgo

패키지 문서에 DNS resolver 관련 내용이 상세하게 나와 있다.

```go
// On Unix systems, the resolver has two options for resolving names.
// It can use a pure Go resolver that sends DNS requests directly to the servers
// listed in /etc/resolv.conf, or it can use a cgo-based resolver that calls C
// library routines such as getaddrinfo and getnameinfo.
```

두 가지 옵션이 있다:
1. Pure Go resolver: Go로 직접 DNS 프로토콜 구현
2. cgo resolver: C 라이브러리 (getaddrinfo) 호출

왜 두 개가 있을까?

```go
// On Unix the pure Go resolver is preferred over the cgo resolver, because a blocked DNS
// request consumes only a goroutine, while a blocked C call consumes an operating system thread.
```

Pure Go resolver가 선호된다. 블로킹 DNS 요청이 고루틴만 점유하지, OS 스레드를 점유하지 않기 때문이다. cgo를 쓰면 C 코드가 블로킹되는 동안 OS 스레드가 묶인다.

하지만 cgo를 써야 하는 경우도 있다.

```go
// When cgo is available, the cgo-based resolver is used instead under a variety of
// conditions: on systems that do not let programs make direct DNS requests (OS X),
// when the LOCALDOMAIN environment variable is present (even if empty),
// when the RES_OPTIONS or HOSTALIASES environment variable is non-empty,
// when /etc/resolv.conf or /etc/nsswitch.conf specify the use of features that the
// Go resolver does not implement.
```

macOS에서는 직접 DNS 요청을 막아서 cgo를 써야 한다. 특정 환경 변수가 설정되어 있거나, /etc/resolv.conf에 Go resolver가 지원하지 않는 기능이 있으면 cgo를 쓴다.

강제로 선택할 수도 있다.

```go
// export GODEBUG=netdns=go    # force pure Go resolver
// export GODEBUG=netdns=cgo   # force native resolver (cgo, win32)
```

---

## 19. listenerBacklog: 연결 대기열 크기

```go
var listenerBacklogCache struct {
    sync.Once
    val int
}

func listenerBacklog() int {
    listenerBacklogCache.Do(func() { listenerBacklogCache.val = maxListenerBacklog() })
    return listenerBacklogCache.val
}
```

TCP 서버의 listen backlog 크기를 캐싱한다. listen()의 두 번째 인자로 들어가는 값이다.

주석이 재밌다.

```go
// listenerBacklog should be an internal detail,
// but widely used packages access it using linkname.
// Notable members of the hall of shame include:
//   - github.com/database64128/tfo-go/v2
//   - github.com/metacubex/tfo-go
//   - github.com/sagernet/tfo-go
//
// Do not remove or change the type signature.
```

"hall of shame". 내부 함수인데 외부 패키지들이 linkname으로 접근해서 호환성 때문에 바꿀 수가 없다고 불평하고 있다.

---

## 20. 전체 흐름 정리: conn.Read() 한 줄의 여정

```go
n, err := conn.Read(buf)
```

이 한 줄이 실행되면:

```
1. conn.Read(buf) 호출
   |
   v
2. conn.ok() 체크
   - conn != nil && conn.fd != nil
   |
   v
3. conn.fd.Read(buf) 호출
   - netFD.Read()로 위임
   |
   v
4. netFD 내부
   - poll.FD.Read() 호출
   - internal/poll 패키지
   |
   v
5. poll.FD.Read()
   - epoll/kqueue에 등록
   - 데이터 올 때까지 고루틴 파킹
   - 데이터 오면 고루틴 깨움
   |
   v
6. 시스템 콜 (read)
   - 커널 버퍼 -> 유저 버퍼 복사
   |
   v
7. 결과 반환
   - 에러 있으면 OpError로 래핑
   |
   v
8. n, err 반환
```

---

## 21. bufio.Reader와의 조합

conn.Read()는 시스템 콜을 직접 호출한다. bufio.Reader를 쓰면 중간에 버퍼가 하나 더 생긴다.

```go
reader := bufio.NewReader(conn)
```

```
[버퍼 없이 - 직접 conn.Read()]

Read(4)   -> netFD.Read -> poll.FD.Read -> syscall -> 커널버퍼에서 4바이트
Read(100) -> netFD.Read -> poll.FD.Read -> syscall -> 커널버퍼에서 100바이트
Read(4)   -> netFD.Read -> poll.FD.Read -> syscall -> 커널버퍼에서 4바이트
= 시스템 콜 3번


[bufio.Reader 사용]

Read(4)
  -> bufio 버퍼 비어있음
  -> conn.Read(4096) 
     -> netFD.Read -> poll.FD.Read -> syscall
     -> 커널버퍼에서 최대 4096바이트
  -> bufio 버퍼에 저장
  -> bufio 버퍼에서 4바이트 반환

Read(100)
  -> bufio 버퍼에 데이터 있음
  -> 시스템 콜 없음!
  -> bufio 버퍼에서 100바이트 반환

Read(4)
  -> bufio 버퍼에 데이터 있음
  -> 시스템 콜 없음!
  -> bufio 버퍼에서 4바이트 반환

= 시스템 콜 1번
```

시스템 콜 비용:
- CPU 모드 전환 (유저 -> 커널 -> 유저)
- 레지스터 저장/복원
- 커널 코드 실행
- 데이터 복사

한 번에 수백 나노초에서 수 마이크로초. 작아 보이지만 초당 수십만 번이면 병목이 된다.

---

## 22. conn과 고루틴의 관계

소스 코드를 보면 conn은 그냥 데이터 구조다.

```go
type conn struct {
    fd *netFD
}
```

고루틴이 아니다. conn은 전화선이고, 고루틴은 전화 받는 사람이다.

```go
// 일반적인 서버 코드
listener, _ := net.Listen("tcp", ":8080")

for {
    conn, _ := listener.Accept()  // conn = 전화선
    go handleConnection(conn)     // 고루틴 = 전화 받는 사람
}

func handleConnection(conn net.Conn) {
    defer conn.Close()
    reader := bufio.NewReader(conn)
    for {
        msg, err := reader.ReadString('\n')
        if err != nil {
            return
        }
        // 처리
    }
}
```

보통 conn 하나당 고루틴 하나를 붙인다. 하지만 필수는 아니다. 하나의 고루틴이 여러 conn을 처리할 수도 있다 (멀티플렉싱). 다만 Go에서는 고루틴이 가볍기 때문에 conn당 고루틴 하나가 더 간단하고 Go스러운 방식이다.

---

## 핵심 정리

net 패키지 소스 코드에서 배운 것들:

```
구조:
  Conn 인터페이스 -> conn 구조체 -> netFD -> poll.FD -> 시스템 콜

conn 구조체:
  - fd *netFD 하나만 가지고 있음
  - Read/Write/Close 모두 netFD로 위임
  - 고루틴이 아님, 그냥 데이터 구조

에러 처리:
  - 모든 에러는 OpError로 래핑
  - Op, Net, Source, Addr, Err 정보 포함
  - EOF는 래핑하지 않음 (정상 종료 신호)
  - Timeout() 메서드로 타임아웃 판단
  - Temporary()는 deprecated

버퍼:
  - SetReadBuffer/SetWriteBuffer = 커널 버퍼 (SO_RCVBUF/SO_SNDBUF)
  - bufio.Reader = 유저 공간 버퍼, 시스템 콜 횟수 감소

동시성:
  - "Multiple goroutines may invoke methods on a Conn simultaneously"
  - Close() 호출하면 블로킹 중인 Read/Write가 즉시 리턴
  - cgo DNS lookup은 500개 동시 제한 (OS 스레드 보호)

최적화:
  - Buffers = writev 시스템 콜로 벡터 I/O
  - TCPConn.ReadFrom = sendfile로 zero-copy 전송
  - buffersWriter 인터페이스로 최적화 경로 분기

Deadline:
  - 절대 시간으로 설정
  - 현재 블로킹 중인 I/O에도 적용
  - 새 Deadline 설정으로 연결 재활성화 가능
```

소스 코드를 직접 읽으면 문서에 없는 것들이 보인다. 왜 이렇게 설계했는지, 어떤 트레이드오프가 있는지, 어디서 최적화가 일어나는지. 네트워크 프로그래밍할 때 문제가 생기면 이 구조를 알아야 어디를 봐야 하는지 감이 온다.
