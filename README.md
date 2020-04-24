# noctx

![](https://github.com/sonatard/noctx/workflows/.github/workflows/ci.yml/badge.svg)

`noctx` finds sending http request without context.Context.

You have to pass a context.Context in order to cancel an http request or get trace information.
If you are providing a library, you have to abide this rule or the users of the library will get in trouble.
But many of the ways to send an HTTP request do not allow you to pass a context.Context.

## Install

```sh
$ go get -u github.com/sonatard/noctx/cmd/noctx
```

## Usage

```sh
$ go vet -vettool=`which noctx` main.go
./main.go:6:11: net/http.Get must not be called
```

## Detection rules
- Executing following functions
  - net/http.Get
  - net/http.Head
  - net/http.Post
  - net/http.PostForm
  - (*net/http.Client).Get
  - (*net/http.Client).Head
  - (*net/http.Client).Post
  - (*net/http.Client).PostForm
- http.Request returned by NewRequest function and passes it to other function.

## How to fix
- Send a request using the (*http.Client).Do(*http.Request) method.
- In Go 1.13 and later, use the http.NewRequestWithContext function instead of using the http.NewRequest function.
- In Go 1.12 and earlier, (http.Request).WithContext(ctx) is executed after the http.NewRequest function is executed.

(http.Request).WithContext(ctx) has a disadvantage of performance because it returns a copy of http.Request. Use the http.NewRequestWithContext function if you only support Go1.13 or later.

## Smaple code

```go
package main

import (
	"context"
	"net/http"
)

func main() {
	const url = "http://example.com"
	http.Get(url) // want `net/http\.Get must not be called`
	http.Head(url)          // want `net/http\.Head must not be called`
	http.Post(url, "", nil) // want `net/http\.Post must not be called`
	http.PostForm(url, nil) // want `net/http\.PostForm must not be called`

	cli := &http.Client{}
	cli.Get(url) // want `\(\*net/http\.Client\)\.Get must not be called`
	cli.Head(url)          // want `\(\*net/http\.Client\)\.Head must not be called`
	cli.Post(url, "", nil) // want `\(\*net/http\.Client\)\.Post must not be called`
	cli.PostForm(url, nil) // want `\(\*net/http\.Client\)\.PostForm must not be called`

	req, _ := http.NewRequest(http.MethodPost, url, nil) // want `Must not use http.NewRequest. Use http.NewRequestWithContext or http.NewRequest with \(\*Request\)\.WithContext`
	cli.Do(req)

	ctx := context.Background()
	req2, _ := http.NewRequestWithContext(ctx, http.MethodPost, url, nil) // OK
	cli.Do(req2)

	req3, _ := http.NewRequest(http.MethodPost, url, nil) // OK
	req3 = req3.WithContext(ctx)
	cli.Do(req3)

	f2 := func(req *http.Request, ctx context.Context) *http.Request {
		return req
	}
	req4, _ := http.NewRequest(http.MethodPost, url, nil) // want `Must not use http.NewRequest. Use http.NewRequestWithContext or http.NewRequest with \(\*Request\)\.WithContext`
	req4 = f2(req4, ctx)
	cli.Do(req4)
}
```

## Reference
- [net/http - NewRequest](https://golang.org/pkg/net/http/#NewRequest)
- [net/http - NewRequestWithContext](https://golang.org/pkg/net/http/#NewRequestWithContext)
- [net/http - Request.WithCOntext](https://golang.org/pkg/net/http/#Request.WithContext)

