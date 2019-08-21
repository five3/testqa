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
sudo yum install influxdb
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
启动grafana-server之后，就可以通过3000端口来范围web页面了，比如：http://localhost:3000。默认的账户为admin：admin，首次登录时会要求修改密码。登录后的界面如下：
![dashboard]()

### 配置数据源


### 配置dashboard


### 查看数据


