# MyBatis多表关联查询

> myBatis文件使用Mybatis Generator生成。



## 一对多

> AcTitle中包含多个AcTitleChild



### 在Module.java中添加Collection

```java
public class AcTitle {
    private BigDecimal titleId;

    private String titleName;

    private String location;

	//新添加的Collection用于关联查询
    private List<AcTitleChild> acTitleChilds = new ArrayList<AcTitleChild>();
  
  	/**getter/setter function*/
}
```



### Mapper.xml

```xml
<resultMap id="BaseResultMap" type="com.demo.AcTitle">
    <constructor>
        <idArg column="TITLE_ID" javaType="java.math.BigDecimal" jdbcType="DECIMAL"/>
        <arg column="TITLE_NAME" javaType="java.lang.String" jdbcType="VARCHAR"/>
    </constructor>
</resultMap>
<!-- 一对多关联 -->
<resultMap id="WithTitleChildResultMap" type="AcTitle" extends="BaseResultMap">
    <collection property="acTitleChilds" javaType="ArrayList" ofType="AcTitleChild"
                resultMap="com.demo.AcTitleChildMapper.BaseResultMap"/>
</resultMap>

<select id="selectAcTitlesByRoleId" parameterType="Integer" resultMap="WithTitleChildResultMap">
  SELECT acTitle.*,acTitleChild.*
  FROM AC_TITLE acTitle
  INNER JOIN AC_TITLE_CHILD acTitleChild ON acTitleChild.TITLE_ID = acTitle.TITLE_ID
  ...        
</select>
```

添加一个ReultMap专门做一对多关联查询，继承自`BseResultMap`, 添加`collection`映射`acTitleChilds`属性，和AcTitle.java中一致。

