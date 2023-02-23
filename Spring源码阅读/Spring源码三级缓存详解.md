### 三级缓存
1. 一级缓存：*singletonObjects*
2. 二级缓存：*earlySingletonObjects*
3. 三级缓存：*singletonFactories*
源码：
```java
	// 从上至下 分表代表这“三级缓存”
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256); //一级缓存
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16); // 二级缓存
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16); // 三级缓存
```
### 三级缓存作用
1. 一级缓存-singletonObjects是用来存放就绪状态的Bean。保存在该缓存中的Bean所实现Aware子接口的方法已经回调完毕，自定义初始化方法已经执行完毕，也经过BeanPostProcessor实现类的postProcessorBeforeInitialization、postProcessorAfterInitialization方法处理；
2. 二级缓存-earlySingletonObjects是用来存放早期曝光的Bean，一般只有处于循环引用状态的Bean才会被保存在该缓存中。保存在该缓存中的Bean所实现Aware子接口的方法还未回调，自定义初始化方法未执行，也未经过BeanPostProcessor实现类的postProcessorBeforeInitialization、postProcessorAfterInitialization方法处理。如果启用了Spring AOP，并且处于切点表达式处理范围之内，那么会被增强，即创建其代理对象。
   这里额外提一点，普通Bean被增强(JDK动态代理或CGLIB)的时机是在AbstractAutoProxyCreator实现的BeanPostProcessor的postProcessorAfterInitialization方法中，而处于循环引用状态的Bean被增强的时机是在AbstractAutoProxyCreator实现的SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference方法中。*（提前AOP）*
3.三级缓存-singletonFactories是用来存放创建用于获取Bean的工厂类-ObjectFactory实例。在IoC容器中，所有刚被创建出来的Bean，默认都会保存到该缓存中。

### Bean的生命周期
假设我们存在三个Service，分别是AService、BService、CService，其中AService注入了BService、CService,BService和CService注入了AService
```java
@Component  
public class AService {  
  
   @Autowired  
   private BService bService;  
  
   public void test(){  
      System.out.println(orderService);  
   }
```

```java
@Component  
public class BService {  
  
   @Autowired  
   private AService aService;  
  }
```

```java
@Component  
public class CService {  
  
   @Autowired  
   private AService Aservice;  
  
```
当我们要注入AService同时调用test方法时：
0.creatingSet<'AService'>*创建一个Set存放创建中的Bean*
1.new AService() -> AService普通对象 -> 放入三级缓存 lambda<beanName,AService普通对象>
2.填充BService属性 -> 去一级缓存单例池中找BService ->没有
	*创建BServiceBean*	
	2.1、new BService对象 -> 普通对象
	2.2、填充AService属性 -> 去单例池找 ->*没有* -> creatingSet(如果存在AService) -> *循环依赖* ->二级缓存找AServce ->三级缓存找AService ->
		 lambda表达式 -> 是否需要AOP -> *需要AOP* -> 进行AOP创建AService -> AService代理对象 -> 存入二级缓存
									-> *不需要AOP* ->AService普通对象 -> 存入二级缓存
	2.3、填充其他属性
	2.4、
	2.5、存入单例池Map
3.填充CService属性 -> 去单例池找CService -> 没有
	*创建CServiceBean*
	3.1、new CService() -> 普通对象
	3.2、填充AService属性 -> 单例池 ->*没有* -> creatingSet(如果存在AService) -> *循环依赖* -> 二级缓存 -> 找到AService（普通对象或是代理对象）
	.....*省略*

4.需不需要进行AOP  -> 有没有提前进行AOP ->AService代理对象

5.单例池 Map<'AService',AService代理对象>

>*Question:第五步单例池中的AService代理对象是怎么得到的？*

第4步执行完成之后，会判断是否发生了循环依赖，如果发生了循环依赖，就直接取二级缓存中的代理对象
如果没有发生循环依赖，就会取第4步进行Aop之后的代理对象

