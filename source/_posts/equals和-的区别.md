---
title: equals和==的区别
date: 2020-03-15 00:53:55
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags:
    - Java
---

## ==在java中有两个主要的做用

1.基础数据类型比较：比较的是他们的值是否相同，如两个int类型的变量，比较的是变量的值是否一样
2.引用数据类型：比较的是引用的地址是否相同，比如我们自定义的对象User，比较的是两个User的地址是否一样

## equals

1.没有重写equals方法，比较自定义的User对象==和equals的做用是相同的，都是比较引用是否指向同一个内存，在equals源码中体现如下
```js
public boolean equals(Object obj) {
    return (this == obj);
}
```

2.重写了equals方法，比如String类型看一下String类型的equals方法是如何实现的
```js
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = length();
        if (n == anotherString.length()) {
            int i = 0;
            while (n-- != 0) {
                if (charAt(i) != anotherString.charAt(i))
                        return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```
先比较了两个String对象的引用地址是否相同，若不相同先判断传入的值是否是String类型，在比较两个String的长度是否相同，最后比较每个字符是否相同完成整个比较过程