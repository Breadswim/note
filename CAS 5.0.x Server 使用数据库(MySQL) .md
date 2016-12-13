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


可以使用 https://localhost:8443/ 访问Tomcat。

![Screen Shot 2016-12-13 at 16.09.41](/Users/zhoujl/Desktop/Screen Shot 2016-12-13 at 16.09.41.png)





