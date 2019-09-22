---
tags:
 - Linux
---

&emsp;&emsp;cron是Linux系统中用于执行计划任务的程序，利用cron可以实现在指定的时间周期性的执行某些任务。

# crontab文件

cron执行的任务由crontab类型的配置文件指定，crontab文件的基本格式如下：

```bash
# 注释
# 设置MAILTO则cron会把每条命令的输出信息通过邮件发送给指定用户
SHELL=指定运行命令的shell
PATH=指定环境变量PATH（注意此处不能通过$PATH调用默认环境变量，必须写出所有路径）
MAILTO=username
# 系统crontab文件任务项：分钟 小时 天 月 星期 用户名(只有系统crontab文件需要) 需要执行的命令
# 用户crontab文件任务项：分钟 小时 天 月 星期 需要执行的命令

# 每分钟执行一次
* * * * * echo "hello cron" 

# 每小时的第三十分钟执行一次
30 * * * * echo "hello cron"

# 每天5时30分执行一次
30 5 * * * echo "hello cron"

# 每天5-7点中每小时的第30分钟执行一次
10 5-7 * * * echo "hello cron"

```

以`#`开头的为注释，每一项定时任务**占用一行**，共有6个字段（系统crontab文件有7个字段，多了一个用户名字段），使用空格分隔每个字段。其中前面5个字段则是该命令执行的时间，最后一个字段是需要执行的命令。

## 时间字段

命令执行时间的取值范围如下：

| 时间类型 | 取值范围                |
| -------- | ----------------------- |
| 分钟     | 0-59                    |
| 小时     | 0-23                    |
| 天       | 1-31                    |
| 月       | 1-12                    |
| 星期     | 0-7（0和7都表示星期日） |

范围时间表示：

| 符号  | 表示内容                                                     |
| ----- | ------------------------------------------------------------ |
| *     | 所有时间，某个时间字段设置为*表示从最小值到最大值的范围      |
| a-b   | a到b范围内的所有时间，如月份中1-12表示从1月到12月每月执行一次 |
| a-b/c | a-b范围内**间隔**为c的所有时间，如月份中1-12/2表示从1月到12月每隔2月执行一次 |

特殊时间表示：

可以用如下表示方式**代替前5个字段**来指定指令执行时间

| 符号      | 表示内容   |
| --------- | ---------- |
| @reboot   | 启动时执行 |
| @yearly   | 每年一次   |
| @annually | 每年一次   |
| @monthly  | 每月一次   |
| @weekly   | 每周一次   |
| @daily    | 每天一次   |
| @hourly   | 每小时一次 |

## 命令字段

命令字段可以是任意有效命令，并且第一个**非转义**（使用`\`转义）的`%`后边的内容会作为该命令的**标准输入**（stdin），并且所有非转义的`%`会被作为换行符处理（在标准输入内容中需要换行时就可以使用`%`来表示）

```bash
* * * * * read var % hello this is input content
```

> 访问http://man7.org/linux/man-pages/man5/crontab.5.html获取关于crontab文件的格式更详细的描述或者在linux终端中键入man 5 crontab来获取


# 创建和修改定时任务的方法

## 编辑系统crontab文件

第一种方法是使用**文本编辑器**直接修改系统crontab文件（main system crontab file）。系统crontab文件路径为`/etc/crontab`，编辑该文件则可以调整系统定时任务的执行，该文件内容如下：

```bash
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )

```

## 使用crontab命令

第二种方式是修改/var/spool/cron/\<username\>用户crontab文件，一般通过**crontab命令**来操作该文件，流程如下：

首先在任意位置创建用户crontab文件，例如建立名为my_crontab文件。

```bash
# min hour day month week
MAILTO=january

* * * * * echo "hello crontab"
```

然后使用**crontab命令**设置该文件为用户crontab文件，crontab命令会把指定文件内容复制到/var/spool/cron/\<username\>文件中

```bash
crontab my_crontab
```

使用`crontab -l`命令来查看/var/spool/cron/\<username\>用户crontab文件

使用`crontab -r`命令来删除/var/spool/cron/\<username\>用户crontab文件，从而停止执行定时任务

使用`crontab -e`命令来直接编辑/var/spool/cron/\<username\>用户crontab文件，从而修改定时任务

> 访问http://man7.org/linux/man-pages/man1/crontab.1.html获取关于crontab命令更详细的描述，或者在linux终端中键入man  crontab来获取。

# 关于MAILTO

指定了MAILTO选项并且在安装了**本地邮件系统**（postfix，sendmail）时，cron程序会把执行命令的输出内容通过邮件发送给MAILTO指定的用户，下面是cron程序发送的邮件：

```
Return-Path: <january@january.laptop>
X-Original-To: january
Delivered-To: january@january.laptop
Received: by january-PC (Postfix, from userid 1000)
	id B38592801EB; Wed, 17 Jul 2019 10:40:01 +0800 (CST)
From: root@january.laptop (Cron Daemon)
To: january@january.laptop
Subject: Cron <january@january-PC> echo "hello crontab"
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <MAILTO=january>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/home/january>
X-Cron-Env: <PATH=/usr/bin:/bin>
X-Cron-Env: <LOGNAME=january>
Message-Id: <20190717024001.B38592801EB@january-PC>
Date: Wed, 17 Jul 2019 10:40:01 +0800 (CST)
X-IMAPbase: 1563331245 5
Status: O
X-UID: 4

hello crontab
```

最后的hello crontab是命令输出内容。

本地用户的邮件会放置在`/var/mail/<username>`文件中，可以安装`mailx`软件来管理和查看

# 关于运行日志

对于安装了`rsyslog`的系统，在`/var/log/`目录中会有cron的运行日志，没有`rsyslog`的系统可以使用`journalctl`命令查看日志，键入`journalctl -u cron.service`或者`journalctl | grep cron`即可。