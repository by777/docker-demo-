# Docker Compose
```bash
# 清理docker环境
docker rmi -f $(docker images -aq)
docker rm -f $(docker ps -aq)

# 卸载与重新安装docker可能会解决部分问题
# https://docs.docker.com/engine/install/
```
## 什么是docker compose
[Docker-compose | 文档](https://docs.docker.com/compose/)

Compose is a tool for defining and running multi-container Docker applications. 
With Compose, you use a YAML file to configure your application’s services. 
Then, with a single command, you create and start all the services from your configuration. 

Using Compose is basically a three-step process:

1. Define your app’s environment with a Dockerfile so it can be reproduced anywhere.

2. Define the services that make up your app in docker-compose.yml so they can be run together in an isolated environment.

3. Run docker compose up and the Docker compose command starts and runs your entire app. You can alternatively run docker-compose up using the docker-compose binary.

A docker-compose.yml looks like this:
```yaml
version: "3.9"  # optional since v1.27.0
services:
    web:
    build: .
    ports:
        - "8000:5000"
    volumes:
        - .:/code
        - logvolume01:/var/log
    links:
        - redis
    redis:
    image: redis
volumes:
    logvolume01: {}
```
## 安装compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose # 速度很慢
# 或者用下面这个
curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## 测试compose
### 编写python脚本
```bash
mkdir composetest && cd composetest
vim app.py
```

```python
# app.py
import time 
import redis
from flask import Flask
app = Flask(__name__)
# dockerpose的好处，通过redis域名访问
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(.5)
@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times. \n' .format(count)
```

### 创建requirements.txt

    flask
    redis

### 创建Dockerfile
```dockerfile
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000
# 把当前目录拷过来
COPY . . 
CMD ["flask", "run"]
```

### 在compose file中定义service
创建docker-compose.yml
```yaml
version: "3.0"
services:
  web:
    build: . # 自己通过Dockerfile生成镜像
    ports:
      - "8000:5000"
  redis:
    image: "redis:alpine" # 使用官方的redis镜像

```
### 运行docker-compose
```bash
docker-compose build
docker-compose up
```
流程观察
1. 创建网络
2. 执行docker-compose.yml
3. 启动所有的服务
   
访问网页
```bash
curl localhost:5000
```

### 补充
如果查看docker-images会看到docker-compose里面的东西被自动下载了

默认的服务名：文件名_服务名_num
num代表副本数量，
未来在集群上，服务不可能只有一个运行实例

### 观察网络规则
```shell
docker network ls
# compose_default
# 项目中的内容都在上面同一个网络下，可以域名访问
```
### 停止

```bash
# ctrl + c OR
docker-compose down # 在yaml目录下
```
# YAML规则
[Compose-yaml | 文档](https://docs.docker.com/compose/compose-file/#build)

+ version：和docker引擎对应
+ services：服务
  + 服务1：web
    + images
    + build
    + network
  + 服务2：
    + ...

+ 其它配置 网络networks、卷volumes、全局规则configs

deponds_on：项目启动有顺序的：redis->web

# wordpress
docker-compose.yml

```yaml

version: '3.0'
service:
    db:
        image: mysql:5.7
        volumes:
            - db_data: /var/lib/mysql
        restart: always
        environment:
            MYSQL_ROOT_PASSWORD: somewordpress
            MYSQL_DATABASE: wordpress
            MYSQL_USER: wordpress
            MYSQL_PASSWORD: wordpress
    wordpress:
        depends_on:
            - db
        image: wordpress:latest
        volumes:
            - wordpress_data: /var/www/html
        ports:
            - "8000:80"
        restart: always
        environment:
            WORDPRESS_DB_HOST: db
            WORDPRESS_DB_USER: wordpress
            WORDPRESS_DB_PASSWORD: wordpress
            WORDPRESS_DB_NAME: wordpress
    volumes:
        db_data: {}


```

```bash
# 后台启动
# docker-compose build
docker-compose up -d
# 假设项目要重新部署可
docker-compose up --build
```
# swarm
    Docker 的集群管理工具。它将 Docker 主机池转变为单个虚拟 Docker 主机。 Docker Swarm 提供了标准的 Docker API，所有任何已经与 Docker 守护程序通信的工具都可以使用 Swarm 轻松地扩展到多个主机。

# docker stack
```bash
# 单机
docker-compose up -d wordpress.yml
# 集群
docker stack deploy wordpress.yml

```
# docker secert
    配置密码，证书
# docker config
```shell
docker config --help
```

# 云原生