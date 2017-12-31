---
layout: post
title: opendaylight控制器执行环境(karaf)目录结构
date: 2017-10-27
categories: [SDN, OpenDaylight, controller]
---

目录名称 | 存储文件说明
---|---
karaf-dist | 根目录
+--bin | 启动脚本和命令
+--configuration|配置文件
/&nbsp;&nbsp;&nbsp;&nbsp;+--factory | 安装的bundles
/&nbsp;&nbsp;&nbsp;&nbsp;+--initial | 默认的日志
/&nbsp;&nbsp;&nbsp;&nbsp;+--ssl | 临时文件
+--data |
/&nbsp;&nbsp;&nbsp;&nbsp;+--cache | 安装的bundles
/&nbsp;&nbsp;&nbsp;&nbsp;+--generated-bundles |
/&nbsp;&nbsp;&nbsp;&nbsp;+--kar |
/&nbsp;&nbsp;&nbsp;&nbsp;+--log | 默认的日志
/&nbsp;&nbsp;&nbsp;&nbsp;+--pax-web-jsp |
/&nbsp;&nbsp;&nbsp;&nbsp;+--tmp | 临时文件
+--deploy |
+--etc | 配置文件
/&nbsp;&nbsp;&nbsp;&nbsp;+--opendaylight |
/&nbsp;&nbsp;&nbsp;&nbsp;/&nbsp;&nbsp;&nbsp;&nbsp;+--current |
/&nbsp;&nbsp;&nbsp;&nbsp;/&nbsp;&nbsp;&nbsp;&nbsp;+--datastore |
/&nbsp;&nbsp;&nbsp;&nbsp;/&nbsp;&nbsp;&nbsp;&nbsp;+--karaf |  
+--instances | 实例管理
+--journal |
+--lib | 核心库
/&nbsp;&nbsp;&nbsp;&nbsp;+--bppt |
/&nbsp;&nbsp;&nbsp;&nbsp;+--endorsed |
/&nbsp;&nbsp;&nbsp;&nbsp;+--ext |
+--patches | 升级包
+--snapshots | 快照
+--system | 系统bundle库

与karaf自身的目录结构相比，多了configuration、journal、patches和snapshots文件夹。
