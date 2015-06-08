# gitlab-docker-ci
基于docker 与 gitlab CI 做一个简单够用的持续集成环境

本次持续集成环境主要构成

1. 代码版本库 Gitlab 
2. CI服务 Gitlab CI
3. CI运行器  GitLab Runner-docker (项目地址 https://gitlab.com/gitlab-org/gitlab-ci-multi-runner)

搭建过程中的主要步骤

1. 在Gitlab上添加代码库
1. 把代码库添加到Gitlab CI中
1. 安装docker
1. 搭建基础 docker image
1. 设置Gitlab Runner-docker
1. 编写测试脚本
1. 编写构建脚本
1. 优化过程
1. 完成一个简单够用的持续集成环境

前面3个步骤较简单，并且网上教程较多，这里不做赘述。

## 搭建基础docker image
Gitlab Runner-docker 默认镜像是Ruby2.1，但是Ruby这门语言我不会，并且我的代码环境不是Ruby所以很多时候就要我们自己去DIY这个基础的运行镜像。

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

##设置Gitlab Runner-docker
Good job，基础镜像我们已经完成了。我们进行下一步。
来来，我们的参考资料 https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/blob/master/docs/install/docker.md
上面给了我们两种存储数据的方式一个是直接存储在本地，另一个是创建docker data container 。（好像第二种比较高端。貌似字数也能多一点。我们就用第二种吧。）
```
docker run -d --name multi-runner-data -v /data busybox:latest /bin/true

docker run -d --name multi-runner --restart always \
    --volumes-from multi-runner-data \
    ayufan/gitlab-ci-multi-runner:latest
```
windows上的同学们，我们要注意`\`，把代码改成同一行运行。

因为我们在这个容器里面还需要运行docker，把命令修改一下
```
docker run -d --name multi-runner --restart always \
     -v /var/run/docker.sock:/var/run/docker.sock \
    --volumes-from multi-runner-data \
    ayufan/gitlab-ci-multi-runner:latest
```
这样我们就能在docker里面运行docker

```
docker exec -it multi-runner gitlab-ci-multi-runner register

Please enter the gitlab-ci coordinator URL (e.g. http://gitlab-ci.org:3000/ )
https://ci.gitlab.org/ 
这里我用官网地址，也可以是自己搭的gitlab-ci地址
Please enter the gitlab-ci token for this runner
这里填gitlab-ci的token 
Please enter the gitlab-ci description for this runner
你的runner的名字
INFO[0034] fcf5c619 Registering runner... succeeded
Please enter the executor: shell, docker, docker-ssh, ssh?
docker
Please enter the Docker image (eg. ruby:2.1):
yeting/laravel-prebuild-docker 
这个是我自己搭的镜像名字，上传到了docker hub上了，

还有一些其他配置 一律回车 过！

INFO[0037] Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
看到这句话就成功了
```
在Github CI 的项目里面的runners选项，看到了一个新的runner，就是那个了。在这个页面可以进行激活、停止等操作

##编写测试脚本
在Github CI 的项目里面的jobs选项，我们看到 Test 和 Deploy。

Test 的脚本执行`success`之后 就会执行 Deploy的。 不然是不会执行的。

里面还有一些功能大家自行摸索。
```
ls -la

composer config -g github-oauth.github.com  XXX // 个人的github token 这个有一个github上面的坑，限制了非认证用户下载dist包的次数，60次/小时。这个限制很容易就超了。属于优化部分
composer config cache-dir /cache/composer // 修改composer 缓存目录 属于优化部分

composer install  // 依赖包安装
phpunit // 单元测试
```

##编写构建脚本
测试完成之后我们就可以部署了。
```
ls -al
ls -al ~/.ssh
envoy run deploy
```
别看这脚本这么这么简单，其实还是有很多事情要做的。

我们在本地需要构建一个 Deploy的镜像，也是为了保密，咱总不能什么东西都往网上丢吧。
目录结构
```
- Dockerfile
- Envoy.blade.php
- ssh
-- config 
-- id_rsa //这个不用说了吧，一个密钥
```
ssh/config 里面的内容
```
ost *
   StrictHostKeyChecking no
```
Envoy.blade.php 具体语法 http://laravel.com/docs/5.0/envoy
```
@servers(['web' => 'ubuntu@xxx.xxx.xxx.xxx'])

@task('deploy')
    cd xx
    git pull
    docker-compose build
    docker-compose up
@endtask
```
Dockerfile
```
FROM yeting/laravel-prebuild-docker:latest 
MAINTAINER YeTing <me@yeting.info>
Copy ./ssh /root/.ssh // 覆盖该docker 容器的ssh
```
这样的我们的构建脚本就完成了80%了，还有什么？
我们还需要在我们的代码库中添加docker-composer.yml
 
docker-composer 具体文档 https://docs.docker.com/compose/
docker-composer 成功之后

好了 这样我们部署脚本就完成了。
