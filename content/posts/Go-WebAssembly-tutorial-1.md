---
title: "Go-WebAssembly-入门（一）"
date: 2019-02-02T00:00:00+08:00
---

有关 WebAssembly 的介绍可以参考 [几张图让你看懂 WebAssembly](https://www.jianshu.com/p/bff8aa23fe4d)
简单来说 WebAssembly 就是将其他语言 C/Go/Rust 等语言编译成 wasm 可执行二进制文件，浏览器来执行 wasm。wasm 相比 JS,拥有体积更小，执行更快，因为最终编译成二进制文件，所以一些安全策略代码也更适合 wasm。
经过尝试 C 和 Go 分别编写 WebAssembly，相较而言我认为 Go 无论从语言层面还是工具链，用起来都更加方便一些。
本文使用原生 go build，生成的 wasm 文件大约在`1.4M`左右，在生产环境中这个体积是很大的，优化 go 的 wasm 体积可以使用 tinygo 来 build，同样的代码使用 tinygo 构建之后约为`22K`，甚至比 C 语言构建 wasm 的体积还要小（C 语言 build 后约为`44K`，不同版本不同环境可能略有差异）。参考[https://tinygo.org/](https://tinygo.org/)

本文介绍 Go WebAssembly 入门，前提已经安装 Go 1.11 及以上版本。
系列文章 [Go WebAssembly 入门（二）](https://www.jianshu.com/p/31f81255c367)

## Getting Started

编辑 main.go

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, Go WebAssembly!")
}
```

把 main.go build 成 WebAssembly(简写为 wasm)二进制文件

```
GOOS=js GOARCH=wasm go build -o lib.wasm main.go
```

把 JavaScript 依赖拷贝到当前路径

```
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" .
```

创建一个 index.html 文件，并引入 wasm_exec.js 文件，调用刚才 build 的 lib.wasm

```
<html>
	<head>
		<meta charset="utf-8">
		<script src="wasm_exec.js"></script>
		<script>
			const go = new Go();
			WebAssembly.instantiateStreaming(fetch("lib.wasm"), go.importObject).then((result) => {
				go.run(result.instance);
			});
		</script>
	</head>
	<body></body>
</html>
```

创建 server.go 监听 8080 端口，serve 当前路径

```
package main

import (
  "flag"
  "log"
  "net/http"
)

var (
  listen = flag.String("listen", ":8080", "listen address")
  dir    = flag.String("dir", ".", "directory to serve")
)

func main() {
  flag.Parse()
  log.Printf("listening on %q...", *listen)
  err := http.ListenAndServe(*listen, http.FileServer(http.Dir(*dir)))
  log.Fatalln(err)
}
```

启动服务

```
go run server.go
```

在浏览器访问 localhost:8080,打开浏览器 console，就可以看到输出`Hello, Go WebAssembly!`。

系列文章 [Go WebAssembly 入门（二）](https://www.jianshu.com/p/31f81255c367)

## reference

- https://github.com/golang/go/wiki/WebAssembly
