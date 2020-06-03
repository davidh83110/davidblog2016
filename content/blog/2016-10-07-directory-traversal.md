---
author: David
categories:
- Security
- OWASP
comments: true
date: "2016-10-07T00:00:00Z"
modified: "2016-10-02"
published: true
title: Directory Traversal - 跨目錄/檔案穿越攻擊 in OWASP
url: /資訊安全/駭客技術/owasp top10/2016/10/07/directory-traversal.html
---

<div class='post-body entry-content' id='post-body-2638476067227899869' itemprop='description articleBody'>
Directory Traversal指的是我們可以在網頁的URL上使用 .. ../.. cat /etc/passwd等探索目錄的行為&#12290;<br />
<br />
很容易理解的一個簡單漏洞&#65292;而會發生這個漏洞原因不外乎是當初設計網站的人忘了將目錄探索功能關閉(像是Apache的FollowSymLinks Deny all;)&#65292;導致惡意人士可以透過輸入網址的方式來瀏覽web端主機內容&#65292;甚至取得全部主機資料&#12289;網站資料&#12290;<br />
<br />
我們今天利用OWASP裡的bWAPP來熟悉這個漏洞&#12290;<br />
<br />
<br />
Directory Traversal - Directory<br />
<br />
先來熟悉目錄穿越的部分&#12290;<br />
<a href="http://1.bp.blogspot.com/-ouIiTDxc8vQ/V_ELuoX3v1I/AAAAAAAAA4U/Msg7eNEIZEo2IILtaUlECCp-pjuzyVj0wCK4B/s1600/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.29.10.png" imageanchor="1"><img border="0" height="200" src="https://1.bp.blogspot.com/-ouIiTDxc8vQ/V_ELuoX3v1I/AAAAAAAAA4U/Msg7eNEIZEo2IILtaUlECCp-pjuzyVj0wCK4B/s320/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.29.10.png" width="320" /></a><br />
一開始我們會看到這是一個可以下載檔案的網頁&#65292;那我們試試在網址上做些修改&#65292;看是否能列出目錄&#12290;<br />
<br />
http://172.20.10.5/bWAPP/directory_traversal_2.php?<span style="color: red;">directory=documents</span><br />
<br />
上列是架在我主機上的OWASP具有Directory Traversal漏洞的網頁之網址<br />
我們可以看到網址後面已經告訴你這一頁的目錄資料夾叫做documents<br />
<br />
那我們試試在後面加上" .. " (為上一層目錄的意思)&#65292;看能否顯示出上一層目錄有什麼&#12290;<br />
http://172.20.10.5/bWAPP/directory_traversal_2.php?<span style="color: red;">directory=documents/..</span><br />
<span style="color: red;"><br /></span>
<span style="color: red;"><a href="http://2.bp.blogspot.com/-hTnrif8rt5s/V_EMvGfOedI/AAAAAAAAA4g/pfC0Fp8UenEEMKemCIfXsC0QuRrsBNhCACK4B/s1600/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.33.38.png" imageanchor="1"><img border="0" height="200" src="https://2.bp.blogspot.com/-hTnrif8rt5s/V_EMvGfOedI/AAAAAAAAA4g/pfC0Fp8UenEEMKemCIfXsC0QuRrsBNhCACK4B/s320/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.33.38.png" width="320" /></a></span><br />
<span style="color: red;"><br /></span>
有發現嗎&#65311; 跟剛剛顯示的目錄內容不一樣了&#65292;代表我們已經成功存取上一層的資料&#12290;<br />
藉由這樣探索的方式&#65292;駭客可以將整個系統的架構摸透&#65292;以尋找後門攻擊或將整個主機資料download下來&#12290;<br />
<br />
那既然可以使用.. 這種相對目錄的寫法&#65292;是不是我們可以使用絕對路徑的寫法&#65311;<br />
<br />
http://172.20.10.5/bWAPP/directory_traversal_2.php?directory=documents<span style="color: red;">/../../../../../etc/passwd</span><br />
<br />
嗯...看來不行&#12290; 這邊使用很多..的意思是強迫回到最前面的/(根目錄)之前的意思&#12290;<br />
<br />
<br />
<br />
那跨檔案攻擊的漏洞是另一個 (Directory Traversal - Files)&#12290;<br />
<br />
<a href="http://4.bp.blogspot.com/-maEJi9UDm1k/V_EPkx-xRQI/AAAAAAAAA4s/7eACe_Z6pSUm-v07LtuBGte9uq0mZtArACK4B/s1600/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.45.40.png" imageanchor="1"><img border="0" height="200" src="https://4.bp.blogspot.com/-maEJi9UDm1k/V_EPkx-xRQI/AAAAAAAAA4s/7eACe_Z6pSUm-v07LtuBGte9uq0mZtArACK4B/s320/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.45.40.png" width="320" /></a><br />
<br />
可以看到網頁內什麼都沒有&#65292;就是叫我們去爬....<br />
那我們試試是否可以爬到系統帳號檔案 (/etc/passwd)<br />
<br />
http://172.20.10.5/bWAPP/directory_traversal_1.php?page=<span style="color: red;">message.txt</span><br />
<br />
以上是網址&#65292;可以看到我們瀏覽的網頁正是message.txt這個檔案所show出來的<br />
那我們把message.txt改成/etc/passwd呢&#65311;<br />
http://172.20.10.5/bWAPP/directory_traversal_1.php?page=<span style="color: red;">/etc/passwd</span><br />
<span style="color: red;"><a href="http://1.bp.blogspot.com/-jFUrRpSZ5II/V_ERfdTIaKI/AAAAAAAAA44/MCadeYz2UgE_HO1dhwt9lz-6kWgvgp_lgCK4B/s1600/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.53.53.png" imageanchor="1"><img border="0" height="200" src="https://1.bp.blogspot.com/-jFUrRpSZ5II/V_ERfdTIaKI/AAAAAAAAA44/MCadeYz2UgE_HO1dhwt9lz-6kWgvgp_lgCK4B/s320/%25E8%259E%25A2%25E5%25B9%2595%25E5%25BF%25AB%25E7%2585%25A7%2B2016-10-02%2B%25E4%25B8%258B%25E5%258D%25889.53.53.png" width="320" /></a></span><br />
<br />
成功&#65281; 我們成功讀取到系統所有使用者的資料了&#12290;<br />
<br />
以上就是Directory Traversal的攻擊手法&#65292;一般來說只要控管好前端Web瀏覽權限&#65292;都可以防範這樣的攻擊&#12290;<br />
這樣的攻擊就是利用網管的疏忽而已&#12290;<br />
<br />
有任何問題或錯誤觀念歡迎來信&#65292;謝謝&#12290;<br />
<br />
<br />
<br />
<br />
<div style="text-align: right;">
2016-10-02 22:06 , David in Taipei</div>
<br />
<span style="color: red;"><br /></span>
<br />
<br />
<div style='clear: both;'></div>
</div>