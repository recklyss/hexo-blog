---
title: 使用springboot进行国际化时自定义读取数据库配置
date: 2019-03-20 19:27:02
categories: [国际化,框架相关]
tags: [springboot,国际化]
---
## 前言
springboot默认就支持国际化的，而且不需要你过多的做什么配置，只需要在`resources/`下创建国际化配置文件即可，注意名称必须以messages开始。 messages.properties （默认的语言配置文件，当找不到其他语言的配置的时候，使用该文件进行展示）。 具体的关于springboot的国际化配置我这边就不再过多介绍(包括Locale的设置以及如何根据区域设置语言等)，关于页面上得使用可以参考：[springboot国际化](!https://www.baidu.com/s?word=springboot+%E5%9B%BD%E9%99%85%E5%8C%96)。在这篇博客中，我要介绍的是一个很有用的功能并且绝大部分人也会用得到，就是
<strong><font color=#0099ff size=5 face="黑体">不使用配置文件`messages.properties`储存国际化语言，而使用数据库进行动态配置，做到无需重启更改配置。</font></strong>
<!-- more -->
## 如何使用
#### MessageSource介绍
Spring提供了一个接口MessageSource用于获取国际化信息，ReloadableResourceBundleMessageSource和ResourceBundleMessageSource都是继承了该接口的一个抽象实现类AbstractMessageSource，在spring官网有一段这样介绍messageSource的话：
![spring官网对于messageSource的介绍](https://img-blog.csdn.net/20180116154941287?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxNDcyMTEzMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "spring官网对于messageSource的介绍")
图中红框画起来的意思就是，上下文加载的时候会查询messageSource的bean，如果没有就会创建一个名为`messageSource`放在上下文中... ...等等。
#### 在springboot中注入自定义messageSource
通过上面的介绍，我们就可以自己定义自己的messageSource进行配置的读取了。
 ** 我这边是把这个放在了业务层，大家用的时候也可以直接放在控制层(一般都放在控制层，要用到)，使用@Compnent("messageSource")注解声明下bean名称即可 **
 ```
 // MyMessageSourceService是我自己的接口 你也可以不需要。使用@Compnent("messageSource")注解就行
 @Service("messageSource")
 public class MyMessageSource extends AbstractMessageSource implements ResourceLoaderAware, MyMessageSourceService {

     ResourceLoader resourceLoader;

     // 这个是用来缓存数据库中获取到的配置的 数据库配置更改的时候可以调用reload方法重新加载
     // 当然 实际使用者也可以不使用这种缓存的方式
     private static final Map<String, Map<String, String>> LOCAL_CACHE = new ConcurrentHashMap<>(256);

     @Autowired
     SysI18nService sysI18nService;

     private final Logger logger = LoggerFactory.getLogger(MyMessageSource.class);

     /**
      * 初始化
      */
     @PostConstruct
     public void init() {
         this.reload();
     }

     /**
      * 重新将数据库中的国际化配置加载
      */
     public void reload() {
         LOCAL_CACHE.clear();
         LOCAL_CACHE.putAll(loadAllMessageResourcesFromDB());
     }

     /**
      * 从数据库中获取所有国际化配置 这边可以根据自己数据库表结构进行相应的业务实现
      * 对应的语言能够取出来对应的值就行了 无需一定要按照这个方法来
      */
     public Map<String, Map<String, String>> loadAllMessageResourcesFromDB() {
         List<SysI18nBO> list = sysI18nService.findList(new SysI18nAO());
         if (CollectionUtils.isNotEmpty(list)) {
             final Map<String, String> zhCnMessageResources = new HashMap<>(list.size());
             final Map<String, String> enUsMessageResources = new HashMap<>(list.size());
             final Map<String, String> idIdMessageResources = new HashMap<>(list.size());
             for (SysI18nBO bo : list) {
                 String name = bo.getModel() + "." + bo.getName();
                 String zhText = bo.getZhCn();
                 String enText = bo.getEnUs();
                 String idText = bo.getInId();
                 zhCnMessageResources.put(name, zhText);
                 enUsMessageResources.put(name, enText);
                 idIdMessageResources.put(name, idText);
             }
             LOCAL_CACHE.put("zh", zhCnMessageResources);
             LOCAL_CACHE.put("en", enUsMessageResources);
             LOCAL_CACHE.put("in", idIdMessageResources);
         }
         return MapUtils.EMPTY_MAP;
     }

     /**
      * 从缓存中取出国际化配置对应的数据 或者从父级获取
      *
      * @param code
      * @param locale
      * @return
      */
     public String getSourceFromCache(String code, Locale locale) {
         String language = locale.getLanguage();
         Map<String, String> props = LOCAL_CACHE.get(language);
         if (null != props && props.containsKey(code)) {
             return props.get(code);
         } else {
             try {
                 if (null != this.getParentMessageSource()) {
                     return this.getParentMessageSource().getMessage(code, null, locale);
                 }
             } catch (Exception ex) {
                 logger.error(ex.getMessage(), ex);
             }
             return code;
         }
     }

     // 下面三个重写的方法是比较重要的
     @Override
     public void setResourceLoader(ResourceLoader resourceLoader) {
         this.resourceLoader = (resourceLoader == null ? new DefaultResourceLoader() : resourceLoader);
     }

     @Override
     protected MessageFormat resolveCode(String code, Locale locale) {
         String msg = getSourceFromCache(code, locale);
         MessageFormat messageFormat = new MessageFormat(msg, locale);
         return messageFormat;
     }

     @Override
     protected String resolveCodeWithoutArguments(String code, Locale locale) {
         return getSourceFromCache(code, locale);
     }
 }
 ```
#### 最后
 至此，自定义国际化配置读取数据库已经完成，只需要在更新数据库配置的时候调用一下reload重置一下缓存中的信息即可。
 > [参考博客：spring xml配置自定义读取数据库的messageSource](!https://blog.csdn.net/u014721131/article/details/79075802)
