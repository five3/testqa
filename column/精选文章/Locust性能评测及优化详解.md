# Locust性能评测及优化详解

> 这篇文章是用来补前一篇文章挖的坑，在解析了Locust的整体流程之后，还是要回归落地，看看它到底好不好用，能不能用。

## 性能评测
在《性能测试工具Locust源码浅析》中，我们进行了一个主流程的分析。本次我们将对Locust进行实际的评测，在具体的评测之前，为了评测结果尽量的准确，我们需要做如下的规约：
- 服务器端没有性能瓶颈（假设有无限能力）
- 系统环境没有限制设定（网络连接数无限制，TIME_WAIT回收及时）
- 外部环境没有额外消耗（网络监控软件、限流软件没有启动）
- 网络带宽没有瓶颈
- 不同待评测工具在同一台机器上进行评测（中间预留足够的资源回收时间）

### 环境准备
#### 1、压测环境准备
- 机器配置：4核8G
- 操作系统：CentOS（尽量选择Linux系统）
- 网络环境：千兆局域网
- 文件句柄数限制设定：65536
- socket连接回收时间：30ms

#### 2、服务环境准备
- 服务端服务：nginx 8 worker挂载一个静态文件（hello world）
- 机器配置：4核8G
- 操作系统：CentOS（尽量选择Linux系统）
- 网络环境：千兆局域网
- 文件句柄数限制设定：65536
- socket连接设置：net.ipv4.tcp_tw_reuse=1，net.ipv4.tcp_timestamps=1，net.ipv4.tcp_tw_recycle=1，net.ipv4.tcp_fin_timeout = 30

#### 3、压测工具准备
- Locust
- Jmeter
- http_load

####  压测开始
在同一套环境分别使用不同的工具来进行相同场景的请求，这里只发送一个请求hello world的静态文件。不同测试之间停留10分钟以上间隔，以保证2台机器各自资源的回收。
- CPU、内存
- Load Avg（系统队列长度）
- socket连接数
- Window Size（TCP窗口）

#### Locust
针对Locust先使用单实例进行压测，脚本中设置min_wait和max_wait均为0；由于Locust使用的是requests.session来发起请求，所以默认支持http的keep-alive；在单实例执行完成后，使用4实例来进行相同场景的压测。

具体的压测脚本如下：
```python
from locust import HttpLocust, TaskSet, task

class WebsiteTasks(TaskSet):
    @task
    def index(self):
        self.client.get("/")

class WebsiteUser(HttpLocust):
    task_set = WebsiteTasks
    host = "http://10.0.0.1"
    min_wait = 0
    max_wait = 0
```
启动Locust的命令如下：
```bash

```

不同并发和实例的压测结果如下：
![]()

#### Jmeter
对于Jmeter工具，首先设置JVM堆大小为固定2G，不设置思考时间，默认勾选keep-alive。分别使用不同的并发数进行场景压测，最终评测出最优并发用户数和最大QPS。

Jmeter的HTTP请求设置如下：
![]()

启动Jmeter的命令如下：
```bash

```

不同并发数下的压测结果如下：
![]()


#### ab
ab是apache服务器中的一个压测工具，如果你不想安装整个apache，那么你可以直接安装httpd-tools即可。ab可以通过-k参数开启keep-alive模式，同时可以指定并发数和请求总数。

ab的启动命令及参数如下：
```bash

```
ab不同并发数下的压测结果如下：
![]()

#### http_load
http_load工具需要下载后在本地编译，由于http_load不支持keep-alive设置，所以只能指定并发数和请求总数。具体的压测命令如下：
```bash

```
http_load不同并发数下的压测结果如下：
![]()


### 压测说明
由于压测场景比较单一，所以数据只能代表在该场景下，各工具在压测能力上的不同体现。如果换作另外的场景，可能工具之间的性能表现会有所变化。但总体来讲应该不会有太多的可变性。

所以各工具的压测能力，基本上与其实现的语言执行效率成正比。C > JAVA > Python。另外，在使用keep-alive的情况下，确实会提高通信性能。

## 性能优化
通过上面简单的对几个工具的评测，从这组数据的体现来讲，Locust是最弱的，Jmeter和网络上的评测结果接近。但是因为Locust属于Python系列，所以还是抱着希望来看看Locust是否还有优化的潜力。

### Locust优化项
为了尝试给Locust进行性能提升，收集并思考从如下几种方式来进行尝试：
- 思考时间设置为0（默认为1秒，上述已设置）
- 使用keep-alive模式（默认为keep-alive，待确认是否生效）
- 替换为urllib3基础库（requests是基于urllib3进行的封装）
- 替换为使用socket库发送请求
- 替换为go实现的客户端发送请求

#### 测试Locust默认是否为keep-alive
为了检测是否使用了keep-alive，可以通过wireshark来进行抓包，并查看不同请求是否复用了一个TCP连接；如果是则为keep-alive，否则就不是keep-alive模式。





