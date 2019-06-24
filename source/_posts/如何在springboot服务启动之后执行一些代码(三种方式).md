---
title: 如何在springboot服务启动之后执行一些代码(三种方式)
date: 2019-04-19 17:01:21
categories: [springboot]
tags: [spring,springboot]
---

### 前言
  通常的我们的项目开发中，经常会遇到那种在服务一启动就需要自动执行一些业务代码的情况。比如将数据库中的配置信息或者数据字典之类的缓存到redis，或者在服务启动的时候将一些配置化的定时任务开起来。关于spring mvc或者springboot如何在项目启动的时候就执行一些代码，方法其实有很多，我这边介绍一下我使用过的三种。

  <!--more-->
  
#### 1、`@PostConstruct` 注解

从Java EE5规范开始，Servlet中增加了两个影响Servlet生命周期的注解，`@PostConstruct`和`@PreDestroy`，这两个注解被用来修饰一个非静态的void（）方法。`@PostConstruct`会在所在类的构造函数执行之后执行，在init()方法执行之前执行。(`@PreDestroy`注解的方法会在这个类的destory()方法执行之后执行。)
- 使用示例：在Spring容器加载之后，我需要启动定时任务去做任务的处理（我的定时任务采用的是读取数据库配置的方式）。在这里我使用`@PostConstruct` 指定了需要启动的方法。
```java
@Component // 注意 这里必须有
public class StartAllJobInit {

    protected Logger logger = LoggerFactory.getLogger(getClass().getName());
    @Autowired
    JobInfoService jobInfoService;

    @Autowired
    JobTaskUtil jobTaskUtil;

    @PostConstruct // 构造函数之后执行
    public void init(){
        System.out.println("容器启动后执行");
        startJob();
    }

    public void startJob() {
        List<JobInfoBO> list = jobInfoService.findList();
        for (JobInfoBO jobinfo :list) {
            try {
                if("0".equals(jobinfo.getStartWithrun())){
                    logger.info("任务{}未设置自动启动。", jobinfo.getJobName());
                    jobInfoService.updateJobStatus(jobinfo.getId(), BasicsConstantManual.BASICS_SYS_JOB_STATUS_STOP);
                }else{
                    logger.info("任务{}设置了自动启动。", jobinfo.getJobName());
                    jobTaskUtil.addOrUpdateJob(jobinfo);
                    jobInfoService.updateJobStatus(jobinfo.getId(), BasicsConstantManual.BASICS_SYS_JOB_STATUS_STARTING);
                }
            } catch (SchedulerException e) {
                logger.error("执行定时任务出错，任务名称 {} ", jobinfo.getJobName());
            }
        }
    }
}
```

#### 2、实现`CommandLineRunner`接口并重写run()方法
`CommandLineRunner`接口文档描述如下：
```java
/**
 * Interface used to indicate that a bean should <em>run</em> when it is contained within
 * a {@link SpringApplication}. Multiple {@link CommandLineRunner} beans can be defined
 * within the same application context and can be ordered using the {@link Ordered}
 * interface or {@link Order @Order} annotation.
 * <p>
 * If you need access to {@link ApplicationArguments} instead of the raw String array
 * consider using {@link ApplicationRunner}.
 *
 * @author Dave Syer
 * @see ApplicationRunner
 */
public interface CommandLineRunner {

	/**
	 * Callback used to run the bean.
	 * @param args incoming main method arguments
	 * @throws Exception on error
	 */
	void run(String... args) throws Exception;

}
```
如上所说：接口被用作加入Spring容器中时执行run(String... args)方法，通过命令行传递参数。SpringBoot在项目启动后会遍历所有实现CommandLineRunner的实体类并执行run方法，多个实现类可以并存并且根据order注解排序顺序执行。这边还有个`ApplicationRunner`接口，但是接收参数是使用的`ApplicationArguments`。这边不再赘述。

**同样是启动时执行定时任务，使用这种方式我的写法如下：**
```java
@Component // 注意 这里必须有
//@Order(2) 如果有多个类需要启动后执行 order注解中的值为启动的顺序
public class StartAllJobInit implements CommandLineRunner {

    protected Logger logger = LoggerFactory.getLogger(getClass().getName());
    @Autowired
    JobInfoService jobInfoService;

    @Autowired
    JobTaskUtil jobTaskUtil;

    @Override
    public void run(String... args) {
        List<JobInfoBO> list = jobInfoService.findList();
        for (JobInfoBO jobinfo :list) {
            try {
                if("0".equals(jobinfo.getStartWithrun())){
                    logger.info("任务{}未设置自动启动。", jobinfo.getJobName());
                    jobInfoService.updateJobStatus(jobinfo.getId(), BasicsConstantManual.BASICS_SYS_JOB_STATUS_STOP);
                }else{
                    logger.info("任务{}设置了自动启动。", jobinfo.getJobName());
                    jobTaskUtil.addOrUpdateJob(jobinfo);
                    jobInfoService.updateJobStatus(jobinfo.getId(), BasicsConstantManual.BASICS_SYS_JOB_STATUS_STARTING);
                }
            } catch (SchedulerException e) {
                logger.error("执行定时任务出错，任务名称 {} ", jobinfo.getJobName());
            }
        }
    }
}
```
#### 3、使用`ContextRefreshedEvent`事件(上下文件刷新事件)

> ContextRefreshedEvent 官方在接口上的doc说明<br>
> Event raised when an {@code ApplicationContext} gets initialized or refreshed.

ContextRefreshedEvent是Spring的ApplicationContextEvent一个实现，ContextRefreshedEvent 事件会在Spring容器初始化完成后以及刷新时触发。

**在这里我需要在springboot程序启动之后加载配置信息和字典信息到Redis缓存中去，我可以这样写：**

```java
@Component // 注意 这个也是必须有的注解 三种都需要 使spring扫描到这个类并交给它管理
public class InitRedisCache implements ApplicationListener<ContextRefreshedEvent> {
    static final Logger logger = LoggerFactory
            .getLogger(InitRedisCache.class);

    @Autowired
    private SysConfigService sysConfigService;

    @Autowired
    private SysDictService sysDictService;


    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        logger.info("-------加载配置信息 start-------");
        sysConfigService.loadConfigIntoRedis();
        logger.info("-------加载配置信息 end-------");

        logger.info("-------加载字典信息 start-------");
        sysDictService.loadDictIntoRedis();
        logger.info("-------加载字典信息 end-------");
    }
}
```java
**注意**：这种方式在springmvc-spring的项目中使用的时候会出现执行两次的情况。这种是因为在加载spring和springmvc的时候会创建两个容器，都会触发这个事件的执行。这时候只需要在`onApplicationEvent`方法中判断是否有父容器即可。
```
@Override  
  public void onApplicationEvent(ContextRefreshedEvent event) {  
      if(event.getApplicationContext().getParent() == null){//root application context 没有parent，他就是老大.  
           //需要执行的逻辑代码，当spring容器初始化完成后就会执行该方法。  
      }  
  }  
```
#### 总结
以上，就是我在实际开发中常用的三种，在项目启动时执行代码的方式，开发者可以根据不同的使用情况选择合适的方法去执行自己的业务逻辑。
