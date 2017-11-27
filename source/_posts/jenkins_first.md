---
title: 初探jenkins
date: 2017-11-27 14:32:22
categories: jenkins
tags: jenkins
---

# 初探jenkins

## 缘起

> 由于项目中使用SpringCloud作为整体开发架构，虽然微服务的架构可以更好的解耦，针对不同的业务可以使用更加适用的技术栈来解决问题，对局部进行扩容也很简单。随然微服务架构相对单体架构有这样那样的好处，但是也有他自身问题。那就是部署方面过于麻烦。由于最近项目需要向线上环境进行部署。从release版本号到构建再到部署，所有的微服务都需要进行相同的动作。在微服务不多的情况下这样重复的动作是没有问题的。但是随着项目的深入我们的微服务越来越多，每次部署都忙一天，而且有大量的重复工作。在完成上次部署上线之后，我深深的感觉微服务搭配jenkins是多么爽的一件事情，这边提交，那边持续集成，持续部署。由于项目在测试环境搭建了jenkins，而在开发和线上环境并没有部署，所以我打算在开发环境部署一套，运行ok之后，再部署到线上环境，解决线上部署的问题。

## 安装

本机是linux,所以直接使用linux的方式安装jenkins.(以下是ubuntu发行版,其他centos发行版请自行做相应的调整)

1. 添加key到系统中

```shell
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
```

2. 添加源地址

```shell
deb https://pkg.jenkins.io/debian-stable binary/
```

3. 升级安装源索引和安装

```shell
sudo apt-get update
sudo apt-get install jenkins
```

## 启动

1. 启动jenkins
```shell
systemctl start jenkins
```

2. 访问web控制台,jenkins默认开放端口为8080.
```shell
http://localhost:8080
```

## 安装插件

1. blue ocean

blue ocean 是jenkins的pipline插件,其插件的界面和功能要比原生的pipline更漂亮更强大。我们可以从，系统管理中选择管理插件进行安装blue ocean插件。同时jenkins还支持很多插件。我们可以在插件管理中进行搜索然后进行安装。在安装blue ocean之后我们开始持续集成的配置工作。

## 编写JenkinsFile文件

首先我们需要编写JenkinsFile文件到需要持续集成的项目根目录下，这里需要强调一下jenkinsfile文件存放的位置。一定要放到项目根目录下面。否则会发生找不到jenkinsfile文件的错误。jenkinsfile文件使用groovy的语法进行编写.首先先给出示例

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
                sh 'gradle clean build'
                dir('./build/libs/') {
                    sh 'echo $PWD'
                    sh 'mv ./iot-update-*.jar iot-update.jar'
                    sh 'scp ./iot-update.jar kdxcloud@iot-update:/home/kdxcloud/devevn'
                }
            }
        }

        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying....'
		        echo '停止iot-update微服务'
                sh 'ssh kdxcloud@iot-update -tt sudo systemctl stop kdxcloud-iot-update'
		        echo '启动iot-update微服务'
                sh 'ssh kdxcloud@iot-update -tt sudo systemctl start kdxcloud-iot-update'
                echo 'I succeeeded!'
            }
        }
    }

}

```

- agent 指哪个jenkins节点执行该pipeline
- stage 指构建的阶段
- step 指某个阶段下执行的具体步骤在steps中命令的执行是串行的

## 互信配置

由于构建的执行包需要上传到远程主机上，所以需要配置互信让部署不需要人为干预。

1. ssh 免密登录

我的机器是linux，所以使用命令ssh-keygen，生成用户的公钥和私钥，然后在用户的家目录下面的.ssh文件夹下面，找到文件id_rsa.pub，里面存储的是刚刚使用命令生成的用户的公钥，然后将文件里面的内容复制并远程主机需要登录的用户的家目录下的.ssh文件夹下面的authorized_keys文件下。这样我们在本机使用ssh命令登录的时候就不需要使用密码了。

2. 将远程主机的用户添加至sudo组

当我们登录到远程主机之后，需要做一些停止现有服务，重新拉起服务的操作，然而这些操作为了安全起见都是只用root用户才可以进行操作的。当我们使用普通的用户进行这样的操作的时候我们一般都会使用sudo命令将普通用户执行该命令时候权限提升到root权限，但是这样一来我们需要使用root密码进行确认之后，命令才可以执行，这样是很不方便的，所以我们将执行重启服务命令的用户添加值sudo组里面这样就可以不需要密码确认了。在远程主机环境下使用visudo命令将编写位于/etc/sudoers的文件，然后找到`root    ALL=(ALL)       ALL`这一行然后在后面添加一行信息`wanglp    ALL=(ALL)       ALL`这样就可以了，当然我在这里偷懒了，将所有root可以执行的命令都免密的交给wanglp这个用户去执行是很不安全的，我们可以选择指定的几个命令记性免密。

在完成以上两步的配置之后就可以在执行jenkinsfile脚本的时候不需要人为的干预了.

## 添加任务

现在我们添加任务在jenkins控制台下面添加构建任务，由于我们添加了blue ocean插件，所以我们使用blue ocean插件进行任务的添加。

1. 点击左侧导航栏上的blue ocean插件

![](http://wx2.sinaimg.cn/large/74b07056ly1flwm551thij20yh0i975o.jpg)

2. 选择代码管理工具，我们项目使用的是gogs所以选择git

![](http://wx3.sinaimg.cn/large/74b07056ly1flwm55hk69j20ig0fq0tb.jpg)

3. 然后再添加项目的url地址，这里需要说明一下由于我们项目没有使用添加公钥私钥的方式来进行代码拉取和推送的验证只是使用了http+密码的方式，所有再添加url之后还需要添加git的用户名和密码。建议使用公钥私钥的验证方式。

![](http://wx3.sinaimg.cn/large/74b07056ly1flwm9muzcmj20is0k1753.jpg)

4. 点击创建pipeline，完成创建过程，需要注意的是在导入该项目之前，在该项目的根路径下一定要存在jenkinsfile文件。这样jenkins才能根据项目中的jenkinsfile文件中的脚本进行项目的集成。



5. 在点击集成任务后我们可以查看具体的执行步骤和执行情况，以及报错信息。

![](http://wx4.sinaimg.cn/large/74b07056ly1flwm56xawyj21z30nb75e.jpg)

## 结束

经过了以上的几步之后我在本地编写完代码之后push之后，再在jenkins控制台页面点击开始任务的按钮就可以完成代码的拉取，jar包上传，服务重启等步骤。简化了很多人为的重复的工作。这就是持续集成带来的好处，但是到这里也只是jenkins的冰山一角的功能，而且到现在为止也只是半自动化，还是需要进入jenkins控制台进行操作。之后的文章会介绍全自动化的持续集成，也就是说我在本地push完代码之后，jenkins自动的触发执行脚本进行项目的集成步骤。
