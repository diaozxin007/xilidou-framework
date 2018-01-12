Spring 作为 J2ee 开发事实上的标准，是每个Java开发人员都需要了解的框架。但是Spring 的 IoC 和 Aop 的特性，对于初级的Java开发人员来说还是比较难于理解的。所以我就想写一系列的文章给大家讲解这些特性。从而能够进一步深入了解 Spring 框架。

读完这篇文章，你将会了解：

* 什么是依赖注入和控制反转
* Ioc有什么用
* Spring的 Ioc 是怎么实现的
* 按照Spring的思路开发一个简单的Ioc框架

## IoC 是什么？

wiki百科的解释是：

>控制反转（Inversion of Control，缩写为IoC），是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度。其中最常见的方式叫做依赖注入（Dependency Injection，简称DI）。通过控制反转，对象在被创建的时候，由一个调控系统内所有对象的外界实体，将其所依赖的对象的引用传递给它。也可以说，依赖被注入到对象中。

## Ioc 有什么用？

看完上面的解释你一定没有理解什么是 Ioc，因为是第一次看见上面的话也觉得云里雾里。

不过通过上面的描述我们可以大概的了解到，使用IoC的目的是为了解耦。也就是说IoC 是解耦的一种方法。

我们知道Java 是一门面向对象的语言，在 Java 中 Everything is Object，我们的程序就是由若干对象组成的。当我们的项目越来越大，合作的开发者越来越多的时候，我们的类就会越来越多，类与类之间的引用就会成指数级的增长。如下图所示：

![](https://ws4.sinaimg.cn/large/006tNc79ly1fnd1qwnp8qj30aa06ot8t.jpg)

这样的工程简直就是灾难，如果我们引入 Ioc 框架。由框架来维护类的生命周期和类之间的应用。我们的系统就会变成这样：

![](https://ws4.sinaimg.cn/large/006tKfTcly1fnd1yq1vzpj30aa06v0sr.jpg)

这个时候我们发现，我们类之间的关系都由 IoC 框架负责维护类的生命周期，同时将类注入到需要的类中。也就是类的使用者只负责使用，而不负责维护。把专业的事情交给专业的框架来完成。大大的减少开发的复杂度。

用一个类比来理解这个问题。Ioc 框架就是我们生活中的房屋中介，首先中介会收集市场上的房源，分别和各个房源的房东建立联系。当我们需要租房的时候，并不需要我们四处寻找各类租房信息。我们直接找房屋中介，中介就会根据你的需求提供相应的房屋信息。大大提升了租房的效率，减少了你与各类房东之间的沟通次数。

## spirng 的 IOC 是怎么实现的

了解Spring框架最直接的方法就阅读Spring的源码。但是Spring的代码抽象的层次很高，且处理的细节很高。对于大多数人来说不是太容易理解。我读了Spirng的源码以后以我的理解做一个总结,Spirng IoC 主要是以下几个步骤。

1. 初始化 IoC 容器。
2. 读取配置文件。
3. 将配置文件转换为容器识别对的数据结构（这个数据结构在Spring中叫做 BeanDefinition） 
4. 利用数据结构依次实例化相应的对象
5. 注入对象之间的依赖关系



## 自己实现一个 IoC 框架

为了方便，我们参考 Spirng 的 IoC 实现，去除所有与核心原理无关的逻辑。极简的实现 IoC 的框架。 项目使用 json 作为配置文件。使用 maven 管理 jar 包的依赖。

在这个框架中我们的对象都是单例的，并不支持Spirng的多种作用域。框架的实现使用了cglib 和 Java 的反射。项目中我还使用了 lombok 用来简化代码。

下面我们就来编写 IoC 框架吧。

首先我们看看这个框架的基本结构：

![](https://ws1.sinaimg.cn/large/006tKfTcly1fndrb3d0ofj308107cmxf.jpg)

从宏观上观察一下这个框架，包含了2个package、在包 bean 中定义了我们框架的数据结构。core 是我们框架的核心逻辑所在。utils 是一些通用工具类。接下来我们就逐一讲解一下：

### 1. bean 定义了框架的数据结构

`BeanDefinition` 是我们项目的核心数据结构。用于描述我们需要 IoC 框架管理的对象。

```java
@Data
@ToString
public class BeanDefinition {

    private String name;

    private String className;

    private String interfaceName;

    private List<ConstructorArg> constructorArgs;

    private List<PropertyArg> propertyArgs;

}
```
包含了对象的 name，class的名称。如果是接口的实现，还有该对象实现的接口。以及构造函数的传参的列表 `constructorArgs` 和需要注入的参数列表 `propertyArgList`。

### 2. 再看看我们的工具类包里面的对象：

`ClassUtils` 负责处理 Java 类的加载,代码如下：
```java
public class ClassUtils {
    public static ClassLoader getDefultClassLoader(){
        return Thread.currentThread().getContextClassLoader();
    }
    public static Class loadClass(String className){
        try {
            return getDefultClassLoader().loadClass(className);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```
我们只写了一个方法，就是通过 className 这个参数获取对象的 Class。

`BeanUtils` 负责处理对象的实例化，这里我们使用了 cglib 这个工具包，代码如下：

```java
public class BeanUtils {
    public static <T> T instanceByCglib(Class<T> clz,Constructor ctr,Object[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clz);
        enhancer.setCallback(NoOp.INSTANCE);
        if(ctr == null){
            return (T) enhancer.create();
        }else {
            return (T) enhancer.create(ctr.getParameterTypes(),args);
        }
    }
}
```

`ReflectionUtils` 主要通过 Java 的反射原理来完成对象的依赖注入：

```java
public class ReflectionUtils {

    public static void injectField(Field field,Object obj,Object value) throws IllegalAccessException {
        if(field != null) {
            field.setAccessible(true);
            field.set(obj, value);
        }
    }
}
```

`injectField(Field field,Object obj,Object value)` 这个方法的作用就是，设置 obj 的 field 为 value。

`JsonUtils` 的作用就是为了解析我们的json配置文件。代码比较长，与我们的 IoC 原理关系不大，感兴趣的同学可以自行从github上下载代码看看。

有了这几个趁手的工具，我们就可以开始完成 Ioc 框架的核心代码了。

### 3. 核心逻辑

我的 IoC 框架，目前只支持一种 ByName 的注入。所以我们的 BeanFactory 就只有一个方法：

```java
public interface BeanFactory {
    Object getBean(String name) throws Exception;
}
```

然后我们实现了这个方法：
```java
public class BeanFactoryImpl implements BeanFactory{

    private static final ConcurrentHashMap<String,Object> beanMap = new ConcurrentHashMap<>();

    private static final ConcurrentHashMap<String,BeanDefinition> beanDefineMap= new ConcurrentHashMap<>();

    private static final Set<String> beanNameSet = Collections.synchronizedSet(new HashSet<>());

    @Override
    public Object getBean(String name) throws Exception {
        //查找对象是否已经实例化过
        Object bean = beanMap.get(name);
        if(bean != null){
            return bean;
        }
        //如果没有实例化，那就需要调用createBean来创建对象
        bean =  createBean(beanDefineMap.get(name));
        
        if(bean != null) {

            //对象创建成功以后，注入对象需要的参数
            populatebean(bean);
            
            //再吧对象存入Map中方便下次使用。
            beanMap.put(name,bean;
        }

        //结束返回
        return bean;
    }

    protected void registerBean(String name, BeanDefinition bd){
        beanDefineMap.put(name,bd);
        beanNameSet.add(name);
    }

    private Object createBean(BeanDefinition beanDefinition) throws Exception {
        String beanName = beanDefinition.getClassName();
        Class clz = ClassUtils.loadClass(beanName);
        if(clz == null) {
            throw new Exception("can not find bean by beanName");
        }
        List<ConstructorArg> constructorArgs = beanDefinition.getConstructorArgs();
        if(constructorArgs != null && !constructorArgs.isEmpty()){
            List<Object> objects = new ArrayList<>();
            for (ConstructorArg constructorArg : constructorArgs) {
                objects.add(getBean(constructorArg.getRef()));
            }
            return BeanUtils.instanceByCglib(clz,clz.getConstructor(),objects.toArray());
        }else {
            return BeanUtils.instanceByCglib(clz,null,null);
        }
    }

    private void populatebean(Object bean) throws Exception {
        Field[] fields = bean.getClass().getSuperclass().getDeclaredFields();
        if (fields != null && fields.length > 0) {
            for (Field field : fields) {
                String beanName = field.getName();
                beanName = StringUtils.uncapitalize(beanName);
                if (beanNameSet.contains(field.getName())) {
                    Object fieldBean = getBean(beanName);
                    if (fieldBean != null) {
                        ReflectionUtils.injectField(field,bean,fieldBean);
                    }
                }
            }
        }
    }
}
```

首先我们看到在 BeanFactory 的实现中。我们有两 HashMap，beanMap 和 beanDefineMap。 beanDefineMap 存储的是对象的名称和对象对应的数据结构的映射。

容器初始化的时候，会调用 `BeanFactoryImpl.registerBean` 方法。把 对象的 BeanDefination 数据结构，存储起来。

当我们调用 getBean() 的方法的时候。会先到 beanMap 里面查找，有没有实例化好的对象。如果没有，就会qubeanDefineMap查找这个对象对应的 BeanDefination。再利用DeanDefination去实例化一个对象。

对象实例化成功以后，我们还需要注入相应的参数，调用 `populatebean()`这个方法。在 populateBean 这个方法中，会扫描对象里面的Field，如果对象中的 Field 是我们IoC容器管理的对象，那就会调用 我们上文实现的 `ReflectionUtils.injectField`来注入对象。

一切准备妥当之后，我们对象就完成了整个 IoC 流程。最后这个对象放入 beanMap 中,方便下一次使用。

所以我们可以知道 BeanFactory 是管理和生成对象的地方。

### 4. 容器

我们所谓的容器，就是对BeanFactory的扩展，负责管理 BeanFactory。我们的这个IoC 框架使用 Json 作为配置文件，所以我们容器就命名为 JsonApplicationContext。当然之后你愿意实现 XML 作为配置文件的容器你就可以自己写一个 XmlApplicationContext，如果基于注解的容器就可以叫AnnotationApplcationContext。这些实现留个大家去完成。

我们看看 ApplicationContext 的代码：

```java
public class JsonApplicationContext extends BeanFactoryImpl{
    private String fileName;
    public JsonApplicationContext(String fileName) {
        this.fileName = fileName;
    }
    public void init(){
        loadFile();
    }
    private void loadFile(){
        InputStream is = Thread.currentThread().getContextClassLoader().getResourceAsStream(fileName);
        List<BeanDefinition> beanDefinitions = JsonUtils.readValue(is,new TypeReference<List<BeanDefinition>>(){});
        if(beanDefinitions != null && !beanDefinitions.isEmpty()) {
            for (BeanDefinition beanDefinition : beanDefinitions) {
                registerBean(beanDefinition.getName(), beanDefinition);
            }
        }
    }
}
```

这个容器的作用就是 读取配置文件。将配置文件转换为容器能够理解的 `BeanDefination`。然后使用 `registerBean` 方法。注册这个对象。

至此，一个简单版的 IoC 框架就完成。

### 5. 框架的使用
我们写一个测试类来看看我们这个框架怎么使用：

首先我们有三个对象

```java
public class Hand {
    public void waveHand(){
        System.out.println("挥一挥手");
    }
}

public class Mouth {
    public void speak(){
        System.out.println("say hello world");
    }
}

public class Robot {
    //需要注入 hand 和 mouth 
    private Hand hand;
    private Mouth mouth;

    public void show(){
        hand.waveHand();
        mouth.speak();
    }
}
```

我们需要为我们的 Robot 机器人注入 hand 和 mouth。

配置文件：
```json
[
  {
    "name":"robot",
    "className":"com.xilidou.framework.ioc.entity.Robot"
  },
  {
    "name":"hand",
    "className":"com.xilidou.framework.ioc.entity.Hand"
  },
  {
    "name":"mouth",
    "className":"com.xilidou.framework.ioc.entity.Mouth"
  }
]
```

这个时候写一个测试类：

```java
public class Test {
    public static void main(String[] args) throws Exception {
        JsonApplicationContext applicationContext = new JsonApplicationContext("application.json");
        applicationContext.init();
        Robot aiRobot = (Robot) applicationContext.getBean("robot");
        aiRobot.show();
    }
}
```
运行以后输出：
```shell

挥一挥手
say hello world

Process finished with exit code 0
```

可以看到我们成功的给我的 aiRobot 注入了 hand 和 mouth。

至此我们 Ioc 框架开发完成。

## 总结

这篇文章读完以后相信你一定也实现了一个简单的 IoC 框架。

虽然说阅读源码是了解框架的最终手段。但是 Spring 框架作为一个生产框架，为了保证通用和稳定，源码必定是高度抽象，且处理大量细节。所以 Spring 的源码阅读起来还是相当困难。希望这篇文章能够帮助理解 Spring Ioc 的实现。

下一篇文章 应该会是 《徒手撸框架--实现AOP》。

github 地址：

