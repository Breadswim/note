# MyBatis分页

> 并非原创，如需删除请[email](zhoujinglong@hotmail.com)告知。

## 逻辑分页

### 介绍

> 逻辑分页是指用SQL把数据库中数据都查出来，通过ResultSet结果集筛选所要分页数据，好处是所有数据库都一致，坏处就是效率低。

### MyBatis实现

只要在查询的Mapper层添加RowBound对象即可

```java
List<Item> getItems(RowBounds bounds);
```

调用

```java
ItemMapper.getItems(new RowBounds(0, 10));
```

## 物理分页

> 分页拦截器，用于拦截需要进行分页查询的操作，然后对其进行分页处理。 利用拦截器实现Mybatis分页的原理：
>
> 要利用JDBC对数据库进行操作就必须要有一个对应的Statement对象，Mybatis在执行Sql语句前就会产生一个包含Sql语句的Statement对象，而且对应的Sql语句是在Statement之前产生的，所以我们就可以在它生成Statement之前对用来生成Statement的Sql语句下手。
>
> 在Mybatis中Statement语句是通过RoutingStatementHandler对象的prepare方法生成的。所以利用拦截器实现Mybatis分页的一个思路就是拦截StatementHandler接口的prepare方法，然后在拦截器方法中把Sql语句改成对应的分页查询Sql语句，之后再调用StatementHandler对象的prepare方法，即调用invocation.proceed()。
>
> 对于分页而言，在拦截器里面我们还需要做的一个操作就是统计满足当前条件的记录一共有多少，这是通过获取到了原始的Sql语句后，把它改为对应的统计语句再利用Mybatis封装好的参数和设置参数的功能把Sql语句中的参数进行替换，之后再执行查询记录数的Sql语句进行总记录数的统计。

### 分页信息对象

```java
public class Pagination<T> {
    private int start = 1;// 页码，默认是第一页
    private int pageSize = 15;// 每页显示的记录数，默认是15
    private int totalRecord;// 总记录数
    private int totalPage;// 总页数
    private List<T> results;// 对应的当前页记录
    private Map<String, Object> params = new HashMap<String, Object>();// 其他的参数我们把它分装成一个Map对象

    public void putParam(String key, Object value) {
        params.put(key, value);
    }
  
    public void setTotalRecord(int totalRecord) {
        this.totalRecord = totalRecord;
        int totalPage = totalRecord % pageSize == 0 ? totalRecord / pageSize :  totalRecord / pageSize + 1;
        this.setTotalPage(totalPage);
    }
	//省略其它getter/setter方法
   
}
```

### 添加MyBatis依赖

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.1.1</version>
</dependency>
```



### 添加StatementInterceptor

```java
@Intercepts({@Signature(method = "prepare", type = StatementHandler.class, args = {Connection.class})})
public class PaginationStatementInterceptor implements Interceptor {
    private String databaseType;// 数据库类型，不同的数据库有不同的分页方法

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 对于StatementHandler其实只有两个实现类，一个是RoutingStatementHandler，另一个是抽象类BaseStatementHandler，
        // BaseStatementHandler有三个子类，分别是SimpleStatementHandler，PreparedStatementHandler和CallableStatementHandler，
        // SimpleStatementHandler是用于处理Statement的，PreparedStatementHandler是处理PreparedStatement的，而CallableStatementHandler是
        // 处理CallableStatement的。Mybatis在进行Sql语句处理的时候都是建立的RoutingStatementHandler，而在RoutingStatementHandler里面拥有一个
        // StatementHandler类型的delegate属性，RoutingStatementHandler会依据Statement的不同建立对应的BaseStatementHandler，即SimpleStatementHandler、
        // PreparedStatementHandler或CallableStatementHandler，在RoutingStatementHandler里面所有StatementHandler接口方法的实现都是调用的delegate对应的方法。
        // 我们在PageInterceptor类上已经用@Signature标记了该Interceptor只拦截StatementHandler接口的prepare方法，又因为Mybatis只有在建立RoutingStatementHandler的时候
        // 是通过Interceptor的plugin方法进行包裹的，所以我们这里拦截到的目标对象肯定是RoutingStatementHandler对象。
        RoutingStatementHandler handler = (RoutingStatementHandler) invocation.getTarget();
        // 通过反射获取到当前RoutingStatementHandler对象的delegate属性
        StatementHandler delegate = (StatementHandler) ReflectUtil.getFieldValue(handler, "delegate");
        // 获取到当前StatementHandler的
        // boundSql，这里不管是调用handler.getBoundSql()还是直接调用delegate.getBoundSql()结果是一样的，因为之前已经说过了
        // RoutingStatementHandler实现的所有StatementHandler接口方法里面都是调用的delegate对应的方法。
        BoundSql boundSql = delegate.getBoundSql();
        // 拿到当前绑定Sql的参数对象，就是我们在调用对应的Mapper映射语句时所传入的参数对象
        Object obj = boundSql.getParameterObject();
        // 这里我们简单的通过传入的是Page对象就认定它是需要进行分页操作的。
        if (obj instanceof Pagination<?>) {
            Pagination<?> page = (Pagination<?>) obj;
            // 通过反射获取delegate父类BaseStatementHandler的mappedStatement属性
            MappedStatement mappedStatement = (MappedStatement) ReflectUtil.getFieldValue(delegate, "mappedStatement");
            // 拦截到的prepare方法参数是一个Connection对象
            Connection connection = (Connection) invocation.getArgs()[0];
            // 获取当前要执行的Sql语句，也就是我们直接在Mapper映射语句中写的Sql语句
            String sql = boundSql.getSql();
            // 给当前的page参数对象设置总记录数
            this.setTotalRecord(page, mappedStatement, connection);
            // 获取分页Sql语句
            String pageSql = this.getPageSql(page, sql);
            // 利用反射设置当前BoundSql对应的sql属性为我们建立好的分页Sql语句
            ReflectUtil.setFieldValue(boundSql, "sql", pageSql);
        }

        return invocation.proceed();
    }

    /**
     * 拦截器对应的封装原始对象的方法
     */
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    /**
     * 设置注册拦截器时设定的属性
     * 该方法，会在配置文件加载前执行
     */
    public void setProperties(Properties properties) {
        this.databaseType = properties.getProperty("databaseType") == null ? "oracle" : properties.getProperty("databaseType");
    }

    /**
     * 根据page对象获取对应的分页查询Sql语句，这里只做了两种数据库类型，Mysql和Oracle 其它的数据库都 没有进行分页
     *
     * @param page 分页对象
     * @param sql  原sql语句
     * @return
     */
    private String getPageSql(Pagination<?> page, String sql) {
        StringBuffer sqlBuffer = new StringBuffer(sql);

        if ("mysql".equalsIgnoreCase(databaseType)) {
            return getMysqlPageSql(page, sqlBuffer);
        } else if ("oracle".equalsIgnoreCase(databaseType)) {
            return getOraclePageSql(page, sqlBuffer);
        }
        return sqlBuffer.toString();
    }

    /**
     * 获取Mysql数据库的分页查询语句
     *
     * @param page      分页对象
     * @param sqlBuffer 包含原sql语句的StringBuffer对象
     * @return Mysql数据库分页语句
     */
    private String getMysqlPageSql(Pagination<?> page, StringBuffer sqlBuffer) {
        // 计算第一条记录的位置，Mysql中记录的位置是从0开始的。
        int offset = (page.getStart() - 1) * page.getPageSize();
        sqlBuffer.append(" limit ").append(offset).append(",").append(page.getPageSize());
        return sqlBuffer.toString();
    }

    /**
     * 获取Oracle数据库的分页查询语句
     *
     * @param page      分页对象
     * @param sqlBuffer 包含原sql语句的StringBuffer对象
     * @return Oracle数据库的分页查询语句
     */
    private String getOraclePageSql(Pagination<?> page, StringBuffer sqlBuffer) {
        // 计算第一条记录的位置，Oracle分页是通过rownum进行的，而rownum是从1开始的
        int offset = (page.getStart() - 1) * page.getPageSize() + 1;
        sqlBuffer.insert(0, "select u.*, rownum r from (").append(") u where rownum < ")
                .append(offset + page.getPageSize());
        sqlBuffer.insert(0, "select * from (").append(") where r >= ").append(offset);
        // 上面的Sql语句拼接之后大概是这个样子：
        // select * from (select u.*, rownum r from (select * from t_user) u
        // where rownum < 31) where r >= 16
        return sqlBuffer.toString();
    }

    /**
     * 给当前的参数对象page设置总记录数
     *
     * @param page            Mapper映射语句对应的参数对象
     * @param mappedStatement Mapper映射语句
     * @param connection      当前的数据库连接
     */
    private void setTotalRecord(Pagination<?> page, MappedStatement mappedStatement, Connection connection) {
        // 获取对应的BoundSql，这个BoundSql其实跟我们利用StatementHandler获取到的BoundSql是同一个对象。
        // delegate里面的boundSql也是通过mappedStatement.getBoundSql(paramObj)方法获取到的。
        BoundSql boundSql = mappedStatement.getBoundSql(page);
        // 获取到我们自己写在Mapper映射语句中对应的Sql语句
        String sql = boundSql.getSql();
        // 通过查询Sql语句获取到对应的计算总记录数的sql语句
        String countSql = this.getCountSql(sql);
        // 通过BoundSql获取对应的参数映射
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        // 利用Configuration、查询记录数的Sql语句countSql、参数映射关系parameterMappings和参数对象page建立查询记录数对应的BoundSql对象。
        BoundSql countBoundSql = new BoundSql(mappedStatement.getConfiguration(), countSql, parameterMappings, page);
        // 通过mappedStatement、参数对象page和BoundSql对象countBoundSql建立一个用于设定参数的ParameterHandler对象
        ParameterHandler parameterHandler = new DefaultParameterHandler(mappedStatement, page, countBoundSql);
        // 通过connection建立一个countSql对应的PreparedStatement对象。
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            pstmt = connection.prepareStatement(countSql);
            // 通过parameterHandler给PreparedStatement对象设置参数
            parameterHandler.setParameters(pstmt);
            // 之后就是执行获取总记录数的Sql语句和获取结果了。
            rs = pstmt.executeQuery();
            if (rs.next()) {
                int totalRecord = rs.getInt(1);
                // 给当前的参数page对象设置总记录数
                page.setTotalRecord(totalRecord);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            try {
                if (rs != null)
                    rs.close();
                if (pstmt != null)
                    pstmt.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 根据原Sql语句获取对应的查询总记录数的Sql语句
     *
     * @param sql
     * @return
     */
    private String getCountSql(String sql) {
        int index = sql.toUpperCase().indexOf("FROM");

        return "select count(*) " + sql.substring(index);
    }

    /**
     * 利用反射进行操作的一个工具类
     */
    private static class ReflectUtil {
        /**
         * 利用反射获取指定对象的指定属性
         *
         * @param obj       目标对象
         * @param fieldName 目标属性
         * @return 目标属性的值
         */
        public static Object getFieldValue(Object obj, String fieldName) {
            Object result = null;
            Field field = ReflectUtil.getField(obj, fieldName);
            if (field != null) {
                field.setAccessible(true);
                try {
                    result = field.get(obj);
                } catch (IllegalArgumentException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
            return result;
        }

        /**
         * 利用反射获取指定对象里面的指定属性
         *
         * @param obj       目标对象
         * @param fieldName 目标属性
         * @return 目标字段
         */
        private static Field getField(Object obj, String fieldName) {
            Field field = null;
            for (Class<?> clazz = obj.getClass(); clazz != Object.class; clazz = clazz.getSuperclass()) {
                try {
                    field = clazz.getDeclaredField(fieldName);
                    break;
                } catch (NoSuchFieldException e) {
                    // 这里不用做处理，子类没有该字段可能对应的父类有，都没有就返回null。
                }
            }
            return field;
        }

        /**
         * 利用反射设置指定对象的指定属性为指定的值
         *
         * @param obj        目标对象
         * @param fieldName  目标属性
         * @param fieldValue 目标值
         */
        public static void setFieldValue(Object obj, String fieldName, String fieldValue) {
            Field field = ReflectUtil.getField(obj, fieldName);
            if (field != null) {
                try {
                    field.setAccessible(true);
                    field.set(obj, fieldValue);
                } catch (IllegalArgumentException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

### *Mapper.xml

```xml
<select id="selectByAdvisorId" parameterType="Pagination" resultMap="BaseResultMap">
    select * from IS_MESSAGE where ADVISOR_ID = #{params.advisorId} ORDER BY LAST_MOD DESC
</select>
```

### MyBatis配置文件

```xml
<plugins>
    <plugin interceptor="com.PaginationStatementInterceptor">
        <property name="databaseType" value="oracle" />
    </plugin>
</plugins>
```
### 测试

```java
@Test
public void selectByAdvisorId() throws Exception {
    Pagination<IsMessage> page = new Pagination<IsMessage>();
    page.setPageSize(6);
    //查询条件
    page.putParam("advisorId", 1);
    List<IsMessage> list = messageService.selectByAdvisorId(page);

    Assert.assertEquals(6, list.size());
  	//对ResultSet拦截，把结果放到Pagination对象中，代码继续向下看
    //Assert.assertEquals(6, page.getResults().size());
}
```



### 附

到这里已经可以实现分页了，但我想把查询出的结果放到Pagination中，根据上面的思路想到对ResultSet拦截

```java
@Intercepts({@Signature(method = "handleResultSets", type = ResultSetHandler.class, args = {Statement.class})})
public class PaginationResultSetInterceptor  implements Interceptor{
    protected static Logger logger = Logger.getLogger(PaginationResultSetInterceptor.class);
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        if (!(invocation.getTarget() instanceof FastResultSetHandler)) {
            return invocation.proceed();
        }

        try {
            FastResultSetHandler resultSetHandler = (FastResultSetHandler) invocation.getTarget();
            MetaObject metaResultSetHandler = MetaObject.forObject(resultSetHandler, new DefaultObjectFactory(), new DefaultObjectWrapperFactory());
            ParameterHandler parameterHandler = (ParameterHandler)metaResultSetHandler.getValue("parameterHandler");

            Object parameterObject = parameterHandler.getParameterObject();
            if (parameterObject instanceof Pagination<?>) {
                Pagination<?> page = (Pagination<?>) parameterObject;
              	//拿到结果后set给Pagination，把结果reutrn.
                Object result = invocation.proceed();
                page.setResults((List)result);
                return result;
            }
        } catch (Exception e) {
            e.printStackTrace();
            logger.error(e.getMessage(), e);
        }

        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}

```

