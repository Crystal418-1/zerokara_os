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

+ 挂载镜像

    在安装了 `Docker` 之后，打开到项目根目录，运行 `docker build -t zerokara:latest docker`，事情就成了。
    镜像可以随便取名，标签也可以随便写，自己记得是哪一个就行了。

+ 使用Docker内的应用程序

    假设我把我的镜像命名为 `zerokara`，我的 **当前目录** 是项目根目录，那么，要使用docker里面的程序，例如，使用 `mdbook`，那么，我可以这么输入命令：

    ```sh
    docker run --rm -it -v $(pwd)/notes:/notes zerokara:latest /root/.cargo/bin/mdbook build /notes
    ```

  + `--rm`：在用完容器以后马上把容器扔了。

    `docker` 运行镜像的时候会产生一个容器，用 `docker ps -a` 可以查看所有的容器。容器之于镜像有点像类的实例之于类。

  + `-it`：`-i` 是交互式(interactive)的意思，`-t` 是终端(tty) 的意思。

  + `-v`：是`volume` 的意思（并不是“音量”，而是一个计算机术语，类似于Windows里的“卷”，一个存储数据的集合）。

    用途就是挂载某个目录。例如，`$(pwd)/notes:/notes` 就是把当前目录下的 `notes` 目录挂载到容器的 `/notes` 目录，容器内的程序放访问这个目录会被自动重定向到容器外的 `notes` 目录。

    > **注意**：以上是类Unix系统的写法，例如`$pwd`、目录分隔符`:`等。我并不清楚Windows下还是不是这么写的。以后有时间会补充的。
