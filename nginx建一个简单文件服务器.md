# 使用Nginx

安装nginx

修改/etc/nginx/nginx.conf配置，增加下面的配置即可，文件存放目录要存在

```
server {
        client_max_body_size 4G;
        ##注意80端口的占用问题
        listen  80;  ## listen for ipv4; this line is default and implied 
        server_name    XXX.XXX.XXX;  ##你的主机名或者是域名
	      root /czsdata/upgrade_file/http;
	          location / {
		            autoindex on; ##显示索引
                autoindex_exact_size on; ##显示大小
		            autoindex_localtime on;   ##显示时间
        }
}
```

# 使用httpd

安装httpd 并启动

```
yum install -y httpd
```

找到 `/var/www/html` , 建个下载文件夹`file`

然后把文件放入`file`中， 然后就可以使用了
