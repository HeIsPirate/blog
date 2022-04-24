# 现象

`AOPContext.currentProxy() != this`

# 知识

1. Spring注入的依赖, 默认是反射得来的对象
2. 只有当开启AOP, 且存在一个被PointCut切入点关联的方法的类, 才会生成代理对象(JDK原生或CGLib)  
   (被@Transaction注解的方法类似)
3. AOPContenxt实现原理是, 通过代理, 在方法执行前, 将代理对象缓存在ThreadLocal中

# 原因

1. 调用当前类的前一个类被AOP切面了, 所以它的Bean是代理对象  
   当前类没有被AOP切面, 是原始对象  
   导致ThreadLocal里缓存的是前一个类

# 其他

1. Spring AOP强依赖AspectJ. 只要开启了切面, 都需要AspectJ  
2. 在实现`Adviser增强处理`操作上, 有两种方式
   1. AspectJ的@Before & @After & @Around & @AfterThrowing  
      原理是静态织入, 操作字节码  
      (但是Spring里面的AspectJ切入是通过"动态代理"实现的, 和AspectJ原生不同)  
      (Spring只是用了AspectJ的风格)
   2. Spring原生, 实现`MethodInterceptor`接口  
      原理是`动态代理`  
      它也是通过AspectJ pointcut实现切入点的