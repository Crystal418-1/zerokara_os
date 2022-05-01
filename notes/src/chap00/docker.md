# 使用Docker

使用Docker的目的是让使用Windows的同学不必开一个庞大臃肿的虚拟机也可以使用Linux的工具和服务。

[菜鸟教程](https://www.runoob.com/docker/docker-tutorial.html) 是一个不错的初学教程，基本上够用了。使用Windows的同学需要安装WSL。其他细节我也不了解，作为理工科的学生，你们应该要学会自食其力。

我们所有和 `Docker` 有关的内容都放在了项目根目录下的 `docker` 文件下了。

这里推荐一个Docker的可视化管理工具 [Portainer](https://www.portainer.io/)。安装方法也很简单，跑一下这个命令：

```sh
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer:latest
```

Linux下在 `~/.bashrc` 或者 `~/.bash_aliases` 里面添加一行 `alias portainer='sudo docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer:latest'` 也不错。

安装结束后可以使用浏览器打开 <https://localhost:9000> 来访问。端口号可以自己随便改。

一般来说运行 `docker` 需要 `sudo` 权限，但是我不是很喜欢直接给某个用户sudo权限。我是这么做的：

```sh
alias docker='sudo docker'
```

## 简单使用方法

在安装了 `Docker` 之后，打开到项目根目录，运行 `docker build -t zerokara:latest docker`，事情就成了。
镜像可以随便取名，标签也可以随便写，自己记得是哪一个就行了。
