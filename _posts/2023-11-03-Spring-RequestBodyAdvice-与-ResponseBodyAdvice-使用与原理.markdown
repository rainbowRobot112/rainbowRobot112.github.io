---
description: Springboot RequestBodyAdvice 与 ResponseBodyAdvice 监听请求/响应
---

# RequestBodyAdvice 与 ResponseBodyAdvice

SpringBoot 内部可自定义请求通知和响应通知，RequestBodyAdviceAdapter 和 ResponseBodyAdvice\<T> 通过通知的方式对请求和响应增强，比如请求响应的日志打印，注意生效逻辑与 MessageConverter 相同，RequestBodyAdviceAdapter 和 ResponseBodyAdvice 都根据响应的类型生效（类型 Object + support() 返回 true，可对全部请求响应生效）

## 接口介绍 <a href="#zciy0" id="zciy0"></a>

RequestBodyAdviceAdapter 抽象类继承自 RequestBodyAdvice 接口，RequestBodyAdviceAdapter 实现部分方法的默认返回，继承它做正式实现，案例见后面部分，这里介绍 RequestBodyAdvice 接口

```
public interface RequestBodyAdvice {

	// 判断是否满足监听条件，首要执行
	boolean supports(MethodParameter methodParameter, Type targetType,
			Class<? extends HttpMessageConverter<?>> converterType);

	// body 为空时执行
	Object handleEmptyBody(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

        // 在 body 被读取或者转化之前执行
	HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException;

	// 在 body 被读取或者转化之后执行
	Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);
```

ResponseBodyAdvice\<T> 接口

```
public interface ResponseBodyAdvice<T> {

	// 是否满足监听条件
	boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);

        // 当被 MessageConverter 选中后（还没执行 MessageConverter 中方法）
        // 在 body 被写入网卡前执行
	T beforeBodyWrite(T body, MethodParameter returnType, MediaType selectedContentType,
			Class<? extends HttpMessageConverter<?>> selectedConverterType,
			ServerHttpRequest request, ServerHttpResponse response);

}
```

## 使用案例 <a href="#ke4h0" id="ke4h0"></a>

### 日志打印逻辑代码 <a href="#beqjv" id="beqjv"></a>

```
@RestControllerAdvice
public class HeadersLogAdvice extends RequestBodyAdviceAdapter implements ResponseBodyAdvice<Object>, HandlerInterceptor {

    private static final String HTTP_HEADERS_LOG_CONTEXT = "httpHeadersLogContext";

    // HandlerInterceptor 记录当前请求统计并设置到 requestAttributes
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        HttpHeadersLogContext httpHeadersLogContext = new HttpHeadersLogContext();
        // 设置初始值
        httpHeadersLogContext.updateReadStartTime();
        httpHeadersLogContext.updateReadEndTime();
        RequestAttributes requestAttributes = RequestContextHolder.currentRequestAttributes();
        requestAttributes.setAttribute(HTTP_HEADERS_LOG_CONTEXT, httpHeadersLogContext, SCOPE_REQUEST);
        return true;
    }

    // RequestBodyAdviceAdapter 相关方法实现
    // 全局支持, 这里要返回 true
    @Override
    public boolean supports(MethodParameter methodParameter,
                            Type targetType,
                            Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    // 读取 request body 后更新结束时间
    @Override
    public Object afterBodyRead(Object requestBody, HttpInputMessage inputMessage,
                                MethodParameter parameter, Type targetType,
                                Class<? extends HttpMessageConverter<?>> converterType) {
        HttpHeadersLogContext httpHeadersLogContext = getHttpHeadersLogContext();
        if (httpHeadersLogContext != null) {
            httpHeadersLogContext.updateReadEndTime();
        }
        return requestBody;
    }

    // RequestBodyAdviceAdapter 相关方法实现
    // 全局支持, 这里要返回 true
    @Override
    public boolean supports(MethodParameter returnType,
                            Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    // 记录 response 写之前的时间
    @Override
    public Object beforeBodyWrite(Object responseBody,
                                  MethodParameter returnType, MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request, ServerHttpResponse response) {
        HttpHeadersLogContext httpHeadersLogContext = getHttpHeadersLogContext();
        if (httpHeadersLogContext != null) {
            httpHeadersLogContext.updateWriteStartTime();
        }
        return responseBody;
    }

    // 响应结束, 打印请求响应情况
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        StringBuilder logBuilder = new StringBuilder();
        logBuilder.append("Http Log:[");
        logBuilder.append("Client IP:").append(request.getRemoteAddr())
                .append(",Request URI:").append(request.getRequestURI())
                .append(",Status Code:").append(response.getStatus())
                .append(",");
        appendRequestHeaders(request, logBuilder);
        appendResponseHeaders(response, logBuilder);

        HttpHeadersLogContext httpHeadersLogContext = getHttpHeadersLogContext();
        if (httpHeadersLogContext != null) {
            if (ex != null) {
                logBuilder.append(",Exception:").append(ex.getMessage());
            }
            long readOpTime = httpHeadersLogContext.getReadEnd() - httpHeadersLogContext.getReadStart();
            long apiHandleOpTime = Math.max(httpHeadersLogContext.getWriteStart() - httpHeadersLogContext.getReadEnd(), 0);
            // 当前时间作为响应写完成后的时间
            long writeOpTime = httpHeadersLogContext.getWriteStart() > 0
                    ? System.currentTimeMillis() - httpHeadersLogContext.getWriteStart() : 0;
            // 总的时间消耗
            long totalOpTime = readOpTime + apiHandleOpTime + writeOpTime;
            totalOpTime = totalOpTime <= 0 ? System.currentTimeMillis() - httpHeadersLogContext.getReadStart() : totalOpTime;
            logBuilder.append(",ReadOpTime:").append(readOpTime).append("ms")
                    .append(",ApiHandleOpTime:").append(apiHandleOpTime).append("ms")
                    .append(",WriteOpTime:").append(writeOpTime).append("ms")
                    .append(",TotalOpTime:").append(totalOpTime).append("ms");

            RequestAttributes requestAttributes = RequestContextHolder.currentRequestAttributes();
            requestAttributes.removeAttribute(HTTP_HEADERS_LOG_CONTEXT, SCOPE_REQUEST);
        }
        logBuilder.append("]");
        if (ex != null) {
            System.out.println(logBuilder.toString());
            ex.printStackTrace();
        } else {
            System.out.println(logBuilder.toString());
        }
    }

    private void appendRequestHeaders(HttpServletRequest request, StringBuilder logBuilder) {
            logBuilder.append(",Request Headers:[");
            Enumeration<String> headers = request.getHeaderNames();
            while (headers.hasMoreElements()) {
                String header = headers.nextElement();
                logBuilder.append(header).append("=").append(request.getHeader(header)).append(",");
            }
            logBuilder.append("]");
    }

    private void appendResponseHeaders(HttpServletResponse response, StringBuilder logBuilder) {
        logBuilder.append(",Response Headers:[");
        for (String header : response.getHeaderNames()) {
            logBuilder.append(header).append("=").append(response.getHeader(header)).append(",");
        }
        logBuilder.append("]");
    }

    private HttpHeadersLogContext getHttpHeadersLogContext() {
        RequestAttributes requestAttributes = RequestContextHolder.currentRequestAttributes();
        return (HttpHeadersLogContext) requestAttributes.getAttribute(HTTP_HEADERS_LOG_CONTEXT, SCOPE_REQUEST);
    }

    private static class HttpHeadersLogContext {
        long readStart;
        long readEnd;
        long writeStart;

        HttpHeadersLogContext() {
        }

        public long getReadStart() {
            return readStart;
        }

        public long getReadEnd() {
            return readEnd;
        }

        public long getWriteStart() {
            return writeStart;
        }

        public void updateReadStartTime() {
            readStart = System.currentTimeMillis();
        }

        public void updateReadEndTime() {
            readEnd = System.currentTimeMillis();
        }

        public void updateWriteStartTime() {
            writeStart = System.currentTimeMillis();
        }
    }
}
```

## 注意 <a href="#vk7ew" id="vk7ew"></a>

RequestBodyAdviceAdapter 和 ResponseBodyAdvice 与 MessageConverter 有很强的联系。RequestBodyAdviceAdapter 的 beforeBodyRead 和 afterBodyRead 分别执行在 MessageConverter 读请求之前和之后，而 ResponseBodyAdvice 的 beforeBodyWrite 方法执行在 MessageConverter 响应写之前。
