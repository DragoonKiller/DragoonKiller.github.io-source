---
title: C#/Mono on Linux 入门指南

tags:
- C#
- Mono
- Linux

date: 2017-12-4 23:57:55
---

教你如何在 Linux 上耍 C#.

**注意:** 如果你选择 MonoDevelop, 那么这篇文章绝大部分对你没有用处.  作为一个vscode选手, 折腾一下这种东西还是有必要的.

<!-- more -->

#### 安装 Mono (不是 MonoDevelop)

全家桶:
```bash
sudo aptitude install mono-complete
```

运行时:
```
sudo aptitude install mono-runtime
```

### 编译C#源程序
编译一个文件, 默认输出为可执行文件. 注意虽然输出的可执行文件后缀名是 `exe`, 但是它是可以在 linux 下跑的...
```
mcs ./x.cs
```
 
指定输出文件名:
```
mcs ./x.cs -out:a.exe
```
注意冒号不能省, 在windows下参数以斜杠 / 开头不过 mono 两者都能用.

可以使用正则匹配.
```
mcs ./*.cs -out:a.exe
```
想链接外部.Net库(通常是**编译成IL的**dll文件):
```
mcs ./*.cs -out:a.exe -resource:./OpenTK.dll
```
使用相对路径. 使用此方法生成的程序(*.exe)需要和依赖库放在同一目录下, 即使编译时它们处于不同目录下.

如果不想把源程序编译成可执行代码, 而是编译成一个dll, 可以:
```
mcs ./*.cs -target:library -out:x.dll
```

### 运行C#程序

使用 mono 运行时:
```
mono ./x.exe
```

似乎在某些情况下可以直接运行:
```
./x.exe
```

### C#工程文件

微软自家的工程以解决方案(Solution)为最顶层的元素.  
一个解决方案包含至少一个项目(Project).  

项目是基本的编译单元, 一般来说一个项目产生一个dll/exe, 但是你可以自定义编译链接流程来执行一些奇怪的操作.(并不确定真的是这样)  

工程文件, 例如 x.csproj, 内容大概长这样:
```xml
<?xml version="1.0" encoding="utf-8"?>
<Project DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ToolVersion="14.0">
  
  <PropertyGroup>
    <OutputType>exe</OutputType>
    <OutputPath>./bin</OutputPath>
    <AssemblyName>Tokamach</AssemblyName>
    <TargetFrameworkVersion>v3.5</TargetFrameworkVersion>
    <FileAlignment>512</FileAlignment>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>
  
  <ItemGroup>
    <Compile Include="./src/*.cs" />
    <Compile Include="./src/**/*.cs" />
    <Reference Include="System" />
    <Reference Include="System.Drawing" />
    <Reference Include="OpenTK">
      <HintPath>./lib/OpenTK.dll</HintPath>
    </Reference>
  </ItemGroup>
  
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
</Project>
```
注意最后一行` Import` 实际上帮我们配置了一堆默认的 Target. 如果你不想手动配置 Target, 这行不能省略. 如果你使用 OmniSharp, 你需要按照要求配很多个Target, 而且好像并不能查到你需要配哪些 Target 并且需要哪些参数, 所以一般都会直接用这行.

从这个例子中我们可以看到:
1. 想要添加源文件, 就在ItemGroup里写上`<Compile Include="..." />`.
1. 支持正则匹配.
1. 想要链接一个公用的运行时库, 就在ItemGroup里写上`<Reference Include="..." />`.
1. 如果你的库是从别的地方搞来的, 在ItemGroup里要标记这个库的名称和相对路径, 例如`<Reference Include="OpenTK"><HintPath>./lib/OpenTK.dll</HintPath></Reference>`. 注意HintPath不要分行写.
1. 编译好的exe文件会被直接扔到`<OutputPath>`标签所在路径.

构建工程:
```
msbuild
```
如果你的目录里有解决方案文件(.sln文件), 它会读sln文件并构建(Build)其中声明的所有项目. 如果没有, 它会寻找项目文件(.csproj文件, .vcproj文件等)并构建它.

注意, 编译完成以后, 所有外部依赖库文件会被全部拷贝到和目标可执行程序相同目录下(就是`<OutputPath>`指明的路径).

### 调用C/C++的动态库
首先按照正常方式写一个动态库:
```cpp
// a-plus-b.cpp
extern "C" int plus(int a, int b) { return a + b; }
```
把它编译成动态库:
```bash
gcc ./a-plus-b.cpp -shared -fPIC -o liba-plus-b.so
```
上述语句中, `-shared` 表示生成一个动态链接库, `-fPIC` 表示生成"地址独立"的函数(呃我也不知道什么意思, 反正是 shared library 必备玩意.)  
根据 [linux的文档](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html), 一般我们会把动态链接库放到 `/lib/` 或 `/usr/lib/` . 那么我们把我们的动态链接库拷到 `/usr/lib/` :
```bash
sudo cp ./liba-plus-b.so /usr/lib/
```
然后就可以使用C#提供的动态链接:
```CSharp
using System.Runtime.InteropServices;
...
    [DllImport("a-plus-b")]
    public static extern int plus(int a, int b);
...
```
然后就可以当普通函数用了.

注意如果在windows环境下, 上面的 C# 程序会搜索 `a-plus-b.dll`, 并且首先搜索程序目录, 其次搜索程序运行目录. 实际上, 尽管 C# 代码不依赖于平台, 你需要为不同的平台提供不同的库文件.

更多详细信息参见 [Mono的文档](http://www.mono-project.com/docs/advanced/pinvoke/).

如果你使用 C++ 的方式编译你的库文件(不加`extern "C"`), 你的函数名称会按照某种规则(目前没找到是啥规则...)被修改. 比如我写:
```cpp
int plus(int a, int b) { return a + b; }
int plus(int a, int b, int c) { return a + b + c; }
```
可以使用`nm`命令查看动态链接库中的函数名称:
```bash
nm -D /usr/lib/liba-plus-b.so
```
输出:
```bash
...
0000000000000680 T _Z4plusii
0000000000000694 T _Z4plusiii
```
于是我们可以在我们的C#程序中这样调用函数:
```CSharp
[DllImport("a-plus-b")]
public static extern int _Z4plusii(int a, int b);
```
很丑..

