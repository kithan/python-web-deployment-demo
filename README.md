# python-web-deployment-demo

deploy python web using nginx, gunicorn, supervisord


## 步骤

### 安装nginx

根据不同机器，自行安装。	

### python依赖

	cd python-web-deployment-demo
	virtualenv venv
	source venv/bin/active
	
	pip install flask gunicorn gevent supervisor

### 编写flask的hello

### 编写wsgi.py

	# -*- coding: utf-8 -*-
	# wsgi.py
	
	from flask import Flask
	from hello.hello import app
	
	def create_app():
	  # 这个工厂方法可以从你的原有的 `__init__.py` 或者其它地方引入。
	  # app = Flask(__name__)
	  return app
	
	application = create_app()
	
	if __name__ == '__main__':
	    application.run()
	    
### 编写nginx

/etc/nginx/sites-available/hello:

	server {
	    listen 80;
	    server_name www.example.org; # 这是HOST机器的外部域名，用地址也行
	
	     # 动态请求转发到8000端口:
	    location / {
	        proxy_pass http://127.0.0.1:8000; # 这里是指向 gunicorn host 的服务地址
	        proxy_set_header Host $host;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	    }
	
	}	

设置软链：

	sudo rm /etc/nginx/sites-enabled/default
	sudo ln -s /etc/nginx/sites-available/hello /etc/nginx/sites-enabled/hello

### 配置gunicorn

这里不编写gunicorn配置文件，直接在命令中指定配置：

	gunicorn --worker-class=gevent -w 4 -b 127.0.0.1:8000 wsgi:application	 > /dev/null 2>&1 &
	
### 配置supervisor

/etc/supervisor/conf.d/hello.conf

	[program:hello]
		command = /opt/python-web-deployment-demo/venv/bin/gunicorn --worker-class=gevent -w 4 -b 127.0.0.1:8000 wsgi:application	 > /dev/null 2>&1 &
		directory = /opt/python-web-deployment-demo/   ; 执行 command 之前，先切换到工作目录
		user = hello
		
### 运行

	supervisorctl start hello
		
	
		


	    
	 
	
	