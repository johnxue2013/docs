# The Go Programming Language 笔记

## 1.1
Go是一门编译型语言(静态编译)

```Go
//hello.go
package main

import "fmt"

func main() {
	fmt.Printf("hello, world!")
}
```

```Bash
$ go run hello.go
```
```
# 运行结果
hello, world!
```
go提供的工具都通过一个单独的命令go调用，go命令有一系列的子命令。最简单的一个子命令就是run。该命令编译一个或多个以.go结尾的源文件，链接库文件，并运行最终生成的可执行文件。

`go build`命令可以保存编译结果，该命令生成一个名为hello的可执行的二进制文件(windows下生成的是hello.exe，增加了.exe后缀)，之后你可以随时运行它

```Bash
$ go build hello.go
$ ./hello
```
```
#运行结果
hello, world!
```

**GOPATH**
自己设置的保存go代码的根目录路径，可通过命令行`go env`查看本机go环境配置，可以多有个目录，多个目录使用:分隔，window使用;分隔。当GOPATH有多个值时，默认将`go get`获取的包存放在第一个目录下

$GOPATH目录约定有三个子目录
- src存放源代码(比如：.go .c .h .s等)   按照golang默认约定，go run，go install等命令的当前工作路径（即在此路径下执行上述命令）。
- pkg编译时生成的中间文件（比如：.a）　　golang编译包时
- bin编译后生成的可执行文件（为了方便，可以把此目录加入到 $PATH 变量中，如果有多个gopath，那么使用${GOPATH//://bin:}/bin添加所有的bin目录

**go install**
`go install`会生成可执行文件直接放到bin目录下，当然这是有前提的
你编译的是可执行文件，如果是一个普通的包，会被编译生成到pkg目录下该文件是.a结尾

**go get命令**
go get会做两件事：
- 从远程下载需要用到的包(将放在GOPATH目录中的第一个目录)
- 执行go install

`import`声明必须跟在文件的`package`声明之后。随后，则是组成程序的函数、变量、常量类型的声明语句(分别由关键字`func`,`var`,`const`,`type`定义).

`go get`可以根据要求和

Go语言不需要在语句或者声明的末尾添加分号，除非一行上有多条语句。实际上，编译器会主动把特定的符号后的换行符转换为分号。
