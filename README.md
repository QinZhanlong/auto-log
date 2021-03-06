# auto-log

[auto-log](https://github.com/houbb/auto-log) 是一款为 java 设计的自动日志监控框架。

[![Build Status](https://travis-ci.com/houbb/auto-log.svg?branch=master)](https://travis-ci.com/houbb/auto-log)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.github.houbb/auto-log/badge.svg)](http://mvnrepository.com/artifact/com.github.houbb/auto-log)
[![](https://img.shields.io/badge/license-Apache2-FF0080.svg)](https://github.com/houbb/auto-log/blob/master/LICENSE.txt)
[![Open Source Love](https://badges.frapsoft.com/os/v2/open-source.svg?v=103)](https://github.com/houbb/auto-log)

## 创作目的

经常会写一些工具，有时候手动加一些日志很麻烦，引入 spring 又过于大材小用。

所以希望从从简到繁实现一个工具，便于平时使用。

## 特性

- 基于注解+字节码，配置灵活

- 自动适配常见的日志框架

- 支持编程式的调用

- 支持注解式，完美整合 spring

- 支持整合 spring-boot

- 支持慢日志阈值指定，耗时，入参，出参，异常信息等常见属性指定

- 支持 traceId 特性

- 支持类级别定义注解

- 支持自定义拦截器和过滤器

## 变更日志

- v0.0.13 变化

支持 `WebParamFilter` 作为默认的参数过滤器

> [变更日志](https://github.com/houbb/auto-log/blob/master/CHANGELOG.md)

# 快速开始

## maven 引入

```xml
<dependency>
    <group>com.github.houbb</group>
    <artifact>auto-log-core</artifact>
    <version>${最新版本}</version>
</dependency>
```

## 入门案例

```java
UserService userService = AutoLogHelper.proxy(new UserServiceImpl());
userService.queryLog("1");
```

- 日志如下

```
[INFO] [2020-05-29 16:24:06.227] [main] [c.g.h.a.l.c.s.i.AutoLogMethodInterceptor.invoke] - public java.lang.String com.github.houbb.auto.log.test.service.impl.UserServiceImpl.queryLog(java.lang.String) param is [1]
[INFO] [2020-05-29 16:24:06.228] [main] [c.g.h.a.l.c.s.i.AutoLogMethodInterceptor.invoke] - public java.lang.String com.github.houbb.auto.log.test.service.impl.UserServiceImpl.queryLog(java.lang.String) result is result-1
```

### 代码

其中方法实现如下：

- UserService.java

```java
public interface UserService {

    String queryLog(final String id);

}
```

- UserServiceImpl.java

直接使用注解 `@AutoLog` 指定需要打日志的方法即可。

```java
public class UserServiceImpl implements UserService {

    @Override
    @AutoLog
    public String queryLog(String id) {
        return "result-"+id;
    }

}
```

## TraceId 的例子

### 代码

```java
UserService service =  AutoLogProxy.getProxy(new UserServiceImpl());
service.traceId("1");
```

其中 traceId 方法如下：

```java
@AutoLog
@TraceId
public String traceId(String id) {
    return id+"-1";
}
```

### 测试效果

```
信息: [ba7ddaded5a644e5a58fbd276b6657af] <traceId>入参: [1].
信息: [ba7ddaded5a644e5a58fbd276b6657af] <traceId>出参：1-1.
```

其中 ba7ddaded5a644e5a58fbd276b6657af 就是对应的 traceId，可以贯穿整个 thread 周期，便于我们日志查看。

# 注解说明

## @AutoLog

核心注解 `@AutoLog` 的属性说明如下：

| 属性 | 类型 | 默认值 | 说明 |
|:--|:--|:--|:--|
| enable | boolean | true | 是否启用 |
| param | boolean | true | 是否打印入参 |
| result | boolean | true | 是否打印出参 |
| costTime | boolean | false | 是否打印耗时 |
| exception | boolean | true | 是否打印异常 |
| slowThresholdMills | long | -1 | 当这个值大于等于 0 时，且耗时超过配置值，会输出慢日志 |
| description | string |"" | 方法描述，默认选择方法名称 |
| interceptor | Class[] | 默认实现 | 拦截器实现，支持指定多个和自定义 |
| paramFilter | Class | WebParamFilter | 入参过滤器，支持自定义 |

## @TraceId

`@TraceId` 放在需要设置 traceId 的方法上，比如 Controller 层，mq 的消费者，rpc 请求的接受者等。

| 属性 | 类型 | 默认值 | 说明 |
|:--|:--|:--|:--|
| id | Class | 默认为 uuid | traceId 的实现策略 |
| putIfAbsent | boolean | false | 是否在当前线程没有值的时候才设置值 |
| enable | boolean | true | 是否启用 |
| interceptor | Class[] | 默认实现 | 拦截器实现，支持指定多个和自定义 |

# 自定义策略

## 自定义日志拦截器（interceptor）

### 内置拦截器

`AutoLogInterceptor` 默认实现

### 定义

直接继承自 `AbstractAutoLogInterceptor` 类，并且实现对应的方法即可。

```java
public class MyAutoLogInterceptor extends AbstractAutoLogInterceptor {

    @Override
    protected void doBefore(AutoLog autoLog, IAutoLogInterceptorContext context) {
        System.out.println("自定义入参：" + Arrays.toString(context.filterParams()));
    }

    @Override
    protected void doAfter(AutoLog autoLog, Object result, IAutoLogInterceptorContext context) {
        System.out.println("自定义出参：" + result);
    }

    @Override
    protected void doException(AutoLog autoLog, Exception exception, IAutoLogInterceptorContext context) {
        System.out.println("自定义异常：");
        exception.printStackTrace();
    }

}
```

### 使用

如下，这样日志输出，就会使用上面的指定策略。

```java
@AutoLog(interceptor = MyAutoLogInterceptor.class)
public String my() {
    return "自定义策略";
}
```

## 自定义入参过滤器（paramFilter）

### 内置

`WebParamFilter` 主要用于过滤 HttpRequest HttpServlet 等无法直接 JSON 序列化的对象。

### 自定义

直接继承 `AbstractParamFilter` 类实现对应的方法即可。

```java
public class MyParamFilter extends AbstractParamFilter {

    @Override
    protected Object[] doFilter(Object[] params) {
        Object[] newParams = new Object[1];
        newParams[0] = "设置我我想要的值";
        return newParams;
    }

}
```

### 使用

指定对应的参数过滤器。这样，无论入参是什么，都会变成我们指定的 `[设置我我想要的值]`。

```java
@AutoLog(paramFilter = MyParamFilter.class)
public String paramFilter() {
    return "自定义入参过滤器";
}
```


# spring 整合使用

完整示例参考 [SpringServiceTest](https://github.com/houbb/auto-log/tree/master/auto-log-test/src/test/java/com/github/houbb/auto/log/spring/SpringServiceTest.java)

## 注解声明

使用 `@EnableAutoLog` 启用自动日志输出

```java
@Configurable
@ComponentScan(basePackages = "com.github.houbb.auto.log.test.service")
@EnableAutoLog
public class SpringConfig {
}
```

## 测试代码

```java
@ContextConfiguration(classes = SpringConfig.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class SpringServiceTest {

    @Autowired
    private UserService userService;

    @Test
    public void queryLogTest() {
        userService.queryLog("1");
    }

}
```

- 输出结果

```
信息: public java.lang.String com.github.houbb.auto.log.test.service.impl.UserServiceImpl.queryLog(java.lang.String) param is [1]
五月 30, 2020 12:17:51 下午 com.github.houbb.auto.log.core.support.interceptor.AutoLogMethodInterceptor info
信息: public java.lang.String com.github.houbb.auto.log.test.service.impl.UserServiceImpl.queryLog(java.lang.String) result is result-1
五月 30, 2020 12:17:51 下午 org.springframework.context.support.GenericApplicationContext doClose
```

# springboot 整合使用

## maven 引入

```xml
<dependency>
    <groupId>com.github.houbb</groupId>
    <artifactId>auto-log-springboot-starter</artifactId>
    <version>最新版本</version>
</dependency>
```

只需要引入 jar 即可，其他的什么都不用配置。

使用方式和 spring 一致。

## 测试

```java
@Autowired
private UserService userService;

@Test
public void queryLogTest() {
    userService.query("spring-boot");
}
```

# 开源地址

> Github: [https://github.com/houbb/auto-log](https://github.com/houbb/auto-log)

> Gitee: [https://gitee.com/houbinbin/auto-log](https://gitee.com/houbinbin/auto-log)

# Road-Map

- [ ] 优化日志中的方法路径名称

考虑补全对应的类信息

- [ ] 全局配置

比如全局的慢日志阈值设置等

- [ ] jvm-sandbox 特性

- [ ] 编译时注解特性 