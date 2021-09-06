# idp安装文档



## 1.环境准备

------

### 1.1. 安装环境

https://blog.csdn.net/u011331844/article/details/103759064

| name         | 版本                           |
| ------------ | ------------------------------ |
| CentOS       | 7.6                            |
| MySql        | 5.7.25(MySQL Community Server) |
| CDH          | 6.3.2或更高                    |
| Hive         | 2.1.1                          |
| Zookeeper    | 3.4.5                          |
| Spark        | 2.4.0                          |
| Impala       | 3.2.0                          |
| Kafka        | 2.2.1                          |
| HBase        | 2.1.4                          |
| Apache Sqoop | 1.4.7                          |

### 1.2 安装的相关约定

| 用户/目录类型    | 用户名        | 说明                                              |
| ---------------- | ------------- | ------------------------------------------------- |
| 部署用户         | idp           | linux系统用户,启动Java服务的用户                  |
| 安装目录         | /opt/idp/     | 子目录backend和frontend，分别是后端和前端部署路径 |
| 租户             | idp           | worker里执行shell脚本时的执行用户                 |
| mysql用户        | idp_8011_sa   |                                                   |
| mysql数据库      | idp_demo_8011 |                                                   |
| hdfs用户         | idp           |                                                   |
| hdfs目录         | /user/idp     |                                                   |
| kerberos ipa用户 | idp           |                                                   |
| kerberos ipa用户 | octopus       |                                                   |

### 1.3.Jdk安装

```shell
#上传本地文件到服务器
#如果本机是windows系统打开CMD执行上传语句
> scp D:\jdk-8u281.tar.gz root@10.24.96.73:/root/
#登录远程服务器
#解压缩安装包
> tar -zxvf jdk-8u181-linux-x64.tar.gz

#移动压缩文件
> mv jdk1.8.0_301/ /usr/java/
```

配置环境变量

```shell
> vi /etc/profile

##把以下语句放到环境变量中 
#set java environment
JAVA_HOME=/usr/java/ 
JRE_HOME=/usr/java/jre     
CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
export JAVA_HOME JRE_HOME CLASS_PATH PATH
```

```shell
#执行
按ESC按键输入:wq!
#修改生效
> source /etc/profile
#测试JDK
> java –version
```

<font color='orange'>安装成功如下图</font>

![image-20210803111024068](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20210803111024068.png)

## 2.idp系统部署

### 2.1 部署用户

```shell
# 添加部署用户idp
> useradd idp
# 为idp用户设置密码 Os@QazXsw
> passwd idp
```

### 2.2 ssh互信

部署用户需要在部署机器和其他安装机器上配置ssh免密登录，如果要在部署机上安装调度，需要配置本机免密登录

- 将 **主机器** 和各个其它机器SSH免密打通

  ```shell
  > su - idp  #切换到部署用户idp
  > ssh-keygen -t rsa #一直回车，会在用户的/home目录下生成.ssh
  > ssh-copy-id idp@localhost #更换@后面的localhost，就可以完成指定用户到对应节点的免密登录，本机免密也一样
  ```

### 2.3部署路径

```shell
#登录root账户
> su - root 
# 为账户添加安装目录的创建权限
> chown -R idp /opt/
# 为租户添加安装目录的访问权限
#usermod -aG octopus idp
#chmod g+rwx /home/octopus/
```

```bash
#在所有需要部署调度的机器上创建部署用户，因为worker服务是以 sudo -u {linux-user} 方式来执行作业，所以部署用户需要有 sudo 权限，而且是免密的。
> vi /etc/sudoers

# 租户idp账号写入
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL
idp  ALL=(ALL)       NOPASSWD: ALL

# 并且需要注释掉 Default requiretty 一行
#Default requiretty
```

```shell
# 创建安装目录
> su - idp 
> mkdir -p /opt/idp/backend
> mkdir -p /opt/idp/frontend/
##上传系统后端及前端安装压缩包到服务器
#如果本机是windows系统打开CMD执行上传语句
> scp D:\idp-1.4.0-release-bin.tar idp@10.24.96.73:/home/idp
> scp D:\dist_8033.gz idp@10.24.96.73:/home/idp
#将系统文件解压
> tar -zvxf idp-1.5.0-release-bin.tar.gz 
> unzip dist-1.4.0.zip
#移动文件夹至指定目录
> mv dist /opt/idp/frontend/
```

### 2.4 生成keytab

```shell
# 需要admin进行操作
> ipa user-add idp # 新增用户
> ipa user-mod idp --password # 修改密码
> kinit idp # 新增用户必须先登录并根据提示修改一次密码，密码可以原先的相同（密码：Os@QazXsw）
> ipa-getkeytab -s ipa.ruisdata.com -p idp@RUISDATA.COM -k /home/idp/idp.keytab -P 
# 请注意此时freeipa会自动给该用户重置密码，因此使用kinit用户名然后输入密码的方式无法登录。
#如果希望仍然可以使用密码认证，可以加入-P参数指定密码
```

* **生成的keytab权限要与租户对应， 例如使用idp租户， 则idp需要有keytab的权限**

* 对于多租户的情况，按需放开keytab的权限到644

* 将多个租户合并到一个keytab中：


```shell
# 不同租户同样指定相同的keytab文件
# 设置密码时， 改成与原先的一致即可 ()
> ipa-getkeytab -s you.kdc.server.com -p octopus_2@REALMS.COM -k <path/xx.keytab> -P 
```



### 2.5 数据库初始化

```shell
#远程到docker服务器
> ssh root@10.24.69.9
#登录docker上的MySql
> docker exec -it mysql5.7 bash 
#登录mysql root 账户
> mysql -u root -p
#输入密码
> 123456
```

#### 2.5.1创建database、user并赋权执行以下命令创建database和账号	

```sql
CREATE DATABASE idp_demo_8011 DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'idp_8011_sa'@'%' IDENTIFIED BY 'Os@123456';
GRANT ALL PRIVILEGES ON idp_demo_8011.* TO 'idp_8011_sa'@'%' IDENTIFIED BY 'Os@123456';
GRANT ALL PRIVILEGES ON idp_demo_8011.* TO 'idp_8011_sa'@'localhost' IDENTIFIED BY 'Os@123456';
flush privileges;

```



#### 2.5.2创建表和导入基础数据 

```shell
#返回系统安装服务器
#登录idp账户
> su - idp
#进入系统文件
> cd /home/idp/idp-1.4.0-release
```

```properties
#修改./conf/application.properties中的下列属性
spring.datasource.url=jdbc:mysql://10.24.69.9:3306/idp_demo_8011?characterEncoding=UTF-8
spring.datasource.username=idp_8011_sa
spring.datasource.password=Os@123456

```

#### 2.5.3执行创建表和导入基础数据脚本

```shell
> sh ./script/create-octopusscheduler.sh
```



```shell
#手动更新
#导入sql脚本
> scp ./sql/dolphinscheduler_mysql.sql root@10.24.
69.9:/root/
#登录数据库服务器
> ssh root@10.24.69.9
#拷贝脚本到mysql根目录
> docker cp dolphinscheduler_mysql.sql  mysql5.7:.
#登录docker上的MySql
> docker exec -it mysql5.7 bash 
#登录mysql idp_8011_sa 账户
> mysql -u idp_8011_sa -p
#输入密码
> Os@123456
#执行sql脚本
mysql> use idp_demozml_8011
mysql> source dolphinscheduler_mysql.sql
```



### 2.6后端配置

#### 2.6.1修改install_config.conf配置文件

```shell
>vi /home/idp/idp-1.4.0-release/conf/config/install_config.conf #根据解压后的路径
```

```properties
#根据实际ip修改下面的配置
installPath=/opt/idp/backend
deployUser=idp
ips=10.24.96.73
sshPort=22
masters=10.24.96.73
workers=10.24.96.73
alertServer=10.24.96.73
apiServers=10.24.96.73
dataAssetsServers=10.24.96.73
```

#### 2.6.2修改install.sh脚本

请参考install.sh中的注释.

```shell
> vi /home/idp/idp-1.4.0-release/install.sh #根据解压后的路径
```

```properties
# idp系统需要的数据库类型（默认mysql）
dbtype="mysql"

# mysql服务器的地址:端口
dbhost="10.24.69.9:3306"

# 数据库名称
dbname="idp_demo_8011"

# 访问数据库的用户名（需要新建一个用户，不要用root）
username="idp_8011_sa"

# 访问数据库用户的密码
passowrd="Os@123456"

# conf/config/install_config.conf config
# idp系统的安装目录（默认都安装在 /opt/idp/backend )
installPath="/opt/idp/backend"

# deployment user
# 部署用户，启动idp服务，执行任务的用户
deployUser="idp"

# zk cluster
#zookeeper服务器地址
#zkQuorum="192.168.xx.xx:2181,192.168.xx.xx:2181"
zkQuorum="10.24.96.73:2181"

# install hosts
# 需要安装idp服务的主机ip列表
#ips="ark0,ark1"
ips="10.24.96.73"

# ssh port, default 22
# ssh的端口（默认是22）
sshPort=22

# run master machine
# 运行master服务的主机ip（至少配置一台）
#masters="ark0,ark1"
masters="10.24.96.73"

# run worker machine
# 运行worker服务的主机ip列表（一般是配置多台）
#workers="ark2,ark3,ark4"
workers="10.24.96.73"
# run alert machine
# 运行告警服务的主机ip（一台就够了）
#alertServer="ark3"
alertServer="10.24.96.73"

# run api machine
# api服务的主机ip（缺省一台；配置多台后，需在nginx里配置upstream）
#apiServers="ark1"
apiServers="10.24.96.73"

# run data asset machine
# 数据资产的api服务（缺省一台）
#dataAssetsServers="ark1"
dataAssetsServers="10.24.96.73"

# alert config
# 告警邮件发送的配置开始...
# 告警配置， 邮件发送协议（默认 SMTP）
mailProtocol="SMTP"

# 邮件发送服务器
mailServerHost="smtp.exmail.qq.com"

# 邮件发送服务器的端口
mailServerPort="25"

# 邮件发送人
mailSender="xxxxxxxxxx"

# 邮件用户（一般和发件人相同）
mailUser="xxxxxxxxxx"

# 邮件用户的密码
mailPassword="xxxxxxxxxx"

# 是否支持StartTLS
starttlsEnable="false"

# ssl信任的主机列表（ 默认* ） 
sslTrust="xxxxxxxxxx"

# SSL mail protocol support
# 是否支持SSL
sslEnable="true"

##例子
#alert.type=EMAIL
#mail.protocol=SMTP
#mail.server.host=stmp.163.com
#mail.server.port=994
#mail.sender=bigtime@163.com
#mail.user=bigtime@163.com
#mail.password=112233
#mail.smtp.starttls.enable=false
#mail.smtp.ssl.enable=true
#mail.smtp.ssl.trust=*
## 告警邮件发送配置结束。


# if resUploadStartupType is HDFS，defaultFS write namenode address，HA you need to put core-site.xml and hdfs-site.xml in the conf directory.
# if S3，write S3 address，HA，for example ：s3a://idp，
# Note，s3 be sure to create the root directory /idp
# 文件服务，支持hdfs和s3
# hdfs的配置，需要在/etc/hadoop/hdfs-site.xml里查找。
defaultFS="hdfs://mycluster:8020"

# if S3 is configured, the following configuration is required.
# 如果上面的配置选择是S3， 需要配置下面三行S3的配置.
s3Endpoint="http://192.168.xx.xx:9010"
s3AccessKey="xxxxxxxxxx"
s3SecretKey="xxxxxxxxxx"

# resourcemanager HA configuration, if it is a single resourcemanager, here is yarnHaIps=""
# yarn集群的地址列表。 如果yarn是单机的这里要改为""
yarnHaIps="192.168.xx.xx,192.168.xx.xx"

# if it is a single resourcemanager, you only need to configure one host name. If it is resourcemanager HA, the default configuration is fine.
# 如果yarn是单个节点的，这里需要配置yarn节点的ip。如果是多个节点这里的配置不用修改。
#singleYarnIp="ark1"


# hdfs root path, the owner of the root path must be the deployment user.
# versions prior to 1.1.0 do not automatically create the hdfs root directory, you need to create it yourself.
# idp在hdfs里使用的目录，缺省是 （/user/idp）
hdfsPath="/user/idp"

# have users who create directory permissions under hdfs root path /
# Note: if kerberos is enabled, hdfsRootUser="" can be used directly.
# 系统里有kerberos开着，这里需要改成"". 如果无kerberos,需要使用有权限的用户，一般是hdfs。
hdfsRootUser="hdfs"

# zk config
# zk root directory
zkRoot="/idp"
```

#### 2.6.3安装系统服务

```shell
 #修改install.sh后， 执行
 > sh install.sh
```

2.6.4 安装licence

```shell
将文件拷贝到 /opt/idp/backend/conf/license
```



在安装完成后, 使用jps查看服务是否启动，在对应节点上有对应的服务， 那么就说明服务安装完成。

![image-20210803141344397](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20210803141344397.png)

#### 2.6.4重启系统服务

```shell
#使用idp账户
> cd /opt/idp/backend/bin/
> sh stop-all.sh
> sh start-all.sh
```



### 2.7前端部署

#### 2.7.1安装nginx

```shell
#下载安装插件
>su - root #登录root 账户
> yum install -y gcc pcre pcre-devel openssl openssl-devel gd gd-devel

#解压nginx安装包
> tar -zvxf nginx-1.18.0.tar.gz
#进入解压后的nginx文件
> cd ./nginx-1.18.0
#更改安装路径
> ./configure --prefix=/usr/local/nginx
#安装
> make && make install
```

#### 2.7.2修改配置文件

```shell
> vi /usr/local/nginx/conf/nginx.conf
#添加以下内容
server {
    listen       8888;# 访问端口
    server_name  10.24.96.73;
    client_max_body_size 1024m; #用户上传文件的最大大小，需要与后端配置文件中设置的值保持一致
    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;
    location / {
        root   /opt/idp/frontend/dist; # 上面前端解压的dist目录地址(自行修改)
        index  index.html index.html;
    }
    location /octopus/scheduler {
    		# prod: 10.104.1.183:12345
        proxy_pass http://10.24.96.73:12345; # 接口地址(自行修改)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header x_real_ipP $remote_addr;
        proxy_set_header remote_addr $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_connect_timeout 4s;
        proxy_read_timeout 30s;
        proxy_send_timeout 12s;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    #error_page  404              /404.html;
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
 }
```

#### 2.7.3启动nginx

```shell
> cd /usr/local/nginx/sbin
> ./nginx
```

```properties
如无法访问查看nginx端口是否开通防火墙可以查看一下文档

https://blog.csdn.net/ysyut/article/details/113519541

完成上述安装后，就可以在浏览器中访问http://本机ip:8888进入系统了，账号密码为: admin/bigtimes2020

测试稳定后，可以将`development.state`改为false，让系统自动清理任务执行完成后的文件，或者也可以配置调度任务定时清理。
```

2.8 修改JVM配置大小

```SHELL
vim /opt/idp/backend/bin/dolphinscheduler-daemon.sh

export DOLPHINSCHEDULER_OPTS="-server -Xmx1g -Xms1g -Xss512k -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70"
```

