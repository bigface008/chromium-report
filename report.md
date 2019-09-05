# Chromium 暑期工作报告

汪喆昊 2019/8/28

- [Chromium 暑期工作报告](#chromium-暑期工作报告)
  - [准备工作](#准备工作)
  - [Chromium 编译](#chromium-编译)
    - [系统要求](#系统要求)
    - [安装 depot_tools](#安装-depot_tools)
    - [获取源码](#获取源码)
    - [安装依赖](#安装依赖)
    - [生成 Build 文件](#生成-build-文件)
    - [加速编译的参数](#加速编译的参数)
      - [`use_jumbo_build`](#use_jumbo_build)
      - [`enable_nacl`](#enable_nacl)
      - [`symbol_level`](#symbol_level)
      - [`blink_symbol_level`](#blink_symbol_level)
    - [编译 Chromium](#编译-chromium)
    - [运行 Chromium](#运行-chromium)
  - [Chromium 模型分离渲染原理](#chromium-模型分离渲染原理)
    - [参考文档/博客列表](#参考文档博客列表)

## 准备工作

需要准备好梯子。

实验室内网可以直接用服务器，所以在命令行内输入

```shell
$ export http_proxy="http://host:port"
$ export https_proxy="http://host:port"
```

也可以放到`.bashrc`，`.zshrc`里面。

接下来需要科学上网的步骤应该都没有问题。当然也可以直接

```shell
$ export ALL_PROXY="http://host:port"
```

Shadowsocks用户可以手动设置全局代理以实现上述效果，不过，shadowsocks-qt5没有自带的全局代理功能。
所以，可以使用如下命令达到上述效果。

```shell
$ export http_proxy="socks5://host:port"
$ export https_proxy="socks5://host:port"
```

或直接设置`ALL_PROXY`。

```shell
$ export ALL_PROXY="socks5://host:port"
```

这样`wget`，`curl`也会使用shadowsocks代理。

网上也有很多人推荐使用[`proxychains`](https://blog.fazero.me/2015/08/31/%E5%88%A9%E7%94%A8proxychains%E5%9C%A8%E7%BB%88%E7%AB%AF%E4%BD%BF%E7%94%A8socks5%E4%BB%A3%E7%90%86/)来做shadowsocks全局代理，不过我用这个方法失败了......

chromium源码有14GB左右，据谷歌官网的说法，速度较快的连接也可能要下30分钟，所以其实不大推荐用代理来下源码（连接速度慢且也许不大稳定，另外，某一个连接时间过长也许会增加服务器被封的几率？我瞎猜的）。我的解决方案是下到远程服务器，然后打包压缩，`scp`到本地。（不过这样搞也有很多问题）

## Chromium 编译

官方编译指南请看[链接](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md)。

### 系统要求

- 64-bit 的 Intel 机器。
- 内存至少 8GB，一般 16GB 以上比较合适。
- 谷歌官网建议保留 100GB 的磁盘剩余空间。
- Git 和 Python2。
- 谷歌官方是在 Ubuntu 16.04 中进行编译的。

### 安装 depot_tools

`depot_tools` 内部包含了一系列管理 Chromium 源码的工具。

```shell
$ # 自然地，请准备好代理。
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

接着将管理工具添加到环境变量 `$PATH`。

```shell
$ export PATH="$PATH:/path/to/depot_tools"
```

注意这里不要使用“~”，而应当使用 ${HOME}，不然接下来的一些命令可能出问题。

```shell
$ export PATH="$PATH:${HOME}/depot_tools"
```

### 获取源码

创建工作目录。

```shell
$ mkdir ~/chromium && cd ~/chromium
```

获取源代码。

```shell
$ # 自然地，请准备好代理。
$ fetch --nohooks chromium
```

如果你不需要源码管理的历史记录，加上 flag `--no-history`。

```
$ fetch --no-history --nohooks chromium
```

### 安装依赖

```shell
$ cd src
$ # 自然地，请准备好代理。
$ ./build/install-build-deps.sh
```

`install-build-deps.sh` 会尝试安装 Chrome OS Font，如果你不想安装的话，加上 flag `--no-chromeos-fonts`。（这些参数打开这个脚本文件就能看到对应的注释）

下载其他编译可能需要的文件。

```shell
$ # 自然地，请准备好代理。
$ gclient runhooks
```

### 生成 Build 文件

Chromium 使用 [Ninja](https://ninja-build.org/) 来作为主要的 Build 工具。先安装 Ninja。

```shell
$ sudo apt install ninja-build
```

`depot_tools` 中的 [`gn`](https://gn.googlesource.com/gn/+/master/docs/quick_start.md) 命令被用来生成 Build 文件。

```shell
$ # 需要的话可以将 Default 换成其他名字。
$ gn gen out/Default
```

可以设置一些参数来加速编译。在 `gn` 命令中加入 `arg` 会启动 vim，接着输入所需的参数并保存，可以创建或者更新 Build 文件。

```shell
$ gn args out/my_build
```

可以通过 flag `--list` 来查看所有的参数值。

```shell
$ gn args --list out/my_build
```

### 加速编译的参数

在使用 `gn args out/my_build` 命令打开 vim 后，输入如下参数。

    use_jumbo_build=true
    enable_nacl=false
    symbol_level=1
    blink_symbol_level=0

#### `use_jumbo_build`

jumbo build 会合并一系列 translation unit（source file）并一起编译，从而达到加速的效果。利用 `use_jumbo_build=true` 来启用这一功能。

#### `enable_nacl`

Native Client（NACL）是一种在浏览器内运行编译过的 C/C++ 代码，且不依赖操作系统的沙箱技术。考虑到我们不大关心这个功能，利用 `enable_nacl=false
` 来关闭这一功能的编译。

#### `symbol_level`

默认情况下，`gn` 创建的 Build 文件是编译模式的，会有设置 `is_debug=true` 和 `symbol_level=2` 。把 `symbol_level` 设置为 0 或 1 可以让编译速度比输出所有symbol的情况快一些。

#### `blink_symbol_level`

使用大量 template 的 Blink 排版引擎会生成大量的 debug symbol。利用 `blink_symbol_level=0` 来阻止其减慢编译。

此外，可以利用 `ccache` 来缓存部分编译所需文件，用 `tmpfs` 来挂载部分文件系统到 RAM，从而加速读取。因为这两种操作不适合我的情况，所以我没有对其进行记录。详细说明请看[这个链接](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md#faster-builds)。

### 编译 Chromium

```shell
$ # chromium 对应的 target 是 chrome。
$ autoninja -C out/Default chrome
```

运行命令 `gn ls out/Default` 可以查看所有的 build target。若要编译其中某个 target（比如 `//chrome/test:unit_tests`），可以这么做。

```shell
$ autoninja -C out/Default chrome/test:unit_tests
```

### 运行 Chromium

```shell
$ out/Default/chrome
```

## Chromium 模型分离渲染原理

### 参考文档/博客列表

- [Chromium多线程模型设计和实现分析](https://blog.csdn.net/luoshengyang/article/details/46855395)
- [Chromium的GPU进程启动过程分析](https://blog.csdn.net/Luoshengyang/article/details/48123761)
- [Siggraph上的Chromium模型分离渲染的论文](https://graphics.stanford.edu/papers/cr/cr_lowquality.pdf)

谷歌官方的Chromium的GPU相关文档请看[这里](https://www.chromium.org/developers/design-documents/chromium-graphics)。

下面的内容是我自己阅读了里面的一些文章以及谷歌之后做出的总结。

