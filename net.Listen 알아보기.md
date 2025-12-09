P2P 네트워크 코드를 작성하다 보면 `net.Listen("tcp", ":8080")`을 수도 없이 쓴다. 한 줄짜리 코드인데, OS 레벨에서는 꽤 많은 일이 일어난다.

블록체인 노드가 피어 연결을 받으려면 이 과정을 이해해야 한다. "왜 포트가 이미 사용 중이라고 뜨지?", "backlog가 뭐지?", "연결이 왜 거부되지?" 같은 문제를 만났을 때 OS 레벨 지식이 없으면 디버깅이 어렵다.

---

## 먼저 알아야 할 것: 파일 디스크립터 (fd)

Unix/Linux에서는 **"모든 것이 파일이다"**라는 철학이 있다.

```
일반 파일     → fd
디렉토리      → fd
소켓         → fd
파이프       → fd
터미널       → fd
```

파일 디스크립터(fd)는 그냥 **정수**다. 커널이 열린 자원을 관리하기 위해 붙여주는 번호표.

```
프로세스가 시작되면 기본으로 3개가 열려있음:

fd=0: stdin  (키보드 입력)
fd=1: stdout (화면 출력)
fd=2: stderr (에러 출력)

그 다음 열리는 건 fd=3, 4, 5, ...
```

### 커널 내부 구조

```
┌─────────────────────────────────────────────────────────────┐
│                      프로세스 (geth)                         │
├─────────────────────────────────────────────────────────────┤
│  fd 테이블 (프로세스마다 있음)                                │
│  ┌─────┬─────────────────────────────────────┐              │
│  │ fd  │ 가리키는 곳                          │              │
│  ├─────┼─────────────────────────────────────┤              │
│  │  0  │ ──→ stdin                           │              │
│  │  1  │ ──→ stdout                          │              │
│  │  2  │ ──→ stderr                          │              │
│  │  3  │ ──→ Listen 소켓 (0.0.0.0:30303)     │              │
│  │  4  │ ──→ Client A 연결 소켓              │              │
│  │  5  │ ──→ Client B 연결 소켓              │              │
│  │  6  │ ──→ leveldb 파일                    │              │
│  │ ... │                                     │              │
│  └─────┴─────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    커널 (공유 자원)                          │
├─────────────────────────────────────────────────────────────┤
│  열린 파일 테이블, 소켓 구조체, 버퍼 등                       │
└─────────────────────────────────────────────────────────────┘
```

### 왜 소켓도 fd인가

소켓도 결국 데이터를 읽고 쓰는 통로다. 파일처럼 취급하면:

```go
// 파일 읽기
file, _ := os.Open("data.txt")
file.Read(buf)
file.Write(data)
file.Close()

// 소켓도 똑같이
conn, _ := listener.Accept()
conn.Read(buf)
conn.Write(data)
conn.Close()
```

같은 인터페이스로 다룰 수 있어서 편하다. Go의 `io.Reader`, `io.Writer`가 이걸 추상화한 것.

### fd 확인하는 법

```bash
# 프로세스가 열고 있는 fd 목록
$ ls -la /proc/$(pgrep geth)/fd
lrwx------ 1 user user 64 ... 0 -> /dev/pts/0
lrwx------ 1 user user 64 ... 1 -> /dev/pts/0
lrwx------ 1 user user 64 ... 2 -> /dev/pts/0
lrwx------ 1 user user 64 ... 3 -> socket:[12345]
lrwx------ 1 user user 64 ... 4 -> socket:[12346]
...

# 열린 fd 개수
$ ls /proc/$(pgrep geth)/fd | wc -l
156
```

---

## net.Listen이 하는 일

```go
listener, err := net.Listen("tcp", ":30303")
```

이 한 줄이 내부에서 세 개의 시스템 콜을 순서대로 호출한다.

```
net.Listen("tcp", ":30303")
        │
        ▼
   ┌─────────┐
   │ socket()│  ← 소켓 생성
   └────┬────┘
        │
        ▼
   ┌─────────┐
   │  bind() │  ← 주소 바인딩
   └────┬────┘
        │
        ▼
   ┌─────────┐
   │ listen()│  ← 연결 대기 상태
   └─────────┘
```

하나씩 보자.

---

## 1단계: socket() - 소켓 생성

```c
// 커널 내부에서 일어나는 일 (의사 코드)
int fd = socket(AF_INET, SOCK_STREAM, 0);
```

| 인자 | 의미 |
|------|------|
| AF_INET | IPv4 사용 |
| SOCK_STREAM | TCP (연결 지향, 스트림) |
| 0 | 프로토콜 자동 선택 |

이 시점에서 커널이 하는 일:

```
1. 소켓 구조체 메모리 할당
2. 송신/수신 버퍼 초기화
3. 파일 디스크립터(fd) 할당해서 반환
```

아직 아무 주소에도 바인딩되지 않은 상태다. 그냥 "TCP 통신 준비된 빈 소켓"이다.

```
소켓 생성 직후 상태:

┌──────────────────────┐
│ Socket (fd=3)        │
├──────────────────────┤
│ Type: TCP            │
│ Local:  (없음)        │
│ Remote: (없음)        │
│ State: CLOSED        │
└──────────────────────┘
```

---

## 2단계: bind() - 주소 바인딩

```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(30303);        // 포트
addr.sin_addr.s_addr = INADDR_ANY;   // 모든 인터페이스

bind(fd, (struct sockaddr*)&addr, sizeof(addr));
```

커널이 하는 일:

```
1. 포트 테이블에서 30303이 사용 가능한지 확인
2. 사용 가능하면 이 소켓에 할당
3. 소켓 구조체에 로컬 주소 기록
```

```
bind() 후 상태:

┌──────────────────────┐
│ Socket (fd=3)        │
├──────────────────────┤
│ Type: TCP            │
│ Local:  0.0.0.0:30303│  ← 바인딩됨
│ Remote: (없음)        │
│ State: CLOSED        │
└──────────────────────┘

커널 포트 테이블:
┌────────┬────────┬─────────┐
│ Port   │ Proto  │ Socket  │
├────────┼────────┼─────────┤
│ 22     │ TCP    │ sshd    │
│ 80     │ TCP    │ nginx   │
│ 30303  │ TCP    │ fd=3    │  ← 등록됨
└────────┴────────┴─────────┘
```

### bind() 에러 케이스

```go
// 이미 사용 중인 포트
listener, err := net.Listen("tcp", ":22")
// Error: bind: address already in use

// 권한 없는 포트 (1024 미만)
listener, err := net.Listen("tcp", ":80")
// Error: bind: permission denied (root가 아니면)

// 잘못된 주소
listener, err := net.Listen("tcp", "999.999.999.999:8080")
// Error: invalid address
```

---

## 3단계: listen() - 연결 대기

```c
listen(fd, backlog);  // backlog는 대기열 크기
```

이 시점에서:

```
1. 소켓 상태가 LISTEN으로 변경
2. 연결 대기 큐(backlog queue) 생성
3. 이제 클라이언트 연결을 받을 준비 완료
```

```
listen() 후 상태:

┌──────────────────────┐
│ Socket (fd=3)        │
├──────────────────────┤
│ Type: TCP            │
│ Local:  0.0.0.0:30303│
│ Remote: (없음)        │
│ State: LISTEN        │  ← 변경됨
├──────────────────────┤
│ Backlog Queue        │
│ ┌─┬─┬─┬─┬─...─┬─┐   │
│ │ │ │ │ │     │ │   │  ← 연결 대기열
│ └─┴─┴─┴─┴─...─┴─┘   │
└──────────────────────┘
```

---

## Backlog Queue가 뭔가

클라이언트가 연결하면 바로 처리되는 게 아니다.

```
연결 과정:

Client                     Server (LISTEN 상태)
   │                              │
   │──── SYN ────────────────────>│
   │                              │  ← SYN Queue에 추가 (반쯤 연결)
   │<─── SYN+ACK ─────────────────│
   │                              │
   │──── ACK ────────────────────>│
   │                              │  ← Accept Queue로 이동 (완전 연결)
   │                              │
   │         (서버가 Accept() 호출할 때까지 여기서 대기)
```

두 개의 큐가 있다:

```
┌─────────────────────────────────────────┐
│              Kernel                      │
├─────────────────────────────────────────┤
│                                          │
│   SYN Queue (반쯤 연결된 것들)            │
│   ┌────┬────┬────┬────┐                 │
│   │ C1 │ C2 │ C3 │ ...│                 │
│   └────┴────┴────┴────┘                 │
│         │                                │
│         │ (3-way handshake 완료)         │
│         ▼                                │
│   Accept Queue (완전히 연결된 것들)        │
│   ┌────┬────┬────┬────┐                 │
│   │ C5 │ C6 │ C7 │ ...│  ← backlog 크기 │
│   └────┴────┴────┴────┘                 │
│         │                                │
│         │ (Accept() 호출)                │
│         ▼                                │
│   Application에서 처리                    │
│                                          │
└─────────────────────────────────────────┘
```

backlog가 가득 차면 새 연결이 거부된다. 블록체인 노드에서 피어가 갑자기 많이 연결하면 이 문제가 생길 수 있다.

### Go의 기본 backlog

```go
// Go 내부 (net/sock_posix.go)
// 보통 128 또는 시스템의 SOMAXCONN 값
```

리눅스에서 확인:

```bash
$ cat /proc/sys/net/core/somaxconn
4096  # 최신 리눅스는 보통 이 정도
```

---

## TCP 상태 전이

`netstat`으로 볼 수 있는 상태들:

```
서버 소켓:
CLOSED → LISTEN

클라이언트 연결 시:
LISTEN → (새 소켓 생성) → ESTABLISHED
```

```bash
$ netstat -tlnp | grep 30303
tcp  0  0  0.0.0.0:30303  0.0.0.0:*  LISTEN  12345/geth
```

---

## Go에서 전체 흐름

```go
func (s *TCPServer) Start() error {
    // socket() + bind() + listen() 한 번에
    listener, err := net.Listen("tcp", s.addr)
    if err != nil {
        return fmt.Errorf("TCP 서버 시작 실패: %w", err)
    }
    s.listener = listener
    return nil
}

func (s *TCPServer) AcceptLoop() {
    for {
        // accept() 시스템 콜
        // Accept Queue에서 연결 하나 꺼내옴
        conn, err := s.listener.Accept()
        if err != nil {
            continue
        }
        
        // 새 고루틴에서 처리
        go s.handleConnection(conn)
    }
}
```

Accept()가 하는 일:

```
1. Accept Queue에서 연결된 클라이언트 하나 꺼냄
2. 새 소켓 생성 (원래 listen 소켓과 별개)
3. 새 소켓의 fd 반환
```

### 왜 새 소켓을 만드는가

Listen 소켓 하나로 여러 클라이언트와 통신하면 데이터가 섞인다. 그래서 클라이언트마다 전용 소켓(fd)을 만들어주는 것.

```
비유:

Listen 소켓 = 식당 입구 안내 직원
  → "나는 입구만 지킴, 새 손님 오면 알려줌"

Accept로 생긴 소켓 = 각 테이블 담당 직원
  → "나는 이 테이블만 담당, 주문 받고 서빙함"

안내 직원은 계속 입구에서 손님 받음
서빙 직원은 각자 맡은 테이블과 대화
```

그림으로 보면:

```
                         Listen 소켓 (fd=3)
                         "나는 입구만 지킴"
                         Local: 0.0.0.0:30303
                         State: LISTEN
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
   Accept() 호출           Accept() 호출           Accept() 호출
        │                       │                       │
        ▼                       ▼                       ▼
   새 소켓 (fd=4)          새 소켓 (fd=5)          새 소켓 (fd=6)
   Remote: 1.2.3.4:5001    Remote: 5.6.7.8:5002    Remote: 9.10.11.12:5003
   Client A와 통신         Client B와 통신         Client C와 통신
```

코드로 보면:

```go
// Listen 소켓 - 한 번만 만들어짐, fd=3 할당
listener, _ := net.Listen("tcp", ":30303")

for {
    // Accept할 때마다 커널이 새 fd 할당
    conn1, _ := listener.Accept()  // fd=4 생성, Client A 전용
    conn2, _ := listener.Accept()  // fd=5 생성, Client B 전용
    conn3, _ := listener.Accept()  // fd=6 생성, Client C 전용
    
    // listener(fd=3)는 계속 살아있음
    // 새 연결 계속 받을 수 있음
}
```

### 각 소켓의 상태

```
Accept() 후 fd 테이블:

┌─────┬────────────────────────────────────────────┐
│ fd  │ 내용                                       │
├─────┼────────────────────────────────────────────┤
│  3  │ Listen Socket                              │
│     │   Local:  0.0.0.0:30303                    │
│     │   Remote: (없음)                            │
│     │   State:  LISTEN                           │
├─────┼────────────────────────────────────────────┤
│  4  │ Connection Socket (Client A)               │
│     │   Local:  0.0.0.0:30303                    │
│     │   Remote: 1.2.3.4:52431                    │
│     │   State:  ESTABLISHED                      │
├─────┼────────────────────────────────────────────┤
│  5  │ Connection Socket (Client B)               │
│     │   Local:  0.0.0.0:30303                    │
│     │   Remote: 5.6.7.8:48921                    │
│     │   State:  ESTABLISHED                      │
└─────┴────────────────────────────────────────────┘

같은 Local 포트(30303)인데 어떻게 구분?
→ (Local IP, Local Port, Remote IP, Remote Port) 4개 조합이 유니크하면 됨
```

---

## 블록체인 노드에서의 실제 사용

Geth를 보면:

```go
// go-ethereum/p2p/server.go
func (srv *Server) Start() error {
    // ...
    if srv.ListenAddr != "" {
        if err := srv.startListening(); err != nil {
            return err
        }
    }
    // ...
}

func (srv *Server) startListening() error {
    listener, err := net.Listen("tcp", srv.ListenAddr)
    if err != nil {
        return err
    }
    srv.listener = listener
    srv.loopWG.Add(1)
    go srv.listenLoop()  // Accept 루프
    return nil
}
```

기본 포트:

| 체인 | 포트 |
|------|------|
| Ethereum (Geth) | 30303 |
| Bitcoin | 8333 |
| Cosmos (CometBFT) | 26656 |

---

## 자주 만나는 문제들

### 1. Address already in use

```go
listener, err := net.Listen("tcp", ":30303")
// Error: bind: address already in use
```

원인:
- 이전 프로세스가 아직 포트 점유
- TIME_WAIT 상태 소켓 남아있음

해결:

```go
// SO_REUSEADDR 옵션 사용
lc := net.ListenConfig{
    Control: func(network, address string, c syscall.RawConn) error {
        return c.Control(func(fd uintptr) {
            syscall.SetsockoptInt(int(fd), syscall.SOL_SOCKET, syscall.SO_REUSEADDR, 1)
        })
    },
}
listener, err := lc.Listen(context.Background(), "tcp", ":30303")
```

### 2. Too many open files

```bash
Error: accept: too many open files
```

이제 이 에러가 왜 뜨는지 이해될 것이다.

```
연결 100개 = fd 100개 필요 (각 클라이언트마다 소켓)
연결 1000개 = fd 1000개 필요

근데 프로세스당 fd 개수 제한이 있음
```

```bash
# 현재 limit 확인
$ ulimit -n
1024  # 기본값이 보통 1024

# 블록체인 노드는 피어가 많으니 부족함!
```

fd가 부족하면 Accept()가 실패한다. 새 소켓을 만들 수 없으니까.

```
┌─────────────────────────────────────────┐
│  fd 테이블 (limit: 1024)                │
├─────┬───────────────────────────────────┤
│  0  │ stdin                             │
│  1  │ stdout                            │
│  2  │ stderr                            │
│  3  │ listen socket                     │
│  4  │ peer connection                   │
│  5  │ peer connection                   │
│ ... │ ...                               │
│ 1023│ peer connection                   │
├─────┴───────────────────────────────────┤
│  꽉 참! 더 이상 Accept 불가             │
└─────────────────────────────────────────┘
```

해결:

```bash
# 임시로 늘리기 (현재 세션만)
$ ulimit -n 65535

# 영구 설정 (/etc/security/limits.conf)
* soft nofile 65535
* hard nofile 65535

# systemd 서비스면 (/etc/systemd/system/geth.service)
[Service]
LimitNOFILE=65535
```

Geth 권장 설정:

```bash
# 최소 10000 이상 권장
# 피어 50개 + DB 파일들 + 여유분
```

### 3. Connection refused

클라이언트 입장에서:

```
서버가 LISTEN 상태가 아님
또는 backlog가 꽉 참
또는 방화벽이 막음
```

---

## 0.0.0.0 vs 127.0.0.1

```go
net.Listen("tcp", ":30303")       // 0.0.0.0:30303 - 모든 인터페이스
net.Listen("tcp", "0.0.0.0:30303") // 위와 동일
net.Listen("tcp", "127.0.0.1:30303") // 로컬만
```

```
0.0.0.0 (INADDR_ANY):
  - 외부에서 접근 가능
  - 모든 네트워크 인터페이스에서 연결 받음
  - 블록체인 노드는 보통 이거

127.0.0.1 (localhost):
  - 로컬에서만 접근 가능
  - 외부 연결 불가
  - RPC 서버는 보안상 이걸로 설정하기도 함
```

---

## Close()하면 무슨 일이 일어나나

```go
conn.Close()  // 연결 소켓 닫기
```

커널에서 일어나는 일:

```
1. TCP FIN 패킷 전송 (연결 종료 시작)
2. 소켓 버퍼의 남은 데이터 처리
3. fd 테이블에서 해당 fd 제거
4. fd 번호 재사용 가능해짐
```

```
Close() 전:                    Close() 후:

fd 테이블:                      fd 테이블:
┌─────┬─────────────┐          ┌─────┬─────────────┐
│  3  │ listen      │          │  3  │ listen      │
│  4  │ Client A    │          │  4  │ (비어있음)   │  ← 재사용 가능
│  5  │ Client B    │          │  5  │ Client B    │
└─────┴─────────────┘          └─────┴─────────────┘
```

주의점:

```go
// Close 안 하면 fd 누수!
func handleConnection(conn net.Conn) {
    // conn.Close() 안 하고 함수 끝나면
    // fd가 계속 점유됨
    // 시간이 지나면 "too many open files"
}

// 올바른 패턴
func handleConnection(conn net.Conn) {
    defer conn.Close()  // 함수 끝나면 반드시 닫힘
    
    // 연결 처리...
}
```

### TIME_WAIT 상태

Close해도 fd가 바로 사라지지 않는 경우가 있다.

```
연결 종료 후 상태 전이:

ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
                                            │
                                            └── 2분간 대기 (2MSL)
```

TIME_WAIT 동안은 같은 (IP, Port) 조합을 재사용 못 한다. 서버 재시작할 때 "address already in use" 뜨는 이유.

```bash
# TIME_WAIT 상태 소켓 확인
$ netstat -an | grep TIME_WAIT | wc -l
234
```

---

## tcp vs tcp4 vs tcp6

```go
net.Listen("tcp", ":30303")   // IPv4, IPv6 둘 다
net.Listen("tcp4", ":30303")  // IPv4만
net.Listen("tcp6", ":30303")  // IPv6만
```

듀얼 스택 시스템에서 "tcp"로 하면 IPv6 소켓이 생기고, IPv4도 같이 처리한다(IPv4-mapped IPv6). 근데 OS 설정에 따라 다를 수 있어서, 명시적으로 하려면 tcp4/tcp6을 쓴다.

---

## 정리

```go
listener, err := net.Listen("tcp", ":30303")
```

이 한 줄이:

1. **socket()**: TCP 소켓 생성, fd 할당 (예: fd=3)
2. **bind()**: 포트 30303에 바인딩
3. **listen()**: 연결 대기 상태로 전환, backlog 큐 생성

그 후:

4. **Accept()**: 새 연결마다 새 fd 할당 (fd=4, 5, 6, ...)
5. **Read/Write**: 각 fd로 데이터 송수신
6. **Close()**: fd 반환, 재사용 가능해짐

```
fd 관점에서 전체 흐름:

net.Listen()  → fd=3 (Listen용)
Accept()      → fd=4 (Client A)
Accept()      → fd=5 (Client B)
Close(fd=4)   → fd=4 반환
Accept()      → fd=4 재사용 (Client C)
```

블록체인 노드가 피어 연결을 받는 첫 단계다. fd 개념을 알면 "too many open files" 같은 에러도 바로 이해되고, 디버깅이 훨씬 쉬워진다.

---

## 참고

- Linux socket(7) man page
- Go net 패키지: https://pkg.go.dev/net
- TCP 상태 다이어그램: RFC 793
- Geth P2P 서버: https://github.com/ethereum/go-ethereum/tree/master/p2p
