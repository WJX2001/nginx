# 1Nginx简介

- 支持高并发，消耗内存资源少
- 具有多种功能

- - 网站web服务功能
  - 网站负载均衡功能
  - 正向代理反向代理

- 网站缓存功能
- 在多种系统平台都可以部署
- Nginx实现网络通讯时使用的是异步网络IO模型：**epoll**模型，查看博客https://segmentfault.com/a/1190000003063859#item-3-13

## 1.1 工作原理

- Nginx由内核一系列模块组成，内核提供web服务的基本功能：

- - 启用网络协议
  - 创建运行环境
  - 接收和分配客户端请求
  - 处理模块之间的交互

- Nginx的各种功能和操作都由模块来实现，Nginx的模块从结构上分为核心模块，基础模块和第三方模块

- - 核心模块：HTTP模块，EVENT模块和MAIL模块
  - 基础模块：HTTP Access模块， HTTP FastCGI模块、HTTP Proxy模块和HTTP Rewrite模块  
  - 第三方模块： HTTP Upstream Request Hash模块、Notice模块和HTTP Access Key模块及用 户自己开发的模块  

## 1.2 编译安装

- 从官方获取源码包

```shell
[root@localhost ~]# wget http://nginx.org/download/nginx-1.18.0.tar.gz -P /usr/local/src/
[root@localhost ~]# cd /usr/local/src
[root@localhost src]# tar xzvf nginx-1.18.0.tar.gz
[root@localhost src]# cd nginx-1.18.0
[root@localhost nginx-1.18.0]# ./configure --help
```

- 编译安装

```shell
[root@localhost nginx-1.18.0]# yum -y install gcc pcre-devel openssl-devel zlib-devel
[root@localhost nginx-1.18.0]# useradd -r -s /sbin/nologin nginx
[root@localhost nginx-1.18.0]# ./configure --prefix=/apps/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-pcre \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module
[root@localhost nginx-1.18.0]# make -j 2 && make install
[root@localhost nginx-1.18.0]# chown -R nginx.nginx /apps/nginx
[root@localhost nginx-1.18.0]# ln -s /apps/nginx/sbin/nginx /usr/bin/
[root@localhost nginx-1.18.0]# nginx -v
```

-  nginx完成安装以后，有四个主要的目录  

- - **conf:**保存nginx所有的配置文件，其中nginx.conf是nginx服务器的最核心最主要的配置文 件，其他的.conf则是用来配置nginx相关的功能的，例如fastcgi功能使用的是fastcgi. conf和 fastcgi.params两个文件，配置文件一般都有个样板配置文件，是文件名. default结尾，使用 的使用将其复制为并将default去掉即可。  
  -  **html:**目录中保存了nginx服务器的web文件，但是可以更改为其他目录保存web文件，另外 还有一个50x的web文件是默认的错误页面提示页面。  
  -  **logs:**用来保存ngi nx服务器的访问日志错误日志等日志，logs目录可以放在其他路径，比 如/var/logs/nginx里面。  
  -  **sbin:**保存nginx二进制启动脚本，可以接受不同的参数以实现不同的功能。  

```shell
[root@localhost nginx-1.18.0]# tree /apps/nginx -C -L 1 /apps/nginx
/apps/nginx
├── conf
├── html
├── logs
└── sbin
```

### 1.2.1 创建nginx自启动文件

```bash
# 复制同一版本的nginx的yum安装生成的service文件
[root@localhost ~]# vim /usr/lib/systemd/system/nginx.service
[Unit]
Description=The nginx HTTP and reverse proxy server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/apps/nginx/run/nginx.pid
ExecStart=/apps/nginx/sbin/nginx -c /apps/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target



[root@localhost ~]# mkdir /apps/nginx/run/
[root@localhost ~]# vim /apps/nginx/conf/nginx.conf
pid /apps/nginx/run/nginx.pid;
```

### 1.2.2 验证Nginx自启动文件

```shell
[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl enable --now nginx
[root@localhost ~]# ll /apps/nginx/run
总用量 4
-rw-r--r--. 1 root root 5 5月 24 10:07 nginx.pid
```

# 2 Nginx架构和进程

## 2.1 Nginx架构

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1661959289369-ec2b2222-9f22-4917-bf8a-030f63faa0c6.png)

## 2.2 Nginx 进程结构

- web处理请求机制

- - **多进程方式：**服务器每接收到一个客户端请求就有服务器的主进程生成一个子进程响应客户端，直到用户关闭连接，这样的优势是处理速度快。子进程之间相互独立，但是如果访问过大会导致服务器资源耗尽而无法提供请求
  - **多线程方式：**与多进程方式类似，但是每收到一个客户端请求会有服务进程派生出一个线程来和多个客户方进行交互，一个线程的开销远远小于一个进程，因此多线程方式在很大程度减轻了web服务器对系统资源的要求，但是多线程也有自己的缺点。即当多个线程位于同一个进程内工作的时侯，可以相互访问同样的内存地址空间，所以他们相互影响，一旦主进程挂掉则所有子线程都不能工作了，IIS服务器使用了多线程的方式，需要价格一段时间就重启一次才能稳定

- Nginx是多进程组织模型，而且是一个由Master主进程和Worker工作进程组成

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1661995798444-d5ab13f0-de15-4017-97bf-fcbc932404d1.png)

- **主进程（master process）的功能：**

- - 对外接口：接收外部的操作（信号）
  - 对内转发：根据外部的操作的不同，通过信号管理worker
  - 监控：监控worker进程的运行状态，worker进程异常终止后，自动重启worker进程
  - 读取Nginx配置文件并验证其有效性和正确性
  - 建立，绑定和关闭socket连接
  - 按照配置生成，管理和结束工作进程
  - 接受外界指令，比如重启，升级及退出服务器等指定
  - 不中断服务，实现平滑升级，重启服务并应用新的配置
  - 开启日志文件，获取文件描述符
  - 不中断服务，实现平滑升级，升级失败进行回滚处理
  - 编译和处理perl脚本

-  **工作进程worker process的功能 ：**

- - 所有Worker进程都是平等的
  - 实际处理：网络请求，由Worker进程处理
  - Worker进程数量：在nginx.conf中配置，一般设置为核心数，充分利用CPU资源，同时，避免进程数量过多，避免进程竞争CPU资源
  - 上下文切换的损耗
  - 接受处理客户的请求
  - 将请求一次送入各个功能模块进行处理
  - I/O调用，获取响应数据
  - 与后端服务器通信，接收后端服务器的处理结果
  - 缓存数据，访问缓存索引，查询和调用缓存数据
  - 发送请求结果，响应客户的请求
  - 接收主程序指令，比如重启，升级和退出等

- ![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1661997279832-7b303e43-1017-488a-a12f-9a4679c89e38.png)



![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1661997453161-00d8c10f-81af-4cda-b9e7-0836a30847d8.png)

### 2.2.1 Master工作过程细节

1. Master建立listen的socket (listenfd)
2. Master，fork出Worker进程（fork：进程镜像）
3. 新请求到来时，所有Worker进程的listenfd都变成可读；
4. Worker进程，竞争accept_mutex，获胜的注册Listenfd的读事件

### 2.2.2 Worker工作过程细节

- 新请求到来时，所有Worker进程的listenfd都变为可读
- Worker进程，竞争accept_mufex,获胜的注册listenfd的读时间
- Worker进程，在读时间中，accept当前连接
- Worker进程，读取请求，解析请求，处理请求，进行响应

# 3 Nginx核心配置详解

## 3.1 默认的nginx.conf配置文件格式说明

主配置文件存放在`etc/nginx/nginx.conf`

如果是编译安装，则主配置文件在`/apps/nginx/conf/nginx.conf`

```shell
user nginx;															# 启动进程，一般注释掉
worker_processes auto;									# 工作进程数，通常等于CPU核数或者二倍
error_log /var/log/nginx/error.log;			# 错误日志路径
pid /run/nginx.pid;											# nginx进程pid存放位置
   
# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;
  
events {
   worker_connections 1024;							# 工作进程最大连接数
}
  
http {
  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
  
  access_log  /var/log/nginx/access.log  main;
  
  sendfile            on;
  tcp_nopush          on;
  tcp_nodelay         on;
  keepalive_timeout   65;
  types_hash_max_size 4096;
  
  include             /etc/nginx/mime.types;
  default_type        application/octet-stream;
  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
  
  access_log  /var/log/nginx/access.log  main; # 访问日志输出格式及存放位置
  
  sendfile            on;	# 文件输出设置
  tcp_nopush          on;
  tcp_nodelay         on;
  keepalive_timeout   65;	# 超出时间（毫秒）
  types_hash_max_size 4096;
  
  include             /etc/nginx/mime.types;	# 指定mime类型和文档/页面/接口输入输出类型
  default_type        application/octet-stream;
  
  # Load modular configuration files from the /etc/nginx/conf.d directory.
  # See http://nginx.org/en/docs/ngx_core_module.html#include
  # for more information.
  include /etc/nginx/conf.d/*.conf; # 嵌入其他配置文件
  
server {
  listen       80;							# 默认监听80端口，可以改成其他端口
  listen       [::]:80;					
  server_name  _;								# 服务名称，域名
  root         /usr/share/nginx/html;

	location / {
    	root 		/usr/share/nginx/html # 代理的根目录
    	index 	index.html index.htm	# 代理的默认页面（根目录下）
  
  # Load configuration files for the default server block.
  include /etc/nginx/default.d/*.conf;
  
  error_page 404 /404.html; 
  location = /404.html {
  }
  
  error_page 500 502 503 504 /50x.html;		# 可以根据http状态返回错误页面
  location = /50x.html {
  }
-----------------------------------------------------------
# 编译版本
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
```

## 3.2 全局配置

### 3.2.1 CPU性能优化

```shell
user   nginx nginx												# 启动nginx工作进程的用户和组
worker_processes [number | auto]; 				# 启动Nginx工作进程的数量，一般设为和CPU核心数相同
worker_cpu_affinity 0001 0010 0100 1000; 	# 将Nginx工作进程绑定到指定的CPU核心，默认1
CPU MASK:	0001    0号CPU 
        	0010    1号CPU 
        	0100    2号CPU 
        	1000    3号CPU 
```

### 3.2.2 错误日志记录配置

```shell
error_log /var/log/nginx/error.log           
```

### 3.2.3 工作优先级与文件并发数

```shell
worker_priority 0;							# 工作进程优先级（-20-19）
worker_rlimit_nofile 65536;			# 所有worker进程能打开的文件数量上线，包括Nginx的所有连接																		（例如与代理服务器的连接等），而不仅仅是与客户端的连接
```

### 3.2.4 其他优化配置

```shell
daemon off								# 前台运行nginx服务，用于测试，docker等环境
master_process off|on;		# 是否开启Nginx的master-worker工作模式，仅用于开发调试场景
events {
	worker_connection 65536;		# 设置单个工作进程的最大并发连接数
	use epoll;									# 使用epoll事件驱动，Nginx支持众多的事件驱动，比如：select,poll,epoll,只能设置在events模块中
	accept_mutex on;			# on为同一时刻一个请求轮流由work进程处理，而防止被同时唤醒所有worker,避免多个睡眠进程被唤醒的设置，默认为off,新请求会唤醒所有worker进程，此过程被称为惊群，因此nginx刚安装完以后要进行适当的优化，建议设置为on
	multi_accept on;			# on时Nginx服务器的每个工作进程可以同时接受多个新的网络连接，此指令默认为OFF,即认为一个工作进程只能一次接受一个新的网络，打开后一个同时接受多个
}
```

## 3.3 http 配置块

http协议相关的配置结构

```shell
http {
	... 
	... #各server的公共配置
  server {		# 每个server用于定义一个虚拟主机，第一个server为默认虚拟服务器
      ...	
  }
	server {
      ...
    	server_name  # 虚拟主机
    	root				 # 主目录
    	alias				 # 路径别名
    	location {OPERATOR} URL {			# 指定URL的特性
        	...
        	if CONDITION {
                	...
          }
  	}
 	}
}
http {
    include       mime.types;   # 导入支持的文件类型，是相对于/apps/nginx/conf的目录,mime.types 多用途的邮件拓展，支持各种资源类型
    default_type  application/octet-stream;	# 除mime.types中文件类型外，设置其他文件默认类型，访问其他类型时，会提示下载不匹配的类型文件

# 日志配置部分
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    #access_log  logs/access.log  main;

# 自定义优化参数
    sendfile        on;
    #tcp_nopush     on; 		# 在开启了sendfile的情况下，合并请求后统一发送给客户端
    #keepalive_timeout  0;	
    keepalive_timeout  65;	# 设置会话保持时间，第二个值为响应首部：keep-Alived:timeout=65,可以和第一个值不同

    #gzip  on;  # 开启文件压缩

    server {
        listen       80;					# 设置监听地址和端口
        server_name  localhost;		# 设置server name,可以以空格隔开写多个并支持正则表达式

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
```

### 3.3.1 MIME

```shell
# 在响应报文中将指定的文件扩展名映射至MIME对应的类型
include   /etc/nginx/mime.types;
default_type 				application/octet-stream;
types {
	text/html html;
	images/gif gif;
	images/jpeg jpg;
}
```

**范例：识别php文件为text/html**

```bash
[root@localhost ~]# cat << eof > /apps/nginx/html/test.php
> <?php
> phpinfo();
> ?>
> eof
```

### 3.3.2 指定响应报文server首部

```shell
# 是否在响应报文中的Content-Type显示指定的字符集，默认off不显示
charset charset | off

# 示例
charset utf-8

# 是否在响应报文的Server首部显示nginx版本
server_tokens on | off | build | string
```

范例：修改server字段

```shell
如果想自定义响应报文的Nginx版本信息，需要修改源码文件，重新编译
如果server_tokens on,修改 /usr/local/src/nginx-1.18.0/src/core/nginx.h
# define NGINX_VERSION  				"1.68.9"
# define NGINX_VER							"wanginx" NGINX_VERSION	
		
如果server_tokens on,修改 /usr/local/src/http/ngx_http_header_filter_module.c
```

## 3.4 核心配置示例

基于不同的IP,不同的端口以及不同域名实现不同的虚拟主机，依赖于核心模块ngx_http_core_module实现

### 3.4.1 新建一个PC web站点

```shell
# 定义子配置文件路径
[root@localhost ~]# mkdir /apps/nginx/conf.d
[root@localhost ~]# vim /apps/nginx/conf/nginx.conf
http {
...
  	 include /apps/nginx/conf.d/*.conf
}


# 创建PC网站配置
[root@localhost nginx]# vim /apps/nginx/conf.d/pc.conf
server {
  listen 80;
  server_name www.test.com;
  location / {
    root /data/nginx/html/pc;
  }
}

[root@localhost ~]# mkdir -p /data/nginx/html/pc
[root@localhost ~]# echo "pc web" > /data/nginx/html/pc/index.html
[root@localhost ~]# systemctl reload nginx
```

### 3.4.2 创建Mobile web站点

```bash
[root@localhost conf.d]# cat /apps/nginx/conf.d/mobile.conf
server {
  listen 80;
  server_name mobile.wjx.org
  location / {
    root /data/nginx/html/mobile;
  }
}
[root@localhost ~]# mkdir -p /data/nginx/html/mobile
[root@localhost ~]# echo "mobile web" > /data/nginx/html/mobile/index.html
```

当出现错误时，如何查看日志

```bash
tail /apps/nginx/logs/error.log
```

### 3.4.3 root与alias

root：指定web的家目录，定义location的时候，文件的绝对路径等于root+location

alias：定义路径别名，会把路径重新定义到其他指定的路径

### 3.4.4 location的详细使用

在一个server中location配置段可存在多个，用于实现从uri到文件系统的路径映射，nginx会根据用户请求的URI来检查定义的所有location，按一定的优化级找出一个最佳匹配，而后应用其配置

在没有使用正则表达式的时候，nginx会先在server中的多个Location选取匹配度最高的一个uri，uri是用户请求的字符串，即域名后面的web文件路径，然后使用该location模块中的正则uri和字符串，如果匹配成功就结束搜索，并使用此location处理此内容

 location官方帮助:http://nginx.org/en/docs/http/ngx_http_core_module.html#location  

```bash
# 语法规则
location [= | ~ | ~* ] uri {...}

=  	# 用于标准uri前，需要请求字串与uri精确匹配，大小敏感,如果匹配成功就停止向下匹配并立即处理请求
^~	# 用于标准uri前，表示包含正则表达式，并且匹配以指定的正则表达式开头,对URI的最左边部分做匹配检查，不区分字符大小写
~		# 用于标准uri前，表示包含正则表达式，并且区分大小写
~* 	# 用于标准uri前，表示包含正则表达式， 并且不区分大写
不带符号 # 匹配起始于此uri的所有的uri

# 匹配优先级从高到低：
=，^~, ~/~*,  不带符号
```

- 官方范例 

- - The `“ / ”` request will match configuration A 
  - the `“ /index.html ”` request will match configuration B 
  - the `“ /documents/document.html ”` request will match configuration C 
  - the `“ /images/1.gif ”` request will match configuration D 
  - the `“ /documents/1.jpg ”` request will match configuration E  

```bash
location = / {
		[ configuration A ]
}

location / {
		[ configuration B ]
}

location /documents/ {
		[ configuration C ]
}

location ^~ /images/ {
		[ configuration D ]
}

location ~* \.(gif|jpg|jpeg)$ {
		[ configuration E ]
}
```

#### 精确匹配

- 精确匹配Logo

```bash
server {
    listen 80;
    access_log /apps/nginx/logs/access_www_test_com.log main;
    error_log  /apps/nginx/logs/error_www_test_com.log;
    server_name www.test.com;
    location / {
        root /apps/nginx/html/www;
    }

    location = /logo.jpg {
        root /apps/nginx/html/images;
    }
}
```

#### 区分大小写

- `~`实现区分大小写的模糊匹配，以下范例中，如果访问uri中包含大写字母的JPG，则以下location匹配Ax.jpg条件不成功，因为`~`区分大小写，那么当用户的请求被执行匹配时发现location中定义的是小写的jpg.则匹配失败

```shell
server {
    listen 80;
    access_log /apps/nginx/logs/access_www_test_com.log main;
    error_log  /apps/nginx/logs/error_www_test_com.log;
    server_name www.test.com;
    location / {
        root /apps/nginx/html/www;
    }

    location ~ /A.?\.jpg {
      	index index.html;
        root /apps/nginx/html/images;
    }
}
```

#### 不区分大小写

- ~*用来对用户请求的uri做模糊匹配，uri中无论都是大写，小写或者大小写混合，都会匹配成功，通常使用此模式匹配用户request中的静态资源，并继续做下一步操作，
- 注意：此方式中，对LINUX文件系统上的文件仍然是区分大小写的，如果磁盘文件不存在，仍然会提示404

```bash
server {
    listen 80;
    access_log /apps/nginx/logs/access_www_test_com.log main;
    error_log  /apps/nginx/logs/error_www_test_com.log;
    server_name www.test.com;
    location / {
        root /apps/nginx/html/www;
    }

    location ~* /A.?\.jpg {
      	index index.html;
        root /apps/nginx/html/images;
    }
}
```

#### URI开始

```bash
server {
    listen 80;
    access_log /apps/nginx/logs/access_www_test_com.log main;
    error_log  /apps/nginx/logs/error_www_test_com.log;
    server_name www.test.com;
    location / {
        root /apps/nginx/html/www;
    }

    location /api {
      	index index.html
        root /apps/nginx/html/api;
    }
}
```

#### 文件名后缀

```bash
location ~* \.(gif|jpg|jpeg|bmp|png|tiff|tif|ico|wmf|js|css)$ {
   index index.html;
   root /data/nginx/static/;
}
```

#### 优先级

```bash
location = /1.jpg {
	index index.html;
	root /data/nginx/static1;
}

location /1.jpg {
	index index.html;
	root /data/nginx/static2;
}

location ~* \.(gif|jpg|jpeg|bmp|png|tiff|tif|ico|wmf|js|css)$ {
	index index.html;
	root /data/nginx/static3;
}
```

- 匹配优先级

- - location `=`
  - location `^~ `
  - location `~,~*`正则
  - location   完整路径
  - location   部分起始路径

## 3.5 Nginx四层访问控制

访问控制基于模块ngx_http_access_module实现，可以通过匹配客户端源IP地址进行限制

范例：

```bash
location = /login/ {
	root /data/nginx/html/pc;
	allow 192.168.46.40
  deny all;
}
```

### 3.5.1 Nginx账户认证功能

 由ngx_http_auth_basic_module模块提供此功能  

 官方帮助：[模块官方文档](http://nginx.org/en/docs/http/ngx_http_auth_basic_module.html)  

```bash
[root@localhost ~]# htpasswd -cb /apps/nginx/conf/.htpasswd user1 123456

# -c     创建用户
# -b     非交互方式提交密码

[root@localhost ~]# vim /apps/nginx/conf.d/pc.conf
server {
    listen 80;
    access_log /apps/nginx/logs/access_www_test_com.log main;
    error_log  /apps/nginx/logs/error_www_test_com.log;
    server_name www.test.com;
    location / {
        root /apps/nginx/html/www;
    }

    location /Aa.jpg {
      auth_basic "login password";
      auth_basic_user_file /apps/nginx/conf/.htpasswd;
      root  /apps/nginx/html/images;
    }
}
```

## 3.6 日志设置

### 3.6.1 自定义错误页面

定义错误页，以指定的响应状态码进行响应，可用位置：http,server,location,if in location 

范例：

```shell
server {
    listen 80;
    access_log /apps/nginx/logs/access_www_test_com.log main;
    charset utf-8;
    error_log  /apps/nginx/logs/error_www_test_com.log;
    server_name www.test.com;
    location / {
        root /apps/nginx/html/;
    }

    error_page 404 /40x.html;
    location = /error.html {
      root /data/nginx/html;
    }
}
```

### 3.6.2 自定义错误日志

可以自定义错误日志

```shell
access_log /apps/nginx/logs/access_www_test_com.log main;
error_log  /apps/nginx/logs/error_www_test_com.log;
```

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664343047738-6cd1b6f5-4cd8-4cae-a0f1-6081210a064f.png)

## 3.7 长连接配置

```bash
keepalive_timeout timeout [header_timeout]
# 设定保持连接超时时长，0表示禁止长连接，默认为75s，通常配置在http字段作为站点全局配置
keeplive_requests number;
# 在一次长连接上所允许请求的资源的最大数量，默认为100次，建议适当调大，比如：500
```

**范例**

```bash
keepalive_request 3;
keepalive_timeout 65 60;
# 开启长连接后，返回客户端的会话保持时间为60s，单次长连接累计请求达到指定次数请求或65秒就会被断开，后面的60为发送给客户端应答报文头部中显示的超时时间设置为60s，如不设置客户端将不显示超时时间。

keep-Alive:timeout=60;
# 浏览器收到的服务器返回的报文
# 如果设置为0表示关闭会话保持功能；将如下显示：
Connection:close		# 浏览器收到的服务器返回的报文
```

## 3.8 作为下载服务器

- `ngx_http_autoindex_module`模块处理以斜杠`"/"`结尾的请求，并生成目录列表可以作为下载服务配置使用00x
- 官方文档：[链接](http://nginx.org/en/docs/http/ngx_http_autoindex_module.html)

```bash
autoindex on|off;
# 自动文件索引功能，默认off
autoindex_exact_size on|off;
# 计算文件确切大小（单位bytes）,off显示大概大小（单位K,M）,默认on
autoindex_location on|off;
# 显示本机时间而非GMT时间，默认off
autoindex_format html | xml | json | jsonp;
# 显示索引的页面分割，默认html
limit_rate rate;
# 限制响应客户端传输速率，表示无限制，此指令由ngx_http_core_module提供
```

**范例：实现下载站点**

```bash
[root@localhost ~]# mkdir -p /apps/nginx/conf.d/
[root@localhost ~]# vim /apps/nginx/conf.d/test.conf
server {
    listen 80;
    access_log /apps/nginx/logs/access_www_file_com.log main;
    charset utf-8;
    error_log  /apps/nginx/logs/error_www_file_com.log;
    server_name www.file.com;
    location / {
        root /apps/nginx/html/file;
        autoindex on;
        autoindex_exact_size on;
        autoindex_localtime on;
    }
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664361918161-9c1b1334-186f-4e2e-a7f3-44f7a612a727.png)

## 3.9 作为上传服务器

```shell
client_max_body_size 1m;
# 设置允许客户端上传单个文件的最大值，默认值为1m,上传文件超过此值会出现413错误
client_body_buffer_size size;
# 用户接受每个客户端请求报文的body部分的缓冲区大小;默认16k;超出此大小是，其将被暂存到磁盘上下面client_body_temp_path指定所定义的位置

client_body_temp_path path [level1 [level 2 [level 3]]];
# 设定存储客户端请求报文的body部分的临时存储路径及子目录结构和数量，目录名为16进制的数字，使用


hash之后的值从后往前截取1位、2位、2位作为目录名
# 1级目录占1位16进制，即2^4=16个目录 0-f
# 2级默认占2位16进制，即2^8=256个目录 00-ff
# 3级目录占2位16进制，即2^8=256个目录
[root@localhost ~]# md5sum /data/nginx/html/www/index.html
```

# 4 Nginx高级配置

## 4.1 Nginx状态页

-  基于nginx模块ngx_http_stub_status_ module实现，在编译安装nginx的时候需要添加编译参数--withhttp_stub_status_module,否则配置完成之后监测会是提示语法错误  
- 注意：状态页显示的是整个服务器的状态，而非虚拟主机的状态

```bash
location /nginx_status {
	stub_status;
	auth_basic "auth login";
	auth_basic_user_file /apps/nginx/conf/.htpasswd;
	allow 192.168.0.0/16;
	allow 127.0.0.1;
	deny all;
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664366702061-87914866-d8b0-46d6-964b-104add81e940.png)

- Active connections：4表示Nginx正在处理的活动连接数4个
- server 41：表示Nginx启动到现在共处理了41个连接
- accepts 41：表示Nginx启动到现在共成功创建41次握手
- handled requests 74：表示总共处理了74次请求
- Reading：Nginx读取到客户端的Header信息数，数值越大，说明排队越长，性能不足
- Writing：Nginx返回给客户端Header信息数，数值越大，说明访问量越大
- Waiting：Nginx 已经处理完正在等候下一次请求指定的驻留链接 （开启keep-alive的情况下，这个值等于 Active-(Reading+Writing)）  

## 4.2 Nginx第三方模块

 第三模块是对nginx的功能扩展，第三方模块需要在编译安装Nginx的时候使用参数--add-module=PATH指定路 径添加，有的模块是由公司的开发人员针对业务需求定制开发的，有的模块是开源爱好者开发好之后上传到 github进行开源的模块，nginx支持第三方模块需要从源码重新编译支持，比如:开源的echo模块https://github. com/openresty/echo-nginx-module  

```shell
location /main {
		 index index.html;
		 default_type text/html;
		 echo "hello world,main-->";
		 echo $remote_addr;
		 echo_reset_timer; # 将计时器开始时间重置为当前时间
		 echo_location /sub1;
		 echo_location /sub2;
		 echo "took $echo_timer_elapsed sec for total.";
}
location /sub1 {
		 echo_sleep 2;
		 echo hello;
}
location /sub2 {
		echo_sleep 1;
		echo world;
}
[root@localhost ~]# cd /usr/local/src
[root@localhost src]# git clone https://github.com/openresty/echo-nginxmodule.git
[root@localhost src]# cd nginx-1.18.0
[root@localhost nginx-1.18.0]# ./configure --prefix=/apps/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-pcre \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module \
--add-module=/usr/local/src/echo-nginx-module
[root@localhost nginx-1.18.0]# make -j 2 && make install
```

## 4.3 Nginx变量使用

- nginx的变量可以在配置文件中使用，作为功能判断或者日志等场景使用
- 变量可以分为内置变量和自定义变量
- 内置变量是由nginx模块自带，通过变量可以获取到众多的与客户端访问相关的值

### 4.3.1 内置变量

```bash
$remote_addr;
# 存放了客户端的地址，注意是客户端的公网IP
$proxy_add_x_forwarded_for;
# 此变量表示将客户端IP追加请求报文中X-Forwarded-For首部字段，多个IP之间用逗号分隔，如果请求
中没有X-Forwarder-For，就使用$remote_addr
$args;
# 变量中存放了URL中的参数
$document_root;
# 保存了针对当前资源的系统根目录
$document_uri;
# 保存了当前请求中不包含参数的URI,注意是不包含请求的指令，比如/img/logo.png
$host;
# 存放了请求的host名称
limit_rate 10240;
echo $limit_rate;
# 如果nginx服务器使用limit_rate配置了显示网络速率，则会显示，如果没有设置，则显示0
$remote_port;
# 客户端请求Nginx服务器时随机打开的端口，这是每个客户端自己的端口
$remote_user;
# 已经经过Auth Basic Module验证的用户名
$request_body_file;
# 做反向代理时发给后端服务器的本地资源的名称
$request_method;
# 请求资源的方式，GET/PUT等等
$request_filename;
# 当前请求的资源文件的磁盘路径，由root或alias指令与URL请求生成的文件绝对路径
# /apps/nginx/html/www/index.html
$request_uri;
# 包含请求参数的原始URI,不包含主机名，相当于:$document_uri?$args
$scheme;
# 请求的协议，例如：http,https,ftp等等
$server_protocol;
# 保存了客户端请求资源使用的协议版本，例如：HTTP/1.0,HTTP/1.1,HTTP/2.0等等
$server_addr;
# 保存了服务器的IP地址
$server_name;
# 请求的服务器的主机名
$server_port;
# 请求的服务器的端口号
$http_<name>
# name为任意请求报文首部字段，表示记录请求报文的首部字段
$http_user_agent;
# 客户端浏览器的详细信息
$http_cookie;
# 客户端的cookie信息
$cookie_<name>
# name为任意请求报文首部字段cookie的key名
```

## 4.4 Nginx自定义访问日志

访问日志是记录客户端即用户的具体请求内容信息，全局配置模块中的error_log是记录nginx服务器运行时的日志保存路径和记录日志的level，因此有着本质的区别，而且Nginx的错误日志一般只有一个，但是访问日志可以在不同server中定义多个，定义一个日志需使用access_log指定日志的保存路径，使用log_format指定日志的格式，格式中定义要保存的具体日志内容

 访问日志由`ngx_http_log_module` 模块实现  

 官方帮助文档：http://nginx.org/en/docs/http/ngx_http_log_module.html  

### 4.4.1 自定义默认格式日志

如果是要保留日志的源格式，只是添加相应的日志内容

```shell
log_format nginx_format1 '$remote_addr - $remote_user [$time_local]
"$request" '
						'$status $body_bytes_sent "$http_referer" '
						'"$http_user_agent" "$http_x_forwarded_for"'
						'$server_name:$server_port';

access_log logs/access.log nginx_format1;
# 重启nginx并访问测试日志格式
# /apps/nginx/logs/access.log
```

# 5 LNMP架构概述

## 5.1 什么是LNMP

- LNMP是一套技术的组合，L=Linux，N=Nginx，M=MySQL，P=PHP

## 5.2 LNMP架构如何工作

- 首先nginx服务是不能请求动态请求，当用户发起动态请求时，nginx无法处理
- 当用户发起http请求，请求会被nginx处理，如果是静态资源请求nginx则直接返回，如果是动态请求，nginx则通过fastcgi协议转交给后端的PHP程序处理

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664418930844-c32abbef-bfef-463c-bb00-8aa31468f0c3.png)

## 5.3 Nginx与fastcgi详细工作流程

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664418958186-83d6c00a-144f-499a-b151-a4e51a073888.png)

1. 用户通过http协议发起请求，请求会先抵达LNMP架构中的Nginx；
2. nginx会根据用户的请求进行location规则匹配；
3. location如果匹配到请求是静态，则由nginx读取本地直接返回；
4. location如果匹配到请求时动态，则由Nginx将请求转发给fastcgi协议
5. fastcgi收到请求交给php-fpm管理进程，php-fpm管理进程接收到后会调用具体的工作进程wrapper;
6. wrapper进程会调用PHP程序进行解析，如果只是解析代码，php直接返回；
7. 如果有查询数据库操作，则由php连接数据库（用户 密码 ip）发起查询的操作；
8. 最终稿数据由mysql->php->php-fpm->fastcgi->nginx->http->user

# 6 LNMP架构环境部署

## 6.1 使用官方仓库安装nginx

```bash
[root@localhost ~]# vim /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1


[root@localhost ~]# yum -y install nginx
```

## 6.2 修改Nginx用户

```bash
[root@localhost ~]# groupadd www -g 666
[root@localhost ~]# useradd www -u 666 -g 666 -s /sbin/nologin -M
[root@localhost ~]# sed -i '/^user/c user www;' /etc/nginx/nginx.conf
```

## 6.3 启动nginx并加入开机自启

```bash
[root@localhost ~]# systemctl start nginx
[root@localhost ~]# systemctl enable nginx
```

## 6.4 使用第三方扩展源安装php7.1 

```bash
# rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest7.noarch.rpm
# rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

[root@localhost ~]# vim /etc/yum.repos.d/php.repo
[php]
name = php Repository
baseurl = http://repo.webtatic.com/yum/el7/x86_64/
gpgcheck = 0

[root@localhost ~]# yum -y install epel-release
[root@localhost ~]# yum -y install php71w php71w-cli php71w-common php71w-devel php71w-embedded php71w-gd php71w-mcrypt php71w-mbstring php71w-pdo php71w-xml php71w-fpm php71w-mysqlnd php71w-opcache php71w-pecl-memcached php71w-peclredis php71w-pecl-mongodb
```

## 6.5 配置php-fpm用户与nginx的运行用户保持一致  

```bash
[root@localhost ~]# sed -i '/^user/c user = www' /etc/php-fpm.d/www.conf
```

## 6.6 启动php-fpm并加入开机自启

```bash
[root@localhost ~]# systemctl start php-fpm
[root@localhost ~]# systemctl enable php-fpm
```

## 6.7 安装mariadb数据库

```bash
[root@localhost ~]# yum install mariadb-server mariadb -y
[root@localhost ~]# systemctl start mariadb
[root@localhost ~]# systemctl enable mariadb
[root@localhost ~]# mysqladmin password '123456'
[root@localhost ~]# mysql -uroot -p123456
```

# 7 LNMP架构环境配置

- 在将nginx与PHP集成的过程中，需要先了解fastcgi代理配置语法

## 7.1 设置fastcgi服务器的地址

- 该地址可以指定为域名或IP地址，以及端口

```bash
Syntax: fastcgi_pass address;
Default:-
Context:location,if in location
#语法示例
fastcgi_pass location:9000;
fastcgi_pass unix:/tmp/fastcgi.socket;
```

## 7.2 设置fastcgi默认的首页文件

- 需结合fastcgi——param一起设置

```bash
Syntax: fastcgi_index name;
Default:-
Context:http,server,location
```

## 7.3 Nginx连接Fastcgi服务器配置

```bash
[root@localhost ~]# vim /etc/nginx/conf.d/php.conf
server {
				listen 80;
				server_name php.test.com;
				root /code;
				location / {
								index index.php index.html;
				}

				location ~ \.php$ {
								fastcgi_pass 127.0.0.1:9000;
								fastcgi_param SCRIPT_FILENAME
$document_root$fastcgi_script_name;
								include fastcgi_params;
				}
}
[root@localhost ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

[root@localhost ~]# systemctl restart nginx
```

## 7.4 测试Fastcgi是否正常

```bash
[root@localhost ~]# vim /code/info.php
<?php
		phpinfo();
?>
```

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664420881026-2704aaf7-cc81-4c96-bdae-6eb86e79afaf.png)

## 7.5 测试数据库连接

```php
[root@localhost ~]# vim /code/mysqli.php
<?php
  $servername = "localhost";
	$username = "root";
	$password = "123456";

	// 创建连接
	$conn = mysqli_connect($servername, $username, $password);

	// 检测连接
	if (!$conn) {
  die("Connection failed: " . mysqli_connect_error());
}
	echo "连接MySQL...成功！";
?>
```

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664420998359-4b05beea-3387-4e4e-a534-b6f9fc00d37f.png)

# 8 部署WordPress

## 8.1 配置Nginx虚拟主机站点

- 部署博客产品WordPress配置Nginx虚拟主机站点，域名blog.test.com

```bash
[root@localhost ~]# vim /etc/nginx/conf.d/wordpress.conf
server {
				listen 80;
				server_name blog.test.com;
				root /code/wordpress;
				index index.php index.html;

				location ~ \.php$ {
								root /code/wordpress;
								fastcgi_pass 127.0.0.1:9000;
								fastcgi_index index.php;
								fastcgi_param SCRIPT_FILENAME
$document_root$fastcgi_script_name;
								include fastcgi_params;
				}
}
[root@localhost code]# nginx -t
[root@localhost code]# systemctl restart nginx
```

## 8.2 下载wordpress源码

```bash
[root@localhost ~]# cd /code
[root@localhost code]# wget https://cn.wordpress.org/latest-zh_CN.tar.gz
[root@localhost code]# tar xzvf latest-zh_CN.tar.gz
[root@localhost code]# chown -R www.www /code/wordpress
```

## 8.3 创建所需数据库

```bash
[root@localhost ~]# mysql -uroot -p123456 -e "create database wordpress;show databases;"
+--------------------+
| Database |
+--------------------+
| information_schema |
| mysql |
| performance_schema |
| test |
| wordpress |
+--------------------+
```

## 8.4 配置wordpress

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664421308077-4c4aea79-48ca-48f6-a1c2-1e2421a97822.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/27037064/1664421327305-ec6b21a9-75f0-4c42-a62f-363b4e901362.png)

## 8.3 设置文件上传大小限制

- 解决Nginx上传文件大小限制，413错误

```bash
[root@localhost ~]# vim /etc/nginx/conf.d/wordpress.conf
server {
				listen 80;
				server_name blog.test.com;
				root /code/wordpress;
				index index.php index.html;
				client_max_body_size 100m;
				location ~ \.php$ {
									root /code/wordpress;
									fastcgi_pass 127.0.0.1:9000;
									fastcgi_index index.php;
									fastcgi_param SCRIPT_FILENAME
$document_root$fastcgi_script_name;
									include fastcgi_params;
				}
}
[root@localhost ~]# nginx -t
[root@localhost ~]# systemctl restart nginx
```

# 9 部署wecenter（贴吧）

## 9.1 配置Nginx

```bash
[root@localhost ~]# vim /etc/nginx/conf.d/wecenter.conf
server {
				listen 80;
				server_name wecenter.test.com;
				root /code/wecenter;
				index index.php index.html;
				
				location ~ \.php$ {
								fastcgi_pass 127.0.0.1:9000;
								fastcgi_index index.php;
								fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
								include fastcgi_params;
      	}
}

[root@localhost ~]# nginx -t
[root@localhost ~]# systemctl restart nginx
```

## 9.2 获取wecenter源码

-  wecenter下载地址：https://download.s21i.faiusr.com/23126342/0/0/ABUIABBPGAAgjvmphwYowKnu xwc.zip?f=WeCenter_3-6-1.zip&v=1625980046  

```bash
[root@localhost ~]# mkdir -p /code/wecenter
[root@localhost ~]# cd /code/wecenter/
[root@localhost wecenter]# unzip WeCenter_3-6-1.zip
[root@localhost wecenter]# chown -R www.www /code/wecenter/
```

## 9.3 配置数据库

- 由于wecenter产品需要依赖数据库，所以需要手动建立数据库

```bash
[root@localhost wecenter]# mysql -uroot -p123456 -e "create database wecenter;show databases;"
+--------------------+
| Database |
+--------------------+
| information_schema |
| mysql |
| performance_schema |
| test |
| wecenter |
| wordpress |
+--------------------+
```

## 9.4 配置Wecenter











# 10 部署EduSoho (网校)

## 10.1 配置nginx

```bash
[root@localhost ~]# vim /etc/nginx/conf.d/edu.conf
server {
				listen 80;
				server_name edu.test.com;
				root /code/edusoho/web;
				client_max_body_size 200m;
				
				location / {
								index app.php;
								try_files $uri @rewriteapp;
				}
				location @rewriteapp {
								rewrite ^(.*)$ /app.php/$1 last;
				}

				location ~ ^/udisk {
								internal;
								root /code/edusoho/app/data/;
				}
				location ~ ^/(app|app_dev)\.php(/|$) {
								fastcgi_pass 127.0.0.1:9000;
								fastcgi_split_path_info ^(.+\.php)(/.*)$;
								include fastcgi_params;
								fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
								fastcgi_param HTTPS off;
								fastcgi_param HTTP_X-Sendfile-Type X-Accel-Redirect;
								fastcgi_param HTTP_X-Accel-Mapping /udisk=/code/edusoho/app/data/udisk;
								fastcgi_buffer_size 128k;
								fastcgi_buffers 8 128k;
					}
					# 配置设置图片格式文件
					location ~* \.(jpg|jpeg|gif|png|ico|swf)$ {
									# 过期时间为3年
									expires 3y;
									# 关闭日志记录
									access_log off;
									# 关闭gzip压缩，减少CPU消耗，因为图片的压缩率不高。
									gzip off;
					}
					# 配置css/js文件
					location ~* \.(css|js)$ {
									access_log off;
									expires 3y;
					}
					# 禁止用户上传目录下所有.php文件的访问，提高安全性
					location ~ ^/files/.*\.(php|php5)$ {
									deny all;
					}
					# 以下配置允许运行.php的程序，方便于其他第三方系统的集成。
					location ~ \.php$ {
									fastcgi_pass 127.0.0.1:9000;
									fastcgi_split_path_info ^(.+\.php)(/.*)$;
									include fastcgi_params;
									fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
									fastcgi_param HTTPS off;
					}
}



[root@localhost ~]# nginx -t
[root@localhost ~]# systemctl restart nginx
```

## 10.2 获取edusoho的源码

```bash
[root@localhost ~]# cd /code/
[root@localhost code]# wget http://download.edusoho.com/edusoho-8.2.17.tar.gz
[root@localhost code]# tar xzvf edusoho-8.2.17.tar.gz
[root@localhost code]# chown -R www.www edusoho
```

## 10.3 调整php的上传大小

```bash
[root@localhost ~]# vim /etc/php.ini
post_max_size = 200M
upload_max_filesize = 200M
[root@localhost ~]# systemctl restart php-fp
```