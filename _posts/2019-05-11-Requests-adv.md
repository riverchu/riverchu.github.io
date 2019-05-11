---
layout:     post
title:      "Requests 一般用法"
subtitle:   "Python模块"
date: 2019-05-11
author:     "ChuRiver"
header-img: "img/post-bg-python.jpg"
tags:
    - Python
---

## index
- [会话对象](#会话对象)
- [请求与响应对象](#请求与响应对象)
- [准备的请求(Prepared Request)](#preparedrequest)
- [SSL 证书验证](#ssl证书验证)
- [客户端证书](#客户端证书)
- [CA证书](#ca证书)
- [响应体内容工作流](#响应体内容工作流)
- [保持活动状态（持久连接）](#持久连接)
- [流式上传](#流式上传)
- [块编码请求](#块编码请求)
- [POST 多个分块编码的文件](#post多个分块编码的文件)
- [事件挂钩](#事件挂钩)
- [自定义身份验证](#自定义身份验证)
- [流式请求](#流式请求)
- [代理](#代理)
    - [SOCKS](#socks)
- [合规性](#合规性)
    - [编码方式](#编码方式)
- [HTTP动词](#http动词)
- [定制动词](#定制动词)
- [响应头链接字段](#响应头链接字段)
- [传输适配器](#传输适配器)
    - [示例: 指定SSL版本](#示例-指定ssl版本)
- [阻塞和非阻塞](#阻塞和非阻塞)
- [Header 排序](#header-排序)
- [超时（timeout）](#超时timeout)

## 会话对象
会话对象可以跨请求保持一些参数. 同一个Session的所有请求会保持同样的cookie. 期间使用urllib3的[connection pooling](http://urllib3.readthedocs.io/en/latest/reference/index.html#module-urllib3.connectionpool)功能. 所以对于同一主机的发送的多个请求会重用底层的TCP连接, 带来性能的显著提升. (参见 [HTTP persistent connection](https://en.wikipedia.org/wiki/HTTP_persistent_connection)).

会话对象具有主要的 Requests API 的所有方法。

保持cookie:
```python
s = requests.Session()

s.get('http://httpbin.org/cookies/set/sessioncookie/123456789')
r = s.get("http://httpbin.org/cookies")

print(r.text)
# '{"cookies": {"sessioncookie": "123456789"}}'
```

会话也可用来为请求方法提供缺省数据。这是通过为会话对象的属性提供数据来实现的:
```python
s = requests.Session()
s.auth = ('user', 'pass')
s.headers.update({'x-test': 'true'})

# both 'x-test' and 'x-test2' are sent
s.get('http://httpbin.org/headers', headers={'x-test2': 'true'})
```
任何传递给请求方法的字典都会与已设置会话层数据合并。方法层的参数**覆盖**会话的参数。

不过方法级别的参数不会被跨请求保持。下面的例子只有第一个请求发送 cookie ，第二个不在方法层指定参数的请求不会携带cookie：
```python
s = requests.Session()

r = s.get('http://httpbin.org/cookies', cookies={'from-my': 'browser'})
print(r.text)
# '{"cookies": {"from-my": "browser"}}'

r = s.get('http://httpbin.org/cookies')
print(r.text)
# '{"cookies": {}}'
```

如果要手动为会话添加 cookie，使用[Cookie utility](http://cn.python-requests.org/zh_CN/latest/api.html#api-cookies)函数来操纵[Session.cookies](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Session.cookies)。

会话还可以用作前后文管理器：
```python
with requests.Session() as s:
    s.get('http://httpbin.org/cookies/set/sessioncookie/123456789')
```
这样就能确保`with`区块退出后会话能被关闭，即使发生了异常也一样。


**从字典参数中移除一个值:**  
*有时会想省略字典参数中一些会话层的键, 只需简单地在方法层参数中将那个键的值设置为None，那个键就会被自动省略掉。*

包含在一个会话中的所有数据都可以直接使用。更多细节参照[会话 API 文档](http://cn.python-requests.org/zh_CN/latest/api.html#sessionapi)。

## 请求与响应对象
任何时候进行了类似`requests.get()`的调用，实际上都在做两件主要的事情。
其一，构建一个Request对象，该对象将被发送到某个服务器请求或查询一些资源。
其二，一旦requests得到一个从服务器返回的响应就会产生一个Response对象。该响应对象包含服务器返回的所有信息，也包含原来创建的Request对象。
如下是一个简单的请求，从Wikipedia的服务器得到一些非常重要的信息：
```python
>>> r = requests.get('http://en.wikipedia.org/wiki/Monty_Python')
```

访问服务器返回的响应头部信息:
```python
>>> r.headers
{'content-length': '56170', 'x-content-type-options': 'nosniff', 'x-cache':
'HIT from cp1006.eqiad.wmnet, MISS from cp1010.eqiad.wmnet', 'content-encoding':
'gzip', 'age': '3080', 'content-language': 'en', 'vary': 'Accept-Encoding,Cookie',
'server': 'Apache', 'last-modified': 'Wed, 13 Jun 2012 01:33:50 GMT',
'connection': 'close', 'cache-control': 'private, s-maxage=0, max-age=0,
must-revalidate', 'date': 'Thu, 14 Jun 2012 12:59:39 GMT', 'content-type':
'text/html; charset=UTF-8', 'x-cache-lookup': 'HIT from cp1006.eqiad.wmnet:3128,
MISS from cp1010.eqiad.wmnet:80'}
```

查看发送到服务器的请求的头部:
```python
>>> r.request.headers
{'Accept-Encoding': 'identity, deflate, compress, gzip',
'Accept': '*/*', 'User-Agent': 'python-requests/0.13.1'}
```

## PreparedRequest
当从API或者会话调用中收到一个[Response](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response)对象时，request属性其实是使用了*PreparedRequest*, 即请求的准备状态, 包含请求的相关状态信息, 如session_id等cookie信息。有时在发送请求之前，需要对body或者header（或者别的什么东西）做一些额外处理，例：
```python
from requests import Request, Session

s = Session()
req = Request('GET', url,
    data=data,
    headers=header
)
prepped = req.prepare()

# do something with prepped.body
# do something with prepped.headers

resp = s.send(prepped,
    stream=stream,
    verify=verify,
    proxies=proxies,
    cert=cert,
    timeout=timeout
)

print(resp.status_code)
```

由于没有对Request对象做什么特殊事情，立即准备和修改了PreparedRequest对象，然后把它和别的参数一起发送到requests.*或者Session.*。

然而，上述代码会失去Requests Session对象的一些优势，尤其Session级别的状态，例如cookie就不会被应用到的请求上去。要获取一个带有状态的PreparedRequest，可以用 `Session.prepare_request()`取代`Request.prepare()`的调用，如下所示：
```python
from requests import Request, Session

s = Session()
req = Request('GET',  url,
    data=data
    headers=headers
)

prepped = s.prepare_request(req)

# do something with prepped.body
# do something with prepped.headers

resp = s.send(prepped,
    stream=stream,
    verify=verify,
    proxies=proxies,
    cert=cert,
    timeout=timeout
)

print(resp.status_code)
```

## SSL证书验证
Requests可以为HTTPS请求验证SSL证书，就像web浏览器一样。SSL验证默认是开启的，如果证书验证失败，Requests会抛出SSLError:
```python
>>> requests.get('https://requestb.in')
requests.exceptions.SSLError: hostname 'requestb.in' doesn't match either of '*.herokuapp.com', 'herokuapp.com'
```

上面的域名没有提供SSL服务，所以失败了。但Github设置了SSL:
```python
>>> requests.get('https://github.com', verify=True)
<Response [200]>
```

可以为verify传入CA_BUNDLE文件的路径，或者包含可信任CA证书文件的文件夹路径：
```python
>>> requests.get('https://github.com', verify='/path/to/certfile')
```

或者将其保持在会话中：
```python
s = requests.Session()
s.verify = '/path/to/certfile'
```

**注解:**
如果verify设为文件夹路径，文件夹必须通过OpenSSL提供的c_rehash工具处理。

还可以通过REQUESTS_CA_BUNDLE环境变量定义可信任CA列表。

如果将verify设置为 False，Requests也能忽略对SSL证书的验证。
```python
>>> requests.get('https://kennethreitz.org', verify=False)
<Response [200]>
```
默认情况下，verify是设置为 True 的。选项verify仅应用于主机证书。

*对于私有证书，也可以传递一个CA_BUNDLE文件的路径给verify。也可以设置REQUEST_CA_BUNDLE环境变量。*

## 客户端证书
也可以指定一个本地证书用作客户端证书，可以是单个文件（包含密钥和证书）或一个包含两个文件路径的元组：
```python
>>> requests.get('https://kennethreitz.org', cert=('/path/client.cert', '/path/client.key'))
<Response [200]>
```

或者保持在会话中：
```python
s = requests.Session()
s.cert = '/path/client.cert'
```

如果指定了一个错误路径或一个无效的证书:
```python
>>> requests.get('https://kennethreitz.org', cert='/wrong_path/client.pem')
SSLError: [Errno 336265225] _ssl.c:347: error:140B0009:SSL routines:SSL_CTX_use_PrivateKey_file:PEM lib
```

**警告**
本地证书的私有key必须是解密状态。目前，Requests不支持使用加密的key。

## CA证书
Requests默认附带了一套它信任的根证书，来自于[Mozilla trust store](https://hg.mozilla.org/mozilla-central/raw-file/tip/security/nss/lib/ckfw/builtins/certdata.txt)。但该证书在每次Requests更新时才会更新。这意味着如果固定使用老版本的Requests, 那么证书有可能已经太旧了。

从Requests 2.4.0版之后，如果系统中装了[certifi](http://certifi.io/)包，Requests会试图使用它里边的证书。这样用户就可以在不修改代码的情况下更新他们的可信任证书。

为了安全起见，最好经常更新certifi！

## 响应体内容工作流
默认情况下，当进行网络请求后，响应体会立即被下载。可以通过stream参数覆盖这个行为，推迟下载响应体直到访问[Response.content](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.content)属性：
```python
tarball_url = 'https://github.com/kennethreitz/requests/tarball/master'
r = requests.get(tarball_url, stream=True)
```

此时仅有**响应头**被下载下来了，连接保持打开状态，因此允许根据条件获取内容：
```python
if int(r.headers['content-length']) < TOO_LONG:
  content = r.content
  ...
```

可以进一步使用[`Response.iter_content`](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.iter_content)和[`Response.iter_lines`](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.iter_lines)方法来控制工作流，或者以[`Response.raw`](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.raw)从底层urllib3的urllib3.HTTPResponse <urllib3.response.HTTPResponse读取未解码的响应体。

如果在请求中把stream设为 True，Requests无法将连接释放回连接池，除非消耗了所有的数据，或者调用了[Response.close](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.close)。这样会带来连接效率低下的问题。如果发现在使用`stream=True`的同时还在部分读取请求的body（或者完全没有读取 body），那么就应该考虑使用`with`语句发送请求，这样可以保证请求一定会被关闭：
```python
with requests.get('http://httpbin.org/get', stream=True) as r:
    # 在此处理响应。
```

## 持久连接
归功于 urllib3，同一会话内的持久连接是完全自动处理的！同一会话内发出的任何请求都会自动复用恰当的连接！

注意：只有所有的响应体数据被读取完毕连接才会被释放为连接池；所以确保将 stream 设置为 False 或读取 Response 对象的 content 属性。

## 流式上传
Requests支持流式上传，这允许发送大的数据流或文件而无需先把它们读入内存。要使用流式上传，仅需为的请求体提供一个类文件对象即可：
```python
with open('massive-body') as f:
    requests.post('http://some.url/streamed', data=f)
```

**警告**  
强烈建议用二进制模式（[binary mode](https://docs.python.org/2/tutorial/inputoutput.html#reading-and-writing-files)）打开文件。因为 requests 可能会为提供 header 中的 Content-Length，在这种情况下该值会被设为文件的字节数。如果用文本模式打开文件，就可能碰到错误。

## 块编码请求
对于出去和进来的请求，Requests也支持分块传输编码。要发送一个块编码的请求，仅需为的请求体提供一个生成器（或任意没有具体长度的迭代器）：
```Python
def gen():
    yield 'hi'
    yield 'there'

requests.post('http://some.url/chunked', data=gen())
```

对于分块的编码请求，最好使用`Response.iter_content()`对其数据进行迭代。在理想情况下，request会设置`stream=True`，这样就可以通过调用`iter_content`并将分块大小参数设为`None`，从而进行分块的迭代。如果要设置分块的最大体积，可以把分块大小参数设为任意整数。

## POST多个分块编码的文件
可以在一个请求中发送多个文件。例如，假设要上传多个图像文件到一个HTML表单，使用一个多文件field 叫做 "images":
```html
<input type="file" name="images" multiple="true" required="true"/>
```

要实现，只要把文件设到一个元组的列表中，其中元组结构为`(form_field_name, file_info)`:
```python
>>> url = 'http://httpbin.org/post'
>>> multiple_files = [
        ('images', ('foo.png', open('foo.png', 'rb'), 'image/png')),
        ('images', ('bar.png', open('bar.png', 'rb'), 'image/png'))]
>>> r = requests.post(url, files=multiple_files)
>>> r.text
{
  ...
  'files': {'images': 'data:image/png;base64,iVBORw ....'}
  'Content-Type': 'multipart/form-data; boundary=3131623adb2043caaeb5538cc7aa0b3a',
  ...
}
```

**警告**  
强烈建议用二进制模式（[binary mode](https://docs.python.org/2/tutorial/inputoutput.html#reading-and-writing-files)）打开文件。这是因为 requests 可能会为提供 header 中的 Content-Length，在这种情况下该值会被设为文件的字节数。如果用文本模式打开文件，就可能碰到错误。

## 事件挂钩
Requests有一个钩子系统，可以用来操控部分请求过程，或信号事件处理。

可用的钩子:
`response`: 从一个请求产生的响应

可以通过传递一个`{hook_name: callback_function}`字典给`hooks`请求参数为每个请求分配一个钩子函数：
```python
hooks=dict(response=print_url)
```
`callback_function`会接受一个数据块作为它的第一个参数。
```python
def print_url(r, *args, **kwargs):
    print(r.url)
```
若执行的回调函数期间发生错误，系统会给出一个警告。

若回调函数返回一个值，默认以该值替换传进来的数据。若函数未返回任何东西，也没有什么其他的影响。

例, 在运行期间打印一些请求方法的参数：
```python
>>> requests.get('http://httpbin.org', hooks=dict(response=print_url))
http://httpbin.org
<Response [200]>
```

## 自定义身份验证
Requests 允许使用自己指定的身份验证机制。

任何传递给请求方法的`auth`参数的可调用对象，在请求发出之前都有机会修改请求。

自定义的身份验证机制是作为`requests.auth.AuthBase`的子类来实现的，也非常容易定义。Requests在`requests.auth`中提供了两种常见的的身份验证方案：`HTTPBasicAuth`和`HTTPDigestAuth`。

假设有一个web服务，仅在`X-Pizza`头被设置为一个密码值的情况下才会有响应。虽然这不太可能，但就以它为例好了。
```python
from requests.auth import AuthBase

class PizzaAuth(AuthBase):
    """Attaches HTTP Pizza Authentication to the given Request object."""
    def __init__(self, username):
        # setup any auth-related data here
        self.username = username

    def __call__(self, r):
        # modify and return the request
        r.headers['X-Pizza'] = self.username
        return r
```

然后就可以使用PizzaAuth来进行网络请求:
```python
>>> requests.get('http://pizzabin.org/admin', auth=PizzaAuth('kenneth'))
<Response [200]>
```

## 流式请求
使用[`Response.iter_lines()`](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.iter_lines) 可以很方便地对流式API（例如 [Twitter 的流式 API](https://dev.twittercom/docs/streaming-api)）进行迭代。简单地设置`stream`为`True`便可以使用[iter_lines](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.iter_lines)对相应进行迭代：
```python
import json
import requests

r = requests.get('http://httpbin.org/stream/20', stream=True)

for line in r.iter_lines():

    # filter out keep-alive new lines
    if line:
        decoded_line = line.decode('utf-8')
        print(json.loads(decoded_line))
```

当使用*decode_unicode=True*在[`Response.iter_lines()`](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.iter_lines) 或 [`Response.iter_content()`](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.iter_content)中时，需要提供一个回退编码方式，以防服务器没有提供默认回退编码，从而导致错误：
```python
r = requests.get('http://httpbin.org/stream/20', stream=True)

if r.encoding is None:
    r.encoding = 'utf-8'

for line in r.iter_lines(decode_unicode=True):
    if line:
        print(json.loads(line))
```

**警告**
[iter_lines](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.iter_lines) 不保证重进入时的安全性。多次调用该方法 会导致部分收到的数据丢失。如果要在多处调用它，就应该使用生成的迭代器对象:
```python
lines = r.iter_lines()
# 保存第一行以供后面使用，或者直接跳过

first_line = next(lines)

for line in lines:
    print(line)
```

## 代理
如果需要使用代理，可以通过为任意请求方法提供`proxies`参数来配置单个请求:
```python
import requests

proxies = {
  "http": "http://10.10.1.10:3128",
  "https": "http://10.10.1.10:1080",
}

requests.get("http://example.org", proxies=proxies)
```

也可以通过环境变量 HTTP_PROXY 和 HTTPS_PROXY 来配置代理。
```Python
$ export HTTP_PROXY="http://10.10.1.10:3128"
$ export HTTPS_PROXY="http://10.10.1.10:1080"

$ python
>>> import requests
>>> requests.get("http://example.org")
```

若的代理需要使用HTTP Basic Auth，可以使用*http://user:password@host/*语法：
```python
proxies = {
    "http": "http://user:pass@10.10.1.10:3128/",
}
```

要为某个特定的连接方式或者主机设置代理，使用*scheme://hostname*作为key，它会针对指定的主机和连接方式进行匹配。
```python
proxies = {'http://10.20.1.128': 'http://10.10.1.10:5323'}
```

注意，代理 URL 必须包含连接方式。

### SOCKS
2.10.0 新版功能.

除了基本的HTTP代理，Request还支持SOCKS协议的代理。这是一个可选功能，若要使用，需要安装第三方库。

可以用pip获取依赖:
```bash
$ pip install requests[socks]
```

安装好依赖以后，使用SOCKS代理和使用HTTP代理一样简单：
```python
proxies = {
    'http': 'socks5://user:pass@host:port',
    'https': 'socks5://user:pass@host:port'
}
```

## 合规性
Requests符合所有相关的规范和RFC，这样不会为用户造成不必要的困难。但这种对规范的考虑导致一些行为对于不熟悉相关规范的人来说看似有点奇怪。

### 编码方式
当收到一个响应时，Requests会猜测响应的编码方式，用于在调用[Response.text](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.text)方法时对响应进行解码。Requests首先在HTTP头部检测是否存在指定的编码方式，如果不存在，则会使用[charade](http://pypi.python.org/pypi/charade)来尝试猜测编码方式。

只有当HTTP头部不存在明确指定的字符集，并且`Content-Type`头部字段包含`text`值之时，Requests才不去猜测编码方式。在这种情况下，[RFC 2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.7.1) 指定默认字符集必须是`ISO-8859-1`。Requests遵从这一规范。如果需要一种不同的编码方式，可以手动设置[Response.encoding](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.encoding)属性，或使用原始的[Response.content](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.content)。

## HTTP动词
Requests提供了几乎所有HTTP动词的功能：GET、OPTIONS、HEAD、POST、PUT、PATCH、DELETE。以下内容为使用Requests中的这些动词以及Github API提供了详细示例。

将从最常使用的动词GET开始。HTTP GET是一个幂等方法，从给定的URL返回一个资源。因而，当试图从一个web位置获取数据之时，应该使用这个动词。一个使用示例是尝试从Github上获取关于一个特定commit的信息。假设想获取Requests的commit a050faf的信息。可以这样去做：
```python
>>> import requests
>>> r = requests.get('https://api.github.com/repos/requests/requests/git/commits/a050faf084662f3a352dd1a941f2c7c9f886d4ad')
```

应该确认GitHub是否正确响应。如果正确响应，想弄清响应内容是什么类型的。
```python
>>> if (r.status_code == requests.codes.ok):
...     print r.headers['content-type']
...
application/json; charset=utf-8
```

可见，GitHub返回了JSON数据，非常好，这样就可以使用r.json方法把这个返回的数据解析成Python对象。
```python
>>> commit_data = r.json()

>>> print commit_data.keys()
[u'committer', u'author', u'url', u'tree', u'sha', u'parents', u'message']

>>> print commit_data[u'committer']
{u'date': u'2012-05-10T11:10:50-07:00', u'email': u'me@kennethreitz.com', u'name': u'Kenneth Reitz'}

>>> print commit_data[u'message']
makin history
```

研究一下GitHub的API文档. 可以借助 Requests 的 OPTIONS 动词来查看刚使用过的 url 支持哪些 HTTP 方法。
```python
>>> verbs = requests.options(r.url)
>>> verbs.status_code
500
```

GitHub与许多 API 提供方一样，实际上并未实现 OPTIONS 方法。这是一个恼人的疏忽，但没关系，可以使用枯燥的文档。然而，如果 GitHub 正确实现了 OPTIONS，那么服务器应该在响应头中返回允许用户使用的 HTTP 方法，例如：
```python
>>> verbs = requests.options('http://a-good-website.com/api/cats')
>>> print verbs.headers['allow']
GET,HEAD,POST,OPTIONS
```

转而去查看文档，看到对于提交信息，另一个允许的方法是 POST，它会创建一个新的提交。由于正在使用 Requests 代码库，应尽可能避免对它发送笨拙的 POST。作为替代，来玩玩 GitHub 的 Issue 特性。

[Issue #482](https://github.com/requests/requests/issues/482), 鉴于该问题已经存在，就以它为例。先获取它。
```python
>>> r = requests.get('https://api.github.com/requests/kennethreitz/requests/issues/482')
>>> r.status_code
200

>>> issue = json.loads(r.text)

>>> print(issue[u'title'])
Feature any http verb in docs

>>> print(issue[u'comments'])
3
```

Cool，有 3 个评论。来看一下最后一个评论。
```python
>>> r = requests.get(r.url + u'/comments')
>>> r.status_code
200
>>> comments = r.json()
>>> print comments[0].keys()
[u'body', u'url', u'created_at', u'updated_at', u'user', u'id']
>>> print comments[2][u'body']
Probably in the "advanced" section
```

嗯，那看起来似乎是个愚蠢的评论。发表个评论来告诉这个评论者他自己的愚蠢。那么，这个评论者是谁呢？
```python
>>> print comments[2][u'user'][u'login']
kennethreitz
```

好，来告诉这个叫 Kenneth 的家伙，这个例子应该放在快速上手指南中。根据 GitHub API 文档，其方法是 POST 到该话题。来试试看。
```python
>>> body = json.dumps({u"body": u"Sounds great! I'll get right on it!"})
>>> url = u"https://api.github.com/repos/requests/requests/issues/482/comments"

>>> r = requests.post(url=url, data=body)
>>> r.status_code
404
```

可能需要验证身份。那就有点纠结了。Requests 简化了多种身份验证形式的使用，包括非常常见的Basic Auth。
```python
>>> from requests.auth import HTTPBasicAuth
>>> auth = HTTPBasicAuth('fake@example.com', 'not_a_real_password')

>>> r = requests.post(url=url, data=body, auth=auth)
>>> r.status_code
201

>>> content = r.json()
>>> print(content[u'body'])
Sounds great! I'll get right on it.
```

如果能够编辑这条评论那就好了！幸运的是，GitHub 允许使用另一个 HTTP 动词 PATCH 来编辑评论。来试试。
```python
>>> print(content[u"id"])
5804413

>>> body = json.dumps({u"body": u"Sounds great! I'll get right on it once I feed my cat."})
>>> url = u"https://api.github.com/repos/requests/requests/issues/comments/5804413"

>>> r = requests.patch(url=url, data=body, auth=auth)
>>> r.status_code
200
```

GitHub 允许使用完全名副其实的 DELETE 方法来删除评论。
```python
>>> r = requests.delete(url=url, auth=auth)
>>> r.status_code
204
>>> r.headers['status']
'204 No Content'
```

很好。不见了。最后一件想知道的事情是已经使用了多少限额（ratelimit）。查查看，GitHub在响应头部发送这个信息，因此不必下载整个网页，将使用一个HEAD请求来获取响应头。
```python
>>> r = requests.head(url=url, auth=auth)
>>> print r.headers
...
'x-ratelimit-remaining': '4995'
'x-ratelimit-limit': '5000'
...
```

很好。是时候写个 Python 程序以各种刺激的方式滥用 GitHub 的 API，还可以使用 4995 次呢。

## 定制动词
有时候会碰到一些服务器，处于某些原因，它们允许或者要求用户使用上述HTTP动词之外的定制动词。比如说WEBDAV服务器会要求使用MKCOL方法。别担心，Requests一样可以搞定它们。可以使用内建的`.request`方法，例如：
```python
>>> r = requests.request('MKCOL', url, data=data)
>>> r.status_code
200 # Assuming your call was correct
```

这样就可以使用服务器要求的任意方法动词了。

## 响应头链接字段
许多HTTP API都有响应头链接字段的特性，它们使得API能够更好地自描述和自显露。

GitHub在API中为[分页](http://developer.github.com/v3/#pagination)使用这些特性，例如:
```python
>>> url = 'https://api.github.com/users/kennethreitz/repos?page=1&per_page=10'
>>> r = requests.head(url=url)
>>> r.headers['link']
'<https://api.github.com/users/kennethreitz/repos?page=2&per_page=10>; rel="next", <https://api.github.com/users/kennethreitz/repos?page=6&per_page=10>; rel="last"'
```

Requests 会自动解析这些响应头链接字段，并使得它们非常易于使用:
```python
>>> r.links["next"]
{'url': 'https://api.github.com/users/kennethreitz/repos?page=2&per_page=10', 'rel': 'next'}

>>> r.links["last"]
{'url': 'https://api.github.com/users/kennethreitz/repos?page=7&per_page=10', 'rel': 'last'}
```

## 传输适配器
从 v1.0.0 以后，Requests的内部采用了模块化设计。部分原因是为了实现传输适配器（Transport Adapter），可以看看关于它的[最早描述](http://www.kennethreitz.org/essays/the-future-of-python-http)。传输适配器提供了一个机制，让可以为HTTP服务定义交互方法。尤其是它允许应用服务前的配置。

Requests 自带了一个传输适配器，也就是[HTTPAdapter](http://cn.python-requests.org/zh_CN/latest/api.html#requests.adapters.HTTPAdapter)。 这个适配器使用了强大的 [urllib3](https://github.com/shazow/urllib3)，为 Requests 提供了默认的 HTTP 和 HTTPS 交互。每当 [Session](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Session) 被初始化，就会有适配器附着在 [Session](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Session) 上，其中一个供 HTTP 使用，另一个供 HTTPS 使用。

Request 允许用户创建和使用他们自己的传输适配器，实现他们需要的特殊功能。创建好以后，传输适配器可以被加载到一个会话对象上，附带着一个说明，告诉会话适配器应该应用在哪个 web 服务上。
```python
>>> s = requests.Session()
>>> s.mount('http://www.github.com', MyAdapter())
```

这个 mount 调用会注册一个传输适配器的特定实例到一个前缀上面。加载以后，任何使用该会话的 HTTP 请求，只要其 URL 是以给定的前缀开头，该传输适配器就会被使用到。

传输适配器的众多实现细节不在本文档的覆盖范围内，不过可以看看接下来这个简单的 SSL 用例。更多的用法，也许该考虑为[BaseAdapter](http://cn.python-requests.org/zh_CN/latest/api.html#requests.adapters.BaseAdapter)创建子类。

### 示例: 指定SSL版本
Requests 开发团队刻意指定了内部库（[urllib3](https://github.com/shazow/urllib3)）的默认 SSL 版本。一般情况下这样做没有问题，不过可能会需要连接到一个服务节点，而该节点使用了和默认不同的 SSL 版本。

可以使用传输适配器解决这个问题，通过利用 HTTPAdapter 现有的大部分实现，再加上一个 ssl_version 参数并将它传递到 urllib3 中。会创建一个传输适配器，用来告诉 urllib3 让它使用 SSLv3：
```python
import ssl

from requests.adapters import HTTPAdapter
from requests.packages.urllib3.poolmanager import PoolManager


class Ssl3HttpAdapter(HTTPAdapter):
    """"Transport adapter" that allows us to use SSLv3."""

    def init_poolmanager(self, connections, maxsize, block=False):
        self.poolmanager = PoolManager(num_pools=connections,
                                       maxsize=maxsize,
                                       block=block,
                                       ssl_version=ssl.PROTOCOL_SSLv3)
```

## 阻塞和非阻塞
使用默认的传输适配器，Requests不提供任何形式的非阻塞 IO。[Response.content](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Response.content)属性会阻塞，直到整个响应下载完成。如果需要更多精细控制，该库的数据流功能（见[流式请求](#流式请求)） 允许每次接受少量的一部分响应，不过这些调用依然是阻塞式的。

如果对于阻塞式 IO 有所顾虑，还有很多项目可以供使用，它们结合了 Requests 和 Python 的某个异步框架。典型的优秀例子是[grequests](https://github.com/kennethreitz/grequests) 和 [requests-futures](https://github.com/ross/requests-futures)。

## Header 排序
在某些特殊情况下也许需要按照次序来提供header，如果向 headers 关键字参数传入一个 OrderedDict，就可以提供一个带排序的 header。然而，Requests 使用的默认 header 的次序会被优先选择，这意味着如果在 headers 关键字参数中覆盖了默认 header，和关键字参数中别的 header 相比，它们也许看上去会是次序错误的。

如果这是个问题，那么用户应该考虑在[Session](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Session) 对象上面设置默认 header，只要将 [Session](http://cn.python-requests.org/zh_CN/latest/api.html#requests.Session.headers) 设为一个定制的 OrderedDict 即可。这样就会让它成为优选的次序。

## 超时（timeout）
为防止服务器不能及时响应，大部分发至外部服务器的请求都应该带着timeout参数。在默认情况下，除非显式指定了timeout值，requests是不会自动进行超时处理的。如果没有timeout，代码可能会挂起若干分钟甚至更长时间。

*连接超时*指的是在的客户端实现到远端机器端口的连接时（对应的是`connect()`_），Request 会等待的秒数。一个很好的实践方法是把连接超时设为比 3 的倍数略大的一个数值，因为 [TCP 数据包重传窗口(TCP packet retransmission window)](http://www.hjp.at/doc/rfc/rfc2988.txt) 的默认大小是 3。

*读取超时*指的就是客户端等待服务器返回请求的时间(从客户端连接到服务器并且发送了HTTP请求开始计时)。（特定地，它指的是客户端要等待服务器发送字节之间的时间。在 99.9% 的情况下这指的是服务器发送第一个字节之前的时间）。

如果制订了一个单一的值作为 timeout，如下所示：
```python
r = requests.get('https://github.com', timeout=5)
```

这一 timeout 值将会用作 connect 和 read 二者的 timeout。如果要分别制定，就传入一个元组：
```python
r = requests.get('https://github.com', timeout=(3.05, 27))
```

如果远端服务器很慢，可以让 Request 永远等待，传入一个 None 作为 timeout 值，然后就冲咖啡去吧。
```python
r = requests.get('https://github.com', timeout=None)
```
