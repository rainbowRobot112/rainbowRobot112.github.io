---
description: Springboot 全局异常机制使用
---

# Springboot 全局异常

## web 异常捕获 <a href="#rjta3" id="rjta3"></a>

SpringBoot 通过载入的 HandlerExceptionResolver 决定接口捕获异常时如何处理，支持将异常包装为 ModelAndView 类型的响应。

通过自定义继承接口 HandlerExceptionResolver 实现的抽象类 AbstractHandlerExceptionResolver 的方式，自行处理接口中的异常包装，源码中在 DispatcherServlet#processHandlerException 方法执行。AbstractHandlerExceptionResolver 继承 Order 接口，支持使用 @Order 编辑执行顺序

通过实现 WebMvcConfigurer#extendHandlerExceptionResolvers 方法添加自定义的 HandlerExceptionResolver，或者使用 bean 注入（不用开启 @EnableWebMvc 注解，默认策略 autoConfiguration 载入）

### 接口介绍 <a href="#q1dyq" id="q1dyq"></a>

拦截异常类型可通过 setMappedHandlers 和 setMappedHandlerClasses 定义，处理逻辑在 doResolveException 方法自定义，如果类型与默认支持拦截异常的类型冲突，需手动处理（比如 Order 排序或者直接替换默认 HandlerExceptionResolver）

```
public abstract class AbstractHandlerExceptionResolver implements HandlerExceptionResolver, Ordered {

    // 处理捕获异常封装的方法，返回接口响应信息
    protected abstract ModelAndView doResolveException(HttpServletRequest request,
			HttpServletResponse response, Object handler, Exception ex);
}
```

### 使用案例 <a href="#ngnxx" id="ngnxx"></a>

下面例子自定义 ServletHandlerExceptionResolver 覆盖 DefaultHandlerExceptionResolver，防止异常发生时被 DefaultHandlerExceptionResolver 捕获异常

```
public class HttpHandlerExceptionResolver extends DefaultHandlerExceptionResolver {

    public HttpHandlerExceptionResolver() {
        setOrder(Ordered.HIGHEST_PRECEDENCE);
    }

    @Override
    protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response,
                                              Object handler, Exception ex) {

        System.out.println("Throws ServletException: " + ex.getMessage()
            + ", request uri: " + request.getRequestURI());
        ex.printStackTrace();

        ModelAndView mv = super.doResolveException(request, response, handler, ex);
        if (mv == null) {
            response.setStatus(HttpServletResponse.SC_EXPECTATION_FAILED);
            return new ModelAndView();
        }
        return mv;
    }

    @NotNull
    protected ModelAndView handleHttpRequestMethodNotSupported(HttpRequestMethodNotSupportedException ex,
                                                               HttpServletRequest request, HttpServletResponse response,
                                                               @Nullable Object handler) throws IOException {

        String[] supportedMethods = ex.getSupportedMethods();
        if (supportedMethods != null) {
            response.setHeader("Allow", StringUtils.arrayToDelimitedString(supportedMethods, ", "));
        }
        response.setStatus(HttpServletResponse.SC_METHOD_NOT_ALLOWED);
        response.getWriter().print(ex.getMessage());
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleHttpMediaTypeNotSupported(HttpMediaTypeNotSupportedException ex,
                                                           HttpServletRequest request, HttpServletResponse response,
                                                           @Nullable Object handler) throws IOException {

        List<MediaType> mediaTypes = ex.getSupportedMediaTypes();
        if (!CollectionUtils.isEmpty(mediaTypes)) {
            response.setHeader("Accept", MediaType.toString(mediaTypes));
            if (request.getMethod().equals("PATCH")) {
                response.setHeader("Accept-Patch", MediaType.toString(mediaTypes));
            }
        }
        response.setStatus(HttpServletResponse.SC_UNSUPPORTED_MEDIA_TYPE);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleHttpMediaTypeNotAcceptable(HttpMediaTypeNotAcceptableException ex,
                                                            HttpServletRequest request, HttpServletResponse response,
                                                            @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_NOT_ACCEPTABLE);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleMissingPathVariable(MissingPathVariableException ex,
                                                     HttpServletRequest request, HttpServletResponse response,
                                                     @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        response.getWriter().print(ex.getMessage());
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleMissingServletRequestParameter(MissingServletRequestParameterException ex,
                                                                HttpServletRequest request, HttpServletResponse response,
                                                                @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        response.getWriter().print(ex.getMessage());
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleServletRequestBindingException(ServletRequestBindingException ex,
                                                                HttpServletRequest request, HttpServletResponse response,
                                                                @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        response.getWriter().print(ex.getMessage());
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleConversionNotSupported(ConversionNotSupportedException ex,
                                                        HttpServletRequest request, HttpServletResponse response,
                                                        @Nullable Object handler) throws IOException {

        sendServerError(ex, request, response);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleTypeMismatch(TypeMismatchException ex,
                                              HttpServletRequest request, HttpServletResponse response,
                                              @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleHttpMessageNotReadable(HttpMessageNotReadableException ex,
                                                        HttpServletRequest request, HttpServletResponse response,
                                                        @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleHttpMessageNotWritable(HttpMessageNotWritableException ex,
                                                        HttpServletRequest request, HttpServletResponse response,
                                                        @Nullable Object handler) throws IOException {

        sendServerError(ex, request, response);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleMethodArgumentNotValidException(MethodArgumentNotValidException ex,
                                                                 HttpServletRequest request, HttpServletResponse response,
                                                                 @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleMissingServletRequestPartException(MissingServletRequestPartException ex,
                                                                    HttpServletRequest request, HttpServletResponse response,
                                                                    @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        response.getWriter().print(ex.getMessage());
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleBindException(BindException ex, HttpServletRequest request,
                                               HttpServletResponse response, @Nullable Object handler) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleNoHandlerFoundException(NoHandlerFoundException ex,
                                                         HttpServletRequest request, HttpServletResponse response,
                                                         @Nullable Object handler) throws IOException {

        pageNotFoundLogger.warn(ex.getMessage());
        response.setStatus(HttpServletResponse.SC_NOT_FOUND);
        return new ModelAndView();
    }
    @NotNull
    protected ModelAndView handleAsyncRequestTimeoutException(AsyncRequestTimeoutException ex,
                                                              HttpServletRequest request, HttpServletResponse response,
                                                              @Nullable Object handler) throws IOException {
        if (!response.isCommitted()) {
            response.setStatus(HttpServletResponse.SC_SERVICE_UNAVAILABLE);
        } else {
            logger.warn("Async request timed out");
        }
        return new ModelAndView();
    }
    @NotNull
    protected void sendServerError(Exception ex, HttpServletRequest request, HttpServletResponse response)
        throws IOException {

        request.setAttribute("javax.servlet.error.exception", ex);
        response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
    }
```

注入自定义 ServletHandlerExceptionResolver

```
@Configuration
public class Configuration implements WebMvcConfigurer {

    @Override
    public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        for (int i = 0; i < resolvers.size(); i++) {
            if (resolvers.get(i) instanceof DefaultHandlerExceptionResolver) {
                resolvers.set(i, new HttpHandlerExceptionResolver());
                break;
            }
        }
    }
}
```

Controller 入口

```
@RestController
@RequestMapping
public class HelloController {

    @RequestMapping("/ex")
    public void ex() {
        throw new RuntimeException();
    }
}
```

### 注意 <a href="#p4md1" id="p4md1"></a>

异常的类型是否会被排序靠前的 HandlerExceptionResolver 捕获到，默认的异常捕获器加载过程可见 WebMvcConfigurationSupport#addDefaultHandlerExceptionResolvers

## 接口异常捕获 <a href="#rdip2" id="rdip2"></a>

Springboot 中，可以通过 @ControllerAdvice 开启全局异常处理，监听所有 Controller 接口，而异常则需要自定义方法并加上 @ExceptionHandler 注解并在注解参数上定义捕获异常的类型即可对捕获的异常进行统一的处理。

本质是通过 AOP 和事务机制拦截和处理异常，@ControllerAdvice 注解本质HandlerMethodReturnValueHandler 的实现，@ExceptionHandler 底层实现还是通过 HandlerExceptionResolver 捕获异常并处理，相比自定义 HandlerExceptionResolver 的实现方式更为快捷。

## 使用案例 <a href="#qh61t" id="qh61t"></a>

定义全局拦截器，将应用内部自定义异常类型捕获并处理

```
@RestControllerAdvice(annotations = Controller.class)
public class RestApiExceptionHandler {

    @ExceptionHandler({Throwable.class})
    public ResponseEntity<Object> handleException(Exception ex, HandlerMethod handlerMethod, HttpServletRequest request) {
        System.out.println("Exception Found in Mapping Method:" + handlerMethod
                + " and URL:" + request.getRequestURI(), ex);
        if (ex instanceof RestApiException) {
            try {
                return ResponseEntity.ok()
                        .contentType(MediaType.APPLICATION_JSON)
                        .body(((RestApiException) ex).toJSONString());
            } catch (IOException e) {
                System.out.println("Exception Found in ExceptionHandler");
                e.printStackTrace();
            }
        }
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
    }
}
```
