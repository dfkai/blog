# docker 部署 django+uwsgi+nginx+supervisor

## 1.安装docker

[官网安装教程](https://docs.docker.com/install/linux/docker-ce/centos/)

## 2.Dockerfile 编写

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
RUN apt-get install -y nginx supervisor uwsgi-plugin-python3
RUN echo "daemon off;" >> /etc/nginx/nginx.conf

COPY ./ /var/www/
RUN pip3 install -i https://mirrors.aliyun.com/pypi/simple/  -r /var/www/requirements.txt

RUN mv /var/www/nginx_django.conf /etc/nginx/sites-enabled/default
RUN mv /var/www/supervisor-app.conf /etc/supervisor/conf.d/

WORKDIR /var/www/
EXPOSE 80
CMD ["supervisord","-n"]
```

1.使用阿里源，国内更快。

2.安装 nginx、supervisor、uwsgi-plugin-python3

3.复制项目到 /var/www/下

4.安装python第三方库

5.复制nginx.conf 和  supervisor.conf到指定目录下 

6.[daemon off; 的意义](https://segmentfault.com/a/1190000009583997)

### nginx配置

```nginx
# nginx-app.conf

# the upstream component nginx needs to connect to
upstream django {
    server unix:/var/www/uwsgi.sock; # for a file socket
    # server 127.0.0.1:8001; # for a web port socket (we'll use this first)
}

# configuration of the server
server {
    # the port your site will be served on, default_server indicates that this server block
    # is the block to use if no blocks match the server_name
    listen      80 default_server;

    # the domain name it will serve for
    #server_name localhost; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
        alias /var/www/loonflow/media;  # your Django project's media files - amend as required
    }

    location /static {
        alias /var/www/loonflow/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /etc/nginx/uwsgi_params; # the uwsgi_params file you installed
    }
}
```

### supervisor.conf

```
[supervisord]
nodaemon=true

[program:uwsgi]
command= uwsgi --ini /var/www/uwsgi_django.ini --die-on-term
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0


[program:nginx]
command=/usr/sbin/nginx
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0


[program:celery]
; 如果使用的是虚拟环境（virtualenv等）要把celery命令路径写全
command=celery -A tasks worker -l info -Q loonflow
 
; Alternatively,
;command=celery --app=your_app.celery:app worker --loglevel=INFO -n worker.%%h
; Or run a script
;command=celery.sh
;项目所在目录
directory=/var/www/loonflow  
user=root
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0


[inet_http_server]
port=0.0.0.0:9001
username=admin
password=admin123


```



### uwsgi.ini配置

```ini
[uwsgi]
# 注释不要放在 变量后面 
# python3 环境
# 使用的是 pip install uWSGI中的uwsgi 运行
socket = /var/www/uwsgi.sock
# 指定工作目录
chdir = /var/www/loonflow/
# 指定wsgi路径
wsgi-file = loonflow/wsgi.py
processes = 4
threads = 2
# 指定uwsgi使用插件吧 没有他会报错
# uwsgi -- unavailable modifier requested: 0 -- 防止报这个错
plugins=python3
# python环境指定
pythonpath = /usr/local/lib/python3.6/site-packages3
master  =true
chmod-socket    = 666
vacuum  =true

```

## 构建docker 镜像

```
# 构建
docker build -t django_web .
# 启动
docker run -p 8000:80 -p 9001:9001 --name django_app -d django_web
```



参考：

[supervisor配置说明](https://www.cnblogs.com/zhoujinyi/p/6073705.html)

[nginx配置说明](http://www.nginx.cn/76.html)

[在线生成nginx配置](https://nginxconfig.io/?0.php=false&0.python&0.django&0.root=false)

[uwsgi配置说明](https://www.cnblogs.com/zhouej/archive/2012/03/25/2379646.html#protocol)

[supervisor启动celery问题解决](https://blog.csdn.net/u010377372/article/details/77863229)

