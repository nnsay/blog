---
title: go wasm
date: 2023-09-09 18:57:42
tags:
  - Code
  - 研究
excerpt: 使用go编译wasm并在浏览器中调用和使用
---

# 1. [WebAssembly](https://webassembly.org/)

WebAseembly(wasm)已经出现多年了, 对于 wasm 的印象就两个: 二进制, 性能好. 最近在调研[网站数据安全防护](https://nnsay.cn/2023/09/05/frontend-security/#more), 其中一个想法就是: 将敏感的信息/算法打包成二进制文件, 然后放到浏览器执行, 如此也会一定程度上帮助网站敏感信息和算法的泄露; 如果算法可以前置到浏览器那么浏览器端也能分担一些后端计算的压力. 所以本文主要探究 wasm, 实现如下内容:

- 使用 go 语言编译 wasm
- 使用 JavaScript 调用和执行 wasm

本文不求甚解, 仅是个 hello world 式的研究.

# 2. Go 和 Wasm

根据 wasm 官网的介绍, Go 的实现有两个:

- [go 实现](https://github.com/golang/go/wiki/WebAssembly#getting-started)
- [tinygo 实现](https://tinygo.org/docs/guides/webassembly/)

其中 tinygo 的实现是基于 go 的, 编辑结果更小, tinygo 适合嵌入式系统开发.

## 2.1 go 实现

### 2.1.1 代码

- 目录结构

  ```
  .
  ├── README.md
  ├── go.mod
  ├── golang-getting-started
  │   ├── index.html
  │   ├── main.go
  ```

- main.go: go 编写的示例代码, 用来编译为 wasm

  ```go
  package main

  func main() {
  	println("hello webassembly")
  }
  ```

- index.html: 前端测试页面, 用来加载 wasm 并执行和验证

  ```html
  <html>
    <head>
      <meta charset="utf-8" />
      <script src="wasm_exec.js"></script>
      <script>
        const exec = async () => {
          const go = new Go();
          const result = await WebAssembly.instantiateStreaming(
            fetch("main.wasm"),
            go.importObject
          );
          await go.run(result.instance);
        };
      </script>
    </head>
    <body>
      <button onclick="exec()">Execute</button>
    </body>
  </html>
  ```

### 2.1.2 编译和测试

```bash
echo "1.拷贝wasm_exec.js"
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" golang-getting-started/

echo "2.编译wasm"
GOOS=js GOARCH=wasm go build -o golang-getting-started/main.wasm golang-getting-started/main.go

echo "3.开启静态网站"
caddy file-server --listen :3000 --root golang-getting-started
```

### 2.1.4 总结

golang 不支持导出方法, 也就是说一个 wasm 文件被执行仅做 main 函数中的操作, 多个方法需要多个 wasm. 结论依据: https://github.com/golang/go/issues/58584

## 2.2 tinygo 实现

### 2.2.1 代码

- 目录结构

  ```
  .
  ├── tinygo-getting-started
  │   ├── index.html
  │   ├── main.go
  ```

- main.go: go 编写的示例代码, 用来编译为 wasm

  ```go
  package main

  import (
  	"fmt"
  )

  func main() {
  	fmt.Println("👋 Hello World 🌍")
  	fmt.Printf("test add function %d\n", add(2, 3))
  	// prevent the function from returning,
  	// which is required in a wasm module
  	<-make(chan bool)
  }

  // This function is imported from JavaScript, as it doesn't define a body.
  // You should define a function named 'add' in the WebAssembly 'env'
  // module from JavaScript.
  //
  //export add
  func add(x, y int) int

  // This function is exported to JavaScript, so can be called using
  // exports.multiply() in JavaScript.
  //
  //export multiply
  func multiply(x, y int) int {
  	return x * y
  }

  ```

- index.html: 前端测试页面, 用来加载 wasm 并执行和验证

  ```html
  <html>
    <head>
      <meta charset="utf-8" />
      <script src="wasm_exec.js"></script>
    </head>
    <body>
      <h1>WASM Experiments</h1>
      <script>
        // This is a polyfill for FireFox and Safari
        if (!WebAssembly.instantiateStreaming) {
          WebAssembly.instantiateStreaming = async (resp, importObject) => {
            const source = await (await resp).arrayBuffer();
            return await WebAssembly.instantiate(source, importObject);
          };
        }

        // Load the wasm file
        const execMain = async () => {
          const go = new Go();
          go.importObject.env.add = function (x, y) {
            return x + y;
          };
          const result = await WebAssembly.instantiateStreaming(
            fetch("main.wasm"),
            go.importObject
          );
          globalThis.wasm = result.instance.exports;
          await go.run(result.instance);
        };
        const execGoFn = () => {
          const result = globalThis.wasm.multiply(5, 3);
          console.log("call go multiply result: %d", result);
        };
      </script>

      <button onclick="execMain()">Execute Main</button>
      <button onclick="execGoFn()">Execute Go Function</button>
    </body>
  </html>
  ```

### 2.2.2 编译和测试

```bash
echo "1.拷贝wasm_exec.js"
cp "$(tinygo env TINYGOROOT)/targets/wasm_exec.js" tinygo-getting-started/

echo "2.编译wasm"
tinygo build -o tinygo-getting-started/main.wasm -target wasm ./tinygo-getting-started

echo "3.开启静态网站"
caddy file-server --listen :3000 --root tinygo-getting-started

```

### 2.2.3 总结

tinygo 实现 wasm 支持 go 的方法导出, javascript 的方法也可以在 golang 中调用, 两种语言通过双向使用, 对比 go 实现来说更方便和高级

# 3. 源码

- 本文代码仓库: [go-wasm](https://github.com/nnsay/go-wasm)
- 社区 golang wasm 示例代码: [01-wasm-golang-browser](https://gitlab.com/k33g_org/wasm.builders/01-wasm-golang-browser/-/tree/main?ref_type=heads)
- [Wasm By Example](https://wasmbyexample.dev/examples/hello-world/hello-world.go.en-us.html)
