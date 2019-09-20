# Spring MVC Document
#spring

# Spring Web MVC
Spring Web MVC 是基于 Servlet API 构建的原始 Web 框架，从一开始就包含在 Spring Framework 中。 正式名称 “Spring Web MVC” 来自其源模块（spring-webmvc）的名称，但它通常被称为 “Spring MVC”。

与 Spring Web MVC 一起，Spring Framework 5.0 引入了一个reactive-stack Web 框架，其名称 “Spring WebFlux” 也基于其源模块（spring-webflux）。 本节介绍 Spring Web MVC。 下一节将介绍 Spring WebFlux。
- - - -
## 1.1. DispatcherServlet
与许多其他 Web 框架一样，Spring MVC 围绕前端控制器模式设计，其核心的Servlet :DispatcherServlet 为请求处理提供共享的处理方法，而实际工作由可配置的delegate components执行。 该模型非常灵活，支持多种工作流程。

DispatcherServlet 与任何 Servlet 一样，需要使用 Java 配置或 web.xml 根据 Servlet 规范进行声明和映射。 反过来，::DispatcherServlet 使用 Spring 配置来发现请求映射，视图解析，异常处理等所需的委托组件。::

以下 Java 配置示例注册并初始化 DispatcherServlet，它由 Servlet 容器自动检测，::下面new了一个 WebApplicaitonContext，接着,使用这个WebApplicaitonContext创建了DispatcherServlet，并且把DispatcherServlet添加到ServletContext 中:::
```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletCxt) {
        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
        ac.register(AppConfig.class);
        ac.refresh();

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(ac);
        ServletRegistration.Dynamic registration = servletCxt.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
```
::所以，要代替 XML 形式的 web.xml配置，我们只需要实现WebApplicationInitializer，并且在 onStartup（）中创建 ApplicaitionContext 时，声明使用的配置类或者配置文件位置即可。::

### 1.1.1 Context Hierarchy
DispatcherServlet 需要一个 ***WebApplicationContext***（普通 ApplicationContext 的扩展）来进行自己的配置。 WebApplicationContext 有一个指向 ServletContext 的链接以及与之关联的 Servlet。它还绑定到 ServletContext。应用程序可以使用RequestContextUtils的静态方法，以访问WebApplicationContext。

对于许多应用程序，拥有单个 WebApplicationContext 足够了。但是，也可以有一个context层次结构，其中根 WebApplicationContext 在多个 DispatcherServlet实例之间共享，每个DispatcherServlet实例都有自己的子 WebApplicationContext 配置。有关上下文层次结构功能的更多信息，请参阅 ApplicationContext 的其他功能。

根 WebApplicationContext 通常包含基础结构 bean，例如需要跨多个 Servlet 实例共享的repositories和services。子 WebApplicationContext 中通常包含给定 Servlet 本地的 bean。下图显示了这种关系：
![](Spring%20MVC%20Document/mvc-context-hierarchy.png)
以下示例配置了一个 WebApplicationContext 层次结构：
```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
```

### 1.1.2 Special Bean Types
DispatcherServlet 委托特殊的 bean 处理请求并解析响应。 “特殊的bean” 是指实现框架契约的， Spring 管理的一些 Object 实例。 那些通常带有内置合同，::您可以自定义其属性并扩展或替换它们::。

下表列出了 ::DispatcherServlet 检测::的特殊 bean：
![](Spring%20MVC%20Document/29ADA8E3-EEA9-4537-954C-F32236273354.png)

### 1.1.3 Web MVC Config(指 DispatchServlet 那些)
应用程序可以声明所需要的特殊 Bean的实现类型，比如HandlerMapping使用RequestMappingHandlerMapping 。 DispatcherServlet 将检查WebApplicationContext来配置这些特殊的 bean。 如果没有用户匹配的特殊bean，它将回退使用 [DispatcherServlet.properties](https://github.com/spring-projects/spring-framework/blob/master/spring-webmvc/src/main/resources/org/springframework/web/servlet/DispatcherServlet.properties) 中列出的默认类型。

### 1.1.4 Servlet Config（指 ApplicationContext 那些）
您也可以选择以编程方式配置 Servlet 容器。
```java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

WebApplicationInitializer 是 Spring MVC 提供的一个接口，可确保检测到您自己提供的实现，并使用其来初始化Servlet 3 容器。 名为 AbstractDispatcherServletInitializer 的 WebApplicationInitializer 的抽象基类，实现通过重写方法来specify the servlet mapping and the location of the DispatcherServlet configuration.，使注册DispatcherServlet更加容易。
```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

AbstractDispatcherServletInitializer 还提供了一种方便的方法来添加 Filter 到容器，并将它们自动映射到 DispatcherServlet，如下例所示：
```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {
    // ...
    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
```

### 1.1.5 Processing

DispatcherServlet 按如下方式处理请求：
* 将请求对应的WebApplicationContext 被绑定到了请求中，作为*attribute*。可以使用 DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE 这个 key 来访问。
* locale resolver绑定到请求，以允许进程中的元素解析处理请求时使用的locale（解析视图，准备数据等）。如果您不需要区域设置解析，则不需要locale resolver.。
* theme resolver绑定到请求，以允许视图等元素确定要使用的主题。如果您不使用主题，则可以忽略它。
* 如果指定了multipart file resolver，则会检查请求，以得到multipart。如果找到了，请求将被包装为MultipartHttpServletRequest，以供进程中的其他元素做进一步处理。详细信息参见[Multipart Resolver](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-multipart)
* 搜索适当的handler。如果找到handler，则执行与处理程序（preprocessors, postprocessors, and controllers）关联的execution chain以准备model或呈现。
* 如果返回了模型，则呈现视图。如果没有返回模型（可能是由于预处理器或后处理器拦截请求，可能是出于安全原因），则不会呈现任何视图，因为该请求可能已经完成。

WebApplicationContext 中声明的 HandlerExceptionResolver bean 用于解决在请求处理期间抛出的异常。 这些异常解析器允许自定义逻辑以解决异常。 

Spring DispatcherServlet 还支持返回*last-modification-date*，确定特定请求的最后修改日期的过程很简单：DispatcherServlet 查找适当的handler映射并测试找到的handler是否实现 了 LastModified 接口。 如果实现了，LastModified 接口的 long getLastModified（request）方法的值将被返回给客户端。

### 1.1.6 Interception

所有 HandlerMapping 实现都支持handler interceptors，当您想要将特定功能应用于某些请求时，这些拦截器很有用 - for example, checking for a principal。 拦截器必须实现HandlerInterceptor，并实现其中的三种方法，这三种方法应该提供足够的灵活性来进行各种预处理和后处理：
* preHandle(..): Before the actual handler is executed
* postHandle(..): After the handler is executed
* afterCompletion(..): After the complete request has finished
preHandle（..）方法返回一个布尔值。您可以使用此方法来中断或继续execution chain的处理。当此方法返回 true 时，handler execution chain继续。当它返回 false 时，DispatcherServlet 假定拦截器本身已处理请求（例如，呈现适当的视图）并且不继续执行execution chain中的other interceptors and the actual handler。

请注意，对于在 postHandle*（） 之前就在 HandlerAdapter 中产生并提交了response的 @ResponseBody 和 ResponseEntity 方法，postHandle 都不会起效果。这意味着对响应进行任何更改都为时已晚，例如添加额外的标头。对于这个情形，您可以实现 ResponseBodyAdvice类型 并将其声明为 Controller Advice bean 或直接在 RequestMappingHandlerAdapter 上进行配置。

### 1.1.7 Exceptions

如果在request mapping期间发生异常或从request handler（例如 @Controller）抛出异常，则 DispatcherServlet 委托给 HandlerExceptionResolver bean 链以解决异常并提供替代处理，通常是产生一个error response。

下表列出了可用的 HandlerExceptionResolver 实现：
![](Spring%20MVC%20Document/255DF867-D0DE-402B-A887-8AB93DB133F3.png)

::您可以通过在 Spring 配置中声明多个 HandlerExceptionResolver bean 并根据需要设置其order属性::，来形成异常解析程序链。 order 属性越高，异常解析器定位的越晚。HandlerExceptionResolver 可以返回：
* **a ModelAndView** that points to an error view.
* **An empty ModelAndView** if the exception was handled within the resolver.
* **null** if the exception remains unresolved, for subsequent resolvers to try, and, if the exception remains at the end, it is allowed to bubble up to the Servlet container.

MVC Config 自动声明了内置的异常解析器，用于默认的 Spring MVC 异常，或者@ResponseStatus的异常，并提供了 @ExceptionHandler 方法的支持。 您可以自定义该列表或替换它。

如果任何 HandlerExceptionResolver 解析链仍未解析异常，并且因此将其传播，或者如果响应状态被设置为错误状态（即 4xx，5xx），那么Servlet 容器可以呈现默认的 HTML错误页面。 要自定义容器的默认错误页面，可以在 web.xml 中声明错误页面的映射。 以下示例显示了如何执行此操作：
```xml
<error-page>
    <location>/error</location>
</error-page>
```

根据前面的示例，当异常冒泡或响应具有错误状态时，Servlet 容器会在容器内把ERROR dispatch到配置的 URL（例如，/error）。 然后由 DispatcherServlet 处理，所以，我们可以将其映射到 @Controller，这个 controller以返回带有模型的错误视图名称或呈现 JSON 响应，如以下示例所示：
```java
@RestController
public class ErrorController {

    @RequestMapping(path = “/error”)
    public Map<String, Object> handle(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put(“status”, request.getAttribute(“javax.servlet.error.status_code”));
        map.put(“reason”, request.getAttribute(“javax.servlet.error.message”));
        return map;
    }
}
```

### 1.1.8. View Resolution

Spring MVC 定义了 ViewResolver 和 View 接口，::使您可以在浏览器中呈现模型，而无需绑定特定的视图技术。 ViewResolver 其实就是做了一个逻辑视图名称view name到实际使用的视图actual view的映射::。 View 在移交给特定视图技术之前解决了数据准备的问题。

Spring Framework本身其实 就带了几个ViewResolver 而已: *InternalResourceViewResolver*, *XmlViewResolver*,*ResourceBundleViewResolver*and a few others.

比如, *InternalResourceViewResolver*这个ViewResolver 允许我们为视图名称设置前缀或后缀等属性，以生成最终视图页面的URL：
```java
@Bean
public ViewResolver internalResourceViewResolver() {
InternalResourceViewResolver bean = new InternalResourceViewResolver();
bean.setViewClass(JstlView.class);
bean.setPrefix("/WEB-INF/view/");
bean.setSuffix(".jsp");
return bean;
```

下表提供了有关 ViewResolver 层次结构的更多详细信息：
![](Spring%20MVC%20Document/FF86C7F8-7C68-4CC4-AFAD-1D2500BF13BF.png)
![](Spring%20MVC%20Document/33497866-27D2-451A-9079-0BECF5A85FC6.png)
* Handling
::您可以通过声明多个解析程序 bean 链接视图解析程序::，并在必要时通过设置 order 属性来指定排序。 请记住，order 属性越高，视图解析器在链中的位置越晚。

ViewResolver 的contract指定它可以返回 null 来表示无法找到该视图。 但是，对于 JSP 和 InternalResourceViewResolver，确定 JSP 是否存在的唯一方法是通过 RequestDispatcher 执行调度。 因此，::您必须始终将 InternalResourceViewResolver 配置为视图解析器的链条中的最后一个。::

配置视图解析，只需要将ViewResolver bean 添加到 Spring 配置。 MVC Config 为 View Resolvers 提供专们的配置 API，并提供了添加logic-less View Controllers的方法， which are useful for HTML template rendering without controller logic.
* Redirecting
在视图名称中使用特殊的::redirect::: 前缀允许您执行重定向。 UrlBasedViewResolver（及其子类）将此识别为需要重定向的指令。 视图名称的其余部分是重定向 URL。

这达到的效果就像是Controller返回了 RedirectView ，但现在控制器可以按逻辑的视图名称（而不是绝对视图名称）操作。 逻辑视图名称（例如 redirect：/myapp/some/resource）基于当前Servlet上下文重定向，而像：http：//myhost.com/some/arbitrary/path 这样的，重定向到绝对 URL。
* Forwarding
您还可以为**最终由** UrlBasedViewResolver 和子类解析的视图名称使用特殊的 forward：前缀。 这将创建一个 ::InternalResourceView，它执行 RequestDispatcher.forward（）::（RequestDispatcher是 Tomcat 中的概念）。 因此，此前缀对于 InternalResourceViewResolver 和 InternalResourceView（对于 JSP）没有用，但如果您使用其他视图技术但仍希望强制 Servlet / JSP 引擎处理资源的转发，则它可能会有所帮助。 
* Content Negotiation
ContentNegotiatingViewResolver 本身不解析视图，而是委托给其他视图解析器，并选择客户端请求中指定的representation的视图。::可以从 Accept header或request参数（例如，“/path？format = pdf”）确定representation::。

ContentNegotiatingViewResolver 通过将request media types与与其每个 ViewResolvers 关联的视所图支持的media type（也称为 Content-Type）进行比较， 并选择适当的 View 来处理请求。 View 的链条中具有兼容 Content-Type 的第一个 View 将表示返回给客户端。 如果 ViewResolver 链无法提供兼容的视图，则会查询通过 DefaultViews 属性指定的视图列表。后一个选项适用于单个视图，它可以呈现当前资源的适当表示，而不管逻辑视图名称如何。 

Accept header可以包含通配符（例如 text / *），在这种情况下，Content-Type 为 text /xml 的 View  也是兼容匹配。

### Local 和 Themes 

用于国际化和多主题展现，跳过，参见[1.1.9 Locale](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-servlet-config)

### 1.1.11 Multipart Resolver

org.springframework.web.multipart 包中的 MultipartResolver 是一种用于解析包括文件上载在内的multipart request的策略。 有一个基于 Commons FileUpload 的实现，另一个实现基于Servlet 3.0 multipart request parsing。

要启用MultiPart处理，需要在 DispatcherServlet Spring 配置一个名为 multipartResolver 的bean，类型为 MultipartResolver。 DispatcherServlet 检测到它并将其应用于传入请求。 当收到内容类型为 multipart /form-data 的 POST 时，解析器会解析内容并将当前的 HttpServletRequest 包装为 MultipartHttpServletRequest，以提供对已解析部分的访问，并将其作为请求参数公开。

要使用 **Apache Commons FileUpload**，您可以配置名为 multipartResolver 的 CommonsMultipartResolver 类型的 bean。 您还需要将 commons-fileupload 作为依赖添加到项目。

若要使用 Servlet 3.0，首先需要通过 Servlet 容器配置启用 Servlet 3.0 multipart parsing。如果使用Java配置，则需要在 Servlet registration 上设置 MultipartConfigElement。
```java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    // ...

    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {

        // Optionally also set maxFileSize, maxRequestSize, fileSizeThreshold
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }
}
```
一旦 Servlet 3.0 配置到位，您就可以添加名为 multipartResolver 的 StandardServletMultipartResolver 类型的 bean。

### 1.1.12. Logging
Spring MVC 中的 DEBUG 级别日志记录旨在实现紧凑，简约和人性化。 它侧重于高价值的信息，这些信息是 useful over and over again 的，另一些可能仅在调试特定问题时有用。

TRACE 级日志记录通常遵循与 DEBUG 相同的原则（例如，也不应该是消防软管），但可以用于调试任何问题。 此外，一些日志消息可能在 TRACE 与 DEBUG 中显示不同的详细程度。

DEBUG 和 TRACE 日志记录可能会记录敏感信息。 这就是默认情况下屏蔽请求参数和header的原因，并且必须通过 DispatcherServlet 上的 enableLoggingRequestDetails 属性，来显式启用，以完整日志记录（含敏感信息）。以下示例说明如何使用 Java 配置执行此操作：
```java
public class MyInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return ... ;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return ... ;
    }

    @Override
    protected String[] getServletMappings() {
        return ... ;
    }

    @Override
    protected void customizeRegistration(Dynamic registration) {
        registration.setInitParameter("enableLoggingRequestDetails", "true");
    }

}
```

- - - -
## 1.2. Filters

Spring-web模块中提供了一些有用的过滤器
*  [Form Data](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#filters-http-put) 
*  [Forwarded Headers](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#filters-forwarded-headers) 
*  [Shallow ETag](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#filters-shallow-etag) 
*  [CORS](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#filters-cors) 

### 1.2.1. Form Data

浏览器只能通过 HTTP GET 或 HTTP POST 提交表单数据，但非浏览器客户端也可以使用 HTTP PUT，PATCH 和 DELETE。 Servlet API 要求 *ServletRequest.getParameter*（）* 方法仅支持 HTTP POST 的表单字段访问。

对此，spring-web 模块提供 **FormContentFilter** 来拦截 HTTP PUT，PATCH 和 DELETE 的表单提交请求，也就是内容类型为 application /x-www-form-urlencoded的请求。并且从request body中读取表单数据，并包装 *ServletRequest* 以表单参数可以通过 ServletRequest.getParameter *（）系列方法获取到。

### 1.2.2. Forwarded Headers

当请求通过代理（例如负载平衡器）时，host，port和shceme可能会发生变化，这使得从客户端角度创建指向正确host，port和shceme的链接成为一项挑战。

RFC 7239 定义了::代理可用::的，用于提供有关原始请求的信息的 **Forwarded** HTTP header。# There are other non-standard headers, too, including X-Forwarded-Host, X-Forwarded-Port , X-Forwarded-Proto ,X-Forwarded-Ssl, and X-Forwarded-Prefix.

ForwardedHeaderFilter 是一个 Servlet 过滤器，它根据 Forwarded header修改请求的主机，端口和方案，然后删除这些header。

Forwarded header存在安全隐患，因为应用程序无法知道header是由proxy添加还是由恶意客户端添加。 这就是为什么应该将代理配置为 删除来自外部的不受信任的Forwarded header。 您还可以使用 removeOnly = true 配置 ForwardedHeaderFilter，在这种情况下，它会删除但不使用标头。

![](Spring%20MVC%20Document/84DBB2E7-4ACE-482D-9596-C646834837D8.png)

### 1.2.3 CORS

Spring MVC 通过控制器上的注释为 CORS 相关的配置提供细粒度的支持。 但是，当与 Spring Security 一起使用时，我们建议CorsFilter这个内置的 filter必须ordered ahead of Spring Security 过滤器链。
- - - -
## 1.3. Annotated Controllers
Spring MVC 提供了一个基于注解的编程模型，其中 @Controller 和 @RestController 组件使用注解来表示请求映射，请求输入，异常处理等。 带注解的控制器具有灵活的方法签名，不必扩展基类，也不必实现特定的接口。 以下示例显示了由注解定义的控制器：
```java
@Controller
public class HelloController {
    @GetMapping("/hello")
    public String handle(Model model) {
        model.addAttribute("message", "Hello World!");
        return "index";
    }
}
```

### 1.3.1. Declaration
您可以使用WebApplicationContext 中的标准 Spring bean 定义来定义一个控制器 bean。 @Controller stereotype允许自动检测控制器，这和与Spring支持检测类路径中的 @Component 类并自动注册它们的 bean 定义是一样的。 它另外还表明它作为 Web 组件的角色。

要启用此类 @Controller bean 的自动检测，您可以将组件扫描添加到 Java 配置中，如以下示例所示：
```java
@Configuration
@ComponentScan("org.example.web")
public class WebConfig {
    // ...
}
```

@RestController 是一个组合注解，它本身以 @Controller 和 @ResponseBody 组成，以表示一个控制器的每个方法都继承了class上的 @ResponseBody 注解，因此直接写入响应主体，而不是解析视图和使用  HTML 模板。

### 1.3.2 Request Mapping
您可以使用 @RequestMapping 注解将请求映射到控制器方法。 它具有各种属性，可根据 URL，HTTP 方法，请求参数，header和媒体类型进行匹配。 您可以在类级别使用它来表示共享映射，或者在方法级别使用它来映射到特定的endpoint。

以下示例同时具有类型和方法级别映射：
```java
@RestController
@RequestMapping("/persons")
class PersonController {
    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Long id) {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void add(@RequestBody Person person) {
        // ...
    }
}
```

#### URI patterns
您可以使用以下全局模式和通配符映射请求：
* ? matches one character
* * matches zero or more characters within a path segment
* ** match zero or more path segments

您还可以使用 @PathVariable 声明 URI 变量并访问它们的值，如以下示例所示：
```java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
```

您可以在类和方法级别声明 URI 变量，如以下示例所示：
```java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {
    @GetMapping("/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
```

URI中的变量将自动转换为适当的类型，或抛出TypeMismatchException。 默认情况下，简单类型（int，long，Date 等）都是支持的，::您可以注册对任何其他数据类型的支持。 请参见 [Type Conversion](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-typeconversion) and [DataBinder](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-initbinder) 。::

您也可以在形参中显式命名 URI 变量（例如，@ PathVariable（“customId”））

语法 {varName：regex} 可以声明一个符合一定正则规则的URI 变量，其正则表达式的语法为 {varName：regex}。 例如，给定 URL“/spring-web-3.0.5 .jar”，以下方法提取名称，版本和文件扩展名：
```java
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String version, @PathVariable String ext) {
    // ...
}
```

URI 路径模式还可以嵌入 $ {...} 占位符来表示引用某些配置值，这些占位符在启动时，通过对本地，系统，环境和其他属性源使用 PropertyPlaceHolderConfigurer 来解析其值。 例如，您可以使用此参数来基于某些外部配置参数化基本的 URL。

#### Pattern Comparison

模式指 requestMapping 中指定的路径串，当多个pattern与 URL请求 匹配时，到底选择哪个控制器呢？必须对它们进行比较以找到最佳匹配。 这是通过使用 AntPathMatcher.getPatternComparator（String path）来完成的，它会查找看起来更specific的pattern。

如果 URI 变量的数量较少（计为 1），*通配符较少（计为 1），**通配符较少（计为 2），则模式的specific较低。 如果匹配分相等，那么将选择较长的pattern。 给定相同的分数和长度，选择具有比通配符更多的 URI 变量的模式。

默认映射模式（/ **）是中排除，并始终排在最后。 此外，前缀模式（例如 /public/ **）are considered less specific than那些不具有双通配符的模式。

有关完整的详细信息，请参阅 AntPathMatcher 中的 AntPatternComparator，您可以自定义 PathMatcher 实现。 See [Path Matching](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-config-path-matching) in the configuration section.

#### Suffix Match
默认情况下，Spring MVC 执行使用通配符 * 的后缀模式匹配，以便映射到 /person 的控制器也隐式映射到 /person.*。 然后使用文件扩展名来推断用于响应的请求内容类型（即，而不是 Accept header） - 例如，/person.pdf，就认定请求类型是 pdf。

当浏览器用于发送难以一致解释的 Accept 标头时，必须以这种方式使用文件扩展名。 其实目前，使用 Accept Header应该是首选。

随着时间的推移，文件扩展名的使用已经证明有多种方式存在问题。 当使用 URI 变量，路径参数和 URI 编码进行覆盖时，它可能会导致歧义。 基于 URL 的授权和安全性的推理也变得更加困难。

要完全禁用文件扩展名，必须同时设置以下两项：
* useSuffixPatternMatching(false) , see [PathMatchConfigurer](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-config-path-matching)
* favorPathExtension(false), see [ContentNegotiationConfigurer](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-config-content-negotiation)

#### Consumable Media Types

您可以根据请求的 Content-Type 缩小请求映射范围，如以下示例所示：
```java
@PostMapping(path = "/pets", consumes = "application/json") 
public void addPet(@RequestBody Pet pet) {
    // ...
}
```
consumes 属性还支持否定表达式 - 例如，！text /plain 表示除 text /plain 之外的任何内容类型。您可以在类级别声明共享的配置。 但是，与大多数其他请求映射属性不同，方法级别的声明是覆盖了，而不是扩展了类级别上的声明。

#### Producible Media Types

您可以根据请求的Accept header和 控制器生成的内容类型列表来缩小请求映射，如以下示例所示：
```java
@GetMapping(path = "/pets/{petId}", produces = "application/json;charset=UTF-8") 
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
```
媒体类型可以指定字符集。 支持否定表达式 - 例如，！text /plain 表示 “text /plain” 以外的任何内容类型。

您可以在类级别声明共享的produces属性。 但是，与大多数其他请求映射属性不同，在类级别使用时，方法级别会生成属性覆盖，而不是扩展类级别声明。

#### Parameters, headers

您可以根据请求参数条件缩小请求映射。  您可以测试是否存在请求参数（myParam）等等,以下示例显示如何测试特定值：
```java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
```
您还可以将其与请求header条件一起使用，如以下示例所示：
```java
@GetMapping(path = "/pets", headers = "myHeader=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
```

#### 处理HEAD, OPTIONS请求方法
HEAD 方法表示只取报文首部,OPTION 方法表示询问支持的请求方法

@GetMapping（和 @RequestMapping（method = HttpMethod.GET））透明地支持对HTTP HEAD方法进行请求映射。控制器方法无需更改。应用于 javax.servlet.http.HttpServlet 的response wrapper将确保将响应的 Content-Length 头设置为写入的字节数（不实际写入响应）。

@GetMapping（和 @RequestMapping（method = HttpMethod.GET））被隐式映射并支持 HTTP HEAD。实际上,处理 HTTP HEAD 请求就像是处理是 HTTP GET 一样，除了不是写入body,而是把写的字节数设置到 Content-Length 头。

默认情况下，不专门写处理OPTIONS请求的控制器方法,而是通过::设置 Allow 响应头::,并且统计OPTION 请求的路径所匹配到的控制器方法的 request method 类型来作为响应. 您也可以将 @RequestMapping 方法显式映射到 HTTP HEAD 和 HTTP OPTIONS，但在常见情况下这不是必需的。

#### Explicit Registrations

您可以以编程方式注册handler方法，可以将其用于动态注册或 更高级情况，例如不同 URL 下的同一handler的不同实例。 以下示例注册一个 handler方法：
```java
@Configuration
public class MyConfig {
    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) 
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); 

        Method method = UserHandler.class.getMethod("getUser", Long.class); 

        mapping.registerMapping(info, handler, method); 
    }

}
```
该方法首先创建了一个 请求映射info,然后根据反射取得一个方法 f,最后把 info和 f绑定在一起. 其中 userhandler是用户自定义的 Controller 类
![](Spring%20MVC%20Document/F8185FB1-02EE-4C73-ABA3-7FB6D18D05A5.png)

### 1.3.3. Handler Methods

@RequestMapping 处理程序方法具有灵活的签名，可以从一系列受支持的控制器方法参数和返回值中进行选择。下面的表列出了可以使用的参数
![](Spring%20MVC%20Document/FEB59D47-FD27-4AAD-89C1-0C283DD1BF74.png)
![](Spring%20MVC%20Document/5685CF1C-200B-4617-A1D1-439F3B8D8E42.png)
![](Spring%20MVC%20Document/021B1E57-95CA-4071-B466-F3E0234CDC97.png)
看一下最后一条,如果一个声明的方法参数不匹配之前的类型,并且是个简单类型,将被解析为相当于带@RequestParam,否则会被解析为相当于带@ModelAttribute

下面的表列出了可以的返回值类型
![](Spring%20MVC%20Document/76E8570A-ECCB-47F2-9042-ACF19CD5877D.png)
![](Spring%20MVC%20Document/24A7A53C-939F-4B4A-BF7C-72DDEB3DD933.png)
![](Spring%20MVC%20Document/05FE3996-2C4A-4F2B-95F7-15A043966A91.png)
后面将详细介绍这些参数

#### Type Conversion

如果参数声明不是 String类型 ，则那些基于String输入的某些控制器参数（例如 @ RequestParam，@ RequestHeader，@ PathVariable，@ MatrixVariable 和 @CookieValue）可能需要进行类型转换。

比如我有一个@RequestParam SchemaRequest sq,那么就需要类型转换. 类型转换基于配置好的 converters 自动执行, 默认情况下，已经支持了支持简单类型（int，long，Date ,URI,Local和其他等等）。自定义类型转换的方法是使用aWebDataBinder(see [DataBinder](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-initbinder) ) 或者实现Formatter,并且把它注册到FormattingConversionService.

#### Matrix Variables

RFC 3986 讨论了path segments中的name-value对。 在 Spring MVC 中，“矩阵变量”也是一种URI path parameters.。

矩阵变量可以出现在任何path segment中，每个变量(k)用则分号分隔，多个值(v)用逗号分隔（例如，/cars; color = red，green; year = 2012）。 也可以通过重复的变量名称指定多个值（例如，color = red; color = green; color = blue）。

如果 URL 预计包含矩阵变量，则控制器方法的请求映射必须使用 URI 变量来屏蔽该变量内容，并确保请求可以成功匹配，而与矩阵变量顺序和存在无关。 以下示例使用矩阵变量：
```java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11
}
```

鉴于所有路径段都可能包含矩阵变量，您有时可能需要消除歧义,保证矩阵变量能找到应该所在的路径段。以下示例说明了如何执行此操作：
```java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
```

要获取所有矩阵变量，可以使用 MultiValueMap，如以下示例所示：
```java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
```

请注意，您需要手动启用矩阵变量的使用。 在 MVC 的Java 配置中，您需要设置UrlPathHelper,将其removeSemicolonContent = false 设置为 UrlPathHelper。 

#### @RequestParam

您可以使用 @RequestParam 将 Servlet 请求参数（即**查询参数或表单数据**）绑定到控制器中的方法参数。
```java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    // ...

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { 
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }
    // ...
}
```
默认情况下，需要使用此注解的方法参数，但您可以通过将 @RequestParam 注解的 required 标志设置为 false , 或通过使用 java.util.Optional wrapper来声明参数来指定方法参数是可选的。

::将参数类型声明为数组或列表允许为同一参数名称解​​析多个参数值。::

当 @RequestParam 注解声明为 Map <String，String> 或 MultiValueMap <String，String > 时，如果注解中未指定参数名称，则会使用每一个请求里给出的参数名称及其参数值填充map。(最好不要用 map 作为请求参数)

默认情况下，::任何简单值类型的参数（由 BeanUtils＃isSimpleProperty 确定）并且不被任何其他参数解析器解析到，都被视为使用 @RequestParam 进行注解。::

而执行复杂类型的默认绑定,如果不是@requestBody 的话,那就用@ModelAttribute 就好,参见 [Spring MVC and the @ModelAttribute Annotation](bear://x-callback-url/open-note?id=E540753D-6630-4CD0-A756-59330ACE4DF2-557-0000006DF4B4CBEA) ,spring 根据命名 convention 配置了绑定规则(按名字填充字段); 另外,若是需要复杂的自定义绑定,比如把(CNY25)这样一个串绑定为一个 Money 对象,而不是标准的 Money JSON 串形式,则需要你自己定义Converter,Formatter 等了.

#### @RequestHeader

您可以使用 @RequestHeader 注解将请求标头绑定到控制器中的方法参数。
```textile
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
```
以下示例获取 Accept-Encoding 和 Keep-Alive 标头的值：
```java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
```

在 Map <String，String>，MultiValueMap <String，String > 或 HttpHeaders 参数上使用 @RequestHeader 注解时，将使用所有header 的值来填充映射。

#### @CookieValue

您可以使用 @CookieValue 注解将 HTTP cookie 的值绑定到控制器中的方法参数。考虑使用以下 cookie 的请求：
```
JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84
```

The following example shows how to get the cookie value:
```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
```
如果目标方法参数类型不是 String，则自动应用类型转换

#### @ModelAttribute

[Spring MVC and the @ModelAttribute Annotation](bear://x-callback-url/open-note?id=E540753D-6630-4CD0-A756-59330ACE4DF2-557-0000006DF4B4CBEA)

您可以在方法参数上使用 @ModelAttribute 注解来从model访问attribute，::如果model 中不存在此 attribute不存在则将其instantiated到 model 中::。 model attribute还能匹配那些名称与其类中的字段名称相匹配的请求参数。 这称为数据绑定，它使您不必自己去parsing and converting individual query parameters and form fields。 以下示例显示了如何执行此操作：
```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { }
```
Pet这个类具有ownerId 以及 petId 两个字段,这里通过@ModelAttribute就将其绑定成了一个 Pet 对象.

上面的 Pet 实例构造解析如下：
* from the model,如果已经使用 Model 添加了这个变量。
* 通过使用 @SessionAttributes 从 HTTP 会话获取。
* From a URI path variable passed through aConverter（参见下一个示例）。
* 从默认构造函数的调用。
* 从调用具有与请求参数匹配的参数的 “主构造函数”。 参数名称通过 JavaBeans @ConstructorProperties 或字节码中的运行时的runtime-retained parameter names确定。(即上面例子的方法)

虽然通常使用 Model 来使用属性填充模型，但另一种替代方法是靠 Converter <String，T> 和使用URI 的convention。 在以下示例中，model attribute的名字account 匹配到了URI 路径变量中的名字=> account，并通过自己写的Converter <String，Account> 来加载帐户：
```java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
```
获取模型属性实例后，将会执行::数据绑定::。 WebDataBinder 类将 Servlet 请求参数名称与目标对象中的的字段名称进行匹配。 在必要时，在应用类型转换后填充匹配字段。 

数据绑定可能导致错误。 默认情况下，会引发 BindException。 但是，要在控制器方法中检查此类错误，可以在 @ModelAttribute的 ::immediately next::添加一个 BindingResult 参数，如以下示例所示：
```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```
在某些情况下，您可能希望在**没有数据绑定**的情况下访问模型属性。 对于这种情况，您可以将模型注入控制器并直接访问它，或者设置 @ModelAttribute（binding = false），如下例所示：
```java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(@Valid AccountForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) { 
    // ...
```
上面的例子不会给 account 做数据绑定,因为url 中实际上只给了一个 accountId,而 acoount 实际上是根据 id 从数据库查出来的

通过添加 javax.validation.@Valid 注解或 Spring 的 @Validated 注解，您可以在数据绑定后自动应用验证。 以下示例显示了如何执行此操作：
```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

请注意，使用 @ModelAttribute 是可选的（例如，设置其属性）。 默认情况下，任何::**非**简单值类型的参数并且未被任何其他参数解析器解析，都被视为使用 @ModelAttribute 进行注解。::

#### @SessionAttributes(Insert)
@SessionAttributes ::用于在多个请求之间,使用会话来**存储**模型attribute::。 它是一个类型级别的注解，来声明特定控制器使用的会话属性。 这通常列出模型属性的名称或模型属性的类型，这些属性将被透明地存储在会话中以供后续访问请求使用。

以下示例使用 @SessionAttributes 注解：
```java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {
    // ...
}
```

当在第一个请求中，名称为 pet 的attribute 被添加到模型中，它将被自动保存到 HTTP session 中去。 它保持不变，直到另一个控制器方法使用 SessionStatus 方法参数来清除存储，如下例所示：
```java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) {
        if (errors.hasErrors) {
            // ...
        }
            status.setComplete(); 
            // Clearing thePetvalue from the Servlet session.
        }
    }
}
```
一般来说 @SessionAttributes 设置的参数只用于暂时的传递，而不是长期的保存，长期保存的数据还是要放到 Session 中。


#### @SessionAttribute(Read)
如果您需要访问全局管理的,之前存在的session属性,并且现在可能存在或不存在，则可以对方法参数使用 @SessionAttribute 注解，如 以下示例显示：
```java
@RequestMapping("/")
public String handle(@SessionAttribute User user) { 
    // ...
}
```
对于需要添加或删除会话属性的用例，请考虑将 WebRequest 或 javax.servlet.http.HttpSession 注入控制器方法。

要在会话中temporary storage模型属性作为控制器工作流的一部分，请考虑使用 @SessionAttributes.

#### @RequestArrtibute
::与 @SessionAttribute 类似::，您可以使用 @RequestAttribute 注解来访问先前创建的,预存在的请求属性（例如，通过 Servlet 过滤器或 HandlerInterceptor）：
```
@GetMapping("/")
public String handle(@RequestAttribute Client client) { 
    // ...
}
```

#### Redirect Attributes/Flash Attributes

默认情况下，在重定向 URL 中,所有model attributes都被视为URI模板变量而暴露出来。其余属性中，原始类型或原始类型的集合或数组的属性会自动append为查询参数。

如果专门为重定向准备了模型实例，则将原始类型属性作为query参数附加也许是期望的结果。 但是，在annotated 控制器中，模型可以包含为渲染目的而添加的其他属性（例如，下拉字段值）。

为了避免在 URL 中使用此类属性，@RequestMapping 方法可以声明 RedirectAttributes 类型的参数，并使用它来指定可供 RedirectView(参见1.1.8)将使用的确切属性。如果方法做了重定向，则使用 RedirectAttributes 的内容。否则，使用model的内容。

RequestMappingHandlerAdapter 提供了一个名为 ignoreDefaultModelOnRedirect 的标志，您可以使用该标志指示如果控制器方法重定向，则永远不应使用默认模型的内容。 MVC Java 配置都将此标志设置为 false，以保持向后兼容性。但是，对于新应用程序，我们建议将其设置为 true。

请注意，扩展重定向 URL 时，当前请求中的 URI template variable是自动可用的，您无需通过 Model 或 RedirectAttributes 显式添加它们。以下示例显示如何定义重定向：
```java
@PostMapping("/files/{path}")
public String upload(...) {
    // ...
    return "redirect:files/{path}";
}
```
将数据传递到重定向目标的另一种方法是使用 flash attributes。与其他重定向属性不同，Flash 属性保存在 HTTP 会话中（因此，不会出现在 URL 中）

#### Multipart

启用 MultipartResolver 后，将解析具有 multipart /form-data 的 POST 请求的内容，并将其作为常规请求参数进行访问。 以下示例访问一个常规表单字段和一个上载文件：
```java
@Controller
public class FileUploadController {
    @PostMapping("/form")
    public String handleFormUpload(@RequestParam("name") String name,
            @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```
将参数类型声明为 List <MultipartFile> 允许为同一参数名称解析多个文件。

当 @RequestParam 注解声明为 Map <String，MultipartFile> 或 MultiValueMap <String，MultipartFile > 时，如果注解中未指定参数名称，则会使用每个给定的参数名称的multipart file填充map。

您还可以将multipart content用作绑定到的数据的一部分。 例如，前面示例中的表单和表单中的文件可以是form object的字段，如以下示例所示：
```java
class MyForm {

    private String name;

    private MultipartFile file;

    // ...
}

@Controller
public class FileUploadController {
    @PostMapping("/form")
    public String handleFormUpload(MyForm form, BindingResult errors) {
        if (!form.getFile().isEmpty()) {
            byte[] bytes = form.getFile().getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```

还可以在 RESTful 服务中,从非浏览器客户端提交MultiPart请求。以下示例展示了一个带有 JSON 的文件：
```java
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
    "name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... File Data ...
```
您可以使用 @RequestParam 作为 String 访问 “元数据” 部分，但您可能希望它从 JSON 反序列化（类似于 @RequestBody）。 在使用 HttpMessageConverter 转换后，其实可以使用 ::@RequestPart:: 注解访问MultiPart：
```java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata,
        @RequestPart("file-data") MultipartFile file) {
    // ...
}
```
您可以将 @RequestPart 与 @Valid 结合使用，或使用 Spring 的 @Validated 注解，这两种注解都会导致应用标准 Bean 验证。 默认情况下，验证错误会导致 MethodArgumentNotValidException，并将其转换为 400（BAD_REQUEST）响应。 或者，您可以通过 Errors 或 BindingResult 参数在控制器内本地处理validation错误，如以下示例所示：
```java
@PostMapping("/")
public String handle(@Valid @RequestPart("meta-data") MetaData metadata,
        BindingResult result) {
    // ...
}
```

#### @RequestBody

您可以使用 @RequestBody 注解通过 HttpMessageConverter 将请求主体读取并反序列化为 Object。 以下示例使用 @RequestBody 参数：(::意思说就通过HttpMessageConverter就能按默认行为转换自定义类型::)
```java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
```
您可以使用 MVC 配置的 “ [Message Converters](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-config-message-converters) ” 选项来配置或自定义消息转换。

您可以将 @RequestBody 与 javax.validation.Valid 或 Spring 的 @Validated 注解结合使用，这两种注解都会导致应用标准 Bean 验证。 默认情况下，验证错误会导致 MethodArgumentNotValidException，并将其转换为 400（BAD_REQUEST）响应。 或者，您可以通过 Errors 或 BindingResult 参数在控制器内本地处理验证错误.

#### HttpEntity

HttpEntity 与使用 @RequestBody 或多或少相同，但它基于一个a container object,其expose请求header和body(能用它来获取 header 信息)。 以下清单显示了一个示例：
```java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
    // ...
}
```

#### @ResponseBody

您可以在方法上使用 @ResponseBody 注解，以通过 HttpMessageConverter 将返回值序列化到响应主体(并不一定是 JSON,也可以根据内容协商这样的手动返回 JSON 或者 XML 等等 body 内容类型)。 以下清单显示了一个示例：
```java
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
    // ...
}
```
类级别也支持 @ResponseBody，在这种情况下，它由所有控制器方法继承。 这就是 @RestController 的效果. 您可以将 @ResponseBody 与reactive types一起使用。

您可以使用 MVC 配置的 “ [Message Converters](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-config-message-converters) ” 选项来配置自定义的消息转换。您可以将 @ResponseBody 方法与JSON serialization views结合使用。 有关详细信息，请参阅 [Jackson JSON](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-jackson) 。

举个示例,在一个 Spring Boot 程序中,若想要返回 XML,在 controller 里你无需做什么,但是在要返回的 pojo 类上加上如上标签
![](Spring%20MVC%20Document/A4EC22BA-ECE0-4862-8780-8F172D0F686D.png)
另外引入JACKSON 对 XML 的支持依赖
![](Spring%20MVC%20Document/C2073317-BC8B-4F91-8D37-830D8C945C8D.png)
并且在 request 指定 Accept 为 application/xml,就可以了,可以参见[79. Spring MVC](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-spring-mvc.html#howto-write-an-xml-rest-service)

#### ResponseEntity

ResponseEntity 与 @ResponseBody 类似，但具有status和header。 例如：
```java
@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).build(body);
}
```

#### Jackson JSON

Spring 为 Jackson JSON 库提供支持。这节主要讲类似如何自定义 view,使得 Json 不渲染对象的全部字段之类的

###### JSON Views
Spring MVC 为  [Jackson’s Serialization Views](http://wiki.fasterxml.com/JacksonJsonViews) 提供内置支持，允许**仅渲染对象中所有字段的子集**。 要将其与 @ResponseBody 或 ResponseEntity 控制器方法一起使用，您可以使用 Jackson 的 @JsonView 注解来激活一个serialization view class，如以下示例所示：
```java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```
@JsonView 允许一组视图类，但每个控制器方法只能指定一个。 如果需要激活多个视图，you can use a composite interface.

对于依赖于视图解析的Controller(非 Rest的)，可以将序列化视图类添加到Model中，如以下示例所示：
```java
@Controller
public class UserController extends AbstractController {
    @GetMapping("/user")
    public String getUser(Model model) {
        model.addAttribute("user", new User("eric", "7!jd#h23"));
        model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);
        return "userView";
    }
}
```

### 1.3.4. Model

您可以在多种地方使用 @ModelAttribute 注解：
* 在 @RequestMapping 方法中的方法参数，用于从模型::创建或访问:: 对象 并通过 WebDataBinder 将其绑定到请求。(前面讲的就是这种)
* 作为 @Controller 或 @ControllerAdvice 类中的方法级注解，以在任何 @RequestMapping 方法调用之前初始化模型。
* 在 @RequestMapping 方法上标记其返回值是一个模型属性。

本节讨论 @ModelAttribute 方法 - 前面列中的第二项。 控制器可以包含任意数量的 @ModelAttribute 方法。 在同一控制器中的 @RequestMapping 方法被调用之前,将调用所有这些方法来初始化 model 内容。 @ModelAttribute 方法也可以通过 @ControllerAdvice 在控制器之间共享。

@ModelAttribute 方法具有灵活的方法签名。 它们支持许多与 @RequestMapping 方法相同的参数，除了@ModelAttribute 本身或任何与request body相关的任何内容。

以下示例显示了 @ModelAttribute 方法：(number 这个@RequestParam 从 request 中取)
```java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
```
以下示例仅添加一个属性(与上面那个等价,就是为了添加 model attribute)：
```JAVA
@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountRepository.findAccount(number);
}
```

您还可以将 @ModelAttribute 用作 @RequestMapping 方法的方法级注解，在这种情况下，@RequestMapping 方法的返回值将被解读为 model attribute。 (也就是说,::默认情况下, 一个 controller 方法,返回值若不是 String,会被当成要添加的model attribute::)这通常不是必需的，因为它是 HTML 控制器的默认行为，除非返回值是一个 String(会被解析为 viewname)。 @ModelAttribute 还可以自定义模型属性名称，如以下示例所示：
```java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
```

### 1.3.5. DataBinder

@Controller 或 @ControllerAdvice 类可以使用 @InitBinder 方法来初始化 ::**WebDataBinder**:: 的实例，而这些方法又可以：
* 将请求参数（即表单或查询数据）绑定到模型对象。
* 将基于字符串的请求值（例如请求参数，路径变量，标题，cookie 等）转换为controller method arguments的目标类型。
* 在呈现 HTML 表单时将模型对象值格式化为 String 值。

@InitBinder 方法可以注册controller-specific的 java.bean.PropertyEditor 或 Spring Converter 和 Formatter 的组件。 此外，您可以使用 MVC config在全局共享的 FormattingConversionService 中注册 Converter 和 Formatter。

@InitBinder 方法支持许多与 @RequestMapping 方法相同的参数，但 @ModelAttribute（命令对象）参数除外。 通常，它们使用 WebDataBinder 参数（用于注册）和 void 返回值声明。 以下清单显示了一个示例：
```java
@Controller
public class FormController {

    @InitBinder 
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }
    // ...
}
```
上面的例子注册了一个 Editor,作用是把 Date 类型和一个 sdf 绑定在一起,到时候传入”yyyy-MM-dd”的串作为 controller 参数,就能被解析为 data 类型了.

或者，当您通过共享的 FormattingConversionService 使用基于 Formatter 的配置时，您可以复用相同的方法并注册controller-specific的 Formatter，如以下示例所示：
```java
@Controller
public class FormController {

    @InitBinder 
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }
    // ...
}
```
![](Spring%20MVC%20Document/822DD15B-A2B1-41E5-9060-51692ADEEE91.png)

### 1.3.6. Exceptions

@Controller 和 @ControllerAdvice 类可以使用 @ExceptionHandler 方法来处理来自controller方法的异常，如下例所示：
```java
@Controller
public class SimpleController {
    // ...

    @ExceptionHandler
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }
}
```
该异常能匹配上top-level异常（即直接抛出的 IOException）,也能匹配上immediate cause within a top-level wrapper exception（例如，包含在 IllegalStateException 内的 IOException）(也就是说, 若IOException 是IllegalStateException这个 ::runtime exception::的 immediate cause,那么这个 Handler 也能匹配上这次异常)。

对于匹配到的异常类型，最好像上面这个例子一样,将目标异常声明为方法参数。 当有多个异常方法匹配上时，root异常匹配通常优先于cause异常。 更具体地说，ExceptionDepthComparator 会根据抛出的异常类型的深度对异常进行排序。

或者，使用注解可以声明以缩小要匹配的异常类型，如以下示例所示：
```
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(IOException ex) {
    // ...
}
```
您甚至可以使用具有very generic argument signature的特定异常类型列表，如以下示例所示(参数中使用了 Exception)：
```java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(Exception ex) {
    // ...
}
```

我们通常建议您在参数签名中尽可能具体，减少 root 和 cause 异常类型之间不匹配的可能性。并考虑将多匹配方法分解为多个单独的 @ExceptionHandler 方法，每个方法通过其签名匹配单个特定异常类型。

在多 @ControllerAdvice 环境中，我们建议在 @ControllerAdvice 上声明您的primary root exception mappings，并使用相应的顺序进行order排序。虽然root异常匹配优先于某个cause，但这是在给定控制器或 @ControllerAdvice 类的方法中定义的,然后,我没拿需要注意到@ControllerAdvice 是有顺序的, 这意味着优先级较高的 @ControllerAdvice bean 上的cause匹配甚至能优先于较低优先级的 @ControllerAdvice bean 上的任何匹配（例如root匹配）。

::Spring MVC 中对 @ExceptionHandler 方法的支持是基于 DispatcherServlet 级别的 HandlerExceptionResolver 机制构建的。::

#### Method Arguments

@ExceptionHandler 方法支持以下参数：
![](Spring%20MVC%20Document/49FB8E82-7E5A-4EE1-AA80-D66BFA19F490.png)
![](Spring%20MVC%20Document/0151C494-77F0-4CB7-A47F-4E3ACD0F096A.png)
可以看出,还可以获取请求的一些信息,或者是 session 的一些信息.

#### Return Values

@ExceptionHandler 方法支持以下返回值：
![](Spring%20MVC%20Document/8C19CAFA-3D78-4E5E-9B27-9BAB1402E5E9.png)
![](Spring%20MVC%20Document/CD048A41-8476-405D-BCDB-3FBBB6D8EB97.png)
可以直接返回 responsebody,也可以做响应的视图呈现,或者是返回void,在同时存在 ServletResponse 这样的参数的情况下,认为已经完全处理好了响应,或者不存在这样的参数情况下,认为 void 表示响应是”no body”的.

#### REST API exceptions
::REST 服务的一个常见要求是在响应正文中包含错误详细信息::。 Spring Framework 不会自动执行此操作，因为响应正文中的错误详细信息的表示是特定于应用程序的。 ::但是，@ RestController 可以使用带有 ResponseEntity 返回值的 @ExceptionHandler 方法来设置响应的状态和正文。:: 这些方法也可以在 @ControllerAdvice 类中声明，以便全局应用它们。

在响应body中实现具有错误详细信息的 global exception handling  的应用应考虑扩展 ::ResponseEntityExceptionHandler::，它提供对 Spring MVC 引发的异常的处理，并提供钩子来定制响应body。 要使用它，请创建 ResponseEntityExceptionHandler 的子类，使用 @ControllerAdvice 注解它，覆盖必要的方法，并将其声明为 Spring bean。

### 1.3.7. Controller Advice

通常，@ExceptionHandler，@InitBinder 和 @ModelAttribute 方法适用于其声明处的@Controller 类。 如果您希望这些方法更全局地应用起来（跨controller），则可以在标有 @ControllerAdvice 或 @RestControllerAdvice 的类中声明它们。

@ControllerAdvice 用 @Component 标记，这意味着可以通过component scan将这些类注册为 Spring bean。 @RestControllerAdvice 也是一个使用 @ControllerAdvice 和 @ResponseBody 标记的元注解，这实际上意味着 @ExceptionHandler 方法通过消息转换呈现给响应body。

在启动时，the infrastructure classes for @RequestMapping and @ExceptionHandler methods检测 @ControllerAdvice 类型的 Spring bean，然后在运行时应用它们的方法。 全局 @ExceptionHandler 方法applied after local ones(::也就是说,controller 内定义的@ExceptionHadler 优先级更高::)。 相比之下，全局的@ModelAttribute 和 @InitBinder 方法比本地的优先级更高。

默认情况下，@ ControllerAdvice 方法适用于每个请求（即所有控制器），但您可以通过使用注解上的属性将其缩小到一个controoler子集，如以下示例所示：
```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```
前面示例中的选择器在**运行时**进行评估，如果过于广泛使用，可能会对性能产生负面影响。

## 1.4 URI Links
本节介绍 Spring Framework 中可用于处理 URI 的各种选项。

### 1.4.1. UriComponents

UriComponentsBuilder 有助于使用变量从 URI 模板构建 URI，如以下示例所示：
```java
UriComponents uriComponents = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")  
        .queryParam("q", "{q}")  
        .encode() 
        .build(); 

URI uri = uriComponents.expand("Westin", "123").toUri(); 
```
encode()将把URI template and URI variables进行编码. 前面的示例可以合并到一个调用链中，并使用buildAndExpand进行，如以下示例所示：
```java
URI uri = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("Westin", "123")
        .toUri();
```
您可以通过直接转到 URI（隐式做了encode）来进一步缩短它，如下例所示：
```JAVA
URI uri = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```
您还可以在最初提供完整的 URI 模板以进一步缩短它，如下例所示：
```java
URI uri = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}?q={q}")
        .build("Westin", "123");
```

### 1.4.2. UriBuilder

上例中的UriComponentsBuilder 实现了 UriBuilder。 其实,您可以使用 UriBuilderFactory 来创建一个 UriBuilder。 UriBuilderFactory 和 UriBuilder 一起提供了一种可插拔的机制，可以根据共享配置（例如基本 URL，encoding preferences…）来从 URI 模板构建 URI。

您可以把RestTemplate 和 WebClient 和UriBuilderFactory一起用来以自定义如何实现 URI。 DefaultUriBuilderFactory 是 UriBuilderFactory 的默认实现, 它在内部其实就是使用了 UriComponentsBuilder 并exposes shared configuration options。

以下示例显示如何配置 RestTemplate：
```java
String baseUrl = "http://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
```
上面的例子相当于为RestTemplate 绑定了一个根据baseUrl来创建 URL 的 builder

此外，您还可以直接使用 DefaultUriBuilderFactory。 它和使用 UriComponentsBuilder也累死，但它不是静态工厂方法，而是一个actual instance that holds configuration and preferences，如下例所示：
```java
String baseUrl = "http://example.com";
DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory(baseUrl);

URI uri = uriBuilderFactory.uriString("/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```

### 1.4.3. URI Encoding

UriComponentsBuilder 在两个级别暴露编码选项：
* 使用UriComponentsBuilder.encode(): 首先对 URI 模板进行预编码，然后在expand时严格编码 URI variable。
* 使用UriComponents.encode() ：expand URI variable后再对 URI component进行编码。

这两个选项都使用转义的octets替换非 ASCII 和非法字符。 但是，第一个还会替换出现在 URI variable中的保留字符。
```
考虑 “;”，这在路径中是合法的,但具有保留意义。 第一种方法在 URI 变量中吧 “;” 替换为“％3B” 但在 URI 模板中并不会替换。 相比之下，第二个选项永远不会替换 “;”，因为它是路径中的合法字符。
```

对于大多数情况，第一个选项可能会给出预期结果，因为它将 URI 变量视为完全编码的不透明数据，而选项 2 仅在 想在URI 变量故意包含一些保留字符时才有用。

以下示例使用第一个选项：
```java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
            .queryParam("q", "{q}")
            .encode()
            .buildAndExpand("New York", "foo+bar")
            .toUri();

    // Result is "/hotel%20list/New%20York?q=foo%2Bbar"
```

estTemplate 通过 UriBuilderFactory 的策略在内部expand和编码 URI 模板。 可以配置自定义策略。 如下例所示：
```java
String baseUrl = "http://example.com";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl)
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);//这里就是 encode策略

// Customize the RestTemplate..
RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
```

DefaultUriBuilderFactory 在内部使用了 UriComponentsBuilder 来expand和encode URI 模板。就像上面的例子一样,可以配置编码方法(setEncodingMode)，可以选择编码模式之一：
* TEMPLATE_AND_VALUES(第一种编码选项)：使用 UriComponentsBuilder.encode（）对 URI 模板进行预编码，并在exnpand时严格编码 URI 变量。
* VALUES_ONLY：不对 URI 模板进行编码，而是通过 UriUtils.encodeUriUriVariables() 来仅对 URI 变量应用严格编码。
* URI_COMPONENTS(第二种编码选项)：使用 UriComponents＃encode（）在 URI expand展开后对 URI 组件值进行编码。
* NONE：不应用编码。

出于历史原因和向后兼容性，RestTemplate 默认配置为 EncodingMode.URI_COMPONENTS,但是建议改为TEMPLATE_AND_VALUES

### 1.4.4. Relative Servlet Requests

您可以使用 ServletUriComponentsBuilder 创建relative to当前请求的 URI，如以下示例所示：
```java
HttpServletRequest request = ...

// Re-uses host, scheme, port, path and query string...
ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromRequest(request)
        .replaceQueryParam("accountId", "{id}").build()
        .expand("123")
        .encode();
```
您可以创建relative to context path的 URI，如以下示例所示：
```java
// Re-uses host, port and context path...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromContextPath(request)
        .path("/accounts").build()
```
您还可以创建相对于 Servlet 的 URI（例如，/main/ *），如以下示例所示：
```java
// Re-uses host, port, context path, and Servlet prefix...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromServletMapping(request)
        .path("/accounts").build()
```

### 1.4.5. Links to Controllers

Spring MVC 提供了一种为控制器方法准备links的机制。 例如，以下 MVC 控制器允许links创建：
```java
@Controller
@RequestMapping("/hotels/{hotel}")
public class BookingController {

    @GetMapping("/bookings/{booking}")
    public ModelAndView getBooking(@PathVariable Long booking) {
        // ...
    }
}
```
您可以通过按控制器法官法名称引用方法来准备链接，如以下示例所示：
```java
UriComponents uriComponents = MvcUriComponentsBuilder
    .fromMethodName(BookingController.class, "getBooking", 21).buildAndExpand(42);

URI uri = uriComponents.encode().toUri();
```
在这个示例中，我们提供了实际的方法参数值（在本例中，long 值：21），用作路径变量并插入到 URL 中。 此外，我们提供值 42 来填充任何剩余的 URI 变量，例如从class级request mapping继承的 hotel 变量。 如果还有更多参数，我们可以为 URL 不需要的参数提供 null值。 通常，只有 @PathVariable 和 @RequestParam 参数与构造 URL 相关。

还有其他方法可以使用 *MvcUriComponentsBuilder*。 例如，您可以使用通过代理进行类似于mock testing的技术，以避免按名称引用控制器方法，如以下示例所示,就没有具体地使用到getBooking()这个控制器方法名
```
UriComponents uriComponents = MvcUriComponentsBuilder
    .fromMethodCall(on(BookingController.class).getBooking(21)).buildAndExpand(42);

URI uri = uriComponents.encode().toUri();
```
上面的示例在 MvcUriComponentsBuilder 中使用静态方法。 在内部实现上，它们依赖 *ServletUriComponentsBuilder* 从scheme, host, port, context path, and servlet path of the current request来准备基本 URL。 

注意,当 controoler方法用于使用fromMethodCall 创建来链接时，它们的签名是受到限制的。 除了需要正确的参数签名之外，返回类型还存在技术限制，因此返回类型不能是 final。 特别是，以String 返回类型来返回视图名称在此处不起作用。 您应该使用 ModelAndView来返回视图。

## 1.5 Asynchronous Requests
Spring MVC 与 Servlet 3.0 中的异步请求处理有着良好的集成：
* DeferredResult 和 Callable 在控制器方法中作为返回值，这为sigle异步返回值提供基本支持。
* 控制器可以 [stream](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-async-http-streaming) 多个值，包括 [SSE](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-async-sse) 和 [raw data](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-async-output-stream) 。
* 控制器可以使用reactive clients并返回 [reactive types](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-async-reactive-types) 以进行响应处理。

### 1.5.1.DeferredResult

一旦在 Servlet 容器中启用了异步请求处理功能，控制器方法就可以使用 DeferredResult 来包装任何支持的控制器方法返回值，如以下示例所示：
```java
@GetMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
    DeferredResult<String> deferredResult = new DeferredResult<String>();
    // Save the deferredResult somewhere..
    return deferredResult;
}

// From some other thread...
deferredResult.setResult(data);
```
Controller可以从不同的线程来异步生成返回值(执行控制器方法中实际的任务) - 例如，响应外部事件（JMS 消息），计划任务或其他事件。事实上,这会导致Controller 的这个请求线程并不阻塞在这里,可以去干别的事,但是 client 拿不到返回,直到controller 方法的实际任务执行完.

### 1.5.2. Callable

控制器可以使用 java.util.concurrent.Callable 来包装任何受支持的返回值，如以下示例所示： 
```java
@PostMapping
public Callable<String> processUpload(final MultipartFile file) {

    return new Callable<String>() {
        public String call() throws Exception {
            // ...
            return "someView";
        }
    };

}
```
然后可以通过TaskExecutor来运行controller 里实际给定的任务来获取返回值。

### 1.5.3. Processing

以下是 Servlet 异步请求处理的简要概述：
* 可以通过调用 *request.startAsync（）*将 *ServletRequest* 置于异步模式。这样做的主要作用是 Servlet（以及任何过滤器）可以exit，但response仍然保持打开状态,以便稍后处理完成。
* 对 request.startAsync（）的调用返回一个 *AsyncContext*，您可以使用它来进一步控制异步处理。例如，它提供了 dispatch()，这类似于 Servlet API 中的 forward，除了它允许app resume request processing on a Servlet 容器线程。
* ServletRequest 提供对当前 *DispatcherType* 的access，您可以使用它来区分”处理初始请求”，”异步dispatch”，”转发”和其他dispatcher types。

*DeferredResult* 处理的工作方式如下：
1. 控制器返回 DeferredResult 并将其保存在可以访问到的某个内存队列或列表中。
2. Spring MVC 调用 request.startAsync（）。
3. 同时，DispatcherServlet 和所有已配置的过滤器退出请求处理线程，但注意响应仍保持打开状态。
4. 计算好以后,应用程序从**某个别的线程**设置 DeferredResult，Spring MVC 将request调度回 Servlet 容器。
5. DispatcherServlet被再次调用，并使用异步的返回值继续处理 response。

*Callable*处理的工作原理如下：
1. 控制器返回 Callable。
2. Spring MVC 调用 request.startAsync（）并将 Callable 提交给 TaskExecutor bean，以便在单独的线程中进行处理。
3. 同时，DispatcherServlet 和所有过滤器退出 Servlet 容器线程，但响应仍保持打开状态。
4. 最终，Callable 生成一个结果，Spring MVC 将请求调度回 Servlet 容器以完成处理。
5. 再次调用 DispatcherServlet，并使用 Callable 中异步生成的返回值继续处理response。

有关更多背景和上下文，您还可以阅读在 Spring MVC 3.2 中引入异步请求处理支持的 [blog posts](https://spring.io/blog/2012/05/07/spring-mvc-3-2-preview-introducing-servlet-3-async-support) 。

使用 DeferredResult 时，可以选择是否把一个异常传入setResult 或 setErrorResult的调用。 在这两种情况下，Spring MVC 都会将请求发送回 Servlet 容器以完成处理,然后异常通过常规异常处理机制（例如，调用 @ExceptionHandler 方法）,nothing special。当您使用 Callable 时，会出现类似的处理逻辑，主要区别在于从 Callable 返回结果，或者由它引发异常。

拦截器HandlerInterceptor也可以是 *AsyncHandlerInterceptor* 类型，用于在初始请求开始异步处理之后来回调afterConcurrentHandlingStarted()。 或者,HandlerInterceptor 的实现还可以注册一个 *CallableProcessingInterceptor* 或 *DeferredResultProcessingInterceptor*，以更深入地集成到异步请求的生命周期里面（例如，处理超时事件）。 有关更多详细信息，请参阅 [AsyncHandlerInterceptor](https://docs.spring.io/spring-framework/docs/5.1.5.RELEASE/javadoc-api/org/springframework/web/servlet/AsyncHandlerInterceptor.html) 。

DeferredResult 提供了 onTimeout（Runnable）和 onCompletion（Runnable）回调。 有关更多详细信息，请参阅 DeferredResult 的  [javadoc](https://docs.spring.io/spring-framework/docs/5.1.5.RELEASE/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html) 。 Callable 可以替代 WebAsyncTask that exposes additional methods for timeout and completion callbacks.

### 1.5.4. HTTP Streaming

您可以将 DeferredResult 和 Callable 用于单个异步返回值。 如果要生成**多个异步值**并将其写入响应，该怎么办？ 本节介绍如何执行此操作。

您可以使用 *ResponseBodyEmitter* 作为返回值来生成对象流，其中每个对象都使用 HttpMessageConverter 进行序列化并写入response，如以下示例所示：
```java
@GetMapping("/events")
public ResponseBodyEmitter handle() {
    ResponseBodyEmitter emitter = new ResponseBodyEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
```
这里就不断地往emitter里send数据,相当于一个数据流,最后调用complete()完成返回值填充. 您还可以使用 ResponseBodyEmitter 作为 ResponseEntity 中的正文，以便自定义响应的status和header。

当emitter抛出 IOException 时（例如，如果远程客户端消失），应用程序不负责清理connection，是不应调用 emitter.complete 或 emitter.completeWithError的。 相反，servlet 容器会自动初始化一个 *AsyncListener* error notification，在其中 Spring MVC 会进行 completeWithError 调用, 转而，此调用会对应用程序执行一次最final ASYNC dispatch，在此期间，Spring MVC 将调用已配置的exception resolvers并完成请求。

SseEmitter（ResponseBodyEmitter 的子类）可以为  [Server-Sent Events](https://www.w3.org/TR/eventsource/) 提供支持，其中从服务器发送的事件根据 W3C SSE 规范进行格式化。 要从控制器生成 SSE 流，请返回 *SseEmitter*，如以下示例所示：
```java
@GetMapping(path="/events", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter handle() {
    SseEmitter emitter = new SseEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
```
虽然 SSE 是使用流式传输到浏览器的主要选项，但请注意 Internet Explorer 不支持 Server-Sent Events。 可以考虑将 Spring 的 [WebSocket messaging](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#websocket) with [SockJS fallback](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#websocket-fallback) transports一起使用,这个对浏览器适配性比较强。

有时，绕过消息convertion并直接stream到响应的OutputStream（例如，用于文件下载）是有用的。 您可以使用 StreamingResponseBody 返回类型来执行此操作，如以下示例所示：
```JAVA
@GetMapping("/download")
public StreamingResponseBody handle() {
    return new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream outputStream) throws IOException {
            // write...
        }
    };
}
```
您可以使用 StreamingResponseBody 作为 ResponseEntity 中的body来自定义响应的status和header。

### 1.5.7. Configuration

必须在 Servlet 容器级别才能启用异步请求处理功能。 MVC 配置还暴露了几个异步请求选项。

##### Servlet 容器
Filter 和 Servlet 声明具有一个asyncSupported 标志，需要将其设置为 true 来启用异步请求处理。此外，应声明Filter mappings 来处理 ASYNC javax.servlet.DispatchType。

在 Java 配置中，当您使用 *AbstractAnnotationConfigDispatcherServletInitializer* 初始化 Servlet 容器时，这将自动完成。而在 web.xml 配置中，您可以吧 <async-supported> true </async-supported > 添加到 DispatcherServlet 和 Filter 声明，并添加 < dispatcher> ASYNC </dispatcher > 以过滤映射。

##### Spring MVC
MVC 配置暴露了以下与异步请求处理相关的选项：
* Java 配置：在 WebMvcConfigurer 上使用 configureAsyncSupport 回调。
* XML配置：使用 <mvc：annotation-driven> 下的 < async-support > 元素。

您可以配置以下内容：
* 异步请求的默认超时值,如果未设置,就取决于基础 Servlet 容器（例如，Tomcat 上的 10 秒）。
* DeferredResultProcessingInterceptor 实现和 CallableProcessingInterceptor 实现。(处理异常)

请注意，您还可以在 DeferredResult，ResponseBodyEmitter 和 SseEmitter 上设置默认超时值。对于 Callable，您可以使用 WebAsyncTask 来提供超时值。

## 1.8 HTTP Caching
HTTP 缓存可以显着提高 Web 应用程序的性能。 HTTP 缓存基于 Cache-Control 响应header，随后是一些 conditional 的请求header（例如 Last-Modified 和 ETag）。 ::Cache-Control 建议私有（例如，浏览器）和公共（例如，代理）缓存如何 来缓存和重用得到的响应。:: 则 ETag 标头用于生成conditional请求，如果内容未更改，该请求可能导致 304（NOT_MODIFIED）状态,并且没有正文。 ETag 可以被视为 Last-Modified header的更复杂的继承者。

![](Spring%20MVC%20Document/B711B712-001D-4319-B227-D9F963D6CC6C.png)

本节介绍 Spring Web MVC 中可用的与 HTTP 缓存相关的选项。

### 1.8.1.CacheControl

CacheControl 支持配置与 Cache-Control header相关的设置，并可以在许多地方作为参数：
* WebContentInterceptor
* WebContentGenerator
* Controller
* 静态资源

虽然 RFC 7234 描述了 Cache-Control response header的所有可能的directives，但 CacheControl 类型采用面向case的方法，该方法侧重于常见场景,下面创建了三种不同的CacheControl：
```java
// Cache for an hour - "Cache-Control: max-age=3600"
CacheControl ccCacheOneHour = CacheControl.maxAge(1, TimeUnit.HOURS);

// Prevent caching - "Cache-Control: no-store"
CacheControl ccNoStore = CacheControl.noStore();

// Cache for ten days in public and private caches,
// public caches should not transform the response
// "Cache-Control: max-age=864000, public, no-transform"
CacheControl ccCustom = CacheControl.maxAge(10, TimeUnit.DAYS).noTransform().cachePublic();
```
WebContentGenerator 还接受一个更简单的 cachePeriod 属性（以秒为单位定义），其工作方式如下：
* -1 值不会生成 Cache-Control 响应头。
* 0 值可防止缓存,相当于使用 “Cache-Control：no-store” 指令，。
* n> 0 值通过使用 'Cache-Control：max-age = n' 将响应缓存 n 秒。

### 1.8.2. Controllers
控制器可以添加对 HTTP 缓存的显式支持。 我们建议这样做，因为资源的 lastModified 或 ETag 值需要先计算才能与条件request header进行比较。 控制器可以向 ResponseEntity 添加 ETag 标头和 Cache-Control 设置，如以下示例所示：
```java
@GetMapping("/book/{id}")
public ResponseEntity<Book> showBook(@PathVariable Long id) {
    Book book = findBook(id);
    String version = book.getVersion();

    return ResponseEntity
            .ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
            .eTag(version) // lastModified is also available
            .body(book);
}
```
如果与conditional request headers的比较表明内容并未更改，则前面的示例将发送带有空主体的 304（NOT_MODIFIED）响应。 否则，ETag 和 Cache-Control 标头将添加到响应中。

您还可以对控制器中的conditional request headers进行检查，如以下示例所示：
```java
@RequestMapping
public String myHandleMethod(WebRequest webRequest, Model model) {

    long eTag = ... 

    if (request.checkNotModified(eTag)) {
        return null; 
    }// The response has been set to 304 (NOT_MODIFIED) — no further processing.

    model.addAttribute(...); 
    return "myViewName";
}
```
有三种变体可用于检查针对 eTag 值，lastModified 值或同时两者的conditional request。 对于条件 GET 和 HEAD 请求，您可以将响应设置为 304（NOT_MODIFIED）。 对于条件 POST，PUT 和 DELETE，您可以将响应设置为 409（PRECONDITION_FAILED），以防止并发修改。

另外,您**应该**使用 Cache-Control 和条件响应头来提供静态资源，以获得最佳性能。 请参阅有关 [Static Resources](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-config-static-resources) 的部分。

您还可以使用 ShallowEtagHeaderFilter 添加computed from the response content的 “shallow” eTag 值，从而节省带宽但不节省 CPU 时间。 见 [Shallow ETag](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#filters-shallow-etag) 。

## 1.9 View Technologies
这里只介绍Jackson

### 1.9.9. Jackson

Spring 为 Jackson JSON 库提供支持。*MappingJackson2JsonView*这个类使用 Jackson 库的 *ObjectMapper* 将response内容呈现为 JSON。 默认情况下，模型映射的全部内容都编码为 JSON。 For cases where the contents of the map need to be filtered，您可以指定一个需要编码的model attributes集合,by使用 modelKeys 这个属性。 您还可以使用 *extractValueFromSingleKeyModel* 属性来把single-key models中的value直接提取和序列化，而不是as a map of model attributes。

::您可以使用 Jackson 提供的annotation根据需要 来自定义 JSON 映射::。 当您需要进一步控制时，可以通过 *ObjectMapper* 这个 property 来注入自定义的 ObjectMapper，当你需要为特定的类提供自定义JSON serializers and deserializers,你就可以这么做。

## 1.10. MVC Config
MVC Java cnfig和 MVC XML 命名空间提供了适用于大多数app的默认配置，以及用于自定义它的配置 API。至于配置 API 中未提供的更高级的自定义选项，请参阅 [Advanced Java Config](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-config-advanced-java) 。您无需理解MVC Java 配置。 如果您想了解更多信息，请参阅 [Special Bean Types](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-servlet-special-bean-types) and [Web MVC Config](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-servlet-config) .

### 1.10.1. Enable MVC Configuration

在 Java 配置中，您可以::使用 @EnableWebMvc 注解启用 MVC config::，如以下示例所示：(SpringBoot 中这是自动的)
``` java
@Configuration
@EnableWebMvc
public class WebConfig {
}
```
上面的示例注册了许多 Spring MVC  [infrastructure beans](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-servlet-special-bean-types) (即那些 special bean)，并adapts to classpath上可用的依赖项（例如，JSON，XML 等的有效负载转换器）。

### 1.10.2. MVC Config API

在 Java 配置中，您可以实现 WebMvcConfigurer 接口，如以下示例所示：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Implement configuration methods...
}
```
这就可以注册自定义的那些 special bean 或者他们的行为方式,在 spring boot 中,webmvcautoconfiguration 就实现了WebMvcConfigurer

### 1.10.3. Type Conversion
默认情况下是已经配置了 *Number* 和 *Date* 类型的 formatter，包括对 *@NumberFormat* 和 *@DateTimeFormat* 注解的支持。 单配类路径中存在 Joda-Time，则还会安装对 Joda-Time formatting library的完全支持。

在 Java 配置中，您可以注册自定义formatter和converter，如以下示例所示,也就是基于1.10.2使用 add方法来进行添加
```java

@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // ...
    }
}
```

### 1.10.4. Validation
默认情况下，如果类路径上存在  [Bean Validation](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/core.html#validation-beanvalidation-overview) 的库（例如，Hibernate Validator），则 *LocalValidatorFactoryBean* 将注册为全局 Validator，以便在controller method arguments上使用 @Valid 和 Validated来做验证。

在 Java 配置中，您可以自定义全局 Validator 实例，如以下示例所示：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public Validator getValidator(); {
        // ...
    }
}
```

请注意，您还可以在本地注册 Validator 实现，如以下示例所示：
```java
@Controller
public class MyController {
    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(new FooValidator());
    }
}
```

### 1.10.5. Interceptors

在Java配置中，您可以注册拦截器以拦截传入的请求，如以下示例所示,注意,每一个拦截器都配置了要拦截的路径模式：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
        registry.addInterceptor(new ThemeChangeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
        registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
    }
}
```

### 1.10.6. Content Types

您可以配置 Spring MVC 如何根据请求地特征来确定requested media types（例如，Accept 标头，URL 路径扩展，查询参数等）。默认情况下，首先检查 URL 路径扩展 - 将 json，xml，rss 和 atom 注册为已知的 extension,接下来会检查 Accept 标头。

请考虑将这些默认值更改为Accept header only，如果一定要使用基于 URL 的content-type解析，请考虑使用query parameter strategy而不是path extensions。 有关详细信息，请参阅 [Suffix Match](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc-ann-requestmapping-suffix-pattern-match) 。

在 Java 配置中，您可以自定义请求的内容类型的解析方式，如以下示例所示：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.mediaType("json", MediaType.APPLICATION_JSON);
        configurer.mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

### 1.10.7. Message Converters

感觉Message Converters和 Converter 的不同就是Converter是针对 pojo 类如何序列化反序列化,而Message Converters是针对格式的,比如 JSON,XML(如何从 request的JSON 中解析数据,又如何把响应写成符合一定格式的 JSON)

您可以通过覆盖 configureMessageConverters（）（替换 Spring MVC 创建的默认转换器）或重写extendMessageConverters（）（可以定制默认转换器或将添加自己实现的转换器,并添加到默认转换器）来自定义 Java 配置中的 *HttpMessageConverter*。

以下示例使用自定义的 ObjectMapper, 而不是默认的 ObjectMapper 添加 XML 和 Jackson JSON 转换器：
```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .modulesToInstall(new ParameterNamesModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.createXmlMapper(true).build()));
    }
}
```
在前面的示例中，Jackson2ObjectMapperBuilder 用于为 MappingJackson2HttpMessageConverter 和 MappingJackson2XmlHttpMessageConverter 创建通用配置，这包括启用缩进，自定义日期格式以及 *jackson-module-parameter-names* 这个可插拔模块的注册，这就支持了访问参数的名称。

这个builder自定义了 Jackson 的默认属性，如下所示：
* DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES 已禁用。
* MapperFeature.DEFAULT_VIEW_INCLUSION 已禁用。

如果在类路径中检测到它们，它还会自动注册以下众所周知的模块：
* jackson-datatype-jdk7：支持 Java 7 类型，例如 java.nio.file.Path。
* jackson-datatype-joda：支持 Joda-Time 类型。
* jackson-datatype-jsr310：支持 Java 8 Date 和 Time API 类型。
* jackson-datatype-jdk8：支持其他 Java 8 类型，例如 Optional。

还有一些其他有趣的JACKSON模块可用：
* jackson-datatype-money：支持 javax.money 类型（非官方模块）。
* jackson-datatype-hibernate：支持特定于 Hibernate 的类型和属性（包括延迟加载方面）。

### 1.10.8. View Controllers

这是定义 ParameterizableViewController 的快捷方式，该方法在调用时立即forwards to视图。 如果在视图生成响应之前,没有要执行的 Java 控制器逻辑，则可以在静态情况下使用它。

以下 Java 配置示例会将request 转发给名为 home 的视图：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
    }
}
```

### 1.10.9. View Resolvers

MVC 配置简化了视图解析器的注册。以下 Java 配置示例使用了 JSP 和 Jackson 作为 JSON 解析的默认视图,来来配置content negotiation view resolution：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
        registry.jsp();
    }
}
```

但请注意，FreeMarker，Tiles，Groovy Markup 和脚本模板也需要配置底层视图技术。在 Java 配置中，您可以添加相应的 Configurer bean，如以下示例所示：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
        registry.freeMarker().cache(false);
    }

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("/freemarker");
        return configurer;
    }
}
```

### 1.10.10. Static Resources

此选项提供了一种从a list of [Resource](https://docs.spring.io/spring-framework/docs/5.1.5.RELEASE/javadoc-api/org/springframework/core/io/Resource.html) -based locations中提供静态资源的便捷方法。

在下一个示例中，给定一个 /resources 开头的请求，相对路径用于在 Web 应用程序根目录下或 /static  目录下的classpath上来 servr相对于 /public 的静态资源。 资源将在未来一年内到期，以确保最大限度地使用浏览器缓存并减少浏览器发出的 HTTP 请求。  同时,还会评估 Last-Modified 标头，如果存在，则返回 304 状态代码。

以下清单显示了如何使用 Java 配置执行此操作：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
            .addResourceLocations("/public", "classpath:/static/")
            .setCachePeriod(31556926);
    }
}
```

resource handler还支持配置一系列 ResourceResolver 实现和 ResourceTransformer 实现，您可以使用它们创建工具链以使用更优化的资源。 您可以将 VersionResourceResolver 用于基于从内容，固定的应用程序版本或其他地方计算的versioned resource URLs based on an MD5 hash。 ContentVersionStrategy（MD5 哈希）是一个不错的选择 - 有一些值得注意的例外，例如JavaScript resources used with a module loader.

以下示例显示如何在 Java 配置中使用 VersionResourceResolver：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public/")
                .resourceChain(true)
                .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
    }
}
```


### 1.10.11. Default Servlet

Spring MVC 允许将 DispatcherServlet 映射到 /（从而覆盖容器中的默认 Servlet 的映射），同时仍允许使用容器的默认 Servlet 来处理静态资源请求。 它使依靠配置DefaultServletHttpRequestHandler实现的，其 URL 映射为 / **，并且相对于其他 URL 映射具有最低优先级。

这个 handler会将所有请求转发到默认 Servlet。 因此，它必须在所有 URL HandlerMappings 中保持最后的顺序。或者，如果您设置了自己的自定义 HandlerMapping 实例，请确保将其 order 属性设置为低于 DefaultServletHttpRequestHandler 的值(低就会更有线)， DefaultServletHttpRequestHandler 的优先级是Integer.MAX_VALUE。

以下示例显示如何使用默认设置启用该功能：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

### 1.10.12. Path Matching

您可以自定义与路径匹配和 URL 处理相关的选项(即路径如何匹配到 handlerMapping)。 有关各个选项的详细信息，请参阅 [PathMatchConfigurer](https://docs.spring.io/spring-framework/docs/5.1.5.RELEASE/javadoc-api/org/springframework/web/servlet/config/annotation/PathMatchConfigurer.html) 的javadoc。以下示例显示如何在 Java 配置中自定义path matching：
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer
            .setUseSuffixPatternMatch(true) //是否使用后缀路径匹配
            .setUseTrailingSlashMatch(false) 
            .setUseRegisteredSuffixPatternMatch(true)
            .setPathMatcher(antPathMatcher()) //使用路径匹配 器
            .setUrlPathHelper(urlPathHelper())
            .addPathPrefix("/api",
                    HandlerTypePredicate.forAnnotation(RestController.class));
    }

    @Bean
    public UrlPathHelper urlPathHelper() {
        //...
    }

    @Bean
    public PathMatcher antPathMatcher() {
        //...
    }

}
```

### 1.10.13. Advanced Java Config

@EnableWebMvc 导入了 DelegatingWebMvcConfiguration，其中：
* 为 Spring MVC 应用程序提供默认的 Spring 配置
* 检测并委派给 WebMvcConfigurer,的实现以定制该配置。

对于高级模式，您可以不使用 @EnableWebMvc ,而是直接继承 DelegatingWebMvcConfiguration 而不是实现 WebMvcConfigurer，如以下示例所示：
```java
@Configuration
public class WebConfig extends DelegatingWebMvcConfiguration {
    // ...
}
```
您可以在 WebConfig 中保留现有方法，但现在您也可以从base class覆盖 bean 声明，并且您仍然可以在类路径上拥有任意数量的其他 WebMvcConfigurer 实现。

## 1.11 HTTP/2
使用Servlet 4 容器要求支持 HTTP / 2，Spring Framework 5 与 Servlet API 4 兼容。其实从编程模型的角度来看，应用程序不需要特定的任何操作。 但是，还是存在与服务器配置相关的注意事项。 有关更多详细信息，请参阅 [HTTP/2 wiki page](https://github.com/spring-projects/spring-framework/wiki/HTTP-2-support) 

Servlet API 确实公开了一个与 HTTP / 2 相关的construct。 您可以使用 javax.servlet.http.PushBuilder 主动将资源推送到client，并且它还可以作为 @RequestMapping 方法的参数。

- - - -
# REST Clients
本节介绍客户端访问 REST 端点的选项。

## 2.1.RestTemplate
RestTemplate 是一个执行 HTTP 请求的同步客户端。 它是最初的 Spring REST 客户端，基于底层 HTTP 客户端库上,暴露了了一个简单,template-method API。 [REST Endpoints](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/integration.html#rest-client-access) 

从 5.0 开始，非阻塞，反应式 WebClient 提供了 RestTemplate 的现代替代方案，同时有效支持同步和异步以及流方案。 RestTemplate 将在未来版本中弃用，并且不会在未来添加主要的新功能。

## 2.2.WebClient
WebClient 是一个执行 HTTP 请求的非阻塞，reactive的客户端。 它是在 5.0 中引入的，它提供了 RestTemplate 的现代替代方案，同时有效支持::同步和异步::以及流方案。

与 RestTemplate 相比，WebClient 支持以下内容：
* 非阻塞 I / O.
* Reactive Streams back pressure.
* 高并发性，硬件资源更少。
* 函数风格，流畅的 API，利用 Java 8 lambdas。
* 同步和异步交互。
* Streaming up to or streaming down from a server.

See [WebClient](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web-reactive.html#webflux-client) for more details.

- - - -
# Testing
测试有关的内容具体参阅文档[Spring测试](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/testing.html#integration-testing)

本节总结了 Spring MVC 应用程序测试时, spring-test 中可用的选项。
* Servlet API Mocks：用于对控制器，过滤器和其他 Web 组件做单元测试. 这是基于Servlet API contract 的的Mock实现。有关更多详细信息，请参阅 [Servlet API](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/testing.html#mock-objects-servlet)mock objects。
* TestContext Framework：支持在 JUnit 和 TestNG 测试中加载 Spring 配置，包括支持高效缓存已加载的config across test methods，以及支持使用 MockServletContext 来加载 WebApplicationContext。有关更多详细信息，请参阅 [TestContext Framework](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/testing.html#testcontext-framework) 。
* Spring MVC Test：一个框架，也称为 *MockMvc*，用于通过 *DispatcherServlet* 来测试annotated控制器，拥有完整的Spring MVC 基础设置,但没有 HTTP 服务器。有关更多详细信息，请参阅  [Spring MVC Test](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/testing.html#spring-mvc-test-framework) 。::(这个可能是我们关注的)::
* 客户端 REST：spring-test 提供了一个 *MockRestServiceServer*，您可以将其用作mock server来测试使用了 RestTemplate 的客户端代码。有关详细信息，请参阅客户端 REST 测试。
* WebTestClient：专为测试 WebFlux 应用程序而构建，但它也可用于通过 HTTP 连接对任何服务器进行**端到端集成测试**。它是一个非阻塞，reactive的客户端，非常适合测试异步和流式方案。



