Go로 개발하다 보면 슬라이스를 특정 기준으로 정렬해야 할 때가 많다. 특히 시간순 정렬은 트랜잭션 로그, 이벤트 기록 등에서 자주 쓰인다. 이 글에서는 `sort.Slice` 함수와 `time.Time`의 비교 메서드들을 정리한다.

## sort.Slice 기본 사용법

`sort.Slice`는 Go 1.8부터 추가된 함수로, 커스텀 정렬 로직을 간단하게 작성할 수 있게 해준다.

```go
func Slice(x any, less func(i, j int) bool)
```

첫 번째 인자로 정렬할 슬라이스를, 두 번째 인자로 비교 함수를 받는다. 비교 함수는 인덱스 i의 요소가 인덱스 j의 요소보다 앞에 와야 하면 true를 반환한다.

### 간단한 예제

```go
numbers := []int{5, 2, 8, 1, 9}

sort.Slice(numbers, func(i, j int) bool {
    return numbers[i] < numbers[j]  // 오름차순
})
// 결과: [1, 2, 5, 8, 9]
```

내림차순으로 바꾸려면 부등호 방향만 바꾸면 된다.

```go
sort.Slice(numbers, func(i, j int) bool {
    return numbers[i] > numbers[j]  // 내림차순
})
// 결과: [9, 8, 5, 2, 1]
```

## time.Time의 비교 메서드

`time.Time` 타입은 시간 비교를 위한 세 가지 메서드를 제공한다.

| 메서드 | 설명 | 반환값 |
|--------|------|--------|
| `t.Before(u)` | t가 u보다 이전인지 | bool |
| `t.After(u)` | t가 u보다 이후인지 | bool |
| `t.Equal(u)` | t와 u가 같은 시점인지 | bool |

### 사용 예시

```go
t1 := time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)
t2 := time.Date(2024, 6, 15, 0, 0, 0, 0, time.UTC)

t1.Before(t2)  // true  - t1이 t2보다 이전
t1.After(t2)   // false - t1이 t2보다 이후가 아님
t1.Equal(t2)   // false - 서로 다른 시점
```

참고로 `==` 연산자 대신 `Equal` 메서드를 쓰는 이유가 있다. `time.Time` 구조체는 타임존 정보를 포함하는데, `Equal`은 두 시간이 같은 순간을 나타내는지를 비교하고, `==`는 타임존까지 완전히 동일한지를 비교한다.

## 구조체 슬라이스를 시간순으로 정렬하기

실제로 많이 쓰는 패턴은 구조체 슬라이스를 시간 필드 기준으로 정렬하는 것이다.

```go
type Transaction struct {
    ID        string
    Amount    int
    Timestamp time.Time
}

txs := []Transaction{
    {"tx1", 100, time.Date(2024, 3, 15, 10, 0, 0, 0, time.UTC)},
    {"tx2", 200, time.Date(2024, 1, 10, 14, 30, 0, 0, time.UTC)},
    {"tx3", 150, time.Date(2024, 2, 20, 9, 0, 0, 0, time.UTC)},
}

// 오래된 순서대로 정렬 (오름차순)
sort.Slice(txs, func(i, j int) bool {
    return txs[i].Timestamp.Before(txs[j].Timestamp)
})
// 결과: tx2 -> tx3 -> tx1

// 최신 순서대로 정렬 (내림차순)
sort.Slice(txs, func(i, j int) bool {
    return txs[i].Timestamp.After(txs[j].Timestamp)
})
// 결과: tx1 -> tx3 -> tx2
```

## sort.SliceStable

정렬 안정성이 필요하다면 `sort.SliceStable`을 쓴다. 동일한 값을 가진 요소들의 상대적 순서가 유지된다.

```go
sort.SliceStable(txs, func(i, j int) bool {
    return txs[i].Timestamp.Before(txs[j].Timestamp)
})
```

같은 시간에 발생한 트랜잭션들이 원래 입력된 순서를 유지해야 할 때 유용하다.

## 다중 조건 정렬

여러 필드를 기준으로 정렬해야 할 때도 있다. 예를 들어 날짜가 같으면 금액순으로 정렬하는 경우다.

```go
sort.Slice(txs, func(i, j int) bool {
    if txs[i].Timestamp.Equal(txs[j].Timestamp) {
        return txs[i].Amount < txs[j].Amount
    }
    return txs[i].Timestamp.Before(txs[j].Timestamp)
})
```

## 정리

`sort.Slice`는 별도의 인터페이스 구현 없이 슬라이스를 정렬할 수 있어서 편하다. 시간 필드가 있는 구조체를 정렬할 때는 `Before`나 `After` 메서드를 활용하면 된다. 정렬 안정성이 필요하면 `sort.SliceStable`을 쓰면 되고, 복잡한 정렬 조건은 비교 함수 안에서 if문으로 처리할 수 있다.
