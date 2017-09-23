---
layout: post
title: 在macOS下动态选择使用Boost的静态库和动态库版本
date: 2017-09-24
tags: [C++, Boost, macOS]
---

在macOS上使用[Boost](http://www.boost.org){:target="_blank"}，静态库和动态库版本都存在的前提下，如何动态选择使用什么版本?

### 问题与解决方案 ###

首先按照经验使用使用-static选择静态库版本，编译的时候提示crt0.o找不到，原来是macOS对于Binary不支持-static，具体参考[Technical Q&A QA1118: Statically linked binaries on Mac OS X](https://developer.apple.com/library/content/qa/qa1118/_index.html){:target="_blank"}；然后参考[Technical Q&A QA1393 Using static versions of existing dynamic libraries](https://developer.apple.com/library/content/qa/qa1393/_index.html){:target="_blank"}这篇文章，将静态库和动态库搞成两个文件夹的方式，总觉得很别扭；最后考虑了一下，觉得采用链接文件重命名的方式更优雅一些: 比如说boost_system模块，静态库和动态库版本分别为:libboost_system.a libboost_system.dylib，不要直接将他们install到/usr/local/lib目录，放在其他任意地方，然后在/usr/local/lib目录下创建对应的链接文件，并命名为：libboost_system.a libboost_system-d.dylib; 使用时候采用-lboost_system和-lboost_system-d来选择对应版本; 此外我只需要多线程版本，链接的时候为了脚本兼容，需要将文件名的mt去掉. 结合brew实现一个自动化脚本如下:

### 自动化脚本 ###

> ```bash
> #!/bin/sh
>
> # 自动安装BOOST库，只安装多线程版本，将库名改为正常方式(不加-mt)，默认使用静态库版本，使用动态库版本-d
>
> _BOOST_LIB_DIR=/usr/local/lib
>
> # clear
> #====================
> brew uninstall boost boost-python boost-mpi
> rm $_BOOST_LIB_DIR/libboost*.*
>
> # install
> #====================
> # boost 如果使用预编译包的话，诸如fiber等没有
> brew install boost --c++11 --with-icu4c --without-single
> # python mpi 如果添加--c++11选项的话，会要求全部编译boost，这里直接安装预编译包
> brew install boost-python
> brew install boost-mpi
>
> # rename linked file
> #====================
> for file in `ls $_BOOST_LIB_DIR/libboost*.*`
> do
>   if [[ !($file == *-mt.*) ]]; then
>     echo 'rm ' $file
>     rm $file
>   fi
> done
>
> for file in `ls $_BOOST_LIB_DIR/libboost*.*`
> do
>   _DEST=`echo $file | sed 's/-mt//'`
>   if [ ${file##*.} == 'dylib' ]; then
>     _DEST=`echo $_DEST | sed 's/.dylib/-d.dylib/'`
>   fi
>   echo 'mv ' $file $_DEST
>   mv $file $_DEST
> done
>
> ```

### 其他 ###

关于-static链接生成Binary，macOS其实是支持的，具体可以参考[这篇文章](https://oao.moe/archives/937/#fn:1){:target="_blank"}.

<br/>

> [原始链接]({{page.url}}) 版权声明：自由转载-非商用-非衍生-保持署名 \| [Creative Commons BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh)
