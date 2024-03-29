---
layout:     post
title:      "Windows 小技巧"
subtitle:   "关闭软件自动更新、自动拨号上网"
date:       2020-12-06 23:23:00
author:     "LpengSu"
header-img: "https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/2020-06-bg.jpg"
catalog: true
tags:
    - 经验总结
---

### 给软件断网：关闭其自动更新的办法

> 以Win10为例（主要是为了盗版软件避免更新后失效）

1. 打开【控制面板】→【系统与安全】→【Windows防火墙】→【高级设置】
2. 选择【出站规则】→【新建规则】→【程序】
3. 点击【浏览】，找到需要阻止访问网络的程序(**Updater.exe**)

### 文件夹不能设为隐藏的解决办法

打开cmd，使用

```
attrib -s 【目标文件夹】
```

### Win10自动拨号上网

> 开通宽带之后使用网线拨号，避免了开机手动连接网络

1、我的电脑，鼠标点击右键点击【管理】

2、任务计划程序，右侧的菜单栏点击【创建基本任务】

3、输入名字 ：宽带连接， 下一步

4、选择当前用户登录，下一步

5、下一步

6、在程序或脚本框输入：rasdial 宽带连接 宽带账户名 密码

7、下一步，出现对话框选择是