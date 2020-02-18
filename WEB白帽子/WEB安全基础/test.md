

哪些地方可以运行JS
1. <script>标签之前
2. 事件属性中
![8.png](./images/8.jpg)


DOM将文档转成树形结构
1. 更直观地了解页面元素
2. 通过js对HTML进行任意操作
![7.png](./images/7.jpg)

 通过JS+DOM访问和操作HTML文档
 
 
 JavaScript BOM:Browser Object Model
 获取浏览器 和操作浏览器
 
 警告alert()  确认 confirm() 提示prompt()常用于简单的调试和信息展示
 
```
<!---->写入cookie
document.cookie =" =   "

<!---->
alert(document.cookie)
``` 

*****
![6.png](./images/9.jpg)

## XSS
跨站脚本攻击: Cross Site Scripting
是一种针对前端的注入，插入恶意脚本

存储型XSS
<image src=a onerror=alert(/xss/)>
![9.png](./images/9.jpg)
![11.png](./images/11.jpg)
![12.png](./images/12.jpg)

反射型XSS
```<?php
echo $_GET["name"];
?>
```
![12.png](./images/13.jpg)



DOM型
浏览器通过JS从URL中获得XSS脚本内容写入DOM中

而反射型XSS是后端程序将XSS脚本写入反射页面中



URL的Hash

## CSRF
跨站请求伪造
iframe嵌入可以提高攻击的隐蔽性
![csrf.png](./images/CSRF.jpg)


## 点击劫持
iframe 内联框架
![15.png](./images/15.jpg)

display属性：block以区块的方式显示
一般为none

这样可以让自己的网站和别的网站看起来一样
![16.png](./images/16.jpg)


点击劫持=UI覆盖攻击
诱导用户进行点击


## URL跳转漏洞
![17.png](./images/17.jpg)

## SQL注入——服务端安全问题

![sql.png](./images/sql.jpg)

union select 
UNION 操作符用于合并两个或多个 SELECT 语句的结果集。
注意，UNION 内部的 SELECT 语句必须拥有相同数量的列。


curl的作用：发送HTTP请求
