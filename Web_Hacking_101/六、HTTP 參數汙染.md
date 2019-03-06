## **六、HTTP 參數汙染**

>作者：Peter Yaworski
>
>譯者：飛龍
>
>協議：CC BY-NC-SA 4.0

## **描述**
HTTP參數污染，或者HPP，在網站接受用戶輸入，將其用於生成發往其它系統的HTTP請求，並且不校驗用戶輸出的時候發生。它以兩種方式產生，通過服務器（後端）或者通過客戶端。
<p>

在 StackExchange 上，SilverlightFox 提供了一個 HPP 服務端攻擊的不錯的例子。假設我們擁有以下站點：`https://www.example.com/transferMoney.php`，它可以通過 POST 方法訪問，帶有以下參數：

<pre>
amount=1000&fromAccount=12345
</pre>

當應用處理請求時，它生成自己的發往其它後端系統的 POST 請求，這實際上會使用固定的`toAccount`參數來處理事務。

分離後端 URL：`https://backend.example/doTransfer.php`
<p>
分離後端參數：`toAccount=9876&amount=1000&fromAccount=12345`
<p>
現在，如果在提供了重複的參數時，後端僅僅接受最後一個參數，

並且假設攻擊者修改了發往網站的 POST 請求來提交`toAccount`參數，像這樣：

<pre>
amount=1000&fromAccount=12345&toAccount=99999
</pre>
存在 HPP 漏洞的站點就會將請求轉發給另一個後端系統，像這樣：
<pre>
toAccount=9876&amount=1000&fromAccount=12345&toAccount=99999
</pre>

這裡，由惡意用戶提交的第二個`toAccount`參數，會覆蓋後端請求，並將錢轉賬給惡意用戶調教得帳戶（`99999`）而不是由系統設置的預期帳戶（`9876`）。
<p>
如果攻擊者打算修改它們自己的請求，並且由漏洞系統處理，這非常實用。但是如果攻擊者可以從另一個攢點生產鏈接，並且誘使用戶無意中提交惡意請求，並帶有由攻擊者附加的額外參數，它也可以對攻擊者更加實用一些。
<p>
另一方面，HPP 客戶端涉及到向鏈接和其它src屬性注入額外的參數。在 OWASP 的一個例子中，假設我們擁有下列代碼：

```php
<? $val=htmlspecialchars($_GET['par'],ENT_QUOTES); ?> <a href="/page.php?action=view&par='.<?=$val?>.'">View Me!</a>
```

它從 URL 接受`par`的值，確保它是安全的，並從中創建鏈接。現在，如果攻擊者提交了：
<pre>
http://host/page.php?par=123%26action=edit
</pre>

產生的連結可能為:
```html
<a href="/page.php?action=view&par=123&amp;action=edit">View Me!</a>
```

這會導致應用接受編輯操作而不是查看操作。
<br >
HPP 服務端和客戶端都依賴於所使用的的後端技術，以及在收到多個名稱相同的參數時，它的行為如何。例如，PHP/Apache 使用最後一個參數，Apache Tomcat 使用第一個參數，ASP/IIS 使用所有參數，以及其他。所以，沒有可用於提交多個同名參數的單一保險的處理方式，發現 HPP 需要一些經驗來確認你所測試的站點如何工作。


---

## **示例**


## **1. HackerOne 社交分享按钮**

```
難度：低

URL：https://hackerone.com/blog/introducing-signal-and-impact
報告鏈接；https://hackerone.com/reports/105953
報告日期：2015.12.18
獎金：$500
```
描述：HackerOne 包含鏈接，用於在知名社交媒體站點上分享內容，例如 Twitter，Fackbook，以及其他。這些社交媒體的鏈接包含用於社交媒體鏈接的特定參數。
<p>
攻擊者可以將另一個 URL 參數追加到鏈接中，並讓其指向任何他們所選的站點。 HackerOne 將其包含在發往社交媒體站點的 POST 請求中，因而導致了非預期的行為。這就是漏洞所在。

漏洞報告中所用的示例是將 URL：
<pre>
https://hackerone.com/blog/introducing-signal
</pre>
修改為：
<p>
<pre>
https://hackerone.com/blog/introducing-signal?&u=https://vk.com/durov
</pre>

要注意額外的參數`u`。如果惡意更新的鏈接有 HackerOne 訪客點擊，嘗試通過社交媒體鏈接分享內容，惡意鏈接就變為：
<pre>
https://www.facebook.com/sharer.php?u=https://hackerone.com/blog/introducing-signal?&u=https://vk.com/durov
</pre>
<p>
這裡，最後的參數u就會擁有比第一個更高的優先級，之後會用於 Fackbook 的發布。在 Twitter 上發佈時，建議的默認文本也會改變：
<pre>
https://hackerone.com/blog/introducing-signal?&u=https://vk.com/durov&text=another_site:https://vk.com/durov
</pre>

>重要結論
>
>當網站接受內容，並且似乎要和其他 Web 服務連接時，例如社交媒體站點，一定要尋找機會。
>這些情況下，被提交的內容可能在沒有合理安全檢查的情況下傳遞。