---
title: HTML标签-链接标签
date: 2017-03-09 12:20:03
tags: HTML标签
---


# HTML链接标签
链接是网络的主要特色,因为链接允许你从一个网页跳转到另一个网页,实现了人们在网上浏览和冲浪的想法.一般情况下,你会遇到下面几种链接:

1. 从一个网站指向另一个网站的链接.
2. 从一个网页指向同一个网站内部另一个网页的链接
3. 从网页的一个位置指向同一网页内另一个位置的链接
4. 在新的浏览器窗口中打开的链接
5. 启动你的电子邮件程序并为其添加收件人的链接

## 指向其他网站的链接
``<a>``

``` html

<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>

<p>BAT:
    <ul>
        <li><a href="http://www.baidu.com">百度</a></li>
        <li><a href="http://www.alibaba.com">阿里巴巴</a></li>
        <li><a href="http://www.tencent.com">腾讯</a></li>
    </ul>
</p>

</body>
</html>

```

效果图:<br />
![linking-to-other-sites](HTML标签-列表标签/linking-to-other-sites.png "linking-to-other-sites")
;;;
标签用法:网页中的链接是通过``<a>``元素建立的,``<a>``元素拥有一个重要的特性:href,href特性的值设定了链接的目标,即网站用户单击链接时所到达的页面地址.当网站用户单击位于链接起始标签``<a>``和结束标签``<a/>``之间的内容时,就会打开href特性所设定的页面.如果链接指向另一个网站,那么href特性的值必须是另一个网站的完整Web地址,也就是**绝对URL**。默认情况下,链接文本在浏览器显示为蓝色并带有下划线.