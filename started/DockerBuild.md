默认情况下 [Kubernetes configs](https://github.com/youtube/vitess/tree/master/examples/kubernetes)
使用  [Docker Hub](https://hub.docker.com/u/vitess/) 中的`vitess/lite`镜像.
这个镜像是定期从github的master分支创建出来的.

如果你想创建自己的镜像可以按下面操作:

1.  安装 [Docker](https://www.docker.com/) .

    我们的脚本还假设您可以运行docker命令而不使用sudo，您可以通过 [设置docker组](https://docs.docker.com/engine/installation/linux/ubuntulinux/#create-a-docker-group)来执行.

2.  创建一个 [Docker Hub](https://docs.docker.com/docker-hub/) 帐号，使用 `docker login` 登录.

3.  进入你的 `$GOPATH/src/github.com/youtube/vitess` 目录.

4.  通常你不需要 [创建你的引导镜像](https://github.com/youtube/vitess/blob/master/docker/bootstrap/README.md)
    除非你要修改 [bootstrap.sh](https://github.com/youtube/vitess/blob/master/bootstrap.sh)
    或者 [vendor.json](https://github.com/youtube/vitess/blob/master/vendor/vendor.json),
    例如添加新的依赖. 如果你不需要重新创建引导镜像，你可以选用下面的镜像:

    ```sh
    vitess$ docker pull vitess/bootstrap:mysql57   # MySQL Community Edition 5.7
    vitess$ docker pull vitess/bootstrap:mysql56   # MySQL Community Edition 5.6
    vitess$ docker pull vitess/bootstrap:percona57 # Percona Server 5.7
    vitess$ docker pull vitess/bootstrap:percona   # Percona Server
    vitess$ docker pull vitess/bootstrap:mariadb   # MariaDB
    ```

    **注意:** 为保证你本地镜像是最新的，你最好每次都执行上面命令拉取镜像

5. 构建`vitess/lite[<flavor>]`镜像。 这将构建`vitess/base[<flavor>]`镜像并运行一个脚本，只提取运行Vitess所需的文件（`vitess/base`包含开发工作所需的所有内容）。

    使用下面的命令之一创建镜像 (`docker_lite` 默认使用MySQL 5.7):

    ```sh
    vitess$ make docker_lite
    vitess$ make docker_lite_mysql56
    vitess$ make docker_lite_percona57
    vitess$ make docker_lite_percona
    vitess$ make docker_lite_mariadb
    ```

6.  修改镜像的tag到你个人的`docker hub`仓库, 然后上传.
    ```sh
    vitess$ docker tag -f vitess/lite yourname/vitess
    vitess$ docker push yourname/vitess
    ```

    **注意:** 如果你使用的不是默认的 `vitess/lite`要改成这种 `vitess/lite:<flavor>`.

7.  修改yaml中的镜像地址为新地址:
    ```sh
    vitess/examples/kubernetes$ sed -i -e 's,image: vitess/lite,image: yourname/vitess:latest,' *.yaml
    ```

    添加`:latest` 的意思是让`kubernetes`在加载镜像时每次都检查是否有新的镜像更新，如有更新的话，新的`pod`就会拉新的镜像了。

    Once you've stabilized your image, you'll probably want to replace `:latest`
    with a specific label that you change each time you make a new build,
    so you can control when pods update.

8.  按 [Vitess on Kubernetes](http://vitess.io/getting-started/) 的方式启动vitess.
