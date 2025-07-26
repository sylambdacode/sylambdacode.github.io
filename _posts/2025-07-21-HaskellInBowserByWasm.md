---
title: 在浏览器中运行Haskell代码
tags: [Haskell, WASM, JavaScript]
template: default
---

<a href="https://sylambdacode.github.io/2025/07/21/HaskellInBowserByWasm.html">在浏览器中运行Haskell代码</a> © 2025 by <a href="https://sylambdacode.github.io/">sylambdacode</a> is licensed under <a href="https://creativecommons.org/licenses/by-nc/4.0/">CC BY-NC 4.0</a><img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;">

# 前置工作
1. 安装GHCup，安装方法详见[官网](https://www.haskell.org/ghcup/)。

# 搭建构建环境
## 1. 安装WASM版本的GHC工具链
通过[ghc-wasm-meta](https://gitlab.haskell.org/haskell-wasm/ghc-wasm-meta)安装支持WASM的GHC编译器与相关工具链，本次安装的版本为wasm32-wasi-9.12。

具体安装方法如下，安装时间较长，请耐心等待。如果出现安装失败的情况，可以检查网络是否正常，或者检查磁盘剩余空间是否足够。

```sh
$ curl https://gitlab.haskell.org/haskell-wasm/ghc-wasm-meta/-/raw/master/bootstrap.sh | SKIP_GHC=1 sh
$ source ~/.ghc-wasm/env
$ ghcup config add-release-channel https://gitlab.haskell.org/haskell-wasm/ghc-wasm-meta/-/raw/master/ghcup-wasm-0.0.9.yaml
$ ghcup install ghc wasm32-wasi-9.12 -- $CONFIGURE_ARGS
```

## 2. 安装相对应版本的Cabal
通常情况下安装默认版本的Cabal即可：

```sh
$ ghcup install cabal
```

# 编辑代码
## 1. 创建一个新项目

使用Cabal创建一个新项目：

```sh
$ cabal init --non-interactive
```

## 2. 编辑Haskell代码

删除自动生成的Main.hs并创建Test.hs文件，写入以下内容：

```haskell
module Test where
import GHC.Wasm.Prim -- 导入JavaScript FFI

-- add函数将被导出并由JavaScript调用
foreign export javascript "add"
    add :: Int -> Int -> Int
add a b = a + b

-- stringTest函数将被导出并由JavaScript调用
foreign export javascript "stringTest"
    stringTest :: JSString -> JSString
stringTest inputJSString = toJSString ("stringTest: " ++ haskellString)
    where haskellString = fromJSString inputJSString
```

## 3. 编辑Cabal配置

打开项目的Cabal配置文件，按照下面的例子进行修改：

```cabal
-- 此处省略部分配置

common warnings
    -- 修改此处配置（使用--export导出Haskell函数）
    ghc-options: -Wall -no-hs-main -optl-mexec-model=reactor -optl-Wl,--export=hs_init,--export=add,--export=stringTest

executable test
    -- Import common warning flags.
    import:           warnings

    -- 修改此处配置，由于Test.hs中没有main方法，所以该配置并不起作用。但是必须设置，否则会报错
    -- .hs or .lhs file containing the Main module.
    main-is:          Test.hs

    -- 修改此处配置，添加ghc-experimental依赖
    -- Other library packages from which modules are imported.
    build-depends:    base ^>=4.21.0.0, ghc-experimental

-- 此处省略部分配置
```

## 4. 编写HTML文件

在一个空目录创建test.html，写入下面的代码：

```html
<!DOCTYPE html>

<html >
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>让Haskell代码运行在浏览器中</title>
    </head>
    <body>
        <h1>让Haskell代码运行在浏览器中</h1>
        <script type="module">
            import { Fd, File, Directory, OpenFile, ConsoleStdout, PreopenDirectory, WASI, strace } from "./browser_wasi_shim/index.js";
            async function runHaskellWasm() {
                let env = [""]; // WASI环境变量
                let args = []; // 如果Haskell代码中存在main函数的话，此变量可以作为命令行参数传入Haskell
                // 处理标准输入、输出、错误流
                let fds = [
                    new OpenFile(new File([])), // 标准输入
                    ConsoleStdout.lineBuffered(msg => console.log(`[WASI stdout] ${msg}`)), // 标准输出流（重定向至控制台）
                    ConsoleStdout.lineBuffered(msg => console.warn(`[WASI stderr] ${msg}`)), // 标准错误流（重定向至控制台）
                    new PreopenDirectory(".", [])
                ];
                let wasi = new WASI(args, env, fds);
                let __exports = {};
                const response = await fetch('./test.wasm'); // 读取test.wasm文件
                const buffer = await response.arrayBuffer();
                let instance = (await WebAssembly.instantiate(buffer, {
                    ghc_wasm_jsffi: (await import("./test.js")).default(__exports),
                    "wasi_snapshot_preview1": wasi.wasiImport
                })).instance; // 创建WASM实例
                Object.assign(__exports, instance.exports);
                wasi.initialize(instance);
                instance.exports.hs_init(0, 0); // 执行Haskell初始化函数
                // 执行Haskell代码中的add函数
                instance.exports.add(1, 2)
                    .then(result => {
                        console.log(`add(1, 2) = ${result}`);
                    });
                // 执行Haskell代码中的stringTest函数
                instance.exports.stringTest("hello")
                    .then(result => {
                        console.log(`stringTest("hello") = "${result}"`);
                    });
            }
            runHaskellWasm();
        </script>
    </body>
</html>
```

# 编译运行

## 1. 编译生成WASM文件

首先将Haskell代码编译为WASM文件：

```sh
$ cabal --with-compiler=wasm32-wasi-ghc-9.12 --with-hc-pkg=wasm32-wasi-ghc-pkg-9.12 --with-hsc2hs=wasm32-wasi-hsc2hs-9.12 --with-haddock=wasm32-wasi-haddock-9.12 build
```

上面的命令执行完成后，将会在Haskell工程的`dist-newstyle`文件夹下生成test.wasm文件，笔者生成的文件路径位于：`/dist-newstyle/build/wasm32-wasi/ghc-9.12.2.20250327/test-0.1.0.0/x/test/build/test/test.wasm`。将该WASM文件复制到test.html文件的同级目录下，以便JavaScript代码能够顺利加载。

## 2. 生成JavaScript FFI代码

通过执行下面的命令，生成JavaScript FFI代码：

```sh
$ $(wasm32-wasi-ghc-9.12 --print-libdir)/post-link.mjs -i test.wasm -o test.js
```

执行成功后，在test.wasm文件同级目录下可以发现test.js文件。**如果没有成功生成test.js文件，请先执行下面的命令：**

```sh
$ source ~/.ghc-wasm/env
```

## 3. 编译[browser_wasi_shim](https://github.com/bjorn3/browser_wasi_shim)

[browser_wasi_shim](https://github.com/bjorn3/browser_wasi_shim)是一个使用纯JavaScript编写而成的WASI环境，能够在浏览器中提供WASI相关接口。

首先使用git克隆[browser_wasi_shim](https://github.com/bjorn3/browser_wasi_shim)项目：

```sh
git clone https://github.com/bjorn3/browser_wasi_shim.git
```

之后进入browser_wasi_shim项目根目录，执行下面的命令：

```sh
$ npm install
$ npm run build
```

执行成功后，在browser_wasi_shim项目根目录下会生成dist文件夹。在test.html文件同级目录下创建browser_wasi_shim文件夹，并将dist文件夹中的内容复制到刚刚创建的browser_wasi_shim目录下。

## 4. 运行

最终的文件目录如下图：

![测试文件目录](/static/images/2025-07-21-HaskellInBowserByWasm/DirectoryStructure.png)

打开浏览器，进入test.html页面（需要用HTTP的形式），并打开开发者工具可以看到执行结果如下图所示。

![运行结果](/static/images/2025-07-21-HaskellInBowserByWasm/RunResult.png)