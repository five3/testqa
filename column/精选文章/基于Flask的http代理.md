# 基于Flask的http代理

> 说到代理，大多数情况我们都会想到通过浏览器设置的正向代理，以及类似nginx的反向代理；而实际上除此之外还有一种基于host方式实现的代理。

本文主要讲述，如何实现一个基于host方式的http代理，以及它与普通代理之间的区别。这种方式的代理主要可以应用于哪些实际的测试场景。

## 与普通代理的区别
所谓的普通代理，就是我们日常会用到的那种代理，通常需要客户端本身支持，使用时对客户端进行代理信息配置。最常见的就是对浏览器、curl等客户端配置代理，一般主要用来翻墙的！
![]()

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


app = Flask(__name__)
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
127.0.0.1 www.baidu.com
```
之后，就可以在浏览器访问`http://www.baidu.com`这个网址了，路径内容和参数随便输，它都会完整把你请求的信息给返回来。效果如下：
![]()

### 代理请求
目前来说，我们已经完成HTTP代理的一半功能了，剩下的就是如何去发送接收到的HTTP请求，之后在把请求响应内容组装好，再发回给浏览器或客户端。首先是组装要发送的请求，样例代码如下：
```python
from requests import request as sender

import urllib.parse


class METHOD:
    GET = 'GET'
    POST = 'POST'
    PUT = 'PUT'
    DELETE = 'DELETE'
    HEAD = 'HEAD'
    OPTIONS = 'OPTIONS'


class Action:
    @staticmethod
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
                if ct == 'application/x-www-form-urlencoded':
                    send_data['data'] = payload['form_data']
                elif 'multipart/form-data' in ct:
                    send_data['data'] = payload['form_data']
                    send_data['files'] = payload['files']
            elif payload['body']:
                send_data['data'] = payload['body']
            elif payload['files']:
                send_data['files'] = payload['files']

        return send_data

    @staticmethod
    def send_request(req):
        return sender(req['method'].lower(), req['url'], headers=req['headers'],
                      data=req['data'], files=req['files'])

    @staticmethod
    def warp_response_data(rep):
        return {
            'code': rep.status_code,
            'headers': dict(rep.headers),
            'body': rep.content
        }
```

### 插件机制



## 应用场景
mock

api




## 后期功能增强
- https
- websocket
- keep-alive
- chunk
- 非透明代理
