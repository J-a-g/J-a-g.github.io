<!-- # Hexo-Theme-Huxblog

> Ported Theme of [Hux Blog](https://github.com/Huxpro/huxpro.github.io), Thank [Huxpro](https://github.com/Huxpro) for designing such a flawless theme.

### [Demo &rarr;](http://kaijun.github.io/hexo-theme-huxblog) -->


<!-- ![](http://huangxuan.me/img/blog-desktop.jpg) -->
# 如何写文章

##### 1.设置本地图片
```
![java-javascript](name.jpg)
```

##### 2.设置网络图片
```
<img src="http://huangxuan.me/js-module-7day/attach/qrcode.png" width="350" height="350"/>
或
![](http://huangxuan.me/img/blog-desktop.jpg)
```

##### 3.图片下面的小字
```
<small class="img-hint">歪果仁的笑话怎么一点都不好笑</small>
```

##### 4.设置标题字体
```
# title1
或
## title2
或
##### title3
```

##### 5.超链接
```
[text](url)
```

##### 6.红色标记
```
`text`
```

##### 7.字体加粗
```
**text**
```

##### 8.引语
```
>text
```

##### 9.如何贴代码
```
···js
text
···
或者
···java
text
···
或者
{% codeblock lang:js %}
Text
{% endcodeblock %}

```

##### 10.空格
```
&emsp;
```

##### 11.‘+’号
```
就是一个点
```

##### 12.站内文章跳转
```
{% post_link 文章文件名（不要后缀） 文章标题（可选） %}
如
{% post_link Hello World 你好世界 %}
```

##### 13.文章内跳转到指定位置
```
需要点击的位置的代码：
[直接跳转](#hahaha)

目标位置的代码：
<span id="hahaha"></span>

设置目标位置的id为hahaha或其他的代号，在实际显示时看不到。
```
或
```
需要点击的位置的代码：
[跳转到黑](#hei)

目标位置的代码：
<div id="hei"></div>
```
或直接跳转到指定标题上
```
# 题目
 
[调转到标题1](#1标题一)
[调转到标题2](#2标题二)
 
## 1.标题一
## 2.标题二
## 3.标题三
```

##### 14.斜体
```
*测试内容*
也可以配合其他符号使用

*`测试内容`* //红色斜体

***测试内容*** //加粗斜体
```

##### 15.菜单
```
# 一级菜单
## 二级菜单
### 三级菜单
#### 四级菜单
##### 五级菜单
```

# Hexo常用命令
```
hexo clean # 清理缓存
hexo generate # 生成静态文件
hexo deploy
hexo new "my first article"
Hexo new post “xxx”
hexo server --port 3000 # 将服务器运行在端口 3000 上
```
