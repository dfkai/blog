# python 使用rpc服务

## 1.安装

```
# python3
pip3 install futures==3.1.1 
pip3 install grpcio protobuf grpcio-tools
# python2
pip install grpcio protobuf grpcio-tools
```

## 2.目录结构

```
.
+-- client
|   +-- __init__.py
|   +-- main.py
+-- proto
|   data.proto
+-- proto_py
|   +-- __init_.py
|   +-- data_pd2.py
|   +-- data_pb2_grpc.py
+-- server
|   +-- __init_.py
|   +-- main.py
```

1. client目录下的 main.py 实现了客户端用于发送数据并打印接收到 server 端处理后的数据
2. server 目录下的 main.py 实现了 server 端用于接收客户端发送的数据，并对数据进行大写处理后返回给客户端
3. proto 包用于编写 proto 文件并生成 data 接口
4. proto_py 存储proto生成的py文件

## 3.数据

- 定义 gRPC 接口：

```protobuf
syntax = "proto3";
package example;
service FormatData { //定义服务,用在rpc传输中
  rpc DoFormat(Data) returns (Data){}
}
message Data {
  string text = 1;
}
```

- 编译 protobuf：

> $ python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. ./data.proto   #在 proto 目录中执行编译，会生成：data_pb2.py 与 data_pb2_grpc.py

- 实现 server 端：

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-
# 实现了 server 端用于接收客户端发送的数据，并对数据进行大写处理后返回给客户端
import grpc
import time
from concurrent import futures
from proto_py import data_pb2, data_pb2_grpc

_ONE_DAY_IN_SECONDS = 60 * 60 * 24
_HOST = 'localhost'
_PORT = '8080'

# 实现一个派生类,重写rpc中的接口函数.自动生成的grpc文件中比proto中的服务名称多了一个Servicer
class FormatData(data_pb2_grpc.FormatDataServicer):
    # 重写接口函数.输入和输出都是proto中定义的Data类型
    def DoFormat(self, request, context):
        str = request.text
        return data_pb2.Data(text=str.upper()) # 返回一个类实例

def serve():
    # 定义服务器并设置最大连接数,corcurrent.futures是一个并发库，类似于线程池的概念
    grpcServer = grpc.server(futures.ThreadPoolExecutor(max_workers=4)) # 创建一个服务器
    data_pb2_grpc.add_FormatDataServicer_to_server(FormatData(), grpcServer) # 在服务器中添加派生的接口服务（自己实现了处理函数） 
    grpcServer.add_insecure_port(_HOST + ':' + _PORT) # 添加监听端口
    grpcServer.start() # 启动服务器
    try:
        while True:
            time.sleep(_ONE_DAY_IN_SECONDS)
    except KeyboardInterrupt:
        grpcServer.stop(0) # 关闭服务器

if __name__ == '__main__':
    serve()
```

- 实现 client 端：

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-
# 实现了客户端用于发送数据并打印接收到 server 端处理后的数据
import grpc
from proto_py import data_pb2, data_pb2_grpc

_HOST = 'localhost'
_PORT = '8080'

def run():
    conn = grpc.insecure_channel(_HOST + ':' + _PORT) # 监听频道
    client = data_pb2_grpc.FormatDataStub(channel=conn) # 客户端使用Stub类发送请求,参数为频道,为了绑定链接
    response = client.DoFormat(data_pb2.Data(text='hello,world!'))  # 返回的结果就是proto中定义的类
    print("received: " + response.text)

if __name__ == '__main__':
    run()
```

- 执行验证结果：

> 1. 先启动 server，之后再执行 client
> 2. client 侧控制台如果打印的结果为：“received: HELLO,WORLD!” ，证明 gRPC 接口定义成功

参考：

1. [Python RPC 之 gRPC](https://www.jianshu.com/p/14e6f5217f40)
2. [python使用rpc框架gRPC的方法](https://www.jb51.net/article/146229.htm)
3. [Python使用gRPC传输协议教程](https://www.jb51.net/article/148949.htm)
4. [谁能用通俗的语言解释一下什么是 RPC 框架？](https://www.zhihu.com/question/25536695/answer/36197244)
5. [tensoflow 安装出现futures requires Python '>=2.6](https://blog.csdn.net/liaoxianfu/article/details/79237046)
6. [github protobuf](https://github.com/protocolbuffers/protobuf/tree/master/python)
7. [protobuf 协议编写](https://developers.google.com/protocol-buffers/docs/overview)

