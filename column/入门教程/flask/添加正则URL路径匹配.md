flask默认情况下是不支持URL的正则匹配的，只会显示的支持几种简单的URL匹配方式：
- default（[^/].*?）
- string
- int
- float
- path
- uuid

而当我们对路径匹配有更高要求的时候，就无法满足我们的需要的；比如：匹配以student_开头后面跟学号的路径。此时就需要使用到正则匹配了。

flask虽然没有显示的支持URL路径的正则匹配，但骨子里还是支持的；并且提供了一个扩展URL路径匹配的接口，我们只要按照要求注册一个URL匹配类即可。

首先，我们需要定义一个正则转换类，该类必须要继承自BaseConverter基类，重点是在__init__方法中需要把args的第一个参数赋值为regex变量。
```python
from werkzeug.routing import BaseConverter

class RegexConverter(BaseConverter):
    def __init__(self, url_map, *args):
        super(RegexConverter, self).__init__(url_map)
        self.url = url_map
        self.regex = args[0]   # 正则的匹配规则

    def to_python(self, value):
        return value
```

接着，就可以开始注册已经已经定义好的URL转换类了；方法如下：
```python
from flask import Flask
from RegexConverter import RegexConverter

app = Flask(__name__)
app.url_map.converters['re'] = RegexConverter   # 注册url转换类


@app.route('/<re(r".*"):path>')         # 设置使用正则匹配url的规则
def test(path):
    return path


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```
只需2步操作就可以使用正则表达式来匹配url路径了，是不是很方便呢！

> 为什么说flask骨子里就支持正则匹配url呢，那是因为上述列出的flask默认支持的url匹配方式，其本质上就是通过正则规则来实现的。只不过提前帮我们把正则匹配规则写好了而已。

