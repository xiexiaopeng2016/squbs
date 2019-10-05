# 应用程序生命周期管理

此页描述了打包、部署和启动squbs应用程序的快速方法。本指南以亚马逊EC2为例，展示如何在不到半小时内运行squbs应用程序。

## 打包

您需要在构建实例上安装以下内容：

- [git](https://git-scm.com/downloads)
- [java 8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- [sbt](http://www.scala-sbt.org/release/docs/Setup.html)

构建的步骤：

- 从git仓库克隆源代码到`<project>`目录
- cd `<project>`
- 运行sbt构建命令，包括`packArchive`，例如：`sbt clean update test packArchive`
- 在`<project>/target`下创建了两个存档
- `<app>-<version>.tar.gz`
- `<app>-<version>.zip`

## 启动

需要在运行的实例上安装以下内容

- [java 8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

运行的步骤：

- 将其中一个存档复制到正在运行的实例
- `<app>-<version>.tar.gz`
- `<app>-<version>.zip`
- 例如，解压 `tar zxvf <app>-<version>.tar.gz` 到 `<app>-<version>` 目录
- 启动应用 `<app>-<version>/bin/run &`
- 你可以从这个实例检查管理 `http://localhost:8080/adm` 或 `http://<host>:8080/adm`

## 关闭

你可以终止正在运行的进程，例如，在linux中`kill $(lsof -ti TCP:8080 | head -1)`。由于应用程序注册了与JVM的关闭钩子，它将正常关闭，除非它已意外关闭。

## Amazon EC2

Log into AWS EC2 and launch an instance

- You can create from free-tier, if the capacity meet your needs
- Security group open (inbound) SSH – port 22, Custom TCP Rule – 8080
- SSH into server (see AWS Console -> Instances -> Actions -> Connect)
- Execute step `Start` and `Shutdown` as described above
