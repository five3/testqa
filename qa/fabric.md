# fabric如何取消退出异常打印
```python
from fabric import Connection, Config

c = Config()
c.run['warn'] = True

shell = Connection(host='host', port='port', user='username',
                   connect_kwargs={'password': 'password'},config=c)
```
