---
title: spring config server + docker + gitlab 实现持续集成部署
date: 2018-05-18 21:06:19
categories: spring-cloud,docker
tags: spring-cloud,docker
---

#  spring config server + docker + gitlab 实现持续集成部署

> 优信支付系统使用spring cloud + docker作为技术选型。配置中心使用spring cloud server。使用docker进行服务部署，使用gitlab CI 进行服务的持续集成持续部署。本文将介绍优信支付如何基于spring config server + docker + gitlab实现构建部署不同环境微服务选择对应环境的配置文件。

## spring config server 

spring config server 是spring cloud组件中的配置文件中心。在微服务的建构中config server将所有微服务的配置信息集中管理。将配置文件与微服务分开，使得服务与配置之间不在强耦合。

spring config server 本身就是一个微服务，作为微服务本身spring config server可以注册到eureka注册中心，提供给注册到eureka的微服务进行调用并且获取微服务配置信息。

作为spring config server可以读取本地的保存的微服务的配置文件，也可以读取git仓库的微服务的配置文件．我们采用了git仓库管理配置文件的方式来管理微服务的配置文件．配置如下：

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: xxx
          search-paths: /
          username: xxx
          password: xxx
```
- uri：配置文件git仓库地址
- search-path: 检索配置文件的相对路径
- username和password：登录仓库的用户名密码

config-server会从配置的git地址进行pull，拉取下来的配置文件会保存在本地．默认是/temp目录下，这里面有一个坑的地方就是这个默认的保存目录．因为是保存在/temp目录下面，linux系统会定时的清理temp目录所以会造成应用无法从config server获取配置信息，在spring cloud config的官方文档里面给出了对应的解决办法．

> With VCS based backends (git, svn) files are checked out or cloned to the local filesystem. By default they are put in the system temporary directory with a prefix of ``config-repo-``. On linux, for example it could be ``/tmp/config-repo-<randomid>``. Some operating systems routinely clean out temporary directories. This can lead to unexpected behaviour such as missing properties. To avoid this problem, change the directory Config Server uses, by setting ``spring.cloud.config.server.git.basedir`` or ``spring.cloud.config.server.svn.basedir`` to a directory that does not reside in the system temp structure. 

官方的解决方案是是通过配置来改变存储地址．

## 微服务配置文件

config server 从指定的git仓库中获取微服务配置信息，对于配置文件的管理方式，我们使用不同分支保存不同环境的配置文件．以下是我们的配置文件的分支．

![001](http://wx1.sinaimg.cn/large/74b07056ly1fr59ar0u7tj20fi02qglj.jpg)

我们有４个分支：
 - dev 生产环境配置文件
 - test 测试环境配置文件
 - stage 预发环境配置文件
 - master 生产环境配置文件

这样我们就可以checkout不同分支修改不同环境的配置信息．另外配置文件的仓库是不允许任何merge操作的．

## 微服务配置文件的选择

config server 搭建完成，配置文件的git仓库也搭建完成之后，我们可以在微服务的bootstrap.yml配置文件中配置．

```yaml
spring:
  application:
    name: service-a
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      label: dev
```

- service-id config server 注册到eureka的服务名称．
- label 获取那个分支的配置文件．

这样配置之后config server 会根据service-a提交的label+name信息从配置文件仓库中获取对应的配置文件的信息．但在实际开发过程中我们一个微服务有四个环境．所以我们在bootstrap.yml配置了四个环境的配置信息．

```yaml
spring:
  profiles:
    active: dev

---
# 开发环境
spring:
  profiles: dev
  application:
    name: xxx
    index: ${random.uuid}
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      label: dev
eureka:
  client:
    service-url:
      defaultZone: http://dev:8000/eureka
---
# 测试环境
spring:
  profiles: test
  application:
    name: xxx
    index: ${random.uuid}
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      label: test
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
---
# 预发布环境
spring:
  profiles: stage
  application:
    name: xxx
    index: ${random.uuid}
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      label: stage
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
---
# 生产环境
spring:
  profiles: prod
  application:
    name: xxx
    index: ${random.uuid}
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      label: master
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
```

``spring.profiles.active`` 表示使用哪个环境配置文件，``spring.profiles``表示当前的配置文件是什么环境．在yaml语法中``---``相当于分割两个配置文件．在第一个分割文件中我们默认使用了``spring.profiles.active:dev``开发环境的配置文件．我们也可以通过启动参数传递的方式进行配置文件的切换．

```shell
java -jar service-a.jar --spring.profiles.active=dev
```

我们使用了通过启动参数传递的方式选择不同配置文件．

## docker的配置

优信支付所有的微服务全部使用docker进行部署．我们制作了基础镜像，所有微服务的镜像均基于这个基础镜像进行再制作．我们将``spring.profiles.active``的值通过环境变量的方式配置进去．

```dockerfile
FROM base-jre-8u151:1.0.1
MAINTAINER wangliping<wangliping@xin.com>
ENV JAVA_OPTS -Xms128m -Xmx256m
ENV ACTIVE dev
VOLUME /data
COPY target/service-a.jar /service-a.jar
EXPOSE 8090
CMD java ${JAVA_OPTS} -jar /service-a.jar --spring.profiles.active=${ACTIVE}
```

可以看到dockerFile文件中ENV字段，我们设置了ENV ACTIVE dev 的环境变量,这样在启动docker后，执行CMD命令时，会从系统获取${ACTIVE}这个变量的值．默认我们设置了dev环境．指定的环境变量可以在运行时被覆盖掉，如``docker run --env <key>=<value> image``

## Gitlab CI/CD

从 GitLab 8.0 开始，GitLab CI 就已经集成在 GitLab 中，我们只要在项目中添加一个 .gitlab-ci.yml 文件，然后添加一个 Runner，即可进行持续集成。 而且随着 GitLab 的升级，GitLab CI 变得越来越强大，我们也是在gitlab上完成ci,cd．

### 一些概念

#### Pipeline

一次 Pipeline 其实相当于一次构建任务，里面可以包含多个流程，如安装依赖、运行测试、编译、部署测试服务器、部署生产服务器等流程。我们可以在.gitlab-ci.yml中定义那些分支的提交会触发响应的任务．如下图所示

```
+------------------+           +----------------+
|                  |  trigger  |                |
|   Commit / MR    +---------->+    Pipeline    |
|                  |           |                |
+------------------+           +----------------+
```

#### Stages

Stages 表示构建阶段，说白了就是上面提到的流程。
我们可以在一次 Pipeline 中定义多个 Stages，这些 Stages 会有以下特点：

- 所有 Stages 会按照顺序运行，即当一个 Stage 完成后，下一个 Stage 才会开始
- 只有当所有 Stages 完成后，该构建任务 (Pipeline) 才会成功
- 如果任何一个 Stage 失败，那么后面的 Stages 不会执行，该构建任务 (Pipeline) 失败

因此，Stages 和 Pipeline 的关系就是：

```
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

Jobs 表示构建工作，表示某个 Stage 里面执行的工作。
我们可以在 Stages 里面定义多个 Jobs，这些 Jobs 会有以下特点：

- 相同 Stage 中的 Jobs 会并行执行
- 相同 Stage 中的 Jobs 都执行成功时，该 Stage 才会成功
- 如果任何一个 Job 失败，那么该 Stage 失败，即该构建任务 (Pipeline) 失败

所以，Jobs 和 Stage 的关系图就是：

```
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

#### gitlab runner

配置的这些job都是由gitlab runner来执行的，为什么不使用gitlab ci执行job是因为执行这些job是很消耗系统资源的，而 GitLab CI 又是 GitLab 的一部分，如果由 GitLab CI 来运行构建任务的话，在执行构建任务的时候，GitLab 的性能会大幅下降。GitLab CI 最大的作用是管理各个项目的构建状态，因此，运行构建任务这种浪费资源的事情就交给 GitLab Runner 来做．

#### .gitlab-ci.yml配置文件

首先我们先看看gitlab-ci文件应该怎么写

```shell
# 定义 stages
stages:
  - build
  - package
  - deploy
# mavne package job
maven-build:
  stage: build
  script: "mvn clean package"
# docker build job
docker-build:
  stage: package
  script:
    - echo 'Building...'
    - docker build -t service-a:latest .
# docker deploy job
docker-deploy:
  stage: deploy
  script:
    - echo 'Deploing...'
    - docker run --env ACTIVE=test service-a:latest
  only:
  - test
```

1. 声明stages,声明pipeline中有几个stages.我这里生命了三个阶段，分别是：

- build 项目构建阶段
- package 构建docker镜像
- deploy 部署运行docker容器

2. 定义各个stage的job,这里我声明对应stage的三个job,当然你也可以一个stage声明多个job:
3. script描述了这个job需要执行的命令
4. 最后我们可以看见在任务```docker-deploy```任务中，有个only标签，它的作用是在test分支提交下触发．这样我们就能做到推送不同分支构建部署不同环境的容器．

在实际生产环境下，gitlab-ci.yml文件的配置会更加复杂，这里只是demo演示．因为我们生产环境使用了kubernetes．涉及到工程构建，镜像构建，推送私有进行仓库，从镜像仓库拉取镜像发布到kubernetes等操作．

