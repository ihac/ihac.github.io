---
title: 使用John the Ripper破解密码
date: 2016-06-01 10:47:39
tags: [Security, Password, John the Ripper]
---

　　假设这么一个场景：你和几个同学在一间教室里上课，大家都通过校园wifi上网，你想获取他们的~~教务网~~密码，于是你开始抓包，然后故意找机会让那几位同学尝试登陆~~教务网~~(这里就是社工了)，最后你根据抓到的数据包开始尝试破解登陆密码。

> Start from here

　　假设这是你抓到的数据包: [password.pcap](http://xiaoan.org/share/password.pcap)。我们知道，即便只是一次普普通通的教务网登陆过程，其中也会涉及到很多次的信息交互，比如TCP三次握手、四次挥手，HTTP各种请求与响应等等，所以我们抓到的包中有很多是无用的，我们真正关心的是那些携带了密码的数据包，那么怎么才能找到这些包呢？
　　这里涉及到HTTP协议的一些细节，我们知道HTTP有两种认证方式，分别为基本认证(Basic Authentication)和摘要认证(Digest Authentication)，具体的细节这里不展开，我们只需要知道一点，无论采用哪种认证方式，HTTP请求包头部都会带有一个`Authorization`的字段，对应的值即为`用户名+盐+密码`经过hash后的字符串。
　　根据这一点特征，我们可以利用`Wireshark`分析抓到的数据包(password.pcap)，然后在filter里填入一个简简单单的规则`http contains Authorization`，即可过滤出真正带有认证信息的数据包：

![Selection_110.png-88kB][1]

　　可以看到，仅有十个数据包通过筛选，并且从HTTP请求头部可以看出，用户身份认证采用的是基本认证的方式，而基本认证无法向传输的Credentials提供`有意义`的安全保护，仅仅是将其进行Base64编码。我们可以直接提取到用户密码信息，比如对于上图显示的HTTP认证包，用户Credentials即为`user1:$1$foo$xxxxxxx`，这里user1表示用户名，\$1表示hash类型为MD5，$foo表示采用foo作为hash的盐，最后一串字符即为密码的hash结果。我们可以将十个数据包里的所有Credentials全部提取下来，存入一个文件中，然后开始对密码进行破解。
　　这里我们使用[John the Ripper](http://xiaoan.org/share/john-1.8.0.tar.xz)对密码进行破解，John是一个非常强大的工具，这里不展开，直接在实践过程中介绍其用法。
　　下载John的源码压缩包之后，将其解压，然后根据README文档的指导直接编译安装即可。安装完成后，进入`run`目录，注意到该目录下有这么几个文件：john、password.lst、john.conf，其中john不用说，就是我们用来破解密码的可执行程序，password.lst是官方默认的一个密码字典，里面存放了3000多个简单密码，john.conf顾名思义就是破解密码时所用到的规则信息。
　　建议大家在真正使用John之前，先看一下doc目录下的各个文档，尤其是`EXAMPLES`和`RULES`两个文档，这两篇文档详细介绍了John的各种使用方法以及所有规则的含义。
　　将前面抓到的10条Credentials保存为文件`allpasswd`，首先，我们尝试最简单的破解方法：
``` bash
~/Documents/*** ⌚ 20:36:24
$ ./john --single allpasswd
Loaded 10 password hashes with no different salts (md5crypt [MD5 32/64 X2])
Press 'q' or Ctrl-C to abort, almost any other key for status
user1            (user1)
1g 0:00:00:00 100% 1.612g/s 11998p/s 11998c/s 108001C/s user21900
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
　　注意到，我们成功破解出了用户user1的密码，即为user1！这个是怎么破解出来的呢？其实我们这里是用到了John一个最基本的模式——Single Crack，根据官方文档的说法(It will use the login names, "GECOS" / "Full Name" fields, and users' home directory names as candidate passwords, also with a large set of mangling rules applied)，我们知道在这种模式下，John直接将用户名等账户信息作为候选密码来进行验证，同时也支持一些mangling rules，即我们能够通过自己设计规则，来生成更多的候选密码。举个例子，假设有这么一条规则：在输入字符串的末尾添加一个字母或数字，作为新的候选密码，那么对于用户user1，根据这么一个规则，我们知道user1、user1a、user1D、user19都将成为候选密码。总的来说，Single Crack这种模式速度非常快，因为它的输入不多，仅有几条有限的用户信息，当然，相对应地，由于它的输入局限于用户账户信息，这也导致它很难破解一些与用户名完全不同的密码。最明显的，上面的实践中，我们提供了10条用户Credentials，但John只破解出了user1的密码。
　　既然Single Crack模式无法破解所有密码，那么我们尝试使用官方自带的字典：
``` bash
~/Documents/*** ⌚ 20:36:29
$ ./john --wordlist=password.lst allpasswd
Loaded 10 password hashes with no different salts (md5crypt [MD5 32/64 X2])
Remaining 9 password hashes with no different salts
Press 'q' or Ctrl-C to abort, almost any other key for status
smile            (user3)
admin            (user2)
2g 0:00:00:00 100% 5.882g/s 10429p/s 10429c/s 81617C/s notused..sss
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
　　我们又破解出了两个新密码！这次我们使用的是Wordlist模式，在这种模式下，John会打开我们指定的密码字典文件，将里面的所有字符串作为候选密码，一个个尝试去验证。所以说，一旦某个用户的密码在password.lst中出现了，那么这种模式就能很快地将其破解出来：
``` bash
~/Documents/*** ⌚ 20:57:05
$ grep -n '^smile$' password.lst
104:smile

~/Documents/*** ⌚ 20:57:20
$ grep -n '^admin$' password.lst
2823:admin
```
　　可以看到，smile和admin这两个密码确实存在于password.lst中。然而，仅仅几千条记录的字典无法覆盖所有用户的密码，如果用户密码没有出现在字典中，但和其中的某条记录很接近，比如字典中存在`hello`，而用户的密码是`hel1lo`，我们该怎么继续利用字典来破解这一密码呢？这里就需要用到John提供的mangling rules了，我们可以设计这么一条规则：
``` bash
>[1-9] i\0[a-zA-Z0-9]
# >[1-9]表示对于1-9中的任意一个数字，当输入单词的长度比它大时，执行之后的规则
# i代表insert，后面跟两个参数，分别为插入位置与插入字符，比如iNX表示在N位置上插入一个字符X
# \0即为插入位置，举个例子，如果输入单词长度为6，那么这里的\0表示[1-5]，如果输入单词长度为12，那么这里的\0表示[1-9]，所以\0可以理解为是规则>[1-9]的满足范围
# [a-zA-Z0-9]即为插入字符，比较容易理解，就是普通正则的用法，这里表示插入一个英文字母或数字。
```
　　为了将规则应用到破解中，我们需要修改john.conf文件，在`[List.Rules:Wordlist]`一节添加这条规则；当然也可以直接新建一个john.conf文件，只使用这一条规则，如下所示：
``` bash
[Options]
# Wordlist file name, to be used in batch mode
Wordlist = $JOHN/password.lst
# Use idle cycles only
Idle = Y
# Crash recovery file saving delay in seconds
Save = 600
# Beep when a password is found (who needs this anyway?)
Beep = N

# "Single crack" mode rules
[List.Rules:Wordlist]
:
>[0-9]  i\0[a-zA-Z0-9&!@#$%^&*=+.|?:"'_-] # 密码不一定是数字或字母，也有可能是特殊字符
```
　　我们再次尝试破解，这次我们需要加上`--rules`选项，表示我们需要将定义的规则应用在字典单词上：
``` bash
~/Documents/*** ⌚ 11:17:11
$ ./john --fork=6 --wordlist=password.lst --rules allpasswd
Loaded 10 password hashes with no different salts (md5crypt [MD5 32/64 X2])
Remaining 7 password hashes with no different salts
Node numbers 1-6 of 6 (fork)
Press 'q' or Ctrl-C to abort, almost any other key for status
alphab6et        (user7)
2 0g 0:00:00:25 100% 0g/s 11697p/s 11697c/s 81879C/s butterfly]1..rastafari]an
6 1g 0:00:00:25 100% 0.03920g/s 11480p/s 11480c/s 79715C/s jesuschri]st
passworld        (user8)
1 0g 0:00:00:26 100% 0g/s 11398p/s 11398c/s 79788C/s jesucrist]o..nightshad]e
Waiting for 5 children to terminate
5 0g 0:00:00:26 100% 0g/s 11170p/s 11170c/s 78193C/s instructo]r..newaccoun]t
3 1g 0:00:00:26 100% 0.03707g/s 10933p/s 10933c/s 76150C/s mancheste]r..bestfrien]ds
4 0g 0:00:00:27 100% 0g/s 10757p/s 10757c/s 75299C/s jethrotul]l
Session completed

~/Documents/*** ⌚ 11:18:12
$ ./john --show allpasswd
user1:user1
user2:admin
user3:smile
user7:alphab6et
user8:passworld

5 password hashes cracked, 5 left

~/Documents/*** ⌚ 11:18:12
$
```
　　现在已经成功破解了5个密码。然后，根据实验提示，字典文件提供了很多单词，而密码可能是两个单词的组合，比如helloworld就是hello和world的组合。所以我们需要将字典文件里的单词两两组合，重新验证，不过John貌似没有规则能够让我们同时接收两个单词，并组合起来进行验证，所以这里需要用到脚本程序，自己重新构造一个具有单词组合的新字典文件。脚本程序如下所示：
``` bash
#########################################################################
# File Name: tran.sh
# Author: Hac
# Mail: hac@zju.edu.cn
# Created Time: Fri 20 May 2016 11:33:08 AM CST
#########################################################################

#!/bin/bash
file=$1
while read a;
do
    while read b;
    do
        c=$a$b
        if [ ${#c} -le 10 ] && [ ${#c} -ge 3 ]; then # 过滤掉那些比较长或比较短的组合
            echo "$a$b" >> newPasswd.lst
        fi
    done < $file
done < $file
```
　　这个程序应该不难理解，直接通过两层循环读取单词，做一个笛卡尔积。运行这个脚本，我们能够在原字典文件的基础上，构造出一个新的字典文件，其中的单词全部是两个基础单词的组合。基于这个新的字典文件，我们再尝试使用之前的规则进行破解：
``` bash
~/Documents/*** ⌚ 22:50:44
$ ./john --fork=6 --wordlist=newPasswd.lst --rules allpasswd
Loaded 10 password hashes with no different salts (md5crypt [MD5 32/64 X2])
Remaining 5 password hashes with no different salts
Node numbers 1-6 of 6 (fork)
Press 'q' or Ctrl-C to abort, almost any other key for status
lovecats         (user6)
password222      (user4)
amy!password     (user9)
love&pizza       (user10)
Session completed

~/Documents/*** ⌚ 23:17:07
$ ./john --show allpasswd
user1:user1
user2:admin
user3:smile
user4:password222
user6:lovecats
user7:alphab6et
user8:passworld
user9:amy!password
user10:love&pizza

9 password hashes cracked, 1 left
```
　　这次破解会花费很长时间，主要是因为所使用的字典文件更大了。原来的字典文件只有3500多行，而我们将其中的单词两两组合(排除过长或过短的密码)后，新字典文件的行数达到了千万级别。花费了非常多的时间，我们的收获还是挺大的，这一次我们直接破解出了4个密码，分别是user4、user6、user9以及user10。现在，10个用户中只剩下user5的密码没有破解出来。
　　还剩下一个用户的密码没有破解。如果此时没有任何提示信息，那么我们就可以动用John的另一种非常强大的工作模式——Incremental。根据官方文档的说明，Incremental是John最强大的一种工作模式，因为它会尝试所有可能的字符组合，说白了就是真正的暴力破解。这种模式并没有尽头，因为字符组合有太多可能性，几乎不太可能在有限时间内穷举出所有字符组合。不过我们可以通过指定密码长度的范围，来减小字符组合的可能性，从而使得Incremental模式下的John能更快地破解出密码。

``` bash
# john.conf
# Incremental modes
[Incremental:ASCII]
File = $JOHN/ascii.chr
MinLen = 4
MaxLen = 8
CharCount = 95

[Incremental:LM_ASCII]
File = $JOHN/lm_ascii.chr
MinLen = 4
MaxLen = 8
CharCount = 69

[Incremental:Digits]
File = $JOHN/digits.chr
MinLen = 4
MaxLen = 8
CharCount = 10
```

　　下面我们尝试使用Incremental模式对最后一个用户密码进行破解：

``` bash
~/Documents/*** ⌚ 23:21:20
$ ./john --fork=6 --incremental allpasswd
Loaded 10 password hashes with no different salts (md5crypt [MD5 32/64 X2])
Remaining 1 password hash
Node numbers 1-6 of 6 (fork)
Press 'q' or Ctrl-C to abort, almost any other key for status
pass2012         (user6)
6 0g 0:00:01:00 0g/s 9890p/s 9890c/s 9890C/s lilachu..lilach1
3 0g 0:00:01:00 0g/s 9650p/s 9650c/s 9650C/s pops1c4..pops1c3
4 0g 0:00:01:00 0g/s 9681p/s 9681c/s 9681C/s 14015j..14014n
2 0g 0:00:01:00 0g/s 10071p/s 10071c/s 10071C/s cattored..cattore1
5 0g 0:00:01:00 0g/s 9949p/s 9949c/s 9949C/s maydi15..maydi16
1 0g 0:00:01:00 0g/s 10252p/s 10252c/s 10252C/s chserz..chsely
Waiting for 5 children to terminate
Session aborted

~/Documents*** ⌚ 23:22:48
$ ./john --show allpasswd
user1:user1
user2:admin
user3:smile
user4:password222
user5:pass2012
user6:lovecats
user7:alphab6et
user8:passworld
user9:amy!password
user10:love&pizza

10 password hashes cracked, 0 left
```

　　如上所示，我们成功通过Incremental模式破解出了最后一个用户的密码。

## Reference

1. [Lecture Homepage](http://10.214.148.181/readings/password-cracking/)(仅学校内网可访问)
2. [John the Ripper官方文档](http://www.openwall.com/john/doc/)(EXAMPLES和RULES两个章节非常重要)

  [1]: http://static.zybuluo.com/hac/d40y6wanri3be8brnr6nchzk/Selection_110.png

