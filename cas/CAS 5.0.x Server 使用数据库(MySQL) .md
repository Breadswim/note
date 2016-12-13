# CAS 5.0.x Server 使用数据库(MySQL)

> Jasing CAS 5.0.x配置有较大变化，结合官方文档做个简单的配置。

## 准备工作

- 使用JDK的keytool命令生成数字证书

`keytool -genkey -keystore "localhost.keystore" -alias localhost -keyalg RSA`

根据提示补全信息，所有的密码这里都使用`changeit`，名字与姓氏使用`localhost`

通过如上步骤，生成证书 localhost.keystore到当前目录。

- 复制 localhost.keystore 到/etc/cas并重命名为thekeystore

`sudo cp localhost.keystore /etc/cas/thekeystore`

- 设置tomcat下的server.xml

打开conf/server.xml，找到

```xml
<!--
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150" SSLEnabled="true">
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="conf/localhost-rsa.jks"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
-->
```


替换为

```xml
<Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
   maxThreads="150" scheme="https" secure="true"
   clientAuth="false" sslProtocol="TLS"
   keystoreFile="/etc/cas/thekeystore" keystorePass="changeit"/>
```


可以使用 [https://localhost:8443](https://localhost:8443) 访问Tomcat。

![](https://cdn.rawgit.com/Evilspirits/note/master/cas/img/Screen Shot 2016-12-13 at 16.09.41.png)



## 配置CAS服务器端

- 我使用的Jasing CAS服务器，可以到其官方的github[下载](https://github.com/apereo/cas-overlay-template.git)

- 在cas-overlay-template下运行build.sh build会生成target/cas.war，把cas.war复制到tomcat/webapp目录，启动tomcat. 访问[https://localhost:8443/cas/login](https://localhost:8443/cas/login) 使用默认`casuser/Mellon`登录。

  ![Screen Shot 2016-12-13 at 15.34.54](https://cdn.rawgit.com/Evilspirits/note/master/cas/img/Screen Shot 2016-12-13 at 15.34.54.png)

  ​

- 修改POM.xml 

  ```xml
  <!--通过在WAR覆盖中包含以下依赖关系来启用数据库认证-->
  <dependency>
  	<groupId>org.apereo.cas</groupId>
  	<artifactId>cas-server-support-jdbc</artifactId>
  	<version>${cas.version}</version>
  </dependency>
  <dependency>
  	<groupId>org.apereo.cas</groupId>
  	<artifactId>cas-server-support-jdbc-drivers</artifactId>
  	<version>${cas.version}</version>
  </dependency>
  ```

  jdbc drivers自动支持以下数据库，其它的需要手动添加

  1. HSQLDB
  2. MySQL
  3. PostgreSQL
  4. MariaDB
  5. Microsoft SQL Server (JTDS)
  6. Sybase

- sudo ./build.sh copy复制配置文件到/etc/cas，打开/etc/cas/config/cas.properties，使用以下配置

  ```properties
  cas.server.name: https://localhost:8443
  cas.server.prefix: https://localhost:8443/cas

  cas.adminPagesSecurity.ip=127\.0\.0\.1

  logging.config: file:/etc/cas/config/log4j2.xml
  #登录认证SQL
  cas.authn.jdbc.query[0].sql=select password from app_user where username=?
  #用于判断数据库是否可用
  cas.authn.jdbc.query[0].healthQuery=SELECT 1
  #数据库连接配置
  cas.authn.jdbc.query[0].url=jdbc:mysql://localhost:3306/cas?ssl=true
  cas.authn.jdbc.query[0].user=root
  cas.authn.jdbc.query[0].password=root
  cas.authn.jdbc.query[0].driverClass=com.mysql.jdbc.Driver
  #记住我
  cas.ticket.tgt.rememberMe.enabled=true
  cas.ticket.tgt.rememberMe.timeToKillInSeconds=28800
  #不再使用静态的认证
  cas.authn.accept.users=
  ```

- 初始化数据库MySQL

  ```mysql
  create database shiro;
  CREATE TABLE app_user (
    username varchar(200) NOT NULL,
    password varchar(512) DEFAULT NULL,
    PRIMARY KEY (username),
    UNIQUE KEY loginUser_username_uindex (username)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  INSERT INTO cas.app_user (username, password) VALUES ('admin', '123');
  INSERT INTO cas.app_user (username, password) VALUES ('test', '123');
  ```

- 重新编译运行cas-overlay-template，使用`admin/123`登录[https://localhost:8443/cas/login](https://localhost:8443/cas/login)。

  ![Screen Shot 2016-12-13 at 16.04.37](https://cdn.rawgit.com/Evilspirits/note/master/cas/img/Screen Shot 2016-12-13 at 16.04.37.png)

  ![Screen Shot 2016-12-13 at 16.04.58](https://cdn.rawgit.com/Evilspirits/note/master/cas/img/Screen Shot 2016-12-13 at 16.04.58.png)

  ​









