---
layout: post
title: 创建macOS High Sierra安装盘
date: 2017-09-29
tags: [macOS, 教程]
---

今天晚上正好有时间，想把系统更新到macOS High Sierra，从商店下载完成后，习惯性的做一个安装盘备用，参考之前的命令行，修改运行:
> ```bash
> sudo /Applications/Install\ macOS\ High\ Sierra.app/Contents/Resources/createinstallmedia --volume /Volumes/I --applicationpath /Applications/Install\ macOS\ High\ Sierra.app —nointeraction
> ```

提示：/Applications/Install macOS High Sierra.app does not appear to be a valid OS installer application.

Google了一下，说什么的都有；
还是先参考知识库文章: [Create a bootable installer for macOS](https://support.apple.com/en-us/HT201372){:target="_blank"}，发现High Sierra不用加--applicationpath参数，删除后运行如下命令：
> ```bash
> sudo /Applications/Install\ macOS\ High\ Sierra.app/Contents/Resources/createinstallmedia --volume /Volumes/I —nointeraction
> ```

还是一样的问题

最后在交流区发现[Install macOS High Sierra.app does not appear to be a valid OS installer application](https://origin-discussions2-us-dr.apple.com/thread/8085073)，又是文件下载错误问题，需要保证安装文件大小为：***5.18G***，重复多次下载后终于下到该大小的安装包，运行命令，问题解决！

<br/>

如果之前没做个安装盘，请遵循以下步骤：

> 1. 从商店下载
> 2. 如果安装器运行，记得退出安装
> 3. 插入U盘或者移动硬盘(请记得分区)，安装分区需要至少有12G空间，我一般预留16G，使用磁盘工具格式化为:MacOS扩展(日志式)格式
> 4. 运行上面的命令(上面两个命令都可以，需要注意将--volume后的参数改为你自己在3.中格式化的盘符名，指定错误毁数据，谨慎操作)

<br/>

> [原始链接]({{page.url}}) 版权声明：自由转载-非商用-非衍生-保持署名 \| [Creative Commons BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh)
