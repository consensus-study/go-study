Go로 API 서버를 만들거나 설정 파일을 읽을 때 JSON은 거의 필수다. Go 표준 라이브러리의 `encoding/json` 패키지가 이 역할을 한다. 마샬링은 Go 데이터를 JSON으로, 언마샬링은 JSON을 Go 데이터로 변환하는 과정이다.

## Marshal: Go 구조체를 JSON으로

`json.Marshal` 함수는 Go 값을 JSON 바이트 슬라이스로 변환한다.

```go
func Marshal(v any) ([]byte, error)
```

### 기본 사용법

```go
type User struct {
    Name  string
    Email string
    Age   int
}

user := User{
    Name:  "김철수",
    Email: "kim@example.com",
    Age:   28,
}

data, err := json.Marshal(user)
if err != nil {
    log.Fatal(err)
}

fmt.Println(string(data))
// {"Name":"김철수","Email":"kim@example.com","Age":28}
```

반환값이 `[]byte`라서 출력하려면 `string()`으로 변환해야 한다.

### 에러 처리를 생략하는 경우

가끔 이런 코드를 볼 수 있다.

```go
payload, _ := json.Marshal(msg)
```

언더스코어로 에러를 무시하는 건데, 구조체가 단순하고 마샬링 실패할 가능성이 없을 때 쓴다. 다만 프로덕션 코드에서는 웬만하면 에러 처리를 해주는 게 좋다. 채널이나 함수 같은 마샬링 불가능한 타입이 들어가면 에러가 난다.

## 구조체 태그로 필드명 제어하기

Go 구조체 필드는 대문자로 시작해야 export 되는데, JSON에서는 보통 소문자 camelCase를 쓴다. 구조체 태그로 이걸 맞출 수 있다.

```go
type User struct {
    Name      string `json:"name"`
    Email     string `json:"email"`
    Age       int    `json:"age"`
    CreatedAt string `json:"created_at"`
}

user := User{Name: "김철수", Email: "kim@example.com", Age: 28}
data, _ := json.Marshal(user)

fmt.Println(string(data))
// {"name":"김철수","email":"kim@example.com","age":28,"created_at":""}
```

### 자주 쓰는 태그 옵션

| 태그 | 설명 |
|------|------|
| `json:"name"` | JSON 키 이름을 name으로 |
| `json:"name,omitempty"` | 값이 비어있으면 필드 생략 |
| `json:"-"` | JSON 변환에서 완전히 제외 |

```go
type User struct {
    Name     string `json:"name"`
    Email    string `json:"email,omitempty"`
    Password string `json:"-"`
    Age      int    `json:"age,omitempty"`
}

user := User{Name: "김철수", Password: "secret123"}
data, _ := json.Marshal(user)

fmt.Println(string(data))
// {"name":"김철수"}
// Email은 빈 문자열이라 생략, Password는 - 태그로 제외, Age는 0이라 생략
```

`omitempty`는 제로값(빈 문자열, 0, nil, false 등)일 때 필드를 아예 빼버린다. API 응답에서 null 필드 안 보내고 싶을 때 유용하다.

## Unmarshal: JSON을 Go 구조체로

`json.Unmarshal`은 JSON 바이트 슬라이스를 Go 값으로 변환한다. 두 번째 인자로 결과를 담을 포인터를 넘겨야 한다.

```go
func Unmarshal(data []byte, v any) error
```

### 기본 사용법

```go
jsonStr := `{"name":"이영희","email":"lee@example.com","age":25}`

var user User
err := json.Unmarshal([]byte(jsonStr), &user)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("%+v\n", user)
// {Name:이영희 Email:lee@example.com Age:25}
```

포인터를 넘기는 이유는 Unmarshal이 그 주소에 직접 값을 써야 하기 때문이다.

### JSON에 없는 필드는 제로값으로

JSON에 일부 필드가 없으면 해당 필드는 Go의 제로값으로 남는다.

```go
jsonStr := `{"name":"박민수"}`

var user User
json.Unmarshal([]byte(jsonStr), &user)

fmt.Println(user.Age)  // 0
fmt.Println(user.Email) // ""
```

### 구조체에 없는 필드는 무시

반대로 JSON에는 있지만 구조체에 정의 안 된 필드는 그냥 무시된다.

```go
jsonStr := `{"name":"박민수","nickname":"민수르","age":30}`

type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

var user User
json.Unmarshal([]byte(jsonStr), &user)
// nickname 필드는 무시됨
```

## 동적 JSON 다루기: map과 interface{}

구조를 미리 모르는 JSON은 map으로 받을 수 있다.

```go
jsonStr := `{"name":"김철수","age":28,"active":true}`

var result map[string]interface{}
json.Unmarshal([]byte(jsonStr), &result)

fmt.Println(result["name"])  // 김철수
fmt.Println(result["age"])   // 28 (float64 타입으로 들어옴)
```

주의할 점은 숫자가 `float64`로 파싱된다는 거다. `result["age"].(int)` 하면 패닉 나고, `result["age"].(float64)` 해야 한다.

```go
age := int(result["age"].(float64))
```

## 중첩 구조체

JSON이 중첩되어 있으면 구조체도 중첩하면 된다.

```go
type Address struct {
    City   string `json:"city"`
    Street string `json:"street"`
}

type User struct {
    Name    string  `json:"name"`
    Address Address `json:"address"`
}

jsonStr := `{
    "name": "김철수",
    "address": {
        "city": "서울",
        "street": "강남대로 123"
    }
}`

var user User
json.Unmarshal([]byte(jsonStr), &user)

fmt.Println(user.Address.City) // 서울
```

## 슬라이스와 배열

JSON 배열은 Go 슬라이스로 매핑된다.

```go
type User struct {
    Name string   `json:"name"`
    Tags []string `json:"tags"`
}

jsonStr := `{"name":"김철수","tags":["developer","golang","backend"]}`

var user User
json.Unmarshal([]byte(jsonStr), &user)

fmt.Println(user.Tags[0]) // developer
```

JSON 배열 자체를 파싱할 수도 있다.

```go
jsonStr := `[{"name":"김철수"},{"name":"이영희"}]`

var users []User
json.Unmarshal([]byte(jsonStr), &users)

fmt.Println(len(users)) // 2
```

## MarshalIndent: 보기 좋게 출력

디버깅할 때 한 줄로 나오면 읽기 힘들다. `json.MarshalIndent`로 들여쓰기된 JSON을 만들 수 있다.

```go
data, _ := json.MarshalIndent(user, "", "  ")
fmt.Println(string(data))
```

결과:
```json
{
  "name": "김철수",
  "email": "kim@example.com",
  "age": 28
}
```

두 번째 인자는 각 줄 앞에 붙는 prefix, 세 번째는 들여쓰기 문자다. 보통 `"", "  "` 조합을 많이 쓴다.

## Encoder와 Decoder: 스트림 처리

파일이나 HTTP 응답처럼 io.Reader/io.Writer를 다룰 때는 Encoder/Decoder가 더 편하다.

```go
// HTTP 응답에 JSON 쓰기
func handler(w http.ResponseWriter, r *http.Request) {
    user := User{Name: "김철수", Age: 28}
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

// HTTP 요청 바디에서 JSON 읽기
func handler(w http.ResponseWriter, r *http.Request) {
    var user User
    json.NewDecoder(r.Body).Decode(&user)
}
```

`Marshal` + `Write` 조합보다 깔끔하고, 중간에 바이트 슬라이스를 안 만들어서 메모리도 절약된다.

## 흔한 실수들

### 필드가 unexported라서 안 됨

```go
type User struct {
    name string  // 소문자라서 export 안 됨
    age  int
}

user := User{name: "김철수", age: 28}
data, _ := json.Marshal(user)
fmt.Println(string(data)) // {}
```

필드 이름이 소문자면 json 패키지에서 접근을 못 한다. 반드시 대문자로 시작해야 함.

### 포인터 안 넘겨서 안 됨

```go
var user User
json.Unmarshal(data, user)  // 이러면 안 됨
json.Unmarshal(data, &user) // 이렇게 포인터로
```

### 숫자가 float64로 들어옴

```go
var m map[string]interface{}
json.Unmarshal([]byte(`{"count":42}`), &m)

count := m["count"].(int) // 패닉!
count := int(m["count"].(float64)) // 이렇게
```

## 정리

`json.Marshal`은 Go 값을 JSON 바이트로, `json.Unmarshal`은 그 반대다. 구조체 태그로 필드명을 제어하고, `omitempty`로 빈 값을 생략할 수 있다. HTTP 핸들러에서는 Encoder/Decoder가 더 편하고, 동적 JSON은 `map[string]interface{}`로 받으면 된다. 필드는 반드시 대문자로 시작해야 한다는 것만 기억하면 대부분의 케이스는 커버된다.
