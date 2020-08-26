# 笔记

## Docker Desktop 安装

1. 进入BIOS面板开启虚拟化技术支持。
2. 进入Windows程序和功能当中启用Hyper-V、容器、适用于Linux的Windows子系统、虚拟机平台功能。
3. 安装Docker，完成后按照提示更新WSL2内核。
4. 可以通过右下角图标在Linux容器和Windows容器之间进行切换，一些应用程序的image只支持Linux容器，需要注意。

安装过程中遇到的问题：

1. 在BIOS面板中把一些看似虚拟化技术支持的功能enable并重启后，在CPU面板仍看到虚拟化已禁止。解决方案：在开启适用于Linux的Windows子系统、虚拟机平台功能并重启后，发现虚拟化已启用。
2. 下载image非常之慢。解决方案：前往阿里云控制台启用镜像加速服务，使用其中用于Docker加速的源。

## 命令

```bash
# 拉取最新镜像
docker pull redis:latest
# 运行容器，6379:6379表示映射容器服务的6379端口到宿主机的6379端口。外部可以直接通过宿主机IP:6379 访问到容器中的Redis服务。
docker run -itd --name redis-test -p 6379:6379 redis
# 运行/bin/bash程序打开Linux系统的shell界面
docker exec -it redis-test /bin/bash
```

## Redis镜像原理

因为Redis服务需要有Linux环境，所以首先需要有一个Linux系统，在其之上安装Redis服务，打开shell界面，输入redis-cli进入交互页面。
