### 1、@EnableFeignClients 注解开启Feign的功能
SpringCloud通过添加 @EnableFeignClients 注解开启Feign的功能，那么我们查看一下这个注解功能做了什么？  
@Import(FeignClientsRegistrar.class)  
引入FeignClientsRegistrar类
```java
@Retention(RetentionPolicy.RUNTIME)  
@Target(ElementType.TYPE)  
@Documented  
@Import(FeignClientsRegistrar.class)  
public @interface EnableFeignClients {

}
```
## 2、分析FeignClientsRegistrar类
首先看这个类实现了一个重要接口ImportBeanDefinitionRegistrar，这个接口是Spring框架对外提供扩展的一个很重的一个知识点。
很多基于Spring框架做扩展都会用到这个接口，例如Spring整合Mybatis就用到了这个接口。
通过ImportBeanDefinitionRegistrar接口的registerBeanDefinitions方法就可以往Spring容器中根据自己的业务来注册bean了。
```java
@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
		/// 该方法里，分为两大步骤
		/// 1、注册Feign用到的默认配置类
		/// 这个方法实现比较简单，主要就是处理
		 EnableFeignClients注解里面
		如果配置了defaultConfiguration属性，这里会把配置的类，注入到Spring容器
		/// 例如：@EnableFeignClients(defaultConfiguration = {XXXXX.class})
		/// 注入到 Spring容器中会是一个 FeignClientSpecification 类
		registerDefaultConfiguration(metadata, registry);
		/// 2、注册Feign客户端到Spring容器(下一步就重点分析这个方法)
		registerFeignClients(metadata, registry);
	}
```
## 3、分析registerFeignClients()方法
方法中的大致做了一下几件事：  
1）通过配置的basePackages或者basePackageClasses扫描包范围，去扫描带有@FeignClient配置的文件，通过BeanDefinition形式返回来  
2）循环遍历出来BeanDefinition集合
```java
///  这里遍历 扫描出来的 BeanDefinition
for (BeanDefinition candidateComponent : candidateComponents) {
	if (candidateComponent instanceof AnnotatedBeanDefinition) {
		// verify annotated class is an interface
		AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
		AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
		Assert.isTrue(annotationMetadata.isInterface(),
				"@FeignClient can only be specified on an interface");

		Map<String, Object> attributes = annotationMetadata
				.getAnnotationAttributes(
						FeignClient.class.getCanonicalName());

		String name = getClientName(attributes);
		  下面两个方法，注册到Spring 容器
		
		/// 这个方法是注册FeignClinton的配置类，在Spring容器中是以FeignClientSpecification形式的Bean呈现的
		/// 这里和上一步的registerDefaultConfiguration方法的作用相同
registerClientConfiguration(registry,name,attributes.get("configuration"));

///  这个方法就是真正注册 feign 客户端 到Spring 容器 （下一步就中的分析这里）
registerFeignClient(registry, annotationMetadata, attributes);
	}
			}

```
## 4、分析registerFeignClient()方法
这个方法会把扫描出来的BeanDefinition会变成 FeignClientFactoryBean 类注入到Spring容器。
方法很简单，就是根据配置设置一些属性，然后注册到容器中。
重点是FeignClientFactoryBean类，这类是一个FactoryBean类，下一步就是重点分析这个类。
```java
private void registerFeignClient(BeanDefinitionRegistry registry,AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		String contextId = getContextId(attributes);
		definition.addPropertyValue("contextId", contextId);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

		String alias = contextId + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
		beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);

		// has a default, won't be null
		boolean primary = (Boolean) attributes.get("primary");

		beanDefinition.setPrimary(primary);

		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}

		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}

```
## 5、分析FeignClientFactoryBean类
为什么要扫描出来的接口，都要变成FeignClientFactoryBean形式的类，放入到Spring容器？？

在学习Spring框架的时候，就有接触过一种特殊bean叫 FactoryBean。 一直都不太理解这种bean有什么特殊的使用场景。现在我们来看看SpringCloud整合OpenFeign的时候是如何使用FactoryBean的。

首先，说一下FactoryBean！ 这是一个接口，接口声明的方法也很简单，主要就一个getObject()。
FactoryBean：是一类特殊的 Bean，他特殊在什么地方呢？就是他是一个能生产对象的工厂Bean，它的实现和工厂模式及修饰器模式很像。

我们将扫描出来的接口，统一使用一个FactoryBean类来进行包装后，放入Spring容器。这样我们通过@Autowired从Spring容器获取Bean的时候，就可以统一进行一些处理这些bean也就是对Bean做了代理，返回来的都是使用OpenFeign代理过的类。这个FactoryBean对应到源码里就是FeignClientFactoryBean。

下面我们来看FeignClientFactoryBean这个Bean中都做了什么，入口方法就是FactoryBean接口的T getObject();方法.源码里getObject()方法调用了getTarget();

1、第一步是获取FeignContext也就是Feign的上下文。
那么这个FeignContext 是什么时候放入到Spring容器中的呢？
这个就需要我们知道 SpringBoot的SPI机制，在Spring中有一种类似与Java SPI的加载机制。Spring会扫描META-INF/spring.factories文件中配置接口的实现类名称，然后在程序中读取这些配置文件并实例化。这种自定义的SPI机制也就是Spring Boot Starter实现的基础。
所以在spring-cloud-openfeign-core.jar的META-INFO目录下查看会发现有一个spring.factories文件，里面配置需要Spring注入到容器中的类。其中FeignAutoConfiguration类里就注入了FeignContext类。


