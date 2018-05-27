---
layout: post
title: Tornado IOStream学习
date: 2018-05-27
category: "Tornado"
tags: [Web Server,Tornado]
author: Lambda
comment: false
---

# Tornado iostream

tornado.iostream中提供了一些工具类用来读写非阻塞的文件和套接字, 其本质就是对文件或套接字进行了一层封装, 借由Ioloop的事件驱动模型来进行读写,

并且其异步过程是通过回调函数来实现的.


## 测试代码

{% highlight python linenos %}
import socket

import tornado.iostream
import tornado.ioloop


def send_request():
    stream.write(b"GET / HTTP/1.0\r\nHost: www.baidu.com\r\n\r\n")
    stream.read_until(b"\r\n\r\n", on_headers)

def on_headers(data):
    headers = {}
    for line in data.split(b"\r\n"):
        parts = line.split(b":")
        if len(parts) == 2:
            headers[parts[0].strip()] = parts[1].strip()
    print(headers)
    stream.read_bytes(int(headers[b"Content-Length"]), on_body)

def on_body(data):
    print(data[:100])
    stream.close()
    tornado.ioloop.IOLoop.current().stop()


if __name__ == "__main__":
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    stream = tornado.iostream.IOStream(s)
    stream.connect(("www.baidu.com", 80), send_request)
    tornado.ioloop.IOLoop.current().start()
{% endhighlight %}


## 代码输出

上述代码用于获取百度的首页, 由于HTML内容过长, 这里只打印了前100个字节

    {b'X-Ua-Compatible': b'IE=Edge,chrome=1', b'Vary': b'Accept-Encoding', b'Cache-Control': b'no-cache', b'P3p': b'CP=" OTI DSP COR IVA OUR IND COM "', b'Content-Length': b'14615', b'Content-Type': b'text/html', b'Pragma': b'no-cache', b'Accept-Ranges': b'bytes', b'Server': b'BWS/1.1'}
    b'<!DOCTYPE html><!--STATUS OK-->\r\n<html>\r\n<head>\r\n\t<meta http-equiv="content-type" content="text/html'