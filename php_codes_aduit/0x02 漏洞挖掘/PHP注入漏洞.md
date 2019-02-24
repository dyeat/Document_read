## **PHP注入漏洞**
PHP腳本中，我們可以對輸入進行注入攻擊。什麼是注入？注入即是通過利用無驗證變數構造特殊語句對服務器進行滲透。
<p>
注入的種類有很多，而不僅僅只是SQL Injection。那麼PHP中有哪些注入呢？

- 命令注入(Command Injection)
- Eval 注入(Eval Injection)
- 客戶端腳本攻擊(Script Insertion)
- 跨網站腳本攻擊(Cross Site Scripting, XSS)
- SQL 注入攻擊(SQL injection)
- 動態函數注入攻擊(Dynamic Variable Evaluation)
- 序列化注入&對象注入


### **命令注入**

PHP中可以使用下列5個函數來執行外部的應用程式或函數

- system()
- exec()
- passthru()
- shell_exec()
- ``(與 shell_exec 功能相同)


### **函數原型**
```
string system(string command, int &return_var)
string exec (string command, array &output, int &return_var)
void passthru (string command, int &return_var)
string shell_exec (string command)

command 要執行的命令
return_var 存放執行命令的執行後的狀態值
output 獲得執行命令輸出的每一行字符串
```

### **漏洞實例**
```
//ex1.php
<?php
$dir = $_GET["dir"];
if (isset($dir))
{
    echo "<pre>";
    system("ls -al ".$dir);
    echo "</pre>";
}
?>
```
我們提交 ```http://www.xxx.com/ex1.php?dir=| cat /etc/passwd```
提交以後，命令變成了 ```system("ls -al | cat /etc/passwd");```

### **eval 注入**
Eval 函數將輸入的字符串參數當作 PHP 程式代碼來執行

## **函數原型**

```
mixed eval(string code_str)
//eval 注入一般發生在攻擊者能控制輸入的字符串的時候
```

### **漏洞實例**
```
//ex2.php
<?php
$var = "var";
if (isset($_GET["arg"]))
{
    $arg = $_GET["arg"];
    eval("\$var = $arg;");
    echo "\$var =".$var;
}
?>
```

當我們提交 ```http://www.xxx.com/ex2.php?arg=phpinfo();``` 漏洞就產生了

### **動態函數注入攻擊**
```
//ex3.php
<?php
func A()
{
    dosomething();
}
func B()
{
    dosomething();
}
if (isset($_GET["func"]))
{
    $myfunc = $_GET["func"];
    echo $myfunc();
}
?>
```
程式員原意是想動態調用A和B函數， 那我們提交```http://www.xxx.com/exp3.php?func=phpinfo``` 漏洞就產生了

### **序列化注入&对象注入**
**unserialize的一个小特性**

unserialize() 函數相關原始碼
```
if ((YYLIMIT - YYCURSOR) < 7) YYFILL(7);
    yych = *YYCURSOR;
    switch (yych) {
    case 'C':
    case 'O':        goto yy13;
    case 'N':        goto yy5;
    case 'R':        goto yy2;
    case 'S':        goto yy10;
    case 'a':        goto yy11;
    case 'b':        goto yy6;
    case 'd':        goto yy8;
    case 'i':        goto yy7;
    case 'o':        goto yy12;
    case 'r':        goto yy4;
    case 's':        goto yy9;
    case '}':        goto yy14;
    default:        goto yy16;
    }
```

經過測試,unserialize()解序列化一個完整序列化語句之後的字符串是會被無視掉的。

```unserialize('a:1:{s:5:"phone";s:11:"12345678901";}')```

等價於

```unserialize('a:1:{s:5:"phone";s:11:"12345678901";}xxxxxxxxxxxxxxxx')```

在一定的條件下、可以構造語句截斷序列化。
<p>

### **其他序列化&對象注入相關漏洞文章**

- [通過PHP反序列化進行遠程代碼執行](http://www.freebuf.com/vuls/80293.html)
- 理解php對象注入
- WordPress < 3.6.1 PHP 對象注入漏洞
- [PHP源碼中unserialize函數引發的漏洞分析](https://www.91ri.org/3960.html)

---