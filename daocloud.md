# DaoCloud-docker-ci
基于docker 与 DaoCloud CI 做一个简单够用的持续集成环境

本次持续集成环境主要构成

1. 代码版本库 coding.net 
2. CI服务 DaoCloud
3. CI运行器  DaoCloud

搭建过程中的主要步骤

1. 在coding.net上添加代码库
1. 申请DaoCloud自有主机服务（目前在内测中需要申请）
1. 安装docker
1. 把代码库添加到DaoCloud生成项目
1. 搭建基础 docker image
1. 编写daocloud.yml
1. 发布镜像
1. 完成一个简单够用的持续集成环境

## 在coding.net上添加代码库
这步不是重点，我只是随便选了一个daocloud支持的代码源。

目前（2015-06-14）DaoCloud支持的代码源是:

1. Github
1. Bitbucker
1. Coding
1. GitCafe

期间有个小插曲刚注册coding在激活用户的时候出现一个Bug。
Bug真的是无处不在。

## 申请DaoCloud 自有主机 服务
DaoCloud自有主机官方介绍
[DaoCloud邀请您体验全新混合式容器主机管理服务](http://blog.daocloud.io/daocloud_sr_alpha/)

完成申请之后自由主机申请之后 
[安装主机监控程序](https://dashboard.daocloud.io/runtimes/new)

应该是需要登陆以及具有自有主机服务的权限才能进入上面链接。我直接把命令复制下来
```
curl -sSL https://get.daocloud.io/docker | sh // 安装docker

curl -sSL https://get.daocloud.io/daomonit/install.sh | sh -s XXXXXXXX //这是自有主机的标识，系统随机生成

sudo service docker restart
```
完成上面步骤之后我们就能在Daocloud/我的主机 看到我们已经链接上的主机信息
包含`主机ip`、`CPU使用率`、`内存使用率`、`磁盘使用率`、`主机上的容器`

`主机上的容器`这个功能真的是太吊了，至于多么吊，大家自己试试看就知道了。所有信息一目了然 有木有!
## 新建DaoCloud项目
起项目名，绑定项目代码源，一个项目就建好了。还是很简单的。

首次生成项目的时候会默认拉取`master`分支，构建一个`master-init`的镜像，只要构建成功，我们就能把这个镜像部署到我们自己的主机上。这个部署可比我上一篇 [基于docker 与 gitlab CI 做一个简单够用的持续集成环境](https://github.com/Ye-Ting/docker-ci/blob/master/gitlab.md)中的部署，简单好多，轻轻一点就行。构建是在DaoCloud服务器上做的，据说他们还备好了梯子给我们拉第三方库的时候好翻墙。

记得在部署的时候选择自己关联到DaoCloud的主机，不然默认会部署在DaoCloud提供的主机上。DaoCloud提供的主机也还不错，就是访问速度慢了点，胜在不用钱的。不过我们都自己搭了主机，当然还是部署到我们自己的主机上是最妥的。

## 搭建基础docker image
DaoCloud 提供了一些基础镜像用于搭建测试环境，貌似他们的提供的环境不是我想要的额，所以很多时候就要我们自己去DIY这个基础的运行镜像。自己DIY还是有蛮多好处的。

1. 每次下的库就那么几个测试用的第三方库就固定那么几个，每次都要下，先不说其他，下载库的时间要等吧？做个基础镜像，也帮DaoCloud省省带宽，重点是我们自己的程序在构建的时间跑的快。不然每次都要等个10+分钟，对于我这种性子急得人真是要命。
2. 自己做的镜像里面有啥玩意自己都知道，能hold住。有坑自己也能填上。
3. 酷，自己搭的镜像逼格比用别人搭的高。

由于我的代码环境是PHP，我这里就拿PHP来举例子，来构建PHP运行及测试环境。

No Code No BB，直接上 Dockerfile:
```
FROM php:5.6-apache
MAINTAINER YeTing <me@yeting.info>

#Download Require
RUN apt-get update && apt-get install -y libmcrypt-dev libz-dev git wget \
	&& docker-php-ext-install mcrypt mbstring zip \
	&& curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer 

ENV PATH=$PATH:~/.composer/vendor/bin

RUN composer global require "phpunit/phpunit=~4.0" "phpspec/phpspec=~2.1" "laravel/envoy=~1.0" && a2enmod rewrite
```
这个dockerfile很简单

武林秘籍第一式：找一个靠谱的镜像
```
FROM php:5.6-apache
MAINTAINER YeTing <me@yeting.info>
```
https://registry.hub.docker.com/ 上面有很多官方的镜像，我自己DIY的镜像就是基于官方`php:5.6-apache`镜像来做的。
个人镜像多多少少会有一些坑等着我们去踏，我们还是踏踏实实的根据我们的需求自己写一个Dockerfile。不过别人写的都可以借(chao)鉴(xi)。

武林秘籍第二式：精确定位
```
RUN apt-get update && apt-get install -y libmcrypt-dev libz-dev git wget \
	&& docker-php-ext-install mcrypt mbstring zip \
	&& curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer 
```
要啥就下啥，知道我们每次下载的东西是拿来干什么的，不然为什么要下呢。
`docker-php-ext-install`是官方镜像中提供的一个配置PHP环境的方法，所以文档我们还是要多看看的。不然好东西都发现不了。`composer`是一个PHP的包管理工具，类似于node与npm的关系。

武林秘籍第三式：欲练此功，必先利其器
```
ENV PATH=$PATH:~/.composer/vendor/bin

RUN composer global require "phpunit/phpunit=~4.0" "phpspec/phpspec=~2.1" "laravel/envoy=~1.0" && a2enmod rewrite
```
前人的经验真是数不胜数，前人创造的工具我们也好好好利用。`phpunit` `phpspec` 是PHP上的测试工具。`envoy`一个简易的部署工具，我们在后面的部署会用到。`a2enmod rewrite`apache开启url重写模块。

基础镜像终于好了，我们的测试，部署可都是要靠它了。

## 编写daocloud.yml
在DaoCloud代码测试需要用到DaoCloud。所以我们要先来看看官方的介绍。
[daocloud.yml的格式](http://help.daocloud.io/v1.0/docs/daocloud-yml)
和 [官方的例子](https://github.com/DaoCloud/daocloud-doc/blob/master/DaoCloudCI.md)

看完了之后，发现还是格式蛮简单的，例子好长，都快看不下去了。就一个重点

### How it works
1. 设置环境变量
1. 执行install脚本
1. 克隆源代码，切换对应的commit
1. 执行before_script脚本
1. 执行script脚本

做好前期准备，我们就来写自己项目的daocloud.yml
```
image: yeting/laravel-prebuild-docker // 我们的基础镜像

env:
    - MYENV = "hello" // 纯粹为了测试代码执行

install:
    - echo $MYENV

before_script: //执行测试前的准备
    - composer config -g github-oauth.github.com XXXX 
// 个人的github token 这个有一个github上面的坑，限制了非认证用户下载dist包的次数，60次/小时。这个限制很容易就超了，超了之后下的库超慢
    - composer install
// composer依赖包安装 composer是php上的包管理工具类似于npm与node的关系

script: // 执行测试
    - phpunit //php单元测试

```

写了daocloud.yml，DaoCloud会在你每次的代码push的时候，进行测试。

不过有点略坑的是，每次测试通过的时候，他不会帮你自动build，需要自己build。
就这一点，联系DaoCloud工作人员。他们说，并不是每一次代码的push都需要build成镜像的，如果需要build，需要在该代码版本上打一个tag，这样DaoCloud才会把代码build成镜像。这样也能解决我们自动build的需求，也还行。

部署的时候就点击打包好的镜像，发布到我们自己的主机上，就可以了，这点真的很赞，好方便。DaoCloud还帮你存了好多个镜像，供你随时切换着版本部署，想想他们也不容易啊。

##总结
一个简单够用的持续集成环境，就这样完成了，真不敢相信，是不是很简单？太酷了。

DaoCloud真的的是一个贴心的开发者伙伴。希望他们在未来的路上走的更远。

Docker是一项好技术，但是我们要以正式的姿势去使用它。

要做一个能hold住技术的开发者，而不是盲目跟随潮流最求新技术的人。

工具能提高人的效率，而产品的还坏最终还是在于人。

这是我个人的第一个系列的技术博客，希望能与更多的人进行交流。

本篇博客为个人原创，转载请联系 me@yeting.info
