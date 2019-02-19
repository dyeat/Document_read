## **常見的INI配置**
<p>

### 安全模式

```safe_mode = on```

### 安全模式下執行程序主目錄

```safe_mode_exec_dir = /var/www/html```

### 魔術引號

```
magic_quotes_gpc = on
```
<p>
防止SQL注入

### HTTP頭PHP版本信息
```
 expose_php = Off
```
### 用戶訪問目錄限制
```
open_basedir = .:/tmp/
```
<p>
表示允許訪問當前目錄(即PHP腳本文件所在之目錄)和/tmp/目錄，有效防止php木馬跨站運行。

### 設置上傳文件大小
```
file_uploads = on

upload_max_filesize = 8M

post_max_size = 8M
```
<p>

POST的數值要大於等於upload，否則upload不起作用。

### 設定一個腳本使用的最大內存

```
memory_limit = 128M
```
### 啟用全局變量
```
register_globals = on
```
<p>
有些程序例如OSC需要啟用全局變量.

### 設置默認編碼
```
default_charset = 'iso--8859-1'
```
<p>

這個一般不需要設置，因為大部分頁面都有製定編碼。

### 是否允許打開遠程文件

```
allow_url_fopen = on
```
### 是否允許包含遠程文件(include/require)
```
allow_url_include = false
```
### 時區
```
date.timezone = UTC
```
<p>
PHP默認採用配置項中的時區，如果項目中涉及時區，請用date_default_timezone_get()/date_default_timezone_set(utc)來設定自己想要的時區。

###禁用類／方法
```
disable_classes =
disable_functions =
```
<p>
我的php.ini原來禁用了很多方法，會影響一些功能的使用，比如遠程鏈接之類的。發送email應該回應想到。

### 設置錯誤報告級別
```
error_reporting =
```
<p>
日誌級別是一些常量，在php.ini中有寫，推薦使用 E_ALL | E_STRICT，即所有級別。

### 輸出錯誤信息
```
display_error = off
```
<p>
是否將錯誤信息作為輸出的一部分，站點發布後應該關閉這項功能，以免暴露信息。調試的時候當然是要On的，不然就什麼錯誤信息也看不到了。

###定義各個級別系統日誌變量
```
define_syslog_variables = off
```
<p>
建議關閉，以提高性能。

### 錯誤日誌
```
error_log =
```
<p>
錯誤日誌的位置，必須對web用戶可寫入，如果不定義則默認寫入到web服務器的錯誤日誌中去.
```
log_errors = on
```
<p>
如上所說，建議將錯誤日誌輸出到文件，而不是直接輸出到前端。
```
log_errors_max_length = 1024
```
<p>
錯誤日誌關聯信息的最大長度，設為0表示無限長度。

### 默認socket時長
```
default_socket_timeout = 60
```
### 腳本最大執行時長
```
max_execution_time = 30
```
<p>
設定每個腳本的最大執行時長，有助於阻止劣質腳本無限制佔用服務器資源；0表示沒有限制。

### 腳本最大輸入時間
```
max_input_time = 60
```
單個腳本最大輸入時長
### 文件上傳臨時目錄
```
upload_tmp_dir =
```
<p>
PHP進程用戶可寫目錄，如果不設置，則採用系統臨時目錄。 （tmp）

### 輸出後自動刷新資料
```
implicit_flush = off
```
<p>
是否在echo();printf();等輸出資料塊之後刷新；對性能影響嚴重，只建議在調試狀態下開啟。