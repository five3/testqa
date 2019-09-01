# 基于host的http代理--hproxy

![banner](https://img-blog.csdnimg.cn/20190829223407940.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

> 说到代理，大多数情况我们都会想到通过浏览器设置的正向代理，以及类似nginx的反向代理；而实际上除此之外还有一种基于host方式实现的代理。

本文主要讲述，如何实现一个基于host方式的http代理，以及它与普通代理之间的区别。这种方式的代理主要可以应用于哪些实际的测试场景。

## 与普通代理的区别
所谓的普通代理，就是我们日常会用到的那种代理，通常需要客户端本身支持，使用时对客户端进行代理信息配置。最常见的就是对浏览器、curl等客户端配置代理，一般主要用来翻墙的！

![001](https://img-blog.csdnimg.cn/20190829223426320.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

而host代理则不是主流的代理方式，它的特点是通过设置host就能实现代理，而不需客户端本身支持，相对应用的访问更广一些。下面我们就来逐一对比下它们的具体区别：

| 对比项 | 普通代理 | HOST代理 |
| --- | --- | --- |
| 需要客户端支持 | 是 | 否 |
| 设置方式 | 配置客户端 | 配置HOST |
| 支持透明代理 | 是 | 是 |
| 支持绝对路径 | 是 | 否 |
| 支持非80端口 | 是 | 否 |
| 实现方式 | socket | http |
| URL路径支持 | 绝对路径 | 相对路径 |
| 代理服务与客户端同机 | 支持 | 不支持 |
| 代理配置方式 | 域名配置灵活 | host配置不灵活 | 

通过对比可以发现它们都能满足基本的HTTP代理功能，主要区别在于适用的场景有所不同；普通代理只要客户端本身支持基本上什么请求都可以代理，HOST代理只要请求是80/443端口都可以代理。

## 实现方式
### 接收请求
实现一个HOST代理是非常简单的，你只需要基于一个现成的WEB框架，比如：Flask，Tornado；再加上一个url请求框架即可，比如：requests。而首先你得实现一个可以接手任意URL路径的请求处理函数，如下：
```python
from werkzeug.routing import BaseConverter
from flask import Flask, request, jsonify


class RegexConverter(BaseConverter):
    def __init__(self, url_map, *args):
        super(RegexConverter, self).__init__(url_map)
        self.url = url_map
        self.regex = args[0]   # 正则的匹配规则

    def to_python(self, value):
        return value


app = Flask(__name__, static_url_path='/do_not_use_this_path__')
app.url_map.converters['re'] = RegexConverter


@app.route('/<re(r".*"):path>')
def proxy(path):
    url = request.base_url
    query = request.args
    method = request.method
    headers = request.headers
    form_data = request.form
    body = request.data
    files = request.files

    payload = {
        'path': f'/{path}',
        'url': url,
        'method': method,
        'headers': dict(headers),
        'query': query,
        'form_data': form_data,
        'body': body.decode('utf-8'),
        'files': files
    }

    return jsonify(payload)


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```
启动这个web服务之后，只要在host文件中配置一个指向该服务所在机器的IP和目标域名映射即可。比如：
```bash
# host文件添加一个映射
10.0.0.1 www.baidu.com
```
之后，就可以在浏览器访问`http://www.baidu.com`这个网址了，路径内容和参数随便输，它都会完整把你请求的信息给返回来，类似一个镜像服务。效果如下：

![002](https://img-blog.csdnimg.cn/20190829223442740.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

### 代理请求
目前来说，我们已经完成HTTP代理的一半功能了，剩下的就是如何去发送获取到的HTTP请求，之后在把请求响应内容组装好，再发回给浏览器或客户端。首先是组装要发送的请求，样例代码如下：
```python
class METHOD:
    GET = 'GET'
    POST = 'POST'
    PUT = 'PUT'
    DELETE = 'DELETE'
    HEAD = 'HEAD'
    OPTIONS = 'OPTIONS'
    
def warp_request_data(payload):
    """
    :param payload: {
        'path': path,
        'url': url,
        'method': method,
        'headers': dict(headers),
        'query': query,
        'form_data': form_data,
        'body': body.decode('utf-8'),
        'files': files
    }
    :return:
    """
    send_data = {'method': payload['method'], 'url': payload['url'],
                 'headers': payload['headers'], 'data': None, 'files': None}

    if payload['method'] in (METHOD.GET, METHOD.HEAD, METHOD.OPTIONS):
        send_data['data'] = payload['query']
    elif payload['method'] in (METHOD.POST, METHOD.DELETE, METHOD.PUT):
        if payload['query']:
            payload['url'] = f"{payload['url']}?{urllib.parse.urlencode(payload['query'])}"
        if payload['form_data']:
            ct = payload['headers'].get('Content-Type')
            if 'application/x-www-form-urlencoded' in ct:
                send_data['data'] = payload['form_data']
            elif 'multipart/form-data' in ct:
                send_data['data'] = payload['form_data']
                send_data['files'] = payload['files']
        elif payload['body']:
            send_data['data'] = payload['body']
        elif payload['files']:
            send_data['files'] = payload['files']

    return send_data
```
接着，通过`requests`发送组装好的请求数据内容。方法如下：
```python
from requests import request as sender

def send_request(req):
    return sender(req['method'].lower(), req['url'], headers=req['headers'],
                  data=req['data'], files=req['files'])
```
最后，把获取到的响应进行再次组装，便于主程序直接返回内容给浏览器或客户端。代码如下：
```python
def warp_response_data(rep):
    body = rep.content
    if 'Transfer-Encoding' in rep.headers:      # 不支持chunk
        del rep.headers['Transfer-Encoding']
        rep.headers['Content-Length'] = len(body)
    if 'Connection' in rep.headers:             # 不支持keep-alive
        del rep.headers['Connection']
    if 'Content-Encoding' in rep.headers:       # 不支持gzip
        del rep.headers['Content-Encoding']
    rep.headers['Server'] = 'host proxy/0.1'    # 修改服务器信息

    return {
        'code': rep.status_code,
        'headers': dict(rep.headers),
        'body': body
    }
```
同时，需要在http处理的主函数中添加对http请求的代理操作。最后的http处理主函数新增内容如下：
```python
@app.route('/<re(r".*"):path>')
def proxy(path):
    req = Action.warp_request_data(payload)
    rep = Action.send_request(req)
    ret = Action.warp_response_data(rep)

    return ret['body'], ret['code'], ret['headers']
```
### 插件机制
目前为止完成了这么多功能之后，一个http的代理就已经初步完成了。只是目前仅仅是做了代理，但是没有任何的作用。因为我们不能对它进行任何操作。要让它变得有意义就得添加插件机制，让用户可以对代理的请求进行处理和操作。

首先，定义一个插件类，用于注册和执行插件内容。代码如下：
```python
class Plugins:
    def fire(self, context):        # 插件事件触发
        for func in self.events:
            func(context)

    def register(self, func):       # 插件事件注册
        self.events.append(func)


class PRE_PROXY(Plugins):
    def __init__(self):
        self.events = []


class POST_PROXY(Plugins):
    def __init__(self):
        self.events = []


pre_proxy = PRE_PROXY()
post_proxy = POST_PROXY()


def before_proxy(func):         # 注册触发代理前装饰器
    pre_proxy.register(func)
    return func


def after_proxy(func):          # 注册触发代理后装饰器
    post_proxy.register(func)
    return func
```
同时，也要在http主函数中添加插件函数的调用，修改后的代码如下：
```python
...
    context = {}
    req = Action.warp_request_data(payload)

    context['request'] = req
    pre_proxy.fire(context)         # 触发代理前事件

    rep = Action.send_request(req)
    ret = Action.warp_response_data(rep)

    context['response'] = ret
    post_proxy.fire(context)        # 触发代理后事件
...
```
最后，在同目录下新建一个插件脚本文件`script.py`。其内容如下：
```python
from .plugins import before_proxy, after_proxy

@before_proxy
def before(context):
    print(context)

@after_proxy
def after(context):
    print(context)
```
这个脚本内容非常的简单，导入2个注册装饰器分别用来注册代理前和代理后的事件。注册函数可以接收到一个请求和响应的上下文对象参数，这里仅仅是打印了出来。

当然，插件还可以做很多其它的事情，比如：过滤特定url并保存请求信息；修改请求和响应信息内容等。完整项目代码请关注公众号并回复`hproxy`即可！

## 应用场景
这类http代理主要应用的场景一般多为测试或者开发，日常生活中翻墙还是要是普通代理。主要可以用于辅助测试，比如：mock系统，api接口测试等。

对于mock系统，可以用来录制mock内容，尤其是针对服务端请求第三方接口的请求录制。比如：
- 录制调用第三方银行的接口请求，作为mock内容
- 选择性的mock同域名下的部分URL请求，其它URL则透传

用于api自动化测试，可以直接录制对应接口的API请求，用于快速生成自动化测试用例。当然还可以用于安全测试，篡改指定的http请求内容。

## 后期功能增强
虽然目前该demo程序已经可以开始用于辅助测试工作了，但是想要更加的易用，还有很多的特性需要支持。
- 协程支持
- 过滤功能
- https支持
- websocket支持
- keep-alive支持
- chunk支持
 

获取更多关于Python和测试开发相关的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190829223353609.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)
