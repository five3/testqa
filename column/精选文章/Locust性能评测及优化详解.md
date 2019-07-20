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
- ab
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
    host = "http://10.168.xx.xx"
    min_wait = 0
    max_wait = 0
```
启动Locust的命令如下：
```bash
# 单实例
locust -f performance.py --no-web -c 2 -r 2 -t 5s
# 分布式
locust -f performance.py --master
locust -f performance.py --slave
# 访问http://127.0.0.1:8089 启动压测
```

不同并发和实例的压测结果如下：
![locust](https://github.com/five3/testqa/blob/master/images/locust/locust-result.png?raw=true)

> 注：分布式场景下，locust停止默认client貌似有bug,web端停止不了。

#### Jmeter
对于Jmeter工具，首先设置JVM堆大小为固定2G，不设置思考时间，默认勾选keep-alive。分别使用不同的并发数进行场景压测，最终评测出最优并发用户数和最大QPS。

Jmeter的HTTP请求设置如下：
![jmeter-config](https://github.com/five3/testqa/blob/master/images/locust/jmeter-config.png?raw=true)

启动Jmeter的命令如下：
```bash
sh jmeter -n -t ../xxx.jmx -l /data/xxxx.jtl
```

不同并发数下的压测结果如下：
![jmeter-result](https://github.com/five3/testqa/blob/master/images/locust/jmeter-result.png?raw=true)


#### ab
ab是apache服务器中的一个压测工具，如果你不想安装整个apache，那么你可以直接安装httpd-tools即可。ab可以通过-k参数开启keep-alive模式，同时可以指定并发数和请求总数。

ab的启动命令及参数如下：
```bash
./ab -n 6000000 -c 150 http://10.168.xx.xx/index/index.html
```
ab不同并发数下的压测结果如下：
![ab-result](https://github.com/five3/testqa/blob/master/images/locust/ab-result.png?raw=true)

> 为什么ab做了这么多次测试呢？因为本来没有想过能压到这么高的并发。另外会发现使用keep-alive性能会提升很高。

#### http_load
http_load工具需要下载后在本地编译，由于http_load不支持keep-alive设置，所以只能指定并发数和请求总数。具体的压测命令如下：
```bash
./http_load -p 100 -f 6000000 http://10.168.xx.xx/index/index.html
```
http_load不同并发数下的压测结果如下：
![http_load-result](https://github.com/five3/testqa/blob/master/images/locust/http_load-result.png?raw=true)

> 因为http_load不支持设置keep-alive，所以它的数据和ab不使用keep-alive时差不多。

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
![requests](https://github.com/five3/testqa/blob/master/images/locust/requestsclient.png?raw=true)

从结果可以看出，requests.session确实默认是支持keep-alive的。所以如果使用locust的默认client，这块是不需要优化的了。

#### 替换为urllib3实现client
因为requests底层使用的是urllib3库，所以这里我们也尝试直接使用urllib3作为locust的client，看在性能上是否有提升。client代码如下：
```python

```

从urllib3请求时录制的TCP通信可以看出，它默认也是使用了keep-alive模式。
![urlib3client](https://github.com/five3/testqa/blob/master/images/locust/urlib3client.png?raw=true)

具体压测执行结果如下：
![urllib3client](https://github.com/five3/testqa/blob/master/images/locust/urllib3-result.png?raw=true)

#### 替换为socket实现client
本来准备继续使用socket来实现client，但是TCP协议编程这块有坑，没有达到理想的效果，这个坑先留着日后再填！

#### 替换为go实现client
在查找Locust优化方案的时候，发现已经有人实现了go语言的client。github地址：[https://github.com/myzhan/boomer](https://github.com/myzhan/boomer)，安装步骤也很简单，按照项目说明即可很快完成。

使用go语言的client也很方便，只要把原来启动slave的命令替换为启动go程序即可。具体命令如下：
```bash
locust -f performance.py --master
./http.out --url http://10.168.xx.xx/index/index.html
```
不同并发数下的压测结果如下：
![go-result](https://github.com/five3/testqa/blob/master/images/locust/go-result.png?raw=true)

在boomer项目里，实现了2个版本的go客户端；除了上面的那个，还有一个fast版本的，启动命令如下：
```bash
locust -f performance.py --master
./fasthttp.out --url http://10.168.xx.xx/index/index.html
```
不同并发数下的压测结果如下：
![fastgo-result](https://github.com/five3/testqa/blob/master/images/locust/fastgo-result.png?raw=true)

> 注意：普通版本client和fast版本的对应go文件分别为`examples/http/client.go`和`examples/fasthttp/client.go`。

# 总结
从当前评测的结果来看，python实现的客户端在压力生成上并没有优势；而像ab这样的工具在场景支持上却不够丰富；如果希望2者兼得，那么go版本的locust客户端或许是个不错的选择！

locust官方github上有一个issue，相关人员对于locust施压能力不足的解释是：“locust主要解决场景开发效率问题，而不是解决生成压力的问题，因为人效成本远大于硬件的成本”。

如果你也在关注和学习性能，或者有觉得有出入的地方，那么欢迎一起来积极讨论，挖掘出即好用又高效的压测方案！

获取更多关于Python和自动化测试的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
