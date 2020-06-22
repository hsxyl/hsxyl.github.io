---
title: 记aop内部调用不生效
date: 2020-06-20 15:32:56
updated: 2020-06-20 15:32:56
categories:
tags:
---
# 工作记录
- **需求:** 有一个执行任务的功能，需要持久化任务的执行状态。
- **为什么用aop:** 执行任务的功能和记录任务状态变化的功能如果是耦合的，就会在执行任务的逻辑里面添加一堆if..else去判断任务的状态然后添加持久化的逻辑，代码可读性，维护性都很差，所以采用aop+注解的方式去解耦。
- **遇到的问题:** 
    1. 非接口定义的方法没有被代理
        - 原因是spring的aop动态代理底层有两层实现：基于JDK的和基于gclib的，两者的一个重要区别在于JDK代理只能代理接口定义的方法，gclib则无该限制，对于接口实现类会默认使用基于JDK的动态代理。
    2. 使用了gclib后，内部调用方法无法被代理
        - gclib动态代理的大致原理是创建一个被代理类的子类，通过实现父类的非final方法实现拦截监听的.(也就是说方法具体的实现gclib管不着只是在执行前后帮你生成了一些代码，所以内部的调用他无能为力)
      
# 代码实例
- 定义service和给出实现，实现类里面存着内部调用关系：
``` java
@Service
public class AopDemoServiceImpl implements AopDemoService {
    @Override
    public void method1() {
        System.out.println("I am method1");
        method2();
    }

    public void method2() {
        System.out.println("I am method2");
    }
}
```
- 定义切点和通知来监听实现类的两个方法：
``` java
@Pointcut(value = "execution(* com.example.demo.service.AopDemoServiceImpl.*(..))")
public void demoPointCut() {}

@Around(value = "demoPointCut()")
public void demoAroud(ProceedingJoinPoint pjp) throws Throwable {
    // 输出方法名
    System.out.println("before method:"+pjp.getSignature());
    pjp.proceed();
    System.out.println("after method:"+pjp.getSignature());
}
```
- 编写一个测试类来看看调用AopDemoServiceImpl.method1()会发生什么：
``` java
    @Autowired
    public AopDemoService aopDemoService;

    @Test
    public void aopTest() {
        aopDemoService.method1();
    }
```
- 我们期望的情况应该是aop里面的提示信息环绕着方法的执行被打印出来,如下：
``` java
before method:void com.example.demo.service.AopDemoServiceImpl.method1()
I am method1
before method:void com.example.demo.service.AopDemoServiceImpl.method2()
I am method2
after method:void com.example.demo.service.AopDemoServiceImpl.method2()
after method:void com.example.demo.service.AopDemoServiceImpl.method1()
```
- 然而实际输出的确是：
``` java
before method:void com.example.demo.service.AopDemoServiceImpl.method1()
I am method1
I am method2
after method:void com.example.demo.service.AopDemoServiceImpl.method1()
```
 原因上文也提到了，gclib生成被代理对象的子类，重写子类的非final方法，但是并非去重写方法的具体细节，而是在方法的前后增加了逻辑，然后再去调用原本的方法，所以对于内部的调用gclib无能为力。

# 解决方案
- 1. 第一种思路是，既然代理类无法感知到内部调用，那我们修改原本的内部调用，去调用代理类生成后的方法。
    - 1.1 使用`AopContext.currentProxy()`去获取代理类： 
    ``` java
    @Override
        public void method1() {
            System.out.println("I am method1");
            // 另外注意：要设置aop实体暴露出来。在springboot的入口类里面加上@EnableAspectJAutoProxy(proxyTargetClass = true, exposeProxy = true)
            ((AopDemoServiceImpl) AopContext.currentProxy()).method2();
        }

        public void method2() {
            System.out.println("I am method2");
        }
    ```
    - 1.2 将代理类生成后注入进来:
    
    从ApplicationContext获取代理对象:
    ``` java
    @Autowired
    private ApplicationContext context;

    private AopDemoServiceImpl proxyObject;
    @PostConstruct
    // 初始化方法，在IOC注入完成后会执行该方法
    private void setSelf() {
        // 从spring上下文获取代理对象（直接通过proxyObject=this是不对的，this是目标对象）
        // 此种方法不适合于prototype Bean，因为每次getBean返回一个新的Bean
        proxyObject = context.getBean(AopDemoServiceImpl.class);
    }
    ```
    或者使用@Lazy注解去延迟加载：
    ```
    private AopDemoServiceImpl proxyObject;

    @Autowired
    @Lazy
    public void setTestService(AopDemoServiceImpl proxyObject) {
        this.proxyObject = proxyObject;
    }
    ```

3. 使用aspectJ织入，aspectJ直接在源类上进行字节码增强，而非代理的方式去实现aop。

# Todo
- [ ] 目前只是从别人的博客上看懂了这g过程，需要深入研究gclib生成过程，学习java编译过程原理，查看gclib生成的class文件，找到实际的证据。 
- [ ] 学习AspectJ
- [ ] AopContext.currentProxy()是如何实现的
- [ ] 延迟加载注解的原理 
        
# 参考
- [CGLIB(Code Generation Library)详解](https://blog.csdn.net/danchu/article/details/70238002)

- [spring aop问题解决](https://www.cnblogs.com/wulm/p/9486531.html)
