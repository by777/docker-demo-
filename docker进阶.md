# Docker Compose
```bash
# 清理docker环境
docker rmi -f $(docker images -aq)
docker rm -f $(docker ps -aq)
```
## 什么是docker compose
[docker compose](https://docs.docker.com/compose/)

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