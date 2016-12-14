> [Jasing CAS 4.2.x](https://github.com/apereo/cas-overlay-template/tree/4.2) 使用MySQL进行用户认证.
>
> 官方文档[Database Authentication](https://apereo.github.io/cas/4.2.x/installation/Database-Authentication.html)

## 准备工作

参照[CAS 5.0.x Server 使用数据库MySQL进行用户认证](CAS 5.0.x Server 使用数据库MySQL进行用户认证.md)中准备工作部分。



## 添加依赖

```xml
<!--添加JDBC支持，自带c3p0连接池 -->
<dependency>
  <groupId>org.jasig.cas</groupId>
  <artifactId>cas-server-support-jdbc</artifactId>
  <version>${cas.version}</version>
</dependency>
<!--添加mysql数据库驱动-->
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.25</version>
</dependency>
```
## 连接池配置

- 在cas/WEB-INF/deployerConfigContext.xml中添加dataSource

```xml
<bean id="dataSource"
  class="com.mchange.v2.c3p0.ComboPooledDataSource"
  p:driverClass="${database.driverClass}"
  p:jdbcUrl="${database.url}"
  p:user="${database.user}"
  p:password="${database.password}"
  p:initialPoolSize="${database.pool.minSize}"
  p:minPoolSize="${database.pool.minSize}"
  p:maxPoolSize="${database.pool.maxSize}"
  p:maxIdleTimeExcessConnections="${database.pool.maxIdleTime}"
  p:checkoutTimeout="${database.pool.maxWait}"
  p:acquireIncrement="${database.pool.acquireIncrement}"
  p:acquireRetryAttempts="${database.pool.acquireRetryAttempts}"
  p:acquireRetryDelay="${database.pool.acquireRetryDelay}"
  p:idleConnectionTestPeriod="${database.pool.idleConnectionTestPeriod}"
  p:preferredTestQuery="${database.pool.connectionHealthQuery}" />
```

```xml
<alias name="queryDatabaseAuthenticationHandler" alias="primaryAuthenticationHandler" />
<alias name="dataSource" alias="queryDatabaseDataSource" />
```

注释掉

```xml
<alias name="acceptUsersAuthenticationHandler" alias="primaryAuthenticationHandler" />
```

- 修改/etc/cas/cas.properties

```properties
# == Basic database connection pool configuration ==
database.driverClass=com.mysql.jdbc.Driver
database.url=jdbc:mysql://localhost:3306/cas?ssl=true
database.user=somebody
database.password=meaningless
database.pool.minSize=6
database.pool.maxSize=18

# Maximum amount of time to wait in ms for a connection to become
# available when the pool is exhausted
database.pool.maxWait=10000

# Amount of time in seconds after which idle connections
# in excess of minimum size are pruned.
database.pool.maxIdleTime=120

# Number of connections to obtain on pool exhaustion condition.
# The maximum pool size is always respected when acquiring
# new connections.
database.pool.acquireIncrement=6

# == Connection testing settings ==

# Period in s at which a health query will be issued on idle
# connections to determine connection liveliness.
database.pool.idleConnectionTestPeriod=30

# Query executed periodically to test health
database.pool.connectionHealthQuery=select 1

# == Database recovery settings ==

# Number of times to retry acquiring a _new_ connection
# when an error is encountered during acquisition.
database.pool.acquireRetryAttempts=5

# Amount of time in ms to wait between successive aquire retry attempts.
database.pool.acquireRetryDelay=2000

cas.jdbc.authn.query.sql=select password from app_user where username=?
```
重新启动容器，不报错即可使用数据库中数据进行认证。