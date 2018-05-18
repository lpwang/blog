---
title: maven 部署应用到远程私有nexus仓库
date: 2018-05-18 21:05:50
categories: maven
tags: maven
---

# maven 部署应用到远程私有nexus仓库

> 写这篇文章是想记录一下遇到的坑，某天下午部署一个我写的工具jar到我们公司的远程nexus仓库的时候失败，前后折腾了一下午才解决．文章将从nexus搭建开始到jar部署．

## nexus配置

1. 首先下载nexus到本地或者开发机安装,我下载到我们的开发机进行安装．
2. 解压之后进入bin目录进行执行 ``sh nexus run``进行安装,我们也可以编写service文件使用systemd进行管理，这里我使用终端进行启动．
3. 启动之后nexus默认的监听端口是8081,所以我们访问nexus后台管理页面，使用这个地址``http://dev:8081``
4. 点击右上角进行登录，默认的用户名密码是
  - username: admin
  - password: admin123

5. 登录之后我们新创建一个测试仓库用来测试本地应用打包上传,创建仓库的时候需要配置：

  - 仓库类型选择：Maven 2 (hosted)
  - Deployment policy : allow redeploy

![001](http://wx4.sinaimg.cn/large/74b07056ly1fra3p0f7msj20hr0gowf3.jpg)

6. 测试仓库创建完成后,我们使用admin用户进行本地应用上传，当然你也可以创建一个部署用户，限制用户的权限．

## pom配置

1. 在项目的pom.xml文件中添加如下配置：

```xml
    <distributionManagement>
        <repository>
            <id>uxinpay-releases</id>
            <url>http://dev:8081/repository/uxinpay-releases/</url>
        </repository>
    </distributionrepositoryManagement>
```

- id指的是在创建nexus时仓库的repository name

## setting配置

1. 首先，在用户环境下的setting.xml文件添加一下配置信息．

```xml
  <servers>
    <server>
      <id>uxinpay-releases</id>
      <username>uxinpay</username>
      <password>123456</password>
    </server>
  </servers>

```

- 其中id与项目pom.xml配置文件中的```<distributionManagement>```中的id保持一直．

2. 执行 mvn deploy 部署到远程仓库中．但是这个时候会出错．报错如下：

![002](http://wx3.sinaimg.cn/large/74b07056ly1fra3snch08j210b02amxs.jpg)

- 错误指出用户验证错误，why?用户名密码和id我又检查了一遍，没有问题．

3. 解决办法，我将server的配置文件添加到全局的setting.xml的配置文件中.全局setting.xml配置文件在maven安装目录下的conf/setting.xml．

4. 修改后执行mvn deploy，上传成功．

