# MyBatis Insert方法使用主键自增长

## Oracle

### *Mapper.xml

```xml
<insert id="insert" parameterType="AcTitle">
    <selectKey resultType="Decimal" keyProperty="titleId" order="BEFORE">
        SELECT SEQ_AC_TITLE.NEXTVAL AS ID FROM DUAL
    </selectKey>
    insert into AC_TITLE (TITLE_ID, TITLE_NAME, LOCATION,URL, TARGET, SERVICE_NAME,SORT, LAST_MOD, MOD_BY_ID)
    values (#{titleId,jdbcType=DECIMAL}, #{titleName,jdbcType=VARCHAR}, #{location,jdbcType=VARCHAR},
    #{url,jdbcType=VARCHAR}, #{target,jdbcType=VARCHAR}, #{serviceName,jdbcType=VARCHAR},
    #{sort,jdbcType=DECIMAL}, #{lastMod,jdbcType=TIMESTAMP}, #{modById,jdbcType=DECIMAL}
    )
</insert>
```

添加`<selectKey>`用于生成主键，

keyProperty：将查询到主键值设置到parameterType指定的对象的哪个属性， 需要和`insert`中的属性名一致。

order：`<selectKey> `执行顺序，相对于insert语句来说它的执行顺序     

resultType：指定`<selectKey>`的结果类型,同样需要和`insert`中的属性类型一致



