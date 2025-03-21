---
layout: post
title: 自动注入接口实现
date: 2025-3-20
tags: java实践
---

# 前言
在面向对象设计与开发过程中，策略模式、模板方法模式、命令模式、责任链模式等设计模式常面临一个共性挑战：当需要横向扩展接口实现时，开发者往往需要在编写新实现类的同时，手动将其注入到工厂类或执行器类的选择逻辑中。这种"新增实现类+修改选择器"的双步操作不仅违背了开闭原则，更埋下了维护隐患——开发者可能因疏忽导致扩展失效，或在多人协作时出现代码冲突。以下是考虑过或曾使用的方案：
1. Spring的List注入机制：通过@Autowired批量注入List<接口>虽能实现自动装配，但强制要求所有实现类必须注册为Spring Bean，在非Spring体系或轻量级工具类场景下难以适用；
2. Java SPI机制：通过META-INF/services配置文件声明实现类，虽解耦了接口与实现，但每次扩展仍需修改配置文件
3. 传统工厂模式：显式维护实现类注册表的方式，在频繁扩展时会导致工厂类持续膨胀，维护成本陡增

现在发现基于`org.reflections`类库的包路径扫描能力，可以完美实现我的需求。开发者只需专注实现业务逻辑，系统即可自动发现并加载所有符合规则的实现类。该方案具有三大核心优势：
1. 零配置扩展：新实现类无需任何注册操作，类加载阶段自动完成发现（如扫描com.example.handler包下所有CommandHandler子类）；
2. 跨框架通用：不依赖Spring等容器，可在纯Java环境中实现动态装配；
3. 自定义筛选机制：支持通过自定义类注解、继承关系、接口方法等，为复杂业务场景提供灵活装配策略。

这种设计将扩展成本降至最低——开发者只需遵循接口契约编写新类，系统自动完成实现类的发现与注入，真正实现了"扩展开放，修改闭合"的理想架构。下文将通过具体代码案例，具体解释如何借助反射扫描技术构建自洽的扩展体系

# 代码实现
## 核心代码-实现包扫描、类加载
```java
public class ReflectionUtils {

    public static <T> T newInstance(Class<T> clazz) {
        try {
            return clazz.newInstance();
        } catch (Exception e) {
            throw new RuntimeException("反射生成对象异常，class:" + clazz.getName());
        }
    }
    /**
     * 动态加载指定包路径下实现指定服务接口/父类的所有子类实例
     *
     * @param service     目标服务接口/父类的Class对象，限定加载范围（例如：MyService.class）
     * @param basePackage 需要扫描的基础包路径（例如："com.example.providers"）
     * @return            包含所有实现类实例的列表，按类加载顺序排列
     * @throws RuntimeException 当通过反射实例化类失败时抛出
     *
     * @implSpec
     * 1. 使用Reflections库扫描指定包路径下所有service的子类
     * 2. 通过反射机制实例化每个找到的子类
     * 3. 要求所有子类必须具有无参构造函数
     */
    public static <T> List<T> load(Class<T> service, String basePackage) {
        Reflections reflections = new Reflections(basePackage);
        Set<Class<? extends T>> impls = reflections.getSubTypesOf(service);
        return impls.stream().map(ReflectionUtils::newInstance).collect(Collectors.toList());
    }
}
```

## 接口类 Example
```java
public interface Animal{
    
    /**
     * 类名全大写作为类型
     *
     * @return code
     */
    default String getType() {
        return this.getClass().getSimpleName().toUpperCase();
    }

    /**
     * 子类行为
     */
    void bark();
}
```

## 实现类 Example
```java
public class Cat implements Animal{

    public void bark(){
        System.out.println("喵喵~");
    }
}

public class Dog implements Animal{

    public void bark(){
        System.out.println("汪汪~");
    }
}
```

## 核心代码-控制器 
```java
public class AnimalSelector{

    private static final AnimalSelector SINGLETON = new AnimalSelector();
    private final Map<String, Animal> animalMap;

    public static AnimalSelector get() {
        return SINGLETON;
    }
    /**
     * 扫描与{@link Animal}接口相同包路径下的实现类，放入animalMap单例池。
     * @tips 如果是只定义接口，不负责实现，打包提供给外部项目使用时，可结合Java SPI，或者设计Configuration配置类，动态配置扫描包路径。
     */
    private AnimalSelector() {
        List<Animal> animals= ReflectionUtils.load(Animal.class, Animal.class.getPackage().getName());
        animalMap = animals.stream().filter(Objects::nonNull).collect(Collectors.toMap(Animal::getType, f -> f, (f1, f2) -> f1));
    }


    /**
     * 摸了动物之后，动物会叫
     */
    void touch(String type){
        Animal animal = animalMap.get(type.toUpperCase());
        if (animal != null) {
            animal.bark();
        } else {
            System.out.println(type + " is not exist.");
        }
    }
}
```

## 测试代码

```java
public class Test {
    public static void main(String[] args) {
        AnimalSelector.get().touch("Cat");
        AnimalSelector.get().touch("Dog");
        AnimalSelector.get().touch("Fox");
    }
}
```