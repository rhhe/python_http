# python_http
Http server and post, get code from Internet
## reference
https://blog.csdn.net/kakakaco/article/details/94929396


# 如何python搭建简单的服务，并实现post、get功能
#### 问题背景
两台机器，其中一台机器A需要计算，并且时刻把结果传输到另一台机器B上。

最simple的方式，机器B搭建一个系列公共目录，其中一个设置A有权限读写。A挂在B的公共目录，把计算结果写进去。B监控公共目录，当有新的内容写入时，读取文件，当然，此处要考虑锁的问题。这里，反过来把公共目录放在B上也是一样的。这种场景下如果一台机器是段侧机器，一台是服务器，那么，在段侧去存放这种目录，需要记得设置文件存储时间、大小，最好写入信息的时候，就查看总的文件大小，免得磁盘写满。

正常情况下，在A或B上假设一个服务，另一台机器去访问就好了。如果是A做计算，相当于A先得到消息，B后得到消息。那么，B上架一个服务，用A去访问它是比较合适的。如果条件允许，在B上同时搞一个数据库存储结果和后续状态，我觉得也是不错的。

这里，只讲讲怎么在B上搭建服务，在A上创建请求。
#### 工具
1. python IDE，此处选择了pycharm
2. postman，下载地址：https://www.getpostman.com/downloads/
#### 参考
1. https://www.jianshu.com/p/279473392f38 【主要参考这个】
2. https://blog.csdn.net/huaxiawudi/article/details/81612831 【简单看看作为理解，实际用还有很多需要补充】

#### 搭建服务
服务就是，运行一个程序，等待发信任给我发数据，我读到这些数据，做出处理，处理完成后，回复一些数据。
在上面的场景中，就是A计算将结果发给B，B得到消息后做处理，处理之后给A回复一条消息，A看到消息，就知道给B的信B已经收到了。
github上的大神分享了自己的code，```参考1```中有解读。实际操作过程如下。
讲下面代码运行，将架起一个服务。
* 搭建服务 
```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import logging


class S(BaseHTTPRequestHandler):
    def do_HEAD(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()

    def do_GET(self):
        paths = {
            '/foo': {'status': 200},
            '/bar': {'status': 302},
            '/baz': {'status': 404},
            '/qux': {'status': 500}
        }

        if self.path in paths:
            self.respond(paths[self.path])
        else:
            self.respond({'status': 500})
        logging.info("GET request,\nPath: %s\nHeaders:\n%s\n", str(self.path), str(self.headers))
        self.wfile.write("GET request for {}".format(self.path).encode('utf-8'))

    def do_POST(self):
        content_length = int(self.headers['Content-Length']) # <--- Gets the size of data
        post_data = self.rfile.read(content_length) # <--- Gets the data itself

        logging.info("POST request,\nPath: %s\nHeaders:\n%s\n\nBody:\n%s\n",
                str(self.path), str(self.headers), post_data.decode('utf-8'))

        res = "You Input: " + post_data.decode('utf-8')

        self.do_HEAD()
        # self.wfile.write("POST request for {}".format(self.path).encode('utf-8'))
        self.wfile.write("{}".format(res).encode('utf-8'))
        # self.wfile.write("POST request for {ASS}".format(data).encode('utf-8'))

    def respond(self, opts):
        response = self.handle_http(opts['status'], self.path)
        self.wfile.write(response)

    def handle_http(self, status_code, path):
        self.send_response(status_code)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        content = '''
           <html><head><title>Title goes here.</title></head>
           <body><p>This is a test.</p>
           <p>You accessed path: {}</p>
           </body></html>
           '''.format(path)
        return bytes(content, 'UTF-8')


def run(server_class=HTTPServer, handler_class=S, port=8080):
    print("run()")
    logging.basicConfig(level=logging.INFO)
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    logging.info('Starting httpd...\n')
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        pass
    httpd.server_close()
    print("httpd.server_close()")
    logging.info('Stopping httpd...\n')


if __name__ == '__main__':
    from sys import argv

    if len(argv) == 2:
        run(port=int(argv[1]))
    else:
        run()
```
当运行这段程序后，你可以利用postman创建一个post查看这个服务的效果。
**记得**要运行此程序，不要退出。服务就是一个一直在**运行**的程序。
#### 查看服务的结果
**首先**看看你的ip地址，windows，cmd，ipconfig，得到我的本机地址是：192.168.1.108。
**然后**在postman中根据代码中写入的端口号8080，填写被访问的地址```http://192.168.1.108:8080/TestServer```，如果你不写后面的后缀，也是可以的，```http://192.168.1.108:8080```，可以试试会看到什么。这个url后缀，会在上面的服务中被解析，相当于处理服务的不同情况。
截图如下，![在这里插入图片描述](https://img-blog.csdnimg.cn/20190707031931158.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tha2FrYWNv,size_16,color_FFFFFF,t_70)

#### python实现post
在上面的场景中，如果B机器假设了服务，那么A机器该如何利用python实现postman这样一个动作呢？postman和K神已经帮我们解决了这个问题。操作如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190707032825728.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tha2FrYWNv,size_16,color_FFFFFF,t_70)
那么，其中的code如下
```python
import requests

url = "http://192.168.1.108:8080/TestServer"

payload = "abcdddddEEEE"
headers = {
    'Content-Type': "text/plain",
    'User-Agent': "PostmanRuntime/7.15.0",
    'Accept': "*/*",
    'Cache-Control': "no-cache",
    'Postman-Token': "d5639969-8fbd-4531-9ad5-9a1bbe99089a,0c1fee63-a3e2-4a81-89ea-8b8861a0aa61",
    'Host': "192.168.1.108:8080",
    'accept-encoding': "gzip, deflate",
    'content-length': "12",
    'Connection': "keep-alive",
    'cache-control': "no-cache"
    }

response = requests.request("POST", url, data=payload, headers=headers)

print(response.text)
```
运行这段代码就实现了一个post功能。


或者，你用下面的这段代码也是可以的。都来自postman。
```python
import http.client

conn = http.client.HTTPConnection("192,168,1,108")

payload = "abcdddddEEEE"

headers = {
    'Content-Type': "text/plain",
    'User-Agent': "PostmanRuntime/7.15.0",
    'Accept': "*/*",
    'Cache-Control': "no-cache",
    'Postman-Token': "b695333f-e7e4-4926-8824-aea76482402b,48e619a9-f5a1-4898-9184-57cde4d72d89",
    'Host': "192.168.1.108:8080",
    'accept-encoding': "gzip, deflate",
    'content-length': "24",
    'Connection': "keep-alive",
    'cache-control': "no-cache"
    }

conn.request("POST", "TestServer", payload, headers)

res = conn.getresponse()
data = res.read()

print(data.decode("utf-8"))
```
#### 总结
按照以上的过程，你应该可以搞定一个python搭建服务，python实现post的基本功能。web服务和访问有很多内容，此处只是应急的自己玩玩的小功能。
