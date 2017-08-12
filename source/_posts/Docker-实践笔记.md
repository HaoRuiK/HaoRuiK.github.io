---
title: Docker 实践笔记
date: 2017-04-28 15:40:27
tags:
---

#### 了解Docker

> 一句话介绍：
>
> ​	docker是一个用来装应用的容器
>
> What is Docker ?
>
> ​	Docker is the world's leading software container platform.
>
> ​	世界领先的软件容器化平台。a

<!-- more -->

#### Docker 思想

- 集装箱
- 标准化
  - 运输方式	
  - 存储方式
  - API接口
- 隔离

#### Docker解决了什么问题？

- 运行环境不一致带来的问题。
- 隔离运行，防止程序受别人程序影响。
- 让快速扩展，弹性伸缩变得简单。

### 走进Docker

#### Docker的核心技术

> 运行过程：
>
> ​	去仓库把镜像下载到本地，然后用命令吧镜像运行起来，变成容器。

- 镜像 —>集装箱—>Build
- 仓库—>超级码头—>Ship
- 容器—>运行程序的地方—>Run

#### Docker镜像

> 多个文件的集合

- 格式：
  - 联合文件系统：
    - 可以将不同的目录挂载到同一个文件系统下。
    - 即：可以在同一个目录下可以看到多个文件夹的集合
    - docker镜像就是利用这种分层的概念，来实现的镜像存储。

#### Docker容器

> 本质就是一个进程，可以想象为一个虚拟机。

​	容器是可以修改的，镜像是不可修改的。

​	所以一个镜像可以生成多个容器，独立运行。

#### Docker仓库

中央仓库

- hub.docker.com
- c.163.com

#### 安装Docker

#### mac

- [下载地址](https://download.docker.com/mac/stable/Docker.dmg 

##### Linux

- uname -r 查看当前系统版本，必须大于3.3
- su 切换至root身份
- apt-get update 更新apt应用
- apt-get install -y docker.io或者curl -s https://get.docker.com
- 输入docker version检查是否安装成功
  - 提示 are you trying to connect to a tls-enabled daemon without tls的解决方法
  - 输入：sudo service docker start

### Docker初体验

#### 第一个docker镜像

- service docker start 开启docker服务


- docker pull [Options] Name[:Tag]
  - option:拉取的参数
  - Tag:拉取的版本，不填写默认为最新版本
  - docker pull hello-world
- docker images '省略左中括号' Options][Repository[:Tag]]

#### 第一个docker容器

- docker run hello-world

#### docker运行Nginx镜像

- 拉取镜像
  - docker pull hub.c.163.com/library/nginx:latest
- 运行镜像
  - docker run -d hub.c.163.com/library/nginx
- 显示运行容器
  - docker ps 
- 进入容器内部
  - docker exec -it [Option] bash
  - 就是进入到了一个虚拟机内部

### docker网络

- 开启容器端口映射
  - docker run -p  
    - -p 开放一个端口到主机上，默认为空
    - -P 开放所有端口到对应的随机端口
  - docker run -d -p 8080:80 hub.c.163.com/library/nginx
    - 8080:主机端口
    - 80:容器端口

#### 第一个javaweb项目

- [下载Jpress](https://github.com/JpressProjects/jpress/raw/master/wars/jpress-web-newest.war)到本地

- 下载tomcat镜像到本地

  - docker pull hub.c.163.com/library/tomcat:latest

- 创建Dockerfile 文件

  - 输入

    ```tex
    # 继承自哪一个基础镜像 --> javaweb程序肯定会用到tomcat
    from hub.c.163.com/library/tomcat
    # 镜像的所有者姓名和联系方式
    MAINTAINER haorui konghaoruis@gmail.com
    # 复制外部应用文件到镜像指定位置
    COPY jpress.war /usr/local/tomcat/webapps
    ```

    ​

- 生成镜像文件

  - docker build -t  jpress:latest .(**注意最后有一个空格和点！**)
    - -t:镜像指定Tag

- 创建容器

  - docker run -d -p 8081:8080 jpress

- 下载mysql镜像

  - docker pull hub.c.163.com/library/mysql:latest

- 运行mysql镜像 

  - docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=jpress hub.c.163.com/library/mysql:latest
    - -e：环境变量 
      - MYSQL_ROOT_PASSWORD:数据库用户密码
      - MYSQL_DATABASE:数据库名