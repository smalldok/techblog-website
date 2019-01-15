---
title: spring-retry 破坑
comments: true
tags:
  - 重试
  - 降级
categories: 服务治理
abbrlink: 26380
date: 2017-08-30 10:12:00
---

## 使用场景
`A接口` -> `B接口` 调用失败，重试、重试失败降级处理；  
如：实现第三方的app Push推送；  
如：消息系统推送消息给订阅方；

## pom.xml
项目基于springboot，在根pom.xml中引用，所以这里没有`<version>`，以springboot中的版本为准;  
项目根`pom.xml`
```JAVA
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.4.0.RELEASE</version>
    <relativePath/>
</parent>
```
项目mode`pom.xml`
```JAVA
<!-- spring-retry -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
</dependency>
```
## spring-retry知识
```JAVA
@Retryable(value= {RemoteAccessException.class},maxAttempts = 3,backoff = @Backoff(delay = 1000l,multiplier = 1))
public void call(String param){
}
@Recover
public void recover(RemoteAccessException e) {
}
```
* `@Retryable`  
`value`选择需要重试的异常，可指定多个；   
`maxAttempts` 最大重试次数,超过则调用@Recover指定的方法(根据异常类型)  
`delay` 每次重试的间隔时间
* `@Recover`  
当超过重试次数，则调用此注解的方法(根据异常类型)

`使用原则：不要全部异常都进行重试，要有选择性的根据异常重试；`

## 两种使用方式
我目前使用的第一种方式，因为业务比较简单；
### 1、@Retryable和@Recover在同一个类中
`@EnableRetry`启动配置  
```java
@Configuration
@EnableRetry
class RetryConfig{
}
```
业务类RemoteService.java
```java
@Service
public class RemoteService {
    @Retryable(value= {RemoteAccessException.class},maxAttempts = 3,backoff = @Backoff(delay = 1000l,multiplier = 1))
    public void call(String param){
        System.out.println("do something...");
        throw new RemoteAccessException("RPC调用异常");
    }
    @Recover
    public void recover(RemoteAccessException e) {
        System.out.println("recover====>"+e.getMessage());
    }
}
```
约束：  
1、@Retryable 和 @Recover必须在同一个类中  
2、这个类必须是受spring管理的bean  
3、方法必须是public


### 2、拦截器
`@EnableRetry`启动配置,且加上`拦截器`
```java
@Configuration
@ConditionalOnProperty(prefix = "mq.consumer.callback", name = {"retry-times","retry-delay-inMilliseconds"} ,matchIfMissing = false)
@EnableRetry
public class RetryConfig{
    @Autowired
    private ConsumerProperties consumerProperties;

    //每一个业务方法对应一个拦截器和自定义recover
    @Bean
    @ConditionalOnMissingBean(name = "retryInterceptor")
    public RetryOperationsInterceptor retryInterceptor() {
        return RetryInterceptorBuilder
                .stateless().recoverer(new CustomMessageRecover())
                .backOffOptions(0L,1D, consumerProperties.getCallback().getRetryDelayInMilliseconds())
                .maxAttempts(consumerProperties.getCallback().getRetryTimes()).build();
    }
    static class CustomMessageRecover implements MethodInvocationRecoverer<Void> {
        @Override
        public Void recover(Object[] args, Throwable cause) {
            System.out.println("IN THE RECOVER ZONE!!!");
            return null;
        }
    }
}
```
业务类RemoteService.java
```java
@Service
public class RemoteService {
    @Retryable(value= {RemoteAccessException.class},interceptor = "retryInterceptor")
    public void call(String param){
        System.out.println("do something...");
        throw new RemoteAccessException("RPC调用异常");
    }
}
```
注意到没？@Recover注解的方法不在里面了；已经由拦截器代理到CustomMessageRecover类
