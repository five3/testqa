> 近期由于有项目需要做性能评测，于是半道出家的我便从选择性能测试工具，开始了我的性能之旅。

## 为什么要做工具评测
作为性能测试的老司机们而言，要么对各大性能测试工具的特性都了然于心了，要么已经使用“惯”了手头上的工具；他们是不会没事做个性能评测的，只有新手们才会认认真真的、按部就班的从第一步走起。

而对于性能测试而言，首要的任务自然是选择工具了。所以就有了性能测试工具评测这一趴！

## 为什么要解析Locust源码
由于Python是我的主语言，所以在选择性能工具评测的时候，自然是会多“关照”下Locust了。因为对评测的结果不是很满意，所以就乘着兴致顺便看了下源码。而本文就是对Locust源码解析的简述。

## Locust的执行流程
首先，来看下Locust的执行命令如下：
```bash
locust -f locustfile.py --host http://www.testqa.cn
```
那么执行了这一条语句后，Locust究竟在后台做了哪些事情呢？请看下面的程序执行流程。
```text
|--Python进程
    |--主线程
        |--参数解析（-f、--host等）
        |--性能场景脚本（-f参数后的文件名）加载、分析VUser数量
        |--协程1（local、master、slave）
            |--计算各VUser的并发数占比（按VUser的权重）
            |--生成VUser启动列表
            |--启动VUser协程组
            |        |--子协程1（对应一个VUser）
            |        |--...
            |        |--子协程N
            |               |--获取VUser任务集
            |               |--循环执行任务（顺序、按权重）
            |               |       |--嵌套执行子任务
            |               |--执行指定时间后停止（需设定）
            |
            |--协程组阻塞等待
```
接下来，我们一个个的来过一下。首先启动进程和主线程这个不用讲，所有的程序都是一样的。然后再是参数解析，这个也是大多数程序都会提供的常规逻辑。

在解析-f参数成功之后（没有指定-f参数则不会启动成功），会去自动的导入该脚本模块；再通过python的自省能力来检查脚本中的VUser类，主要检查继承自Locust且带有task_set属性的子类；一个子类相当于一个VUser。通过-l参数则可以直接列出脚本中所有的VUser名称且不会执行脚本。

当VUser类都检查完毕之后，会把这些VUser类收集到一个列表中去；之后就会根据指定的启动模式（local、no-web、master、slave）来启动一个协程，并且会把VUser列表和解析后的命令行参数内容都作为参数传递过去。

在该协程中会先计算各VUser的权重，这会影响VUser被执行的次数。具体的实现代码如下：
```python
    def weight_locusts(self, amount, stop_timeout = None):
        """
        Distributes the amount of locusts for each WebLocust-class according to it's weight
        returns a list "bucket" with the weighted locusts
        """
        bucket = []
        weight_sum = sum((locust.weight for locust in self.locust_classes if locust.task_set))
        for locust in self.locust_classes:
            ... # 部分代码省略
            # create locusts depending on weight
            percent = locust.weight / float(weight_sum)
            num_locusts = int(round(amount * percent))
            bucket.extend([locust for x in xrange(0, num_locusts)])
        return bucket
```
代码的主要实现逻辑是，先把所有的权重数都加起来求总和，再计算每个VUser的权重百分比；接着用总的VUser数乘以这个百分比后取整，就得到了该VUser需要启动数量，最后把指定数量的VUser都填充到队列中再返回。举个例子：
```text
VUser1.weight = 1
VUser2.weight = 2
VUser3.weight = 3
假设现在需要启动12个VUser，那么各VUser的数量是：
VUser1  => 1/(1+2+3)*12=2
VUser2  => 2/(1+2+3)*12=4
VUser3  => 3/(1+2+3)*12=6
经过这个函数处理之后，会得到一个如下的列表：
[VUser1, VUser1, VUser2, VUser2, VUser2, VUser2, VUser3, VUser3, VUser3, VUser3, VUser3, VUser3]
```
拿到这个VUser启动列表之后，会依次随机pop一个VUser类，然后新起一个协程来实例化它，实例之后调用它的run方法开始执行该VUser的任务内容，直到所有VUser都实例化完成。
```python
from gevent.pool import Group
...
self.locusts = Group()
...
locust = bucket.pop(random.randint(0, len(bucket)-1))
occurence_count[locust.__name__] += 1
def start_locust(_):
    try:
        locust().run(runner=self)
    except GreenletExit:
        pass
new_locust = self.locusts.spawn(start_locust, locust)
...
if wait:
    self.locusts.join()
```
实例化VUser的协程会在一个协程组内，该协程组会根据外部参数确定是否阻塞主线程。

## VUser的执行流程
上面介绍了Locust从启动后，开始执行性能测试的整体流程。而在这个整体流程内其实还包含另外一个子流程，就是VUser执行任务的流程。在介绍具体的流程之前，可以先看下Locust的脚本文件样例：
```python
from locust import HttpLocust, TaskSet, task

class WebsiteTasks(TaskSet):
    @task
    def index(self):
        self.client.get("/")

class WebsiteUser(HttpLocust):
    task_set = WebsiteTasks
    host = "https://www.baidu.com"
    min_wait = 5000
    max_wait = 15000
```
这是一个最简答的Locust的性能脚本文件，其中WebsiteUser就代表了VUser，它具备成为VUser的2个充要条件：Locust的子类、具有task_set属性且为真。（HttpLocust是Locust的子类）

task_set就是该VUser要执行的请求任务集合，这个集合里面可以有1或N个任务，还可以包含子任务集；子任务集还可以包含任务和子子任务集，所以任务集是可以嵌套的。

而VUser在实例化之后，通过调用run方法就会开始执行真正的请求任务。整体的示意流程如下：
```text
|--VUser
    |--思考时间(默认1秒)
    |--host
    |--client（可自定义）
    |--任务集
        |--子任务集
        |--普通任务
            |--request（requests.session）
            |   |--get
                |--post
            |--check
            |--result（response.success）
```
首先会存储相关的执行属性，比如：思考时间，host等。这里需要注意的是，Locust默认会把思考时间设置为1秒，所以如果你不期望有思考时间，那么你最好显式的把min_wait和max_wait都设置为0。

与此同时还会实例化真正的请求客户端，以便于在后面直接可以用来发送请求，而默认Locust发送请求的客户端其实就是requests。具体源码如下：
```python
import requests
...
class HttpSession(requests.Session):
...
class HttpLocust(Locust):
    client = None
    def __init__(self):
        super(HttpLocust, self).__init__()
        if self.host is None:
            raise LocustError("You must specify the base host. Either in the host attribute in the Locust class, or on the command line using the --host option.")
        
        self.client = HttpSession(base_url=self.host)   # 实例化发送请求的client
```
这些准备工作都完成之后，就开始实例化task_set变量对应的类，也就是样例代码中的WebsiteTasks类（TaskSet的子类）；接着会调用TaskSet实例的run方法来执行所有的任务。

在TaskSet.run方法内，会先检查是否有on_start方法，如果有会执行它；然后会进入一个while死循环，循环内每次会获取一个要执行的任务并执行完成，直到执行时间结束或者主动中断。

在获取执行任务的逻辑中会分2种情况：一种是随机，另一种是按顺序。这主要取决于你在标注任务方法时，使用的是@task装饰器，还是@seq_task装饰器。除此之外，task也是有权重的概念，通常权重越高的task被执行的概率就越高。
```python
...
    for item in six.itervalues(classDict):
        if hasattr(item, "locust_task_weight"):
            for i in xrange(0, item.locust_task_weight):
                new_tasks.append(item)
    classDict["tasks"] = new_tasks  
...
def get_next_task(self):
    return random.choice(self.tasks)
```
这个是随机获取任务的实现片段，首先在生成tasks列表的时候，会根据任务的locust_task_weight属性值来添加同等数量的任务；之后在获取任务的时候，直接使用随机函数从tasks列表中获取即可。因为权重越高在tasks列表中出现的次数就越多，所以被随机选到的概率就越高。

```python
class TaskSequence(TaskSet):
    def __init__(self, parent):
        super(TaskSequence, self).__init__(parent)
        self._index = 0
        self.tasks.sort(key=lambda t: t.locust_task_order if hasattr(t, 'locust_task_order') else 1)

    def get_next_task(self):
        task = self.tasks[self._index]
        self._index = (self._index + 1) % len(self.tasks)
        return task
```
这个是顺序获取任务的实现，首先在实例化时就根据locust_task_order属性对任务进行排序，之后获取任务的get_next_task方法内会按照索引依次获取任务，并且支持无限循环的获取方式。

最后则是执行具体任务的逻辑，任务分3种：TaskSet实例的成员方法；子任务集；普通函数。根据任务类型的不同，会执行相应的调用操作：
- 如果是TaskSet成员方法，会直接调用
- 如果是子任务集，会递归调用子任务集的run方法
- 如果是普通函数，会直接调用并把Locust实例作为第一参数

```python
    def execute_task(self, task, *args, **kwargs):
        # check if the function is a method bound to the current locust, and if so, don't pass self as first argument
        if hasattr(task, "__self__") and task.__self__ == self:
            # task is a bound method on self
            task(*args, **kwargs)
        elif hasattr(task, "tasks") and issubclass(task, TaskSet):
            # task is another (nested) TaskSet class
            task(self).run(*args, **kwargs)
        else:
            # task is a function
            task(self, *args, **kwargs)
```
> 需要注意的是：TaskSet中的子任务集是通过tasks成员变量来获取的，不同于VUser中使用task_set成员变量。

## 小结
分析到这里其实会发现Locust的逻辑还是蛮清晰的，这些主要逻辑只包含在2个文件中。而通过源码分析也解答了我的一个疑惑，就是虽然各VUser之间是并发执行的，但是VUser内的请求确实顺序执行的。而这与浏览器行为是有所差异的，现代浏览器通常可以支持同时6-8个并发请求。正是因为想解开这个迷惑，所以才有查看Locust代码的想法；显然它和Jmeter是一样的，VUser内的请求是顺序的。

PS：除了源码分析，还对Locust进行了性能评测及优化实验，感觉兴趣的同学可以关注公众号，后期会分享相关文章哦！！

获取更多关于Python和自动化测试的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
 
# 新书推荐
![Python Web自动化测试设计与实现](https://img-blog.csdnimg.cn/20190117100818307.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)