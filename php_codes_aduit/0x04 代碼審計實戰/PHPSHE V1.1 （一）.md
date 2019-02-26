### **PHPSHE V1.1 （一）**

## **安裝phpshe**
/install/index.php
安裝成功後、分別打開前台/和後台```/admin.php?mod=do&act=login```


## 框架概覽

用submlie text 2打開源碼文件夾，用CTags生成標籤索引。

## **目錄結構**

根目錄/下面有：

- /data
- /hook
- /include
- /module
- /template
- robots.txt
- index.php
- admin.php
- config.php
- common.php
- .htaccess
- httpd.ini
接下來，分別瀏覽以上文件、分析起作用。


## **robots.txt**

```
#
# robots.txt for phpshe1.1
#
User-agent: * 
Disallow: /data
Disallow: /hook
Disallow: /include
Disallow: /module
Disallow: /template
Disallow: /admin.php
Disallow: /config.php
Disallow: /common.php
```

該文件洩漏了程序版本及文件結構訊息

**index.php 和 admin.php**
這兩個文件分別是前台和後台的入口文件,稍後仔細分析。


### **config.php**
資料庫配置訊息
<br >

---
### **common.php**
該文件作為來源資料處理和URL路由配置以及資料庫連接。
<br >

---

### **.htaccess 和 httpd.ini**
WEB服務程序的地址重新配置文件

---

## **來源資料處理文件 common.php**
```
//Line:6

error_reporting(E_ALL ^ E_NOTICE);
```

開啟了日誌報告、很多錯誤訊息都會顯示、也就可能有不少訊息洩漏的漏洞
```
//Line:15-19

//url路由配置
$module = $mod = $act = 'index';
$mod = $_POST['mod'] ? $_POST['mod'] : ($_GET['mod'] ? $_GET['mod'] : $mod);
$act = $_POST['act'] ? $_POST['act'] : ($_GET['act'] ? $_GET['act'] : $act);
$id = $_POST['id'] ? $_POST['id'] : ($_GET['id'] ? $_GET['id'] : $id);
```

`$mod`、`$act`、`$id`這幾個變數並沒有進過過濾處理。

## **前台入口 index.php**

該文件首先包含common.php，包含了各種類文件。
<br >
通過cache::get()獲取一些緩存訊息。
<br >
根據$module和$mod加載不同的模塊。
```
//Line:15

include("{$pe['path_root']}module/{$module}/{$mod}.php");
```
`$mod`沒有經過處理就直接引入到`include()`函數中。
<br >
雖然限定了`.php`後綴，但是並沒有什麼用。
PHP使用C語言開發的，%00也就是\0是結束符，可以截斷字符串。

### **【Bug 0x01: 任意文件包含漏洞 有限制條件】**
```/index.php?mod=../../robots.txt%00```


### **前臺模塊**
接下來，我們在跟進分析`/module/index/`裏面的前天模塊：
- index.php
- arcticle.php
- page.php
- order.php
- product.php
- user.php

### **index.php**
文件路徑 `/module/index/index.php`
<br />
看了下、這個文件沒什麼好分析的

### **arcticle.php**
文件路徑 `/module/index/arcticle.php`
模塊內容直接就是判斷`$act`選擇文章列表或者文章內容。
<br />
我們跟進文章列表
```
//Line:7

$info_list = $db->pe_selectall('article', array('class_id'=>$class_id, 'order by'=>'`article_atime` desc'), '*', array(20, $_g_page));
```

此處有可以疑點，變數`$_g_page`未經過濾引入sql查詢。
我們跟進函數`pe_selectall()`，跳轉到`/include/class/db.class.php`文件


```
//Line:129

public function pe_selectall($table, $where = '', $field = '*', $limit_page = array())｛
    ...
    return $this->sql_selectall("select {$field} from `".dbpre."{$table}` {$sqlwhere}", $limit_page);
```

繼續跟進sql_selectall()函數
```
//Line:82

public function sql_selectall($sql, $limit_page = array())｛
    ...
    $this->page = new page($allnum, $limit_page[1], $limit_page[0]);
```

繼續跟進page類
```
//Line:17

function __construct($allnum, $page = null, $listnum = null, $pagenum = null)｛
    ...
    $this->page = $page === null ? 1 : $page;
    $this->listnum = $listnum === null ? 20 : $listnum;
    ...
    $this->limit = $this->get_limit();
```

繼續跟進get_limit()函數
```
//Line:30    

function get_limit()
{
    empty($this->page) && $this->page = 1;
    $limit = ($this->page - 1) * $this->listnum;
    return " limit {$limit}, {$this->listnum}";
}
```

這時我們可以知道，變數`$_g_page`最終被引入到了`$limit`之中帶入了sql查詢語句。
<br />
當然`$limit`是`$this->listnum`和`$this->page - 1`的乘積，這樣就限制了很多利用方法。

### **【Bug 0x02: 訊息洩露 爆路徑 Fatal error】**
當`$_g_page`是陣列，即訪問`index.php?mod=article&id=1&act=list&page[]=1`

```Fatal error: Unsupported operand types in E:\SourceCodes\phpshe1.1\include\class\page.class.php```

### **【Bug 0x03: 訊息洩露 爆路徑 Warning】**

當`$_g_page`是字符串，即訪問`index.php?mod=article&id=1&act=list&page=a`

```Warning: mysql_fetch_assoc() expects parameter 1 to be resource, boolean given in E:\SourceCodes\phpshe1.1\include\class\db.class.php```