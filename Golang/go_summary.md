# go summary

## go command

### go env

`go env`命令用于查看Go语言工具涉及的所有环境变量的值，包括未设置环境变量的默认值。

环境变量`GOPATH`用来指定当前工作目录。`GOPATH`对应的工作区目录有三个子目录。其中`src`子目录用于存储源代码，`pkg`子目录用于保存编译后的包的目标文件，`bin`子目录用于保存编译后的可执行程序。

环境变量`GOROOT`用来指定Go的安装目录，还有它自带的标准库包的位置。
GOROOT的目录结构和GOPATH类似，因此存放fmt包的源代码对应目录应该为$GOROOT/src/fmt。
用户一般不需要设置GOROOT，默认情况下Go语言安装工具会将其设置为安装的目录路径。

`GOOS`环境变量用于指定目标操作系统（例如android、linux、darwin或windows）。
`GOARCH`环境变量用于指定处理器的类型，例如amd64、386或arm等。

### go get

`go get`命令支持当前流行的托管网站GitHub、Bitbucket和Launchpad，可以直接向它们的版本控制系统请求代码。
对于其它的网站，你可能需要指定版本控制系统的具体路径和协议，例如 Git或Mercurial。
运行go help importpath获取相关的信息。

`go get`命令获取的代码是真实的本地存储仓库，而不仅仅只是复制源文件，因此你依然可以使用版本管理工具比较本地代码的变更或者切换到其它的版本。例如golang.org/x/net包目录对应一个Git仓库：

```shell
$ cd $GOPATH/src/golang.org/x/net
$ git remote -v
origin  https://go.googlesource.com/net (fetch)
origin  https://go.googlesource.com/net (push)
```

如果指定`-u`命令行标志参数，`go get`命令将确保所有的包和依赖的包的版本都是最新的，然后重新编译和安装它们。
如果不包含该标志参数的话，而且如果包已经在本地存在，那么代码将不会被自动更新。

`go get -u`命令只是简单地保证每个包是最新版本，如果是第一次下载包则是比较方便的；
但是对于发布程序则可能是不合适的，因为本地程序可能需要对依赖的包做精确的版本依赖管理。
通常的解决方案是使用`vendor`的目录用于存储依赖包的固定版本的源代码，对本地依赖的包的版本更新也是谨慎和持续可控的。

通过`go help gopath`命令查看`Vendor`的帮助文档。（墙内用户在上面这些命令的基础上，还需要学习用翻墙来go get）

### go build

`go build`命令编译命令行参数指定的每个包。如果包是一个库，则忽略输出结果；这可以用于检测包是可以正确编译的。如果包的名字是main，`go build`将调用链接器在当前目录创建一个可执行程序；以导入路径的最后一段作为可执行程序的名字。

默认情况下，`go build`命令构建指定的包和它依赖的包，然后丢弃除了最后的可执行文件之外所有的中间编译结果。
依赖分析和编译过程虽然都是很快的，但是随着项目增加到几十个包和成千上万行代码，依赖关系分析和编译时间的消耗将变的可观，有时候可能需要几秒种，即使这些依赖项没有改变。

`go install`命令和`go build`命令很相似，但是它会保存每个包的编译成果，而不是将它们都丢弃。被编译的包会被保存到`$GOPATH/pkg`目录下，目录路径和 `src`目录路径对应，可执行程序被保存到`$GOPATH/bin`目录。（很多用户会将$GOPATH/bin添加到可执行程序的搜索列表中。）

还有，`go install`命令和`go build`命令都不会重新编译没有发生变化的包，这可以使后续构建更快捷。为了方便编译依赖的包，`go build -i`命令将安装每个目标所依赖的包。

因为编译对应不同的操作系统平台和CPU架构，go install命令会将编译结果安装到GOOS和GOARCH对应的目录。

更多细节，可以参考go/build包的构建约束部分的文档。

```shell
$ go doc go/build
```

### 包文档

首先是go doc命令，该命令打印其后所指定的实体的声明与文档注释，该实体可能是一个包：

```shell
$ go doc time
```

或者是某个具体的包成员：

```shell
$ go doc time.Since
```

或者是一个方法：

```shell
$ go doc time.Duration.Seconds
```

### 查询包

go list命令可以查询可用包的信息。其最简单的形式，可以测试包是否在工作区并打印它的导入路径：

```shell
$ go list github.com/go-sql-driver/mysql
github.com/go-sql-driver/mysql
```

go list命令的参数还可以用"..."表示匹配任意的包的导入路径。我们可以用它来列出工作区中的所有包：

```shell
$ go list ...
```

或者是特定子目录下的所有包：

```shell
$ go list gopl.io/ch3/...
```

go list命令还可以获取每个包完整的元信息，而不仅仅只是导入路径，这些元信息可以以不同格式提供给用户。其中-json命令行参数表示用JSON格式打印每个包的元信息。

```shell
$ go list -json hash
```

命令行参数-f则允许用户使用text/template包（§4.6）的模板语言定义输出文本的格式。下面的命令将打印strconv包的依赖的包，然后用join模板函数将结果链接为一行，连接时每个结果之间用一个空格分隔：

```shell
$ go list -f '{{join .Deps " "}}' strconv
go list -f '{{.ImportPath}} -> {{join .Imports " "}}' compress/...
```

我们可以用go list命令查看包对应目录中哪些Go源文件是产品代码，哪些是包内测试，还哪些测试扩展包。
我们以fmt包作为一个例子：GoFiles表示产品代码对应的Go源文件列表；也就是go build命令要编译的部分。

```shell
$ go list -f={{.GoFiles}} fmt
[doc.go format.go print.go scan.go]
```

TestGoFiles表示的是fmt包内部测试测试代码，以_test.go为后缀文件名，不过只在测试时被构建：

```shell
$ go list -f={{.TestGoFiles}} fmt
[export_test.go]
```

XTestGoFiles表示的是属于测试扩展包的测试代码，也就是fmt_test包，因此它们必须先导入fmt包。同样，这些文件也只是在测试时被构建运行：

```shell
$ go list -f={{.XTestGoFiles}} fmt
[fmt_test.go scan_test.go stringer_test.go]
```

### go test

在*_test.go文件中，有三种类型的函数：测试函数、基准测试函数、示例函数。

一个测试函数是以Test为函数名前缀的函数，用于测试程序的一些逻辑行为是否正确；
go test命令会调用这些测试函数并报告测试结果是PASS或FAIL。

基准测试函数是以Benchmark为函数名前缀的函数，它们用于衡量一些函数的性能；
go test命令会多次运行基准函数以计算一个平均的执行时间。

示例函数是以Example为函数名前缀的函数，提供一个由编译器保证正确性的示例文档。

go test命令会遍历所有的*_test.go文件中符合上述命名规则的函数，然后生成一个临时的main包用于调用相应的测试函数，
然后构建并运行、报告测试结果，最后清理测试中生成的临时文件。

参数 `-v` 可用于打印每个测试函数的名字和运行时间。

参数 `-run` 对应一个正则表达式，只有测试函数名被它正确匹配的测试函数才会被 go test 测试命令运行：

```shell
go test -v -run="French|Canal"
```

那么对于一个随机的输入，我们如何能知道希望的输出结果呢？这里有两种处理策略。

- 第一个是编写另一个对照函数，使用简单和清晰的算法，虽然效率较低但是行为和要测试的函数是一致的，然后针对相同的随机输入检查两者的输出结果。
- 第二种是生成的随机输入的数据遵循特定的模式，这样我们就可以知道期望的输出的模式。

要注意的是在测试代码中并没有调用log.Fatal或os.Exit，因为调用这类函数会导致程序提前退出；调用这些函数的特权应该放在main函数中。
如果真的有意外的事情导致函数发生panic异常，测试驱动应该尝试用recover捕获异常，然后将当前测试当作失败处理。
如果是可预期的错误，例如非法的用户输入、找不到文件或配置文件不当等应该通过返回一个非空的error的方式处理。
