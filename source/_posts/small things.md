---
title: '小记, 一些琐碎的东西'
date: 2017-12-4 23:51:12
tags:
---

记录一些非常重要, 但是并不值得开一篇文章大谈特谈的坑.

<!-- more -->

1. github page 的repo名称如果不设置为 [your_name].github.io 会被认为是项目 page , 会使用 Jekyll 来组织页面. 然后 hexo 推上去就会报错. mdzz.

1. 如果你配置 vscode-OmniSharp 发现工程文件无法读取, 那可能是你的电脑安装的 OmniSharp 版本太老或者有 Bug . 我用 Ubuntu 16.04 LTS 上的 apt-get 下载安装的 OmniSharp 恰好是那个出问题的版本(啊这群维护repo的在想什么). 去官网下载一个新的, 直接make, 就能解决问题了.

1. hexo 文章在首页的截断符是`<!-- more -->`, 注意空格, 注意空格, 注意空格.

1. C# 的迭代器是左开右闭. C++ 是左闭右开. 

1. MSBuild 里的通配符 `**` 表示部分路径, `*` 表示零个或多个字符. 想要匹配目录 `Project` 及其子目录下的所有 `.cs` 文件, 可以这么写 `Project/**/*.cs`. [参见这个网页.](https://stackoverflow.com/questions/8532929/two-asterisks-in-file-path) 

1. 写 csproj 时用 HintPath 指定目录中的内容不要换行, 否则会无法识别. 如果写成这样能识别通常是因为在输出目录(OutputPath)里找到了对应的dll文件.