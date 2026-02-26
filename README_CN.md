[![Issues](https://img.shields.io/github/issues/thewover/donut)](https://github.com/TheWover/donut/issues)
[![Contributors](https://img.shields.io/github/contributors/thewover/donut)](https://github.com/TheWover/donut/graphs/contributors)
[![Stars](https://img.shields.io/github/stars/thewover/donut)](https://github.com/TheWover/donut/stargazers)
[![Forks](https://img.shields.io/github/forks/thewover/donut)](https://github.com/TheWover/donut/network/members)
[![License](https://img.shields.io/github/license/thewover/donut)](https://github.com/TheWover/donut/blob/master/LICENSE)

**[English](README.md)**

![Donut Logo](https://github.com/TheWover/donut/blob/master/img/donut_logo_white.jpg?raw=true)

当前版本：[v1.1](https://github.com/TheWover/donut/releases)

## 目录

1. [简介](#简介)
2. [工作原理](#工作原理)
3. [编译构建](#编译构建)
4. [使用方法](#使用方法)
5. [子项目](#子项目)
6. [二次开发](#二次开发)
7. [问题讨论](#问题讨论)
8. [免责声明](#免责声明)

## 简介

**Donut** 是一个位置无关的 Shellcode 生成器，支持在内存中执行 VBScript、JScript、EXE、DLL 文件和 .NET 程序集。生成的模块可以从 HTTP 服务器分阶段加载，也可以直接嵌入到加载器中。模块可选使用 [Chaskey](https://tinycrypt.wordpress.com/2017/02/20/asmcodes-chaskey-cipher/) 分组密码和 128 位随机密钥进行加密。文件在内存中加载执行后，原始引用会被擦除以对抗内存扫描。

生成器和加载器支持以下功能：

- 使用 aPLib 和 LZNT1、Xpress、Xpress Huffman（通过 RtlCompressBuffer）压缩输入文件
- 使用熵值生成 API 哈希和字符串
- 128 位对称加密
- 覆写原生 PE 头
- 将原生 PE 存储在 MEM_IMAGE 内存中
- 绕过反恶意软件扫描接口（AMSI）和 Windows 锁定策略（WLDP）
- 绕过 Windows 事件跟踪（ETW）
- 修补 EXE 文件的命令行
- 修补退出相关 API 以避免终止宿主进程
- 多种输出格式：C、Ruby、Python、PowerShell、Base64、C#、十六进制、UUID 字符串

Linux 和 Windows 均提供动态库和静态库，可集成到你自己的项目中。还有一个 Python 模块，详见 [构建和使用 Python 扩展](https://github.com/TheWover/donut/blob/master/docs/2019-08-21-Python_Extension.md)。

## 工作原理

Donut 为每种支持的文件类型提供了独立的加载器：

**dotNET EXE/DLL 程序集**：使用非托管 CLR 宿主 API 加载公共语言运行时（CLR）。CLR 加载到宿主进程后，创建新的应用程序域（AppDomain）来运行程序集。AppDomain 就绪后，通过 `AppDomain.Load_3` 方法加载 .NET 程序集，最后调用用户指定的入口点方法。参考 MSDN 的[非托管 CLR 宿主 API](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/clr-hosting-interfaces) 文档。独立 CLR 宿主示例见[此处](https://github.com/TheWover/donut/blob/master/DonutTest/rundotnet.cpp)。

**VBScript / JScript**：使用 IActiveScript 接口执行，并对 Windows Script Host（wscript/cscript）的部分方法提供最小支持。独立示例见[此处](https://gist.github.com/odzhan/d18145b9538a3653be2f9a580b53b063)。详细说明：[内存执行 JavaScript、VBScript、JScript 和 XSL](https://modexp.wordpress.com/2019/07/21/inmem-exec-script/)。

**原生 EXE/DLL**：使用自定义 PE 加载器执行，支持延迟导入、TLS 和命令行修补。仅支持包含重定位信息的文件。详见 [内存执行 DLL](https://modexp.wordpress.com/2019/06/24/inmem-exec-dll/)。

**安全绕过**：加载器可以禁用 AMSI 和 WLDP 以规避内存中恶意文件的检测。详见 [Red Team 如何绕过 AMSI 和 WLDP](https://modexp.wordpress.com/2019/06/03/disable-amsi-wldp-dotnet/)。支持使用 aPLib 或 RtlDecompressBuffer API 在内存中解压文件。详见[数据压缩](https://modexp.wordpress.com/2019/12/08/shellcode-compression/)。

**ETW 绕过**：v1.0 起支持。默认绕过方案源自 XPN 的研究。详见 [隐藏你的 .NET - ETW](https://blog.xpnsec.com/hiding-your-dotnet-etw/)。

**PE 头处理**：默认情况下，加载器会覆写非托管 PE 的头部（从基地址到 `IMAGE_OPTIONAL_HEADER.SizeOfHeaders`）。如果未使用诱饵模块，PE 头会被清零；如果使用了诱饵模块，则用诱饵模块的 PE 头覆写。用户也可以选择保留原始 PE 头。

完整技术细节参考 [Donut - 将 .NET 程序集注入为 Shellcode](https://thewover.github.io/Introducing-Donut/) 和 [开发者笔记](https://github.com/TheWover/donut/blob/master/docs/devnotes.md)。

## 编译构建

### 克隆仓库

```bash
git clone http://github.com/thewover/donut.git
```

### Windows

使用 MSVC（启动 x64 Developer Command Prompt）：

```
nmake -f Makefile.msvc
```

使用 MinGW-64：

```
make -f Makefile.mingw
```

### Linux

```
make
```

将生成动态库 `donut.so`、静态库 `donut.a` 和生成器 `donut`。

### Python 模块

```bash
# 卸载旧版本
pip3 uninstall donut-shellcode

# 从源码安装
pip3 install .

# 或从 PyPI 安装
pip3 install donut-shellcode
```

详见 [构建和使用 Python 扩展](https://github.com/TheWover/donut/blob/master/docs/2019-08-21-Python_Extension.md)。

### Docker

```bash
# 构建容器
docker build -t donut .

# 运行
docker run -it --rm -v "${PWD}:/workdir" donut -h
```

### 辅助工具

Donut 包含几个可单独编译的辅助工具：

- `hash.exe` / `encrypt.exe` — 用于 Shellcode 生成
- `inject.exe` — 通过 PID 或进程名将 loader.bin 注入到目标进程
- `inject_local.exe` — 将 loader.bin 注入到自身进程（测试用）

```
nmake inject_local -f Makefile.msvc
```

### 历史版本

- [v0.9.3, TBD](https://github.com/TheWover/donut/releases/tag/v0.9.3)
- [v0.9.2, Bear Claw](https://github.com/TheWover/donut/releases/tag/v0.9.2)
- [v0.9.1, Apple Fritter](https://github.com/TheWover/donut/releases/tag/v0.9.1)
- [v0.9.0, Initial Release](https://github.com/TheWover/donut/releases/tag/v0.9)

其他生成器实现：
- [C# 版 by n1xbyte](https://github.com/n1xbyte/donutCS)
- [Go 版 by awgh](https://github.com/Binject/go-donut)

## 使用方法

| 参数 | 值 | 说明 |
|------|-----|------|
| `-a` | arch | 目标架构：1=x86, 2=amd64, 3=x86+amd64（默认） |
| `-b` | level | AMSI/WLDP 绕过行为：1=不绕过, 2=失败则中止, 3=失败则继续（默认） |
| `-k` | headers | PE 头处理：1=覆写（默认）, 2=保留 |
| `-j` | decoy | 诱饵模块路径（用于模块覆盖） |
| `-c` | class | .NET 类名（.NET DLL 必填），可含命名空间：`namespace.class` |
| `-d` | name | .NET AppDomain 名称，启用熵值时随机生成 |
| `-e` | level | 熵值级别：1=无, 2=随机名称, 3=随机名称+对称加密（默认） |
| `-f` | format | 输出格式：1=二进制（默认）, 2=Base64, 3=C, 4=Ruby, 5=Python, 6=PowerShell, 7=C#, 8=十六进制 |
| `-m` | name | DLL 的方法/函数名（.NET DLL 必填） |
| `-n` | name | HTTP 分阶段加载的模块名，启用熵值时随机生成 |
| `-o` | path | 输出路径，默认为当前目录的 `loader.bin` |
| `-p` | params | DLL 方法/函数或 EXE 的参数（用引号包裹） |
| `-r` | version | CLR 运行时版本，默认使用 MetaHeader 或 v4.0.30319 |
| `-s` | server | HTTP 服务器 URL，支持凭据：`https://user:pass@192.168.0.1/` |
| `-t` | | 以线程方式运行非托管 EXE 入口点并等待结束 |
| `-w` | | 以 UNICODE 格式传递命令行给非托管 DLL（默认 ANSI） |
| `-x` | option | 退出方式：1=退出线程（默认）, 2=退出进程, 3=不退出，无限阻塞 |
| `-y` | addr | 创建新线程运行加载器，完成后跳转到宿主进程的指定偏移地址继续执行 |
| `-z` | engine | 压缩引擎：1=无, 2=aPLib, 3=LZNT1, 4=Xpress, 5=Xpress Huffman（后三种仅 Windows） |

### Payload 要求

#### .NET 程序集

- 入口点方法只能接受字符串参数或无参数
- 入口点方法必须标记为 `public static`
- 包含入口点的类必须标记为 `public`
- 不能是混合程序集（同时包含托管和原生代码）
- 不能包含非托管导出

#### 原生 EXE/DLL

- 不支持 Cygwin 编译的二进制文件（Cygwin 的初始化例程依赖磁盘文件）

#### 非托管 DLL

- 用户指定的入口点方法只能接受一个字符串参数或无参数。[示例](https://github.com/TheWover/donut/blob/master/DonutTest/dlltest.c/)

## 子项目

| 工具 | 说明 |
|------|------|
| DemoCreateProcess | 测试用 .NET 程序集示例，接受两个命令行参数指定要执行的程序 |
| Donut简单的 C# Shellcode 注入器，用于测试 Donut（Shellcode 需 Base64 编码后作为字符串传入） |
| ModuleMonitor | 概念验证工具，检测 CLR 注入（如 Donut 和 Cobalt Strike 的 execute-assembly） |
| ProcessManager | 进程发现工具，可查看运行中的进程属性及是否加载了 CLR |

## 二次开发

如需添加更多 Payload 类型支持、修改功能或将 Donut 集成到现有工具中，请参考[开发者文档](https://github.com/TheWover/donut/blob/master/docs/devnotes.md)。

建议扩展方向：

- 添加环境密钥绑定
- 实现多态性——每次生成 Shellcode 时混淆加载器
- 将 Donut 作为模块集成到你的 RAT/C2 框架中

## 问题讨论

如有任何问题或建议，请加入 [BloodHound Gang Slack](https://bloodhoundgang.herokuapp.com/) 的 #Donut 频道。

## 免责声明

我们不对本软件或技术的任何滥用负责。Donut 作为 CLR 注入和通过 Shellcode 进行内存加载的演示项目，旨在为红队提供模拟对手的手段，为防御方提供构建分析和缓解措施的参考框架。这不可避免地存在被恶意软件作者和威胁行为者滥用的风险，但我们认为净收益大于风险。如果 EDR 或 AV 产品能够通过签名或行为模式检测到 Donut，我们不会更新 Donut 来对抗检测。请勿就此提出要求。
