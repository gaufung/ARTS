---
title: Go工程Makefile管理
date: 2018-06-10
status: public
---
# 背景
最近一直使用Go语言编写公司相关的项目，主要是涉及Web API编写，由于先前项目技术栈是`Python`和`Openresty`的组合，其中为了高并发要求，`Python`使用了`Tornado`框架，`Openresty`则选择更加小众的`Lua`语言，采用异步非阻塞的方式完成高并发的请求。使用`jekenis`项目集成管理平台，`k8s`负责整`Docker`生产环境。目前主要项目的面临的挑战有：
- 脚本语言在针对大型项目显得力不从心，由于采用微服务，每个项目的代码量并不是很大，矛盾还不是很突出；
- `lua`脚本语言比较小众，在招聘市场上比较棘手；
- `lua`语言第三方驱动较少，比如用户消息记录存放在阿里云的`tablestore`中，而官方提供了`Java`,`Python`,`Go`等常见编程语言的SDK；
因此目前打算部分服务往`Go`语言上迁移，由于`Go`对高并发的支持，而且`Go`语言的生态逐步成熟起来，似乎目前存在的问题都能很好的解决。
# 问题
由于先前整个服务端的同学都采用`Openresty-Lua`组合，脚本语言的项目只需要将源代码使用`Dockerfile`，将全部代码拷贝到容器中即可。但是类似`Go`静态语言有着很大的选择难题：
1. 在`Jekenis`中编译完毕拷贝可执行文件至容器，还是在容器中编译？
2. `go` 语言中相关的`$GOPATH`设置相当头疼，对于运维人员来说也是巨大的挑战
针对第一个问题，如何`jekenis`中的操作系统和处理器架构与`Docker`中容器不同，这样在编译的时候需要加入编译参数。对于第二个问题是个挑战，对于没有`Go`语言编程经验的运维同学来讲，更是相当头疼。
起初我是手把手教运维同学怎么使用`go run`,`go build`等相关命令，虽说这些写在脚本中，但是每次有新的第三方包更新，需要在钉钉上告诉他。尽管可以写在脚本中，但是这些仍然属于`笨拙`的方法。想来想去想到C和C++项目中常见的`Makefile`管理代码工程，在查阅相关资料后，查到了不错的解决方案。
# Makefile
`Makefile`编写很简单，将`go`提供了工具进行组合，而且极大的方便运维同学，相当方便，接下来以一个例子作为介绍：
```makefile 
# Go Parameter
GOCMD=go
GOBUILD=$(GOCMD) build
GOCLEAN=$(GOCMD) clean
GOTEST=$(GOCMD) test
GOGET=$(GOCMD) get
BINARY_NAME=$(shell echo $${PWD\#\#*/})
BINARY_LINUX=$(BINARY_NAME)_linux
BINARY_DARWIN=$(BINARY_NAME)_darwin
BINARY_WINDOW=$(BINARY_NAME)_windows
GITTAG=$(shell git describe --abbrev=0)
BUILD_TIME=$(shell date +%F@%T)
LDFLAGS=-ldflags "-X main.GitTag=${GITTAG} -X main.BuildTime=${BUILD_TIME}"
all: deps test build
build:
	$(GOBUILD) $(LDFLAGS) -o $(BINARY_NAME) -v
test:
	-$(GOTEST) -v ./...
clean:
	$(GOCLEAN)
	rm -rf $(BINARY_NAME)
	rm -rf $(BINARY_LINUX)
	rm -rf $(BINARY_WINDOW)
	rm -rf $(BINARY_DARWIN)
run:
	$(GOBUILD) -o $(BINARY_NAME) -v
	./$(BINARY_NAME)
deps:
	$(GOGET) -u github.com/gorilla/mux
	$(GOGET) -u github.com/go-redis/redis
	$(GOGET) github.com/koding/cache
	$(GOGET) gopkg.in/jarcoal/httpmock.v1
# cross compilation
build-linux:
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 $(GOBUILD) -o $(BINARY_LINUX) -v
build-darwin:
	CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 $(GOBUILD) -o $(BINARY_DARWIN) -v
build-windows:
	CGO_ENABLED=0 GOOS=windows GOARCH=amd64 $(GOBUILD) -o $(BINARY_WINDOW) -v
```
## go tool alias
`Makefile`中前面数行中，首先使用重新命令
- build
- clean
- test
- get
## Binary Name
使用`Makefile`所在文件夹为可执行文件名称，接下来就是定义了不同平台操作可执行文件名称。
## Make target
- build：构建项目
- test: 测试项目，由于测试可能依赖一些数据库，因此在这里使用`-`前缀，过滤到测试安排。
- clean: 删除编译后的成果
- run: 运行不编译程序
- deps: 安装依赖
- all： `deps`，`test`，`build`
- build-linux, build-darwin, build-windows 不同平台编译。
## Version
目前几乎所有项目使用`git`进行代码管理，公司内部搭建了`gitlab`， 所有代码项目中，如果使用git push <tag>操作，将会触发自动化部署工作。因此在构建的过程中，可以通过`git tag`最新标签，作为改版本后，在运行程序过程中，通过`./a -v`显示程序的版本号和构建时间。这样避免用开发人员每次修改代码中的软件版本号，尽量做到自动化工作。
那么在代码中如何处理呢？
```go
var (
	GitTag = "non-tag"
	BuildTime = "non-time"
)
func main() {
	version := flag.Bool("v", false, "version")
	flag.Parse()
	if *version {
		fmt.Println("Version: " + GitTag)
		fmt.Println("Build Time: "+ BuildTime)
	}else{
    	   // ....
    	}
```
这样做的话，只要在编译项目的时候，添加如何参数
```makefile
GITTAG=$(shell git describe --abbrev=0)
BUILD_TIME=$(shell date +%F@%T)
LDFLAGS=-ldflags "-X main.GitTag=${GITTAG} -X main.BuildTime=${BUILD_TIME}"
## go build 
$(GOBUILD) $(LDFLAGS) -o $(BINARY_NAME) -v
```
通过这种方式，在`make`编译过程中，或者`tag`标签和编译时间戳。

项目版本控制使用的`git flow`工具，项目主要有两个分支：`master`和`develop`。主要开发工作使用`git flow`等相关命令进行切换分支，当往gitlab上提交分支的引起Jekenis编译，但是代码是从master分支上拉取代码，私以为这是不妥的做法，不过运维那边选择目前还是无能无力。因此每次都要切换分支至master,推送master分支和tag，因此在makefile中增加一个push命令
```makefile
push:
        git checkout master
        git push origin master
        git checkout develop
        git push origin develop
        git push origin ${GITTAG}
```

# 结语
对于开发人员，自动化工作永远是比程序员来讲更加靠谱，因此在工作中，不管是运维人员，开发人员都需要采用工具来辅助工作，而不是人为去做一些琐碎的事情。