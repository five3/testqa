
# [node安装错误]/usr/bin/env: node: No such file or directory
这是因为没有给node添加到全局变量
```bash
ln -s /usr/local/node/bin/node /usr/bin/node
ln -s /usr/local/node/bin/npm /usr/bin/npm
```

# [appium在linux安装错误] Error: EACCES: permission denied, mkdir '/usr/local/lib/node_modules/appium/node_modules/appium-chromedriver
非root用户安装的node，使用如下命令：
```bash
sudo cnpm install -g appium
```
root用户安装的node，使用如下命令：
```bash
cnpm install -g appium --unsafe-perm=true --allow-root
```
`注意`：cnmp需要自己安装，方式如下：
```bash
npm install -g cnpm --registry=https://registry.npm.taobao.org
ln -s $NODE_HOME/lib/node_modules/cnpm/bin/cnpm /usr/bin/cnpm
```
`引用`：[https://github.com/appium/appium/issues/10020](https://github.com/appium/appium/issues/10020)
