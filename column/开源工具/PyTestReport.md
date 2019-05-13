# PyTestReport单元测试报告框架

> 如果你是Java栈的同学，那么你可能知道extent report测试报告框架。而Python你可能知道HTMLTestRunner测试报告框架。今天要介绍的则是基于HTMLTestRunner的新测试报告框架PyTestReport。

如果Python语言界已经有了`HTMLTestRunner`，那么为什么还要一个`PyTestReport`测试框架呢？原因很简单，因为与`Extent Report`框架相比，`HTMLTestRunner`在界面的优越性上面还是有不少的差距，而`PyTestReport`则意在成为Python语言中的`Extent Report`。

## 开局一张图
![PyTestReport Sample](https://github.com/five3/testqa/blob/master/images/default_theme.png?raw=true)

这个就是`PyTestReport`的默认主题模板，虽然看上去跟`Extent Report`的众多模块相比，还是稍有逊色显得不是很专业。但是这并不影响我们介绍这款框架，因为它在基于`HTMLTestRunner`进行改良之后开始支持模板扩展了，也就是说将来它可以拥有和`Extent Report`同步样式的报告模板。

所以如果你对此框架还有兴趣，就一起来看下如果快速的使用和扩展该框架的主题模板。如果你具有较好的CSS功底，那么欢迎来为我们的主题模板舔砖加瓦^_^！开源合作相关请点击[PyTestReport的github地址](https://github.com/five3/PyTestReport.git)查看详情。

## 安装
### 通过pip安装
```bash
pip install PyTestReport 
```

### 通过安装包
可通过发布的安装包进行安装，具体安装包可在dist目录查找。
```bash
pip install PyTestReport-0.1.X-py3-none-any.whl
```

### 通过源码（最新版本）
```bash
pip install git+https://github.com/five3/PyTestReport.git
```
或者
```bash
git clone https://github.com/five3/PyTestReport.git
cd PyTestReport
python setup.py build
python setup.py install
```

## 使用
PyTestReport可用通过多种方式运行，分别如下：
- 单元测试 
- lib库引入（后续支持）
- 命令行（后续支持）
- REST API（后续支持）

### 单元测试使用样例
```python
import unittest
import pytestreport

class MyTest(unittest.TestCase):
    def testTrue(self):
        self.assertTrue(True)
        
if __name__ == '__main__':
    pytestreport.main(verbosity=2)
```
以这种方式执行之后，默认会在当前文件夹下生成一个`PyTestReport.html`日志文件，且这个文件名和样式模板都不可以重新指定的。

> 注意：这种方式执行时，如果使用Pycharm等IDE，确保不是以IDE的内建单元测试框架来执行的；或者直接通过命令行来执行。

```python
import unittest
from pytestreport import TestRunner

class MyTest(unittest.TestCase):
    def testTrue(self):
        self.assertTrue(True)

if __name__ == '__main__':
    suite = unittest.TestSuite()
    suite.addTests(unittest.TestLoader().loadTestsFromTestCase(MyTest))
    
    with open(r'/path/to/report.html', 'wb') as fp:
        runner = TestRunner(fp, title='测试标题', description='测试描述', verbosity=2)
        runner.run(suite)
```
这种方式适合批量加载和执行测试用例，从测试文件的外部来导入测试用例并执行。这里可以指定具体的结果文件路径和测试标识等信息。

> 这里使用的是默认模板主题，如果想要使用其它模板主题，可以通过制定模板的主题文件来实现。比如：使用遗留模板的方式如下所示。
```python
from pytestreport import TestRunner
...
runner = TestRunner(fp, title='测试标题', description='测试描述', verbosity=2, 
                    htmltemplate='legency.html', stylesheet='legency.css', javascript='legency.js')
```

# 附录
- [github](https://github.com/five3/PyTestReport.git)

获取更多关于Python和自动化测试的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
 