

### NGINX安装

##### 普通安装

http://nginx.org/en/linux_packages.html#RHEL-CentOS

```bash
http://nginx.org/en/download.html
http://nginx.org/download/nginx-1.18.0.tar.gz


./configure --prefix=/usr/nginx
make && make install 




```



##### docker安装

```bash
#通过本地的docker获取nginx镜像


docker load --input nginx.1.19.10.tar 

docker run -d \
    --name nginx \
    -p 8800:80 \
    -v /opt/nginx/conf:/etc/nginx/conf.d \
    -v /opt/nginx:/opt \
    nginx:1.19.10
    
docker run -d \
    --name nginx \
    -p 8800:80 \
    -v /opt/nginx/conf:/etc/nginx/conf.d \
    -v /opt/idp/frontend:/opt \
    nginx:1.19.10
    
docker logs nginx
docker rm nginx

```



```bash
#前端界面
/opt/nginx/dist

#nginx配置
/opt/nginx/conf

#nginx.conf配置
server{
	listen 80;
	server_name 172.16.90.137;
	location /{
		root /opt/dist;
		index index.html index.htm;		
	}
	
	location /octopus/scheduler {
		proxy_pass http://172.16.90.137:12345; 
		...
	}
	
	error_page 500 502 503 504 /50x.html;
	
	location = /50x.html {
	    root html;
	}
}
```





### su 不用密码

```bash
vim /etc/pam.d/su

#su without a password
auth    sufficient pam_wheel.so trust use_uid

```

