# Grafana安装与简易使用指南

Grafana是一个跨平台的开源的度量分析和可视化工具，可以通过将采集的数据查询然后可视化的展示，并及时通知。它主要有以下六大特点：
- 展示方式：快速灵活的客户端图表，面板插件有许多不同方式的可视化指标和日志，官方库中具有丰富的仪表盘插件，比如热图、折线图、图表等多种展示方式；
- 多数据源支持：Graphite，InfluxDB，OpenTSDB，Prometheus，Elasticsearch，CloudWatch和KairosDB等；
- 通知提醒：以可视方式定义最重要指标的警报规则，Grafana将不断计算并发送通知，在数据达到阈值时通过Slack、PagerDuty等获得通知；
- 混合展示：在同一图表中混合使用不同的数据源，可以基于每个查询指定数据源，甚至自定义数据源；
- 注释：使用来自不同数据源的丰富事件注释图表，将鼠标悬停在事件上会显示完整的事件元数据和标记；
- 过滤器：Ad-hoc过滤器允许动态创建新的键/值过滤器，这些过滤器会自动应用于使用该数据源的所有查询。

这里我们介绍它，自然是与InfluxDB结合来展示性能监控平台的数据，由于它有良好的图表和高度的查询定制能力，所以非常适合用于性能监控数据的实时展示。

## 安装
> 官方下载地址： https://grafana.com/grafana/download

### YUM安装
对于Centos用户，可以用下面的命令添加InfluxDB的仓库
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
```
然后安装、运行InfluxDB服务：
```bash
sudo yum install grafana
sudo service grafana-server start
sudo /sbin/chkconfig --add grafana-server  ## service服务开机自启动
```
如果你的系统可以使用Systemd，也可以这样启动：
```bash
sudo yum install grafana-server
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server.service  ## systemd服务开机自启动
```

### RPM安装
假定你跟我一样是Centos的用户，那么安装命令是这样的：
```bash
wget https://dl.grafana.com/oss/release/grafana-6.3.3-1.x86_64.rpm
sudo rpm -ivh grafana-6.3.3-1.x86_64.rpm
grafana-server -config /etc/grafana/grafana.ini -homepath /usr/share/grafana
```

### 汉化
[Grafana v5.2.1版本汉化方法](https://blog.csdn.net/w958660278/article/details/80437978)

## 使用
启动grafana-server之后，就可以通过3000端口来访问web页面了，比如：[http://localhost:3000](http://localhost:3000)。 默认的账户为admin：admin，首次登录时会要求修改密码。登录后的界面如下：
![dashboard](https://raw.githubusercontent.com/five3/testqa/master/images/grafana/grafana-001.png)

### 配置数据源
登录之后，首先要做的事情就是添加数据源；前面我们也提到过`grafana`可以支持很多数据源，这里只选择`influxdb`作为数据源，其添加内容的界面如下：
![datasource](https://raw.githubusercontent.com/five3/testqa/master/images/grafana/grafana-002.png)

只需要选择好数据类型为`influxdb`，然后配置好相应的访问url和数据库即可。

### 配置dashboard
接下来就是添加面板，也就是想要展示数据的图表；`influxdb`不仅支持普通曲线图，还是支持很多的可视化图；此外还支持使用第三方已经配置好的模板和插件，非常的好用。

比如：对于jmeter性能数据就有专门的第三方模板，直接使用即可无需自己配置；还有像nginx、redis、mysql、mongo等都有专门的三方模板可以选用。

而如果你只想安静地展示自己独有的数据，那么你也可以选择自定义一个模板；grafana新建模板有2种方式可选，图示如下：
![create](https://raw.githubusercontent.com/five3/testqa/master/images/grafana/grafana-003.png)

- Add Query是添加一个普通的曲线图表来展示数据
- Visualization是添加一个可视化的图表，就是那种比较炫酷的图形

这2种方式的配置步骤和内容基本一样，只是对于图形展示的属性有所不同而已；而最重要的就是配置`influxdb`的数据读取语句。
![query](https://raw.githubusercontent.com/five3/testqa/master/images/grafana/grafana-004.png)
这个样例里从cpu_load表中读取value字段的数值并计算平均值再展示。

### 查看数据
配置好面板的基本数据之后，记得保存然后返回主面板页面，默认显示为No Data，需要你插入一些真实数据，比如我插入的数据如下：
```bash
INSERT cpu_load,host=serverA,region=us_west value=0.14 1566618365111359200
INSERT cpu_load,host=serverA,region=us_west value=0.24 1566619375111135920
INSERT cpu_load,host=serverA,region=us_west value=0.34 1566620385111135920
INSERT cpu_load,host=serverA,region=us_west value=0.53 1566621395111135920
INSERT cpu_load,host=serverA,region=us_west value=0.68 1566622405111135920
INSERT cpu_load,host=serverA,region=us_west value=0.78 1566623415111135920
INSERT cpu_load,host=serverA,region=us_west value=0.84 1566624425111135920
INSERT cpu_load,host=serverA,region=us_west value=0.94 1566625435111135920
INSERT cpu_load,host=serverA,region=us_west value=0.75 1566626445111135920
INSERT cpu_load,host=serverA,region=us_west value=0.63 1566627455111135920
INSERT cpu_load,host=serverA,region=us_west value=0.56 1566628465111135920
INSERT cpu_load,host=serverA,region=us_west value=0.73 1566629475111135920
INSERT cpu_load,host=serverA,region=us_west value=0.64 1566630485111135920
INSERT cpu_load,host=serverA,region=us_west value=0.58 1566631495111135920
```

插入之后就能看到数据的展示情况了。如下图：
![show](https://raw.githubusercontent.com/five3/testqa/master/images/grafana/grafana-005.png)
这里刚好配置了2种形式的图表，上面是普通的，下面则是可视化的；现在知道它们的区别了吧！

# 总结
`grafana`可以说是即强大又简单的数据展示工具，可以支持很多的数据源，提供了很多的图表格式，还有三方模板库可以直接使用；而对于简单需求的配置却是如此的简单。

获取更多关于Python和测试开发相关的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/2019072113294796.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)
