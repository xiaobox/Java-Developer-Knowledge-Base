# arthas 高级玩法

## 获取 spring context  然后为所欲为？

执行 `tt` 命令来记录 `RequestMappingHandlerAdapter#invokeHandlerMethod` 的请求

```bash
tt -t org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter invokeHandlerMethod

```

请求以后将看到

```bash
[arthas@38341]$ tt -t org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter invokeHandlerMethod
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 60 ms, listenerId: 2
 INDEX      TIMESTAMP                  COST(ms)      IS-RET     IS-EXP     OBJECT              CLASS                                    METHOD
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 1000       2021-08-03 15:34:31        473.709743    true       false      0x545d772e          RequestMappingHandlerAdapter             invokeHandlerMethod

```

可以用 `tt` 命令的 -i 参数来指定 `index`，并且用 -w 参数来执行 `ognl` 表达式来获取 `spring context` ：

```bash
[arthas@38341]$ tt -i 1000 -w 'target.getApplicationContext()'
@AnnotationConfigServletWebServerApplicationContext[
    reader=@AnnotatedBeanDefinitionReader[org.springframework.context.annotation.AnnotatedBeanDefinitionReader@28aa9b60],
    scanner=@ClassPathBeanDefinitionScanner[org.springframework.context.annotation.ClassPathBeanDefinitionScanner@1c138ae],
    annotatedClasses=@LinkedHashSet[isEmpty=true;size=0],
    basePackages=null,
    logger=@SLF4JLocationAwareLog[org.apache.commons.logging.impl.SLF4JLocationAwareLog@450746b3],
    DISPATCHER_SERVLET_NAME=@String[dispatcherServlet],
    webServer=@TomcatWebServer[org.springframework.boot.web.embedded.tomcat.TomcatWebServer@c5697fe],
    servletConfig=null,
    serverNamespace=null,
    servletContext=@ApplicationContextFacade[org.apache.catalina.core.ApplicationContextFacade@705b6c8f],
    themeSource=@ResourceBundleThemeSource[org.springframework.ui.context.support.ResourceBundleThemeSource@78d73e62],
    beanFactory=@DefaultListableBeanFactory[org.springframework.beans.factory.support.DefaultListableBeanFactory@4277127c: defining beans [org.springframework.context.annotatio
n.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnno
tationProcessor,org.springframework.context.event.internalEventListenerProcessor,org.springframework.context.event.internalEventListenerFactory,mealBootstrapApplication,bootstr
apApplicationListener.BootstrapMarkerConfiguration,org.springframework.boot.autoconfigure.internalCachingMetadataReaderFactory,glRocketMQConfig,globalConfig,mybatisPlusConfig,o
kHttpConfiguration,mealQueryBaseDataController,mealAppBannerController,appOrderController,mealBannerController,mealReturnCouponsTipsController,appletsOrderController,h5OrderCon
troller,thirdNoticeController,aliPayNoticeController,payController,wxPayNoticeController,mealGoodsExclusivePricePcController,mealManagerController,mealOrderManagerController,ba
seServiceClient,fileServiceClient,userCenterClient,mealGlobalConfigServiceImpl,mealGoodsCostServiceImpl,mealGoodsCostTableServiceImpl,mealGoodsExclusivePriceServiceImpl,mealBan
nerServiceImpl,mealManagerPcServiceImpl,mealOrderManagerPcServiceImpl,mealQueryBaseDataServiceImpl,mealReturnCouponsTipsServiceImpl,internalOrderMq,glCompensationFuncTM,glRocke
tMQLocalTM,orderPayResultServiceImpl,orderServiceImpl,orderJobServiceImpl,aliPayConfig,wxAppPayProperties,wxPayConfiguration,aliAppletCyPayServiceImpl,aliPayServiceImpl,mealPay
FlowServiceImpl,payProfitSharingFlowServiceImpl,payResultServiceImpl,weChatPayServiceImpl,payFlowHandler,payJobServiceImpl,refundServiceImpl,couponsUtil,idUtils,thirdConfig,goo
dsSyncServiceImpl,okHttpUtil,reqUtil,syncGoodsTaskHandler,costTableHandler,mealOrderHandler,payHandler,statisticTaskHandler,globalExceptionHandler,springWebMvcConfiguration,asy
ncUtils,feignHystrixConcurrencyStrategy,springSecurityConfiguration,authenticationCustomizeEntryPoint,authAccessDeniedHandler,RBACService,memberExtraInfoClientFallback,memberIn
foClientFallback,memberRiskClientFallback,memberWalletClientFallback,memberWalletFlowClientFallback,riskEventClientFallback,userCenterMemberInfoClientFallback,userCenterOperato
rClientFallback,constants,utils,captchaClientFallback,channelIsolationClientFallback,channelPriorityClientFallback,complexMessageClientFallback,JPushClientFallback,messageFeish
uBotClientFallback,messageFeiShuBotClientV2Fallback,stationLetterClientFallback,idGenerateClientFallback,couponClientFallBack,couponManagerClientFallBack,couponMemberClientFall
Back,normalFileClientFallback,transactionAutoConfiguration,webAutoConfiguration,authAutoConfiguration,cacheAutoConfiguration,transactionMQProducer,defaultMQPushConsumer,enumCus
tomizer,paginationInterceptor,okHttpClient,x509TrustManager,sslSocketFactory,pool,aliAppPayProperties,aliAppletPayProperties,appAlipayClient,appletAlipayClient,wxAppletPayServi
ce,wxAppPayService,org.springframework.boot.context.properties.ConfigurationPropertiesBindingPostProcessor,org.springframework.boot.context.internalConfigurationPropertiesBinde
rFactory,org.springframework.boot.context.internalConfigurationPropertiesBinder,org.springframework.boot.context.properties.BoundConfigurationProperties,org.springframework.boo
t.context.properties.ConfigurationPropertiesBeanDefinitionValidator,org.springframework.boot.context.properties.ConfigurationBeanFactoryMetadata,wx.applet.pay-com.demo.meal.se
rvice.pay.config.WxAppletPayProperties,errorPageRegistrar,corsFilterRegistrationBean,org.springframework.security.config.annotation.configuration.ObjectPostProcessorConfigurati
on,objectPostProcessor,org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration,authenticationManagerBuilder,enableGlobalAuthenti
cationAutowiredConfigurer,initializeUserDetailsBeanManagerConfigurer,initializeAuthenticationProviderBeanManagerConfigurer,org.springframework.security.config.annotation.web.co
nfiguration.WebSecurityConfiguration,delegatingApplicationListener,webSecurityExpressionHandler,springSecurityFilterChain,privilegeEvaluator,conversionServicePostProcessor,auto
wiredWebSecurityConfigurersIgnoreParents,org.springframework.security.config.annotation.web.configuration.WebMvcSecurityConfiguration,requestDataValueProcessor,org.springframew
ork.scheduling.annotation.SchedulingConfiguration,org.springframework.context.annotation.internalScheduledAnnotationProcessor,org.springframework.boot.autoconfigure.AutoConfigu
rationPackages,com.demo.MealBootstrapApplication#MapperScannerRegistrar#0,default.com.demo.MealBootstrapApplication.FeignClientSpecification,memberExtraInfoClient.FeignClient
Specification,com.demo.user.common.client.MemberExtraInfoClient,MemberInfoClient.FeignClientSpecification,com.demo.user.common.client.MemberInfoClient,MemberRiskClient.FeignC
lientSpecification,com.demo.user.common.client.MemberRiskClient,MemberWalletClient.FeignClientSpecification,com.demo.user.common.client.MemberWalletClient,memberWalletFlowCli
ent.FeignClientSpecification,com.demo.user.common.client.MemberWalletFlowClient,riskEventClient.FeignClientSpecification,com.demo.user.common.client.RiskEventClient,userCenterMemberInfoClient.FeignClientSpecification,com.demo.user.common.client.UserCenterMemberInfoClient,userCenterOperatorClient.FeignClientSpecification,com.demo.user.common.client.UserCenterOperatorClient,captchaClient.FeignClientSpecification,com.demo.message.rest.CaptchaClient,channelIsolationClient.FeignClientSpecification,com.demo.message.rest.ChannelIsolationClient,channelPriorityClient.FeignClientSpecification,com.demo.message.rest.ChannelPriorityClient,complexMessageClient.FeignClientSpecification,com.demo.message.rest.ComplexMessageClient,jPushClient.FeignClientSpecification,com.demo.message.rest.JPushClient,messageFeishuBotClient.FeignClientSpecification,com.demo.message.rest.MessageFeishuBotClient,messageFeiShuBotClientV2.FeignClientSpecification,com.demo.message.rest.MessageFeiShuBotClientV2,stationLetterClient.FeignClientSpecification,com.demo.message.rest.StationLetterClient,idGenerateClient.FeignClientSpecification,com.demo.id.client.IdGenerateClient,couponClient.FeignClientSpecification,com.demo.marketing.client.coupon.CouponClient,couponManagerClient.FeignClientSpecification,com.demo.marketing.client.coupon.CouponManagerClient,couponMemberClient.FeignClientSpecification,com.demo.marketing.client.coupon.CouponMemberClient,normalFileClient.FeignClientSpecification,c],
    resourceLoader=null,
    customClassLoader=@Boolean[false],
    refreshed=@AtomicBoolean[true],
    MESSAGE_SOURCE_BEAN_NAME=@String[messageSource],
    LIFECYCLE_PROCESSOR_BEAN_NAME=@String[lifecycleProcessor],
    APPLICATION_EVENT_MULTICASTER_BEAN_NAME=@String[applicationEventMulticaster],
    logger=@SLF4JLocationAwareLog[org.apache.commons.logging.impl.SLF4JLocationAwareLog@71add15],
    id=@String[application-1],
    displayName=@String[org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext@58740366],
    parent=@AnnotationConfigApplicationContext[org.springframework.context.annotation.AnnotationConfigApplicationContext@62d363ab, started on Tue Aug 03 15:13:22 CST 2021],
    environment=@StandardServletEnvironment[StandardServletEnvironment {activeProfiles=[local, common-local, common], defaultProfiles=[default], propertySources=[MapPropertySource {name='server.ports'}, ConfigurationPropertySourcesPropertySource {name='configurationProperties'}, EncryptablePropertySourceWrapper {name='servletConfigInitParams'}, EncryptablePropertySourceWrapper {name='servletContextInitParams'}, EncryptableMapPropertySourceWrapper {name='systemProperties'}, EncryptableSystemEnvironmentPropertySourceWrapper{name='systemEnvironment'}, EncryptablePropertySourceWrapper {name='random'}, EncryptableMapPropertySourceWrapper {name='springCloudClientHostInfo'}, EncryptableMapPropertySourceWrapper {name='applicationConfig: [classpath:/application-common.yml]'}, EncryptableMapPropertySourceWrapper {name='applicationConfig: [classpath:/application-common-local.yml]'}, EncryptableMapPropertySourceWrapper {name='applicationConfig: [classpath:/application-local.yml]'}, EncryptableMapPropertySourceWrapper {name='applicationConfig: [classpath:/application.yml]'}, EncryptableMapPropertySourceWrapper {name='springCloudDefaultProperties'}, EncryptablePropertySourceWrapper {name='cachedrandom'},  {name='Management Server'}]}],
    beanFactoryPostProcessors=@ArrayList[isEmpty=false;size=5],
    startupDate=@Long[1627974802713],
    active=@AtomicBoolean[true],
    closed=@AtomicBoolean[false],
    startupShutdownMonitor=@Object[java.lang.Object@553a67da],
    shutdownHook=@[Thread[SpringContextShutdownHook,5,main]],
    resourcePatternResolver=@ServletContextResourcePatternResolver[org.springframework.web.context.support.ServletContextResourcePatternResolver@499ea1d],
    lifecycleProcessor=@DefaultLifecycleProcessor[org.springframework.context.support.DefaultLifecycleProcessor@61ac01da],
    messageSource=@DelegatingMessageSource[Empty MessageSource],
    applicationEventMulticaster=@SimpleApplicationEventMulticaster[org.springframework.context.event.SimpleApplicationEventMulticaster@ee42df5],
    applicationListeners=@LinkedHashSet[isEmpty=false;size=38],
    earlyApplicationListeners=@LinkedHashSet[isEmpty=false;size=19],
    earlyApplicationEvents=null,
    classLoader=@AppClassLoader[jdk.internal.loader.ClassLoaders$AppClassLoader@9e89d68],
    protocolResolvers=@LinkedHashSet[isEmpty=true;size=0],
    resourceCaches=@ConcurrentHashMap[isEmpty=true;size=0],
]
Affect(row-cnt:1) cost in 4 ms.
```

利用 `tt` 命令通过 1000 这个 `index` 我们已经可以得到 `spring context` 了，得到后就可以利用 `spring context` 进行各种操作了。比如我们 get 一个 `controller` 的 `bean` 并执行一下方法

```bash
tt -i 1000 -w 'target.getApplicationContext().getBean("mealManagerController").listMealByCategoryId(1379999659854008320L)'
```

## 查看SQL语句

**watch Connection**

```bash
watch java.sql.Connection prepareStatement '{params,throwExp}'  -n 5  -x 3 
```

**watch BoundSql (mybatis)**

```bash
watch org.apache.ibatis.mapping.BoundSql getSql '{params,returnObj,throwExp}'  -n 5  -x 3 
```

## 热修复三板斧（生产慎用！）

### 反编译输出 .java 源码

```bash
jad --source-only com.demo.meal.pc.MealManagerController > /tmp/MealManagerController.java
```

### 修改代码

利用 vim 直接修改上面反编译出的文件内容

### 编译

搜索到对应类的classloader

```bash
[arthas@38341]$ sc -d *MealManagerController |grep classLoader
 classLoaderHash   9e89d68
```

mc(Memory Compiler) 命令来编译

```bash
mc -c  9e89d68 /tmp/MealManagerController.java -d /tmp
```

### 热加载

重新热加载(无需重启服务), 并且非侵入, 只是临时修改

```bash
[arthas@38341]$ redefine /tmp/com/gaolv/meal/pc/MealManagerController.class
redefine success, size: 1, classes:
com.gaolv.meal.pc.MealManagerController
```

### 限制

*   redefine的class不能修改、添加、删除类的field和method，包括方法参数、方法名称及返回值

*   redefine命令和jad/watch/trace/monitor/tt等命令会冲突。执行完redefine之后，如果再执行上面提到的命令，则会把redefine的字节码重置。&#x20;

## 远程打断点 Debug?&#x20;

虽然可以，但不建议

[https://github.com/alibaba/arthas/issues/222](https://github.com/alibaba/arthas/issues/222 "https://github.com/alibaba/arthas/issues/222")

## 参考

*   [https://www.yuque.com/arthas-idea-plugin/help/gega6l](https://www.yuque.com/arthas-idea-plugin/help/gega6l "https://www.yuque.com/arthas-idea-plugin/help/gega6l")

*   [http://hengyunabc.github.io/arthas-online-hotswap/](http://hengyunabc.github.io/arthas-online-hotswap/ "http://hengyunabc.github.io/arthas-online-hotswap/")
