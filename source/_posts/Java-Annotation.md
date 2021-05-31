---
title: Java Annotation
date: 2020-11-10 21:44:55
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags:
    - Java
    - Android
---

## 1.概念

> An annotation is a form of metadata, that can be added to Java source code. Classes, methods, variables, parameters and packages may be annotated. Annotations have no direct effect on the operation of the code they annotate.
>能够添加到Java源代码的语法元数据。类、方法、变量、参数、包都可以被注解，可用来将信息元数据与程序元素进行关联。

## 2.作用
&emsp;&emsp;1. 标记，用于告诉编译器一些信息
&emsp;&emsp;2. 编译时动态处理，如动态生成代码
&emsp;&emsp;3. 运行时动态处理，如得到注解信息

## 3.Annotation 分类

&emsp;&emsp;**标准 Annotation**，如 Override, Deprecated, SuppressWarnings 标准 Annotation 是指 Java 自带的几个 Annotation，上面三个分别表示重写函数，不鼓励使用(有更好方式、使用有风险或已不在维护)，忽略某项 Warning
&emsp;&emsp;**元 Annotation**，@Retention, @Target, @Inherited, @Documented 元 Annotation是指用来定义 Annotation 的 Annotation，在后面 Annotation 自定义部分会详细介绍含义
&emsp;&emsp;**自定义 Annotation**，表示自己根据需要定义的 Annotation，定义时需要用到上面的元 Annotation 这里是一种分类而已，也可以根据作用域分为源码时、编译时、运行时 Annotation，后面在自定义 Annotation 时会具体介绍

## 4.Annotation 自定义

**4.1 定义**
```js
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface MyAnnotation {
    int id();
    String name() default "hello world";
}
```
这里是 MyAnnotation 的实现部分
(1). 通过 @interface 定义，注解名即为自定义注解名
(2). 注解配置参数名为注解类的方法名，且：
&emsp;&emsp;a. 所有方法没有方法体，没有参数没有修饰符，实际只允许 public & abstract 修饰符，默认为 public，不允许抛异常
&emsp;&emsp;b. 方法返回值只能是基本类型，String, Class, annotation, enumeration 或者是他们的一维数组
&emsp;&emsp;c. 若只有一个默认属性，可直接用 value() 函数。一个属性都没有表示该 Annotation 为 Mark Annotation
(3). 可以加 default 表示默认值

**4.2 元 Annotation**
&emsp;&emsp;**@Documented** 是否会保存到 Javadoc 文档中
&emsp;&emsp;**@Retention** 保留时间，可选值 SOURCE（源码时），CLASS（编译时），RUNTIME（运行时），默认为 CLASS，SOURCE 大都为 Mark Annotation，这类 Annotation 大都用来校验，比如 Override, SuppressWarnings,`作用：表示需要在什么级别保存该注释信息，用于描述注解的生命周期（即：被描述的注解在什么范围内有效）`
&emsp;&emsp;取值（RetentionPoicy）有：
&emsp;&emsp;&emsp;&emsp;1.SOURCE:在源文件中有效（即源文件保留）
&emsp;&emsp;&emsp;&emsp;2.CLASS:在class文件中有效（即class保留）
&emsp;&emsp;&emsp;&emsp;3.RUNTIME:在运行时有效（即运行时保留）
&emsp;&emsp;**@Target** 可以用来修饰哪些程序元素，如 TYPE, METHOD, CONSTRUCTOR, FIELD, PARAMETER 等，未标注则表示可修饰所有,`作用：用于描述注解的使用范围（即：被描述的注解可以用在什么地方）`
&emsp;&emsp;取值(ElementType)有：
&emsp;&emsp;&emsp;&emsp;1.CONSTRUCTOR:用于描述构造器
&emsp;&emsp;&emsp;&emsp;2.FIELD:用于描述域
&emsp;&emsp;&emsp;&emsp;3.LOCAL_VARIABLE:用于描述局部变量
&emsp;&emsp;&emsp;&emsp;4.METHOD:用于描述方法
&emsp;&emsp;&emsp;&emsp;5.PACKAGE:用于描述包
&emsp;&emsp;&emsp;&emsp;6.PARAMETER:用于描述参数
&emsp;&emsp;&emsp;&emsp;7.TYPE:用于描述类、接口(包括注解类型) 或enum声明
&emsp;&emsp;**@Inherited** 是否可以被继承，默认为 false,
当 @Inherited annotation类型标注的annotation的Retention是RetentionPolicy.RUNTIME，则反射API增强了这种继承性。如果我们使用java.lang.reflect去查询一个
@Inherited annotation类型的annotation时，反射代码检查将展开工作：检查class和其父类，直到发现指定的annotation类型被发现，或者到达类继承结构的顶层

**4.3 使用**
```js
public class App {

    @MyAnnotation(id=100, name = "App_Name")
    public void annotationMethod(){

    }
}
```

## 5.Annotation 解析

**5.1 运行时 Annotation 解析**
运行时 Annotation 指 @Retention 为 RUNTIME 的 Annotation，可手动调用下面常用 API 解析
```js
method.getAnnotation(MyAnnotation.class);
method.getAnnotations();
method.isAnnotationPresent(MyAnnotation.class);
```
其他 @Target 如 Field，Class 方法类似
getAnnotation(MyAnnotation.class) 表示得到该 Target 某个 Annotation 的信息，因为一个 Target 可以被多个 Annotation 修饰
getAnnotations() 则表示得到该 Target 所有 Annotation
isAnnotationPresent(MyAnnotation.class) 表示该 Target 是否被某个 Annotation 修饰

测试代码如下：
```js
public class MyClass {
    public static void main(String str[]){
        try {
            Class cls = Class.forName("com.example.lib.App");
            for (Method method : cls.getMethods()) {
                MyAnnotation myAnnotation = method.getAnnotation(
                        MyAnnotation.class);
                if (myAnnotation != null) {
                    System.out.println("method name:" + myAnnotation.name());
                    System.out.println("method id:" + myAnnotation.id());
                }
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}


> Task :lib:MyClass.main()
method name:App_Name
method id:100
```
以之前自定义的 MyAnnotation 为例，利用 Target（这里是 Method）getAnnotation 函数得到 Annotation 信息，然后就可以调用 Annotation 的方法得到响应属性值
**5.1 编译时 Annotation 解析**
(1) 编译时 Annotation 指 @Retention 为 CLASS 的 Annotation，甴编译器自动解析。需要做的
&emsp;&emsp;a. 自定义类集成自 AbstractProcessor
&emsp;&emsp;b. 重写其中的 process 函数
实际是编译器在编译时自动查找所有继承自 AbstractProcessor 的类，然后调用他们的 process 方法去处理
(2) 假设 MyAnnotation 的 @Retention 为 CLASS，解析示例如下：
```js
@SupportedAnnotationTypes({ "com.example.lib.annotations.MyAnnotationClass" })
public class MyAnnotationProcessor extends AbstractProcessor {


    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        HashMap<String, String> map = new HashMap<String, String>();
        for (TypeElement te : annotations) {
            for (Element element : env.getElementsAnnotatedWith(te)) {
                MyAnnotationClass annotation = element.getAnnotation(MyAnnotationClass.class);
                map.put(element.getEnclosingElement().toString(), annotation.name());
            }
        }

        return false;
    }
}

```