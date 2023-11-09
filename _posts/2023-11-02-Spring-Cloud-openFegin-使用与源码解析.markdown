---
description: Spring Cloud openFegin 使用与源码学习
---

# Spring Cloud openFegin

实现 Http 请求通常需要取得 HttpClient 并设置请求的头信息和请求体，还要获取请求的 url，特别是在微服务环境下通常包含负载均衡、熔断机制，整个请求编写过程会显得很繁琐。

Feign 组件作为一款 HttpClient 绑定器，在 Spring Cloud 微服务调用中提供微服务间的声明式调用，内部集成负载均衡和断路器组件，通过注解的方式声明服务调用，非常易用。

## 使用案例 <a href="#wxafq" id="wxafq"></a>

1、引入依赖，feign 是 Spring Cloud 默认支持的组件，在父项目依赖中声明版本为 2021.0.4，子项目可简单依赖，feign 对应版本为 3.1.4

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2、定义 feign 接口，通过 @FeignClient 来实现要调用的微服务接口

```
@FeignClient(name = "user-service", fallback = UserServiceClient.UserServiceFallback.class)
public interface UserServiceClient {

    @GetMapping(value = "/user/getUser")
    String print();

    class UserServiceFallback implements UserServiceClient {
        @Override
        public String print() {
            return "Service User failed! Trigger Fallback";
        }
    }
}
```

3、配置 feign 参数，设置默认全局连接超时时间

```
server:
  port: 9011 # 端口
spring:
  application:
    name: find-service # 服务名称
  cloud:
    nacos:
      server-addr: localhost:8848 # nacos地址
    loadbalancer:
      nacos:
        enabled: true


feign:
  client:
    config:
      default:
        connectionTimeout: 5000
        readTimeout: 5000
```

default 表示默认全局，支持按照服务名分别设置参数

## 工作原理 <a href="#gpgnv" id="gpgnv"></a>

### 支持配置 <a href="#rquii" id="rquii"></a>

支持的配置详情在 FeignClientProperties.FeignClientConfiguration 中定义

```
public static class FeignClientConfiguration {

        // 打印日志级别
        // 支持 NONE：不打印日志
        //     BASIC：只打印请求、url、响应状态码和执行时间
        //     HEADERS：打印基本信息，包含请求和响应头
        //     FULL：打印头、body、请求和响应的元数据
		private Logger.Level loggerLevel;

        // 连接超时时间
		private Integer connectTimeout;

        // 读信息超时时间
		private Integer readTimeout;

        //  重试策略
		private Class<Retryer> retryer;

        // 非2xx响应码时的反编码策略
		private Class<ErrorDecoder> errorDecoder;

        // 请求拦截器
		private List<Class<RequestInterceptor>> requestInterceptors;

        // 默认请求头
		private Map<String, Collection<String>> defaultRequestHeaders;

        // 默认请求参数
		private Map<String, Collection<String>> defaultQueryParameters;

        // 反编码 404 支持
		private Boolean decode404;

        // 反编码逻辑
		private Class<Decoder> decoder;

        // 编码逻辑
		private Class<Encoder> encoder;

        // 定义元数据的注解和值的验证
		private Class<Contract> contract;

        // 发生异常时的策略，默认抛出 RetryableException 异常
        // 开启 UNWRAP 后抛出 RetryableException 包装前的异常
		private ExceptionPropagationPolicy exceptionPropagationPolicy;

        // 定制 feign 部分核心功能，详情见 Capability 接口
		private List<Class<Capability>> capabilities;

        // 请求参数编码逻辑，支持单个请求参数编码
		private Class<QueryMapEncoder> queryMapEncoder;

        // 统计功能支持，默认 true
		private MetricsProperties metrics;

        // 重定向支持，默认 true
		private Boolean followRedirects;

        ......
}
```

默认配置可查看 FeignClientFactoryBean#configureUsingProperties

官方文档中给出全配置项如下

```
feign:
    client:
        config:
            feignName:
                connectTimeout: 5000
                readTimeout: 5000
                loggerLevel: full
                errorDecoder: com.example.SimpleErrorDecoder
                retryer: com.example.SimpleRetryer
                defaultQueryParameters:
                    query: queryValue
                defaultRequestHeaders:
                    header: headerValue
                requestInterceptors:
                    - com.example.FooRequestInterceptor
                    - com.example.BarRequestInterceptor
                decode404: false
                encoder: com.example.SimpleEncoder
                decoder: com.example.SimpleDecoder
                contract: com.example.SimpleContract
                capabilities:
                    - com.example.FooCapability
                    - com.example.BarCapability
                queryMapEncoder: com.example.SimpleQueryMapEncoder
                metrics.enabled: false
```

FeignClientConfiguration 相比官方给出配置项少了 capabilities 相关配置，该配置的作用是统计可视相关的定制化需求，默认提供 MicrometerCapability（需满足 [feign-metrics 条件](https://docs.spring.io/spring-cloud-openfeign/docs/3.1.8/reference/html/#feign-metrics)），如果开启 @EnableCaching 注解，CachingCapability 会注册到使用 @Cache\* 注解的 Fegin 客户端接口中。

Fegin 支持请求/响应压缩，如果考虑 [Fegin 请求/响应支持 GZIP 压缩](https://docs.spring.io/spring-cloud-openfeign/docs/3.1.8/reference/html/#feign-requestresponse-compression)，可以开启下面配置

```
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```

压缩支持通过 media 类型和请求长度过滤

```
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

注意 okHttpClient 的压缩逻辑需要自己通过 okHttpClient 的拦截器实现，可参考[RequestBodyCompression](https://github.com/square/okhttp/blob/7135628c645892faf1a48a8cff464e0ed4ad88cb/samples/guide/src/main/java/okhttp3/recipes/RequestBodyCompression.java)。

### @EnableFeignClients <a href="#xjgi8" id="xjgi8"></a>

项目启动类开启 @EnableFeignClients 注解后，组件开启 feign 客户端类扫描，自动装配 fegin 相关的功能

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {

    // 定义扫描包名
	String[] value() default {};

	// 同上
	String[] basePackages() default {};

	// 定义扫描类
	Class<?>[] basePackageClasses() default {};

	// 定制配置文件，参考 FeignClientsConfiguration
	Class<?>[] defaultConfiguration() default {};

	// 扫描的 @FeignClient 所在类
	Class<?>[] clients() default {};

}
```

可见注解有 @Import 注解，FeignClientsRegistrar 在包扫描前提前注入的 bean，方便在后续被调用使用。

```
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
    ......
    @Override
	public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}
    ......
}
```

FeignClientsRegistrar 类继承自 ImportBeanDefinitionRegistrar，ImportBeanDefinitionRegistrar 是 Spring 提供的动态刷新的扩展点，每次执行 Spring 重新加载 AbstractApplicationContext#refresh 方法时都会执行继承和实现 ImportBeanDefinitionRegistrar#registerBeanDefinitions 方法，达到动态更新的效果。

### @FeignClient <a href="#q6s0p" id="q6s0p"></a>

配置 @FeignClient 的类会被扫描到 FeignClientsRegistrar 扫描到

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface FeignClient {

	// 定义 service id
	@AliasFor("name")
	String value() default "";

    // 定义 bean 的名称，注意不是服务名
	String contextId() default "";

	// 定义 service id
	@AliasFor("value")
	String name() default "";

	/**
	 * @return the <code>@Qualifier</code> value for the feign client.
	 * @deprecated in favour of {@link #qualifiers()}.
	 *
	 * If both {@link #qualifier()} and {@link #qualifiers()} are present, we will use the
	 * latter, unless the array returned by {@link #qualifiers()} is empty or only
	 * contains <code>null</code> or whitespace values, in which case we'll fall back
	 * first to {@link #qualifier()} and, if that's also not present, to the default =
	 * <code>contextId + "FeignClient"</code>.
	 */
	@Deprecated
	String qualifier() default "";

	/**
	 * @return the <code>@Qualifiers</code> value for the feign client.
	 *
	 * If both {@link #qualifier()} and {@link #qualifiers()} are present, we will use the
	 * latter, unless the array returned by {@link #qualifiers()} is empty or only
	 * contains <code>null</code> or whitespace values, in which case we'll fall back
	 * first to {@link #qualifier()} and, if that's also not present, to the default =
	 * <code>contextId + "FeignClient"</code>.
	 */
	String[] qualifiers() default {};

	// 定义 url，如果定义则不过负载均衡
	String url() default "";

	// 开启 404 反编码
	boolean decode404() default false;

	// 自定义配置类
	Class<?>[] configuration() default {};

	// 自定义失败回退逻辑，必须可验证的 spring bean
	Class<?> fallback() default void.class;

	// 定义回滚工厂，必须可验证的 spring bean
	Class<?> fallbackFactory() default void.class;

	// 定义路径前缀
	String path() default "";

	// 将代理标记为主 bean
	boolean primary() default true;

}
```

其次是加载扫描到 @FeignClient 注解的逻辑，执行 registerDefaultConfiguration 和 registerFeignClients 方法，将扫描到 @FeignClient 的类通过 FeignClientFactoryBean 解析为 BeanDefinition 并注入 Spring 容器中，FeignClientFactoryBean 逻辑

```
public class FeignClientFactoryBean
		implements FactoryBean<Object>, InitializingBean, ApplicationContextAware, BeanFactoryAware {
            @Override
	public Object getObject() {
		return getTarget();
	}

	/**
	 * @param <T> the target type of the Feign client
	 * @return a {@link Feign} client created with the specified data and the context
	 * information
	 */
	<T> T getTarget() {
		FeignContext context = beanFactory != null ? beanFactory.getBean(FeignContext.class)
				: applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(url)) {

			if (LOG.isInfoEnabled()) {
				LOG.info("For '" + name + "' URL not provided. Will try picking an instance via load-balancing.");
			}
			if (!name.startsWith("http")) {
				url = "http://" + name;
			}
			else {
				url = name;
			}
			url += cleanPath();
			return (T) loadBalance(builder, context, new HardCodedTarget<>(type, name, url));
		}
		if (StringUtils.hasText(url) && !url.startsWith("http")) {
			url = "http://" + url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof FeignBlockingLoadBalancerClient) {
				// not load balancing because we have a url,
				// but Spring Cloud LoadBalancer is on the classpath, so unwrap
				client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
			}
			if (client instanceof RetryableFeignBlockingLoadBalancerClient) {
				// not load balancing because we have a url,
				// but Spring Cloud LoadBalancer is on the classpath, so unwrap
				client = ((RetryableFeignBlockingLoadBalancerClient) client).getDelegate();
			}
			builder.client(client);
		}

		applyBuildCustomizers(context, builder);

		Targeter targeter = get(context, Targeter.class);
		return (T) targeter.target(this, builder, context, new HardCodedTarget<>(type, name, url));
	}

    protected <T> T loadBalance(Feign.Builder builder, FeignContext context, HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			applyBuildCustomizers(context, builder);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-loadbalancer?");
	}
}
```

当 url 参数不存在时调用 loadBalance 逻辑通过注册中心的服务列表进行负载均衡获取当前调用的服务 url。

如果配置 url 则直接通过 url 地址访问接口，Spring Cloud openFeign 目前只能执行同步 http 请求，暂不支持异步 http 请求。

Feigin 支持的 Http 客户端有 okHttp、HttpClient、HttpClient5 以及内部 DefaultClient。如果要支持 ssl 认证等功能，比如 DefaultFeignLoadBalancerConfiguration，需自己实现注入 Client 时设置所需参数。

```
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(LoadBalancerClientsProperties.class)
class DefaultFeignLoadBalancerConfiguration {

	@Bean
	@ConditionalOnMissingBean
	@Conditional(OnRetryNotEnabledCondition.class)
	public Client feignClient(LoadBalancerClient loadBalancerClient,
			LoadBalancerClientFactory loadBalancerClientFactory) {
		return new FeignBlockingLoadBalancerClient(new Client.Default(null, null), loadBalancerClient,
				loadBalancerClientFactory);
	}

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
	@ConditionalOnBean(LoadBalancedRetryFactory.class)
	@ConditionalOnProperty(value = "spring.cloud.loadbalancer.retry.enabled", havingValue = "true",
			matchIfMissing = true)
	public Client feignRetryClient(LoadBalancerClient loadBalancerClient,
			LoadBalancedRetryFactory loadBalancedRetryFactory, LoadBalancerClientFactory loadBalancerClientFactory) {
		return new RetryableFeignBlockingLoadBalancerClient(new Client.Default(null, null), loadBalancerClient,
				loadBalancedRetryFactory, loadBalancerClientFactory);
	}

}
```

如果启动 loadbalance，fegin 在 FeignBlockingLoadBalancerClient 在自动注入阶段会连同 LoadBalancerClient、LoadBalancerClientFactory 依赖进行注入。并在 FeignClientFactoryBean#getOptional 方法执行时响应注入的 Client，实际为 FeignBlockingLoadBalancerClient。

LoadBalancer 策略只有两种轮询（默认）、随机，如果需要实现额外的负载均衡算法需继承 ReactorServiceInstanceLoadBalancer 自行定义，注入方式可参考 LoadBalancerClientConfiguration。

通过注入的 Targeter 类的注入断路器，见 FeignClientFactoryBean#loadBalance 中包含获取 Targeter 逻辑，这里加断路器到 client。对应 Configuration

```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(Feign.class)
@EnableConfigurationProperties({ FeignClientProperties.class, FeignHttpClientProperties.class,
		FeignEncoderProperties.class })
public class FeignAutoConfiguration {
        ......

		@Bean
		@ConditionalOnMissingBean
		@ConditionalOnBean(CircuitBreakerFactory.class)
		public Targeter circuitBreakerFeignTargeter(CircuitBreakerFactory circuitBreakerFactory,
				@Value("${feign.circuitbreaker.group.enabled:false}") boolean circuitBreakerGroupEnabled,
				CircuitBreakerNameResolver circuitBreakerNameResolver) {
			return new FeignCircuitBreakerTargeter(circuitBreakerFactory, circuitBreakerGroupEnabled,
					circuitBreakerNameResolver);
		}
        ......
}
```

ReflectiveFeign 使用动态代理的方式增强原本接口的方法构建 rpc 远程调用机制

```
public class ReflectiveFeign extends Feign {
    ......
    @Override
  public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if (Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
  ......
}
```

### 执行逻辑 <a href="#pic98" id="pic98"></a>

当 feign 客户端被调用时，先路由找到对应 FeignCircuitBreakerInvocationHandler 执行 invoke 方法（如果没开启 CircuitBreaker 则执行 SynchronousMethodHandler#invoke），如果执行失败则进行重试逻辑，如果重试失败则抛出异常，异常按照 ExceptionPropagationPolicy 策略决定是否拆还原包异常堆栈

```
final class FeignCircuitBreakerInvocationHandler implements MethodHandler {
    ......
    @Override
	public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwable {
		// early exit if the invoked method is from java.lang.Object
		// code is the same as ReflectiveFeign.FeignInvocationHandler
		if ("equals".equals(method.getName())) {
			try {
				Object otherHandler = args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
				return equals(otherHandler);
			}
			catch (IllegalArgumentException e) {
				return false;
			}
		}
		else if ("hashCode".equals(method.getName())) {
			return hashCode();
		}
		else if ("toString".equals(method.getName())) {
			return toString();
		}

		String circuitName = circuitBreakerNameResolver.resolveCircuitBreakerName(feignClientName, target, method);
		CircuitBreaker circuitBreaker = circuitBreakerGroupEnabled ? factory.create(circuitName, feignClientName)
				: factory.create(circuitName);
		Supplier<Object> supplier = asSupplier(method, args);
		if (this.nullableFallbackFactory != null) {
			Function<Throwable, Object> fallbackFunction = throwable -> {
				Object fallback = this.nullableFallbackFactory.create(throwable);
				try {
					return this.fallbackMethodMap.get(method).invoke(fallback, args);
				}
				catch (Exception exception) {
					unwrapAndRethrow(exception);
				}
				return null;
			};
			return circuitBreaker.run(supplier, fallbackFunction);
		}
		return circuitBreaker.run(supplier);
	}
    ......
}
```

## 使用技巧 <a href="#mavd2" id="mavd2"></a>

#### Fegin 远程调用缺少请求头 <a href="#fj0av" id="fj0av"></a>

feign 调用接口相当于起了一个新的请求进行，自身不带额外请求头

解决方法自定义 RequestInterceptor 拦截器

```
@Component
    public class FeignHeadInterceptor implements RequestInterceptor {
        @Override
        public void apply(RequestTemplate template) {
            ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (requestAttributes != null) {
                HttpServletRequest request = requestAttributes.getRequest();
                Enumeration<String> headers = request.getHeaderNames();
                if (headers != null) {
                    while (headers.hasMoreElements()) {
                        String name = headers.nextElement();
                        if ("extend-head".equals(name)) {
                            String value = request.getHeader(name);
                            template.header(name, value);
                        } else {
                            template.header("extend-head", "none");
                        }
                    }
                }
            }
        }
    }
```

### Fegin Client 方法存在并发情况导致无法正确设置请求头 <a href="#dojzx" id="dojzx"></a>

原因：RequestContextHolder.getRequestAttributes() 方法取自 ThreadLocal，不具备传递子线程的特性并发中会丢失。而 RequestContextHolder 中inheritableRequestAttributesHolder 虽然是 InheritableThreadLocal 可以传递给子线程，但是一旦主线程关闭就会回收，这时不能保证子线程已经处理完成，存在缺陷

解决方法：自定义 InheritableThreadLocal 管理 HttpServletRequest

1、自定义 InheritableThreadLocalUtil 管理 HttpServletRequest，这里为了方便仅简单处理，建议在请求访问接口时封装一套子类可继承的 RequestContext 逻辑

```
public class InheritableThreadLocalUtil {

    private static final InheritableThreadLocal<Map<String,Object>> headerMap = new InheritableThreadLocal<Map<String, Object>>(){
        @Override
        protected Map<String, Object> initialValue() {
            return new HashMap<>();
        }
    };

    public static Map<String,Object> get(){
        return headerMap.get();
    }

    public static Object get(String key) {
        return headerMap.get().get(key);
    }

    public static void set(String key, Object value){
        headerMap.get().put(key,value);
    }
}
```

2、并发 controller 改造

```
@RestController
@RequestMapping
public class FindController {

    @Autowired
    DiscoveryClient discoveryClient;

    @Autowired
    UserServiceClient serviceClient;

    @RequestMapping("/async")
    public String Async() {
        InheritableThreadLocalUtil.set("request", RequestContextHolder.getRequestAttributes());
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            RequestContextHolder.setRequestAttributes((RequestAttributes)InheritableThreadLocalUtil.get("request"));
            ServiceInstance serviceInstance = discoveryClient.getInstances("user-service").get(0);
            return serviceInstance.getServiceId() + " (" + serviceInstance.getHost() + ":" + serviceInstance.getPort() + ")" + "===> " + serviceClient.print();
        });
        String response = "";
        try {
            response = future.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        return response;
    }
}
```

## 参考 <a href="#tdvhu" id="tdvhu"></a>

官方文档 --- [https://docs.spring.io/spring-cloud-openfeign/docs/3.1.8/reference/html/#spring-cloud-feign-overriding-defaults](https://docs.spring.io/spring-cloud-openfeign/docs/3.1.8/reference/html/#spring-cloud-feign-overriding-defaults)
