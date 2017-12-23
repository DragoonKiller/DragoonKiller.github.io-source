---
title: C#反射及其高级操作
date: 2017-12-05 19:55:45
tags:
---

详细讲解C#的反射, 实时编译, 动态加载程序集(Assembly)等功能.

<!-- more -->

### C#的编译机制

C#的编译器会把C#源代码会编译成IL代码, 存在.exe或.dll文件中. 除了IL代码以外, 文件中还存有一些别的信息, 比如类型信息, 以及用户自定义的特性(Attribute). 这些代码逻辑之外的信息被叫做元数据(metadata).  
反射就是读取并使用这些元数据的一种机制, 是C#元编程的途径之一.  

程序集(Assembly)是最小的编译目标(Target). 通常我们会把某些文件编译到一个.exe或.dll, 它就是一个程序集. 这个概念和java的包(package), 以及python的模块(module)差不多. 不过需要注意一下, 尽管它的后缀名是 .exe/.dll, 它的格式与VisualC++编译结果有很大不同.

如果你写的程序要使用一个程序集(或者说一个外部库), 我们会在编译或构建的时候指明:  
`csc ./x.cs -out:x.exe -reference:SomeLib.dll # for csharp compiler.`  
或者写到msbuild中:  
`<Reference Include="SomeLib"> <HintPath> ... </HintPath> </Reference>`  
在我们的源代码里就可以使用了:  
`using SomeLib; ...; `

### 反射

你可以通过反射机制调用你自己的程序中的元数据. 像这样:
```CSharp
using System.Reflection;
using static System.Console;
public class __Main__
{
    public int x { get; private set; }
    public int y;
    public int One(int g) { return g; }
    
    public static void Main(string[] args)
    {
        Type t = typeof(__Main__); // 对于一个(编译期的)已知类型, 使用typeof关键字.
        
        // 输出"__Main__"
        WriteLine(t.name);
        
        // 输出所有成员函数名称, 包括属性(properties), 默认构造方法(.ctor)等.
        foreach(var m in t.GetMembers())
            WriteLine(m.Name);
        
        // 调用方法; 输出"2". 注意对于变长参数和虚函数, 第三个参数Binder需要自定义.
        WriteLine((int)t.InvokeMember("One", BindingFlags.InvokeMethod, null, null, new object[]{2}));
    }
}
```
可以使用 `Activator.CreateInstance(Type)` 创建对应类型的对象.  
可以使用 `object.GetType()` 获取一个表示对象的实际类型的Type对象.  
此外还有 `Type.GetFields()` 等函数, 可以拿到除了源代码意外的一切信息(源码被编译成IL或汇编了).  
可以使用 `Type.GetTypeCode(Type)` 获取内置基本类型的类型编码(用于switch-case).

### 动态编译

CodeDom的全称是**Code Documentation Object Model**, 是以XML的形式描述语言的一套规则和配套API. 它提供了类似于类型机制, 语法结构树等一堆用来描述代码的类. 这些类可以用来方便地生成支持语言的源代码, 或者从源代码提取逻辑. 不过我们这里不使用它们...  
CodeDom提供了一套编程语言支持系统. 据说功能和编译器**Roslyn**重复了, 不过资料还是够多.  
CodeDom内置三种语言的编译, 分别是C#, VB .Net, 还有JScript(微软自家的web编程语言, 好像是TypeScript前身).  

我们可以用CodeDom进行实时文件编译, 像这样:
```CSharp
using System.CodeDom;
using System.CodeDomCompiler;

// 下方代码写在函数里...
var pr = CodeDomProvider.CreateProvider("C#");
var opt = new CompilerParameters();
opt.GenerateExecutable = false; // true则生成可执行文件(带有入口点的程序集), false生成链接库. 默认false.
opt.GenerateInMemory = true; // 如果为true则直接生成到内存中, 不会保存到文件, 程序结束后直接销毁. 默认false.
opt.ReferencedAssemblies.Add("System"); // 外部库的绝对路径. 除非是系统库(放在编译器相同目录下)可以不写路径.
CompilerResults cr = pr.CompileAssemblyFromSource(opt, new string[]{source}); // 编译.
if(am == null) // 检查编译结果.
{
    WriteLine("Compile failed!");
    foreach(var err in cr.Errors) WriteLine(err.ToString());
    return;
}[p
Assembly am = cr.CompiledAssembly; // 编译成功了就拿下程序集.
foreach(Type c in am.GetTypes()) // 枚举程序集中每一个类. 然后干该干的事情...
{
    WriteLine("Class/Struct: " + c.Namespace + "::" + c.Name);
    c.InvokeMember("gg", BindingFlags.InvokeMethod, null, null, null);
    ...
}
```
除此以外, CodeDomProvider还能解释执行C#代码.

### 加载和删除程序集
应用程序域(AppDomain, 下文简称域)是应用程序(Application)的一个子环境. 每个域都包含若干个被加载入内存中的程序集. 一个运行中的应用程序由至少一个域组成, 程序中的每一个线程都仅在一个域内运行其中的代码, 但是一个线程可以跳到别的域执行. 主线程(Main函数所在线程)运行在主应用程序域(main AppDomain)中. 在程序一开始执行Main函数之前, 运行时环境(CLR)会创建主域, 并加载所有需要的程序集——就是在编译选项中用`<Refernece ...>`指定的那些库, 然后创建主线程, 运行主函数.  
加载进应用程序域的程序集不能卸载. CLR会解析程序集并把它编译到应用程序域中, 组织函数的入口点, 编译跳转指令(大概就是运行时链接吧). CLR还可能根据程序的运行状况和代码特点做内联编译来加速运行. 创建一个应用程序域并加载一个程序集的例子如下:
```CSharp
// u.cs, 编译到u.dll.
// 注意编译时的文件名就是编译后这个程序集的名称. 所以要先编译到 *.dll, 再拷贝到想要的目录下.
using System;
using static System.Runtime.InteropServices.Marshal;
public class FunctionInvoker : MarshalByRefObject // 继承自MarshalByRefObject
                                                  // 会使对象在域之间传递时作引用传递而不是值传递.f
{
    ...
}
```
```CSharp
using System;
// 某个函数里...
WriteLine(AppDomain.CurrentDomain.FriendlyName); // 使用当前线程所在程序集.

AppDomain domain = new AppDomain("NewDomain"); // 给创建的域一个"友好的"名字.
WriteLine(domain.FriendlyName); // 输出"NewDomain".

// 使用下面方法把程序集加载到指定的域中.
// 注意我们不使用 AppDomain.Load 函数, 设定上, 这个函数只能被当前域调用.
// 这个函数在NewDomain内创建一个程序集 y.dll 中定义的DK.Test对象.
// 注意这个操作会让NewDomain加载 y.dll 程序集, 但是当前域不会.
domain.CreateInstanceFrom("./bin/y.dll", "DK.Test");

// 现在我们拿到了NewDomain域里的一个新对象的引用. 注意这个操作*会*把对应程序集加载到当前域.
var r = domain.CreateInstanceFromAndUnwrap(
    "./bin/u.dll",
    "FunctionInvoker");
```
现在当前域加载了程序集 `u.dll`, 我们新建的域加载了程序集 `u.dll` 和 `y.dll` ; 新建的域中还存有一个 `DK.Test` 对象, 一个 `FunctionInvoker` 对象; 在当前域中存有对新的域的引用, 还有一个对上面提到的 `FunctionInvoker` 对象的引用.  
显然, 当前域加载了程序集 `u.dll` , 这样当前域和那个叫做 NewDomain 的域分别持有一份对程序集`u.dll` 的一份拷贝. 现在我们希望在不加载 `y.dll` 的情况下访问另一个AppDomain中的函数.  

如果读者足够细心应该会发现上面某个类叫做 `FunctionInvoker`. 我们需要一种协议来在域之间做通信. 这种协议可以用反射实现, 举例如下:
```CSharp
// u.cs -> u.dll -> ./bin/u.dll
using System;
using System.Reflection;
public class FunctionInvoker : MarshalByRefObject
{
    public void Invoke(string type, string name, object target, params object[] args)
    {
        foreach(var a in AppDomain.CurrentDomain.GetAssemblies())
        {
            Type t = a.GetType(type);
            if(t != null)
            {
                t.InvokeMember(name, BindingFlags.InvokeMethod, null, target, args);
            }
        }
    }
}
```
```CSharp
// 在某函数中...
// 创建新域.
AppDomain domain = new AppDomain("NewDomain");
// 新域加载程序集 y.dll 并创建 DK.Test 对象.
domain.CreateInstanceFrom("./bin/y.dll", "DK.Test");
// 新域加载程序集 u.dll 并创建 FunctionInvoker 对象. 当前域加载程序集 u.dll.
var r = (FunctionInvoker)domain.CreateInstanceFromAndUnwrap(
    "./bin/u.dll",
    "FunctionInvoker");
// 调用新域中的对象的函数.
r.Invoke("DK.Test", "Out", null);
```
注意这个`Invoke`函数. 它在程序集 u.dll 中被定义, 被两个域一起加载. 那这个`Invoke`是调用哪个域中的呢? 答案是新域NewDomain. 可以在 Invoke 函数中把 CurrentDomain.FriendlyName 输出出来印证这一点. 这样上面的 Invoke函数会执行并调用新域中的程序集 y.dll 中 DK.Test 类的 Out 方法. 由于是静态方法所以不需要传递对象.

于是我们在当前域未加载 y.dll 的情况下调用了另一个域中程序集 y.dll 中的函数.

如果前面 FunctionInvoker 不继承 MarshalByRefObject, 那么通过反射调用的是自己的应用程序域中的函数.

另外, 执行Invoke函数的线程会跳到另一个域中执行.

删除程序集很简单, 把AppDomain删掉即可: `AppDomain.Unload(domain);`
结合上边的即使编译, 我们就可以在不掐掉程序的情况下动态实时更新程序集了. 不过如果我们需要修改协议类FunctionInvoker, 那就只能重启程序了.
