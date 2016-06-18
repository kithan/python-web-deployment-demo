title: python应用部署
date: 2016-06-16 10:59:00
tags:
- gunicorn
- wsgi
- python
- gevent
- supervisord


# python应用部署

## 简介

python应用的部署有非常多的方式。

对于一些脚本任务，可以直接通过`后台脚本运行`：

比如直接通过shell执行: 
		
	nohup python app.py > /dev/null 2>&1  &
		
crontab添加定时任务：
	
	30 7-22/1 * * * python app.py >> /temp/log/app.log 2>&1
		
对于python的web应用的部署，一般都是: `HTTP服务器 + WSGI服务器 + supervisord + web应用:app`

* `WSGI服务器`就是实现了：WSGI(Web Server Gateway Interface)接口的服务。主要功能就是：接受HTTP请求、解析HTTP请求、发送HTTP响应等苦力活。我们的app应用就需要依赖它运行。
* `HTTP服务器`：`Apache`、`Nginx`、`Lighttpd`功能类似于`WSGI服务器`。主要功能进行网站代理。
* `supervisord`：用于监控程序的状态，程序崩溃了会进行重启策略。一般用来处理：通过`WSGI`启动的`app`意外崩溃后，进行重启
* `web应用`：python web应用app

脚本的部署很简单。下面主要介绍python web项目的部署。

## python web部署

上面提到的4个部分，其实必要的就是：`WSGI服务器` 和 `web应用`。因此部署方式非常多，比如：

* uwsgi + app
* uwsgi + app + supervisor
* uwsgin + app + supervisor + nginx
* gunicorn + app
* gunicorn + app + nginx
* gunicorn + app + nginx + supervisor
* aiohttp + app + nginx + supervisor
* ...


本文介绍这种部署方式：

![python-deploy-arch](https://raw.githubusercontent.com/zhuwei05/blog-resources/master/python-deploy-arch.png)

* Nginx：高性能Web服务器+负责反向代理
* gunicorn: 运行app应用
* Supervisor: 监控gunicorn的运行情况
* app：python web应用，如flask，django等

总结步骤：

1. 安装nginx，Gunicorn， supervisord
2. 给你的应用配置nginx
3. 使用编写app入口（文章以code.py文件为例），用Gunicorn启动code.py
4. 配置supervisord

下面分别进行介绍：



### HTTP服务器

对于python web应用来说，一般选用`nginx`作为方向代理。

#### 配置nginx

nginx的配置不做介绍了，可参考：[nginx配置详解](http://)

#### 实例

直接新建文件, 名字最好和app名字相同, 这里以`/etc/nginx/sites-available/app`举例：

简易版：

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
	
复杂版：	

	server {
		listen 80 default_server;  # 监听80端口
		listen [::]:80 default_server ipv6only=on;
	
		server_name www.example.org; # 这是HOST机器的外部域名，用地址也行
		
		access_log /var/log/nginx/app.access.log;
		error_log /var/log/nginx/app.error.log;
	
		# 动态请求转发到8000端口:
	  location / {
	                proxy_pass http://127.0.0.1:8000;  # 这里是指向 gunicorn host 的服务地址
	                proxy_set_header Host $host;
	                proxy_set_header X-Real-IP $remote_addr;
	                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        }
	
		# 处理静态资源: 比如jquery，bootstrap库等文件
		location /static {
		    root /var/www/html/app/static;
		}
	
		# 媒体地址
		location /media {
		    root /var/www/html/app/media;
		}
	
	}


注意：`proxy_pass http://127.0.0.1:8000;` 这里是指向 `gunicorn host`的服务地址。后面通过`Gunicorn`绑定的地址和这个相同。

删除默认配置，建立软链接：

	sudo rm /etc/nginx/sites-enabled/default
	sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app
	
复杂配置还要确保目录都存在：

	sudo mkdir -p /var/www/html/app/media
	sudo mkdir -p /var/www/html/app/static
	
重新加载配置或重启：

	sudo /usr/sbin/nginx -s reload
	或：
	sudo service nginx restart		

### WSGI

`Gunicorn`是一个Python WSGI UNIX的HTTP服务器。该`Gunicorn`服务器与各种Web框架兼容，我们只要简单配置执行，轻量级的资源消耗，以及相当迅速。它的特点是与各个web结合紧密，部署特别方便。 

安装：

	pip install gunicorn

有需要的话，安装`gevent`:

	pip install gevent

#### gunicorn 的用法介绍

最简单的运行方式(运行`code.py`文件的`appliction`实例)：

	gunicorn code:application
	
绑定地址：

	gunicorn -b 127.0.0.1:8000 code:application
	
gunicorn并发进程：

	gunicorn -w 8 code:application
	
gunicorn 默认使用同步阻塞的网络模型(`-k sync`)，对于大并发的访问可能表现不够好， 它还支持其它更好的模式，比如：`gevent`或`meinheld`			

	# gevent
	gunicorn -k gevent code:application
	# meinheld
	gunicorn -k egg:meinheld#gunicorn_worker code:application
	
可以将各种参数写入到配置文件: `gun.conf`。通过 `-c` 指定配置： `gunicorn -c gun.conf code:application`

	import os
	bind = '127.0.0.1:8000'
	workers = 4
	backlog = 2048
	worker_class="gevent" #sync, gevent,meinheld
	debug = True
	proc_name = 'gunicorn.proc'
	pidfile = '/tmp/gunicorn.pid'
	logfile = '/var/log/gunicorn/debug.log'
	loglevel = 'debug'	
		
gunicorn的配置详解可通过：`gunicorn -h` 或 [gunicorn](http://docs.gunicorn.org/en/latest/settings.html)


#### 实例

新建`gun.conf`:
	
	import os
	bind = '127.0.0.1:8000'
	workers = 4
	backlog = 2048
	worker_class="gevent" #sync, gevent,meinheld
	debug = True
	proc_name = 'gunicorn.proc'
	pidfile = '/tmp/gunicorn.pid'
	logfile = '/var/log/gunicorn/debug.log'
	loglevel = 'debug'	

新建`code.py`

	
	# code.py
	from flask import Flask

	def create_app():
	  # 这个工厂方法可以从你的原有的 `__init__.py` 或者其它地方引入。
	  app = Flask(__name__)
	  return app
	
	application = create_app()
	
	if __name__ == '__main__':
	    application.run()	

使用gunicorn运行程序：

	gunicorn -c gun.conf code:application
	或 后台运行：
	nohup gunicorn -c gun.conf code:application > /dev/null 2>&1 &
	
	
如果不想写配置，也可以：

	gunicorn --worker-class=gevent -w 4 -b 127.0.0.1:8000 code:application	
	或 后台运行：
	nohup gunicorn --worker-class=gevent -w 4 -b 127.0.0.1:8000 code:application	 > /dev/null 2>&1 &

> `p.s.：`如果使用virtualenv安装gunicorn，需要使用全路径的/path/to/venv/gunicorn

下面介绍通过`supervisor`来执行上面用`gunicorn`运行code:application的命令。因为：上面我们在终端和后台运行的名利如果意外退出了，就不会启动了。通过`supervisor`会帮我们重启拉起程序。


### supervisor

安装：

	pip install supervisor

#### 配置supervisor

supervisor的配置不做介绍了，可参考：[supervisord配置详解](http://supervisord.org/configuration.html)

`supervisord` 似乎默认是启动的，可以 `ps -aux | grep supervisord` 检测一下。

之后是几个常用的操作：

* `supervisord`，初始启动Supervisord，启动、管理配置中设置的进程。
* `supervisorctl stop programxxx`，停止某一个进程(programxxx)，programxxx为[program:app]里配置的值，这个示例就是app。
* `supervisorctl start programxxx`，启动某个进程
* `supervisorctl restart programxxx`，重启某个进程
* `supervisorctl stop groupworker`: ，重启所有属于名为groupworker这个分组的进程(start,restart同理)
* `supervisorctl stop all`，停止全部进程，注：start、restart、stop都不会载入最新的配置文件。
* `supervisorctl reload`，载入最新的配置文件，停止原有进程并按新的配置启动、管理所有进程。
* `supervisorctl update`，根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启。

	注意：显示用stop停止掉的进程，用reload或者update都不会自动重启。

#### 实例

在 `/etc/supervisor/conf.d` 路径下面创建 `app.conf` 文件：

简单版：

	[program:app]
	command = /path/to/app/venv/bin/gunicorn --worker-class=gevent -w 4 -b 127.0.0.1:8000 code:application	 > /dev/null 2>&1 &
	directory = /path/to/app   ; 执行 command 之前，先切换到工作目录
	user = app
	
这里面因为 `gunicorn` 是安装在 `venv` 里面的，所以需要写全路径才能够启动	

## 示例

`talk is cheap, show me the code!`

[python部署flask](https://github.com/zhuwei05/python-web-deployment-demo)


## 参考

* [廖雪峰老师的部署Web App](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014323392805925d5b69ddad514511bf0391fe2a0df2b0000)：主要是 Nginx + Supervisor，讲的原理会比较清晰
* [Flask + Gunicorn + Nginx 部署](http://www.cnblogs.com/Ray-liang/p/4837850.html): 原理和配置介绍都简单易懂
* [flask+gevent+gunicorn+nginx 初试](http://blog.csdn.net/angel22xu/article/details/25638477)：只介绍配置和步骤，快速参考
* [Flask Gunicorn Supervisor Nginx 项目部署小总结](https://gist.github.com/binderclip/f6b6f5ed4d71fa64c7c5)
* [gunicorn官方](http://gunicorn.org/)：Gunicorn
* [用gunicorn和gevent提高python web框架的性能](http://xiaorui.cc/2014/11/22/%E7%94%A8gunicorn%E5%92%8Cgevent%E6%8F%90%E9%AB%98python-web%E6%A1%86%E6%9E%B6%E7%9A%84%E6%80%A7%E8%83%BD/)：Gunicorn
* [supervisord官方配置](http://supervisord.org/configuration.html)
* [Python 进程管理工具 Supervisor 使用教程](https://www.restran.net/2015/10/04/supervisord-tutorial/)：中文supervisor配置简介
* [nginx官方配置](https://nginx.org/en/docs/http/request_processing.html)
* [Nginx配置文件详细说明](http://www.cnblogs.com/xiaogangqq123/archive/2011/03/02/1969006.html)：nginx配置中文讲解


