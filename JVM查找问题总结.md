> Java应用服务器运行时, 经常发生这三类问题: 
>
> 1. CPU占用100%
> 2. 请求卡死
> 3. 请求执行慢   
>
> 解决这些问题的步骤是, 先查找出问题发生点, 然后解决问题  
> "问题发生点", 即导致问题发生的方法

## CPU占用100%

### 确定问题

1. top命令查看CPU占用高的进程

   ```shell
   [root ~]# top 
     PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                          
   18453 root      20   0 27.7g 6.4g  39m S 57.0 13.5 196:53.52 java                                                                                                                                                 
    8341 root      20   0 1455m  27m 7284 S  8.9  0.1 693:58.34 edr_agent                                                                                                                                            
    2071 root      20   0  171m 4552 3436 S  0.3  0.0 130:20.43 vmtoolsd                                                         
   ```

2. `top -Hp <Java进程ID>`查看Java内部线程占用CPU情况, 确定占用CPU高的Java线程ID  

   ``` shell
   [root ~]# top -Hp 18453
     PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                                                                                                             
   18496 root      20   0 27.7g 6.2g  23m S  5.9 13.1   3:22.28 java                                                                                                                                                 
   18785 root      20   0 27.7g 6.2g  23m S  2.0 13.1   0:14.10 java                                                                                                                                                 
   18453 root      20   0 27.7g 6.2g  23m S  0.0 13.1   0:00.00 java
   ```

   

3. `printf "%x\n" <线程ID>`将线程ID转换为十六进制  

    ```shell
    [root ~]# printf '%x\n' 18496
    4840
    ```

4. `jstack <Java进程ID> | grep --color -C 20 <十六进制线程ID>`查看JVM线程堆栈情况  

   ```
   "NioBlockingSelector.BlockPoller-1" #22 daemon prio=5 os_prio=0 tid=0x00007f8190539000 nid=0x4840 runnable [0x00007f81151d2000]
      java.lang.Thread.State: RUNNABLE
   	at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
   	at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
   	at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
   	at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
   	- locked <0x00000004cdca2448> (a sun.nio.ch.Util$3)
   	- locked <0x00000004cdca2458> (a java.util.Collections$UnmodifiableSet)
   	- locked <0x00000004cdca2400> (a sun.nio.ch.EPollSelectorImpl)
   	at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
   	at org.apache.tomcat.util.net.NioBlockingSelector$BlockPoller.run(NioBlockingSelector.java:342)
   ```

   `nid`即线程ID

5. 确定问题   

    > 栈顶即线程当前执行方法  
	
	当前执行方法可能在有这几种:
    >
    > 1. Java代码
    > 2. GC
    > 3. 数据库查询

​       这几种情况需要不同的解决方法

### 解决问题

#### Java代码

需要排查代码， 比如是否有死循环等

#### GC

1. `free -g`确定内存使用情况  

   ``` 
   [root ~]# free -g
                total       used       free     shared    buffers     cached
   Mem:            47         38          8          0          0         28
   ```

   这里free(空闲)内存有8G, 是正常的, 当空闲空间为0时, 就可能会导致JVM 频繁GC

2. `jmap -F -dump:format=b,file=<文件名> <Java进程ID>`将JVM内存dump下来

3. 使用[mat工具](https://www.eclipse.org/mat/)查看  
   重点查看unreachable objects不可达对象, dominator tree支配树 等  
   支配树可以看到内存关系  
   不可达对象过大可能是内存泄漏了, 需要配合jstack进一步排查泄漏点

#### 数据库查询

需要结合数据库慢查询日志/耗时SQL/CPU占用SQL等, 进一步排查

## 请求执行卡死

1. 确定问题: jstack查看请求当前正在执行哪个方法
2. 解决问题: 同

## 请求执行慢

> 先排查外部资源问题, 如内存/CPU/硬盘占用后
>
> 执行慢但是仍然在执行, 可以通过重复执行复现问题

### 确定问题

#### 日志

通过已有日志, 或新增日志, 推算出执行慢的方法

#### Arthas

通过[Arthas trace](https://arthas.aliyun.com/doc/trace.html) 统计方法的执行时间, 如:

``` 
trace demo.MathGame run
```

可以统计到run方法体各个语句的执行时间, 如:  

```
[arthas@59161]$ trace demo.MathGame run
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 112 ms, listenerId: 1
`---ts=2020-07-09 16:48:11;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
    `---[1.389634ms] demo.MathGame:run()
        `---[0.123934ms] demo.MathGame:primeFactors() #24 [throws Exception]
```

在这里示例里, run方法执行了1.389634ms, 其中调用primeFactors方法花费了0.123934ms.  
但是无法统计到primeFactors内部的具体执行时间, 因此提前预判有问题的方法, 或trace多个方法

### 解决问题

同
