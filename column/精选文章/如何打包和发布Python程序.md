# 如何打包和发布Python程序
> 在使用Python的过程中，我们经常需要做的一件事情就是通过pip来安装第三方的包。那么你是否也曾想过pip安装的包是怎么被打包并发布上去的呢？今天就来说一说Python的第三方包的打包与发布流程！

## 打包
想要发布一个第三方的包，首先你得有一个需要发布的项目。这个项目可以完成任何有意义的事情。比如：今天的样例就是一个Python的测试报告第三方库。

当我们把功能模块已经实现好之后，就可以按照python打包的目录结构要求来搭建了。具体步骤为：
- 新建一个目录作为根目录
- 把功能模块以Python包的形式放入根目录
- 在根目录中创建setup.py文件，并填写相应内容
- 在根目录创建其它描述文件，如：LISENCE，README等

这里以PyTestReport项目为例，介绍Python打包的目录结构，当然还可以有其它形式的目录结构。
```bash
PyTestReport
    |-- pytestreport
            |-- __init__.py
            |-- HTMLTestRunner.py
            |-- templates
                   |-- default.html
            |-- static
                   |-- css
                        |-- default.css
                   |-- js
                        |-- default.js
    |-- test
    |-- LICENSE
    |-- README.md
    |-- setup.py
```
上面的目录结构就是一个典型的Python打包目录结构。其中最重要的是setup.py文件，而这个项目的功能模块就是pytestreport这个包。接下来最重要的就是如何编写setup.py文件。

### 编写setup.py文件
直接上PyTestReport的参考样例，然后我们再看看几个重要的字段就基本可以了！
```python
#!/usr/bin/env python
# coding=utf-8
from setuptools import setup, find_packages

setup(
    name="PyTestReport",
    version="0.1.1",
    keywords=("test report", "python unit testing"),
    description="The HTML Report for Python unit testing Base on HTMLTestRunner",
    long_description="The HTML Report for Python unit testing Base on HTMLTestRunner",
    license="MIT",

    url="https://github.com/five3/PyTestReport",
    author="Xiaowu Chen",
    author_email="five3@163.com",

    package_dir={'pytestreport': 'pytestreport'},         # 指定哪些包的文件被映射到哪个源码包
    packages=['pytestreport'],       # 需要打包的目录。如果多个的话，可以使用find_packages()自动发现
    include_package_data=True,
    py_modules=[],          # 需要打包的python文件列表
    data_files=['pytestreport/templates/default.html', 'pytestreport/static/css/default.css', 'pytestreport/static/js/default.js'],          # 打包时需要打包的数据文件
    platforms="any",
    install_requires=[      # 需要安装的依赖包
        'Flask>=1.0.2'
    ],
    scripts=[],             # 安装时复制到PATH路径的脚本文件
    entry_points={
        'console_scripts': [    # 配置生成命令行工具及入口
            'PyTestReport.shell = pytestreport:shell',
            'PyTestReport.web = pytestreport:web'
        ]
    },
    classifiers=[           # 程序的所属分类列表
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],
    zip_safe=False
)
```
这个文件的上半部分字段可以自行查阅即可。这里有几个可能入坑的字段来看一下。更多详情可参考附录部分！
- packages：需要打包的Python包目录；注意如果有子包则必须显式的列出来，如：pytestreport.api
- data_files：需要打包的非.py文件；
- entry_points：指定安装包之后的程序入口脚本；比如：pip命令就是通过这种方式创建的

### 编译
项目目录结构和setup.py文件都就绪之后，就可以开始编译并打包了；首先最好升级下打包相关的基础库。
```bash
python -m pip install --upgrade pip
pip install --upgrade setuptools wheel
```
接着，从命令行进入项目的根目录，通过如下命令即可进行编译打包操作：
```bash
python setup.py sdist       # 打源码包
python setup.py bdist       # 打二进制包
python setup.py bdist_egg       # 打egg包
python setup.py bdist_wheel     # 打wheel包
```
执行完命令之后，会在项目的根目录创建一个dist目录，所有打包的文件都存放在此目录下。

在打包后，可以本地进行安装和使用测试，当然你也可以通过如下命令直接安装：
```bash
python setup.py build
python setup.py install
```

## 发布
当我们的项目打包并测试完成之后，就可以开始发布之旅了！首先你得需要安装另一个基础库。
```bash
pip install twine
```
此外，你还需要在PYPI的官网上进行账号的注册！当然官方会建议你先在一个叫TestPYPI的测试镜像服务上先进行预发布。当在TestPYPI服务上发布成功并进行完安装和使用测试之后，再把项目包发布到PYPI服务上。

### TestPYPI发布
首先在[https://test.pypi.org/account/register/](https://test.pypi.org/account/register/)注册一个账户。然后在项目根目录执行如下命令：
```bash
twine upload --repository-url https://test.pypi.org/legacy/ dist/*
# entry your username and password
```
过程中会需要你输入注册的账号信息，等上传完成之后可以通过如下的方式来进行包的安装。
```bash
pip install --index-url https://test.pypi.org/simple/ PyTestReport
```
> 你也可以直接通过[https://test.pypi.org/manage/projects/](https://test.pypi.org/manage/projects/)来查看你已经上传的项目，并通过点击[View]来查看项目的具体信息。
![PYPI Projects](https://github.com/five3/testqa/blob/master/images/pypi_001.png?raw=true)

安装完成之后，则需要测试下安装包是否能正常的工作，指定的入口脚本是否安装并正常使用，如果一切正常那么恭喜你了发布到正式的PYPI服务了！

### PYPI发布
同样的你需要在PYPI的官网[https://pypi.org/account/register/](https://pypi.org/account/register/)注册一个账号。然后执行一个上传操作:
```bash
twine upload dist/*
# entry your username and password
```
上传完成之后通过如下命令可直接安装：
```bash
pip install PyTestReport
```
同样的，你也可以通过[https://pypi.org/manage/projects/](https://pypi.org/manage/projects/)来查看和管理已上传的项目。
![PYPI Projects](https://github.com/five3/testqa/blob/master/images/pypi_002.png?raw=true)

## 附录
- [https://packaging.python.org/tutorials/packaging-projects/](https://packaging.python.org/tutorials/packaging-projects/)
- [http://blog.konghy.cn/2018/04/29/setup-dot-py/](http://blog.konghy.cn/2018/04/29/setup-dot-py/)
- [http://www.cnblogs.com/UnGeek/p/5922630.html](http://www.cnblogs.com/UnGeek/p/5922630.html)

# 新书推荐
![Python Web自动化测试设计与实现](https://img-blog.csdnimg.cn/20190117100818307.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)

获取更多关于Python和自动化测试的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
 