---
title: 手贱
date: 2016-08-08 22:42:48
tags: 
- git
- config
categories: git
---

手贱。
前几天乱翻wiki的时候看到git一节，里面说，可以通过设置`color.ui=always`把输出弄的美美的。
<!-- more-->
提交代码是用一个组里人写的python脚本，然后今天晚上提交代码的时候，就怎么都提交不了，显示`git diff`没有内容，但我明明有内容。
于是我就去那个upload.py的脚本里去看，里面是parse `git diff` 命令的输出，然后找到你做更改的地方，我就打印出来，发现每一行前后都有一些什么`\x1bm[`什么的，看起来是跟终端输出的时候文字的颜色有关，我以为是我.bashrc里修改了PS1导致的，所以就改回去，发现还不行。
后来尝试在那个脚本里修改一下，把每行的前后都strip掉刚刚那个字符串，可以提交了，但是提交的内容看不了，所以可能diff出来的每行前后不都是那个字符串。况且我的.bashrc已经返璞归真了，应该跟我的环境没有关系了。
无意识的`git config --list`了一下，发现了`color.ui=always`，恍然大悟，`git config --global color.ui no`，然后就好了。白白搞了一个多小时。
手贱。

--------
今天简直糟透了，不过还是要恭喜标宝~ 为你感到开心~ 
