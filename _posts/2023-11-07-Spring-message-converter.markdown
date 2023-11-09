---
description: Springboot MessageConverter 消息转换使用
---

# MessageConverter

web 开发中经常遇到请求体、响应体的格式转换，Spring Web 为我们提供 MessageConverter 消息转换器实现 java 数据类型、json/xml、自定协议类型的转化。

流程：

1. Controller 内部方法上添加注解 @ResponseBody 或者请求参数上注解 @RequestBody
2. WebMvcConfigurationSupport#**extendMessageConverters** 方法设置自定义 MessageConverters 或者使用 Bean 对象设置消息转换
3. 当处理请求或者响应时执行 AbstractMessageConverterMethodArgumentResolver#**readWithMessageConverters** 或者 AbstractMessageConverterMethodProcessor#**writeWithMessageConverter**，两个方法会遍历存在的 MessageConverter，并判断是否执行内部方法
4. 获得 MessageConverter 消息转换器，检验 support 类型，如果通过则进行内容转化
5. 获得请求对象或者转化对象为指定响应格式

## 接口介绍

继承 AbstractHttpMessageConverter 类实现对输入输出请求体控制，下面介绍待实现的方法

```
public abstract class AbstractHttpMessageConverter<T> implements HttpMessageConverter<T> {
    // 判断支持的类型，传入 class 类型作为判断依据
    protected abstract boolean supports(Class<?> clazz);
​
    // 满足读请求条件，执行实际请求读取逻辑
    protected abstract T readInternal(Class<? extends T> clazz, HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException;
​
    // 满足写响应条件，执行实际响应写入逻辑
    protected abstract void writeInternal(T t, HttpOutputMessage outputMessage)
            throws IOException, HttpMessageNotWritableException;
}
```

必须在 Controller 中存在 @ResponseBody 或者 @RequestBody 注解才能执行 ，否则将跳过 HttpMessageConverter 逻辑，比如只注解 @ResponseBody，则 readInternal 方法不执行。这是由于 Spring web 通过 AOP 监听到对应注解才会执行接下来的逻辑。

## 使用案例

### Controller 类

controller 类作为请求的入口和响应的出口，需特别注意必须加上对应注解才能启动 MessageConverters 执行消息转换，其中@RestController 包含了 @ResponseBody 注解。

```
@RestController
@RequestMapping
public class HelloController {
​
    @RequestMapping("/user")
    public User user(@RequestBody User user) {
        System.out.println(user.toString());
        return new User("1", "1");
    }
}
```

User 对象

```
public class User {
​
    private String id;
    private String name;
​
    public User() {
    }
​
    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }
​
    public String getId() {
        return id;
    }
​
    public void setId(String id) {
        this.id = id;
    }
​
    public String getName() {
        return name;
    }
​
    public void setName(String name) {
        this.name = name;
    }
​
    @Override
    public String toString() {
        return "User{" +
            "id='" + id + '\'' +
            ", name='" + name + '\'' +
            '}';
    }
}
```

### 自定义 MessageConverter

自定义 MessageConverter，先定义支持的类型和转化条件，再设置请求 body 和响应 body 的转化逻辑

```
public class HttpMessageConvertor extends AbstractHttpMessageConverter<User> {
​
    private Gson gson;
​
    public HttpMessageConvertor(Gson gson) {
        this.gson = gson;
        setSupportedMediaTypes(new ArrayList<>(Collections.singletonList(MediaType.APPLICATION_JSON)));
    }
​
    @Override
    protected boolean supports(Class<?> clazz) {
        return clazz == User.class;
    }
​
    @Override
    protected User readInternal(Class<? extends User> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        MediaType mediaType = inputMessage.getHeaders().getContentType();
        if (mediaType != null && mediaType.includes(MediaType.APPLICATION_JSON)) {
            return gson.fromJson(new InputStreamReader(inputMessage.getBody()), User.class);
        }
        return new User("", "");
    }
​
    @Override
    protected void writeInternal(User user, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        user.setName("3");
        outputMessage.getBody().write(gson.toJson(user).getBytes(StandardCharsets.UTF_8));
    }
}
```

### MessageConverter 注入

通过 Bean 的形式注入，也可以通过重写 extendMessageConverters 方法设置（特别是存在冲突 MessageConverter 时）

```
@Configuration
public class Configuration {
​
    @Bean
    public Gson getGson() {
        return new Gson();
    }
​
    @Bean
    public HttpMessageConvertor webMvcConfigurer() {
        return new HttpMessageConvertor(getGson());
    }
}
```

## 注意

自定义 MessageConverter 不要与其他启动的 MessageConverter 所支持 contentType 冲突，会导致自定义 MessageConverter 被跳过，如果出现冲突可以通过 webmvcconfig 手动调整顺序或者覆盖。默认开启的 MessageConverter 在可在 WebMvcConfigurationSupport#addDefaultHttpMessageConverters 方法中查看。
