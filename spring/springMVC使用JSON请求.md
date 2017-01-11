# springMVC处理JSON请求

## 添加jar包依赖

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.4.1</version>
</dependency>
<dependency>
    <groupId>org.codehaus.jackson</groupId>
    <artifactId>jackson-mapper-asl</artifactId>
    <version>1.9.13</version>
</dependency>
```

## Controller

```java
/**
 *请求是json格式，返回也是json格式
 */
@RequestMapping("/requestJson")
    public @ResponseBody ItemsCustom requestJsonTest(@RequestBody ItemsCustom itemsCustom, HttpServletRequest request) throws Exception {

        HttpSession session = request.getSession();
        //@ResponseBody将itemsCustom转成json输出
        return itemsCustom;
    }
/**
 *请求不是json格式，返回也是json格式
 */
@RequestMapping("/responseJson")
public @ResponseBody ItemsCustom responseJson(ItemsCustom itemsCustom){

    //@ResponseBody将itemsCustom转成json输出
    if (itemsCustom != null && itemsCustom.getPrice() != null) {
        itemsCustom.setPrice(itemsCustom.getPrice() +100);
    }
    return itemsCustom;
}
```

## 前台jsp 

```jsp
//请求json，输出是json
        function requestJson(){
            $.ajax({
                type:'post',
                url:'requestJson.action',
                contentType:'application/json;charset=utf-8',
                //数据格式是json串，商品信息
                data:'{"name":"手机","price":999}',
                success:function(data){//返回json结果
                    console.log(data)
                }
            });
        }
        //请求key/value，输出是json
        function responseJson(){
            $.ajax({
                type:'post',
                url:'${pageContext.request.contextPath }/responseJson.action',
                contentType:'application/x-www-form-urlencoded;charset=utf-8',
                //数据格式是json串，商品信息
                data:'name=手机&price=999',
                success:function(data){//返回json结果
                    console.log(data)
                }
            });
        }
```

##测试截图

![](img/jsonrequest.png)



![](img/jsonresponse.png)



## 如果Controller 返回的是String 类型，并且请求能到服务器，但是服务器返回的数据不能被ajax success方法接收，去掉ajax请求的参数 dataType:'json'可解决。

## Ajax 返回中文乱码问题, 使用StringHttpMessageConverter可解决。

```xml
<mvc:annotation-driven>
   <!-- utf-8编码 -->
   <mvc:message-converters register-defaults="true">
      <bean class="org.springframework.http.converter.StringHttpMessageConverter">
         <constructor-arg value="UTF-8" />
      </bean>
   </mvc:message-converters>
</mvc:annotation-driven>
```