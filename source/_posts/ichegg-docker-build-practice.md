---
title: "爱扯淡 - docker build的好实践和坏实践"
date: 2020-05-21T15:20:09+08:00
lastmod: 2020-05-21T15:20:09+08:00
description: ""
tags: ["爱扯淡","docker","实践"]
categories: ["爱扯淡"]
chronologies: ["2020"]
author: ""
---

## 引言

Docker已经妥妥的成为了容器的业界标准，虽然不时的还有一些什么rkt, lxd将会是接下来的潮流之类的预言，但是至今也还未见能挑起大梁的后起之秀出现，我等俗人还是先顺应当前的潮流吧。而build镜像则是如何把docker使用好的占比非常大的一块内容，今天的话题也是始于群里面一个如何能更快的build docker image的问题。

## Best Practice
其实我们并没有讨论什么坏实践，更多的是在围绕我们觉得好的实践是什么和疑问来展开的，这次我就不按照讨论的节奏来总结了，直接上汇总的结果了。总体来讲，build镜像的目标就是尽可能的**让镜像小并能兼顾高效和安全**。

### 官方的best practice

[Docker官方有一个best practice的文档](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)，这个里面写了一些非常好的通用的实践，估摸着大部分人应该都看过了，我在这里摘抄一下核心的内容，鉴于文档里面都有详细的说明和例子，我简单的说一下我的理解。

- **构建可随时终止的容器(Create ephemeral containers)**，这一点咋看起来似乎和build image没有什么关系，但是其实是一个大的宗旨，就是说在build镜像的时候，你要以镜像将来创建出来的容器是可随时终止和替换的为目标，就和[12因子](https://12factor.net/zh_cn/)应用里面对于[进程](https://12factor.net/zh_cn/processes)的定义类似，容器只是业务运行的一个临时栖所，你不应该把有状态或者需要传承的东西放在镜像里面。
- **使用.dockerignore排除不相干的文件(Exclude with .dockerignore)**，docker是一个C/S架构的应用，build这个过程是在S端进行的，所以build的时候C端会把当前目录的文件当作[build context](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#understand-build-context)传给S端，如果传了很多在build image的过程中用不到的文件，就会有无谓的性能和时间上的损失，所以合理的使用.dockerignore文件排除不相干的文件。
- **使用多阶段构建(Use multi-stage builds)**，这个不多讲，很多的程序是或者业务最终需要的其实是一些编译的产出物，但是编译的过程会需要很多在运行时不会用到的依赖，多阶段构建就是为了解决这个问题的。
- **只安装必须的文件包(Don’t install unnecessary packages)**，这个不解释。
- **解耦业务(Decouple applications)**，这个说白了就是docker设计的核心理念，一个容器应该只有一个进程，现实中可的业务需求真的很难完全做到这个，但是尽量吧。
- **最小化层数(Minimize the number of layers)**，这一点，我个人其实持保留态度，因为他和接下来的使用Cache这个在某种程度上是有冲突的，具体根据情况做取舍吧。
- **使用Cache(Leverage build cache)**，这个作为如何快速的构建镜像的金律，人人都知道，不多说。
- **使用多行参数并排序(Sort multi-line arguments)**，程序员的基础素养，写清晰易读的代码。

### 我们的best practice

除了官方的之外，我们在平时使用中也有一些适合自己的实践，列在下面供大家参考。

- **尽可能使用Alpine版本的基础镜像**，这个可以理解，小嘛，但是里面的工具和服务都是定制过的，通用性相对差一些，和其他的流行的版本还是有一些差别的，可能需要你对使用到的工具的参数，服务的配置做一些处理。

- **锁死镜像的版本**，非精细化的版本tag其实是会随时更新的，比如`ruby:2`，你也不知道哪天它会从2.1升到2.2了，很可能这个变化就导致了不期望的或者和之前不一致的结果，所以有一个办法就是带上镜像的sha256签名，比如`ruby:2@sha256:15083783ce61a90002eb175a0de2c198afb74b49bd87d52d329e2f4a57b21562`，这样你永远拿到的都是那个唯一的版本，如果你需要知道某一个tag的最新的sha256，推荐一个小工具 [dfresh](https://github.com/realestate-com-au/dfresh)。不过这个实践你要灵活使用，比如说我的项目需求是即使这个代码库已经没有新的feature开发了，但是还是需要定期的更新语言和依赖到最新的版本，以确保我的代码不是僵尸代码，而且是有最新的安全补丁的，这种情况下，你可能也需要类似于dfresh这样的工具更新基本镜像的实际版本，或者说如果你能有办法知道某一个workable的基础镜像的sha256也是可以的。

- **干净的依赖安装**，这个可能是很多人都没有关注到一个点，比如说我们使用的是nodejs，大部分人的Dockerfile中相关的部分可能会是下面这样的。

  ```dockerfile
  FROM node:10
  
  COPY . .
  RUN yarn install
  ```

  这个看起来好像没有什么问题，但是如果你的dockerignore文件如果没有处理好，有可能就会把本机的node_modules在安装之前就一起copy进去了，这样其实你就污染了build环境，同样的代码，别人打出来的包和你的包有可能是不一致的，即使看起来我们也使用了lock文件。推荐的写法是：

  ```dockerfile
  FROM node:10
  
  COPY yarn.lock .
  RUN yarn install
  COPY . .
  ```
  
  甚至在有必要的时候使用Volume这个黑魔法。

- **必要的时候使用Volume**，Volume是一个你掌握好了就像金手指开挂，掌握不好就给自己挖了一个大坑一样的东西，所以我个人建议你在使用之前去了解一下Volmue的原理，可以去看一下队长之前的[容器化的课程](https://drive.google.com/file/d/1cbqEHxQlvVqYXs5RLZ7f3gOCw0pfV2d-/view?usp=sharing)中相关的部分，从大概1小时10分时开始。简单的来讲，在dockerfile中，你可以使用它锁死某个目录的状态，比如说下面的这个dockerfile，你猜你用它生成的镜像起一个容器的时候，/app目录下面都有什么文件，内容是什么？

  ```dockerfile
  FROM alpine:3
  
  WORKDIR /app
  RUN echo 1 > file
  VOLUME /app
  RUN echo 2 > file && touch new
  ```

  如果不知道的话自己试一试吧，然后在需要的时候合理的利用这个黑魔法。

- **不要使用root帐号**，这个其实是安全实践中的[最小化权限原则](https://en.wikipedia.org/wiki/Principle_of_least_privilege)，在容器中使用root用户是有一定程度的安全风险，尤其是如果使用了privileged模式的情况下，[这篇文章](https://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b)给了一个栗子。

- **不要把ssh key保留在镜像中**，这个也是一个安全方面的风险，私钥打包在镜像中和把密码明文放在代码库里面基本上是一个性质的。这个需求多是源于要连接一个私有的仓库之类做一些事情，一种解决方法是把本地的key文件或者目录绑到容器的对应目录，这种方式会受限于key文件的命名规则。另外一种方式是把key文件的内容当作arg传入，这种方式需要你在生成临时key文件并在同一层做完操作然后删除，否则这个文件就保留在镜像中了。还有一种办法是使用docker的[buildkit](https://docs.docker.com/develop/develop-images/build_enhancements/#using-ssh-to-access-private-data-in-builds)来做，但是这个目前还处于测试阶段，另外注意在pipeline中使用时开启静默模式，否则你的pipeline的log会很难看。

- **慎用profile/rc文件**，这类文件是在使用tty/pty/console，有用户session的时候才会加载的文件，所以这里面配置的东西在build的时候是不会有的，这种情况下，你可能会很迷惑，明明我用这个镜像起一个容器能看到这些配置生效了，为什么在build的时候就不生效呢？
- **慎用latest tag**，latest是默认的一个tag，它代表的我最近一次打出来的镜像，并不是代表这个可用的最新的版本，因为有可能最新的这次build的image是有问题的，而且很大程度上说，也很难定位当前的latest对应的等价的tag是什么。如果造化弄人，那些小概率事件让你碰到，比如说有可能有一些粗心大意的小伙伴会每次都把latest镜像push到registry，而这个latest是有问题的，如果你的业务这个时候重新拉了latest tag的镜像，可能画面就会很美好了。
- **使用WORKDIR而非cd命令**，cd用多了会产生混乱，尤其是使用相对目录的情况下，调整顺序绝大部分情况下会产生错误。

## 其他

- **如何更快的build镜像**，这个是最初引发这个讨论的让人头疼的主题，但是追根究底来讲其实是一个网速问题，因为我们在安装和编译过程中是会下载大量的依赖的。所以，目前用的比较多的方法有二。使用cache或私有仓库是一个方法，不论是docker的cache还是agent的cache，亦或是一个proxy的cache也可以，实在不行创建自己私有的包管理仓库吧，这也是很多企业最后的选择，省时间也省流量。另外一个办法就是做一个base image，但是这个取决于所需依赖和安装包的更新频率，很多的时候，一些团队采用的折衷策略就是把这个base image做一个nightly的更新，但是不管怎么样，都是有额外的管理成本在其中的，自己取舍吧。
- **一些操作到底是放在dockerfile中还是entrypoint的脚本中？** 这个只能说是个人偏好了，没有所谓的对错，震宇举了一个很好的例子，就如同jenkinsfile一样，你是把命令直接写在step里面，还是写在一个脚本里面然后调用它。我个人坚持步骤和逻辑分离的原则，在dockerfile中我更倾向于尽量少的做逻辑处理和配置相关的操作，这些内容放在初始化的脚本里面更符合我的习惯，除了逻辑部分集中之外，另外一个好处是在测试改动的时候不需要重新build镜像，把这个文件挂载进容器就可以了，anyway，使用你自己舒服的方式来做事。


## 参考资料

- [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/), Docker官方的Best Practice
- [12因子应用](https://12factor.net/zh_cn/)
- [dfresh](https://github.com/realestate-com-au/dfresh)
- [基于容器化的DevOps实践](https://slides.com/phoenix_zhu/deck)
- [TOC容器化课程](https://drive.google.com/drive/folders/1oMAmflF8LVHkkGuLEHfnR6MOPeAxtcEm)
- [Build images with BuildKit](https://docs.docker.com/develop/develop-images/build_enhancements/)
- [最小化权限原则](https://en.wikipedia.org/wiki/Principle_of_least_privilege)
- [Processes In Containers Should Not Run As Root](https://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b)
