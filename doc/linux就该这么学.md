# Linux就该这么学

## 第一章
- grep命令用于对文本进行搜索  
格式为 grep[选项] [文件]
搜索某个关键词: grep 关键词 文本文件  

|参数|作用|
|:---------|:---------|
|-b | 将可执行文件(binary)当做文本文件(text)来搜索|  
|-c | 仅显示找到的次数|
|-i | 忽略大小写|
|-n | 显示行号|
|-v | 方向选择--仅仅列出没有“关键词”的行|

- uname 查看系统内核版本等信息
格式为 uname -a

- free命令用于查看系统当前内存使用情况  
格式为：free [-m/-g]


## 第二章
### 管道命令符
管道命令符"|"的作用是将前一个命令的标准输出当做后一个命令的标准输入，格式为:"命令A|命令B"。
如 找出被限制登录用户的命令是:grep "/sbin/nologin" /etc/passwed

统计文本行数的命令则是：wc -l
现在要做的就是讲搜索命令的输出值传递给统计命令，其实只要吧管道符加载中间就可以了
```/bin/bash
grep "/sbin/nologin" /etc/passwd | wc -l
```

### 输入输出重定向
>、>>或者>、>>

### 通配符
- * 匹配0个或多个字符
- ? 匹配人任意单个字符
- [0-9] 范围内的数字
- [abc] 匹配已出的任意字符

vim编辑器快捷方式
:set nu 显示行号
:set nonu 不显示行号
:整数 跳转到该行

> cat /etc/shells查看系统中所有可用的Shell解释器  

```bin/bash
➜  Downloads cat /etc/shells
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

一个Shell脚本应该包括:"脚本声明"、"注释信息"、"可执行语句"
- 脚本声明  
  使用"#!"告知系统使用何种shell来解释脚本
- 注释信息  
  使用"#"对可执行语句或程序功能做介绍，可以不写
- 可执行语句  
  执行的具体命令

如：
```bin/bash
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

接收用户参数  


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
