### MySQL安装

####  普通安装

https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/

```bash
shell> sudo grep 'temporary password' /var/log/mysqld.log
shell> mysql -uroot -p
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'Btn@20210218';

```



```bash
 yum install -y vim 
 rpm  -Uvh mysql57-community-release-el7-10.noarch.rpm
 
 yum install mysql-community-client  #安装客户端 （每台需要访问数据库的电脑上）
 yum install mysql-community-server  #安装服务端 （只需安装一台mysql服务器）
```

```sql
#数据库备份用户
GRANT  SELECT,LOCK TABLES,SHOW VIEW,TRIGGER ON octopusscheduler.* TO 'octopus_backup'@'localhost' IDENTIFIED BY 'Bk@123456';

GRANT  RELOAD,REPLICATION CLIENT ON *.* TO 'octopusscheduler'@'localhost' IDENTIFIED BY 'Bk@123456';

GRANT PROCESS ON *.* to 'octopus_backup'@'localhost';
```

#### 数据库备份

配置

```bash
cat /etc/my_dump.cnf

[mysqldump]
user=octopus_backup
password=Bk@123456
```

备份命令

```bash
mkdir -p /home/octopus/backup/mysql
cat /home/octopus/backup/mysql_backup.sh

#!/bin/bash
attime=`date +%Y%m%d%H%M`
backupFileName="/home/octopus/backup/mysql/backup_$attime.sql"
backupTgzFileName="/home/octopus/backup/mysql/backup_$attime.sql.tar.gz"
echo "$backupFileName"
mysqldump --defaults-file="/etc/my_dump.cnf"  --databases octopusscheduler  > $backupFileName
tar czvf $backupTgzFileName  $backupFileName
rm -f $backupFileName
exit 0
```

添加到cron里

```bash
#每天中午11:30执行数据库备份
crontab -l
30 11 * * * /home/octopus/backup/mysql_backup.sh
```



通过光盘安装

```bash
mount /dev/cdrom  /mnt
vi /etc/yum.repos.d/CentOS-Media.repo
## 修改后的内容 
[c7-media]
name=CentOS-$releasever - Media
baseurl=file:///mnt/
gpgcheck=0
enabled=1
##

yum --disablerepo=\* --enablerepo=c7-media  install gcc
yum --disablerepo=\* --enablerepo=c7-media  install mariadb
yum --disablerepo=\* --enablerepo=c7-media  install mariadb-server

systemctl start mariadb
mysql_secure_installation
数据库用户和密码： root / ZD@123456

 mysql -uroot -h localhost -p
 
 
 
```

##### Docker安装

```bash
# 通过本地的docker获取mysql镜像
docker pull mysql:5.7
#  保存成镜像包
docker save -o mysql.tar mysql:5.7
# 启动
docker run -itd --name mysql-idp  -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql


# 进入容器
docker exec -it mysql bash


# 参考： https://www.runoob.com/docker/docker-install-mysql.html
```



##### 导出表结构和数据

```bash
# 导出整个数据库结构和数据
mysqldump -h localhost -uroot -p123456 database > dump.sql 

#导出单个数据表结构和数据
mysqldump -h localhost -uroot -p123456  database table > dump.sql

#导出整个数据库结构（不包含数据）
mysqldump -h localhost -uroot -p123456  -d database > dump.sql

#导出单个数据表结构（不包含数据）
mysqldump -h localhost -uroot -p123456  -d database table > dump.sql
```

