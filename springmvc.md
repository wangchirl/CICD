### Spring mvc 原理
#### 1、需要知道的前置知识
##### 1. 三大容器
- <font color="#FF7F24">`Servlet`容器（tomcat）
- <font color=#FF00FF>`Spring` 容器
- <font color=#CD1076>`Spring mvc` 容器
##### 2. 三大容器的启动顺序
- <font color=blue>`Servlet` 容器最先启动（tomcat）
- <font color=green>`Spring` 容器次启动（父容器 WebApplicationContext [XmlWebApplicationContext]）
- <font color=orange>`Spring mvc` 容器最后启动（子容器 WebApplicationContext [XmlWebApplicationContext]）
##### 3. Servlet 生命周期
- <font color=green>init 初始化方法，只调用一次，首次访问时调用或容器启动时调用（load-on-startup设置大于0的值时）
- <font color=green>service 请求处理方法，调用多次，每次对应请求时都调用
- <font color=green>destroy 销毁方法，只调用一次，Serlvet 销毁时或容器关闭时调用
#### 2、三大容器启动源码
##### 1.**启动 Servlet 容器（tomcat BootStrap 启动）**
> 1. 读取 web.xml 配置信息
>
> - `Spring 的配置信息`
>  ```xml
>  <!-- 监听器 -->
>  <listener>
>        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
>  </listener>
>  <!--Spring配置文件 -->
>  <context-param>
>        <param-name>contextConfigLocation</param-name>
>        <param-value>classpath:spring-config.xml</param-value>
>  </context-param>
>  ```
> - `Spring mvc 的配置信息`
>  ```xml
>  <!-- SpringMVC 前端控制器 DispatcherServlet --> 
>  <servlet>
>        <servlet-name>mvc</servlet-name>
>        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
>        <!--SpringMVC配置文件-->
>        <init-param>
>            <param-name>contextConfigLocation</param-name>
>            <param-value>classpath:applicationContext.xml</param-value>
>        </init-param>
>        <!-- 容器启动时初始化：即调用 init 方法 -->
>        <load-on-startup>1</load-on-startup>
>  </servlet>
>  
>  <servlet-mapping>
>        <servlet-name>mvc</servlet-name>
>        <url-pattern>/</url-pattern>
>  </servlet-mapping>
>  ```
>
> 2. tomcat 的 StandardContext#listenerStart() 获取监听器并回调监听器方法 contextInitialized() 
>
>  ```java
>  protected synchronized void startInternal() throws LifecycleException {
>       ....
>            try {
>                ....
>                    // Configure and call application event listeners
>                    // ① 监听器处理 
>                    if (ok) {
>                        if (!listenerStart()) {
>                            log.error(sm.getString("standardContext.listenerFail"));
>                            ok = false;
>                        }
>                    }
>                ....
>                    // Configure and call application filters
>                    // ② 过滤器处理
>                    if (ok) {
>                        if (!filterStart()) {
>                            log.error(sm.getString("standardContext.filterFail"));
>                            ok = false;
>                        }
>                    }
>                // Load and initialize all "load on startup" servlets
>                // ③ 加载并初始化所有的 load-on-startup=1 的 servlet => DispatcherServlet
>                if (ok) {
>                    if (!loadOnStartup(findChildren())){
>                        log.error(sm.getString("standardContext.servletFail"));
>                        ok = false;
>                    }
>                }
>                ....
>            } finally {
>                ....
>            }
>       ....
>  }
>  ```
>
>  ```java
>  public boolean listenerStart() {
>       ....
>            for (Object instance : instances) {
>                ....
>                    try {
>                        ....
>                            if (noPluggabilityListeners.contains(listener)) {
>                                listener.contextInitialized(tldEvent);
>                            } else {
>                                // ④ 监听器回调方法 => ContextLoaderListener#contextInitialized()
>                                listener.contextInitialized(event);
>                            }
>                        ....
>                    } catch (Throwable t) {
>                       ....
>                    }
>            }
>        return ok;
>  }
>  ```
>
>  总结：`启动 Servlet 容器，读取 web.xml 配置信息，在StandardContext#listenerStart() 获取监听器并回调监听器方法 contextInitialized()`
##### 2.**启动 Spring 容器（ContextLoaderListener#contextInitialized）**
> 1. ContextLoaderListener#contextInitialized
>  ```java
>  public void contextInitialized(ServletContextEvent event) {
>        // ① 入口
>        initWebApplicationContext(event.getServletContext());
>  }
>  ```
>
> 2. initWebApplicationContext()
>
>  ```java
>  public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
>       ...
>            try {
>                // 创建 WebApplicationContext => XmlWebApplicationContext
>                if (this.context == null) {
>                    this.context = createWebApplicationContext(servletContext);
>                }
>                if (this.context instanceof ConfigurableWebApplicationContext) {
>                    ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
>                    if (!cwac.isActive()) {
>                        if (cwac.getParent() == null) {
>                            // 父容器为 null
>                            ApplicationContext parent = loadParentContext(servletContext);
>                            cwac.setParent(parent);
>                        }
>                        // ② 关键方法 refresh()
>                        configureAndRefreshWebApplicationContext(cwac, servletContext);
>                    }
>                }
>                // 当前容器也即spring容器存储到 servlet 上下文中
>                servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
>               ....
>                    return this.context;
>            }
>        catch (RuntimeException | Error ex) {
>           ....
>        }
>  }
>  ```
>
>  ```java
>  protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
>        ....
>           // spring容器持有servlet上下文，相互持有
>            wac.setServletContext(sc);
>        // web.xml 配置的 spring 的配置信息
>        // contextConfigLocation:classpath*:spring.xml
>        String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
>        if (configLocationParam != null) {
>            wac.setConfigLocation(configLocationParam);
>        }
>       ....
>            // ③ AbstractApplicationContext#refresh()
>            wac.refresh();
>  }
>  ```
>
> 3. AbstractApplicationContext#refresh() [spring源码知识]
>  ```java
>  public void refresh() throws BeansException, IllegalStateException {
>        synchronized (this.startupShutdownMonitor) {
>            // 1
>            prepareRefresh();
>            // 2
>            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
>            // 3
>            prepareBeanFactory(beanFactory);
>            try {
>                // 4
>                postProcessBeanFactory(beanFactory);
>                // 5
>                invokeBeanFactoryPostProcessors(beanFactory);
>                // 6
>                registerBeanPostProcessors(beanFactory);
>                // 7
>                initMessageSource();
>                // 8
>                initApplicationEventMulticaster();
>                // 9
>                onRefresh();
>                // 10
>                registerListeners();
>                // 11
>                finishBeanFactoryInitialization(beanFactory);
>                // 12
>                finishRefresh();
>            }
>            catch (BeansException ex) {
>                // 13
>                destroyBeans();
>                // 14
>                cancelRefresh(ex);
>                throw ex;
>            }
>            finally {
>                // 15
>                resetCommonCaches();
>            }
>        }
>  }
>  ```
>
>  总结：在 Servlet 回调监听器 contextInitialized() 方法过程中，ContextLoaderListener 中的
>
>  contextInitialized() 方法完成 Spring IOC 容器的创建并 refresh() 刷新工厂，并与 Servlet 上下文相互持有引用，即：将Spring 容器设置到 Servlet 上下文中，Servlet 上下文设置到 Spring 容器中，以便在创建 Spring MVC 容器时设置 Spring 容器为其父容器
##### 3. **启动 Spring MVC 容器**
> 1. tomcat 的 StandardContext#loadOnStartup()
>  ```java
>  protected synchronized void startInternal() throws LifecycleException {
>       ....
>            try {
>                ....
>                    // Configure and call application event listeners
>                    // ① 监听器处理 
>                    if (ok) {
>                        if (!listenerStart()) {
>                            log.error(sm.getString("standardContext.listenerFail"));
>                            ok = false;
>                        }
>                    }
>                ....
>                    // Configure and call application filters
>                    // ② 过滤器处理
>                    if (ok) {
>                        if (!filterStart()) {
>                            log.error(sm.getString("standardContext.filterFail"));
>                            ok = false;
>                        }
>                    }
>                // Load and initialize all "load on startup" servlets
>                // ③ 加载并初始化所有的 load-on-startup=1 的 servlet => DispatcherServlet
>                if (ok) {
>                    if (!loadOnStartup(findChildren())){
>                        log.error(sm.getString("standardContext.servletFail"));
>                        ok = false;
>                    }
>                }
>                ....
>            } finally {
>                ....
>            }
>       ....
>  }
>  ```
>
>  ```java
>  public boolean loadOnStartup(Container children[]) {
>       ....
>            // Load the collected "load on startup" servlets
>            // load on startup = 1 的 Servlet => DispatcherServlet 
>            for (ArrayList<Wrapper> list : map.values()) {
>                for (Wrapper wrapper : list) {
>                    try {
>                        // ④
>                        wrapper.load();
>                    } catch (ServletException e) {
>                        ....
>                    }
>                }
>            }
>        return true;
>  }
>  ```
>
>  StandarWrapper#load()
>  ```java
>  public synchronized void load() throws ServletException {
>       // 加载 Servlet  
>        instance = loadServlet();
>        if (!instanceInitialized) {
>            // ⑤ init Servlet
>            initServlet(instance);
>        }
>       ....
>  }
>  ```
>  ```java
>  private synchronized void initServlet(Servlet servlet)
>        throws ServletException {
>       ....
>            try {
>                // ⑥ Servlet 的 init 方法 => javax.servlet-api=> GenericServlet
>                // 最终调用 DispatcherServlet#init()方法
>                servlet.init(facade);
>                ....
>            } catch (UnavailableException f) {
>                ....
>            } 
>        ....
>  }
>  ```
> 2. DispatcherServlet、FrameworkServlet、HttpServletBean
>  HttpServletBean#init()
>  ```java
>  public final void init() throws ServletException {
>       ...
>            // Let subclasses do whatever initialization they like.
>            // ① 入口
>            initServletBean();
>  }
>  ```
>  FrameworkServlet#initServletBean()
>
>  ```java
>  protected final void initServletBean() throws ServletException {
>        ....
>            try {
>                // ② 创建 SpringMVC 容器
>                this.webApplicationContext = initWebApplicationContext();
>                initFrameworkServlet();
>            }
>        catch (ServletException | RuntimeException ex) {
>            ....
>        }
>       ....
>  }
>  ```
>
>  ```java
>  protected WebApplicationContext initWebApplicationContext() {
>        // 得到父容器，即：Spring IOC 容器
>        WebApplicationContext rootContext =     WebApplicationContextUtils.getWebApplicationContext(getServletContext());
>        WebApplicationContext wac = null;
>       ....
>            if (wac == null) {
>                // ③ 创建 Spring MVC 容器，并传入父容器
>                wac = createWebApplicationContext(rootContext);
>            }
>       ....
>            return wac;
>  }
>  ```
>
>  ```java
>  protected WebApplicationContext createWebApplicationContext(@Nullable WebApplicationContext parent) {
>        // ④ 
>        return createWebApplicationContext((ApplicationContext) parent);
>  }
>  
>  protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
>        // 得到 XmlWebApplicationContext 类
>        Class<?> contextClass = getContextClass();
>       // 反射创建对象
>        ConfigurableWebApplicationContext wac =
>            (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
>       // 设置环境及父容器
>        wac.setEnvironment(getEnvironment());
>        wac.setParent(parent);
>        // 获取spring mvc 的配置并设置 classpath*:spring-mvc.xml
>        String configLocation = getContextConfigLocation();
>        if (configLocation != null) {
>            wac.setConfigLocation(configLocation);
>        }
>        // ⑤ 刷新工厂
>        configureAndRefreshWebApplicationContext(wac);
>        return wac;
>  }
>  ```
>
>  ```java
>  protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
>       ....
>            // 设置 Servlet上下文到 springmvc容器中
>            // ServletContext 是整个 web.xml 对象
>            // ServletConfig 是单个 Servlet 对象
>            wac.setServletContext(getServletContext());
>        wac.setServletConfig(getServletConfig());
>        wac.setNamespace(getNamespace());
>        // ⑥ 这里注册了监听器ContextRefreshListener，监听事件 ContextRefreshedEvent
>        // 用于回调 ContextRefreshListener#onApplicationEvent()
>        wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
>       ....
>            postProcessWebApplicationContext(wac);
>        applyInitializers(wac);
>        // ⑦ 刷新工厂，需要注意前面有注册监听器，在 refresh方法后面的finishRefresh有发布事件
>        wac.refresh();
>  }
>  ```
>
>  ```java
>  protected void finishRefresh() {
>       ....
>            // ⑧ 发布 ContextRefreshedEvent 事件
>            publishEvent(new ContextRefreshedEvent(this));
>       ....
>  }
>  ```
>
>  ContextRefreshListener#onApplicationEvent()事件回调
>  ```java
>  public void onApplicationEvent(ContextRefreshedEvent event) {
>        // ⑨ 事件回调
>        FrameworkServlet.this.onApplicationEvent(event);
>  }
>  ```
>
>  ```java
>  public void onApplicationEvent(ContextRefreshedEvent event) {
>        this.refreshEventReceived = true;
>        synchronized (this.onRefreshMonitor) {
>            // ⑩ DispatcherServlet#onRefresh()
>            onRefresh(event.getApplicationContext());
>        }
>  }
>  ```
>
>  ```java
>  protected void onRefresh(ApplicationContext context) {
>        // 初始化 springmvc 的九大内置组件
>        initStrategies(context);
>  }
>  
>  protected void initStrategies(ApplicationContext context) {
>        // 1.文件上传组件 MultipartResolver
>        initMultipartResolver(context);
>        // 2.国际化组件 LocaleResolver
>        initLocaleResolver(context);
>        // 3.主题组件  ThemeResolver
>        initThemeResolver(context);
>        // 4.处理器映射器 HandlerMapping
>        initHandlerMappings(context);
>        // 5.处理器适配器 HandlerAdapter
>        initHandlerAdapters(context);
>        // 6.处理器异常解析器  HandlerExceptionResolver
>        initHandlerExceptionResolvers(context);
>        // 7.请求路径转视图名称转换器 RequestToViewNameTranslator
>        initRequestToViewNameTranslator(context);
>        // 8.视图解析器  ViewResolver
>        initViewResolvers(context);
>        // 9.重定向参数传递组件 FlashMapManager
>        initFlashMapManager(context);
>  }
>  ```
>
>  总结：Servlet 标准生命周期：init() => service() => destroy()
>
>  load-on-startup 设置为 1 的，tomcat会提前初始化此 Servlet，即会调用 Servlet#init()，也即会调用 DispatcherServlet 的 init() 方法，在其父类 HttpServletBean 中实现，最终完成 springmvc 容器的创建，并通过注册监听器的方式回调方法完成springmvc的九大内置组件的初始化
>
>  SpringMVC 九大内置组件：
>
>  - **MultipartResolver**    文件上传相关解析器
>  - **LocaleResolver**                   国际化相关解析器
>  - **ThemeResolver**                   主题相关解析器
>  - **HandlerMapping**                  处理器映射器
>  - **HandlerAdapter**                  处理器适配器
>  - **HandlerExceptionResolver**        处理器异常解析器
>  - **RequestToViewNameTranslator**   请求视图名称转换相关
>  - **ViewResolver**                     视图解析器
>  - **FlashMapManager**                重定向参数传递相关
#### 3、SpringMVC 九大内置组件初始化
##### 1. `MultipartResolver`
> 1. 文件上传相关组件，主要实现有
>
>  - org.springframework.web.multipart.commons.CommonsMultipartResolver
>  
>   ```xml
>   <dependency>
>        <groupId>commons-fileupload</groupId>
>        <artifactId>commons-fileupload</artifactId>
>   </dependency>
>   ```
>  
>  - org.springframework.web.multipart.support.StandardServletMultipartResolver
>
> 2. 默认没有注入相关的 bean ，也即 multipartResolver = null，如果需要使用到文件上传相关的功能，需要注入上面的2个实现中的任意一个到容器中
>
>  ```xml
>  <!-- 文件上传 -->
>  <bean class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
>        <property name="defaultEncoding" value="UTF-8"/>
>        <property name="maxUploadSize" value="52428800" />
>  </bean>
>  ```
>
> 3. 源码
>
> ```java
> private void initMultipartResolver(ApplicationContext context) {
>       try {
>            // 这里查询不到 bean 会抛异常进入 catch
>            this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
>            ....
>       }
>       catch (NoSuchBeanDefinitionException ex) {
>            // 默认 null
>            this.multipartResolver = null;
>            ....
>       }
> }
> ```
>
> 
##### 2. `LocaleResolver`
> 1. 国际化相关组件，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
>
> 2. 源码
>
>  ```java
>  private void initLocaleResolver(ApplicationContext context) {
>        try {
>            // 从容器中查询 bean，这里查询不到抛异常，进入catch赋予默认值
>            this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
>           ....
>        }
>        catch (NoSuchBeanDefinitionException ex) {
>            // 从 DispatcherServlet.properties 读取默认值
>            this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
>           ....
>        }
>  }
>  ```
##### 3. `ThemeResolver`
> 1. 主题相关组件，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.theme.FixedThemeResolver
>
> 2. 源码
>
>  ```java
>  private void initThemeResolver(ApplicationContext context) {
>        try {
>            // 从容器中查询 bean，这里查询不到抛异常，进入catch赋予默认值
>            this.themeResolver = context.getBean(THEME_RESOLVER_BEAN_NAME, ThemeResolver.class);
>            ....
>        }
>        catch (NoSuchBeanDefinitionException ex) {
>            // 从 DispatcherServlet.properties 读取默认值
>            this.themeResolver = getDefaultStrategy(context, ThemeResolver.class);
>            ....
>        }
>  }
>  ```
##### 4. `HandlerMapping`
> 1. 处理器映射器相关组件，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
>  - org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping
>  - org.springframework.web.servlet.function.support.RouterFunctionMapping [不怎么使用]
>
> 2. 源码
>
>  ```java
>  private void initHandlerMappings(ApplicationContext context) {
>        ....
>            if (this.handlerMappings == null) {
>                // 从 DispatcherServlet.properties 读取默认值
>                this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
>                ....
>            }
>  }
>  ```
>
> 3. RequestMappingHandlerMapping 的创建过程
>
>  - `ApplicationContextAware` 子类 setApplicationContext() 方法<font color=blue> 初始化拦截器
>
>   ```java
>   public final void setApplicationContext(@Nullable ApplicationContext context) throws BeansException {
>        if (context == null && !isContextRequired()) {
>            ....
>        }
>        else if (this.applicationContext == null) {
>            ....
>                // ①  
>                initApplicationContext(context);
>        }
>        else {
>            ....
>        }
>   }
>   
>   protected void initApplicationContext(ApplicationContext context) {
>        // ②
>        super.initApplicationContext(context);
>        ....
>   }
>   
>   protected void initApplicationContext(ApplicationContext context) throws BeansException {
>        // ③
>        initApplicationContext();
>   }
>   
>   // ④ 拦截器初始化
>   protected void initApplicationContext() throws BeansException {
>        extendInterceptors(this.interceptors);
>        detectMappedInterceptors(this.adaptedInterceptors);
>        initInterceptors();
>   }
>   ```
>
>  - `InitializingBean` 子类 afterPropertiesSet() 方法找到控制器相关<font color=blue>方法请求映射注册到 MappingRegistry
>
>   `RequestMappingHandlerMapping`
>
>   ```java
>   public void afterPropertiesSet() {
>        ....
>            // ①
>            super.afterPropertiesSet();
>   }
>   ```
>
>   `AbstractHandlerMethodMapping`
>
>   ```java
>   public void afterPropertiesSet() {
>        // ②
>        initHandlerMethods();
>   }
>   
>   protected void initHandlerMethods() {
>        for (String beanName : getCandidateBeanNames()) {
>            if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
>                // ③
>                processCandidateBean(beanName);
>            }
>        }
>        handlerMethodsInitialized(getHandlerMethods());
>   }
>   
>   protected void processCandidateBean(String beanName) {
>        ....
>            if (beanType != null && isHandler(beanType)) {
>                // ④
>                detectHandlerMethods(beanName);
>            }
>   }
>   // 请求方法映射信息保存
>   protected void detectHandlerMethods(Object handler) {
>        Class<?> handlerType = (handler instanceof String ?
>                                obtainApplicationContext().getType((String) handler) : handler.getClass());
>   
>        if (handlerType != null) {
>            Class<?> userType = ClassUtils.getUserClass(handlerType);
>            Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
>                                                                      (MethodIntrospector.MetadataLookup<T>) method -> {
>                                                                          ....
>                                                                              // ⑤ 找到请求对应的方法 
>                                                                              return getMappingForMethod(method, userType);
>                                                                          ....
>                                                                      });
>            ....
>                methods.forEach((method, mapping) -> {
>                    Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
>                    // ⑥ 注册 MappingRegistry 中 已被后续请求进行查找使用
>                    registerHandlerMethod(handler, invocableMethod, mapping);
>                });
>        }
>   }
>   ```
##### 5. `HandlerAdapter`
> 1. 处理器适配器相关组件，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter
>  - org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter
>  - org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
>  - org.springframework.web.servlet.function.support.HandlerFunctionAdapter [不怎么使用]
>
> 2. 源码
>
>  ```java
>  private void initHandlerAdapters(ApplicationContext context) {
>        ....
>            if (this.handlerAdapters == null) {
>                // 从 DispatcherServlet.properties 读取默认值
>                this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
>                ....
>            }
>  }
>  ```
>
> 3. 作用
>
>  - 根据我们的处理器不同的实现方式进行适配
>   - 实现 org.springframework.web.servlet.mvc.Controller 接口
>   - 实现 org.springframework.web.HttpRequestHandler 接口
>   - 使用 org.springframework.stereotype.Controller 注解
>  - Controller 接口 => SimpleControllerHandlerAdapter
>  - HttpRequestHandler 接口 => HttpRequestHandlerAdapter
>  - @Controller 注解 => RequestMappingHandlerAdapter
>
> 4. `RequestMappingHandlerAdapter` 的创建过程
>
>  - ApplicationContextAware 子类 setApplicationContext() 方法这里啥也没干
>
>  - InitializingBean 子类 afterPropertiesSet() 方法 初始化 ControllerAdvice、argumentResolvers、initBinderArgumentResolvers、returnValueHandlers
>
>   ```java
>   public void afterPropertiesSet() {
>        // 解析 @ControllerAdvice 注解修饰的类
>        // 解析 @ExceptionHandler、@ResponseBody、@ResponseEntity 修饰的方法
>        initControllerAdviceCache();
>        // 参数解析器 26 个
>        if (this.argumentResolvers == null) {
>            List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
>            this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
>        }
>        // initBinder 参数解析器 12 个
>        if (this.initBinderArgumentResolvers == null) {
>            List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
>            this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
>        }
>        // 返回值处理器 15 个
>        if (this.returnValueHandlers == null) {
>            List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
>            this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
>        }
>   }
>   ```
>
>   - HandlerMethodArgumentResolver 处理器方法参数解析器
>   - HandlerMethodReturnValueHandler 处理器方法返回值处理器
##### 6. `HandlerExceptionResolver`
> 1. 处理器异常解析器，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver
>  - org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver
>  - org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
>
> 2. 源码
>
>  ```java
>  private void initHandlerExceptionResolvers(ApplicationContext context) {
>        ....
>            if (this.handlerExceptionResolvers == null) {
>                // 从 DispatcherServlet.properties 读取默认值
>                this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
>                ....
>            }
>  }
>  ```
>
> 3. 作用
>
>  - ExceptionHandlerExceptionResolver
>
>   @ExceptionHandler 注解修饰的方法
>
>  - ResponseStatusExceptionResolver
>
>   @ResponseStatus 注解修饰的方法
>
>  - DefaultHandlerExceptionResolver
>
>   解决标准Spring MVC异常并将其转换为相应的异常 HTTP 状态码
##### 7. `RequestToViewNameTranslator`
> 1. 请求url转为视图名称处理器，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator
>
> 2. 源码
>
>  ```java
>  private void initRequestToViewNameTranslator(ApplicationContext context) {
>        try {
>            ....
>        }
>        catch (NoSuchBeanDefinitionException ex) {
>            // 从 DispatcherServlet.properties 读取默认值
>            this.viewNameTranslator = getDefaultStrategy(context, RequestToViewNameTranslator.class);
>            ....
>        }
>  }
>  ```
>
> 3. 作用
>
>  只是转换请求 URI 将传入请求转换为视图名
>
>  http://localhost:8080/gamecast/display => display
>
>  http://localhost:8080/gamecast/admin/index => admin/index
##### 8. `ViewResolver`
> 1. 视图解析器，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.view.InternalResourceViewResolver
>
> 2. 源码
>
>  ```java
>  private void initViewResolvers(ApplicationContext context) {
>        ....
>            if (this.viewResolvers == null) {
>                // 从 DispatcherServlet.properties 读取默认值
>                this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
>                ....
>            }
>  }
>  ```
>
> 3. 作用
>
>  找到视图文件，渲染视图
##### 9. `FlashMapManager`
> 1. 重定向参数传递相关组件，DispatcherServlet.properties 中配置了默认值
>
>  - org.springframework.web.servlet.support.SessionFlashMapManager
>
> 2. 源码
>
>  ```java
>  private void initFlashMapManager(ApplicationContext context) {
>        try {
>            ....
>        }
>        catch (NoSuchBeanDefinitionException ex) {
>            // 从 DispatcherServlet.properties 读取默认值
>            this.flashMapManager = getDefaultStrategy(context, FlashMapManager.class);
>            ....
>        }
>  }
>  ```
>
> 3. 作用
>
>  在重定向过程中，实现参数的传递
#### 4、SpringMVC处理请求流程
##### 1. [Tomcat](./tomcat.md)
- HTTP 服务
 - 建立连接，接收请求
 - 请求解析，请求适配
 - 转发请求给具体的 Serlvet
- Servlet 容器
 - Engine  -> StandardEngine -> Pipeline -> StandardPipeline [xxxValve ... StandardEngineValve]
 - Host     -> StandardHost   -> Pipeline -> StandardPipeline [xxxValve ... StandardHostValve]
 - Context  -> StandardContext -> Pipeline -> StandardPipeline [xxxValve ... StandardContextValve]
 - Wrapper  -> StandardWrapper -> Pipeline -> StandardPipeline [xxxValve ... StandardWrapperValve] -> ApplicationFilterChain 过滤器链 -> servlet.service(request, response)
 责任链模式依次执行 Valve，Wrapper 是Servlet的包装体，Wrapper 是最后一个执行的容器组件，内部匹配过滤器，创建过滤器链，依次执行过滤器，最终执行 servlet.service() 方法处理请求
##### 2. FrameworkServlet
- service 方法
 ```java
protected void service(HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
    // 获取请求方式 GET/POST...
    HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
    if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
        processRequest(request, response);
    }
    else {
        // ① 父类 HttpServlet.service() 方法: 请求分发，调用对应的方法 doGet/doPost...
        super.service(request, response);
    }
}
 ```
 ```java
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
    // ② 最终调用 processRequest
    processRequest(request, response);
}
protected final void doPost(HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
    // ② 最终调用 processRequest
    processRequest(request, response);
}
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
    ....
        try {
            // ③ DispatherServlet#doService 方法
            doService(request, response);
        }
    catch (ServletException | IOException ex) {
        ....
    }
    catch (Throwable ex) {
        ....
    }
    finally {
        ....
            // 发布请求处理完成事件  
            publishRequestHandledEvent(request, response, startTime, failureCause);
    }
}
 ```
##### 3. DispatherServlet
- doService 方法
 ```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    logRequest(request);
    ....
        try {
            // ④ DispatherServlet#doDispatch 方法
            doDispatch(request, response);
        }
    finally {
        ....
    }
}
 ```
- doDispatch 方法
 ```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    // 异步管理器
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    try {
        ModelAndView mv = null;
        Exception dispatchException = null;
        try {
            // ① 检查是否上传请求
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);
            // ② 根据请求路径获取处理器链：handler + interceptors
            // 处理器 handler + 拦截器 interceptors
            mappedHandler = getHandler(processedRequest);
            // 没找到就抛异常
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }
            // ③ 根据 handler 获取处理器适配器 HandlerAdaptor
            // 1.RequestMappingHandlerAdapter => @Controller 注解
            // 2.SimpleControllerHandlerAdapter => Controller 接口
            // 3.HttpRequestHandlerAdapter => HttpRequestHandler 接口
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
            // 缓存处理
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
            // ④ 执行拦截器 preHandle 方法
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }
            // ⑤ 真正执行控制器方法逻辑
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
            // 默认视图处理器名称设置
            applyDefaultViewName(processedRequest, mv);
            // ⑥ 执行拦截器 postHandle 方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // ⑦ 请求响应结果、视图渲染等处理
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        // 异常后的清理工作
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        // 异常后的清理工作
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
 ```
 > 结论：
 >
 > 1、检查是否上传请求
 >
 > 2、根据请求找到 handler 对象，其结合拦截器组成处理器拦截器链
 >
 > 3、根据 handler 找到 HandlerAdaptor
 >
 > 4、处理 last_modified 是否缓存逻辑
 >
 > 5、执行拦截器的 preHandle 方法
 >
 > 6、执行真正的业务方法（Controller逻辑）
 >
 > 7、检查是否异步处理
 >
 > 8、检查是否需要设置默认视图名称
 >
 > 9、执行拦截器的 postHandle 方法
 >
 > 10、处理视图，渲染视图对象
 >
 > 11、执行拦截器的 afterCompletion 方法
- checkMultipart 方法
 ```java
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
    // this.multipartResolver.isMultipart 根据请求头 Content-Type 判断是否 multipart
    if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
        ....
            try {
                // 处理上传请求并对请求进行包装
                return this.multipartResolver.resolveMultipart(request);
            }
        catch (MultipartException ex) {
            if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
                logger.debug("Multipart resolution failed for error dispatch", ex);
                // Keep processing error dispatch with regular request handle below
            }
            else {
                throw ex;
            }
        }
        ....
            return request;
    }
 ```
 > 结论：根据请求头 Content-Type 检查是否上传请求，如果是上传请求，则对请求进行解析包装请求为MultipartHttpServletRequest
- getHandler 方法
 ```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            // 循环处理器映射器，找到就直接返回
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 根据请求路径找到处理器
    Object handler = getHandlerInternal(request);
    ....
        // 创建处理器拦截器链
        HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    ....
        return executionChain;
}
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
                                   (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
    // 获取请求路径
    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
    // 遍历拦截器
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
        if (interceptor instanceof MappedInterceptor) {
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
            // 符合匹配条件的加入链中
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        }
        else {
            chain.addInterceptor(interceptor);
        }
    }
    return chain;
}
 ```
 > 结论：循环处理器映射器，找到对应的处理器 bean 对象，并匹配对应的拦截器对象，将处理器与拦截器组装为 HandlerExecutionChain
- getHandlerAdapter 方法
 ```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        // 循环处理器适配器
        for (HandlerAdapter adapter : this.handlerAdapters) {
            // 是否支持对应的处理器
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    ....
}
 ```
 > 结论：根据 handler 的实现方式匹配合适的处理器适配器对象
 >
 > 1.RequestMappingHandlerAdapter => @Controller 注解
 > 2.SimpleControllerHandlerAdapter => Controller 接口
 > 3.HttpRequestHandlerAdapter => HttpRequestHandler 接口
- applyPreHandle 方法
 ```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // 获取到拦截器
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        for (int i = 0; i < interceptors.length; i++) {
            HandlerInterceptor interceptor = interceptors[i];
            // 执行拦截器的 preHandle 方法
            if (!interceptor.preHandle(request, response, this.handler)) {
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
    }
    return true;
}
 ```
 > 结论：拦截器 preHandle 方法执行
- ha.handle 方法
 ```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
    throws Exception {
    // ① 入口
    return handleInternal(request, response, (HandlerMethod) handler);
}
protected ModelAndView handleInternal(HttpServletRequest request,
                                      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    ....
        // Execute invokeHandlerMethod in synchronized block if required.
        if (this.synchronizeOnSession) {
            HttpSession session = request.getSession(false);
            if (session != null) {
                Object mutex = WebUtils.getSessionMutex(session);
                synchronized (mutex) {
                    // ② 最终调用
                    mav = invokeHandlerMethod(request, response, handlerMethod);
                }
            }
            else {
                // ② 最终调用
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
    else {
        // ② 最终调用
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }
    ....
        return mav;
}
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                           HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    // 请求和响应包装在一个 ServletWebRequest 对象中
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        // @InitBinder
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        // @ModelAttribute
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
        // 创建 ServletInvocableHandlerMethod 对象
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        ....
            // ③ 执行方法
            invocableMethod.invokeAndHandle(webRequest, mavContainer);
        ....
            return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
                            Object... providedArgs) throws Exception {
    // ④ 执行方法
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    ....
        try {
            // ⑦ 返回值处理器处理返回值
            this.returnValueHandlers.handleReturnValue(
                returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
        }
    catch (Exception ex) {
        ....
    }
}
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
                               Object... providedArgs) throws Exception {
    // ⑤ 参数处理器处理请求参数
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }
    // ⑥ 反射执行请求方法
    return doInvoke(args);
}
 ```
 > 结论：根据对应的处理器适配器执行对应的方法，请求前使用参数处理器处理请求参数，执行完成后，使用返回值处理器处理返回值
- applyPostHandle 方法
 ```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
    throws Exception {
    // 获取到拦截器
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 循环执行拦截器的 postHandle 方法
        for (int i = interceptors.length - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            interceptor.postHandle(request, response, this.handler, mv);
        }
    }
}
 ```
 > 结论：拦截器 postHandle 方法执行
- processDispatchResult
 ```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
                                   @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
                                   @Nullable Exception exception) throws Exception {
    ....
        // Did the handler return a view to render?
        if (mv != null && !mv.wasCleared()) {
            // 视图渲染
            render(mv, request, response);
            ....
        }
    ....
}
 ```
 > 结论：视图渲染
- triggerAfterCompletion 方法
 ```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
    throws Exception {
    // 获取到拦截器
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 循环执行拦截器的 afterCompletion 方法
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            }
            catch (Throwable ex2) {
                logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
            }
        }
    }
}
 ```
 > 结论：拦截器 afterCompletion 方法执行
##### 4. 请求处理主流程图
![](./springmvc核心请求逻辑.png)
#### 5. Spring mvc简略图
![](./Spring MVC.png)
#### 6. Spring mvc 全流程简略图
![](./传统Spring MVC项目启动流程及jar启动流程.png)