# 标准库启动Web服务

Go 语言内置了 `net/http` 库，封装了HTTP网络编程的基础的接口。

[01-standrd/base1/main.go](/web-frame/01-standard/base1/main.go)

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", indexHandler)
	http.HandleFunc("/hello", helloHandler)
	log.Fatal(http.ListenAndServe(":9999", nil))
}

// handler echoes r.URL.Path
func indexHandler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
}

// handler echoes r.URL.Header
func helloHandler(w http.ResponseWriter, req *http.Request) {
	for k, v := range req.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
}
```

用 curl 这个工具测试一下，将会得到如下的结果。

```js
$ curl http://localhost:9999/
URL.Path = "/"
$ curl http://localhost:9999/hello
Header["Accept"] = ["*/*"]
Header["User-Agent"] = ["curl/7.54.0"]
```

main 函数的最后一行，是用来启动 Web 服务的，第一个参数是地址，`:9999` 表示在 9999 端口监听。而第二个参数则代表处理所有的 HTTP 请求的实例，nil 代表使用标准库中的实例处理。第二个参数，则是我们基于 net/http 标准库实现 Web 框架的入口。

# 实现 http.Handler 接口

```go
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```

通过查看 net/http 的源码可以发现，Handler 是一个接口，需要实现方法 ServeHTTP ，只要传入任何实现了 ServerHTTP 接口的实例，所有的HTTP请求，就都交给了该实例处理。

[01-standard/base2/main.go](/web-frame/01-standard/base2/main.go)

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

// Engine is the uni handler for all requests
type Engine struct{}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/":
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	case "/hello":
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	default:
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}

func main() {
	engine := new(Engine)
	log.Fatal(http.ListenAndServe(":9999", engine))
}
```

- 定义了一个空的结构体 Engine，实现了方法 ServeHTTP。这个方法有2个参数，第二个参数是 Request ，该对象包含了该HTTP请求的所有的信息，比如请求地址、Header 和 Body 等信息；第一个参数是 ResponseWriter ，利用 ResponseWriter 可以构造针对该请求的响应。

- 在 main 函数中，给 ListenAndServe 方法的第二个参数传入了刚才创建的 engine 实例。至此，我们走出了实现Web框架的第一步，即，将所有的HTTP请求转向了我们自己的处理逻辑。在实现 Engine 之前，我们调用 http.HandleFunc 实现了路由和 Handler 的映射，也就是只能针对具体的路由写处理逻辑。比如 /hello 。但是在实现 Engine 之后，我们拦截了所有的 HTTP 请求，拥有了统一的控制入口。在这里我们可以自由定义路由映射的规则，也可以统一添加一些处理逻辑，例如日志、异常处理等。

代码的运行结果与之前的是一致的。

# 框架雏形

```tree
spirit/
  |--spirit.go
  |--go.mod
main.go
go.mod
```

## go.mod

[01-standard/base3/go.mod](/web-frame/01-standard/base3/go.mod)

```go
module example

go 1.19

require spirit v0.0.0

replace spirit => ./spirit
```

## main.go

[01-standard/base3/main.go](/web-frame/01-standard/base3/main.go)

```go
package main

import (
	"fmt"
	"net/http"

	"spirit"
)

func main() {
	r := spirit.New()
	r.GET("/", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})

	r.GET("/hello", func(w http.ResponseWriter, req *http.Request) {
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	})

	r.Run(":9999")
}
```

spirit 框架的设计以及 API 均参考了 gin。使用 New() 创建 gee 的实例，使用 GET()方法添加路由，最后使用 Run() 启动 Web 服务。这里的路由，只是静态路由，不支持 /hello/:name 这样的动态路由，动态路由我们将在下一次实现。

## spirit.go

[01-standard/base3/spirit/spirit.go](/web-frame/01-standard/spirit/spirit.go)

```go
package spirit

import (
	"fmt"
	"net/http"
)

// HandlerFunc defines the request handler used by gee
type HandlerFunc func(http.ResponseWriter, *http.Request)

// Engine implement the interface of ServeHTTP
type Engine struct {
	router map[string]HandlerFunc
}

// New is the constructor of gee.Engine
func New() *Engine {
	return &Engine{router: make(map[string]HandlerFunc)}
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	key := method + "-" + pattern
	engine.router[key] = handler
}

// GET defines the method to add GET request
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
func (engine *Engine) POST(pattern string, handler HandlerFunc) {
	engine.addRoute("POST", pattern, handler)
}

// Run defines the method to start a http server
func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	key := req.Method + "-" + req.URL.Path
	if handler, ok := engine.router[key]; ok {
		handler(w, req)
	} else {
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}
```

- 首先定义了类型 HandlerFunc，这是提供给框架用户的，用来定义路由映射的处理方法。我们在 Engine 中，添加了一张路由映射表 router，key 由请求方法和静态路由地址构成，例如 GET-/、GET-/hello、POST-/hello，这样针对相同的路由，如果请求方法不同,可以映射不同的处理方法 (Handler)，value 是用户映射的处理方法。

- 当用户调用 (*Engine).GET() 方法时，会将路由和处理方法注册到映射表 router 中，(*Engine).Run()方法，是 ListenAndServe 的包装。

- Engine实现的 ServeHTTP 方法的作用就是，解析请求的路径，查找路由映射表，如果查到，就执行注册的处理方法。如果查不到，就返回 404 NOT FOUND 。

执行 `go run main.go`，再用 curl 工具访问，结果与最开始的一致。

```js
$ curl http://localhost:9999/
URL.Path = "/"
$ curl http://localhost:9999/hello
Header["Accept"] = ["*/*"]
Header["User-Agent"] = ["curl/7.54.0"]
curl http://localhost:9999/world
404 NOT FOUND: /world
```

至此，整个Gee框架的原型已经出来了。实现了路由映射表，提供了用户注册静态路由的方法，包装了启动服务的函数。当然，到目前为止，我们还没有实现比 net/http 标准库更强大的能力，不用担心，很快就可以将动态路由、中间件等功能添加上去了。

