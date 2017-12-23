---
title: OpenTK 入门指南
date: 2017-10-24 20:26:25
tags:
- C#
- OpenTK
---

OpenTK 是C#对 OpenGL, OpenAL 和 OpenCL 的封装.  
这里只讲 OpenTK 基础用法, 以及简要介绍其中一些组件, 不教以上三套API.

<!-- more -->

从网上下载OpenTK源代码然后编译得到一个`OpenTK.dll`.  
在工程中引用它:
```xml
<ItemGroup>
...
    <Reference Include="System.Drawing" />
    <Reference Include="OpenTK">
        <HintPath>./lib/OpenTK.dll<HintPath>
    </Reference>
...
</ItemGroup>
```
注意 `System.Drawing` 是OpenTK的一个依赖库.  
如果你不喜欢用工程文件而更喜欢手动编译, 那么也可以:
```
mcs ... ... -reference:./OpenTK.dll
```
然后就可以开始写代码了.


```CSharp
using OpenTK;
using OpenTK.Graphics;
using OpenTK.Graphics.OpenGL;
using System.Drawing;
```
OpenTK实际上自己搞了一个和 GLFW 差不多的东西, 不过使用了一些C#自己的特性(比如委托和事件), 用法大概是这样:
```CSharp
...
    GameWindow window = new GameWindow(1024,768, ...);
    window.MakeCurrent();
    window.Viewport(0, 0, 1024, 768);
    
    window.UpdateFrame += (object s, FrameEventArgs e) =>
    {
        window.ProcessEvents();
        // ...
        // 核心逻辑...
        // ...
    };
    
    window.RenderFrame += (object s, FrameEvertArgs e) =>
    {
        // ...
        // 做一些绘制工作...
        // ...
        window.SwapBuffers();
    };
    
    window.Run(60, 60);
...
```
然后你会发现一件事情, 那就是如果没有自动补全你写代码的感受和c++差不多, 但是有了自动补全之后就变得非常快~\\(≧▽≦)/~  

OpenTK 内置了一些枚举类型用来替代 C 语言的 OpenGL 那个恶心人的 GLenum. (C 语言没有枚举类型咯 ???...). 举几个例子:
```CSharp
    GL.BufferData(BufferTarget.ArrayBuffer, data.Length * sizeof(float), data, BufferUsageHint.StaticDraw);

    GL.VertexAttribPointer(0, 2, VertexAttribPointerType.Float, false, sizeof(float) * 2, 0);
    
    int vert = GL.CreateShader(ShaderType.VertexShader);
    
    GL.DrawArrays(PrimitiveType.TriangleStrip, 0, 3);
```

虽然代码变得更长了.... 但是有自动补全这显然不是什么大问题.....
