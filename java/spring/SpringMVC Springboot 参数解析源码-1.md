# SpringMVC Springboot 参数解析源码

### 打断点的步骤:
```
1. DispatcherServlet.java 的 ex1.handle(processedRequest, response, mappedHandler.getHandler()); 就是handlerAdapter 的调用

2. AbstractHandlerMethodAdapter.java
handle()方法

3. RequestMappingHandlerAdapter.java
handleInternal()方法

3. RequestMappingHandlerAdapter.java
invokeHandlerMethod()方法
	4. 底下有个requestMappingMethod.invokeAndHandle()方法
		5. 进入 -> ServletInvocableHandlerMethod.java 的 invokeAndHandle()方法 
			6. 底下有方法 —> ServletInvocableHandlerMethod.java 的 this.invokeForRequest()方法
				7. 底下有方法 -> ServletInvocableHandlerMethod .java this.getMethodArgumentValues() 这一步用来获取请求参数(controller 方法的参数，并注入)
					这时候进入 getMethodArgumentValues() 方法，就是方法参数的各种解析了
						现在方法参数主要分成3种
						1.get请求参数
						2.post请求参数
						3.dto请求实体
						具体查看spring mvc 怎么进行解析的
						8. HandlerMethodArgumentResolverComposite.java resolveArgument()方法，获取对应的 resolver
						这里就是参数解析的主体了！

tip:根据上述的 `-`或者序号 底下的类，进入， F5 进行搜索到对应方法然后打断点


```

### ServletInvocableHandlerMethod .java 继承 InvocableHandlerMethod.java

```
大部分参数解析都在 InvocableHandlerMethod.java 中

```

### ServletInvocableHandlerMethod .java 的方法 this.getMethodArgumentValues() 代码：

```

    private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    	/*
			已经通过解析controller 的具体请求方法，抽象成了 MethodParameter 的类型了 底下有 MethodParameter 的属性解析
    	*/
        MethodParameter[] parameters = this.getMethodParameters();
        Object[] args = new Object[parameters.length];

        for(int i = 0; i < parameters.length; ++i) {
            MethodParameter parameter = parameters[i];
            /*
				初始化参数解析器 LocalVariableTableParameterNameDiscoverer.java 一个通过字节码文件解析，获取参数对应的命名
            */
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            GenericTypeResolver.resolveParameterType(parameter, this.getBean().getClass());
            args[i] = this.resolveProvidedArgument(parameter, providedArgs);
            if(args[i] == null) {
                if(this.argumentResolvers.supportsParameter(parameter)) {
                    try {
                        args[i] = this.argumentResolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
                    } catch (Exception var9) {
                        if(this.logger.isTraceEnabled()) {
                            this.logger.trace(this.getArgumentResolutionErrorMessage("Error resolving argument", i), var9);
                        }

                        throw var9;
                    }
                } else if(args[i] == null) {
                    String msg = this.getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
                    throw new IllegalStateException(msg);
                }
            }
        }

        return args;
    }

```

### MethodParameter.java 的类，解释一下一些属性

```
public class MethodParameter {
	//controller 对应一个请求的方法
    private final Method method;
    private final Constructor constructor;
    //参数的index 位置
    private final int parameterIndex;
    // 参数类型
    private Class<?> parameterType;
    private Type genericParameterType;
    // 参数上对应的注解类
    private Annotation[] parameterAnnotations;
    // 参数解析器 (因为jdk 即便到了1.8 也无法解析到方法的参数名称，所以需要一个字节码解析类来进行解析)
    private ParameterNameDiscoverer parameterNameDiscoverer;
    // 参数名称 一般通过 parameterNameDiscoverer 去解析到
    private String parameterName;
    private int nestingLevel;
    Map<Integer, Integer> typeIndexesPerLevel;
    Map<TypeVariable, Type> typeVariableMap;
    private int hash;

    ... 省略方法代码
}
```



### HandlerMethodArgumentResolverComposite.java 的 getArgumentResolver()方法获取对应的参数解析器:
```
1.AbstractNamedValueMethodArgumentResolver.java 用来解析@RequestParam 的参数(不添加注解默认是这个)
2.RequestResponseBodyMethodProcessor.java 用来解析请求体的参数
```

#### 源码：
```

    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
        HandlerMethodArgumentResolver result = (HandlerMethodArgumentResolver)this.argumentResolverCache.get(parameter);
        if(result == null) {
            Iterator var4 = this.argumentResolvers.iterator();

            while(var4.hasNext()) {
                HandlerMethodArgumentResolver methodArgumentResolver = (HandlerMethodArgumentResolver)var4.next();
                if(this.logger.isTraceEnabled()) {
                    this.logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" + parameter.getGenericParameterType() + "]");
                }

                if(methodArgumentResolver.supportsParameter(parameter)) {
                    result = methodArgumentResolver;
                    this.argumentResolverCache.put(parameter, methodArgumentResolver);
                    break;
                }
            }
        }

        return result;
    }


```

### HandlerMethodArgumentResolver.java 接口，参数解析器接口
```
public interface HandlerMethodArgumentResolver {
	//支持的参数类型解析
    boolean supportsParameter(MethodParameter var1);
    //解析参数
    Object resolveArgument(MethodParameter var1, ModelAndViewContainer var2, NativeWebRequest var3, WebDataBinderFactory var4) throws Exception;
}

```

### 以下是实现类和它的代码：

#### RequestParamMethodArgumentResolver.java 继承 AbstractNamedValueMethodArgumentResolver.java 
``` 
resolveArgument(); 处理参数的方法
supportsParameter();方法用来判断可以解析的参数类型
```
#### 源码如下：

```
    public boolean supportsParameter(MethodParameter parameter) {
        Class paramType = parameter.getParameterType();
        if(parameter.hasParameterAnnotation(RequestParam.class)) {
            if(Map.class.isAssignableFrom(paramType)) {
                String paramName = ((RequestParam)parameter.getParameterAnnotation(RequestParam.class)).value();
                return StringUtils.hasText(paramName);
            } else {
                return true;
            }
        } else {
            return parameter.hasParameterAnnotation(RequestPart.class)?false:(!MultipartFile.class.equals(paramType) && !"javax.servlet.http.Part".equals(paramType.getName())?(this.useDefaultResolution?BeanUtils.isSimpleProperty(paramType):false):true);
        }
    }




    protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
        HttpServletRequest servletRequest = (HttpServletRequest)webRequest.getNativeRequest(HttpServletRequest.class);
        MultipartHttpServletRequest multipartRequest = (MultipartHttpServletRequest)WebUtils.getNativeRequest(servletRequest, MultipartHttpServletRequest.class);
        Object arg;
        if(MultipartFile.class.equals(parameter.getParameterType())) {
            this.assertIsMultipartRequest(servletRequest);
            Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
            arg = multipartRequest.getFile(name);
        } else if(this.isMultipartFileCollection(parameter)) {
            this.assertIsMultipartRequest(servletRequest);
            Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
            arg = multipartRequest.getFiles(name);
        } else if("javax.servlet.http.Part".equals(parameter.getParameterType().getName())) {
            this.assertIsMultipartRequest(servletRequest);
            arg = servletRequest.getPart(name);
        } else {
            arg = null;
            if(multipartRequest != null) {
                List paramValues = multipartRequest.getFiles(name);
                if(!paramValues.isEmpty()) {
                    arg = paramValues.size() == 1?paramValues.get(0):paramValues;
                }
            }

            if(arg == null) {
            	//直接拿参数的值，然而post请求从这里去不到参数值
                String[] paramValues1 = webRequest.getParameterValues(name);
                if(paramValues1 != null) {
                    arg = paramValues1.length == 1?paramValues1[0]:paramValues1;
                }
            }
        }

        return arg;
    }

```


### 所以当请求是post 方法时，request 中的parameter中是不存在参数的，所以如果是普通的请求像这样
### 请求信息：
```
content-type : application/json 时 
request-body : {
	"name" : "zhuangjiesen" ,
	"value" : "iamvalue"
}

```
#### controller 的代码
```
    @RequestMapping(value = "/test.do" , method = {RequestMethod.POST})
    @ResponseBody
    public String test(String name , String value, HttpServletRequest request, HttpServletResponse response , ModelAndView modelAndView){
        return "test success !.";
    }




    @RequestMapping("/posttest.do")
    @ResponseBody
    public String posttest(
            String name ,
//            String value,
            @RequestBody Map<String , Object> postRequest ,
            HttpServletRequest request,
            HttpServletResponse response ,
            ModelAndView modelAndView) throws Exception {
        String s1 = request.getParameter("name");
        String s2 = request.getParameter("value");
        System.out.println("request : " + request.getClass().getName());
        System.out.println("postRequest : " + JSONObject.toJSONString(postRequest));
        return "posttest success !.";
    }



    @RequestMapping("/posttestArticle.do")
    @ResponseBody
    public String posttestArticle(
            String name ,
//            String value,
            @RequestBody Article article ,
            HttpServletRequest request,
            HttpServletResponse response
            ) throws Exception {

        System.out.println("request : " + request.getClass().getName());
        System.out.println("article : " + JSONObject.toJSONString(article));
        return "posttest success !.";
    }



```
#### 结论

```
 String name , String value 都是取不到值的，因为没配置注解，默认是 RequestParam.class 
请求处理器就会是 RequestParamMethodArgumentResolver.java
 ,源码中会去 HttpServletRequest request 取 parameter 
 如：String s1 = request.getParameter("name") 
 所以拿不到post请求的参数
```


#### RequestResponseBodyMethodProcessor.java 
```
resolveArgument(); 处理参数的方法
supportsParameter();方法用来判断可以解析的参数类型

角色：解析 RequestBody(注解) 类型时的参数
```

#### 源码如下：
```
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestBody.class);
    }


    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
    	/*
    		先将 request 请求转成 ServletServerHttpRequest 通过 HttpMessageConverter.java 去处理 http body 的数据
    		转换成服务器的json 或者 xml 可以通过配置实现数据实体化
    		Object arg -> 对象是个 LinkedHashMap 类型
    	*/ 
        Object arg = this.readWithMessageConverters(webRequest, parameter, parameter.getParameterType());
        Annotation[] annotations = parameter.getParameterAnnotations();
        Annotation[] var10 = annotations;
        int var9 = annotations.length;

        /*
        	这里通过对于注解的数据绑定
        	如果在@RequestBody 中定义的是对象就通过反射注入
        	如果是map 就直接put 进进去
        */
        for(int var8 = 0; var8 < var9; ++var8) {
            Annotation annot = var10[var8];
            if(annot.annotationType().getSimpleName().startsWith("Valid")) {
                String name = Conventions.getVariableNameForParameter(parameter);
                WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
                Object hints = AnnotationUtils.getValue(annot);
                binder.validate(hints instanceof Object[]?(Object[])hints:new Object[]{hints});
                BindingResult bindingResult = binder.getBindingResult();
                if(bindingResult.hasErrors()) {
                    throw new MethodArgumentNotValidException(parameter, bindingResult);
                }
            }
        }

        return arg;
    }
    ... 省略部分代码


    protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter methodParam, Class<T> paramType) throws IOException, HttpMediaTypeNotSupportedException {
    	/*
    		将body 的值解析出来
    	*/
        ServletServerHttpRequest inputMessage = this.createInputMessage(webRequest);
        return this.readWithMessageConverters((HttpInputMessage)inputMessage, methodParam, paramType);
    }

    protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter methodParam, Class<T> paramType) throws IOException, HttpMediaTypeNotSupportedException {
        MediaType contentType = inputMessage.getHeaders().getContentType();
        if(contentType == null) {
            contentType = MediaType.APPLICATION_OCTET_STREAM;
        }

        Iterator var6 = this.messageConverters.iterator();

        while(var6.hasNext()) {
            HttpMessageConverter messageConverter = (HttpMessageConverter)var6.next();
            /*
            	这里判断 messageConverter 是否能进行解析
            	这边是通过 MappingJackson2HttpMessageConverter.java 进行解析的
            */
            if(messageConverter.canRead(paramType, contentType)) {
                if(this.logger.isDebugEnabled()) {
                    this.logger.debug("Reading [" + paramType.getName() + "] as \"" + contentType + "\" using [" + messageConverter + "]");
                }
                /*
                	这里进行解析的
                */
                return messageConverter.read(paramType, inputMessage);
            }
        }

        throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
    }

    protected ServletServerHttpRequest createInputMessage(NativeWebRequest webRequest) {
        HttpServletRequest servletRequest = (HttpServletRequest)webRequest.getNativeRequest(HttpServletRequest.class);
        return new ServletServerHttpRequest(servletRequest);
    }

```


到这里，参数解析过程结束

### 结论：
```
1. controller 请求处理的方法参数被解析成属性类 MethodParameter.java 
2. 参数解析的入口 ：ServletInvocableHandlerMethod .java 的方法 this.getMethodArgumentValues()
3. 解析方法中，定义的参数名称 - 也就是代码里写的参数名，但是在jvm运行时，通过反射获取 Methed.java 的参数名效果是 arg0 arg1 的形式的，跟注入参数的业务需求不符合，所以这里会有一个 parameterNameDiscoverer 即 ：LocalVariableTableParameterNameDiscoverer.java 
4. 根据参数的类型 ： 
	1.定义成 @RequestParam 的参数 (默认) -> AbstractNamedValueMethodArgumentResolver.java
		- 这里的参数直接从request 对象获取
	2.定义成 @RequestBody 的参数 -> RequestResponseBodyMethodProcessor.java
		- 这里的参数获取步骤：
			1. 先判断mediaType 也就是 Content-Type 类型
			2. 获取对应的 messageConverter 
			3. 读取 http 请求体(body)的数据
			4. messageConverter.read() 方法，把请求解析出来对应到具体的定成的 @RequestBody 的对象
			这边默认的是application/json 类型，所以messageConverter 对象是 MappingJackson2HttpMessageConverter.java 
	3. 就是系统默认的 request / response 
```

### 后续
```
//TODO 
post请求参数(application/json)的获取都需要用 @RequestBody 来获取吗
//TODO
spring boot 参数解析是不是一样的
```


