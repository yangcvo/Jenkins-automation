---
title: Ansible + Jenkins+Maven＋Nginx搞定自动发布，构建程序的持续集成平台
date: 2016-09-12 13:47:27
tags:
- Jenkins
- Manven
- jdk
categories: automation
---


# Ansible+Jenkins搞定自动发布，构建程序的持续集成平台

Ansible是相对简单的批量管理工具，支持模板管理等高级功能。搞定了自动发布，开发的服务器需求已经明显下降，只要把代码提交到 Git主干，就会自动触发发布。

今天有空就简单记录下用Ansible + Jenkins搞定自动发布 。这篇是主要环境安装。

### 一、什么是持续集成

**1、什么是集成**

指的是代码由编译、发布和测试、直到上线的一个过程

**2、什么持续集成**


高效的、持续性质的不断迭代代码的集成工作

**3、如何高效准确的实现持续集成**

必不可少的需要一套能自动化、并且可视化的平台来帮助我们。

那么总结来看，Jenkins就是一个可以帮助我们实现持续集成的平台。

### 二、为什么Jenkins能帮助我们进行持续集成

 理由有如下几点：

**1、Jenkins是一个开源的、且基于JAVA代码开发的持续集成系统。**

因为含有非常丰富的插件支持所以我们可以方便的打通版本库、测试构建环境、线上环境的所有环节。并且丰富友好的通知使用者和开发、管理人员。

**2、安装维护简单**

安装Jenkins，不太复杂。且支持通用的平台。

**3、Java 应用 常用**

在企业的软件构建过程中，JAVA的应用工程稍显复杂，由于复杂构建、和代码上线、并且服务的重启。整个过程下来，消耗的时间较多，Jenkins却能很好的集成maven的编译方式，且利用自动化的插件、和自定义开发脚本的支持。所以目前广泛的应用于JAVA工程的持续集成平台。

好了，那么接下来我就来介绍，如何搭建一套快速有效的Jenkins自动化发布持续集成平台。


## 前言

为了提高工作效率，避免重复的手动发包部署工作，特搭建Jenkins+Ansible的自动部署平台。
主要实现原理是：

* 由Jenkins拉取git代码  
* 使用Maven命令编译打包  
* 通过Ansible 发送war包到对应的服务器  
* 通过Ansible执行远程服务器的重启命令 
* 发布完成。

### 一. 环境介绍

本平台搭建在CentOS6环境中，其他linux环境情况类似。
在自动部署服务器中需要安装以下软件：
1.               JDK
2.               Git
3.               Maven
4.               Tomcat
5.               Jenkins
6.               Ansible

### 二. 步骤：

0. ansible安装与配置；
1. Jenkins的安装；
2. Jenkins的配置；
3. 安装配置tomcat
4. 安装配置Java；
5. 安装配置maven；
6. nginx反向代理配置访问域名；
7. 相关插件安装；
8. 系统设置明细；
9. slave节点配置；
10. 一些依赖项的安装。



#### 1.1	Ansible简单介绍
Ansible是一个部署一群远程主机的工具。远程的主机可以是本地或者远程的虚拟 机，也可以是远程的物理机。
Ansible通过SSH协议进行管理节点和远程节点之间的通信。理论上说管理员通过 ssh到一台远程主机上能做的操作Ansible都可以做。包括：拷贝文件、安装包、启动服务...


总结：**Ansible把一些shell命令封装成一个个模块，并通过SSH连接，在远程机器上执行这些模块包含的脚本。**


#### 1.2	Ansible安装与配置
系统环境：centos7.1
直接使用yum安装ansible

```bash
# yum install http://mirrors.sohu.com/fedora-epel/6/x86_64/epel-release-6-8.noarch.rpm
# yum install ansible
```

安装完成之后，需要`ansible`中的`hosts`文件分组，编辑`/etc/ansible/hosts`文件，将目标机器加入分组，并配置好面秘钥登录。
Ansible更多详细内容可以参考我之前写的[ansibles学习篇](http://blog.yangcvo.me/2016/03/07/%E8%87%AA%E5%8A%A8%E5%8C%96/Ansible/Ansible%20%E5%B0%8F%E7%BB%93%EF%BC%88%E4%B8%80%EF%BC%89ansible%E7%9A%84%E5%AE%89%E8%A3%85%20/)


#### 二. 安装Jenkins环境


1)	安装并配置好JAVA、JDK、Tomcat，并去官网下载最新版本的war包。

去官网下载包资源：

```bash   
apache-tomcat-8.0.28.tar.gz
jenkins.war
jdk1.8.0_66.tar.gz
maven.tar.gz
```

没有下载成功的我这里已下载好了。最新版本
[jenkins.war](http://oak0aohum.bkt.clouddn.com/jenkins.war)
[tomcat.tar.gz](http://oak0aohum.bkt.clouddn.com/apache-tomcat-8.0.28.tar.gz)
[maven.tar.gz](http://oak0aohum.bkt.clouddn.com/maven.tar.gz)

##### 2.1 jdk安装：

```bash
tar -zxvf jdk1.8.0_66.tar.gz -C /srv/.
         
这里在配置环境变量。

vim /etc/profile.d/java.sh
export JAVA_HOME=/srv/jdk1.8.0_66
export CLASS_PATH="$JAVA_HOME/lib:$JAVA_HOME/jre/lib"
export PATH=$PATH:$JAVA_HOME/bin

[root@Jenkins ~]# java -version
java version "1.8.0_66"
Java(TM) SE Runtime Environment (build 1.8.0_66-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.66-b17, mixed mode)
```

##### 2.2 Tomcat安装配置：

Jenkins官网：https://jenkins.io/index.html 
可参考我之前写的文档：[搭建配置tomcat环境](http://blog.yangcvo.me/2015/11/21/tomcat/%E6%90%AD%E5%BB%BA%E9%85%8D%E7%BD%AEtomcat%E7%8E%AF%E5%A2%83/)

```bash 
tar -zxvf apache-tomcat-8.0.28.tar.gz -C /srv/.
mv apache-tomcat-8.0.28 tomcat
useradd -s /sbin/nologin tomcat 
chown -R tomcat:tomcat tomcat
```

2)	放在配置好Tomcat，启动Jenkins，根据提示，输入安装秘钥。


##### 2.3 jenkins安装配置：

```bash  
mv jenkins.war /srv/tomcat/webapps/. 
cd /tomcat
./bin/startup.sh
然后打开地址：http://192.168.1.183:8080/jenkins
新版本会提示下载安装插件。下载以后会让你设置账号和密码还有邮箱。
```
            
进入安装界面: 
第一次，登录，需要进行一个解锁 ，页面也会有提示.

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins1.png)

秘钥的具体位置在：`/root/.jenkins/secrets/initialAdminPassword`中，我们可以通过这个文件中查看密码，并输入。

3)	设置好登录账号密码，根据推荐选项安装插件，等待下载安装完成。在插件安装过程中可能有些安装不成功，暂时先不用管。

它会给我们安装一些基础的插件 
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins_01.png)
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins_02.png)
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins_03.png)
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins_04.png)
早期jenkins默认是不需要登陆的。



#### 1.3 Maven 安装和配置

官网：<http://maven.apache.org/>
官网下载：<http://maven.apache.org/download.cgi>
历史版本下载：<https://archive.apache.org/dist/maven/binaries/>
此时（20160208） Maven 最新版本为：**3.3.9**

Maven 3.3 的 JDK 最低要求是 JDK 1.7 - 我个人习惯 `/opt` 目录下创建一个目录 `setups` 用来存放各种软件安装包；在 `/srv` 目录下创建一个 `program` 用来存放各种解压后的软件包，下面的讲解也都是基于此习惯.
      

下载压缩包：`wget http://mirrors.cnnic.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz`
解压指定到我个人习惯的安装目录下：`tar zxvf apache-maven-3.3.9-bin.tar.gz -C /srv/program/.`
移到我个人习惯的安装目录下：`mv maven3.3.9/ /srv/program/maven3`

```bash
环境变量设置：vim /etc/profile.d/maven.sh
    # Maven
    MAVEN_HOME=/srv/program/maven3
    PATH=$PATH:$MAVEN_HOME/bin
    MAVEN_OPTS="-Xms256m -Xmx356m"
    export MAVEN_HOME
    export PATH
    export MAVEN_OPTS
```

刷新配置文件：`source /etc/profile`
测试是否安装成功：`mvn -version`


#### 1.3.1 Maven 配置

- 配置项目连接上私服
- 全局方式配置：

``` xml
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">

    <!--本地仓库位置-->
    <localRepository>/srv/mv2</localRepository>

    <pluginGroups>
    </pluginGroups>

    <proxies>
    </proxies>

    <!--设置 Nexus 认证信息-->
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
    </servers>

    <!--设置 Nexus 镜像，后面只要本地没对应的以来，则到 Nexus 去找-->
    <mirrors>
        <mirror>
            <id>nexus-releases</id>
            <mirrorOf>*</mirrorOf>
            <url>http://localhost:8081/nexus/content/groups/public</url>
        </mirror>
        <mirror>
            <id>nexus-snapshots</id>
            <mirrorOf>*</mirrorOf>
            <url>http://localhost:8081/nexus/content/groups/public-snapshots</url>
        </mirror>
    </mirrors>


    <profiles>
        <profile>
            <id>nexus</id>
            <repositories>
                <repository>
                    <id>nexus-releases</id>
                    <url>http://nexus-releases</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                </repository>
                <repository>
                    <id>nexus-snapshots</id>
                    <url>http://nexus-snapshots</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <id>nexus-releases</id>
                    <url>http://nexus-releases</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                </pluginRepository>
                <pluginRepository>
                    <id>nexus-snapshots</id>
                    <url>http://nexus-snapshots</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>nexus</activeProfile>
    </activeProfiles>

</settings>
```

* jenkins发布修改过以后：

```
<settings xmlns="http://maven.apache.org/POM/4.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                               http://maven.apache.org/xsd/settings-1.0.0.xsd">
                               
                               
         <!--表示本地库的保存位置，也就是maven2主要的jar保存位置，默认在${user.dir}/.m2/repository，如果需要另外设置，就换成其他的路径。 -->
         
  <localRepository>/srv/m2</localRepository>
  <interactiveMode/>
  <usePluginRegistry/>
  <offline>false</offline>
  
  
  <!--当插件的组织Id（groupId）没有显式提供时，供搜寻插件组织Id（groupId）的列表。该元素包含一个pluginGroup元素列表，每个子元素包含了一个组织Id（groupId）。
当我们使用某个插件，并且没有在命令行为其提供组织Id（groupId）的时候，Maven就会使用该列表。默认情况下该列表包含了org.apache.maven.plugins。 -->

 <pluginGroups>
  <!--plugin的组织Id（groupId） -->
  <pluginGroup>org.codehaus.mojo</pluginGroup>
<pluginGroup>org.apache.tomcat.maven</pluginGroup>
<pluginGroup>org.mortbay.jetty</pluginGroup>
 </pluginGroups>
  <servers>
  <server>
        <id>tomcat</id>
        <username>tomcat</username>
        <password>keyfree123</password>
        </server>
        <server>
                <id>haozhuo</id>
                <username>haozhuo</username>
                <password>haozhuo123</password>
    </server>
        <!--d:server 的id,用于匹配distributionManagement库id，比较重要。 -->
         <server>
     <id>nexus-releases</id>
        <!--username, password:用于登陆此服务器的用户名和密码 -->
     <username>admin</username>
      <password>123456</password>
   </server>
   <server>
     <id>nexus-snapshots</id>
     <username>admin</username>
     <password>123456</password>
   </server>
  </servers>
  <mirrors>
  <!--表示镜像库，指定库的镜像，用于增加其他库 -->

   
    <mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>*</mirrorOf>
        <name>Nexus aliyun</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
        <mirror>
        <id>hz</id>
      <name>hz Central</name>
      <url>http://nexus.haozhuo.com:8083/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>

    <mirror>
      <id>CN</id>
      <name>OSChina Central</name>
      <url>http://maven.oschina.net/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>



          <mirror>
           <id>ibiblio.org</id>
           <name>ibiblio Mirror of http://repo1.maven.org/maven2/</name>
           <url>http://mirrors.ibiblio.org/pub/mirrors/maven2</url>
           <mirrorOf>central</mirrorOf>

     </mirror>
        
           <!--
     <mirror>
         <id>jboss-public-repository-group</id>
         <mirrorOf>central</mirrorOf>
         <name>JBoss Public Repository Group</name>
         <url>http://repository.jboss.org/nexus/content/groups/public</url>
     </mirror>
            -->
            
 </mirrors>
  <activeProfiles>
    <!--make the profile active all the time -->
     <!--<activeProfile>nexus</activeProfile>   -->
  </activeProfiles>
</settings>

```


### 三. 配置nginx反向代理.

在nginx配置目录/usr/local/nginx/conf/nginx.conf，配置文件内容如下

```bash
   upstream tomcat-jenkins {
        server  192.168.1.220:8080 weight=1;
    }

    server {
        listen       80;
        server_name  jenkins.yjk.cn;
        location / {
           index        index.html index.php index.jsp index.htm;
           proxy_pass           http://tomcat-jenkins;
           proxy_ignore_client_abort on;
           proxy_redirect               off;
           proxy_set_header     Host    $host;
           proxy_set_header     X-Real-IP       $remote_addr;
           proxy_set_header     X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

```
        
修改/etc/hosts，在末尾新增一行 :

```bash
127.0.0.1     jenkins.yjk.cn
```

重启nginx，`/usr/sbin/nginx -s reload`，即可以地址 `jenkins.yjk.cn` 访问Jenkins主页.



### 四. 安装部分Jenkins插件

访问Jenkins主页：http://jenkins.yjk.cn 
系统管理 --> 管理插件 --> 可选插件;
安装所需要的插件(根据需要自行选择)，如 `GitBucket Plugin、 FindBugs Plug-in 、Cobertura Plugin 、 Violations Plugin 、Email Extension Plugin`等

⚠️插件安装不成功处理办法：

进入左侧菜单栏的系统管理，发现很多插件无法安装成功，如下图：
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.cah.png)


复制安装失败的名称，点击右侧按钮，选择可选插件，输入名称重新安装。

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.1.png)

如果还是失败，请注意提示信息，可直接下载该插件手动上传安装，手动下载该插件，在插件管理的高级标签中，选择上传安装。

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.2.png)
如果上传失败，还有个方法：直接把下载下来的插件上传到Jenkins服务器上面。默认安装的话目录在：/root/.jenkins/plugins/  重启服务即可.


### 五. 系统设置以及介绍（可跳过）

#### 1.1 系统配置


进入首页系统管理--->系统设置。
1. 配置maven
2. 配置邮件服务
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.3.png) 

主目录默认在 /root/.jenkins  下面，此目录保存jenkins 的所有配置和插件等，具体可以于服务器中查看该目录。
下面可以继续配置maven，jdk，git等等，也可以使用默认的配置。
 
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.4.png)
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.5.png)

#### 1.2 项目配置

新建一个自由风格的项目，输入项目名称。

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.6.png)
1.参数配置

简单的从Git中拉取代码无法满足我们的需求，最理想的状态是可以对发布的分支进行动态选择。所以我们需要安装一个配置动态参数的插件：[Dynamic Parameter Plug-in](https://wiki.jenkins-ci.org/display/JENKINS/Dynamic+Parameter+Plug-in) 

安装好插件之后，在项目配置，参数化构建过程中添加动态参数：

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.7.png)
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.8.png)

输入变量名称和脚本，在接下来的配置中就可以使用`$Name `的方式，使用变量了。如：`$branch`。


![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.9.png)

```bash
def gettags = ("git ls-remote -h git@gitlab.ihaozhuo.com:Java_Service/YJK-Java.git").execute() 
gettags.text.readLines().collect { it.split()[1].replaceAll('refs/heads/', '')  }.unique()
```

 这是一段`Groovy`脚本，用来获取Git地址的分支，将Git地址改成自己的就行。

#### 1.3接下来选择扩展功能。

由于我们的项目模块较多，所有的服务全部重启会非常浪费时间，我们希望能对单独的模块进行部署，这时候需要添加一个可选择的参数`Extensible Choice Parameter`

扩展选择参数插件`Extensible Choice Parameter`灵活使用实现部署多应用模块服务同时发布.单模块服务单独打包快速发布.

实现效果：

发布选择应用模块，之前只能单个选择发布，效率太低，安装choice扩展性的插件，可以选择多个模块应用一起部署。非常方便
下载插件：
[Extended Choice Parameter Plug-In](https://plugins.jenkins.io/extended-choice-parameter)
[Extensible Choice Parameter plugin](https://plugins.jenkins.io/extensible-choice-parameter)

这里jenkins 第一步安装扩展插件：

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.1.png)
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.2.png)
安装好插件以后 进入创建项目配置参数：

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.3.png)

`Value` ：这里选择模块名字最好是git拉取下来的名称一样。这样下面打包可以选择对应模块名称一样。
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.4.png)

这里构建使用shell 做一层`if` 判断  选择`all` 全部发布。和单独模块发布。

#### 1.4Git配置

初次使用Jenkins需要添加一个有访问Git权限的用户。
我直接使用系统自带的ssh-key，也可以自己去生成。
在gitlab上添加ssh-key公钥 这个是你发布机器的公钥。

1.源码管理选择Git，输入URL，并选择用户。

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.10.png)

从Jenkins发布服务器上面查看公钥上传到gitlab.

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansbile.11.png)

2.在jenkins上点击添加私钥
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.12.png)
3.选择SSH...key，输入用户名，输入私钥，输入密码，添加完成。
在Jenkins发布服务器上面查看私钥：

```bash
[root@ansible_02 yaml]# cat /root/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAv5BSNmykQcvklQdb+3jV8d1I2tjq4tIiyVbQDhDBs6UFR4hj
20EHg/3Tjvw+4mHtzWFYv1/rYeTg7xJ/bIf3XRIeRcrc/b2rvqo3wQBupRjHM6wb
nd+Jgyv2v8Lk2c8yNzMoUIuYzcG5Dpv5mWyg1akbWcIH3tsGyTVXxxmAQ78L+J1N
zj3a9fImQcx5ocKlV/Y1v9B+Uv4gTBmSBR7akvNg7E+3JKTphbbUv3rFrG6zqvTW
CtEA/h9X//zPWTiwTjJv87NJkHQLqoo1TPYZcNjDEfki+jfc9gJzJvZ/5D0A3fA/
mO8ApW9VMsaLiiwQvGP2RoiTVwU2G1iNASBcWwIDAQABAoIBABh64fawhYEfBDQD
P77wHy8MXz4QUFvyDJ38KRRTEd3aLcWJaXFgawx0CHASThrx9sizMvspz9Ovwwrq
Kzx8V6EeKp4yoXEPpv3zlLJmUr1oYDR7PwA6y8Dmgl7ZEhO/haRGNlWssTdCFVsH
lasElb0YIjWjNQxGoyRdW71GxfxiGdZXgWDT7yYGvdavO5tGzDPhh6/vf5lpU2F2
nJbMyXzWgrrqXqIG3W+jqg2kVAfiMdNgeUOn6+SA/7uK8rNhxG/5dr7WKJE31uRS
FJiVCATcHQBIRScDDd/nI8rGiVQOjHFt2oNV82DOIhNXhbzdWGROUcMiT04OoJgQ
JeNL7IkCgYEA8wrgamw1w5dBoynzV6tUPgSdHhAMoMhPrzJTaG8qMamnHzZgz1fl
3jqI7qCNsj1n6Y1UbzhKILHcTIoUj7lj41mY4XVBHqDLXh9Socf9IPrSabZIfe2n
wMLMhdhoaMPUWtHj9SUn/Y+ESLuFLtF6LjR1Eghu8xKRwULo1GscSyUCgYEAycbY
j1/KaFAcwSl7teRiYEY+LRHB/AUa7lc+r+p2YMjGQ3CiaN9wKqu1E4jEvL0eZXDK
aXJ6qcDJiHa2M+Bf8qcVoyXtKFtlTfSVPJDviZjl3EPmhXbuKzW3et2w5zWv8lAU
2fWU+3ta3fVrFXz2zlY1j47Dv9zUlDeCF+lRMX8CgYEA1hkKwDU612XzSEy4NM6U
k111GvqAZVKP/4GRwDnNLZqJwhEhDwYbVLyzy6JbsFwvoaoCa0dm5Y5IxpQMsN9b
Z9GGH7CWLEJ7a4gJmrYQYpECgYAIqe8WiOhp/jad3KghMUNAGwQEb2TC630yirB4
YTrgAP7yWl2+3wkz69eElTTNXdl2RZeLW40EyPBeWaqNI686/g2hybkbKIF7DWtz
BE4kvFnyUUAOrwKe/Fl6fxZfdyCs6N9cVH0nJy7JpQYKECmQxobaOSkSjerayl9d
x1mI3d6fwqmAbbyiolLXbn3JySkxTaVBS8gjnQh6LO6JbpTspKlz3R2/CzUhDhEY
n3vhU2trAGh6hY/bizI1oKNxwp/b9uAaLVmmHGNV/+V5glnDH+luQA==
-----END RSA PRIVATE KEY-----
```
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.ansible.13.png)

此处要注意，将默认的master分支改为我们之前定义好的分支变量.
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.5.png)

#### 1.5项目打包

由于我们使用手动出发构建，触发器我们此处不需要配置。
构建环境暂时也不需要配置，这里调用ansible去执行yml代码发布。项目打包优化了。

关键在于构建的步骤。
我们可以使用Shell命令，进入项目目录，使用Maven命令进行打包。

这里主要是`if` 判断是 执行对应模块打包。这样节省了打包时间，如果不做对应发布模块发布，第一条如何`model`等于`all` 文件：全部执行 : `mvn clean install -U -T 1C -Dmaven.test.skip=true `

如果不等于这里指定对应模块打包： `mvn clean install -pl $model -amd -U -T 1C -Dmaven.test.skip=true`

```bash
echo $model
if [[ $model == all* ]]; then
cd /srv/ && rm dev-properties/ -rf && git clone git@gitlab.ihaozhuo.com:dev-properties/dev-properties.git && cd /root/.jenkins/workspace/yjk-dev_master/haozhuo/ && mvn clean install -U -T 1C -Dmaven.test.skip=true  -Dmaven.compile.fork=true -Pdev -Dautoconfig.userProperties=/srv/dev-properties/dev1.properties -Dautoconfig.charset=utf-8
else
cd /srv/ && rm dev-properties/ -rf && git clone git@gitlab.ihaozhuo.com:dev-properties/dev-properties.git && cd /root/.jenkins/workspace/yjk-dev_master/haozhuo/ && mvn clean install -pl $model -amd -U -T 1C -Dmaven.test.skip=true  -Dmaven.compile.fork=true -Pdev -Dautoconfig.userProperties=/srv/dev-properties/dev1.properties -Dautoconfig.charset=utf-8
fi
```
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.6.png)

所有的脚本和变量的参数列表统一。

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi8.png)

这里说下发布模块 结合`ansible-Playbook`基本语法写控制发布服务器，然后在`jenkins`上面写ansible 发布脚本&命令。
model.sh 脚本： 

```bash
#!/bin/sh
  IFS=','
  arr=$1
  for split in $arr
  do
          echo $split
       /usr/bin/ansible-playbook /srv/yaml/$split.yml
          if [ $split = 'all' ]
          then
                  echo "model:all"
                  break
          fi
  done
```

#### 1.6发送邮件

配置好构建失败后发送邮件到指定的邮箱，多个邮箱用逗号隔开。


![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.7.png)

至此项目配置结束。

#### 六.运行效果，打包效果。

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.9.png)
指定模块去打包，终端上查看发布日志 这里变量生效效果：
![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins.peizhi.10.png)

### 七. Jenkins延伸学习


效果图：

![](http://7xrthw.com1.z0.glb.clouddn.com/jenkins_05.png)

Blue Ocean

这里可以查看下我在我github上面扩展[更新jenkins新功能Blue Ocean实现先进的可视化精确定位问题
]()
现在我们Git使用的是 GitLab，同时为了安全我们做了一层LDAP代理，效果相当于“将军令”，操作机、Git和Jenkins用 OpenLDAP 做统一认证，后续用到的Redmine、Grafana、Zabbix 等都接入了OpenLDAP认证，每个人都有个动态口令，每次验证都需要用到。


