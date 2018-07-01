---
title: Linux/Mac命令行翻墙
date: 2018-07-01 23:45:30
tags: [Shadowsocks, Tutorial]
---

之前翻墙的方式比较2——用`ShadowSocks GUI`做代理客户端，然后将`Network Manager`设置为`Proxy`模式，通过手动切换`ShadowSocks GUI`的代理模式来决定是否使用代理。这样配置起来很方便，但说实话，用起来还是挺难受的。主要有这么几点原因：

1. 暑期实习让我养成了禁用`Network Manager`的习惯，直接配置`/etc/network/interfaces`或`/etc/sysconfig/network-scripts/ifcfg-xxx`更加直观方便。
2. 每次切换代理模式都要先打开`ShadowSocks GUI`的设置页面，然后点击选择想要的模式，这样一来还是挺麻烦的，并且会干扰工作状态。
3. `PAC`名单的设置非常麻烦，需要自己打开文本，然后手动添加一行规则。试想，一个网站页面打开失败，可能是因为获取脚本没有回应，这时候你需要找到所有失败的请求，将对应的host添加到名单中，这无疑是一个超级麻烦的工作。
4. 最关键的是，这种翻墙方式只能在桌面环境下使用。桌面环境对服务器来说只是浪费资源，而且远程shell也无法操作桌面。最直观的例子就是，aliyun主机无法使用这种方式翻墙。

综合以上几点原因，我决定还是花点时间，重新配置一下翻墙方式。网上查询一些资料后，个人感觉`ShadowSocks`+`Privoxy`+`SwichyOmega`最适合我：兼容shell翻墙，不依赖于桌面环境，而且操作简单，唯一的缺点就是配置起来稍微麻烦一点。试用了一段时间，感觉确实不错，于是一口气在我所有的机器（`CentOS 7.2`, `Linux Mint 17.2`, `Mac OS 10.11.6`, `Debian 8.6`）上配置了代理。

网上有好几篇文章介绍这种翻墙方式的配置（关键词搜索`命令行翻墙`、`Privoxy`即可），我也是参照它们的步骤来实现的。但是感觉它们都不是很完善，因为我在配置的时候还是遇到了不少问题。因此，我还是决定将自己的配置过程贴出来，如果有需求的人正好翻到了我这篇博文，也算是做了一点小贡献；另外，自己以后配置新机器也会更加方便。

---

## 配置ShadowSocks

`ShadowSocks`分为server端与client端，缺一不可。我的server端部署在`Bandwagon`的vps上，因此，还需要在自己机器上配置client 端。

1. 安装`python`、`pip`，这里可以直接从源安装，Mac好像是自带？

   ```shell
   sudo apt install python-pip # for Debian
   sudo yum install python-pip # for RedHat
   ```

2. 安装`ShadowSocks`：

   ```shell
   pip install shadowsocks
   ```

3. 创建`ShadowSocks`启动配置文件`/etc/shadowsocks.json`：

   ```json
   {
       "server": "server_ip_addr",
       "server_port": 443,
       "local_address": "127.0.0.1",
       "local_port": 1080,
       "password": "server_ss_password",
       "method": "aes-256-cfb"
   }
   ```

   各个字段依照server端的配置来填写。

4. 从上述配置文件启动client端：

   ```shell
   sslocal -c /etc/shadowsocks.json
   ```

   此时，`ShadowSocks`将监听本地的1080端口，将收到请求转发到server端的443端口，从而实现代理的功能。

但这样还不能实现翻墙，因为`ShadowSocks`使用的是`SOCKS5`协议，只支持`SOCKS5`代理，不支持`HTTP`、`HTTPS`代理，下面我们继续配置一个可以将`SOCKS5`代理转换为其他协议代理的工具——`Privoxy`。

## 配置Privoxy

`Privoxy`

1. 安装`Privoxy`，这里不同操作系统的安装方式不一样。

   Linux是最方便的，直接从源安装即可：

   ```shell
   sudo apt install privoxy # for Debian
   sudo yum install privoxy # for RedHat
   ```

   Mac OS可以从官网直接下载安装包（[link](http://www.privoxy.org/sf-download-mirror/Macintosh%20%28OS%20X%29/3.0.26%20%28stable%29/Privoxy%203.0.26%2064%20bit.pkg)），如果无法访问官网，也可以从我这儿下载`3.0.26 (stable)`版（[link](http://www.xiaoan.org/share/privoxy_3.0.26_64bit.pkg)）

2. 修改配置文件。

   对于Linux，配置文件的路径一般为`/etc/privoxy/config`，而Mac OS上路径稍微有些不同，为`/usr/local/etc/privoxy/config`。如果你找不到配置文件，建议直接全局搜索关键词`privoxy`+`config`。

   在配置文件中找到以下两行，进行修改。

   ```shell
   forward-socks5   /               127.0.0.1:1080 . # 修改为ShadowSocks配置中的local_port
   listen-address  localhost:8118
   ```

3. 启动`Privoxy`：

   ```shell
   sudo privoxy /etc/privoxy/config			# for Linux
   sudo /Application/Privoxy/startPrivoxy.sh	# for Mac OS
   ```

现在`Privoxy`已经在后台运行，监听8118端口，将请求转发到1080端口，实现`HTTP`、`HTTPS`代理与`SOCKS5`代理的转换。

## 一些零碎的配置

安装完`ShadowSocks`与`Privoxy`后，我们已经可以命令行翻墙了。

```shell
sslocal -c /etc/shadowsocks.json 1>/dev/null 2>&1 & # 后台运行ShadowSocks Client
sudo privoxy /etc/privoxy/config					# 开启privoxy服务
# sudo /Application/Privoxy/startPrivoxy.sh for MacOS
export http_proxy=http://localhost:8118				# 设置HTTP代理地址
```

效果如下图所示：

![ss+privoxy](shadowsocks-privoxy.png)

可以看到，本机的外网ip发生了变化，这说明代理配置成功，此时也可以试一试`wget www.google.com`。

如果我们不想用代理该怎么办呢？很简单，直接把变量`http_proxy`给注销掉（`unset http_proxy`）即可。不过，这样的操作不够优雅，用户并不会关心（甚至讨厌）配置细节，他们要的仅仅是`开启代理`、`关闭代理`两个操作而已。而且注意到`ShadowSocks`一直运行在后台，一旦这个shell被关闭，那么`ShadowSocks`进程也会被杀死，其他shell想用代理时只能重新运行`ShadowSocks`，这太糟糕了。

为了解决以上两个问题，我们可以做一下小配置。

1. 使用`nohup`让`ShadowSocks`一直运行：

   ```shell
   nohup sslocal -c /etc/shadowsocks.json 1>/dev/null 2>&1 &
   ```

2. 将所有命令操作封装一下。

   在主目录下新建脚本文件`ss.sh`：

   ```shell
   #! /bin/bash
   nohup sslocal -c /etc/shadowsocks.json >/dev/null 2>&1 &
   sudo privoxy /etc/privoxy/config # only for Linux
   # for Mac OS, sudo /Application/Privoxy/startPrivoxy.sh
   ```

   在当前shell的profile（`.bashrc`、`.zshrc`等等，也可以写在`/etc/profile`）里，实现两个函数，分别用于开启代理与关闭代理：

   ```shell
   function sson() {
       export http_proxy=http://localhost:8118;
       export https_proxy=http://localhost:8118;
       export ftp_proxy=http://localhost:8118;
   }
   function ssoff() {
       unset http_proxy https_proxy ftp_proxy
   }
   ```

   现在，你可以运行一次脚本`ss.sh`，以后想用代理时，直接命令行输入命令`sson`即可，同理，关闭代理使用`ssoff`。如果你想实现开机自动运行`ShadowSocks`和`Privoxy`（即开机运行`ss.sh`脚本），直接在`/etc/rc.local`中添加运行`ss.sh`的命令即可。

## 配置Chrome

最后，我还需要为浏览器配置代理，这步就比较简单了，全部鼠标操作。

1. 下载并安装插件`SwitchyOmega`，由于未翻墙不能访问chrome webstore，建议从github（[link](https://github.com/FelisCatus/SwitchyOmega/releases/download/v2.3.21/SwitchyOmega.crx)）下载，我这里提供`2.3.21`版本的下载（[link](http://www.xiaoan.org/share/SwitchyOmega.crx)）。

2. 进入`SwitchyOmega`的`options`（选项）界面，在侧边栏中选择`proxy`（代理），按照下图填写：

   ![proxy](proxy.png)

3. 在侧边栏中选择`auto switch`（自动切换），按照下图填写：

   ![auto-switch](auto-switch.png)

   注意，填写完`Rule List URL`后，需要点击下方的`Download Profile Now`下载翻墙名单`gfwlist.txt`。

4. 填写完后，点击`Apply changes`（确认修改）。

现在可以使用`Chrome`翻墙啦。将插件设置为`auto switch`模式，此时，名单`gfwlist.list`内的所有网址会自动走代理，而不在名单的网址不走代理。

回顾最开始我提出来的四个问题，现在好像还剩一个没有解决——`PAC`列表修改起来很麻烦。其实不然，`SwitchyOmega`为我们提供了一个很优雅的操作方式。一旦有请求失败，`SwitchyOmega`会将它们记录下来，此时，我们可以为这些失败的请求设置规则：

![add condition](add-condition.png)

如上所示，直接点击`Add condition`即可将`*.atdmt.com`设置为`proxy`模式，以后每次访问以`.atdmt.com`结尾的网址时，浏览器都会自动走代理。终于，我们不需要自己手动设置`PAC`名单啦。

---

以上就是我正在使用的，也是自己非常喜欢的一种翻墙方式，有翻墙需求的朋友不妨尝试一下。
