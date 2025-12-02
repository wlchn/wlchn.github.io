---
title: "Go-WebAssembly-入门（二）"
date: 2019-02-02T01:00:00+08:00
---

系列文章 [Go WebAssembly 入门（一）](https://www.jianshu.com/p/b3ef3be94fa7)

## Getting Started

编写 main.go

```go
package main

import (
  "strconv"
  "syscall/js"
)
// 传入value1, value2, result三个元素的id，将value1+value2结果赋给result元素
func add(ids []js.Value) {
  // 根据id获取输入值
  value1 := js.Global().Get("document").Call("getElementById", ids[0].String()).Get("value").String()
  value2 := js.Global().Get("document").Call("getElementById", ids[1].String()).Get("value").String()

  int1, _ := strconv.Atoi(value1)
  int2, _ := strconv.Atoi(value2)
  // 将相加结果set给result元素
  js.Global().Get("document").Call("getElementById", ids[2].String()).Set("value", int1+int2)
}

// 添加监听事件
func registerCallbacks() {
  js.Global().Set("add", js.NewCallback(add))
}

func main() {
  c := make(chan struct{}, 0)
  println("Go WebAssembly Initialized!")
  registerCallbacks()

  <-c
}

```

将 main.go 编译成 lib.wasm

```
GOOS=js GOARCH=wasm go build -o lib.wasm main.go
```

在 index.html 中调用 lib.wasm

```
<html>
  <head>
    <meta charset="utf-8">
    <script src="wasm_exec.js"></script>
    <script>
      if (!WebAssembly.instantiateStreaming) { // polyfill
        WebAssembly.instantiateStreaming = async (resp, importObject) => {
          const source = await (await resp).arrayBuffer();
          return await WebAssembly.instantiate(source, importObject);
        };
      }

      const go = new Go();
      let mod, inst;
      WebAssembly.instantiateStreaming(fetch("lib.wasm"), go.importObject).then(async (result) => {
        mod = result.module;
        inst = result.instance;
        await go.run(inst)
      });
    </script>
  </head>
  <body>
    <input type="text" id="value1"/>
    <input type="text" id="value2"/>
    <button type="button" id="add" onClick="add('value1', 'value2', 'result');">add</button>
    <input type="text" id="result"/>
  </body>
</html>
```

打开 server，在浏览器打开即可调用 WebAssembly 二进制文件执行。

```
go run server.go
```

## 示例代码 GitHub

- https://github.com/wlchn/go-webassembly

## reference

- https://tutorialedge.net/golang/go-webassembly-tutorial/
