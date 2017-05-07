---
title: Custom labstack/echo Context
date: 2017-05-07 21:54:59
desc: '如何在 Golang echo 中使用自定義的 Context + Google JSON Style Guide 來統一回傳的格式'
type: post
tags:
  - APIs
	- Golang
	- echo
---

[labstack/echo](https://github.com/labstack/echo) 一個輕量、高效的 `Golang` 網頁框架，讓使用者非常容易的建立 RESTful API 伺服器

```go
package main

import (
	"net/http"

	"github.com/labstack/echo"
)

func main() {
	e := echo.New()
	e.GET("/", func(c echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})
	e.Logger.Fatal(e.Start(":1323"))
}
```

```bash
# run echo API server
$ go run main.go
⇛ http server started on [::]:1323

# get request
$ curl http://localhost:1323
Hello, World!
```

<!--more-->

`echo` 的 API 非常容易的擴充，一個好的 RESTful API 應該有定義嚴謹的 Response 格式。雖然 `echo` 的回傳函式已經有提供 `JSON(code int, i interface{}) error` 的方式可以讓我們回傳 JSON 的格式。

```go
// 使用 JSON(code int, i interface{}) error 回傳 JSON 資料

e.GET("/json", func(c echo.Context) error {
  return c.JSON(http.StatusOK, map[string]interface{}{
    "message": "Hello, World",
  })
})
```

當 API 的數量慢慢的成長，遇到修改回傳格式時就會照成維護的成本大大增加，這時候 `ehco` 也提供了 `Context` 擴充的方式，讓我們透過加載客製的 `middleware` 方法來擴充 API 回傳的函式，再搭配依照 [Google JSON Style Guide](https://google.github.io/styleguide/jsoncstyleguide.xml) 的規範統一回傳的格式


```go
// API call with success response
// embed context.SuccessData in your response struct
e.GET("/json2", func(c echo.Context) error {
  type Res struct {
    SuccessData
    Name string
  }

  res := &Res{}
  res.Name = "Cage"

  return c.(Ctx).Res(http.StatusOK).Data(res).Do()
})
```

```json
{
  "apiVersion": "v1",
  "data": {
    "code": 0,
    "Name": "Cage"
  }
}
```

```go
// embed context.SuccessData in your response struct
// put array in items property
e.GET("/json3", func(c echo.Context) error {
  type Res struct {
    SuccessData
  }

  res := &Res{}
  res.Items = []interface{}{"Cage", "Sunrain"}

  return c.(Ctx).Res(http.StatusOK).Data(res).Do()
})
```

```json
{
  "apiVersion": "v1",
  "data": {
    "code": 0,
    "items": [
      "Cage",
      "Sunrain"
    ]
  }
}
```

```go
// API call fail with custom error code and error message
return c.(Ctx).Res(http.StatusBadRequest).Error("message here").Code(4001).Do()
```

```json
{
  "apiVersion": "v1",
  "error": {
    "code": 4001,
    "message": "message here",
  }
}
```

```go
// API call fail with custom error code and error detail message
erros := make([]interface{}, 1)
erros[0] = "333"
return c.(Ctx).Res(http.StatusBadRequest).Error("message here").Code(4001).Errors(erros).Do()
```

```json
{
  "apiVersion": "v1",
  "error": {
    "code": 4001,
    "message": "message here",
    "errors": [
      "333"
    ]
  }
}
```

## How to custom context

__main.go__

```go
e.Use(func(next echo.HandlerFunc) echo.HandlerFunc {
  return func(c echo.Context) error {
    return next(Ctx{c})
  }
})
```

__context/context.go__

```go
type Ctx struct {
	echo.Context
}

func (c Ctx) Res(httpStatus int) *jsonCall {
	rs := &jsonCall{
		c:          echo.Context(c),
		httpStatus: httpStatus,
	}
	return rs
}

...
```

使用自己的 `Context` 最核心的部份是定義一個 `Struct` 並將 `echo` 中的 `echo.Context` 嵌入其中，這時候使用 `Golang` 特有的方式來對這一個 `Struct` 進行 `func` 的擴充，再完善自定欺的 `Context` 後即是需在透過 `middleware` 的方式掛載到 `echo` 中, 這樣一來在所有的 API 路中由就可以透過型別轉換的方式來操作自己義定的方法。

## Github

Repo [Custom labstack/echo Context](https://github.com/cage1016/Custom-echo-Context)


```bash
# clone repo
$ git clone git@github.com:cage1016/Custom-echo-Context.git

# install go-packages
$ glide i

# run echo API server
$ go run main.go
⇛ http server started on [::]:1323

# api test
$ curl http://localhost:1323/json
{"message":"Hello, World"}

$ curl http://localhost:1323/json2
{"apiVersion":"v1","data":{"code":0,"Name":"Cage"}}

$ curl http://localhost:1323/json3
{"apiVersion":"v1","data":{"code":0,"items":["Cage","Sunrain"]}}

$ curl http://localhost:1323/json4
{"apiVersion":"v1","error":{"code":400001,"message":"Bad request"}}

$ curl http://localhost:1323/json5
{"apiVersion":"v1","error":{"code":400001,"message":"Bad request","errors":["333"]}}

# test
$ go test context/context_test.go
ok  	command-line-arguments	0.010s
```
