@[TOC] 

# 安装Android SDK
国内Android相关资源包下载地址：http://tools.android-studio.org/index.php/sdk
```bash
wget http://dl.google.com/android/android-sdk_r24.4.1-linux.tgz
tar -zxvf android-sdk_r24.4.1-linux.tgz
cd android-sdk-linux/tools
./android update sdk --no-ui        # 更新SDK，下载platform-tools、add-ons
```
安装完成后执行adb命令提示错误`libc.so.6: version GLIBC_2.14 not found`。安装网上搜索的文章按步骤解决即可。
[https://blog.csdn.net/heylun/article/details/78833050](https://blog.csdn.net/heylun/article/details/78833050)。
最后要把adb添加环境变量即可。 

# 安装nodejs、NPM、cnpm
cnpm = cn + npm。即国内npm，由淘宝团队维护的国内npm完全镜像，常用于安装被墙的node包。
```bash
wget https://nodejs.org/dist/v10.15.3/node-v10.15.3-linux-x64.tar.xz
xz -d node-v10.15.3-linux-x64.tar.xz
tar -xvf node-v10.15.3-linux-x64.tar
mv node-v10.15.3-linux-x64 /usr/local/node
ln -s /usr/local/node/bin/node /usr/bin/node
ln -s /usr/local/node/bin/npm /usr/bin/npm
# 安装cnpm
npm install -g cnpm --registry=https://registry.npm.taobao.org
ln -s /usr/local/node/lib/node_modules/cnpm/bin/cnpm /usr/bin/cnpm
```

# 安装Appium
如果的node是通过非root用户安装的，那么appium的安装命令如下：
```bash
sudo cnpm install -g appium
```
如果你是通过root用户安装的话，则需要使用下面的命令来安装appium：
```bash
cnpm install -g appium --unsafe-perm=true --allow-root
```

# 安装Python和基础库
通过源码或者pyenv安装指定版本的Python，在通过pip安装appium-client。
```bash
pip install Appium-Python-Client
```

# 设置WIFI连接及调试
手机通过USB线连接到电脑，设置手机支持USB调试，允许电脑进行设备访问权限。
```bash
adb tcpip 9999
# 拔掉usb线
adb connect mobile.ip:9999
```
appium启动成功，adb通过wifi连接成功，通过如下代码测试环境是否正常。
```python
from appium import webdriver

desired_caps = dict()
desired_caps['platformName'] = 'Android'
desired_caps['platformVersion'] = '6.0.1'
desired_caps['deviceName'] = 'a84fcc5c'
desired_caps['appPackage'] = 'com.xxx.xxx'  # 相应修改
desired_caps['appActivity'] = '.xxxActivity'  # 相应修改
driver = webdriver.Remote('http://pcma.corpautohome.com:4723/wd/hub', desired_caps)
print(driver.page_source)
driver.quit()
```

# 错误解决方法
## [node安装错误]/usr/bin/env: node: No such file or directory
这是因为没有给node添加到全局变量
```bash
ln -s /usr/local/node/bin/node /usr/bin/node
ln -s /usr/local/node/bin/npm /usr/bin/npm
```

## [appium在linux安装错误] Error: EACCES: permission denied, mkdir '/usr/local/lib/node_modules/appium/node_modules/appium-chromedriver
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

> 获取更多感兴趣的文章，请扫描如下二维码！
![https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
 