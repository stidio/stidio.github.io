---
layout: post
title: 解决因macOS High Sierra更新Ruby到2.3造成的Jekyll不能启动的问题
date: 2017-12-28
tags: [macOS, Ruby, Jekyll, 教程]
---

这几天有兴致研究了一些东西想更新下博客，但发现Jekyll不能运行，把过程中踩到的一些坑记录下，以便他人：

首先提示版本错误：
> zsh: /usr/local/bin/jekyll: bad interpreter: /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin: no such file or directory

运行 **ruby --version** 查看系统ruby版本已经升级到了2.3.3，运行命令更新jekyll和bundler，因为新版macOS */usr/bin*不能写，需要手动指定安装到*/usr/local/bin*目录
> ```bash
> sudo gem install -n /usr/local/bin jekyll bundler
> ```

<br/>

升级后，运行**jekyll serve**，又提示错误:
> /Library/Ruby/Site/2.3.0/bundler/spec_set.rb:88:in \`block in materialize': Could not find public_suffix-2.0.4 \> in any of the sources (Bundler::GemNotFound)  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler/spec_set.rb:82:in \`map!'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler/spec_set.rb:82:in \`materialize'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler/definition.rb:170:in \`specs'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler/definition.rb:237:in \`specs_for'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler/definition.rb:226:in \`requested_specs'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler/runtime.rb:108:in \`block in definition_method'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler/runtime.rb:20:in \`setup'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Site/2.3.0/bundler.rb:107:in \`setup'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Gems/2.3.0/gems/jekyll-3.6.2/lib/jekyll/plugin_manager.rb:50:in \`require_from_bundler'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /Library/Ruby/Gems/2.3.0/gems/jekyll-3.6.2/exe/jekyll:11:in \`<top (required)>'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /usr/local/bin/jekyll:23:in \`load'  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; from /usr/local/bin/jekyll:23:in \`<main>'  

Google了一下没找到答案，看了下代码，觉得应该是_config.yml相关依赖库没升级造成的问题，运行命令升级依赖库：
> ```bash
> bundle update
> ```

<br/>

再次运行，能启动起来了，但有个警告：
> Deprecation: The 'gems' configuration option has been renamed to 'plugins'. Please update your config file accordingly.

打开_config.yml将gems修改为plugins，问题解决

<br/>

> [原始链接]({{page.url}}) 版权声明：自由转载-非商用-非衍生-保持署名 \| [Creative Commons BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh)
