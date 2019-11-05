### dubbo 服务暴露梳理

主要流程
ServiceBean//开始
	ServiceConfig.export
		ServiceConfig.doExport()
			ServiceConfig.doExportUrls()
				ServiceConfig.doExportUrlsFor1Protocol()
					ServiceConfig.exportLocal()//本地暴露
						ProxyFactory$Adaptive.getInvoker()//获取invoker
							ProtocolFilterWrapper.export()
								ListenerExporterWrapper.export()
									InjvmProtocol.export()//真正开始本地暴露的地方
										new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);//主要将InjvmExporter放到InjvmProtocol类的内存map中
					//远程暴露
					ProxyFactory$Adaptive.export()
							ProtocolFilterWrapper.export()
								ListenerExporterWrapper.export()
									RegistryProtocol.export()//注册中心协议暴露
										RegistryProtocol.doLocalExport()
											DubboProtocol.export()//开始dubbo协议
												//省略wrapper
												实例化exporer，并存放map缓存
												DubboProtocol.openServer（）
													DubboProtocol.createServer（）
														Exchangers.bind()
															HeaderExchanger.bind()
																Transporters.bind()
																	Transporter$Adaptive.bind()
																		NettyTransporter.bind()//实例化nettyServer
													
												
								

### filter
主要构造 链代码

```Java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

    if (!filters.isEmpty()) {
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {

                @Override
                public Class<T> getInterface() { return invoker.getInterface();}

                @Override
                public URL getUrl() { return invoker.getUrl(); }

                @Override
                public boolean isAvailable() { return invoker.isAvailable(); }

                @Override
                public Result invoke(Invocation invocation) throws RpcException {
                    Result asyncResult;
                    //····省略
                    return asyncResult;
                }

                @Override
                public void destroy() { invoker.destroy(); }

                @Override
                public String toString() { return invoker.toString(); }
            };
        }
    }
    return new CallbackRegistrationInvoker<>(last, filters);
}
```



### listener

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    if (REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        return protocol.export(invoker);
    }
  
  //执行完暴露服务操作后，实例化ListenerExporterWrapper，构造函数中执行listener
    return new ListenerExporterWrapper<T>(protocol.export(invoker),
            Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                    .getActivateExtension(invoker.getUrl(), EXPORTER_LISTENER_KEY)));
}
```

构造函数

```java
public ListenerExporterWrapper(Exporter<T> exporter, List<ExporterListener> listeners) {
    if (exporter == null) {
        throw new IllegalArgumentException("exporter == null");
    }
    this.exporter = exporter;
    this.listeners = listeners;
  
  	//执行
    if (CollectionUtils.isNotEmpty(listeners)) {
        RuntimeException exception = null;
        for (ExporterListener listener : listeners) {
            if (listener != null) {
                try {
                		//循环执行listener
                    listener.exported(this);
                } catch (RuntimeException t) {
                    logger.error(t.getMessage(), t);
                    exception = t;
                }
            }
        }
        if (exception != null) {
            throw exception;
        }
    }
}
```