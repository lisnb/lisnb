title: 为Hexo添加了置顶功能
date: 2014-11-01 20:51:24
tags:
- hexo
- pacman

categories: hexo

---

通过修改主题，为Hexo添加了置顶功能。
<!--more-->
是在[Pacman](https://github.com/A-limon/pacman)基础上做的修改，然后在主题的配置文件中标明要把哪一篇置顶。其实在Hexo中设置应该也可以。
但因为是在主题中做的置顶，所以就放在主题的配置文件中设置了。
可以设置一篇或者多篇，在`_config.yml`中：

    stickes: #直接在这里写，是添加一篇
    - foo #写在这就可以添加多篇
    - bar
    
修改后的主题现在叫做`Pacwoman`

Fork it on Github: [Pacwoman](https://github.com/lisnb/pacwoman)。