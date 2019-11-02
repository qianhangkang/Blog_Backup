---
title: Spring相关注解笔记
date: 2019-11-02 14:45:21
tags: ['Java','Spring']
---

整理spring注解~

<!--more-->

# @Autowired

```java
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {

   /**
    * Declares whether the annotated dependency is required.
    * <p>Defaults to {@code true}.
    */
   boolean required() default true;

}
```

自动注入bean；

@Autowired(required = false)表示即使被修饰的bean无法被注入，也不会报错；否则spring容器不会运行起来

# @Qualifier

```java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Qualifier {

   String value() default "";

}
```

通过 @Qualifier 注释指定注入 Bean 的名称，防止容器中有多个名字相同的bean导致@Autowired注入引起歧义。

1. value，对应被注入的bean的名字

# @AliasFor

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface AliasFor {

	/**
	 * Alias for {@link #attribute}.
	 * <p>Intended to be used instead of {@link #attribute} when {@link #annotation}
	 * is not declared &mdash; for example: {@code @AliasFor("value")} instead of
	 * {@code @AliasFor(attribute = "value")}.
	 */
	@AliasFor("attribute")
	String value() default "";

	/**
	 * The name of the attribute that <em>this</em> attribute is an alias for.
	 * @see #value
	 */
	@AliasFor("value")
	String attribute() default "";

	/**
	 * The type of annotation in which the aliased {@link #attribute} is declared.
	 * <p>Defaults to {@link Annotation}, implying that the aliased attribute is
	 * declared in the same annotation as <em>this</em> attribute.
	 */
	Class<? extends Annotation> annotation() default Annotation.class;

}
```

用来声明属性的别名

# @Schedule

```java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(Schedules.class)
public @interface Scheduled {

   /**
    * A cron-like expression, extending the usual UN*X definition to include triggers
    * on the second as well as minute, hour, day of month, month and day of week.
    * <p>E.g. {@code "0 * * * * MON-FRI"} means once per minute on weekdays
    * (at the top of the minute - the 0th second).
    * @return an expression that can be parsed to a cron schedule
    * @see org.springframework.scheduling.support.CronSequenceGenerator
    */
   String cron() default "";

   /**
    * A time zone for which the cron expression will be resolved. By default, this
    * attribute is the empty String (i.e. the server's local time zone will be used).
    * @return a zone id accepted by {@link java.util.TimeZone#getTimeZone(String)},
    * or an empty String to indicate the server's default time zone
    * @since 4.0
    * @see org.springframework.scheduling.support.CronTrigger#CronTrigger(String, java.util.TimeZone)
    * @see java.util.TimeZone
    */
   String zone() default "";

   /**
    * Execute the annotated method with a fixed period in milliseconds between the
    * end of the last invocation and the start of the next.
    * @return the delay in milliseconds
    */
   long fixedDelay() default -1;

   /**
    * Execute the annotated method with a fixed period in milliseconds between the
    * end of the last invocation and the start of the next.
    * @return the delay in milliseconds as a String value, e.g. a placeholder
    * @since 3.2.2
    */
   String fixedDelayString() default "";

   /**
    * Execute the annotated method with a fixed period in milliseconds between
    * invocations.
    * @return the period in milliseconds
    */
   long fixedRate() default -1;

   /**
    * Execute the annotated method with a fixed period in milliseconds between
    * invocations.
    * @return the period in milliseconds as a String value, e.g. a placeholder
    * @since 3.2.2
    */
   String fixedRateString() default "";

   /**
    * Number of milliseconds to delay before the first execution of a
    * {@link #fixedRate()} or {@link #fixedDelay()} task.
    * @return the initial delay in milliseconds
    * @since 3.2
    */
   long initialDelay() default -1;

   /**
    * Number of milliseconds to delay before the first execution of a
    * {@link #fixedRate()} or {@link #fixedDelay()} task.
    * @return the initial delay in milliseconds as a String value, e.g. a placeholder
    * @since 3.2.2
    */
   String initialDelayString() default "";

}
```

被@Schedule修饰的方法会按照相应的参数定时执行

1. cron表达式
2. zone，cron表达式对应的时区，默认是本地服务器的时区
3. fixedDelay，在最后一次调用结束和下一次调用开始之间以固定周期（以毫秒为单位）执行带注释的方法，ps：类似于串行执行，只不过中间有一个固定的delay。
4. fixedDelayString
5. fixedRate，不同于fixedDelay，方法之间的调用总是会以固定的周期调用，不管上一个方法结束与否
6. fixedRateString
7. initialDelay，第一次调用方法之前delay的时间
8. initialDelayString

## Spring如何实现@Schedule

1. 首先需要开启`@EnableScheduling`注解，开启该注解后Spring会在配置类`SchedulingConfiguration`中自动注入`ScheduledAnnotationBeanPostProcessor`
2. `ScheduledAnnotationBeanPostProcessor`类中会将被`@Scheduled`修饰的方法注册到`ScheduledTaskRegistrar`中，同时注入默认的线程池`DEFAULT_TASK_SCHEDULER_BEAN_NAME`（这个线程池的配置可以自己通过实现接口`SchedulingConfigurer`来自定义），**默认的线程池是单线程的**。方法会根据注解中给定的参数在线程池中执行

# @Async

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Async {

   /**
    * A qualifier value for the specified asynchronous operation(s).
    * <p>May be used to determine the target executor to be used when executing this
    * method, matching the qualifier value (or the bean name) of a specific
    * {@link java.util.concurrent.Executor Executor} or
    * {@link org.springframework.core.task.TaskExecutor TaskExecutor}
    * bean definition.
    * <p>When specified on a class level {@code @Async} annotation, indicates that the
    * given executor should be used for all methods within the class. Method level use
    * of {@code Async#value} always overrides any value set at the class level.
    * @since 3.1.2
    */
   String value() default "";

}
```

该注释作用于方法上，被修饰的方法会以异步的方式执行。

1. value，对应执行该方法的线程池名字，默认springboot会去匹配Executor或TaskExecutor

被修饰的方法可以存在方法参数，可以有返回值（`Future`AND`CompletableFuture`）

**NOTE：该注解不能同时与@PostConstruct一起使用，但是可以再@PostConstruct中调用被@Async修饰的方法**

> ```
> @Async` can not be used in conjunction with lifecycle callbacks such as `@PostConstruct
> ```

# @PostConstruct

```java
@Documented
@Retention (RUNTIME)
@Target(METHOD)
public @interface PostConstruct {
}
```

该注释只能作用于方法上，该方法会被执行在对应的类依赖注入完成之后。一般用于初始化操作，如读取相应的缓存等

# @Retryable

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Retryable {

	/**
	 * Retry interceptor bean name to be applied for retryable method. Is mutually
	 * exclusive with other attributes.
	 * @return the retry interceptor bean name
	 */
	String interceptor() default "";

	/**
	 * Exception types that are retryable. Synonym for includes(). Defaults to empty (and
	 * if excludes is also empty all exceptions are retried).
	 * @return exception types to retry
	 */
	Class<? extends Throwable>[] value() default {};

	/**
	 * Exception types that are retryable. Defaults to empty (and if excludes is also
	 * empty all exceptions are retried).
	 * @return exception types to retry
	 */
	Class<? extends Throwable>[] include() default {};

	/**
	 * Exception types that are not retryable. Defaults to empty (and if includes is also
	 * empty all exceptions are retried).
	 * @return exception types to retry
	 */
	Class<? extends Throwable>[] exclude() default {};

	/**
	 * A unique label for statistics reporting. If not provided the caller may choose to
	 * ignore it, or provide a default.
	 *
	 * @return the label for the statistics
	 */
	String label() default "";

	/**
	 * Flag to say that the retry is stateful: i.e. exceptions are re-thrown, but the
	 * retry policy is applied with the same policy to subsequent invocations with the
	 * same arguments. If false then retryable exceptions are not re-thrown.
	 * @return true if retry is stateful, default false
	 */
	boolean stateful() default false;

	/**
	 * @return the maximum number of attempts (including the first failure), defaults to 3
	 */
	int maxAttempts() default 3;

	/**
	 * @return an expression evaluated to the maximum number of attempts (including the first failure), defaults to 3
	 * Overrides {@link #maxAttempts()}.
	 * @since 1.2
	 */
	String maxAttemptsExpression() default "";

	/**
	 * Specify the backoff properties for retrying this operation. The default is a
	 * simple {@link Backoff} specification with no properties - see it's documentation
	 * for defaults.
	 * @return a backoff specification
	 */
	Backoff backoff() default @Backoff();

	/**
	 * Specify an expression to be evaluated after the {@code SimpleRetryPolicy.canRetry()}
	 * returns true - can be used to conditionally suppress the retry. Only invoked after
	 * an exception is thrown. The root object for the evaluation is the last {@code Throwable}.
	 * Other beans in the context can be referenced.
	 * For example:
	 * <pre class=code>
	 *  {@code "message.contains('you can retry this')"}.
	 * </pre>
	 * and
	 * <pre class=code>
	 *  {@code "@someBean.shouldRetry(#root)"}.
	 * </pre>
	 * @return the expression.
	 * @since 1.2
	 */
	String exceptionExpression() default "";

}
```

















> *大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某*

