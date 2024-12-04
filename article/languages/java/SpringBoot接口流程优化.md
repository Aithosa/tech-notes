# SpringBoot 接口流程优化

> 前言: 最近才发现之前很多以为理所当然的事情其实背后都是专门做了特殊的处理。比如我之前以为在接口流程里抛了异常，理所应当接口返回能透露出异常的信息，尤其是接口字段校验不通过时应该返回具体信息；再或者正常调了一个接口，至少日志上应该能看出有接口调用过的痕迹(暂不讨论日志应不应该打印相关信息)。

目前前后端交互基本都是通过接口调用的形式，而对于调接的过程中，我是有以下预期的：

1. 日志显示有接口被调用，应该打印调的哪个接口并且请求和响应是什么(有开关控制)
2. 对于请求里参数的校验，不通过应该给出提示
3. 接口的业务流程
4. 全局异常的捕获
5. 对接口返回的统一封装

本文就针对以上几点来优化一下 SpringBoot 接口调用的流程。

## 接口调用的日志记录

一般对于一个接口的调用来说有以下几点是需要关注的：接口的请求类型、接口的地址、请求的参数、响应的参数。
为了方便管理、减小代码冗余，比较好的方式是通过切面来处理。

切面具体代码如下，该方法会拦截所有的 controller 方法，打印出请求和响应信息：

```Java
@Slf4j
@Aspect
@Configuration
public class ControllerLogAspect {
    @Autowired
    private ObjectMapper objectMapper = new ObjectMapper();

    /**
     * 定义切点 切点为controller下所有的类
     * 其中类里的所有方法为连接点
     */
    @Pointcut("execution(* *.*.*.controller..*.*(..))")
    public void httpLog() {
    }

    /**
     * 环绕通知
     */
    @Around(value = "httpLog()")
    public Object httpLogAround(ProceedingJoinPoint joinPoint) throws Throwable {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes == null) {
            return joinPoint.proceed();
        }

        HttpServletRequest request = attributes.getRequest();

        String httpMethod = request.getMethod();
        String requestUri = request.getRequestURI();
        String requestBody = getParams(joinPoint);
        // 需要拼出完整的方法、url和请求参数
        log.info("receive {} request: {}, requestBody: {}", httpMethod, requestUri, requestBody);

        Object proceed = joinPoint.proceed();

        log.info("send response: {}, responseBody: {}", requestUri, objectMapper.writeValueAsString(proceed));

        return proceed;
    }

    private String getParams(JoinPoint joinPoint) {
        StringBuilder params = new StringBuilder();
        if (joinPoint.getArgs() != null && joinPoint.getArgs().length > 0) {
            for (int i = 0; i < joinPoint.getArgs().length; i++) {
                Object arg = joinPoint.getArgs()[i];
                if ((arg instanceof HttpServletResponse) || (arg instanceof HttpServletRequest)
                        || (arg instanceof MultipartFile) || (arg instanceof MultipartFile[])) {
                    continue;
                }
                try {
                    params.append(objectMapper.writeValueAsString(joinPoint.getArgs()[i]));
                } catch (Exception e1) {
                    log.error(e1.getMessage());
                }
            }
        }
        return params.toString();
    }
}
```

## 全局异常的捕获

如果不做特殊处理的话，接口流程里抛出了异常，或是请求字段校验不通过(如`@NotNull`)，接口响应并不会返回异常相关的信息，而是 500 错误：

```Json
{
    "timestamp": "2023-03-08T09:05:08.767+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "path": "/simple/query"
}
```

对于这种情况，一般有两种解决方法：

1. 捕获异常，并且在发生错误需要中断流程时，根据响应的结构赋值错误码和错误描述返回给调用方
2. 自动捕获异常，然后在发生异常处中断流程并自动返回响应(包含错误码和错误描述)

第一种方法需要手工处理的地方较多，而且接口逻辑层次过深时需要把异常/错误信息一层一层带出来，然后返回。这种方法会比较麻烦，因此我们选择第二种，全局捕获然后自动返回响应，这样我们只需要在合适的位置抛出业务错误，并不需要关注后续的处理流程。

为了便于前端处理，接口的响应通常会定义统一的结构，其会使用错误码和错误描述类似的字段来表示处理的结果，如下：

```Java
public class Response<T> {
    @ApiModelProperty(value = "错误码，0表示操作成功", required = true, example = "0")
    protected String resultCode = "0";

    @ApiModelProperty(value = "错误描述", required = true, example = "操作成功")
    protected String description = "操作成功";

    @ApiModelProperty(value = "响应数据", notes = "没有具体类型的可以忽略")
    protected T data;
}
```

我们需要做的就是捕获全局的异常，然后赋值异常信息给通用响应然后返回。

这里推荐在业务中自定义异常来抛出业务错误，自定义业务异常可以如下(记得维护好业务错误信息)：

```Java
public class BizError extends RuntimeException {
    private static final long serialVersionUID = -2648423593534352966L;

    private final String code;

    private transient Object[] args = null;

    public BizError(String code, String message) {
        super(message);
        this.code = code;
    }
}
```

捕获全局异常本文是使用`@RestControllerAdvice`来实现的，它可以捕获特定的异常，然后把错误信息赋值给响应然后返回，因此我们可以指定只捕获特定的异常，或是根据不同的异常分别做特殊处理。具体实现如下：

```Java
@RestControllerAdvice
public class ExceptionControllerAdvice {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Response MethodArgumentNotValidExceptionHandler(MethodArgumentNotValidException e) {
        // 从异常对象中拿到ObjectError对象
        ObjectError objectError = e.getBindingResult().getAllErrors().get(0);
        // 提取错误提示信息进行返回
        return Response.fail(ErrorCodes.INVALID_PARA, objectError.getDefaultMessage());
    }

    @ExceptionHandler(BindException.class)
    public Response BindExceptionHandler(BindException e) {
        // 从异常对象中拿到ObjectError对象
        ObjectError objectError = e.getBindingResult().getAllErrors().get(0);
        // 提取错误提示信息进行返回
        return Response.fail(ErrorCodes.INVALID_PARA, objectError.getDefaultMessage());
    }

    @ExceptionHandler(BizError.class)
    public Response BizErrorHandler(BizError e) {
        return Response.fail(e.getCode());
    }

    @ExceptionHandler(Exception.class)
    public Response ExceptionHandler(Exception e) {
        return Response.fail(ErrorCodes.FAIL, e.getMessage());
    }
}
```

在上文中，MethodArgumentNotValidException 和 BindException 就是参数校验抛出的异常，而 BizError 是我们自定义的异常，Exception 是捕获剩余的所有异常，保证接口能返回所有错误信息(实际根据需要可以在接口内手动捕获特定异常再处理，防止安全信息泄露或是把不合适的错误信息返回给接口调用方)。

## 全局统一返回

如果使用统一响应结构的话，每次返回结果前我们需要手动调用`Response.success(data)`，全局统一返回是指接口处理只需要返回 data 字段，处理流程会自动把 data 字段赋值给 Response 的 data，然后返回 Response。这个功能不是那么必要，因此不详细说明了，代码如下：

```Java
@ControllerAdvice(basePackages = "com.example.exception.controller")
public class GlobResponseBodyAdvice implements ResponseBodyAdvice<Object> {
    @Autowired
    private ObjectMapper objectMapper;

    /**
     * 是否开启功能 true:开启
     */
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    /**
     * 处理返回结果
     */
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request, ServerHttpResponse response) {
        // 处理字符串类型数据
        if (body instanceof String) {
            try {
                return objectMapper.writeValueAsString(Response.success(body));
            } catch (JsonProcessingException e) {
                e.printStackTrace();
            }
        }

        // 返回类型是否已经封装
        if (body instanceof Response) {
            return body;
        }

        return Response.success(body);
    }
}
```

注意字符串和 Response 需要特殊处理。

## 参考

统一接口返回和全局异常

- [SpringBoot 统一接口返回和全局异常](https://blog.csdn.net/qq_20957669/article/details/122405812)

日志

- [使用 Spring 的 AOP 打印 HTTP 接口出入参日志](https://www.51cto.com/article/719612.html)
- [服务端接口日志打印的几种方法](https://zhuanlan.zhihu.com/p/338208389)
- [通过反射获取注解属性](https://blog.csdn.net/weixin_54280625/article/details/117459001)
- [SpringBoot 打印请求参数与响应参数](https://www.jianshu.com/p/aa4f3191aecb)
- [【Spring Boot 系列】如何优雅的处理请求和响应的日志？](https://zhuanlan.zhihu.com/p/592371773)
- [Spring Boot 2.1 之后如何在启动日志中打印请求路径列表](https://blog.csdn.net/j3T9Z7H/article/details/104305894)
- [SpringBoot 项目优雅日志打印请求参数及返回参数](https://blog.csdn.net/HuHao_CSDN/article/details/105048588)
