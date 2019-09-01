# Telegraf安装与简易使用指南

![banner](https://img-blog.csdnimg.cn/20190829223407940.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

前面我们依次介绍了Influxdb、Grafana的安装和基本使用，这次我们来看看Telegraf的安装与基本使用。Telegraf是InfluxData旗下的数据采集工具，主要用来采集各类服务的信息数据，比如：系统cpu、内存，redis、nginx等服务；跟Influxdb是兄弟产品。

Telegraf、Influxdb、Grafana三者一起共同组成性能监控的三驾马车；数据采集、数据存储、数据展示。除此之外，我们还可以把性能压测数据也一并用这套系统管理起来，完整的性能监控平台的数据结构是这样的。
![](001)

今天我们还是主要介绍Telegraf的相关基本信息，它除了可以监控windows和linux的系统资源，常用服务之外，还可以通过插件扩展来定制自己想要的采集行为，可以说是即强大又灵活。

## 安装   

### YUM安装
对于Centos用户，可以用yum安装
```bash
yum install telegraf
systemctl start telegraf
systemctl restart telegraf
systemctl status telegraf
```
或者先下载在安装：
```bash
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.11.5-1.x86_64.rpm
sudo yum localinstall telegraf-1.11.5-1.x86_64.rpm
systemctl enable telegraf.service  ## systemd服务开机自启动
telegraf -config /etc/telegraf/telegraf.conf  # 手动启动
```

## 使用
### 配置
在正式启动之前，需要先进行相关配置，告诉telegraf对哪些数据进行采集，配置文件地址`/etc/telegraf/telegraf.conf`.修改如下内容：
```bash
[[outputs.influxdb]]
   urls = ["http://localhost:8086"] # required 
   database = "telegraf" # required
   retention_policy = ""
   precision = "s"
   timeout = "5s"
   username = ""
   password = ""
```
主要是配置一个influxdb的地址和数据库，让telegraf知道把数据存放到哪里。当telegraf服务启动之后，我们就可以去influxdb查看相应数据是否提交成功，成功后再通过Grafana来配置数据展示。

默认telegraf会采集cpu、内存、磁盘的数据信息，这里是采集的数据样例通过Grafana配置之后的展示效果如下：
![](002)

采集的数据内容非常的多，这里只配置了几个指标值而已。

### 插件使用
telegraf的常用的插件有2类：一类是input，一类是output；所谓input就是采集插件，比如：系统cpu、网络。所谓output就是数据存储插件，比如：influxdb、kafka等。

telegraf默认情况下开启的output插件是influxdb，并且默认配置到本机地址，可以根据需要修改。而input插件正如上面所示默认仅开启了cpu、内存等差插件，需要使用其它扩展插件就需要手动开启并配置。

配置插件的方式也有两种：一种是直接在默认配置文件中修改，因为它包含了几乎所有支持的配置项，但是非默认的都被注释掉了；另一种是新生成一个配置文件，并存放在`/etc/telegraf/telegraf.d`目录下，这样就可以支持多插件配置文件了。

生成一个telegraf配置文件的命令：
```bash
# 当前目录下生成一个telegraf的默认配置文件
telegraf config > telegraf.conf   
# 当前目录下生成一个包含cpu、内存、磁盘、磁盘io、网络作为输入插件，以及influxdb作为输出插件的配置文件
telegraf --input-filter cpu:mem:disk:diskio:net --output-filter influxdb config > telegraf.conf
```

除了采集默认的系统数据，telegraf还可以采集mysql、redis、nginx、apache、prometheus等服务，这里以采集nginx服务数据为例，介绍如何配置插件。

首先，确保先有一个nginx的服务，且该ngixn安装时支持`http_stub_status_module`模块，通过`nginx -v`可以查看到是否安装了此模块。如果没有安装的话则需要重新编译，因为就是通过该模块来监控nginx的。

如果nginx服务已经带有`http_stub_status_module`模块，则需要在nginx配置添加对应的请求入口，来返回nginx的状态信息。样例如下：
```bash
location /nginx-status {
       allow 127.0.0.1; # 允许的IP
       deny all;
       stub_status on;
       access_log off;
}
```
执行`nginx -s reload`命令使修改配置生效，再通过`curl http://127.0.0.1/nginx-status`命令来查看是否能正常获取信息。
![](003)

> active connections – 活跃的连接数量
server accepts handled requests — 总共处理了11989个连接 , 成功创建11989次握手, 总共处理了11991个请求
reading — 读取客户端的连接数.
writing — 响应数据到客户端的数量
waiting — 开启 keep-alive 的情况下,这个值等于 active – (reading+writing), 意思就是 Nginx 已经处理完正在等候下一次请求指令的驻留连接.

接着，新生产一个采集nginx的配置文件，并存储到`/etc/telegraf/telegraf.d`目录下:
```bash
cd /etc/telegraf/telegraf.d
telegraf --input-filter nginx --output-filter influxdb config > nginx.conf
```
修改`/etc/telegraf/telegraf.d/nginx.conf`的内容如下：
```bash
[[inputs.nginx]]
  # An array of Nginx stub_status URI to gather stats.
  urls = ["http://localhost/nginx-status"]

  ## Optional TLS Config
#  tls_ca = "/etc/telegraf/ca.pem"
#  tls_cert = "/etc/telegraf/cert.cer"
#  tls_key = "/etc/telegraf/key.key"
  ## Use TLS but skip chain & host verification
#  insecure_skip_verify = false

  # HTTP response timeout (default: 5s)
  response_timeout = "5s"
```
测试下nginx插件配置文件格式是否正确。
```bash
telegraf  -config /etc/telegraf/telegraf.d/nginx.conf -input-filter nginx -test
```
输出内容如下则表示正常：
```bash
2019-09-01T14:04:38Z I! Starting Telegraf 1.11.5
> nginx,host=861a6da23d20,port=80,server=localhost accepts=5i,active=1i,handled=5i,reading=0i,requests=5i,waiting=0i,writing=1i 1567346678000000000
```
最后，我们还需要重启下`telegraf`服务，让新增的插件配置生效。最后查看nginx监控数据的效果如下：
![](004)

# 总结
`telegraf`是一个非常强大且跨平台，可以说开箱即用的工具，只需简单的部署和配置就能采集到丰富的数据，而且还支持非常方便的扩展。配合influxdb、grafana等工具一起就可以轻松实现性能监控平台搭建。

获取更多关于Python和测试开发相关的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190829223353609.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)
