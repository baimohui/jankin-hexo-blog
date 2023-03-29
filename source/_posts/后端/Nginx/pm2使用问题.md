---
title: 解决 pm2 command not found 问题
categories: 
- Nginx
tags:
- Nginx
- Linux
- Nuxt
---

在 Linux 环境通过 npm 全局安装 pm2 后，运行 pm2 命令却显示没有找到该命令：-bash: pm2: command not found。上网查找资料发现，这是因为没有创建软连接导致的。<!--more-->软连接类似 windows 系统的快捷方式，里面存放的是源文件的路径，指向源文件。可以通过下面语句来创建软连接。

```shell
ln -s /data/services/nodejs/lib/node_modules/pm2/bin/pm2 /usr/local/bin
```

bin 是 binary 的缩写，即计算机可直接运行的字节码，对应 windows 系统的 exe 文件。一般都将可执行文件放到 bin 目录中。

其中 `/data/services/nodejs/lib/node_modules/pm2/bin/pm2` 目录是我的 pm2 全局安装的位置。而 `/usr/local/bin` 目录是给用户放置可执行程序的地方（放在这里，不会被系统升级而覆盖同名文件）。

但运行这条命令后出现如下错误。

```
ln: failed to create symbolic link 'usr/local/bin/pm2': File exists
```

这是说软连接已存在，不能再创建。进入 usr/local/bin 目录，将 pm2 链接直接删除，再次执行语句即可。

但问题到这里并没有结束，当我再次运行 pm2 命令时，出现了如下的报错信息。

![image-20210702085100381](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20210702085100.png)

在 stack overflow 网上查到可以通过 `npm i request` 安装所缺失的模块。然而这样做后再运行，仍然出现上述错误。无奈之下，我选择重装 pm2。但通过 `npm i pm2 -g` 重新全局安装 pm2 时，却发生报错。

![image-20210702083153019](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20210702083200.png)

这里说 pm2 文件已经存在，要想全局安装，要么把已存在的 pm2 文件删掉，要么安装时带上 --force 强制重写原文件。然而上图中可以看到我是已经带上 --force 了。

看来只能先将 pm2 卸载，可是问题并没有这么简单。通过 `npm uninstall pm2 -g` 来卸载，会出现和安装时一样的错误。而当我直接进入 pm2 安装目录下删除文件，出现了弹窗报错。

![image-20210702090446751](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20210702090447.png)

心力交瘁之下，我想了一个土方法：既然卸不掉你，那么就把你名字改了。于是我把 pm2 文件夹的名字随意改成了 pm22。在重新连接服务器后，全局安装 pm2。

![image-20210702091101341](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20210702091101.png)

再次运行 pm2 命令，终于成功运行！

![image-20210702091504131](https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20210702091504.png)