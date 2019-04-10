## **0x03-CRLF 注入**

>作者：Peter Yaworski
>
>譯者：飛龍
>
>協議：CC BY-NC-SA 4.0

CRLF 注入是一類漏洞，在用戶設法向應用插入 CRLF 時出現。在多種互聯網協議中，
<br>
包括 HTML，CRLF 字符表示了行的末尾，通常表示為`\r\n`，編碼後是`%0D%0A`。
<br>
在和 HTTP 請求或響應頭組合時，這可以用於表示一行的結束，並且可能導致不同的漏洞，包括 HTTP 請求走私和 HTTP 響應分割。

對 HTTP 請求走私而言，它通常在 HTTP 請求傳給服務器，服務器處理它並傳給另一個服務器時發生，例如代理或者防火牆。這一類型的漏洞可以導致：

- 緩存污染，它是一種場景，攻擊者可以修改緩衝中的條目，並託管惡意頁面（即包含 JavaScript）而不是合理的頁面。
- 防火牆繞過，它是一種場景，請求被構造，用於避免安全檢查，通常涉及 CRLF 和過大的請求正文。
- 請求劫持：它是一種場景，攻擊者惡意盜取 HTTPOnly 的 Cookie，以及 HTTP 驗證信息。這類似於 XSS，但是不需要攻擊者和客戶端之間的交互。

現在，雖然這些漏洞是存在的，它們難以實現。我在這裡引用了它們，所以你對如何實現請求走私有了更好的了解。
<p>
而對於HTTP 響應分割來說，攻擊者可以設置任意的響應頭，控制響應正文，或者完全分割響應來提供兩個響應而不是一個，它在示例#2 （Shopify 響應分割）中演示（如果你需要HTTP 請求和響應頭的備忘錄，請回到"背景"一章）。


---

## **1. Twitter HTTP 響應分割**

```
難度：高

URL：https://twitter.com/i/safety/report_story
報告鏈接：https://hackerone.com/reports/52042
報告日期：2015.4.21
獎金：$3500
```
描述：
<br>
2015 年 4 月，有報告稱，Twitter 存在一個漏洞，允許攻擊者通過將信息添加到發往 Twitter 的請求，設置任意 Cookie。

本質上，在生成上面 URL 的請求之後（一個 Twitter 的遺留功能，允許人們報告廣告），Twitter 會為參數`reported_tweet_id`返回 Cookie。但是，根據報告，Twitter 的驗證存在缺陷，它用於確認推文是否是數字形式。

<p>

雖然 Twitter 驗證了換行符`0x0a`不能被提交時，驗證機制可以通過將字符編碼為 UTF-8 來繞過。這麼做之後，Twitter 會將字符轉換會原始的 Unicode，從而避免了過濾。這是所提供的示例：

<pre>%E5%98%8A => U+560A => 0A</pre>

<p>
這非常重要，因為換行符在服務器上被解釋為這樣的東西，創建新的一行，服務器讀取並執行它，這裡是用於添加新的 Cookie。

<p>
現在，當 CRLF 攻擊允許 XSS 攻擊的時候（請見 XSS 一章），它們還會更加危險。這種情況下，由於 Twitter 的過濾器被繞過了，包含 XSS 攻擊的新的響應可能返回給用戶，這裡是 URL：

<pre>
https://twitter.com/login?redirect_after_login=https://twitter.com:21/%E5%98%8A
%E5%98%8Dcontent-type:text/html%E5%98%8A%E5%98%8Dlocation:%E5%98%8A%E5%98%8D
%E5%98%8A%E5%98%8D%E5%98%BCsvg/onload=alert%28innerHTML%28%29%E5%98%BE
</pre>

要注意`%E5%E98%8A`佈滿了這個 URL。如果我們使用了這些字符，並且實際添加了換行符，這個就是協議頭的樣子

<pre>
https://twitter.com/login?redirect_after_login=https://twitter.com:21/
content-type:text/html
location:%E5%98%BCsvg/onload=alert%28innerHTML%28%29%E5%98%BE
</pre>


你可以看到，換行符允許了創建新的協議頭，並和可執行的 JavaScript 一起返回：`svg/onload=alert(innerHTML)`。使用這個程式碼，惡意用戶就能夠盜取任何無防備的受害者的 Twitter 會話信息。

>重要結論
>
>好的攻擊是觀察與技巧的組合這裡，報告者`@filedescriptor`了解之前的 Firefox 編碼漏洞，它錯誤處理了編碼。對這個知識的了解就可以用於測試 
>Twitter 上相似的編碼來插入換行。
>
>當你尋找漏洞時，始終記住要解放思想，並提交編碼後的值來觀察站點如何處理輸入。


---

## **2. Shopify 響應分割**

```
難度：中

URL：v.shopify.com/last_shop?x.myshopify.com
報告鏈接：https://hackerone.com/reports/106427
報告日期：2015.12.22
獎金：$500
```
描述：

Shopify 包含了一些隱藏功能，會在你的瀏覽器上設置 Cookie，它指向你所登錄的最後一個商店。它通過終端`/last_shop?SITENAME.shopify.com`來實現。
<p>

在 2015 年 12 月，有人發現，Shopify 不驗證在調用中傳入的`shop`參數。所以，使用 Burp Suite，白帽子就能夠使用`%0d%0a`來修改請求，並生成協議頭返回給用戶。這裡是截圖：

![1](https://raw.githubusercontent.com/dyeat/Document_read/master/Web_Hacking_101/image/7-1.jpg)

這裡是惡意程式碼

```
%0d%0aContent-Length:%200%0d%0a%0d%0aHTTP/1.1%20200%20OK%0d%0aContent-Type:%20te\
xt/html%0d%0aContent-Length:%2019%0d%0a%0d%0a<html>deface</html>
```

這裡，`%20`表示空格，`%0d%0a`是 CRLF。所以瀏覽器收到了兩個協議頭，並渲染了第二個，它能夠導致很多漏洞，包括 XSS

>重要結論
>
>一定要尋找這樣的機會，其中站點接受你的輸入，並且將其用於返回協議頭的一部分。
>這裡，Shopify 使用last_shop值創建了 Cookie，它實際上可悲用戶克隆的 URL 參數污染。
>這是一個不錯的信號，它可能存在 CRLF 注入漏洞。


## **總結**
<p>

良好的攻擊是觀察和技巧的結合。了解如何使用編碼字符串來發現漏洞是一個不錯的技巧。 `%0D%0A`可以用於測試服務器，以及判斷他們是否存在 CRLF 漏洞。如果存在，進一步嘗試使用 XSS 注入來組合蓋漏洞（請見第七節）。
<p>

另一方面，如果服務器不響應`%0D%0A`，要考慮如何再次編碼這些字符，並測試服務器，以便觀察它是否解碼雙重編碼的字符，就像`@filedescriptor`所做的那樣。
<p>

一定要尋找這樣的機會，其中站點使用提交的值來返回一些類型的協議頭，例如創建 Cookie。