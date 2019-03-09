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


---

## **2. Twitter 取消訂閱提醒**

```
難度：低

URL：twitter.com
報告鏈接：https://blog.mert.ninja/twitter-hpp-vulnerability/
報告日期：2015.8.23
獎金：$700

```

描述：

2015 年 8 月，黑客 Mert Tasci 在取消接收 Twitter 的提醒時，注意到一個有趣的 URL。

<pre>https://twitter.com/i/u?t=1&cn=bWV&sig=657&iid=F6542&uid=1134885524&nid=22+26</pre>
（我在書裡面把它縮短了一些）。你注意到參數 UID 了嘛？這碰巧是你的 Twitter 賬戶 UID。
<br>
現在，要注意，他做了我認為多數黑客都會做的事情，他嘗試將 UID 修改為其它用戶，沒有其它事情。 
<br>
Twitter 返回了錯誤。
<br>
考慮到其他人可能已經放棄了，Mert 添加了第二個 UID 參數，所以 URL 看起來是這樣：

<pre>https://twitter.com/i/u?iid=F6542&uid=2321301342&uid=1134885524&nid=22+26</pre>
然後就成功了。他設法取消訂閱了其它用戶的郵件提醒。這就說明，Twitter 存在 HPP 取消訂閱的漏洞。

>重要結論
>
>通過一段簡短的描述，Mert 的努力展示了堅持和知識的重要性。如果它在測試另一個作為唯一參數的 UID 之後，遠離了這個漏洞，或者它根本不知道 
>HPP 類型漏洞，他就不會收到 $700 的獎金。
>
>同時，要保持關注參數，類似 UID，它們包含在 HTTP 請求中，因為我在研究過程中見過很多報告，它們涉及到操縱參數的值，並且 Web 
>應用做出了非預期的行為。

---

## **3. Twitter Web Intents**

```
難度：低

URL：twitter.com
報告鏈接：https://ericrafaloff.com/parameter-tampering-attack-on-twitter-web-intents
報告日期：2015.11
獎金：未知
```
描述：

根據它們的文檔，Twitter Web Intents，提供了彈出優化的數據流，用於處理 Tweets & Twitter 
用戶：發推、回复、轉發、喜歡和關注。它使用戶能夠在你的站點上下文中，和 Twitter 的內容交互，而不需要離開頁面或者授權新的應用來交互。這裡是它的一個示例：

![1](https://raw.githubusercontent.com/dyeat/Document_read/master/Web_Hacking_101/image/6-1.jpg)

Twitter Intent
充分測試之後，黑客 Eric Rafaloff 發現，全部四個 Intent 類型：關注用戶、喜歡推文、轉發和發推，都存在 HPP 漏洞。
根據他的博文，如果 Eric 創建帶有兩個`screen_name`參數的 URL：
<pre>https://twitter.com/intent/follow?screen_name=twitter&scnreen_name=erictest3</pre>
Twitter 會通過讓第二個screen_name比第一個優先，來處理這個請求。根據 Eric，Web 表單類似這樣：

```html
<form class="follow " id="follow_btn_form" action="/intent/follow?screen_name=er\ icrtest3" method="post"> <input type="hidden" name="authenticity_token" value="...">
    <input type="hidden" name="screen_name" value="twitter">

    <input type="hidden" name="profile_id" value="783214">

    <button class="button" type="submit">
        <b></b><strong>Follow</strong>
    </button>
</form>
```

受害者會看到在一個`screen_name`中定義的用戶資料，`twitter`，但是點擊按鈕後，它們會關注`erictest3`。
<p>

與之類似，當展現 intent 用於喜歡時，Eric 發現它能夠包含`screen_name`參數，雖然它和喜歡這個推文毫無關係，例如：

<pre>https://twitter.com/intent/like?tweet_id=6616252302978211845&screen_name=erictest3</pre>
喜歡這個推文會向受害者展示正確的用戶資料，但是點擊"關注"之後，它仍然會關注`erictest3`。

>重要結論
>
>這個類似於之前的 Twitter UID 漏洞。不出意料，當一個站點存在 HPP 漏洞時，它就可能是更廣泛的系統化問題的指標。
>有時如果你找到了類似的漏洞，它值得花時間來整體探索該平台，來看看是否存在其它可以利用相似行為的地方。
>這個例子中，就像上面的 UID，Twitter 接受用戶標識，`screen_name`，它基於後端邏輯易受 HPP 攻擊。


## **總結**

HTTP 參數污染的風險實際上取決於後端所執行的操作，以及被污染的參數提交到了哪裡。
<p>
發現這些類型的漏洞實際上取決於經驗，比其他漏洞尤甚，因為網站的後端行為可能對於黑客來說是黑盒。常常，作為一個黑客，對於後端在接收了你的輸入之後進行了什麼操作，你需要擁有非常細微的洞察力。
<p>
通過嘗試和錯誤，你可能能夠發現一些情況，其中站點和其它服務器通信，之後開始測試參數污染。社交媒體鏈接通常是一個不錯的第一步，但是要記住保持挖掘，並且當你測試類似 UID 的參數替換時，要想到 HPP。
