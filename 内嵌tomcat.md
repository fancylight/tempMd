# 概述
本文用来描述springboot使用内嵌tomcat的方式
## springBoot流程
### 工厂模式
- ServletWebServerFactory
用来创建相应的web容器,如tomcat,jetty
- ServletWebServerFactoryConfiguration
与之相对的starter,用来动态创建不同的web工厂
### 工厂对象配置
- ServerProperties
starer的属性对象,对应server属性
- ServletWebServerFactoryCustomizer
使用ServerProperties设置工厂
- TomcatServletWebServerFactoryCustomizer
使用ServerProperties,设置TomcatServletWebServerFactory
- WebServerFactoryCustomizerBeanPostProcessor
后处理器,当web工厂被ioc创建时调用,用来使用上述两个类来设置工厂属性
- ServletWebServerFactoryAutoConfiguration
创建上述三个类到ioc容器中去
### TomcatServletWebServerFactory
该类会被ServletWebServerApplicationContext,即springBoot中创建的context调用
```java
private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			ServletWebServerFactory factory = getWebServerFactory();
			this.webServer = factory.getWebServer(getSelfInitializer()); //构建WebServer对象,内部封装Tomcat或Jeety
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
		initPropertySources();
	}
	
	//-----------------ServletContextInitializer对象是springBoot用来回调的类
		private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
		return this::selfInitialize;
	}

	private void selfInitialize(ServletContext servletContext) throws ServletException {
		prepareWebApplicationContext(servletContext);
		registerApplicationScope(servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
		for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
			beans.onStartup(servletContext);
		}
	}
	protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
		return new ServletContextInitializerBeans(getBeanFactory());
	}
```

ServletContextInitializer和WebApplicationInitializer区别:
后者在servlet3.0中和@HandType连用来配置Context,默认的实现用来配置context,详情见ServletContainerInitializer使用
前者的作用是 通过获取ioc中实现,并通过ServletContextInitializerBeans类向ServletContext中配置servlet,filter,listener
- WebServer 
springBoot用来封装web容器的类
```java
public interface WebServer {

	/**
	 * Starts the web server. Calling this method on an already started server has no
	 * effect.
	 * @throws WebServerException if the server cannot be started
	 */
	void start() throws WebServerException;

	/**
	 * Stops the web server. Calling this method on an already stopped server has no
	 * effect.
	 * @throws WebServerException if the server cannot be stopped
	 */
	void stop() throws WebServerException;

	/**
	 * Return the port this server is listening on.
	 * @return the port (or -1 if none)
	 */
	int getPort();

}
```
### 内嵌tomcat
- Tomcat类
该类封装了server,service,host,engine,context
- tomcat中StanderContext和ServletContext
前者是tomcat容器组件,后者是servlet标准,实际上在tomcat的实现中后者持有前者
- context内部有WebResourceRoot对应,改类也是LifeCycle组件
当context发生start时期,该类也会进行start,并加载内部资源
```java
 protected void startInternal() throws LifecycleException {
        mainResources.clear();

        main = createMainResourceSet();

        mainResources.add(main); //加载docBase资源

        for (List<WebResourceSet> list : allResources) {
            // Skip class resources since they are started below
            if (list != classResources) {
                for (WebResourceSet webResourceSet : list) {
                    webResourceSet.start();
                }
            }
        }

        // This has to be called after the other resources have been started
        // else it won't find all the matching resources
        processWebInfLib();
        // Need to start the newly found resources
        for (WebResourceSet classResource : classResources) {
            classResource.start();
        }

        cache.enforceObjectMaxSizeLimit();

        setState(LifecycleState.STARTING);
    }

    protected WebResourceSet createMainResourceSet() {
        String docBase = context.getDocBase(); //context的docBase属性

        WebResourceSet mainResourceSet;
        if (docBase == null) {
            mainResourceSet = new EmptyResourceSet(this);
        } else {
            File f = new File(docBase);
            if (!f.isAbsolute()) {
                f = new File(((Host)context.getParent()).getAppBaseFile(), f.getPath());
            }
            if (f.isDirectory()) {
                mainResourceSet = new DirResourceSet(this, "/", f.getAbsolutePath(), "/");
            } else if(f.isFile() && docBase.endsWith(".war")) {
                mainResourceSet = new WarResourceSet(this, "/", f.getAbsolutePath());
            } else {
                throw new IllegalArgumentException(
                        sm.getString("standardRoot.startInvalidMain",
                                f.getAbsolutePath()));
            }
        }

        return mainResourceSet;
    }
```
- 关于context的docBase属性
在外置的tomcat是用户通过xml设置的,在springBoot环境则是创建tomcat对象过程设置的,
即在TomcatServletWebServerFacotry中
```java

```