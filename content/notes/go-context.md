+++
title = "Golang의 Context 패키지"
date = 2025-03-15
description = "context 패키지로 원활한 리소스 관리"
[taxonomies]
categories = ["BE"]
tags = ["Golang"]
[extra]
lang = "ko"
toc = true
+++

Golang에서는 시스템 리소스를 원활하게 관리할 수 있도록 돕는 Context 패키지를 제공한다. Context를 활용하면 요청이 취소되거나 timeout되었을 때 요청에 대한 모든 goroutine을 종료하고 시스템이 사용중인 리소스를 회수하도록 구성할 수 있게 된다.

서버에 들어오는 요청이 context를 생성하고 서버에서 나가는 호출은 context를 전달받으며 파생된 Context들을 전파하는 방식으로 실행된다. Context에는 데드라인, 취소 요청, 요청 범위 데이터 등이 포함된다.
[go standard package - context](https://pkg.go.dev/context)

## Context type

Context type은 다음과 같다.

```go
type Context interface {
	Done() <- chan struct {}
	Err() error
	Deadline() (deadline time.Time, ok bool)
	Value(key interface{}) interface{}
}
```

- `Done` : context에 취소나 timeout이 발생하면 닫히는 채널을 반환한다.
- `Err` : Done 채널이 닫힌 후에 context가 취소된 이유를 나타내는 에러를 반환한다.
- `Deadline` : context가 취소되는 시간을 반환한다.
- `Value`: key와 관련된 값을 반환한다. 요청 범위 데이터 전달에 사용된다.

## Derived Context

### `Background`

모든 `Context` 트리의 루트로 빈 Context를 반환한다. 들어오는 요청의 최상단 Context로 쓰인다.

```go
type emptyCtx struct {}

type backgroundCtx struct { emptyCtx }

func Background() Context {
	return backgroundCtx{}
}
```

### `WithCancel`

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
type CancelFunc func()
```

`WithCancel` 함수로 생성된 Context의 `Done` 채널은 `cancel` 함수가 호출되거나 `parent.Done` 채널이 닫힐 때 닫힌다.

### `WithDeadline` :

```go
func WithDeadline(parent Context, d Time.time) (Context, CancelFunc)
```

`WithDeadline` 함수로 생성된 Context의 `Done` 채널은 시간이 Deadline에 도달하거나, `cancel` 함수가 호출되거나, `parent.Done` 채널이 닫힐 때 닫힌다.
여기서 Deadline은 `parent`의 Deadline과 `d` 중 더 가까운 시간의 값이 사용된다.

### `WithTimeout` :

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

기본적으로 `WithDeadline`과 유사하나, 절대 시간 대신 현재를 기준으로 상대적인 시간을 입력받는다.

### `WithValue` :

```go
func WithValue(parent Context, key interface{}, val interface{}) Context
```

특정 key에 대해 지정된 값을 반환하는 부모 Context의 복사본을 반환한다.

### `WithCancelCause`, `WithDeadlineCause`, `WithTimeoutCause`

오류를 가져와 취소 원인으로 기록하는 `CancelCauseFunc`을 반환한다.

## Context 규칙

1. Context를 ctx라는 이름의 첫 번째 매개변수로 전달해야 한다. struct 안에 담아서는 안된다.

   ```go
   func DoSomething(ctx context.Context, arg Arg) error {
   	//...
   }
   ```

2. 사용될 Context가 사용될 지 불분명하거나 아직 사용할 수 없는 경우 nil context 대신 `context.TODO`를 사용하여 전달한다.

   ```go
   // context.TODO도 context.Background와 같이 빈 struct를 반환한다.
   type todoCtx struct { emptyCtx }

   func TODO() Context {
	 	return todoCtx{}
   }
	 ```

3. Context는 여러 고루틴에서 동시에 사용할 수 있다.

4. 프로세스 및 API를 통해 전송되는 요청 범위 데이터에만 Context를 사용한다.



## 어떻게 코드에 적용할 것인가

### `WithCancel`

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 부모 컨텍스트 생성
	parent := context.Background()
	// withCancel로 취소 가능한 컨텍스트 생성
	ctx, cancel := context.WithCancel(parent)

	// 고루틴에서 작업을 수행
	go func() {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("WithCancel: 작업 취소됨")
				return
			default:
				fmt.Println("WithCancel: 작업 수행 중...")
				time.Sleep(500 * time.Millisecond)
			}
		}
	}()

	// 2초 후에 컨텍스트 취소
	time.Sleep(2 * time.Second)
	fmt.Println("WithCancel: 컨텍스트 취소 시작")
	cancel()

	// 작업이 취소될 때까지 대기
	time.Sleep(1 * time.Second)
	fmt.Println("WithCancel: 프로그램 종료")
}
```

### `WithTimeout`

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 2초 후에 자동으로 취소되는 컨텍스트 생성
	ctx, _ := context.WithTimeout(context.Background(), 2 * time.Second)

	// 고루틴에서 작업 수행
	go func() {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("WithTimeout: 작업 취소됨:", ctx.Err())
				return
			default:
				fmt.Println("WithTimeout: 작업 수행 중...")
				time.Sleep(500 * time.Millisecond)
			}
		}
	}()

	// 컨텍스트가 취소될 때까지 대기
	time.Sleep(3 * time.Second)
	fmt.Println("WithTimeout: 프로그램 종료")
}
```


### `WithDeadline`

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 2초 후의 시간을 마감 시간으로 설정
	deadline := time.Now().Add(2 * time.Second)
	// 지정된 마감 시간에 취소되는 컨텍스트 생성
	ctx, _ := context.WithDeadline(context.Background(), deadline)

	// 고루틴에서 작업 수행
	go func() {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("WithDeadline: 작업 취소됨:", ctx.Err())
				return
			default:
				fmt.Println("WithDeadline: 작업 수행 중...")
				time.Sleep(500 * time.Millisecond)
			}
		}
	}()

	// 컨텍스트가 취소될 때까지 대기
	time.Sleep(3 * time.Second)
	fmt.Println("WithDeadline: 프로그램 종료")
}

```

### `WithValue`

```go
package main

import (
	"context"
  "fmt"
)

type favoriteContextKey string

func main() {
	// 컨텍스트에 값 저장
	ctx := context.WithValue(context.Background(), favoriteContextKey("language"), "Go")

	// 고루틴에서 컨텍스트 값 읽기
	go func(ctx context.Context) {
		if lang, ok := ctx.Value(favoriteContextKey("language")).(string); ok {
			fmt.Println("WithValue: 좋아하는 언어:", lang)
		} else {
			fmt.Println("WithValue: 언어 정보를 찾을 수 없음")
		}
	}(ctx)

	// 잠시 대기
	time.Sleep(1 * time.Second)
	fmt.Println("WithValue: 프로그램 종료")
}
```
