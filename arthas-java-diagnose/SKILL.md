---
name: arthas-java-diagnose
description: 使用 Arthas 诊断 Java 应用程序故障，包括 CPU 高、内存泄漏、线程死锁、慢调用、GC 问题、OOM 等常见线上问题。
version: 1.0.0
---

# Java 故障诊断（Arthas）

使用 Arthas 诊断 Java 应用程序的线上故障。Arthas 是阿里巴巴开源的 Java 诊断工具，可以实时附加到运行中的 JVM 进程进行问题排查。

## 前置条件

- 目标机器上已安装 Arthas（或可通过 `java -jar arthas-boot.jar` 启动）
- 有足够的权限连接到目标 JVM 进程
- 目标 JVM 运行在 JDK 6+

### 安装 Arthas

```bash
# 方式一：一键安装
curl -O https://arthas.aliyun.com/arthas-boot.jar

# 方式二：使用 as.sh
curl -L https://arthas.aliyun.com/install.sh | sh
```

## 工作流程

1. 确认用户描述的故障现象（高 CPU、OOM、线程阻塞、响应慢等）
2. 找到目标 Java 进程 PID
3. 启动 Arthas 附加到目标 JVM
4. 根据故障类型执行相应的诊断命令
5. 分析输出结果，定位根因
6. 给出解决建议

## 查找 Java 进程

```bash
# 找到 Java 进程 PID
ps aux | grep java
jps -l

# 确认进程状态
top -p <PID>
```

## 启动 Arthas

```bash
# 交互式启动（选择目标进程）
java -jar arthas-boot.jar

# 直接附加到指定 PID
java -jar arthas-boot.jar <PID>

# 以脚本模式执行命令（非交互式）
java -jar arthas-boot.jar <PID> -c "command"
```

## 诊断命令参考

### 1. 高 CPU 问题

```bash
# 查看最耗 CPU 的线程（按 CPU 使用率排序，前 5 个）
thread -n 5

# 查看指定线程的堆栈
thread <thread-id>

# 查看 CPU 使用率排名并显示详细信息
thread --state RUNNABLE
```

### 2. 死锁检测

```bash
# 自动检测死锁并显示详情
thread -b

# 查看所有线程及状态
thread --state BLOCKED
```

### 3. 内存泄漏 / OOM 排查

```bash
# 查看堆内存使用概况
memory

# 查看 GC 信息
memory | grep -E "heap|non-heap"

# 查看 Class 加载信息
classloader

# 统计所有类加载数量
classloader -t

# 查看 TOP 大对象（前 10 个）
vmtool --action getInstances --className java.lang.String --limit 10

# 查看某个类实例数量
vmtool --action countInstances --className com.example.MyClass
```

### 4. GC 问题

```bash
# 查看 GC 实时信息
dashboard

# 每隔 5 秒刷新一次
dashboard -i 5000

# 查看 JVM 内存概况
memory

# 触发 Full GC
vmtool --action forceGc
```

### 5. 慢调用 / 接口耗时

```bash
# 监控某个方法的调用耗时（5 秒内）
trace <全限定类名> <方法名> -n 5

# trace 示例：监控 Controller 接口
trace com.example.controller.UserController getUserInfo -n 5

# 方法调用链路监控（包含参数）
trace com.example.service.UserService queryUser -n 5 --skipJDKMethod false

# 耗时大于 100ms 的调用
trace com.example.service.UserService queryUser '#cost > 100'
```

### 6. 方法调用观测

```bash
# 查看方法入参、返回值和异常
watch com.example.service.UserService queryUser "{params,returnObj,throwExp}" -x 3

# 监听方法调用（每 5 秒输出一次）
watch com.example.service.UserService queryUser "{params,returnObj}" -x 3 -n 10

# 监听特定异常
watch com.example.service.UserService queryUser "{throwExp}" -e -x 3

# 方法执行耗时监控
monitor com.example.service.UserService queryUser -c 5
```

### 7. 查看入参 / 返回值详情

```bash
# 查看方法入参（展开深度 3 层）
watch com.example.controller.UserController getUserInfo "{params}" -x 3

# 查看返回值
watch com.example.controller.UserController getUserInfo "{returnObj}" -x 3

# 条件过滤：仅当第一个参数为特定值时触发
watch com.example.controller.UserController getUserInfo "{params,returnObj}" "params[0].toString().contains('admin')"
```

### 8. 反编译代码

```bash
# 反编译类（确认线上运行的代码版本）
jad com.example.service.UserServiceImpl

# 反编译指定方法
jad com.example.service.UserServiceImpl queryUser

# 显示行号和源码
jad --source-only com.example.service.UserServiceImpl
```

### 9. 修改日志级别（临时排障）

```bash
# 查看某个类的日志级别
logger -c com.example.service.UserService

# 临时修改日志级别为 DEBUG
logger --name ROOT --level DEBUG

# 查看所有 Logger 信息
logger
```

### 10. 查看 JVM 参数和属性

```bash
# 查看 JVM 启动参数
vmoption

# 查看指定 JVM 参数
vmoption MaxHeapSize
vmoption PrintGC

# 查看系统属性
sysprop

# 修改 JVM 参数（临时生效，不重启）
vmoption PrintGC true
```

### 11. 查看 Spring Bean（如使用 Spring）

```bash
# 通过 ognl 查看 Spring Context 中的 Bean
ognl '#context=@org.springframework.web.context.ContextLoader@getCurrentWebApplicationContext().getBean("userService")'

# 调用 Bean 方法
ognl '#context=@org.springframework.web.context.ContextLoader@getCurrentWebApplicationContext().getBean("userService").queryUser(1)'
```

## 常见故障诊断流程

### 场景一：CPU 100%

```
1. thread -n 5              → 定位最耗 CPU 的线程
2. thread <thread-id>       → 查看该线程堆栈，找到业务代码
3. 分析代码逻辑（死循环、频繁 GC、正则回溯等）
```

### 场景二：OOM / 内存泄漏

```
1. memory                   → 查看堆/非堆内存使用
2. dashboard -i 5000        → 观察内存变化趋势
3. classloader              → 检查是否有类加载泄漏
4. vmtool --action countInstances → 统计疑似泄漏的类实例数
5. jmap dump                → 获取 heap dump（Arthas 外）
   jmap -dump:live,format=b,file=/tmp/heap.hprof <PID>
```

### 场景三：接口响应慢

```
1. trace <Controller> <method> '#cost > 100'  → 找到慢调用链
2. trace <Service> <method> '#cost > 50'      → 逐层深入
3. 分析是 SQL 慢查询、RPC 调用慢还是业务逻辑问题
```

### 场景四：线程死锁 / 线程阻塞

```
1. thread -b               → 检测死锁
2. thread --state BLOCKED  → 查看被阻塞的线程
3. thread <thread-id>      → 查看具体堆栈
```

### 场景五：请求结果不符合预期

```
1. watch <类> <方法> "{params,returnObj}" -x 3  → 查看入参和返回值
2. 对比入参和返回值，定位逻辑错误
```

## 注意事项

- Arthas 会通过 JVM Attach API 连接到目标进程，**不会修改业务代码**
- 诊断命令具有**运行时观测能力**，不会重启 JVM
- 生产环境使用请谨慎，避免高频 trace/watch 造成性能开销
- `-n` 参数限制命令执行次数，用完即止
- `-x` 参数控制结果展开深度，避免输出过多
- 诊断完成后建议执行 `stop` 或 `exit` 退出 Arthas
- 若目标进程以 `-XX:+DisableAttachMechanism` 启动，则无法 Attach

## 示例输出

```
$ thread -n 3
"http-nio-8080-exec-10" Id=45 cpuUsage=85% prio=5 state=RUNNABLE
    at com.example.service.UserService.queryUser(UserService.java:125)
    at com.example.controller.UserController.getUserInfo(UserController.java:45)
    ...

$ memory
 heap                                                                   usage
  PS Eden Space                                     256.00M   12.50%
  PS Survivor Space                                   5.00M    0.00%
  PS Old Gen                                       1024.00M   89.30%
 non-heap
  Metaspace                                         256.00M   75.60%
  Code Cache                                        128.00M   45.20%
```