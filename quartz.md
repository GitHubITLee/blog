## Quartz Properties  加载问题

由于项目历史原因需要调整线程池数量，各种百度、google无果（无法从根本上说明白问题）
后查看源码最终发现问题。



###  StdSchedulerFactory

StdSchedulerFactory实现了SchedulerFactory 完成了QuartzScheduler实例的创建及属性文件的加载。

```
String schedName = cfg.getStringProperty(PROP_SCHED_INSTANCE_NAME,
                "QuartzScheduler");
```

接下来我们重点说说properties加载的问题，看到好多小伙伴说不管把配置放在任何位置都无法准确加
载到配置文件，接下来我们就来趴一趴源码看看Quartz是怎么加载配置的。

```
public void initialize() throws SchedulerException {
 // short-circuit if already initialized
 if (cfg != null) return;
        if (initException != null) throw initException;

        String requestedFile = System.getProperty(PROPERTIES_FILE);
        String propFileName = requestedFile != null ? requestedFile
                : "quartz.properties";
        File propFile = new File(propFileName);
```

Quartz首先会检查PropertiesParser是否已经加载，很多小伙伴说的Spring必须这样配置才会生效
原因就在这里：

```
<bean id="schedulerFactory" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="configLocation" value="classpath:config/quartz.properties"></property>
</bean>
```

Quartz接下来会检查环境变量org.quartz.properties是否存在，如果找不到会默认加载jar包中的配置，
这就是很多小伙伴怎么配置都不生效也不报错的原因：

```
String requestedFile = System.getProperty(PROPERTIES_FILE);
        String propFileName = requestedFile != null ? requestedFile
                : "quartz.properties";
```

最后在不使用Spring的环境下怎么顺利的加载自定义配置呢？我们接着往下看：

```
File propFile = new File(propFileName);
```
是的没错Quartz确实是用JDK的File加载配置的，所有我们只需要把配置文件放在项目的根目录下就OK了，
不管是普通的项目还是Maven构建的项目都是一样的。

自此很多的小伙伴可能会有这样的疑问如果把配置文件放在项目根目录会不会不太优雅，而且从目前
项目的结构上来看基本都是通过Resource目录去读取配置的。所以在不破坏源代码的前提下做了测试：

```
String requestedFile = System.getProperty("org.quartz.properties");
String propFileName = requestedFile != null ? requestedFile : 
    quartzTest.class.getResource("/config/quartz.properties").getFile();
File propFile = new File(propFileName);
System.out.println(propFile.exists());
System.out.printf("Canonical Path: %s", propFile.getCanonicalPath());
```
通过类加载路径寻找配置文件的file path得出的最终答案如下：

```dtd
true
Canonical Path: D:\quartzTest\target\test-classes\config\quartz.properties
```


