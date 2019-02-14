# Linux就该这么学

## 第二章
- date命令用于显示/设置系统的时间或日期。格式为 date [选项] [+指定的格式]
  如：
  查看系统当前时间
  `date '+%Y-%m-%d %H:%M:%S'`

  设置系统当前时间
  `date set -s '20150901 8:30:00'`

- reboot命令用于重启系统(仅root用户可以使用)。格式为`reboot`

- uname命令用于查看系统内核版本等信息.格式为`uname [-a]`

- uptime 命令用于查看系统的负载情况，格式为:`uptime`
可以使用`watch -n 1 uptime`来每一秒获得当前系统负载情况.输出内容分别为当前时间、系统已运行时间、当前在线用户以及平均值负载。而平均值负载分为最近1分钟、5分钟、15分钟的系统负载情况，负载值越低越好（小于1是正常）。

- free命令用于查看系统当前内存使用情况  
格式为：free [-m/-g]

- who命令用于查看**当前**登入主机的用户情况。格式为`who [参数]`

- last 查看所有系统的登入记录，格式为`last [参数]`

- history 显示历史执行过的命令
>历史命令被保存在用户家目录中的'.bash_history'文件中。history默认会保存1000条执行过的命令，若要修改可直接编辑/etc/profile文件的HISTSIZE值

- pwd 显示当前的工作目录，pwd -P 显示真实路径(即非快捷链接的地址)

- ls 查看目录中有哪些文件格式为 `ls [选项] [文件]`

| 参数    | 作用     |
| :------------- | :------------- |
| -a       | 查看全部文件(包括隐藏文件) |
| -d| 仅看目录本身|
| -h | 易读的文件容量(如k,m,g)|
|-l| 显示文件的详细信息|

如`ls -lh`

- cat 查看纯文本文件 格式为`cat [选项] [文件]`

| 选项     | 作用     |
| :------------- | :------------- |
| -n | 显示行号       |
| -b | 显示行号(不包括空行)|
|-A| 显示出“不可见”的符号，如空格 tab建等|

- grep命令用于对文本进行搜索  
格式为 grep [OPTION]... PATTERN [FILE]...
Search for PATTERN in each FILE or standard input.
PATTERN is, by default, a basic regular expression (BRE).
Example: grep -i 'hello world' menu.h main.c

|参数|作用|
|:---------|:---------|
|-b | 将可执行文件(binary)当做文本文件(text)来搜索|  
|-c | 仅显示找到的次数|
|-i | 忽略大小写|
|-n | 显示行号|
|-v | 反向选择--仅仅列出没有“关键词”的行|

- tr 命令用于转换文本文件中的字符，格式为 `tr [原始字符] [目标字符]`
如:`cat a.txt | tr [a-z] [A-Z]`
将a.txt中的字符变成大写

- wc用于统计指定文本的行数、字数、字节数，格式为"wc [参数] 文本"

| 参数 | 作用    |
| :------------- | :------------- |
| -l       | 只显示行数       |
|-w| 只显示单词|
|-c| 只显示字节数|


```shell
[root@bogon ~]# cat a.txt
a
bv
adfa
adfa
[root@bogon ~]# grep a a.txt
a
adfa
adfa
```
查询a在a.txt中出现的地方

- find 命令用于查找文件 格式:`find [查找路径] 寻找条件 操作`
小技巧：对于搜索路径有几个小窍门:
  - "~"代表用户家目录
  - "."代表当前目录
  - "/"代表根目录

  如
  ```shell
  [root@bogon ~]# find . -name a.txt
  ./a.txt
  ```


## 第三章
### 管道命令符
管道命令符"|"的作用是将前一个命令的标准输出当做后一个命令的标准输入，格式为:"命令A|命令B"。
如 找出被限制登录用户的命令是:grep "/sbin/nologin" /etc/passwed

统计文本行数的命令则是：wc -l
现在要做的就是讲搜索命令的输出值传递给统计命令，其实只要吧管道符加载中间就可以了
```/bin/bash
grep "/sbin/nologin" /etc/passwd | wc -l
```
统计文件中单词出现次数:
cat <file> | grep -o <word> | wc -l

### 输入输出重定向  
`>`、>>或者<、<<
对于输出重定向:
命令 > 文件 :将标准输出重定向到一个文件中(清空原有文件的数据)
命令 >> 文件: 将标准输出重定向到一个文件中(追加到原有内容后面)

对于输入重定向:
命令 < 文件: 将文件作为命令的标准输入
命令 << 分界符: 从标准输入中读入，直到遇见“分界符”才停止
命令 < 文件1 > 文件2: 将文件1作为命令的标准输入并将标准输出到文件2

demo:
```/bin/bash
#将man命令的帮助文档写入到 /root/man.txt中
➜ man bash > man.txt

#将man.txt文件作为输入重定向给wc -l命令来计算行数，命令等同于"cat man.txt | wc -l"
➜ wc -l < man.txt
```


### 通配符
- * 匹配0个或多个字符
- ? 匹配人任意单个字符
- [0-9] 范围内的数字
- [abc] 匹配已出的任意字符

vim编辑器快捷方式
:set nu 显示行号
:set nonu 不显示行号
:整数 跳转到该行


### 实用的PATH变量
alias 命令用于设置命令的别名，格式为:"alias 别名=命令"
eg: alias cp="cp -i"

unalias命令用于取消命令的别名，格式为:"unalias 别名"

>系统中的类似$PATH这类的环境变量，可以通过env命令查看
```Shell
➜  ~ env
TERM_SESSION_ID=w0t0p0:6C5DD8A5-11F2-4F04-9F89-85280CFED2CD
SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.NO7wlV0t75/Listeners
Apple_PubSub_Socket_Render=/private/tmp/com.apple.launchd.if3QE7g7rj/Render
COLORFGBG=7;0
ITERM_PROFILE=Default
XPC_FLAGS=0x0
PWD=/Users/johnxue
SHELL=/bin/zsh
LC_CTYPE=UTF-8
TERM_PROGRAM_VERSION=3.2.7beta1
TERM_PROGRAM=iTerm.app
PATH=/usr/local/mysql/bin:/developTools/zookeeper-3.4.9/bin:/developTools/apache-maven-3.3.9/bin:/usr/local/mysql/bin:/Library/Frameworks/Python.framework/Versions/3.6/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Applications/VMware Fusion.app/Contents/Public:/usr/local/go/bin:/Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home/bin:/opt/gradle/gradle-4.0.1/bin:/developTools/btrace-bin-1.3.9/bin:/Users/johnxue/Library/Android/sdk/platform-tools:/Users/johnxue/Documents/code/gothird/bin:/Users/johnxue/Documents/code/mygo/bin:/Users/johnxue/.rvm/bin
COLORTERM=truecolor
TERM=xterm-256color
HOME=/Users/johnxue
TMPDIR=/var/folders/mg/13rgl7zx78n4pvmryqk44zv80000gn/T/
USER=johnxue
XPC_SERVICE_NAME=0
LOGNAME=johnxue
__CF_USER_TEXT_ENCODING=0x0:25:52
ITERM_SESSION_ID=w0t0p0:6C5DD8A5-11F2-4F04-9F89-85280CFED2CD
SHLVL=1
OLDPWD=/Users/johnxue
JAVA_7_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_76.jdk/Contents/Home
JAVA_8_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home
JAVA_10_HOME=/Library/Java/JavaVirtualMachines/jdk-10.0.1.jdk/Contents/Home
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home
CLASSPATH=.:%JAVA_HOME%/lib/dt.jar:%JAVA_HOME%/lib/tools.jar
ANDROID_HOME=/Users/johnxue/Library/Android/sdk
NGINX=/usr/local/nginx/sbin
GOPATH=/Users/johnxue/Documents/code/gothird:/Users/johnxue/Documents/code/mygo
ZSH=/Users/johnxue/.oh-my-zsh
PAGER=less
LESS=-R
LSCOLORS=Gxfxcxdxbxegedabagacad
rvm_prefix=/Users/johnxue
rvm_path=/Users/johnxue/.rvm
rvm_bin_path=/Users/johnxue/.rvm/bin
_system_type=Darwin
_system_name=OSX
_system_version=10.14
_system_arch=x86_64
rvm_version=1.27.0 (latest)
_=/usr/bin/env
```
如果有需求可以直接修改。

假设设置一个变量"WORKDIR"，让每个用户执行"cd $WORKDIR"都登录到/home/workdir目录中，可如下设置
```/bin/bash
> mkdir /home/workdir
> export WORKDIR=/home/workdir
> cd $WORKDIR
> pwd
/home/workdir
```
**export关键字将局部变量WORKDIR提升为全局变量**
这样当切换用户时，WORKDIR对于新切换的对象也是可见的，新用户可以通过cd $WORKDIR进入/home/workdir

> cat /etc/shells查看系统中所有可用的Shell解释器  

```shell
➜  cat /etc/shells
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```
查看正在使用的shell
```shell
echo $SHELL
```

一个Shell脚本应该包括:"脚本声明"、"注释信息"、"可执行语句"
- 脚本声明  
  使用"#!"告知系统使用何种shell来解释脚本
- 注释信息  
  使用"#"对可执行语句或程序功能做介绍，可以不写
- 可执行语句  
  执行的具体命令

如：
```shell
vim Example.sh
  #!bin/bash
  #for example by han.xue
  pwd
  ls -al
```
执行脚本的方法有三种：
1. 脚本文件路径: ./Example.sh
2. sh 脚本文件路径: sh Example.sh
3. source脚本文件路径: source Example.sh

只要脚本路径没有写错，sh或source命令都可以直接执行该脚本，但直接访问脚本(./XXX.sh的方式)的方式有点特殊。使用直接访问脚本的方式提示权限不足，此时需要为脚本设置可执行权限后才能顺利运行
```shell
chmod u+x xxx.sh
./xxx.sh
```

### 接收用户参数  


|$0|当前执行Shell脚本的程序名|
|-------|-------|
|$1-9,${10},${11},${11}...| 参数的位置变量|
|$#|一共有多少个参数|
|$*| 所有位置变量的值|
|$?|判断上一条命令是否执行成功，0为成功，非0为失败|  
```bin/bash
vim Example.sh
#!bin/bash
echo "当前脚本名称为$0"
echo "总共有$#个参数，分别是$*"
echo "第1个参数为$1，第5个为$5"
```
使用sh来执行脚本，并附带6个参数
```bin/bash
sh Example.sh one two three four five six
```

### 用户身份与文件权限
Linux系统中**一些都是文件**，文件和目录的**所属**与**权限**---**来分别规定所有者、所有组、其余的人读、写、执行权限。**

读(read)，写(write),执行e(xecute)简写为(r,w,x),亦可用数字(4,2,1)表示

| 权限项|读|写|执行|读|写|执行|读|写|执行
| :---- | :----- |
| 字符表示 |r|w|x|r|w|x|r|w|x
|数字表示|4|2|1|4|2|1|4|2|1
|权限分配|文件所有者|文件所属组|其他用户|

```shell
[root@bogon ~]# ls -lh a.txt
-rw-r--r--. 1 root root 15 Jan 31 22:30 a.txt
```
权限位的第一个减号-代表的是文件类型
- - 表示普通文件
- d 表示目录文件
- l 表示链接文件
- b 表示块设备文件
- c 表示字符设备文件
- 管道文件

文件的权限为rw-r--r--也就是分别表示所有者(属主)有读写权限，所有组（属组）有读权限，其余人也仅有读权限。

第一个root表示属主
第二个root表示属组

对于目录文件x(可执行)表示用户可以进入到该目录中
