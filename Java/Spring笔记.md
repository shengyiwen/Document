#### 1. Aop的调用顺序

aop都有哪些注解

- @Aroud    环绕
- @Before   先前调用
- @After    后置调用
- @AfterReturning   后置返回
- @AfterThrowing    后置异常

正常流程

| spring4         | spring5         |
|-----------------|-----------------|
| @AroundBegin    | @AroundBegin    |
| @Before         | @Before         |
| @AroundEnd      | @AfterReturning |
| @After          | @After          |
| @AfterReturning | @AroundEnd      |

异常流程

| spring4        | spring5        |
|----------------|----------------|
| @AroundBegin   | @AroundBegin   |
| @Before        | @Before        |
| @After         | @AfterThrowing |
| @AfterThrowing | @After         |

#### 2. Spring的三级缓存

spring的三级缓存主要是在DefaultSingletonBeanRegistry类中

- singletonObjects

一级缓存，保存已经完整的spring bean对象的容器

    private final Map<String, Object> singletonObjects = new ConcurrentHashMap();

- singletonFactories

三级缓存，保存类工厂的容器

    Map<String, ObjectFactory<?>> singletonFactories = new HashMap(16)
    
    // 此缓存主要是提供获取spring bean的
    @FunctionalInterface
    public interface ObjectFactory<T> {
        T getObject() throws BeansException;
    }

- earlySingletonObjects

二级缓存，保存已经初始化，但是未赋值的spring bean对象的容器

    Map<String, Object> earlySingletonObjects = new HashMap(16)

### 完整的获取流程

    getBean -> doGetBean -> doCreateBean -> popluateBean -> addSingleton

- getBean(A.class)的时候，首先对A进行了初始化，但是还未赋值，这个时候将发现A需要B.class对象，因此将A获取的函数封装，放入到三级缓存中，然后获取B

- getBean(B.class)的时候，对B进行了初始化，然后对A.class字段进行赋值，查一级缓存没找到A，然后去二级缓存没找到A，然后到三级缓存找A，找到了工厂方法获取到A，将A赋值给B，然后将A放入到二级缓存，并从三级中删除。此时B完成初始化，放入到一级缓存中

- 这个时候A调用populateBean已经结束，A这个时候能拿到B了，然后将B赋值给A的属性，然后将A从二级缓存删除，放入到一级缓存中去
