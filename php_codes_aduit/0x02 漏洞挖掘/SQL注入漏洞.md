## **SQL注入漏洞**
SQL攻擊（SQL injection，稱作SQL資料隱碼攻擊），簡稱注入攻擊，是發生於應用程序之資料庫層的安全漏洞。
<p>
簡而言之，是在輸入的字符串之中註入SQL指令，在設計不良的程序當中忽略了檢查，
<p>
那麼這些注入進去的指令就會被資料庫服務器誤認為是正常的SQL指令而運行，因此遭到破壞。
<p>
有部份人認為SQL注入攻擊是只針對Microsoft SQL Server而來，但只要是支持批處理SQL指令的資料庫服務器，都有可能受到此種手法的攻擊。


## **原因**
在應用程序中若有下列狀況，則可能應用程序正暴露在SQL Injection的高風險情況下：
- 在應用程序中使用字符串聯結方式組合SQL指令。
- 在應用程序鏈接資料庫時使用權限過大的賬戶（例如很多開發人員都喜歡用sa【內置的最高權限的系統管理員賬戶】連接Microsoft SQL Server資料庫）。
- 在資料庫中開放了不必要但權力過大的功能（例如在Microsoft SQL Server資料庫中的xp_cmdshell延伸預存程序或是OLE Automation預存程序等）太過於信任用戶所輸入的資料，未限制輸入的字符數，以及未對用戶輸入的資料做潛在指令的檢查。

## **作用原理**
SQL命令可查詢、插入、更新、刪除等，命令的串接。而以分號字符為不同命令的區別。 （原本的作用是用於SubQuery或作為查詢、插入、更新、刪除……等的條件式）
<p>
SQL命令對於傳入的字符串參數是用單引號字符所包起來。 《但連續2個單引號字符，在SQL資料庫中，則視為字符串中的一個單引號字符》
<p>
SQL命令中，可以注入註解《連續2個減號字符--後的文字為註解，或/*與*/所包起來的文字為註解》
<p>
因此，如果在組合SQL的命令字符串時，未針對單引號字符作取代處理的話，將導致該字符變量在填入命令字符串時，被惡意竄改原本的SQL語法的作用。


## **SQL注入攻擊的主要原因**
1. PHP 配置文件 php.ini 中的 magic_quotes_gpc選項沒有打開，被置為 off；
2. 開發者沒有對資料類型進行檢查和轉義。
<p>
不過事實上，第二點最為重要。我認為,對用戶輸入的資料類型進行檢查，向 MYSQL 提交正確的資料類型，這應該是一個 web 程序員最最基本的素質。但現實中，常常有許多小白式的 Web 開發者忘了這點，從而導致後門大開。
<p>
為什麼說第二點最為重要？因為如果沒有第二點的保證，magic_quotes_gpc 選項，不論為 on，還是為 off，都有可能引發 SQL 注入攻擊。下面來看一下技術實現：

**agic_quotes_gpc = Off**

magic_quotes_gpc = Off 是 php 中一種非常不安全的選項。新版本的 php 已經將默認的值改為了 On。但仍有相當多的服務器的選項為 off。畢竟，再古董的服務器也是有人用的。
<p>
當magic_quotes_gpc = On　時，它會將提交的變量中所有的 '(單引號)、"(雙號號)、(反斜線)、空白字符，都會在前面自動加上 。
<p>
如果沒有轉義，即 off 情況下，就會讓攻擊者有機可乘。
以下列測試腳本為例：

```php
<?
if (isset($_POST["f_login"])) 
	{

		// 連接資料庫... 程式碼略...
		// 檢查用戶是否存在
		$t_strUname = $_POST["f_uname"];
		$t_strPwd = $_POST["f_pwd"];
		$t_strSQL = "SELECT * FROM tbl_users WHERE username='$t_strUname' AND password = '$t_strPwd' LIMIT 0,1";
		if ($t_hRes = mysql_query($t_strSQL)) 
		{
	    // 成功查詢之後的處理. 略...
		}
}
?>
<form method="post" action="">
  Username: <input type="text" name="f_uname" size=30><br>
  Password: <input type=text name="f_pwd" size=30><br>
  <input type="submit" name="f_login" value="登錄">
</form>
```

在這個腳本中，當用戶輸入正常的用戶名和密碼，假設值分別為 zhang3、abc123，則提交的 SQL 語句如下：

```SQL
SELECT * FROM tbl_users WHERE username='zhang3' AND password = 'abc123' LIMIT 0,1
```
如果攻擊者在 username 字段中輸入：zhang3' OR 1=1 #，在 password 輸入 abc123，則提交的 SQL 語句變成如下：
```
SELECT * FROM tbl_users WHERE username='zhang3' OR 1=1 #' AND password = 'abc123' LIMIT 0,1
```
由於 # 是 mysql中的註釋符,#之後的語句不被執行，實現上這行語句就成了：

```
SELECT * FROM tbl_users WHERE username='zhang3' OR 1=1
```
這樣攻擊者就可以繞過認證了。如果攻擊者知道資料庫結構，那麼它構建一個 UNION SELECT，那就更危險了：假設在 username 中輸入：```zhang3 ' OR 1 =1 UNION select cola, colb,cold FROM tbl_b #,在password 輸入： abc123```
則提交的 SQL 語句變成：

```
SELECT * FROM tbl_users WHERE username='zhang3 ' OR 1 =1 UNION select cola, colb,cold FROM tbl_b #' AND password = 'abc123' LIMIT 0,1
```
**magic_quotes_gpc = On**

當 magic_quotes_gpc = On 時，攻擊者無法對字符型的字段進行 SQL 注入。這並不代表這就安全了。這時，可以通過數值型的字段進行SQL注入。
<p>
在最新版的 MYSQL 5.x 中，已經嚴格了資料類型的輸入，已默認關閉自動類型轉換。數值型的字段，不能是引號標記的字符型。也就是說，假設 uid 是數值型的，在以前的 mysql 版本中，這樣的語句是合法的：
<p>

```
INSERT INTO tbl_user SET uid="1";
SELECT * FROM tbl_user WHERE uid="1";
```

在最新的 MYSQL 5.x 中，上面的語句不是合法的，必須寫成這樣：
```
INSERT INTO tbl_user SET uid=1;
SELECT * FROM tbl_user WHERE uid=1;
```

那麼攻擊者在 magic_quotes_gpc = On 時，他們怎麼攻擊呢？很簡單，就是對數值型的字段進行 SQL 注入。以下列的 php 腳本為例：

```php
<?
if (isset($_POST["f_login"])) 
{
		// 連接資料庫...程式碼略...
		// 檢查用戶是否存在
		$t_strUid = $_POST["f_uid"];
		$t_strPwd = $_POST["f_pwd"];
		$t_strSQL = "SELECT * FROM tbl_users WHERE uid=$t_strUid AND password = '$t_strPwd' LIMIT 0,1";
		if ($t_hRes = mysql_query($t_strSQL)) 
		{
	    // 成功查詢之後的處理. 略...
		}
}
?>
<form method="post" action="">
  User ID: <input type="text" name="f_uid" size=30><br>
  Password: <input type=text name="f_pwd" size=30><br>
  <input type="submit" name="f_login" value="登錄">
</form>
```
上面這段腳本要求用戶輸入 userid 和 password 登入。一個正常的語句，用戶輸入 1001和abc123，提交的 sql 語句如下：
```
SELECT* FROMtbl_users WHEREuserid=1001 ANDpassword= 'abc123'LIMIT0,1
```
如果攻擊者在 userid 處，輸入：1001 OR 1 =1 #，則注入的sql語句如下：
```
SELECT* FROM tbl_users WHERE userid=1001 OR 1=1 # AND password= 'abc123'LIMIT 0,1
```
攻擊者達到了目的。

## **MySql注入**

傳送門

[MySQL注入總結-獨自等待](http://www.waitalone.cn/mysql-injection-summary.html)

## **挖掘技巧**
- 瀏覽全文、整理網站結構
- 找到資料庫連接、配置等相關信息
	- 編碼方式GBK存在寬字節注入
- 找到過濾函數，能否繞過
	- 有無GPC過濾
	- 自定義函數-字符串替換
	- 自定義函數-正則匹配替換
- 全文搜索，查找SQL語句
	- 無引號保護的變量可能存在註入
	- 是否存在urldecode導致注入
- 根據流程，尋找是否存在二次注入
**功力深厚的可以手工測試跟進注入，懶一點的扔到sqlmap跑一下。**


## **手工測試**
- 開啟vMysqlMonitoring對資料庫執行的語句進行監視。
- PHP源碼相關語句後面插入 var_dump($result); 輸出查詢結果
- 用Burp抓包、重放攻擊！