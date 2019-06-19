# 基于GitLab CI搭建Golang自动构建环境

## Golang发布遇到的问题

对于golang的发布，之前一直没有一套规范的发布流程，来看看之前发布流程：

### 方案一

- 开发者本地环境需要将环境变量文件改为正式环境配置
- 编译成可执行文件
- 发送给运维
- （运维）将文件覆盖为线上
- （运维）重启进程

（可谓“又臭又长”）

### 方案二

- 开发者将代码commit到gitlab上交给运维同学
- （运维）pull代码
- （运维）编译成可执行文件
- （运维）覆盖线上文件
- （运维）重启进程

这种对于运维属于重度依赖，而运维同学又需要去关心代码的编译，增加了运维同学的工作了。

以上两种方案都是之前项目中发生过的，对于发版来说可谓是一种“噩梦”，易出错，流程长，运维要是不在根本无法操作。

## 解决方案

为了解决上面提到的两种发布问题，目前我们做了如下的设计方案：

![](https://rawcdn.githack.com/WilburXu/blog/2a44ce35957c6c0a3a7808536f865bf4700c77a6/other/images/gitlab-ci集成_1.png)

1. 开发者提交代码到GitLab服务器
2. 添加一个Tag触发构建（.gitlab-ci.yml+makefile）
3. 将构建后的文件打包添加上版本号（1.0.0+）后推送到版本服务器
4. 在Jenkins选择对应的版本号发布到指定Web Server上

发布时，开发只需要编写“.gitlab-ci.yml”以及makefile对项目进行构建，而运维只需要配置jenkins，将文件发布到Web Server上即可，完成了开发和运维的解耦。

## 什么是GitLab-CI

`GitLab CI` 是 `GitLab Continuous Integration` （Gitlab 持续集成）的简称。从 `GitLab` 的 8.0 版本开始，`GitLab` 就全面集成了 `Gitlab-CI`,并且对所有项目默认开启。只要在项目仓库的根目录添加 `.gitlab-ci.yml` 文件，并且配置了 `Runner`（运行器），那么每此添加新的`tag`都会触发 `CI pipeline`。

### 一些概念	

在介绍 GitLab CI 之前，我们先看看一些持续集成相关的概念。

![CI/CD Overview](https://about.gitlab.com/images/blogimages/cicd_pipeline_infograph.png)

#### Pipeline

一次 Pipeline 其实相当于一次构建任务，里面可以包含多个流程，如安装依赖、运行测试、编译、部署测试服务器、部署生产服务器等流程。 任何提交或者 Merge Request 的合并都可以触发 Pipeline，如下图所示：

```go
+------------------+           +----------------+
|                  |  trigger  |                |
|   Commit / MR    +---------->+    Pipeline    |
|                  |           |                |
+------------------+           +----------------+
```

#### Stages

Stages 表示构建阶段，说白了就是上面提到的流程。 我们可以在一次 Pipeline 中定义多个 Stages，这些 Stages 会有以下特点：

- 所有 Stages 会按照顺序运行，即当一个 Stage 完成后，下一个 Stage 才会开始
- 只有当所有 Stages 完成后，该构建任务 (Pipeline) 才会成功
- 如果任何一个 Stage 失败，那么后面的 Stages 不会执行，该构建任务 (Pipeline) 失败

因此，Stages 和 Pipeline 的关系就是：

```go
+--------------------------------------------------------+
|                                                        |
|  Pipeline                                              |
|                                                        |
|  +-----------+     +------------+      +------------+  |
|  |  Stage 1  |---->|   Stage 2  |----->|   Stage 3  |  |
|  +-----------+     +------------+      +------------+  |
|                                                        |
+--------------------------------------------------------+
```

#### Jobs

Jobs 表示构建工作，表示某个 Stage 里面执行的工作。 我们可以在 Stages 里面定义多个 Jobs，这些 Jobs 会有以下特点：

- 相同 Stage 中的 Jobs 会并行执行
- 相同 Stage 中的 Jobs 都执行成功时，该 Stage 才会成功
- 如果任何一个 Job 失败，那么该 Stage 失败，即该构建任务 (Pipeline) 失败

所以，Jobs 和 Stage 的关系图就是：

```go
+------------------------------------------+
|                                          |
|  Stage 1                                 |
|                                          |
|  +---------+  +---------+  +---------+   |
|  |  Job 1  |  |  Job 2  |  |  Job 3  |   |
|  +---------+  +---------+  +---------+   |
|                                          |
+------------------------------------------+
```



## 什么是MakeFile

Makefile文件的作用是告诉make工具需要如何去编译和链接程序，在需要编译工程时只需要一个make命令即可，避免了每次编译都要重新输入完整命令的麻烦，大大提高了效率，也减少了出错率。

### 基本介绍

我们平常很多时候都是直接在命令行输入go build进行编译的：

```go
go build .
```

或者测试使用go run运行项目：

```go
go run main.go
```

我看有很多大型开源项目都是如下方式 ：

```makefile
make build
# 或者
make install
```

我们打包运行这个过程，还有一个更加贴切的词语叫做构建项目。

### 案例

我们先创建一个新的工程，目录如下：

- main.go
- Makefile

**make.go源码：**

```go
package main

import "fmt"

func main() {
   fmt.Println("hi, pang pang.")
}
```

就多了一个Makefile文件，如果要使用Makefile去构建你项目，就需要在你的项目里面新建这个Makefile文件。

这里我贴一个简单的`Makefile`文件的源码：

```makefile
BINARY_NAME=hello
build:
    go build -o $(BINARY_NAME) -v
    ./$(BINARY_NAME)
```

解释下上面各行的意思：

- 第一行，声明了一个变量`BINARY_NAME`他的值是`hello`，方便后面使用
- 第二行，声明一个 `target`，其实你可以理解成一个对外的方法
- 第三行，这就是这个`target`被调用时执行的脚本，这行就是build这个项目，编译后的二进制文件放在当前工程目录下，名字是变量`BINARY_NAME`的值
- 第四行，这一行就是直接执行当前目录下的二进制文件

**构建**

我们打开我们的终端，直接执行：

```makefile
make build
```

将生成一个可执行文件`hello`，这就是对Makefile的简单介绍，具体命令推荐 [阮一锋的Makefile教程](http://www.ruanyifeng.com/blog/2015/02/make.html) 



## 部署流程

前面大致介绍了构建自动化构建的用到的一些概念和工具，那么我们就可以开始我们的部署了。

### GitLab Runner

了解上面了基本的概念后，那由何人来执行这些任务呢？那就是 GitLab Runner（运行器）

#### 安装

![](https://rawcdn.githack.com/WilburXu/blog/98380bf89bf578f98ba3222d5868ff52b3ec2442/other/images/gitlab-ci集成_3.png)

安装 GitLab Runner 太简单了，按照着 [官方文档](https://docs.gitlab.com/runner/#install-gitlab-runner) 的教程来就好拉！ 下面是 Debian/Ubuntu/CentOS 的安装方法，其他系统去参考官方文档：

```bash
# For Debian/Ubuntu
$ curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-ci-multi-runner/script.deb.sh | sudo bash
$ sudo apt-get install gitlab-ci-multi-runner

# For CentOS
$ curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-ci-multi-runner/script.rpm.sh | sudo bash
$ sudo yum install gitlab-ci-multi-runner
```

#### 注册 Runner

安装好 GitLab Runner 之后，我们只要启动 Runner 然后和 CI 绑定就可以了：

- 打开你 GitLab 中的项目页面，在项目设置中找到 runners
- 运行 `sudo gitlab-ci-multi-runner register`
- 输入 CI URL
- 输入 Token
- 输入 Runner 的名字
- 选择 Runner 的类型，简单起见还是选 Shell 吧
- 完成

当注册好 Runner 之后，可以用 `sudo gitlab-ci-multi-runner list` 命令来查看各个 Runner 的状态：

```bash
$ sudo gitlab-runner list
Listing configured runners          ConfigFile=/etc/gitlab-runner/config.toml
my-runner                           Executor=shell Token=cd1cd7cf243afb47094677855aacd3 URL=http://mygitlab.com/ci
```



### .gitlab-ci.yml编写

```makefile
before_script:
  - export GOPATH=$GOPATH:/usr/local/${CI_PROJECT_NAME}
  - export # 引入环境变量
  - cd /usr/local/${CI_PROJECT_NAME}/src/${CI_PROJECT_NAME}
  - export VERSION=`echo ${CI_COMMIT_TAG} | awk -F"_" '{print $1}'`

# stages
stages:
  - build
  - deploy
# jobs
build-tags:
  stage: build
  script:
  	## 执行makefile文件
    - make ENV="prod" VERSION=${VERSION}
    - if [ ! -d "/data/code/project_name/tags/${VERSION}" ]; then
      mkdir -p "/data/code/project_name/tags/${VERSION}/";
      fi
    - cd compiler/
    - mv -f project_name.tar.gz /data/code/project_name/tags/${VERSION}/
  only:
    - tags
deploy-tags:
  stage: deploy
  script:
    - cd /data/code/project_name/tags/
    - svn add ${VERSION}
    - svn commit -m "add ${CI_COMMIT_TAG}"
  only:
    - tags
```

### Makefile编写

```makefile
export VERSION=1.0.0
export ENV=prod
export PROJECT=project_name

TOPDIR=$(shell pwd)
OBJ_DIR=$(OUTPUT)/$(PROJECT)
SOURCE_BINARY_DIR=$(TOPDIR)/bin
SOURCE_BINARY_FILE=$(SOURCE_BINARY_DIR)/$(PROJECT)
SOURCE_MAIN_FILE=main.go

BUILD_TIME=`date +%Y%m%d%H%M%S`
BUILD_FLAG=-ldflags "-X main.version=$(VERSION) -X main.buildTime=$(BUILD_TIME)"

OBJTAR=$(OBJ_DIR).tar.gz

all: build pack
   @echo "\n\rALL DONE"
   @echo "Program:       "  $(PROJECT)
   @echo "Version:       "  $(VERSION)
   @echo "Env:          "  $(ENV)

build:
   @echo "start go build...."
   @rm -rf $(SOURCE_BINARY_DIR)/*
   @go build $(BUILD_FLAG) -o $(SOURCE_BINARY_FILE) $(SOURCE_MAIN_FILE)

pack:
   @echo "\n\rpacking...."
   @tar czvf $(OBJTAR) -C $(OBJ_DIR) .
```

## 执行

完成的部署流程（其实很简单），然后每次要构建只需要执行：

```bash
git commit -a -m "我准备打tag测试啦"
git push

## 打tag
git tag -a "1.0.0" -m"1.0.0"
git push origin 1.0.0
```

那么在GitLab就会看到：

![](https://rawcdn.githack.com/WilburXu/blog/4d5f4cab03fdbb283586397ec8afc1a824843cc0/other/images/gitlab-ci_2.png)

大功告成，以上！~



## 总结

其实发布流程有无数种，每一个团队都有合适自己的发布流程，如果项目小，团队小，也许手动发布就足够了，而对于项目有一定规模的团队，或许需要更加规范的发布流程来保障项目的稳定发版。

但是对于项目的发版，每个人都应该要有根据团队目前现状去选择和制定最合适的方案的能力，除此之外，上面提到的工具和语言也是不错的工具噢！~