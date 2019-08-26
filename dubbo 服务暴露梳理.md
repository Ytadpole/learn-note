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
													
												
								


