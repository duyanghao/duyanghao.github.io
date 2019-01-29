---
layout: post
title: IDE Summary
date: 2016-12-29 18:10:31
category: 技术
tags: Languages Tools
excerpt: IDE Summary during work……
---

This article gives a Summary of Tools usage during work(only for having a good view of project)……

Now, i will introduce my IDE choice and basic usage for each of Language.   

# c/c++ - [Source Insight(3.50.0076)](http://www.sourceinsight.com/update.html)

### create project

* 1.create project
 
![](/public/img/Tool_summary/1.png)

* 2.new project

![](/public/img/Tool_summary/2.png)

* 3.new project settings

![](/public/img/Tool_summary/3.png)

* 4.add project files

![](/public/img/Tool_summary/4.png)

* 5.close

![](/public/img/Tool_summary/5.png)

* 6.font and filter files(alt+T)

![](/public/img/Tool_summary/6.png)

* 7.color

![](/public/img/Tool_summary/7.png)

* 8.view

![](/public/img/Tool_summary/si_view.png)

### keyboard shortcuts

* find(file scope) -> ctrl+f
* find(project scope) -> ctrl+shift+f
* find pre -> F3
* find next -> F4 
* jump to definition -> ctrl+= or ctrl+mouse
* mark -> shift+F8

# script(bash,perl) - [sublime_text](https://www.sublimetext.com/)

### [Package Control Install](https://packagecontrol.io/packages/ModernPerl)

>1.Install this package with **Package Control** (or otherwise).

>2.In Sublime, use View > Syntax > Open all with current extension as… to reopen all current Perl files with ModernPerl.

>3.Fresh Perl files should automatically open with ModernPerl, while Perl files that have previously been opened with Sublime will tend to keep the syntax they were last opened with.
To check this, open a Perl file that has never been opened with Sublime before (create a new .pl file if necessary), and check that it opens with ModernPerl.

>4.If fresh files do not open with ModernPerl, use Preferences > Settings – Syntax Specific – User on a file opened with ModernPerl to open up ModernPerl.sublime-settings and put the following into it:
{ "extensions": ["pl", "PL", "pm", "pod", "t"] }

>5.Whenever you open a Perl file that has previously been opened with Sublime, check which syntax it opens with, and manually switch it to ModernPerl if necessary.

### open project

File->Open Folder...(directly open project directory)

### keyboard shortcuts

* find(file scope) -> ctrl+f
* find(project scope) -> ctrl+shift+f

## python - [pycharm](https://www.jetbrains.com/pycharm/download/#section=windows)

### python Language Install

please install [python Language](https://www.python.org/downloads/) before install pycharm.

### python interpreter configuration(File->Settings)

![](/public/img/Tool_summary/8.png)

### color theme

![](/public/img/Tool_summary/9.png)

### font

![](/public/img/Tool_summary/12.png)

### open project

File->Open...(directly open project directory)

### keyboard shortcuts

* find(file scope) -> ctrl+f
* find(project scope) -> ctrl+shift+f
* jump to definition -> ctrl+b

# markdown,go - [VSCode](http://code.visualstudio.com/docs/setup/windows#_installation)

### Golang Install

please install [Go Language](https://golang.org/dl/) before install VSCode.

**attention**:

* setting GOPATH and PATH environment variable

### Git Install

please install [Git](https://git-scm.com/downloads) for `go get` cmd before install VSCode.

### [VSCode Install](http://code.visualstudio.com/docs/setup/windows#_installation)

### proxy configuration

if you are in the company,you should set the http and https proxy as below:

文件-》首选项-》用户设置

>
```
{
    "http.proxy": "http://proxy:port",
    "https.proxy": "http://proxy:port",
    "http.proxyStrictSSL": false
}
```

### [vscode-go](https://github.com/Microsoft/vscode-go) Install

follow these steps to install vscode-go after install VSCode.

* 1.ctrl+shift+p(please input `ext install`)

![](/public/img/Tool_summary/10.png)

* 2.choose vscode-go(with most downloads)
* 3.install Tools

1) create a directory for Tools(suppose to be "D://Go_project/Vs_code_project").

2) enter `cmd`

3) set GOPATH to the above directory path(suppose to be "D://Go_project/Vs_code_project")

4) set http and https proxy environment variable if necessary

5) install Tools as below:

>
```
go get -u -v github.com/nsf/gocode
go get -u -v github.com/rogpeppe/godef
go get -u -v github.com/zmb3/gogetdoc
go get -u -v github.com/golang/lint/golint
go get -u -v github.com/lukehoban/go-outline
go get -u -v sourcegraph.com/sqs/goreturns
go get -u -v golang.org/x/tools/cmd/gorename
go get -u -v github.com/tpng/gopkgs
go get -u -v github.com/newhook/go-symbols
go get -u -v golang.org/x/tools/cmd/guru
go get -u -v github.com/cweill/gotests/...
```

### vscode-go configuration

文件-》首选项-》用户设置

>
```
{
    //go
    "go.buildOnSave": true,
    "go.lintOnSave": true,
    "go.vetOnSave": true,
    "go.buildTags": "",
    "go.buildFlags": [],
    "go.lintTool": "golint",
    "go.lintFlags": [],
    "go.vetFlags": [],
    "go.coverOnSave": false,
    "go.useCodeSnippetsOnFunctionSuggest": false,
    "go.formatOnSave": true, 
    "go.formatTool": "goreturns",
    "go.formatFlags": [],
    "go.gocodeAutoBuild": false,
    "go.goroot": "D://Go_dir",
    //editor
    "workbench.editor.enablePreview" : false
}
```

### open project

* 1.文件-》首选项-》工作区设置

>
```
{
    "go.gopath": "`your_project_GOPATH`;D://Go_project/Vs_code_project"
}
```

* 2.文件-》打开文件夹

### keyboard shortcuts

* find(file scope) -> ctrl+f
* find(project scope) -> ctrl+shift+f
* 选中按TAB右移，按SHIFT+TAB左移
* 后退 -> alt + left arrow
* 前进 -> alt + right arrow
* Find next/previous -> F3 / Shift + F3

# scala - [Intellij IDEA](https://www.jetbrains.com/idea/download/)

### open scala project(eg:[spark](https://github.com/apache/spark))

File->Open…(directly open project directory)

![](/public/img/Tool_summary/11.png)

### keyboard shortcuts

* find(file scope) -> ctrl+f
* find(project scope) -> ctrl+shift+f
* jump to definition -> ctrl+b

# Addition

* `java`——[eclipse](https://www.eclipse.org/)
* `常用编辑`——[Notepad++](https://notepad-plus-plus.org/) + [sublime text](https://www.sublimetext.com/)
* `c/c++/c#`——[Microsoft Visual Studio](https://www.visualstudio.com/zh-hans/vs/)
* `手游`——[Unity](https://unity3d.com/cn)
* 记笔记——[Evernote](https://evernote.com/intl/zh-cn)
* 抓包——[wireshark](https://www.wireshark.org/download.html)
* 文本比较——[Beyond Compare](https://www.scootersoftware.com/features.php?zz=features_multifaceted)
* 制作图像素材——[Photoshop](https://creative.adobe.com/products/download/photoshop?store_code=hk_en)
* 画流程图——[亿图图示专家](http://www.edrawsoft.cn/)
* 思维导图——[XMind](https://www.xmind.cn/)
* 计算器——系统键+R -> calc
* 谷歌翻译插件——更多工具 -> 扩展程序 -> Google 翻译

# Refs

* [VS Code 快捷键（中英文对照版）](https://segmentfault.com/a/1190000007688656)