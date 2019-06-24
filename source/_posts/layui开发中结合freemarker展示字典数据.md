---
title: layui结合freemarker+springboot进行前端数据表格字典转义
date: 2019-03-16 19:51:53
categories: [springboot]
tags: [开发日记,springboot,freemarker,工厂模式,layui]
---

### 前言
在layui的开发中，我们经常会用到表格数据展示。但是在数据库中我们通常保存的一些状态等数据，都是枚举值，而我们在前端展示的时候就不能使用这些枚举值了，而要展示枚举值对应的意义数据。比如状态status，1=启用，0=停用。
那么在layui的数据表格中，我们要展示这样的数据，写法可以是：
```javascript
templet: function (d) {
  if(d.status === 1){
      return "启用";
  } else if(d.status === 0){
      return "停用";
  }
}
```
但是这样的写法很low啊，我们在数据库中定义多少枚举值在这里就要写多少代码，一旦有重复使用的情况，这种写法会让我们痛不欲生。
在考虑到后台使用freemarker的情况下，配置freemarker自定义标签就能很好地解决这个问题。
最后我们的写法就会简化成：`<@th type="template" nid="basics_sys_status" objName="status"></@th>`，接下来看看如何在springboot中配置使用吧。
<!-- more -->
### freemarker自定义标签介绍及使用
`TemplateDirectiveModel`接口是freemarker自定标签或者自定义指令的核心处理接口。当模板页面遇到用户自定义的标签指令时，`execute()`方法会被执行。`execute()`方法如下。
```java
public void execute(
  Environment env, Map params, TemplateModel[] loopVars, TemplateDirectiveBody body
) throws TemplateException, IOException;
```
我们在使用freemarker自定义标签的时候需要实现该接口并且重写execute方法。
#### `execute()`方法参数解释
- *Environment env*：系统环境变量，通常用它来输出相关内容，如`Writer out = env.getOut();`
- *Map params*：自定义标签传过来的对象，就是从页面上获取的参数，其key=自定义标签的参数名，value值是TemplateModel类型，而TemplateModel是一个接口类型，通常我们都使用TemplateScalarModel接口来替代它获取一个String 值，如TemplateScalarModel.getAsString();当然还有其它常用的替代接口，如TemplateNumberModel获取number，TemplateHashModel等。
  在本例使用时，我们会将map转成我们自己的对象进行数据保存。
- *TemplateModel[] loopVars*：循环替代变量
- *TemplateDirectiveBody body*：标签中嵌套的内容，如`<@tag>body</@tag>`，就是这个body

#### 开始使用

###### 定义接收页面参数的对象
```java
@Getter
@Setter
public class TableThTag {
    /**
     * 对象属性名【需要进行对象属性获取】
     */
    private String objName;
    /**
     * 字典标识
     */
    private String nid;

    /**
     * 类型
     */
    private String type;
}
```
###### 实现`TemplateDirectiveModel`接口并重写`execute`方法
```java
@Component
@org.springframework.context.annotation.Configuration
public class TableThDirective implements TemplateDirectiveModel {

    Logger logger = LoggerFactory.getLogger(getClass().getName());

    /**
     * FreeMarker自定义指令
     */
    @Override
    public void execute(Environment environment, Map map, TemplateModel[] templateModels,
                        TemplateDirectiveBody templateDirectiveBody) throws TemplateException, IOException {
        TableThTag tableThTag = new TableThTag();
        //校验参数
        try {
            //  用来将一些 key-value 的值（例如 hashmap）映射到 bean 中的属性
            BeanUtils.populate(tableThTag, map);
            if (StringUtils.isEmpty(tableThTag.getNid()) || StringUtils.isEmpty(tableThTag.getType())) {
                throw new IllegalArgumentException("nid,type不能为空");
            }
        } catch (Exception e) {
            logger.error("数据转化异常", e);
        }
        StringBuilder html = new StringBuilder();
        // 根据类型创建不同的HTML生成器
        ThFormatterInterface thFormatterInterface = ThFormatterFactory.createThFormatter(tableThTag.getType());
        if (thFormatterInterface != null) {
            String dictHtml = thFormatterInterface.buildFormatterHtml(tableThTag.getNid(), tableThTag.getFieldName());
            html.append(dictHtml);
        }
        // 执行真正指令的执行部分:
        Writer out = environment.getOut();
        out.write(html.toString());
        if (templateDirectiveBody != null) {
            templateDirectiveBody.render(environment.getOut());
        }

    }

    public static BeansWrapper getBeansWrapper() {
        BeansWrapper beansWrapper =
                new BeansWrapperBuilder(Configuration.VERSION_2_3_21).build();
        return beansWrapper;
    }

}
```
大家可以看到，在这个方法中，我将页面上的参数转为`TableThTag `对象。然后再根据前端页面不同的type类型对应
实现了`ThFormatterInterface `的工厂对象，创建不同的html生成器。（这里考虑到扩展性，可能以后不光创建数据表格会用的到，比如下拉框什么的，也可以使用这种方式创建，所以在这里使用抽象工厂依据类型动态创建。）
下面就是创建html的具体工厂以及实现方法。
###### `ThFormatterInterface `抽象工厂创建html生成器
接口
```java
public interface ThFormatterInterface {
    /**
     * 构造生成枚举html
     * @param nid
     * @return
     */
    String buildFormatterHtml(String nid, String fieldName);
}
```
工厂
```java
public class ThFormatterFactory {

    private static Logger logger = LoggerFactory.getLogger(ThFormatterFactory.class);

    public static ThFormatterInterface createThFormatter(String type){
        if(StringUtils.isEmpty(type)){
            return  new ThFormatterTemplate();
        }
        // 文件名 如果type传template 就需要有一个名为ThFormatterTemplate的文件
        // 并且实现了ThFormatterInterface以及重写生成html的方法
        String fileName = "ThFormatter" + StringUtil.firstCharUpperCase(type);
        //类路径 通过反射去创建实现类
        String className = "com.module.freemarker.impl."+fileName;
        //生成表头格式实现类
        ThFormatterInterface thFormatterInterface = null;
        try {
            thFormatterInterface =(ThFormatterInterface) Class.forName(className).newInstance();
        } catch (Exception e) {
            logger.error(e.getMessage(),e);
        }
        return thFormatterInterface;
    }
}
```
实现类
```java
public class ThFormatterTemplate implements ThFormatterInterface {

    @Override
    public String buildFormatterHtml(String nid, String fieldName) {
        Assert.notEMPTY(nid, "nid不能为空");
        Assert.notEMPTY(fieldName, "objName不能为空");
        SysDictService sysDictService = SpringContextHolder.getBean(SysDictService.class);
        // 通过nid查询字典类 这里不需要进行照抄 每个人都会有自己的实现方法
        List<SysDictBO> sysDictModelList = sysDictService.findByPartnerNid(nid);
        StringBuilder dictHtml = new StringBuilder();
        // 反正目的就是根据字典类生成对应的html就行了 需要生成的格式对照template原本应该有的写法就行了
        dictHtml.append("templet: function(d){ ");
        for (SysDictBO sysDict : sysDictModelList) {
            dictHtml.append("if(d." + fieldName + " == '" + sysDict.getValue() + "'){ return '" + sysDict.getName() + "';}");
        }
        dictHtml.append("}");

        return dictHtml.toString();
    }
}
```
###### 最后将自定义标签注入到freemarker标签中去
```java
@org.springframework.context.annotation.Configuration
public class FreemarkerConfig {

    @Resource
    private Configuration configuration;
    @Resource
    private TableThDirective tableThDirective;

    @PostConstruct
    public void setSharedVariable(){
        configuration.setSharedVariable("th",tableThDirective);
        configuration.setSharedVariable("shiro",new ShiroTags());
    }

}
```
---
这样就大功告成了。

在页面上进行使用吧：`<@th type="template" nid="basics_sys_status" objName="status"></@th>`

以后进行扩展什么的也方便，比如生成下拉框：`<@th type="select" nid="basics_sys_status" objName="status"></@th>`这样然后自动生成html的时候查出来所有的字典，根据类型生成多个<option>出来就行了。
