# grpc 之 word2pdf使用

> ​	做一个word转pdf的服务，采用grpc，使用libreoffice命令。

## 1.构建libreoffice镜像

```dockerfile
FROM python:3.6

ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN cd /etc/apt \
    && mv sources.list sources.list.bak \
    && echo "deb http://mirrors.aliyun.com/debian/ stretch main non-free contrib \
deb-src http://mirrors.aliyun.com/debian/ stretch main non-free contrib \
deb http://mirrors.aliyun.com/debian-security stretch/updates main \
deb-src http://mirrors.aliyun.com/debian-security stretch/updates main \
deb http://mirrors.aliyun.com/debian/ stretch-updates main non-free contrib \
deb-src http://mirrors.aliyun.com/debian/ stretch-updates main non-free contrib \
deb http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib \
deb-src http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib" > sources.list
RUN apt-get update
RUN apt-get install -y libreoffice
COPY ./ /root/
RUN mv /root/simsun.ttc /usr/share/fonts && mv /root/simhei.ttf /usr/share/fonts && cd /usr/share/fonts && fc-cache -fv
# docker build -t libreoffice .
```

1. 采用python3.6镜像 
2. 使用阿里源
3. 安装libreoffice
4. 解决中文乱码 加入中文字体

## 2.grpc服务端、客户端

1. 创建proto配置文件 然后编译

2. 服务端与客户端 采用二进制  数据进行传输

   

- 服务端

  ```python
  #!/usr/bin/env python
  # -*- coding: utf-8 -*-
  # @Time    : 2019/7/9 0009 16:41
  # @File    : word2pdf_server_main.py
  # @author  : dfkai
  # @Software: PyCharm
  
  # python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. ./data.proto
  
  import os
  import pathlib
  import time
  import traceback
  import uuid
  from concurrent import futures
  
  import grpc
  
  from proto_py import word2pdf_pb2, word2pdf_pb2_grpc
  
  _ONE_DAY_IN_SECONDS = 60 * 60 * 24
  _HOST = os.environ.get("HOSTNAME", "localhost")
  _PORT = '8080'
  
  
  class FormatData(word2pdf_pb2_grpc.FormatDataServicer):
      def DoFormat(self, request, context):
          """
          proto 定义方法
          :param request:
          :param context:
          :return:
          """
          data = request.text
          doc_path, pdf_path, pdf_file_path = self.get_doc_pdf_path()
          with open(doc_path, "wb") as f:
              f.write(data)
          if self.word2pdf_linux(doc_path, pdf_path):
              try:
                  with open(pdf_file_path, "rb") as f:
                      pdf_data = f.read()
              except:
                  traceback.format_exc()
          else:
              pdf_data = b"fail"
          return word2pdf_pb2.Data(text=pdf_data)
  
      def get_doc_pdf_path(self):
          """
          获取文件路径
          :return:
          """
          baseDir = os.getcwd()
          p = pathlib.Path(baseDir)
          u_name = str(uuid.uuid4()).replace("-", "")
          doc_name = u_name + ".docx"
          pdf_name = u_name + ".pdf"
          pdf_path = p / f"filepath/pdf/"
          doc_path = p / f"filepath/doc/{doc_name}"
          pdf_file_path = p / f"filepath/pdf/{pdf_name}"
          print(doc_path, pdf_path, pdf_file_path)
          return rf"{doc_path}", rf"{pdf_path}", rf"{pdf_file_path}"
  
      def word2pdf_win(self, doc_path, pdf_path):
          """
          windows 生成
          :param doc_path:
          :param pdf_path:
          :return:
          """
          from win32com import client
          import pythoncom
          pythoncom.CoInitialize()
          # word = client.Dispatch("Word.Application")
          word = client.DispatchEx("Word.Application")
          worddoc = word.Documents.Open(doc_path)
          try:
              worddoc.SaveAs(pdf_path, FileFormat=17)
          except Exception as e:
              print(e)
              return False
          finally:
              worddoc.Close()
          return True
  
      def word2pdf_linux(self, doc_path, pdf_path):
          """
          linux 生成 pdf ,利用 libreoffice 命令
          :param doc_path:
          :param pdf_path:
          :return:
          """
          try:
              os.system(f"soffice --headless --invisible --convert-to pdf {doc_path} --outdir {pdf_path} ")
          except:
              traceback.format_exc()
              return False
          return True
  
  
  def serve():
      """
      rpc 服务
      :return:
      """
      grpcServer = grpc.server(futures.ThreadPoolExecutor(max_workers=4))
      word2pdf_pb2_grpc.add_FormatDataServicer_to_server(FormatData(), grpcServer)
      grpcServer.add_insecure_port(_HOST + ':' + _PORT)
      grpcServer.start()
      try:
          while True:
              time.sleep(_ONE_DAY_IN_SECONDS)
      except KeyboardInterrupt:
          grpcServer.stop(0)
  
  
  if __name__ == '__main__':
      serve()
  
  ```

- 客户端

  ```python
  #!/usr/bin/env python
  # -*- coding: utf-8 -*-
  # @Time    : 2019/7/9 0009 16:40
  # @File    : word2pdf_client_main.py
  # @author  : dfkai
  # @Software: PyCharm
  import traceback
  import grpc
  from proto_py import word2pdf_pb2, word2pdf_pb2_grpc
  
  _HOST = 'localhost'
  _PORT = '8080'
  
  
  def run():
      file_name = "test"
      doc_name = file_name + '.doc'
      conn = grpc.insecure_channel(_HOST + ':' + _PORT)
      client = word2pdf_pb2_grpc.FormatDataStub(channel=conn)
      with open(doc_name, "rb") as f:
          data = f.read()
      response = client.DoFormat(word2pdf_pb2.Data(text=data))
      if response.text == b"fail":
          # 发送消息 生成失败
          pass
      else:
          pdf_name = file_name + f'.pdf'
          try:
              with open(pdf_name, "wb") as f:
                  f.write(response.text)
          except:
              traceback.format_exc()
              # 发送消息 生成失败
          else:
              # 发送消息 生成成功
              pass
  
  
  if __name__ == '__main__':
      import time
      beg = time.time()
      run()
      end = time.time()
      print(end - beg)
  
  ```

- proto配置文件

  ```protobuf
  syntax = "proto3";
  package example;
  service FormatData {
    rpc DoFormat(Data) returns (Data){}
  }
  message Data {
    bytes text = 1;
  }
  ```

  进入文件目录，构建命令：python -m grpc_tools.protoc -I. --python_out=./proto_py/ --grpc_python_out=./proto_py/ ./proto/word2pdf.proto

## 3.构建rpc服务端镜像

```dockerfile
FROM libreoffice

ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

COPY ./ /root/
WORKDIR /root/word2pdfRPC
RUN pip3 install -i https://mirrors.aliyun.com/pypi/simple/ -r requirments.txt
EXPOSE 8080
CMD python server_main.py
# docker build -t word2pdf .
# docker run -d -p 8080:8080 -v /root/data/word2pdf/:/root/word2pdfRPC/filepath/ --name word2pdf word2pdf

```

- reuirements.txt

  ```
  futures==3.1.1
  grpcio==1.22.0
  grpcio-tools==1.22.0
  protobuf==3.8.0
  ```

  

参考、推荐:

1. [protobuf和thrift对比](https://blog.csdn.net/xqy1522/article/details/6942344)
2. [Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)
3. [Encoding](https://developers.google.com/protocol-buffers/docs/encoding)

