---
layout: post
title: Nginx+tomcat+redis 实现负载均衡
categories: [tomcat,nginx]
description: 使用nginx+tomcat进行动静分离+负载均衡
keywords: 负载均衡, 动静分类, session共享
---

### nginx

1. 使用nginx可以进行请求分发,可以把请分发到不同的 tomcat 用于缓解服务器压力，当然tomcat可以位于不同的服务器上。
2. nginx 在处理静态文件的吞吐量上比tomcat好很多。因此可以将静态请求(css,jpg,js)交给nginx，动态请求(jsp,action)交给tomcat，从而提高性能。

## 软件安装 配置

1. nginx 服务器
2. redis 
3. tomcat * 2

### nginx 安装

1. 使用apt-get命令安装
```sh
sudo apt-get install nginx (80端口被占用，安装会报错？)
```
2. 运行 nginx `service nginx start`
3. 停止 nginx `service nginx stop`
4. 重启 nginx `service nginx reload`
5. 查询 nginx 进程 `ps -ef | grep nginx`

### nginx 配置

对 nginx的配置在`/etc/nginx/nginx.conf`中进行
使用 `whereis nginx` 查看 nginx 路径
在http下插入如下配置
```
        server {
        	#服务器开放8080端口
            listen 8080;
            server_name server_nginx;

            location / {
            	#页面存放位置 html
                root   html;
                #欢迎页面
                index  index.html index.htm;
            }
            #错误页
            error_page   500 502 503 504 404  /50x.html;
            location = /50x.html {
                root   html;
            }
        }

```
使用 `service nginx start` 启动服务器
访问 `ip:8080`  如下说明 nginx 安装配置成功
注释 配置文件的 include 语句

  <img height="100%" src="/images/posts/nginx/01.PNG">

### 安装 redis

```
sudo apt-get install redis-server
```
找到文件`redis.conf` 修改`daemonize`为**yes**  让redis-server变为守护进程后台运行
使用此配置文件运行redis-server 
`./redis-server redis.conf`  (whereis redis.conf)
运行`./redis-cli`验证是否能与redis-server服务交互

## 开始主题

### 使用 nginx 进行反向代理

基本过程是客户端浏览器访问 nginx ，然后 nginx 将请求转发给 tomcat 进行处理
先启动一个tomcat 开放在80端口
然后再 nginx 配置文件 `nginx.conf` 进行如下配置
```
        server {
            listen 8080;
            server_name server_nginx;
            #所有请求转发到http://ambermoe.cn:80 进行处理
            location / {
                proxy_pass http://ambermoe.cn:80;
                client_max_body_size 50m; 
            }
            # css|js|png|jpg等静态请求 交由 nginx 处理
            location ~ .*\.(css|js|png|jpg|gif|ico|jpeg)$ {
                #静态文件所在目录
                root /usr/share/tomcat-7/webapps/tmall_ssh;
                #上传大小限制 还要添加tomcat connector 属性 maxPostSize=“0”
                client_max_body_size 50m; 
            }
            error_page   500 502 503 504 404  /50x.html;
            location = /50x.html {
                root   html;
            }
        }

```
配置gzip 对页面进行gzip压缩传到浏览器时再解压可以有效的减少带宽
```
gzip on;
    gzip_min_length 1k;  #最小1K
    gzip_buffers 16 64K;
    gzip_http_version 1.1;
    gzip_comp_level 6;
    gzip_types text/plain application/x-javascript text/css application/xml application/javascript;
    gzip_vary on;
```
压缩效果如下(没压缩之前大小大概1.8m)

 <img height="100%" src="/images/posts/nginx/02.PNG">

重启 nginx 访问ip:8080 可以发现已经反向代理到tomcat了 

## 多服务器负载均衡

### 使用 redis 解决多服务器的 session 共享问题

分别在两个 tomcat 的`tomcat/conf/context.xml` 之中进行如下配置
```
 <Valve className="com.orangefunction.tomcat.redissessions.RedisSessionHandlerValve" />  
  <Manager className="com.orangefunction.tomcat.redissessions.RedisSessionManager"  
   <!-- host redis服务器地址 开放在6379端口 -->
   host="ambermoe.cn"  
   port="6379"  
   database="0"  
   maxInactiveInterval="60" /> 
```
并且 tomcat 连接 redis 需要三个 jar 包`jedis-2.5.2.jar`，`commons-pool2-2.0.jar`,`tomcat-redis-session-manager1.2.jar`
复制到lib目录下

### nginx 中配置 upstream

```
    upstream tomcat_80_8322{
        #weight表示权重，值越大，被分配到的几率越大
        server  ambermoe.cn:80 weight=1;
        server  ambermoe.cn:8322 weight=1;
    }
```

反向代理到 upstream
```
location / {
            proxy_pass http://tomcat_80_8322;
    }
```

  <img height="100%" src="/images/posts/nginx/03.PNG">

从图片中可以看到请求被随机分配到了两台服务器上,但前台还是一直保持登录的。
现在两台服务器(~~没说你，nginx~~)无论挂了那一台 网站还是能正常运行的。
*运行时发现 JSESSIONID 在不停变化 导致登录状态丢失 不知道什么原因 重启tomcat就好了*

### 设置图片上传服务器

现在还有最后一个问题。 因为在这个项目中有图片上传功能。现在一共有 nginx n、tomcat A、tomcat B、三个 web 服务器，上面配置了静态请求由 nginx 处理，图片css等资源是直接到 tomcat A 下取得。现在配置了负载均衡，在图片上传时是随机转发给A或B处理。如果图片传到B下。那么 nginx 是取不到的。所以现在还需配置 tomcat A 为图片上传服务器。所有图片上传请求全部转发给A处理。
根据以上要求做如下配置
```
        # 指定图片上传服务器
        upstream image_server{
            server ambermoe.cn:80 weight=1;
        }
        
        # admin_productImage_add admin_productImage_delete
        # admin_category_update admin_category_add
        # admin_category_delete 这五个都与图片增删改有关
        # ~ 代表区分大小写的正则匹配 .* 代表任意个任意字符
        location ~ .*(admin_productImage_add|admin_productImage_delete|admin_category_update|admin_category_add|admin_category_delete).*{
            proxy_pass http://image_server;
            client_max_body_size 2m;
        }


```
关于location 路径匹配的问题
``` 
    =       开头表示精确匹配 
    ^~      开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa
            可以被规则^~ /static/ /aa匹配到（注意是空格）。 
    ~       开头表示区分大小写的正则匹配 
    ~*      开头表示不区分大小写的正则匹配 
    !~和!~* 分别为区分大小写不匹配及不区分大小写不匹配 的正则 
    /       通用匹配，任何请求都会匹配到

```
 **优先级(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)**
 
后面就是常规的正则表达式了