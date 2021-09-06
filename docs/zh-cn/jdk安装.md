#### oracle jdk 1.8.0

下载和安装

```bash
# 下载jdk
# 创建目录
mkdir -p /usr/java
tar xvf jdk-8u281-linux-x64.tar
```

配置环境变量

```bash
#打开 /etc/profile 文件，在末尾添加下面内容
export JAVA_HOME=/usr/java/jdk1.8.0_281
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```

保存并运行

```bash
source /etc/profile

#测试一下
java -version
```

