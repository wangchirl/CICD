### Tomcat
#### 1、简述
- Tomcat 是一款开源的轻量级的 Web 服务器，实现了 Servlet 规范，提供 Http 服务
  - Java 语言开发
- 架构设计优雅，灵活使用模板方法模式，层次结构清晰
#### 2、架构图
- HTTP 服务器 + Servlet 容器
 ![1627606761419](.1627606761419.png)
- Connector + Container
- 处理Socket连接，负责网络字节流与Request和Response对象的转化
- 加载和管理Servlet，以及具体处理Request请求
 ![1627606858543](.1627606858543.png)
- 连接器组件
 ![](.1627630810666.png)
- EndPoint(AbstractEndpoint)
- Acceptor 用于监听Socket连接请求，通过 Poller 将请求转发给 SocketProcessor
- SocketProcessor 用于处理接收到的Socket请求，为了提高性能，SocketProcessor 被提交到线程池来执行
> Coyote 通信端点，即通信监听的接口，是具体Socket接收和发送处理器，是对传输层的抽象，因此EndPoint用来实现TCP/IP协议的
- Processor(Http11Processor)
> Processor用来实现HTTP协议，Processor接收来自EndPoint的Socket，读取字节流解析成Tomcat Request和Response对象
- ProtocolHandler(Http11NioProtocol)
> Coyote 协议接口， 通过Endpoint 和 Processor ， 实现针对具体协议的处理能力
- Adaptor(CoyoteAdaptor)
> CoyoteAdapter负责将Tomcat Request，Response转成ServletRequest，ServletResponse然后调用容器的 Pipeline 调用 Valve 组件
  ![1627633018218](.1627633018218.png)
- 整体架构图
 ![](./tomcat.jpg)
1. 客户端发起请求
2. Acceptor 接收 socket 连接，通过 Poller 将请求交给 SocketProcessor 处理
3. Http11Processor 构建 Tomcat 内部的 Request，Response
4. CoyoteAdaptor 将 Tomcat 内部的 Request，Response 转换为标准的 ServletRequest，ServletResponse
5. 依次调用容器组件的 Valve 组件
   - StandardEngine#StandardPipeline -> StandardEngineValve
   - StandardHost#StandardPipeline ->StandardHostValve
   - StandardContext#StandardPipeline -> StandardContextValve
   - StandardWrapper#StandardPipeline -> StandardWrapperValve
6. 在 StandardWrapperValve 中根据请求url或servlet名称匹配过滤器，并创建过滤器链ApplicationFilterChain，并执行ApplicationFilterChain#doFilter，依次调用匹配上的过滤器的doFilter 方法，最后调用 servlet的service方法处理请求
- Catalina 各个组件的职责

  
| 组件       | 职责 |
| ----      | ----|
| Catalina | 负责解析Tomcat的配置文件 , 以此来创建服务器Server组件，并根据命令来对其进行管理 |
| Server  | 服务器表示整个Catalina Servlet容器以及其它组件，负责组装并启动Servlet引擎,Tomcat连接器。Server通过实现Lifecycle接口，提供了一种优雅的启动和关闭整个系统的方式 |
| Service  | 服务是Server内部的组件，一个Server包含多个Service。它将若干个Connector组件绑定到一个Container（Engine）上 |
| Connector | 连接器，处理与客户端的通信，它负责接收客户请求，然后转给相关的容器处理，最后向客户返回响应结果 |
| Container | 容器，负责处理用户的servlet请求，并返回对象给web用户的模块  |
- Container 结构


| 容器  | 说明                             |
| ------- | ------------------------------------------------------------ |
| Engine | 表示整个Catalina的Servlet引擎，用来管理多个虚拟站点，一个Service最多只能有一个Engine，但是一个引擎可包含多个Host |
| Host  | 代表一个虚拟主机，或者说一个站点，可以给Tomcat配置多个虚拟主机地址，而一个虚拟主机下可包含多个Context |
| Context | 表示一个Web应用程序， 一个Web应用可包含多个Wrapper      |
| Wrapper | 表示一个Servlet，Wrapper 作为容器中的最底层，不能包含子容器 |
 ```xml
<Server>
    <Linstener />
    <Linstener />
    <GlobalNamingResources>
        <Resource />
    </GlobalNamingResources>
    <Service>
        <Connector />
        <Connector />
        <Executor />
        <Executor />
        <Engine>
            <Host>
                <Context>
                    <Wrapper>
                    </Wrapper>
                </Context>
            </Host>
        </Engine>
    </Service>
</Server>
 ```
 ![1627638684595](.1627638684595.png)
#### 3、源码分析
##### 1. [tomcat-8.5.56](https://github.com/wangchirl/apache-tomcat-8.5.56-src)
> 启动参数：
> ‐Dcatalina.home=D:/workspace/apache‐tomcat‐8.5.56‐src/home
> ‐Dcatalina.base=D:/workspacet/apache‐tomcat‐8.5.56‐src/home
> ‐Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
> ‐Djava.util.logging.config.file=D:/workspace/apache‐tomcat‐8.5.56‐src/home/conf/logging.properties
> 可能遇到的问题：
> 1. JSP解析器缺失
>  ContextConfig#configureStart方法的 webConfig(); 代码块下添加 JSP解析器
>  ```java
>  webConfig();
>  // JSP Parse ...
>  context.addServletContainerInitializer(new JasperInitializer(),null);
>  ```
##### 2. 源码目录说明
| 目录名称      | 基本功能             |
| ------------------- | --------------------------------- |
| org.apache.catalina | Servlet 容器相关组件       |
| org.apache.coyote  | 连接器相关组件（ProtocolHandler） |
| org.apache.el    | EL 表达式相关功能         |
| org.apache.jasper  | JSP 相关功能组件         |
| org.apache.juli   | 日志相关功能组件         |
| org.apache.naming  | 命名服务相关功能组件       |
| org.apache.tomcat  | 一些公共的相关组件        |
![1627865759301](.1627865759301.png)
##### 3. Catalina 容器
![1627865830444](.1627865830444.png)
##### 4. 各层基础组件
![1627889787076](.1627889787076.png)
![1627889811071](.1627889811071.png)
##### 5. Tomcat 启动流程
###### 1. 启动流程图
![1627869947250](.1627869947250.png)
###### 2. 启动流程步骤
- `init 阶段`
   - 启动 tomcat，bin/startup.sh ，调用 catalina.sh
   - catalina.sh 脚本调用 Bootstrap 中的 main 方法
   - Bootstrap 的 main 方法中调用了 init 方法，来创建 Catalina 及 初始化 ClassLoader 类加载器
   - Bootstrap 的 main 方法调用 load 方法，内部反射调用 Catalina 的 load 方法
   - Catalina 的 load 方法中进行一些初始化工作，并对构建 Digester对象对 server.xml 进行解析，对 server.xml 中配置的内容进行对象创建，比如 StandardServer 对象
   - 然后调用 StandardServer#init 方法，内部循环调用 StandardService#init 方法
   - StandardService#init 方法内部调用 StandardEngine#init 方法、StandardThreadExecutor#init 方法、Connector#init 方法
   - 这里主要跟 Connector#init 方法，在Connector#init 方法内部，创建 CoyoteAdapter 并调用 AbstractHttp11Protocol#init 方法，并在其父类 AbstractProtocol#init 方法中调用 NioEndPoint#init 方法
   - 在 NioEndPoint 父类 AbstractEndpoint 中的 bind 方法进行端口绑定
- `start 阶段`
   - Bootstrap 的 main 方法调用了 start 方法，内部反射调用 Catalina 的 start 方法
   - Catalina#start 方法内部调用 StandardServer#start 方法
   - StandardServer#start 方法内部循环调用 StandardService#start 方法
   - StandardService#start 方法内部调用 StandardEngine#start 方法、StandardThreadExecutor#start 方法、Connector#start 方法
 > StandardEngine#start 方法内部使用异步的方式调用其子容器 StandardHost 的 start 方法 
 > ```java
 > for (Container child : children) {
 >     // child => StandardHost
 >     results.add(startStopExecutor.submit(new StartChild(child)));
 > }
 > ```
 >
 > 同样的情况，在 StandardHost 设置 state 状态时发布的事件，最终在HostConfig 中异步方式调用 StandardContext#start 方法
 >
 >```java
 > results.add(es.submit(new DeployWar(this, cn, war)));
 > ```
 - Connector#start 方法内部调用 AbstractProtocol#start 方法，内部调用 NioEndPoint#start 方法
 - 在NioEndPoint#start 方法中，创建 Poller 数组对象(默认大小2)及 Acceptor 数组对象(默认大小1)，二者皆实现了 Runnable 接口，在创建完成后启动线程，至此启动完成，准备接受请求
 > 请求接收线程 Acceptor 启动，Poller 线程启动，run 方法是其具体的处理逻辑
###### 3. 源码分析
 `init阶段`
- Bootstrap#main
 ```java
// 入口
public static void main(String args[]) {
    synchronized (daemonLock) {
        if (daemon == null) {
            // 创建 Bootstrap 对象
            Bootstrap bootstrap = new Bootstrap();
            try{
                // ① 调用 Bootstrap的init方法
                bootstrap.init();
            } catch(Throwable t){
                ....
            }
            // daemon 持有 Bootstrap的引用
            daemon = bootstrap;
        }
    }
    try{
        ....
            // ② > load方法，内部调用 Catalina#load => Digester创建 Server，并调用其init
            daemon.load(args);
        	// ③ start方法
        	daemon.start();
        ....
    } 
}
}
// ① BootStrap#init
public void init() throws Exception {
    // ① 初始化类加载器，common、server、shared
    initClassLoaders();
    // 设置线程上下文类加载器，目的是在父类加载器中可加载子类加载器管辖的类
    Thread.currentThread().setContextClassLoader(catalinaLoader);
    ....
        // catalinaLoader类加载器加载 Catalina 类  
        Class<?> startupClass = catalinaLoader.
        loadClass("org.apache.catalina.startup.Catalina");
    // 反射创建Catalina对象
    Object startupInstance = startupClass.getConstructor().newInstance();
    ....
        // 找到Catalina的setParentClassLoader方法，设置父类加载器为sharedLoader  
        String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    // Catalina的setParentClassLoader调用
    method.invoke(startupInstance, paramValues);
    // catalinaDaemon持有Catalina对象的引用
    catalinaDaemon = startupInstance;
}
// ② Bootstrap#load
private void load(String[] arguments) throws Exception {
    // 找到 Catalina的load方法并执行
    String methodName = "load";
    ....
        Method method =
        catalinaDaemon.getClass().getMethod(methodName, paramTypes);
    ....
        // 反射执行 Catalina#load方法  
        method.invoke(catalinaDaemon, param);
}
 ```
- Catalina#load
 ```java
public void load() {
    ....
        // ① 使用 Digester 解析 server.xml 
        Digester digester = createStartDigester();
    ....
        try {
            ....
                // ② 这里解析过程中会创建 server.xml中配置的对象，比如 StandardServer  
                digester.parse(inputSource);
        } catch (SAXParseException spe) {
            ....
        }
    ....
        try {
            // ③ 调用 StandardServer#init 方法，模板方法模式，最终调用 initInternal 方法
            getServer().init();
        } catch (LifecycleException e) {
            ....
        }
}
 ```
- StandardServer#initInternal
 ```java
protected void initInternal() throws LifecycleException {
    ....
        // 循环调用 StandardService 的init方法，模板方法模式，最终调用 initInternal 方法
        for (Service service : services) {
            service.init();
        }
}
 ```
- StandardService#init
 ```java
protected void initInternal() throws LifecycleException {
    if (engine != null) {
        // ① StandardEngine#init方法，模板方法模式，最终调用 initInternal 方法
        engine.init();
    }
    for (Executor executor : findExecutors()) {
        ....
            // ② Executor#init方法，模板方法模式，最终调用 initInternal 方法  
            executor.init();
    }
    // ③ Mapper#init方法，模板方法模式，最终调用 initInternal 方法
    mapperListener.init();
    // ④ 循环执行 Connector#init方法，模板方法模式，最终调用 initInternal 方法
    synchronized (connectorsLock) {
        for (Connector connector : connectors) {
            try {
                connector.init();
            } catch (Exception e) {
                ....
            }
        }
    }
}
 ```
- Connector#init
 ```java
protected void initInternal() throws LifecycleException {
    ....
        // ① 创建 CoyoteAdaptor 对象  
        adapter = new CoyoteAdapter(this);
    protocolHandler.setAdapter(adapter);
    ....
        try {
            // ② AbstractHttp11Protocol#init 方法，模板方法模式，最终调用 initInternal 方法
            protocolHandler.init();
        } catch (Exception e) {
            ....
        }
}
 ```
- AbstractHttp11Protocol#init
 ```java
public void init() throws Exception {
    // 父类 AbstractProtocol#init 方法
    super.init();
}
 ```
- AbstractProtocol#init
 ```java
public void init() throws Exception {
    ....
        // NioEndpoint#init方法，最终父类 AbstractEndpoint#init
        endpoint.init();
}
 ```
- AbstractEndpoint#init
 ```java
public void init() throws Exception {
    if (bindOnInit) {
        // 绑定端口，NioEndpoint#bind
        bind();
        bindState = BindState.BOUND_ON_INIT;
    }
    ....
}
 ```
- NioEndpoint#bind
 ```java
public void bind() throws Exception {
    if (!getUseInheritedChannel()) {
        // java nio api
        serverSock = ServerSocketChannel.open();
        socketProperties.setProperties(serverSock.socket());
        InetSocketAddress addr = (getAddress()!=null? 
                                  new InetSocketAddress(getAddress(),getPort()) :
                                  new InetSocketAddress(getPort()));
        // socket绑定端口
        serverSock.socket().bind(addr,getAcceptCount());
    } else {
        ....
    }
    // 服务端接收请求的 serverSocket 是阻塞的
    serverSock.configureBlocking(true); //mimic APR behavior
    ....
}
 ```
`start阶段`

- Bootstrap#main
 ```java
public static void main(String args[]) {
    synchronized (daemonLock) {
        if (daemon == null) {
            // 创建 Bootstrap 对象
            Bootstrap bootstrap = new Bootstrap();
            try{
                // ① 调用 Bootstrap的init方法
                bootstrap.init();
            } catch(Throwable t){
                ....
            }
            // daemon 持有 BootStrap的引用
            daemon = bootstrap;
        }
    }
    try{
        ....
            // ② load方法，内部调用 Catalina#load => Digester创建 Server，并调用其 init
            daemon.load(args);
        // ③ > start方法
        daemon.start();
        ....
    } 
}
}
// Bootstrap#start
public void start() throws Exception {
    // 反射执行 Catalina的start方法
    Method method = catalinaDaemon.getClass().getMethod("start", (Class [])null);
    method.invoke(catalinaDaemon, (Object [])null);
}
 ```
- Catalina#start
 ```java
public void start() {
    ....
        try {
            // StandardServer#start 方法，模板方法模式，最终调用 initInternal 方法
            getServer().start();
        } catch (LifecycleException e) {
            ....
        }
    ....
}
 ```
- StandardServer#start
 ```java
protected void startInternal() throws LifecycleException {
    ....
        synchronized (servicesLock) {
        // 循环执行 StandardService#start方法，模板方法模式，最终调用 initInternal 方法
        for (Service service : services) {
            service.start();
        }
    }
}
 ```
- StandardService#start
 ```java
protected void startInternal() throws LifecycleException {
    ....
        if (engine != null) {
            synchronized (engine) {
                // ① StandardEngine#start方法，模板方法模式，最终调用 initInternal 方法
                engine.start();
            }
        }
    synchronized (executors) {
        // ② Executor#start方法，模板方法模式，最终调用 initInternal 方法
        for (Executor executor: executors) {
            executor.start();
        }
    }
    // ③ StandardEngine#start方法，模板方法模式，最终调用 initInternal 方法
    mapperListener.start();
    // ④ 循环执行 Connector#start 方法，模板方法模式，最终调用 initInternal 方法
    synchronized (connectorsLock) {
        for (Connector connector: connectors) {
            try {
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            } catch (Exception e) {
                ....
            }
        }
    }
}
 ```
- StandardEngine#start
 ```java
// ContainerBase#startInternal
protected synchronized void startInternal() throws LifecycleException {
    ....
        // 找到子容器，也即 StandardHost  
        Container children[] = findChildren();
    List<Future<Void>> results = new ArrayList<>();
    // 交给线程池去异步执行
    for (Container child : children) {
        results.add(startStopExecutor.submit(new StartChild(child)));
    }
    ....
}
 ```
- StandardHost#start
 ```java
protected synchronized void startInternal() throws LifecycleException {
    ....
        super.startInternal();
}
// ContainerBase#startInternal
protected synchronized void startInternal() throws LifecycleException {
    .... 
        setState(LifecycleState.STARTING);
    ....
}
// LifecycleBase#setState
protected synchronized void setState(LifecycleState state) throws LifecycleException {
    setStateInternal(state, null, true);
}
// LifecycleBase#setStateInternal
private synchronized void setStateInternal(LifecycleState state, Object data, 
                                           boolean check) throws LifecycleException {
    ....
        this.state = state;
    String lifecycleEvent = state.getLifecycleEvent();
    if (lifecycleEvent != null) {
        // 发布事件
        fireLifecycleEvent(lifecycleEvent, data);
    }
}
// LifecycleBase#fireLifecycleEvent
protected void fireLifecycleEvent(String type, Object data) {
    LifecycleEvent event = new LifecycleEvent(this, type, data);
    for (LifecycleListener listener : lifecycleListeners) {
        listener.lifecycleEvent(event);
    }
}
// HostConfig#lifecycleEvent
public void lifecycleEvent(LifecycleEvent event) {
    ....
        // Process the event that has occurred
        if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
            ....
        } else if (event.getType().equals(Lifecycle.START_EVENT)) {
            // 启动
            start();
        } 
    ....
}
// HostConfig#start
public void start() {
    ....
        if (host.getDeployOnStartup())
            // 部署项目
            deployApps();
}
// HostConfig#deployApps，在部署方式代码中调用 StandardContext#start方法
protected void deployApps() {
    File appBase = host.getAppBaseFile();
    File configBase = host.getConfigBaseFile();
    String[] filteredAppPaths = filterAppPaths(appBase.list());
    // 部署方式一
    deployDescriptors(configBase, configBase.list());
    // 部署方式二
    deployWARs(appBase, filteredAppPaths);
    // 部署方式三
    deployDirectories(appBase, filteredAppPaths);
}
 ```
- StandardContext#start
 ```java
protected synchronized void startInternal() throws LifecycleException {
    ....
        // 创建 WebappLoader，setLoader方法回调 start 方法创建 ClassLoader 
        if (getLoader() == null) {
            WebappLoader webappLoader = new WebappLoader();
            webappLoader.setDelegate(getDelegate());
            setLoader(webappLoader);
        }
    ....
        // ① 这里将调用 StandardWrapper#start方法
        // 构建 ContextConfig 创建 Wrapper
        fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);
    ....
        // ② 调用所有实现了 ServletContainerInitializer 接口的类的 onStartup 方法
        // springmvc spi => SpringServletContainerInitializer 
        for (Map.Entry<ServletContainerInitializer, Set<Class<?>>> entry :
             initializers.entrySet()) {
            try {
                entry.getKey().onStartup(entry.getValue(),
                                         getServletContext());
            } catch (ServletException e) {
                log.error(sm.getString("standardContext.sciFail"), e);
                ok = false;
                break;
            }
        }
    if (ok) {
        // ③ 调用所有实现了 ServletContextListener 的监听器的contextInitialized方法
        //  常见的是 springmvc 中的 ContextLoaderListener
        if (!listenerStart()) {
            log.error(sm.getString("standardContext.listenerFail"));
            ok = false;
        }
    }
    ....
        if (ok) {
            // ④ 过滤器处理
            if (!filterStart()) {
                log.error(sm.getString("standardContext.filterFail"));
                ok = false;
            }
        }
    if (ok) {
        // ⑤ Servlet 初始化 load on startup 为大于 0 的 Servlet
        if (!loadOnStartup(findChildren())){
            log.error(sm.getString("standardContext.servletFail"));
            ok = false;
        }
    }
    ....
}
 ```
- Connector#start
 ```java
protected void startInternal() throws LifecycleException {
    ....
        try {
            // Http11NioProtocol#start 
            protocolHandler.start();
        } catch (Exception e) {
            ....
        }
}
 ```
- Http11NioProtocol#start
 ```java
// Http11NioProtocol父类AbstractProtocol#start
public void start() throws Exception {
    ....
        // NioEndpoint#start 
        endpoint.start();
    ....
}
 ```
- NioEndpoint#start
 ```java
// 父类 AbstractEndpoint#start
public final void start() throws Exception {
    ....
        // NioEndpoint#startInternal  
        startInternal();
}
public void startInternal() throws Exception {
    if (!running) {
        ....
            // 创建 Poller 并启动线程，默认个数 2
            pollers = new Poller[getPollerThreadCount()];
        for (int i=0; i<pollers.length; i++) {
            pollers[i] = new Poller();
            Thread pollerThread = new Thread(pollers[i], getName() + 
                                             "-ClientPoller-"+i);
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();
        }
        // 创建 Acceptor 并启动线程
        startAcceptorThreads();
    }
}
// startAcceptorThreads
protected final void startAcceptorThreads() {
    // Acceptor 默认个数 1
    int count = getAcceptorThreadCount();
    acceptors = new Acceptor[count];
    // 创建 Acceptor 并启动线程
    for (int i = 0; i < count; i++) {
        acceptors[i] = createAcceptor();
        String threadName = getName() + "-Acceptor-" + i;
        acceptors[i].setThreadName(threadName);
        Thread t = new Thread(acceptors[i], threadName);
        t.setPriority(getAcceptorThreadPriority());
        t.setDaemon(getDaemon());
        t.start();
    }
}
 ```
> 至此，启动完成，准备接受请求，使用 Acceptor 接受请求，通过 Poller 对请求进行分派
##### 6. Tomcat 请求处理流程
> - Tomcat是怎么确定每一个请求应该由哪个Wrapper容器里的Servlet来处理的呢？
>
>  > Tomcat是用Mapper组件来完成这个任务的，原理是：
>  >
>  > Mapper组件里保存了Web应用的配置信息，其实就是容器组件与访问路径的映射关系，比如Host容器里配置的域名、Context容器里的Web应用路径，以及Wrapper容器里Servlet映射的路径，你可以想象这些配置信息就是一个多层次的Map，当一个请求到来时，Mapper组件通过解析请求URL里的域名和路径，再到自己保存的Map里去查找，就能定位到一个Servlet。请你注意，一个请求URL最后只会定位到一个Wrapper容器，也就是一个Servlet
>
> ![1627890009459](.1627890009459.png)
###### 1. 请求流程图
![1627890061105](.1627890061105.png)
###### 2. 请求流程步骤
- Connector组件Endpoint中的Acceptor监听客户端套接字连接并接收Socket
- 将连接通过 Poller 交给线程池 Executor 处理，开始执行请求响应任务
- Processor组件读取消息报文，解析请求行、请求体、请求头，封装成tomcat内部Request对象
- Mapper组件根据请求行的URL值和请求头的Host值匹配由哪个Host容器、Context容器、Wrapper容器处理请求
- CoyoteAdaptor组件负责将Connector组件和Engine容器关联起来，将tomcat内部的Request、Response转为ServletRequest、ServletResponse，并把生成的Request对象和响应对象Response传递到Engine容器中，调用 Pipeline
- Engine容器的管道开始处理，管道中包含若干个Valve、每个Valve负责部分逻辑处理。执行完Valve后会执行基础的 Valve--StandardEngineValve，负责调用Host容器的Pipeline
- Host容器的管道开始处理，流程类似，最后执行 Context容器的Pipeline
- Context容器的管道开始处理，流程类似，最后执行 Wrapper容器的Pipeline
- Wrapper容器的管道开始处理，流程类似，调用 StandardWrapperValve，在StandardWrapperValve 找到请求匹配的Servlet，并匹配对应的过滤器，封装为 ApplicationFilterChain 对象
- ApplicationFilterChain#doFilter方法中依次执行过滤器 doFilter方法，最后执行Servlet对象的service方法
![请求流程步骤图](.1627633018218.png)
###### 3. 源码分析
- NioEndpoint$Acceptor#run
 ```java
public void run() {
    ....
        while (running) {
            ....
                try {
                    ....
                        try {
                            // ① 请求未到达时在这里阻塞 serverSock.configureBlocking(true)
                            socket = serverSock.accept();
                        } catch (IOException ioe) {
                            ....
                        }
                    ....
                        if (running && !paused) {
                            // ② 请求到达后调用 setSocketOptions
                            if (!setSocketOptions(socket)) {
                                closeSocket(socket);
                            }
                        } else {
                            closeSocket(socket);
                        }
                } catch (Throwable t) {
                    ....
                }
        }
    ....
}
 ```
- NioEndpoint#setSocketOptions
 ```java
protected boolean setSocketOptions(SocketChannel socket) {
    try {
        ....
            // 调用 Poller 分派请求，最终到达 Poller#run 方法 
            getPoller0().register(channel);
    } catch (Throwable t) {
        ....
    }
    return true;
}
 ```
- NioEndpoint$Poller#run
 ```java
public void run() {
    // 一直循环
    while (true) {
        ....
            Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            ....
                if (attachment == null) {
                    iterator.remove();
                } else {
                    iterator.remove();
                    // ① 有事件产生，进行处理，内部调用 processSocket 方法将请求交给线程池处理
                    processKey(sk, attachment);
                }
        }//while
        ....
    }//while
    ....
}
// processSocket 方法
public boolean processSocket(SocketWrapperBase<S> socketWrapper,
                             SocketEvent event, boolean dispatch) {
    try {
        ....
            SocketProcessorBase<S> sc = processorCache.pop();
        ....
            // 获取线程池，将 SocketProcessor 交给线程池处理
            Executor executor = getExecutor();
        if (dispatch && executor != null) {
            executor.execute(sc);
        } 
        ....
    } catch (RejectedExecutionException ree) {
        ....
    }
    return true;
}
 ```
- NioEndpoint$SocketProcessor#doRun
 ```java
protected void doRun() {
    NioChannel socket = socketWrapper.getSocket();
    SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
    try {
        ....
            if (handshake == 0) {
                SocketState state = SocketState.OPEN;
                // Process the request from this socket
                if (event == null) {
                    state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                } else {
                    // 得到 ProtocolHandler(AbstractProtocol) 处理请求
                    state = getHandler().process(socketWrapper, event);
                }
                ....
            }
        ....
    } catch (CancelledKeyException cx) {
        ....
    }
    ....
}
 ```
- ProtocolHandler#process
 ```java
public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
    ....
        try {
            if (processor == null) {
                ....
                    if (processor == null) {
                        // ① 创建 Processor（Http11Processor）
                        processor = getProtocol().createProcessor();
                        ....
                    }
                ....
                    do {
                        // ② Http11Processor#process方法，最终调用 service方法
                        state = processor.process(wrapper, status);
                        ....
                    }
            }
 ```
- Http11Processor#service
 ```java
public SocketState service(SocketWrapperBase<?> socketWrapper)
    throws IOException {
    ....
        // 得到 CoyoteAdaptor，调用其 service 方法
        getAdapter().service(request, response);
    ....
}
 ```
- CoyteAdaptor#service
 ```java
public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
    throws Exception {
    ....
        // tomcat的 Request 对象转 Servlet的 Request 对象
        // tomcat的 Response 对象转 Servlet的 Response 对象
        postParseSuccess = postParseRequest(req, request, res, response);
    if (postParseSuccess) {
        ....
            // 调用 Container（StandardEngine）的 Pepiline，执行其 Valve
            // 链式调用 StandardEngineValve -> StandardHostValve -> 
            // StandardContextValve -> StandardWrapperValve
            connector.getService().getContainer().getPipeline().getFirst()
            .invoke(request, response);
    }
    ....
}
// StandardEngineValve#invoke
public final void invoke(Request request, Response response)
    throws IOException, ServletException {
    ....
        host.getPipeline().getFirst().invoke(request, response);
}
// StandardHostValve#invoke
public final void invoke(Request request, Response response)
    throws IOException, ServletException {
    ....
        context.getPipeline().getFirst().invoke(request, response);
    ....
}
// StandardContextValve#invoke
public final void invoke(Request request, Response response)
    throws IOException, ServletException {
    ....
        wrapper.getPipeline().getFirst().invoke(request, response);
}
// StandardWrapperValve#invoke
public final void invoke(Request request, Response response)
    throws IOException, ServletException {
    ....
        // Allocate a servlet instance to process this request
        try {
            if (!unavailable) {
                // ① 得到 Servlet
                servlet = wrapper.allocate();
            }
        } catch (UnavailableException e) {
            ....
        }
    // ② 创建过滤器链
    ApplicationFilterChain filterChain =
        ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
    try {
        if ((servlet != null) && (filterChain != null)) {
            ....
        } else {
            if (request.isAsyncDispatching()) {
                ....
            } else {
                // ③ 执行过滤器的逻辑，在执行完过滤器链后，执行Servlet的service方法
                filterChain.doFilter(request.getRequest(), 
                                     response.getResponse());
            }
        }
    }
} catch (ClientAbortException | CloseNowException e) {
    ....
}
}
 ```
- ApplicationFilterChain#doFilter
 ```java
public void doFilter(ServletRequest request, ServletResponse response)
    throws IOException, ServletException {
    if( Globals.IS_SECURITY_ENABLED ) {
        ....
    } else {
        // 执行内部方法
        internalDoFilter(request,response);
    }
}
// internalDoFilter 方法
private void internalDoFilter(ServletRequest request,
                              ServletResponse response)
    throws IOException, ServletException {
    // ① 执行每一个匹配的过滤器的doFilter方法
    if (pos < n) {
        ApplicationFilterConfig filterConfig = filters[pos++];
        try {
            Filter filter = filterConfig.getFilter();
            ....
                filter.doFilter(request, response, this);
            ....
        } catch (IOException | ServletException | RuntimeException e) {
            ....
        }
        // 这里 return 表示 Servlet#service方法只能被调用一次
        return;
    }
    try {
        ....
            // ② 执行 Servlet#service 方法
            servlet.service(request, response);
        ....
    } catch (IOException | ServletException | RuntimeException e) {
        ....
    }
}
 ```
> 至此，tomcat 已经将请求交给具体的 Servlet 的 service 方法进行处理了，经历的过程有：
>
> Valve链 -> ApplicationFilterChain 链(过滤器的doFilter方法) -> service 方法
>
> ![](./tomcat.jpg)
#### 4.Jasper 编译JSP原理
> Tomcat 在默认的web.xml 中配置了一个org.apache.jasper.servlet.JspServlet，用于处理所有的.jsp 或 .jspx 结尾的请求，该Servlet 实现即是运行时编译的入口，其是 Servlet 接口的实现类，遵循 Servlet 的生命周期方法 init -> service -> destroy
###### 1. 原理图
![1627894701339](.1627894701339.png)
###### 2. JSP 编译过程图
![1627897046539](.1627897046539.png)
###### 3. 源码分析
- JspServlet#service
 ```java
public void service (HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
    // jsp 文件路径
    String jspUri = jspFile;
    if (jspUri == null) {
        ....
            // 获取 jsp 的路径
            jspUri = request.getServletPath();
        String pathInfo = request.getPathInfo();
        if (pathInfo != null) {
            jspUri += pathInfo;
        }
    }
    ....
        try {
            // 判定是否预编译请求
            boolean precompile = preCompile(request);
            // ① serviceJspFile
            serviceJspFile(request, response, jspUri, precompile);
        } catch (RuntimeException e) {
            ....
        }
}
// serviceJspFile 方法
private void serviceJspFile(HttpServletRequest request,
                            HttpServletResponse response, String jspUri,
                            boolean precompile)
    throws ServletException, IOException {
    // 获取 JspServletWrapper 
    JspServletWrapper wrapper = rctxt.getWrapper(jspUri);
    ....
        try {
            // JspServletWrapper#service
            wrapper.service(request, response, precompile);
        } catch (FileNotFoundException fnfe) {
            ....
        }
}
 ```
- JspServletWrapper#service
 ```java
public void service(HttpServletRequest request,
                    HttpServletResponse response,
                    boolean precompile)
    throws ServletException, IOException, FileNotFoundException {
    Servlet servlet;
    try {
        ....
            // ① 编译
            if (options.getDevelopment() || mustCompile) {
                synchronized (this) {
                    if (options.getDevelopment() || mustCompile) {
                        // JspCompilationContext#compile 编译 jsp
                        ctxt.compile();
                        mustCompile = false;
                    }
                }
            } else {
                ....
            }
        // ② 获取编译后的 servlet，并销毁旧的 servlet
        servlet = getServlet();
        ....
    } catch (ServletException ex) {
        ....
    }
    try {
        ....
            // ③ servlet#service 方法
            if (servlet instanceof SingleThreadModel) {
                synchronized (this) {
                    servlet.service(request, response);
                }
            } else {
                servlet.service(request, response);
            }
    } catch (UnavailableException ex) {
        ....
    }
}
// getServlet 方法
public Servlet getServlet() throws ServletException {
    if (getReloadInternal() || theServlet == null) {
        synchronized (this) {
            if (getReloadInternal() || theServlet == null) {
                // 调用旧theServlet对象的destroy方法
                destroy();
                final Servlet servlet;
                try {
                    // 创建新 servlet 对象
                    InstanceManager instanceManager = 
                        InstanceManagerFactory.getInstanceManager(config);
                    servlet = (Servlet) instanceManager.newInstance(ctxt.getFQCN(), 
                                                                    ctxt.getJspLoader());
                } catch (Exception e) {
                    ....
                }
                // servlet#init 方法
                servlet.init(config);
                // 新Servlet对象覆盖旧Servlet对象
                theServlet = servlet;
                reload = false; 
            }
        }
    }
    return theServlet;
}
 ```
- JspCompilationContext#compile
 ```java
public void compile() throws JasperException, FileNotFoundException {
    // 创建编译器
    createCompiler();
    if (jspCompiler.isOutDated()) {
        ....
            try {
                ....
                    // Compiler#compile 进行编译  
                    jspCompiler.compile();
                ....
            } catch (JasperException ex) {
                ....
            }
    }
}
 ```
- Compiler#compile
 ```java
// 重载方法
public void compile(boolean compileClass, boolean jspcMode)
    throws FileNotFoundException, JasperException, Exception {
    ....
        try {
            // 最后修改时间
            final Long jspLastModified = ctxt.getLastModified(ctxt.getJspFile());
            // ① 生成 java 文件 .java
            String[] smap = generateJava();
            ....
                if (compileClass) {
                    // ② 生成 class 文件 .class
                    generateClass(smap);
                    ....
                }
        } finally {
            ....
        }
}
 ```
- 查看生成的 java及 class 文件
 tomcat的 web.xml 中 jsp servlet配置中配置如下参数
 ```xml
<init-param>
    <param-name>scratchdir</param-name>
    <param-value>D:/tem/jsp/</param-value>
</init-param>
 ```
#### 5. tomcat 配置文件详解
##### 1. server.xml
- Server
 > Server是server.xml的根元素，用于创建一个Server实例，默认使用的实现类是 StandardServer
 >
 > Server内嵌的子元素为 Listener、GlobalNamingResources、Service
 ```xml
<Server port="8005" shutdown="SHUTDOWN">
    ....
</Server>
 ```
 - port：Tomcat 监听的关闭服务器的端口
 - shutdown：关闭服务器的指令字符串
- Listener
 > Server 内嵌默认5个 Listener
 ```xml
<!‐‐ 用于以日志形式输出服务器 、操作系统、JVM的版本信息 ‐‐>
<Listener className="org.apache.catalina.startup.VersionLoggerListener"/>
<!‐‐ 用于加载（服务器启动） 和 销毁 （服务器停止） APR。 如果找不到APR库， 则会
输出日志， 并不影响Tomcat启动 ‐‐>
<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
<!‐‐ 用于避免JRE内存泄漏问题 ‐‐>
<Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
<!‐‐ 用户加载（服务器启动） 和 销毁（服务器停止） 全局命名服务 ‐‐>
<Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener"/>
<!‐‐ 用于在Context停止时重建Executor 池中的线程， 以避免ThreadLocal 相关的内存泄漏 ‐‐>
<Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
 ```
- GlobalNamingResources 
 > 全局命名服务
 ```xml
<!‐‐ Global JNDI resources Documentation at /docs/jndi‐resources‐howto.html ‐‐>
<GlobalNamingResources>
    <!‐‐ Editable user database that can also be used by
    UserDatabaseRealm to authenticate users
    ‐‐>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat‐users.xml" />
</GlobalNamingResources>
 ```
- Service
 > 该元素用于创建 Service 实例,默认使用 StandardService，Tomcat 仅指定了Service 的名称， 值为 "Catalina"，一个Server服务器，可以包含多个Service服务
 >
 > Service 可以内嵌的元素为 ： Listener、Executor、Connector、Engine
 >
 > - Listener 用于为Service添加生命周期监听器
 > - Executor 用于配置Service 共享线程池
 > - Connector 用于配置 Service 包含的链接器
 > - Engine 用于配置Service中链接器对应的Servlet 容器引擎
 ```xml
<Service name="Catalina">
    ....
</Service>
 ```
- Executor
 > 默认情况下，Service 并未添加共享线程池配置，可进行配置
 ```xml
<Executor name="tomcatThreadPool"
          namePrefix="catalina‐exec‐"
          maxThreads="200"
          minSpareThreads="100"
          maxIdleTime="60000"
          maxQueueSize="Integer.MAX_VALUE"
          prestartminSpareThreads="false"
          threadPriority="5"
          className="org.apache.catalina.core.StandardThreadExecutor"/>
 ```
 属性说明：
| 属性          | 含义                             |
| ----------------------- | ------------------------------------------------------------ |
| name          | 线程池名称，用于 Connector中指定               |
| namePrefix       | 所创建的每个线程的名称前缀，一个单独的线程名称为namePrefix+threadNumber |
| maxThreads       | 池中最大线程数                        |
| minSpareThreads     | 活跃线程数，也就是核心池线程数，这些线程不会被销毁，会一直存在 |
| maxIdleTime       | 线程空闲时间，超过该时间后，空闲线程会被销毁，默认值为6000（1分钟），单位毫秒 |
| maxQueueSize      | 在被执行前最大线程排队数目，默认为Int的最大值，也就是广义的无限。除非特殊情况，这个值不需要更改，否则会有请求不会被处理的情况发生 |
| prestartminSpareThreads | 启动线程池时是否启动 minSpareThreads部分线程。默认值为false，即不启动 |
| threadPriority     | 线程池中线程优先级，默认值为5，值从1到10           |
| className        | 线程池实现类，未指定情况下，默认实现类为org.apache.catalina.core.StandardThreadExecutor。如果想使用自定义线程池首先需要实现org.apache.catalina.Executor接口 |
- Connector
 > Connector 用于创建链接器实例，默认情况下，server.xml 配置了两个链接器，一个支持HTTP协议，一个支持AJP协议
 ```xml
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000"
           redirectPort="8443" />
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
<!-- 完整Connector -->
<Connector port="8080"
           protocol="HTTP/1.1"
           executor="tomcatThreadPool"
           maxThreads="1000"
           minSpareThreads="100"
           acceptCount="1000"
           maxConnections="1000"
           connectionTimeout="20000"
           compression="on"
           compressionMinSize="2048"
           disableUploadTimeout="true"
           redirectPort="8443"
           URIEncoding="UTF‐8" />
 ```
 - port：端口号，Connector 用于创建服务端Socket 并进行监听， 以等待客户端请求链接。如果该属性设   置为0，Tomcat将会随机选择一个可用的端口号给当前Connector使用
 - protocol：当前Connector 支持的访问协议。 默认为 HTTP/1.1 ， 并采用自动切换机制选择一个基于JAVA NIO 的链接器或者基于本地APR的链接器（根据本地是否含有Tomcat的本地库判定）
  ```java
// HTTP 协议
org.apache.coyote.http11.Http11NioProtocol ， 非阻塞式 Java NIO 链接器
org.apache.coyote.http11.Http11Nio2Protocol ， 非阻塞式 JAVA NIO2 链接器
org.apache.coyote.http11.Http11AprProtocol ， APR 链接器
// AJP 协议
org.apache.coyote.ajp.AjpNioProtocol ， 非阻塞式 Java NIO 链接器
org.apache.coyote.ajp.AjpNio2Protocol ，非阻塞式 JAVA NIO2 链接器
org.apache.coyote.ajp.AjpAprProtocol ， APR 链接器
  ```
 - connectionTimeOut : Connector 接收链接后的等待超时时间， 单位为 毫秒。 -1 表示不超时
 - redirectPort：当前Connector 不支持SSL请求， 接收到了一个请求， 并且也符合security-constraint 约束， 需要SSL传输，Catalina自动将请求重定向到指定的端口
 - executor ： 指定共享线程池的名称， 也可以通过maxThreads、minSpareThreads等属性配置内部线程池
 - URIEncoding : 用于指定编码URI的字符编码， Tomcat8.x版本默认的编码为 UTF-8 ,Tomcat7.x版本默认
      为ISO-8859-1
- Engine
 > Engine 作为Servlet 引擎的顶级元素，内部可以嵌入： Cluster、Listener、Realm、Valve和Host
 ```xml
<Engine name="Catalina" defaultHost="localhost">
    ....
</Engine>
 ```
 - name： 用于指定Engine 的名称， 默认为Catalina 。该名称会影响一部分Tomcat的存储路径（如临时文件）
 - defaultHost ： 默认使用的虚拟主机名称， 当客户端请求指向的主机无效时， 将交由默认的虚拟主机处理， 默认为localhost
- Host
 > Host 元素用于配置一个虚拟主机， 它支持以下嵌入元素：Alias、Cluster、Listener、Valve、Realm、Context
 >
 > 如果在Engine下配置Realm， 那么此配置将在当前Engine下的所有Host中共享
 >
 > 在Host中配置Realm ， 则在当前Host下的所有Context中共享
 >
 > Context中的Realm优先级 > Host 的Realm优先级 > Engine中的Realm优先级
 >
 > 通过给Host添加别名，我们可以实现同一个Host拥有多个网络名称
 ```xml
<Host name="localhost" appBase="webapps" unpackWARS="true" autoDeploy="true">
    ....
    <Alias>www.shadow.com</Alias>
</Host>
 ```
 - name: 当前Host通用的网络名称， 必须与DNS服务器上的注册信息一致。 Engine中包含的Host必须存在
    一个名称与Engine的defaultHost设置一致
 - appBase： 当前Host的应用基础目录， 当前Host上部署的Web应用均在该目录下（可以是绝对目录，相对路径）。默认为webapps
 - unpackWARs： 设置为true， Host在启动时会将appBase目录下war包解压为目录。设置为false， Host将直接从war文件启动
 - autoDeploy： 控制tomcat是否在运行时定期检测并自动部署新增或变更的web应用
- Context
 > Context 用于配置一个Web应用
 >
 > 它支持的内嵌元素为：CookieProcessor， Loader， Manager，Realm，Resources，WatchedResource，JarScanner，Valve
 ```xml
<Context docBase="myApp" path="/myApp">
    ....
</Context>
 ```
 - docBase：Web应用目录或者War包的部署路径。可以是绝对路径，也可以是相对于Host appBase的相对路径
 - path：Web应用的Context 路径。如果我们Host名为localhost， 则该web应用访问的根路径为： 
    http://localhost:8080/myApp
 ```xml
<Host name="www.tomcat.com" appBase="webapps" unpackWARs="true" autoDeploy="true">
    <Context docBase="D:bbsApp" path="/myApp"></Context>
    <Valve className="org.apache.catalina.valves.AccessLogValve"
           directory="logs"
           prefix="localhost_access_log" suffix=".txt"
           pattern="%h %l %u %t "%r" %s %b" />
</Host>
 ```
##### 2. tomcat-users.xml
 `该配置文件中，主要配置的是Tomcat的用户，角色等信息，用来控制Tomcat中manager， host-manager的访问权限`
##### 3. web.xml
 `web.xml 是web应用的描述文件， 它支持的元素及属性来自于Servlet 规范定义 。 在Tomcat 中， Web 应用的描述信息包括 tomcat/conf/web.xml 中默认配置 以及 Web应用 WEB-INF/web.xml 下的定制配置`
#### 6. tomcat 类加载器
##### 1. 需要解决的问题
- 一个web容器可能需要部署两个应用程序，不同的程序可能会依赖同一个第三方类库的不同版本，不能要求同一个类库在同一个服务器只有一份，因此要保证每个应用程序的类库都是独立的，保证相互隔离
- 部署在同一个web容器中相同的类库相同的版本可以共享，否则如果部署多个应用程序，那么就需要多次加载相同的类到虚拟机
- web容器，也即tomcat自身有依赖的类库，不能与应用程序的类库混淆，基于安全考虑，需要让容器的类库和应用程序的类库隔离开来
- web容器要支持 jsp 的修改而不用重启web容器
##### 2. 默认的类加载机制能否解决问题
- 问题一不能解决，类的完全限定名唯一且只有一份
- 问题二可以解决
- 问题三与问题一一样
- 问题四在jsp被编译为class文件后，被加载到虚拟机中，修改jsp内容，但是类名还是一样，在方法区已经存在，不会进行重载，因此需要对加载到虚拟机中的class进行卸载，那么就需要卸载加载此class的类加载器，然后重新创建类加载器，重新加载jsp文件
##### 3. tomcat 解决方案
- tomcat 类加载器体系
 ![](./classloader.png)
- tomcat 类加载器体系说明
   - BootStrap ClassLoader、Extension ClassLoader、Application ClassLoader 为虚拟机默认类加载器
   - Common ClassLoader、Catalina ClassLoader、Shared ClassLoader、WebappClassLoader 是 tomcat 定义的类加载器，在 tomcat5及以前，分别加载 /common/*、/server/*、/shared/*，在 tomcat6及以后，合并到根目录下 lib 和 /Webapp/WEB-INF/*中的java类库
   - 其中 WebApp 类加载器和 Jsp 类加载器通常存在多个，每一个 Web 应用程序对应一个 WebApp 类加载器，每一个 jsp 文件对应一个 Jsp 类加载器
 - 类加载器的权限范围说明
     - Common ClassLoader ： tomcat 最基本的类加载器，加载的class可被tomcat容器本身及各个WebApp访问
     - Catalina ClassLoader ：tomcat 容器私有的类加载器，加载的class对 WebApp 不可见
     - Shared ClassLoader ：各个 WebApp 共享的类加载器，对所有的 WebApp 可见，但对 tomcat 容器不可见
     - WebApp ClassLoader ：各个 WebApp 私有的类加载器，加载项目路径下的class，只对当前 WebApp 可见
     - Jasper Loader ：每个 jsp 文件对应的类加载器，每次修改 jsp 文件会被销毁然后进行重新创建
 > 问题：Common ClassLoader 想加载 WebApp ClassLoader 中的类怎么办？
 >
 > 答：使用线程上下文类加载器，可以让父类加载器请求子类加载器完成类加载，JNDI+SPI 
##### 4. 源码分析
- Common ClassLoader、Catalina ClassLoader、Shared ClassLoader
 ```java
private void initClassLoaders() {
    try {
        // ① 创建Common ClassLoader，父类加载器传 null，后续调用创建URLClassLoader
        //  指定其父类加载器为 系统类加载器 Application ClassLoader
        commonLoader = createClassLoader("common", null);
        ....
            // ② 创建 Catalina ClassLoader，父类加载器指定为 Common ClassLoader   
            catalinaLoader = createClassLoader("server", commonLoader);
        // ③ 创建 Shared ClassLoader，父类加载器指定为 Common ClassLoader
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {
        ....
    }
}
 ```
- WebApp ClassLoader
 StandardContex#startInternal
 ```java
....
    // ④ 创建 WebappLoader
    if (getLoader() == null) {
        WebappLoader webappLoader = new WebappLoader();
        webappLoader.setDelegate(getDelegate());
        setLoader(webappLoader);
    }
....
 ```
- JasperLoader
 JspServletWrapper#service
 ```java
....
    // JSP文件编译为class后，重新加载 class文件
    servlet = getServlet();
....
 ```
- JspServletWrapper#getServlet

 ```java
public Servlet getServlet() throws ServletException {
    ....
        // ctxt.getJspLoader() 重新创建类加载器
        servlet = (Servlet) instanceManager.newInstance(ctxt.getFQCN()
                                                        , ctxt.getJspLoader());
    ....
        return theServlet;
}
 ```
- JspCompilationContext#getJspLoader

 ```java
public ClassLoader getJspLoader() {
    if( jspLoader == null ) {
        // ⑤ 创建 JasperLoader
        jspLoader = new JasperLoader
            (new URL[] {baseUrl},
             getClassLoader(),
             rctxt.getPermissionCollection());
    }
    return jspLoader;
}
 ```

- DefaultInstanceManager#newInstance

 ```java
public Object newInstance(final String className, final ClassLoader classLoader)
    throws IllegalAccessException, NamingException, InvocationTargetException,
InstantiationException, ClassNotFoundException, IllegalArgumentException,
NoSuchMethodException, SecurityException {
    // 类加载器加载 class 文件到内存
    Class<?> clazz = classLoader.loadClass(className);
    // 创建Servlet对象
    return newInstance(clazz.getConstructor().newInstance(), clazz);
}
 ```



#### 7. tomcat 安全

> - 删除webapps目录下的所有文件，禁用tomcat管理界面
>
> - 注释或删除tomcat-users.xml文件内的所有用户权限
>
> - 更改关闭tomcat指令或禁用
>
>```xml
>  <!-- 1.更改端口号和指令 -->
>  <Server port="8456" shutdown="shadow_shut">
>  <!-- 2.禁用端口 -->
>  <Server port="‐1" shutdown="SHUTDOWN">  
>  ```
> - 定义错误页面，不会看到异常的堆栈信息，提高了用户体验，也保障了服务的安全性
#### 8. 应用安全
> - Apache Shiro
> - SpringSecurity
> - 自定义权限管理框架

#### 9. 传输安全
> HTTPS的全称是超文本传输安全协议（Hypertext Transfer Protocol Secure），是一种网络安全传输协议。在HTTP的基础上加入SSL/TLS来进行数据加密，保护交换数据不被泄露、窃取
>
> HTTPS和HTTP的区别主要为以下四点：
>
> - HTTPS协议需要到证书颁发机构CA申请SSL证书, 然后与域名进行绑定，HTTP不用申请证书
> - HTTP是超文本传输协议，属于应用层信息传输，HTTPS 则是具有SSL加密传安全性传输协议，对数据的传输进行加密，相当于HTTP的升级版
> - HTTP和HTTPS使用的是完全不同的连接方式，用的端口也不一样，前者是8080，后者是8443
> - HTTP的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比HTTP协议安全