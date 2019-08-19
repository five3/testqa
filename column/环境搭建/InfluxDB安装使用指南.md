# InfluxDB安装使用指南
InfluxDB是目前比较主流的时序数据库，而时序数据库则是以时间序列为轴的数据库，与关系型数据库相比它有几个特点：
- 每条记录都必须有时间戳字段（不设置会自动生成，类似关系型数据库的主键）
- 提供海量数据的写入和读取能力
- 提供针对时序的聚合函数，方便查询数据的聚合
- 没有固定的schema设计

之所时序数据库要被设计成包含这些特性，是因为它天生就是为特定场景业务而生的；主要针对那些写多读少、大量数据写入需求、按时间维度进行聚合查询的业务场景，比如：数据监控。

数据监控方面细分还是可以分出很多的场景；比如：气象数据、天文数据、人口分布、工资水平、运维资源等等，生活中方方面面的行业都可以使用的到，而在时序数据库之前，人们通常都会使用关系型数据库来代替，但显然需要付出更大的代价才能满足需求。

同样的在测试领域中也是有很多的业务数据，可以使用到时序数据库；比如：产品质量数据，性能压测数据、服务器资源数据等等；所以今天就来介绍下如何安装和简单使用时序数据库。后面再分享如何基于时序数据库展示图表。

## 安装
> 安装InfluxDB包需要root或是有管理员权限才可以。

对于Ubuntu用户，可以用下面的命令添加InfluxDB的仓库
```bash
curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
```
然后安装、运行InfluxDB服务：
```bash
sudo apt-get update && sudo apt-get install influxdb
sudo service influxdb start
```
如果你的系统可以使用Systemd(比如Ubuntu 15.04+, Debian 8+），也可以这样启动：
```bash
sudo apt-get update && sudo apt-get install influxdb
sudo systemctl start influxdb
```

## 配置
首先，你可以通过命令influxd config来查看默认配置，而配置文件的路径为：`/etc/influxdb/influxdb.conf`，想要某个配置项生效则直接取消注释并设置相应值即可。

另外，想要用自定义的配置文件来运行InfluxDB可以有两种方法：
- 运行的时候通过可选参数-config来指定：
```bash
influxd -config /etc/influxdb/influxdb.conf
```
- 设置环境变量INFLUXDB_CONFIG_PATH来指定，例如：
```bash
echo $INFLUXDB_CONFIG_PATH
/etc/influxdb/influxdb.conf

influxd
```
其中-config的优先级高于环境变量。

## 操作
安装完成之后，我们再来看看如何访问和使用InfluxDB进行数据操作。默认会提供一个`influx`命令行工具（原理是发送HTTP请求），它会默认连接本机的InfluxDB服务；当然你也可以通过发送HTTP请求来完成相同的操作。

另外在具体操作之前，我们可以理解下时序数据库与关系型数据库在概念上的差异和对标。具体如下：

| 关系型数据库 | 时序数据库 |
| --- | --- |
| database | database |
| table | measurement |
| row | point |
| index field | tag |
| field | field |
| primary key | timestamp |

### database操作
```bash
> influx -precision rfc3339
Connected to http://localhost:8086 version 1.2.x
InfluxDB shell 1.2.x
> CREATE DATABASE mydb
> SHOW DATABASES
name: databases
---------------
name
_internal
mydb

> USE mydb
Using database mydb

> 
```
可以看到除了第一条命令跟mysql的稍微有点差异，其它的命令跟mysql的如出一辙，这说明在用户友好型方面也是做了考虑，避免大家重复学习无用的内容。

### 插入记录
```bash
> INSERT cpu,host=serverA,region=us_west value=0.64
```
cpu是measurement（table）的名称；host，region是tag（index field）的名称；value是field的名称；整行则是一个point（row）数据；默认这条记录被存储在当前的database下。

### 查询记录
```bash
> SELECT "host", "region", "value" FROM "cpu"
name: cpu
---------
time                                     host         region   value
2015-10-21T19:28:07.580664347Z  serverA      us_west     0.64

```
这语法跟mysql几乎没有区别，这里主键time是自动生成的，因为插入时没有带。

> 时序数据库一般只进行CR操作，而UD操作通常很少执行，所以大部分的时候不会涉及到更新和删除语法。

## Python接口
上面介绍的操作都是通过InfluxDB自带的influx命令行工具操作的，而在程序化时我们则可以直接通过其HTTP接口来执行同样的操作，下面就介绍如何通过Python来进行InfluxDB的操作。

### database操作
```python
import requests
"""
数据库查询相关的HTTP请求内容如下：
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE mydb"
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=SHOW DATABASES"
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=USE mydb"
"""

r = requests.post("http://localhost:8086/query", data="q=CREATE DATABASE mydb")
print(r.text)
r = requests.post("http://localhost:8086/query", data="q=SHOW DATABASES")
print(r.text)
r = requests.post("http://localhost:8086/query", data="q=USE mydb")
print(r.text)
```

### 插入记录
```python
import requests
"""
curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'
"""

r = requests.post("http://localhost:8086/write?db=mydb", data=b"cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000")
print(r.text)

r = requests.post("http://localhost:8086/write?db=mydb", file=open('points.txt', 'rb'))
print(r.text)
```

### 查询记录
```python
import requests
"""
curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mydb" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
"""

data = {"db": "mydb", "q": "SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"}
r = requests.post("http://localhost:8086/query?pretty=true", data=data)
print(r.text)
```

## Python三方库
很显然，像上面这种操作怎么可能没有第三方库呢，所以直接安装第三方库可能是最方便的选择。
```bash
pip install influxdb
```
具体的API使用样例如下：
```python
from influxdb import InfluxDBClient

json_body = [
    {
        "measurement": "cpu_load_short",
        "tags": {
            "host": "server01",
            "region": "us-west"
        },
        "time": "2009-11-10T23:00:00Z",
        "fields": {
            "value": 0.64
        }
    }
]

client = InfluxDBClient('localhost', 8086, 'root', 'root', 'mydb')
client.create_database('mydb')
client.write_points(json_body)
result = client.query('select value from cpu_load_short;')
print("Result: {0}".format(result))
```

## 总结
关于InfluxDB的简单介绍就到这里，如果想更深层次的了解和使用InfluxDB，可以去官网查阅相关内容。这篇文章仅仅介绍了如何安装InfluxDB本身，而存储数据的本质其实是用于查询和展示，后面会有文章介绍如何与grafana结合并展示图表数据。

获取更多关于Python和测试开发相关的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/2019072113294796.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)
