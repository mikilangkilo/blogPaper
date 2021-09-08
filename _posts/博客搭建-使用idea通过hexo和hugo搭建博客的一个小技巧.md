---
title: "博客搭建-使用idea通过hexo和hugo搭建博客的一个小技巧"
date: 2019-09-04T22:35:27+08:00
tag : "blog"
category : "博客搭建"
description : "记录一个小技巧"
---

使用hexo，或者hugo，在兴建文章的时候总是很麻烦，需要执行命令，不是hexo n 就是hugo new，而且需要切换到对应的目录，否则很容易生成到错误的地方，构建也是。

而使用idea则可以轻易的解决这个问题。

首先针对新建文章来讲，我们先写一段shell脚本

```
#!/usr/bin/env bash
read -p "Input name:" -s name
echo $name
cd ~/Desktop/hugoSite/content/ && hugo new post/$name.md
```

就是这么简单，其作用就是读取一段文字，然后切到hugo项目的对应位置，帮你创建这篇文章。

我们将这段脚本命名为createFile.sh,同时使用下面的命令给其升权

```jshelllanguage
chmod +x createFile.sh
```

这样这段脚本就可以有运行的权限了。

然后我们点击Preferences->Tools->External Tools,在这里面点击+，将自己的环境附上去

![hugo便捷方式](/images/hugo/hugo便捷脚本方式1.png)

name什么自不必讲，program是指可运行文件路径，workingdirectory则是项目路径，另外注意一定要勾选advanced中的2个选项

然后apply，之后去Preferences->Keymap 搜索你命名的脚本名，然后添加快捷键，我创建文章用的command+N

之后就可以随时command+N即可通过命令拦输入文章标题，自动创建一篇文章，在idea里面之后就可以双击sheft即可搜索文章名然后写文章了。

另外还有构建的脚本，原理同上，附录一下

```
#!/usr/bin/env bash
cd ~/Desktop/hugoSite/ && hugo && git add . && git commit -m "rebase" && git push && cd public/ && git add . && git commit -m "rebase" && git push
```

构建的快捷键我是control+command+回车。