---
layout: post
title: opendaylight控制器源码编译
date: 2017-10-25
categories: [SDN, OpenDaylight, controller]
---
# {{ page.title }}

## 1. 源码下载
&emsp;&emsp;使用[git](https://git-scm.com/)或[Github Desktop](https://desktop.github.com/)从[Github](https://www.github.com)克隆需要的源码。
1. [控制器controller](https://www.github.com/opendaylight/controller)，已包含md-sal、config、archetypes、commons、blueprint和model等features；
2. [网络配置组件netconf](https://www.github.com/opendaylight/netconf)，已包含protocol-framework和restconf等features；
3. [WEB服务核dlux](https://www.github.com/opendaylight/dlux)；
4. [WEB应用dluxspps](https://www.github.com/opendaylight/dluxapps)；
5. [交换机功能组件l2switch](https://www.github.com/opendaylight/l2switch)；
6. [OF协议openflowplugin](https://www.github.com/opendaylight/openflowplugin)，已包含openflowjava等features。

## 2. 集成features
&emsp;&emsp;与其他早期的版本不同，当前版本的ODL控制器源码中，并没有包含相应的features，需要自己按需集成。虽然也可以编译安装完ODL控制器后，再执行其它features的编译和安装工作，但直接将必要的features源码集成到ODL控制器源码，有利于控制器的部署工作。

### 2.1. 拷贝feature源码
&emsp;&emsp;实际操作时，请用待集成的feature的名称替换下文中的FeatureName，如netconf或dlux等。

#### 2.1.1 新建文件夹
1.  /pathto/controller/features/FeatureName    
2.  /pathto/contorller/opendaylight/FeatureName

#### 2.1.2 拷贝文件
1. /pathto/FeatureName/features/\*.\* __TO__ /pathto/controller/features/FeatureName/    
2. /pathto/FeatureName/\*.\* __TO__ /pathto/contorller/opendaylight/FeatureName/
>FeatureName中与features和karaf等独立部署时需要使用的文件夹无需拷贝。    
>例如：    
>netconf的根文件夹中包含有docs、netconf、restconf、protocol-framework和karaf文件夹，其中的karaf就不需要拷贝，docs也可不拷贝。

### 2.2. 修改feature的包管理文件pom.xml
&emsp;&emsp;修改文件/pathto/controller/opendaylight/FeatureName/pom.xml的modules节，将其中不需要的模块去掉。
>例如：
>对netconf（/pathto/controller/opendaylight/netconf/pom.xml），删除或注释(\<!-- 原有标记行   -->)掉features和karaf模块。

### 2.3. 修改ODL控制器的包管理文件pom.xml
#### 2.3.1. 添加模块

&emsp;&emsp;在文件/path/controller/pom.xml的moudles节中，逐个添加模块。    
```
<module>opendaylight/FeatureName</module>    
 ......
```

#### 2.3.2. 添加features

&emsp;&emsp;在/path/controller/features/pom.xm的moudles节中，逐个添加模块。    
```
<module>FeatureName</module>    
......
```

&emsp;&emsp;__注意：__ 两个文件中的模块添加方式不一样，前者需要加上“opendaylight/FeatureName”，后者则直接使用FeatureName。

### 2.4. 修改ODL控制器Karaf的包管理文件pom.xml
&emsp;&emsp;修改文件/pathto/controller/karaf/opendaylight-karaf/pom.xml，在其依赖集“dependencies”节中添加features。通常，在文件/path/controller/features/FeatureName/features-FeatureName/pom.xml中，会有添加feature所需要的信息，包括groupId、artifactId和version等。

&emsp;&emsp;例如，添加netconf的features的代码如下：    
```
<dependency>
	<groupId>org.opendaylight.netconf</groupId>
	<artifactId>features-netconf</artifactId>
	<version>1.4.0-SNAPSHOT</version>
	<classifier>features</classifier>
	<type>xml</type>
	<scope>runtime</scope>
</dependency>
```    
其中的classifier、type和scope基本上都是固定的，主要是修改groupId、artifactId和version这三个标记。

## 3. 源码编译
&emsp;&emsp;由于maven-plugin-plugin、karaf-maven-plugin和maven-antrun-plugin这三个插件Eclipse已不集成，所以不能直接在Eclipse中编译OpenDaylight。而且，也不能在IDEA Intellij中编译。因此，只能在终端中使用mvn命令进行编译工作。初次编译会耗费比较长的时间。        
&emsp;&emsp;&nbsp;整体编译控制器，打开命令窗口，使用mvn编译即可。     
```
cd /pathto/controller
mvn clane install
```

&emsp;&emsp;编译时可用的命令行参数：

参数  |  说明
|---|---|
|-Dmaven.repo.local=/pathto/repository | 设置本地仓库的存储位置
|-D skipTests | 跳过测试|
|-e | 显示详细的编译错误信息|
|--settings=/pathto/settings.xml | 指定mvn的工作时使用的配置文件|
|-rf :MoudleName | 从指定模块开始编译|

## 4. 运行

&emsp;&emsp;在命令终端中，运行karaf，即可启动控制器容器，但并没有全部安装需要的features。
### 4.1. 启动karaf

```
cd /pathto/controller/karaf/opendaylight-karaf/target/assembly/bin
./karaf
```
### 4.2. 加载features

&emsp;&emsp;启动容器后，可以使用```feature:list[ | grep xxx]```查看所有的features或能与给定字符串匹配的features。

```
feature:install odl-restconf
feature:install odl-l2switch-switch
feature:install odl-mdsal-apidocs
feature:install odl-dlux-core
feature:install odl-dluxapps-applications
feature:install odl-dluxapps-yangutils
```

## 5. 测试

&emsp;&emsp;启动浏览器，输入```http://IP地址:8181/index.html```，页面正常显示后，以用户名“admin”和密码“admin”登录，即可进入ODL控制器的管理界面。

&emsp;&emsp;如果是其他情况，请注意karaf控制台的输出信息和日志。

## 6. 调试OpenDaylight
&emsp;&emsp;由于不能在IDE中编译和运行OpenDaylight，所以，只能通过远程的方式调试。 

### 6.1. 启动控制器

&emsp;&emsp;带debug参数启动OpenDaylight，早期使用的-debug，但从Helium版开始，直接使用debug即可。

```
cd \pathto\karafdist\bin
[./]karaf debug
```

&emsp;&emsp;注意ODL启动时给出的端口号。

### 6.2. 调试工具设置

&emsp;&emsp;Eclipse和Interllij的设置都差不多，都是在运行（Run）的配置中将运行中的OpenDaylight和调试器进行关联，包括远程主机的IP地址和端口（上一步中的），以及配置源代码的位置。
