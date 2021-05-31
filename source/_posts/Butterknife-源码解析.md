---
title: Butterknife 源码解析
date: 2020-12-27 18:32:06
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags:
    - Java
    - Android
---

## 1. 前言

Butterknife 依赖注入框架，简化手动书写 findViewById 等等，这里我们将深入源码来看看 Butterknife 内部的实现原理。

## 2. 基本使用

2.1 在app的build.gradle中添加ButterKnife依赖
```js
  ...
  // Butterknife requires Java 8.
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}

dependencies {
  implementation 'com.jakewharton:butterknife:10.2.3'
  annotationProcessor 'com.jakewharton:butterknife-compiler:10.2.3'
}
```
如何使用
```js
class ExampleActivity extends Activity {
  @BindView(R.id.user) EditText username;
  @BindView(R.id.pass) EditText password;

  @BindString(R.string.login_error) String loginErrorMessage;

  @OnClick(R.id.submit) void submit() {
    // TODO call server...
  }

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```

## 3. 先说结论

先大致整体的整理一下，ButterKnife 的整理流程:
&emsp;1.首先，在编译期间扫描声明的注解（如 BindView ），通过 ButterKnifeProcessor 注解处理器相应的解析以及调用 Javapoet 库生成 _ViewBinding 模板代码。
&emsp;&emsp;&emsp;1.1.主要完成目标类信息的收集,返回Map<TypeElement, BindingSet> bindingMap，里面包含了目标类信息
&emsp;&emsp;&emsp;1.2.遍历 bindingMap，根据BindingSet获得一个JavaFile对象，而后输入java 类这个过程使用JavaPoet开源库实现，该提供了一种友好的方式来辅助生成java类的代码，同时将类代码生成文件。不然，须要本身拼接字符串来实现。
&emsp;2.然后，在我们调用 ButterKnife.bind(this) 方法的时候，通过类加载器加载指定的类（如 MainActivity_ViewBinding ），并通过反射方式实例化该模板类，从而实现了对标识注解的控件赋值。
&emsp;&emsp;&emsp;2.1从BINDINGS缓存中获取Class对象的构造器，若是存在，则直接return
&emsp;&emsp;&emsp;2.2获取Class对象的全限定名称(包名+类型)，经过PathClassLoader类加载器，经过反射获取MainActivity_ViewBinding的构造器
&emsp;&emsp;&emsp;2.3调用MainActivity_ViewBinding的构造器，传入的值是Activity和Activity的顶层视图DecorView，该步骤主要就是在构造体中完成findViewById或者setOnClickLisener等操作，并将控件赋值到Activity的控件，完成绑定工作

## 4. 源码解析_后续补充