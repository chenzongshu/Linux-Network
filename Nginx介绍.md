# 简介

Nginx是有名的Web服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器

# 常用命令

```
检查 nginx.conf配置文件
nginx -t

重启
nginx -s reload

停止
nginx -s stop

```

# 配置

## 基本配置

```
# 全局块
# 全局块：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
user www www;  # Nginx运行的用户和用户组,系统中必须有此用户,可以是nologin
worker_processes  1; # 启动进程,通常设置成和cpu的数量相等
error_log  /var/log/nginx/error.log; # 错误日志定义等级，[ debug | info | notice | warn | error | crit ]
pid        /var/run/nginx.pid;
worker_rlimit_nofile 65535; # 一个nginx进程打开的最多文件描述符数目
----------------------
# events块
# events块：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。

events {         
    use   epoll;    # epoll是多路复用IO(I/O Multiplexing)中的一种方式,但是仅用于linux2.6以上内核,可以大大提高nginx的性能
    worker_connections  1024;#单个后台worker process进程的最大并发链接数 （最大连接数=连接数*进程数）
    # multi_accept on;
}
-----------------
# http块
# http块：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。

http      
{
    # http全局块
    #设定mime类型,类型由mime.type文件定义
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    #设定日志格式
    access_log    /var/log/nginx/access.log;
    #sendfile 指令指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，对于普通应用，
    #必须设为 on,如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，以平衡磁盘与网络I/O处理速度，降低系统的uptime.
    sendfile        on;
    #tcp_nopush     on;
    #连接超时时间
    #keepalive_timeout  0;
    keepalive_timeout  65;
    tcp_nodelay        on;
    
    #开启gzip压缩
    gzip  on;
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";
    #设定请求缓冲
    client_header_buffer_size    1k;
    large_client_header_buffers  4 4k;
    #包含其它配置文件，如自定义的虚拟主机
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
    
    #设定负载均衡的服务器列表
upstream mysvr {
    #weigth参数表示权值，权值越高被分配到的几率越大
    server 192.168.8.1x:3128 weight=5;#本机上的Squid开启3128端口
    server 192.168.8.2x:80  weight=1;
    server 192.168.8.3x:80  weight=6;
}

    --------------
    # 虚拟主机server块
    server        
    { 
        # server全局块
        # 侦听80端口
        listen       80;
        #定义使用www.xx.com访问
        server_name  www.xx.com;
        #设定本虚拟主机的访问日志
        access_log  logs/www.xx.com.access.log  main;
        ------------
        # location块
        # location块：配置请求的路由，以及各种页面的处理情况。
        location / {
                  root   /root;      #定义服务器的默认网站根目录位置
                  index index.php index.html index.htm;   #定义首页索引文件的名称
                  fastcgi_pass  www.xx.com;
                  fastcgi_param  SCRIPT_FILENAME  $document_root/$fastcgi_script_name; 
                  include /etc/nginx/fastcgi_params;
            }
            # 定义错误提示页面
            error_page   500 502 503 504 /50x.html;  
                location = /50x.html {
                root   /root;
            }
            #静态文件，nginx自己处理
            location ~ ^/(images|javascript|js|css|flash|media|static)/ {
                root /var/www/virtual/htdocs;
                #过期30天，静态文件不怎么更新，过期可以设大一点，如果频繁更新，则可以设置得小一点。
                expires 30d;
            }
            #PHP 脚本请求全部转发到 FastCGI处理. 使用FastCGI默认配置.
            location ~ \.php$ {
                root /root;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME /home/www/www$fastcgi_script_name;
                include fastcgi_params;
            }
            #设定查看Nginx状态的地址
            location /NginxStatus {
                stub_status            on;
                access_log              on;
                auth_basic              "NginxStatus";
                auth_basic_user_file  conf/htpasswd;
            }
            #禁止访问 .htxxx 文件
            location ~ /\.ht {
                deny all;
            }

    }
    server
    {
      ...
    }
    # http全局块
    ...     
}
```

## 第一台虚拟服务器

```
server {
    #侦听192.168.8.x的80端口
    listen       80;
    server_name  192.168.8.x;
    
    #对aspx后缀的进行负载均衡请求
    location ~ .*\.aspx$ {
      root   /root;      #定义服务器的默认网站根目录位置
      index index.php index.html index.htm;   #定义首页索引文件的名称
      proxy_pass  http://mysvr ;#请求转向mysvr 定义的服务器列表
      
      #以下是一些反向代理的配置.
      proxy_redirect off;
      #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      client_max_body_size 10m;    #允许客户端请求的最大单文件字节数
      client_body_buffer_size 128k;  #缓冲区代理缓冲用户端请求的最大字节数，
      proxy_connect_timeout 90;  #nginx跟后端服务器连接超时时间(代理连接超时)
      proxy_send_timeout 90;        #后端服务器数据回传时间(代理发送超时)
      proxy_read_timeout 90;         #连接成功后，后端服务器响应时间(代理接收超时)
      proxy_buffer_size 4k;             #设置代理服务器（nginx）保存用户头信息的缓冲区大小
      proxy_buffers 4 32k;               #proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
      proxy_busy_buffers_size 64k;    #高负荷下缓冲大小（proxy_buffers*2）
      proxy_temp_file_write_size 64k;  #设定缓存文件夹大小，大于这个值，将从upstream服务器传
   }
 }
}
```
