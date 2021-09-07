# Common questions about MacOS.
### 超过最大进程数

* 问题表象如下

  ![cq 1](https://raw.githubusercontent.com/chan132/summary/main/images/mac/img_cq_1.png)

* 解决方案：暂时提升进程数【详见：[Insufficient resources error when trying to launch a simulator](https://help.apple.com/simulator/mac/9.0/index.html#/dev8a5f2aa4e)】



### 更新系统至Big Sur版本之后，在Android Studio中构建项目时，找不到java安装目录

* 问题表象如下

  ![cq 2](https://raw.githubusercontent.com/chan132/summary/main/images/mac/img_cq_2.png)

* 解决方案

  * 参考文章：[stackoverflow](https://stackoverflow.com/questions/55286542/kotlin-could-not-find-the-required-jdk-tools-in-the-java-installation)

  * 卸载JRE：

    $ `cd /Library/Internet\ Plug-Ins/`

    $ `mv JavaAppletPlugin.plugin DELETED-JavaAppletPlugin.plugin`

    $ `cd /Library/PreferencePanes/`

    $ `mv JavaControlPanel.prefPane DELETED-JavaControlPanel.prefPane`





