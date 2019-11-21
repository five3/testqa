# 如何通过Docker管理自动化测试数据
![关注二维码](https://www.testqa.cn/static/banner.png)

> 我们知道自动化测试都会有前提准备的步骤，而这个环节一般都是基础数据的准备。本文将会介绍如何通过Docker来管理基于Mysql的测试数据。

通常自动化用例执行的步骤大概如下：
1. setup
1. do testing
1. teardown

其中`setup`和`teardown`就是给正式测试做前提准备和收尾的工作，而数据的准备和恢复就经常会出现在这2个环节。对于少量的数据可以通过mysql快速恢复，或者干脆直接生造出来；但是当数据量太大或者数据结构变复杂的情况，就需要一种快速的数据恢复机制。

如今的技术中，能够实现快速恢复又简单方便的技术自然就要选Docker了。虽然Docker并不是为了测试技术而生的，但是却天然有着独属于测试的基因。


## 方案选定
通过Docker实现数据环境的恢复有很多种场景，这里主要讲的是恢复mysql数据的场景。在Dockerhub上可以直接拉去mysql的镜像，但是却不能直接满足我们的需求。

主要原因是官方提供的mysql镜像默认会把mysql的数据目录映射到宿主机中，并且即使你进行数据变更后再commit镜像，重启后依然会使用新的宿主机的映射路径。导致每次重启容器都没有保留之前DB中执行的变更。

为了解决这个问题，实现从特定的镜像启动容器时，容器内能够保留我们预定的基准测试数据。这里有2种方案：
1. 启动容器时挂载一个指定的宿主机路径作为mysql的数据目录
1. 通过修改后的mysql的Dockerfile来构建镜像，不把mysql数据目录挂载到宿主机

这2种方式都是可以实现我们的目标，方案1的好处是由于我们的数据目录在宿主机上，所以即使哪天镜像被删除或者docker出问题了，但数据不会丢失；方案2的优点是操作更加简单和快速，符合常规的Docker操作场景。

## 指定挂载目录
由于官方提供的mysql镜像，在构建的时候通过volume来挂载mysql的数据目录；所以每次新启动的时候，都会重新使用新的宿主机目录来进行挂载，导致容器中的mysql变更不能被保存下来。

为了能够实现目录，该方案的具体步骤如下：
1. 通过mysql官方镜像启动容器
1. 对mysql容器进行数据初始化操作
1. 复制容器中的mysql数据目录到宿主机路径
1. 停止mysql容器并删除容器
1. 再次通过mysql官方镜像启动容器，并挂载之前复制到宿主机的mysql数据目录

通过上述步骤之后，再次的启动容器，之前变更的mysql数据就会被正常的保留。具体的操作命令如下：
```bash
docker pull mysql
docker run -d --name mysql -p 33060:3306 mysql:latest
# 数据库初始化测试基础数据
docker cp 96f7f14e99ab:/var/lib/mysql /data/docker/mysql-base
docker stop 96f7f14e99ab
docker rm 96f7f14e99ab
docker run -d -v /data/docker/mysql-base:/var/lib/mysql --name mysql -p 33060:3306 mysql:latest
```
> 上述命令中，假设第一次启动的容器id为96f7f14e99ab。

正式测试的时候，则需要先复制一份mysql的基础数据目录，然后在启动的时候挂载这个备份的mysql数据目录即可。具体命令如下：
```bash
rm -fR /data/docker/mysql4testing
cp /data/docker/mysql-base /data/docker/mysql4testing
docker run -d -v /data/docker/mysql4testing:/var/lib/mysql --name mysql -p 3307:3306 mysql:latest
# do testing
docker stop 96f7f14e99ad
docker rm 96f7f14e99ad
```

## 修改Dockerfile
第一种方法虽然可以满足需求，但是需要每次都额外的cp一次mysql数据目录；如果想避免种操作，通过下面的步骤同样也可以实现：
1. 下载官方mysql的Dockerfile
1. 注释掉Dockerfile中`volume /var/lib/mysql`的那一行
1. 本地构建mysql镜像
1. 通过该镜像启动mysql容器
1. 对mysql容器进行数据初始化操作
1. 提交mysql容器变更并标志为新tag
1. 停止并删除mysql容器
1. 通过新的mysql镜像启动容器

最后启动的mysql容器，也会保留之前的数据库中的变更信息，以后每次想要恢复数据库，也只要重新启动一个新的容器即可。上述步骤对应的命令如下：
```bash
git clone https://github.com/mysql/mysql-docker
cd mysql-docker/5.7
sed 's/VOLUME /var/lib/mysql/# VOLUME /var/lib/mysql/' Dockerfile
doker build -f mysql:base .
docker run -d --name mysql -p 33060:3306 mysql:base  # container id: 96f7f14e99ab
# 进行测试数据初始化操作
docker commit -m "init database" 96f7f14e99ab mysql:init
docker stop 96f7f14e99ab
docker rm 96f7f14e99ab
docker run -d --name mysql -p 3307:3306 mysql:init
```
选择这种方案之后，正常执行自动化测试时，只需要每次重新启动一个容器就可以了。示例命令如下：
```bash
docker run -d --name mysql -p 3307:3306 mysql:init  # container id：96f7f14e99ad
# do testing
docker stop 96f7f14e99ad
docker rm 96f7f14e99ad
```

![关注二维码](https://www.testqa.cn/static/book.jpg)