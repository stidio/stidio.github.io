---
layout: post
title: 在macOS下使用Visual Studio Code进行C/C++开发
date: 2017-01-23
tags: [C++, macOS, 教程, 工具]
---

平时工作做C/C++方面的开发更多还是在Windows下使用Visual Studio，几乎都是做跨平台开发，Windows上跑通后Linux上编译发布，基本上都没什么问题，如果的确需要调试，有VisualGDB的存在也很方便；但在macOS下使用过IDE性质的XCode,CLion,Qt Creator，也使用过轻量级的诸如TextMate, Sublime Text，但始终找不到Visual Studio的感觉，回想起来macOS下使用最多的C/C++开发环境居然是CodeRunner；直到最近这几天有空折腾了一下Vistual Studio Code，瞬间找到了初恋的感觉!

### 需求 ###

粗略想了下，进行C/C++开发，我需要的功能大概有：

1. 格式化
2. 自动完成
3. Lint
4. 符号检索
5. 方便的跳转和查看
6. 可视化调试(别给我提GDB，你能苛求一个连VIM都讨厌的人使用GDB?)

而上面这些功能使用Visual Studio Code和必要的插件几乎可以达到Vistual Studio 80%的体验

### 打造 ###

1. 安装[Visual Studio](http://code.visualstudio.com){:target="_blank"}
2. 安装[cpptools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools){:target="_blank"}插件，这个Micrsoft出品的插件几乎囊括了我前面说的所有功能，但他的自动完成是Fuzzy的，非常糟糕；另外他的Lint功能也进行只局限于文件包含；*注：该插件安装完成后会自动安装必要的依赖更新*.
3. 安装[C/C++ Clang](https://marketplace.visualstudio.com/items?itemName=mitaki28.vscode-clang){:target="_blank"}插件，这个插件只有两个功能，**自动完成**和**诊断Diagnostic(Lint)**，其使用Clang实时分析，什么模板嵌套都能分析，功能异常强大；
4. 上面两个如果同时启用的话会发生冲突，并且[C/C++ Clang]插件默认不进行C++11分析，点击[Code]->[首选项]->[用户设置]进行如下配置:

    > ```json
    > "C_Cpp.autocomplete": "Disabled",
    > "clang.cxxflags": ["-std=c++11"]
    > ```

> 注：仅仅需要上面两个插件就够了，高安装量的[C++ Intellisense](https://marketplace.visualstudio.com/items?itemName=austin.code-gnu-global)插件千万别装，它会和上述两个插件冲突，从而出现各种奇怪问题；

### 体验 ###

1. 打开一个包含有C/C++文件的目录；
2. 使用[F1]或[⇧⌘P]打开命令模式，选择[C/Cpp: Edit Configurations]命令，其会在目录的.vscode配置目录下生成一个c\_cpp\_properties.json文件，修改Mac节点下的includePath变量添加C++11跳转支持：

    > ```json
    > "includePath": ["/usr/include", "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1"],
    > ```
    > *注：如果引入其他库，将库路径按以上方式附在后面即可*

3. 打开命令模式，选择[Tasks: Configure Task Runner]命令，其会在目录的.vscode配置目录下生成一个tasks.json文件，修改配置：

    > ```json
    > {
    >     // See https://go.microsoft.com/fwlink/?LinkId=733558
    >     // for the documentation about the tasks.json format
    >     "version": "0.1.0",
    >     "command": "clang++",
    >     "isShellCommand": true,
    >     "args": ["main.cpp", "-std=c++11", "-g"],
    >     "showOutput": "always"
    > }
    > ```
    > *注：main.cpp是入口文件，你可以修改成你自己的，如果是有多个入口的测试项目，我一般定义为${file}，只是必须选定文件启动；另外args中必须添加-g选项，否则调试无效；要支持C++11，必须添加-std=c++11选项；如果还有其他编译要求，将选项附加在后面即可(调试模式我经常使用–save-temps来查看编译中间文件)；之后[⌘P]然后执行[task clang++]，或者直接[⇧⌘B]就可自动编译*.

4. 打开命令模式，选择[Debug: Open launch.json]命令，其会在目录的.vscode配置目录下生成一个launch.json文件；修改program为:${workspaceRoot}/a.out；如果需要读取参数，修改args配置；我一般会习惯性的加上"preLaunchTask": "clang++"配置，这样当源代码发生改变时，启动调试会自动编译；
5. 之后的调试流程，基本上就和Vistual Studio一样了，附一张调试图:

![](/assets/use_vscode_for_c_c++_development_in_macos/01.png)


### 参考资料 ###

[C/C++ for VS Code (Preview)](http://code.visualstudio.com/docs/languages/cpp#_getting-started){:target="_blank"}

<br/>

> [原始链接]({{page.url}}) 版权声明：自由转载-非商用-非衍生-保持署名 \| [Creative Commons BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh)
