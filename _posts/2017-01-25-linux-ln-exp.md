---
published: true
layout: post
title: Linux指令 - ln 慘痛的教訓
author: David
modified: 2017-01-25
categories: [技術簡介 , 雜七雜八]
tags: 
  - linuxsst
  - centos
  - redhat
  - ln
  - rm
  - unlink
comments: true
---
用慘痛教訓換來經驗<br />
常常使用Linux做為系統的工程師應該都不陌生ln這個指令<br />
<span style="color: red;">ln是對資料夾建立超連結的指令</span><br />
<br />
以下範例<br />
<br />
root@David-MacBook: ~ -&gt; cd /tmp<br />
root@David-MacBook: tmp -&gt; &nbsp;ln -s /opt/web/home/<br />
<div class="p2">
<span class="s3"><br /></span>
<span class="s3"><br /></span>
第一行cd切換目錄到/tmp<br />
第二行是建立超連結到當前目錄（/tmp）並指向/opt/web/home<br />
<br />
會得到結果<br />
<br />
<br />
root@David-MacBook: tmp -&gt; ll<br />
total 0<br />
lrwxr-xr-x 1 root wheel 14 Jan 25 22:59 home -&gt;/opt/web/home</div>
<div class="p2">
<br />
<br />
可以看到我們將home這個資料夾成功建立超連結到/tmp<br />
<br />
<br />
<br />
<b><span style="color: red;">那什麼是超連結？</span></b><br />
所謂超連結就是<span style="color: red;">類似windows作業系統的"捷徑"</span>，可以藉由超連結的資料夾讀取與原資料夾資料同步的資料<br />
很難理解嗎？ &nbsp; 其實就是<span style="color: red;">資料夾的影分身</span>這麼簡單而已，主體是同一個。<br />
<br />
<br />
<br />
那ln這麼簡單的指令會出現哪些問題？ 問題可大了<br />
魔鬼藏在細節裡<br />
<br />
<br />
<br />
當你要解除超連結，你會怎麼下指令？ rm吧<br />
對 就是rm，<span style="color: red;">反正直接移除超連結跟windows移除捷徑一樣，真正的資料夾是不會影響的吧</span><br />
<br />
錯！ 大錯特錯！！<br />
<br />
root@David-MacBook: tmp -&gt; ll home/<br />
total 0<br />
-rw-r--r-- 1 root whell 0 Jan 25 23:13 test<br />
root@David-MacBook: tmp -&gt; rm -rf home/<br />
root@David-MacBook: tmp -&gt; ll<br />
total 0<br />
lrwxr-xr-x 1 root wheel 14 Jan 25 22:59 home -&gt; /opt/web/home<br />
root@David-MacBook: tmp -&gt; ll home/<br />
<br />
ls: home/: No such file or directory<br />
<br />
發現了嗎？<br />
<br />
<br />
<span style="color: red;">當你下rm -rf home/ &nbsp; 超連結的資料夾並未真正被刪除</span><br />
<span style="color: red;">但是，裡面的資料卻全部都被刪除了</span><br />
這是為什麼？ 因為超連結被linux視為檔案的一種<br />
而你參數-r 代表遞迴，所以他會將你資料夾內檔案遞迴刪除，但資料夾本身不會被刪除。<br />
<br />
這很嚴重，而且難以發現。<br />
<br />
<br />
<br />
<br />
<br />
那正確的指令下法呢？<br />
<br />
root@David-MacBook: tmp -&gt; ll home/<br />
total 0<br />
-rw-r--r-- 1 root wheel 0 Jan 25 23:18 test<br />
root@David-MacBook: tmp -&gt; rm -rf home<br />
root@David-MacBook: tmp -&gt; ll<br />
total<br />
<br />
<br />
看出差別了嗎？<br />
<br />
<span style="color: red;">只差了一個/</span><br />
<span style="color: red;"><br /></span>
<span style="color: red;">如果你下rm -rf 目錄/ ，便會遞迴刪除裡面的檔案</span><br />
<span style="color: red;">但如果是rm -rf 目錄 ，便可以成功移除超連結</span><br />
<span style="color: red;"><br /></span>
<span style="color: red;">或是使用unlink也可以成功移除超連結</span><br />
<br />
<br />
<br />
今天上班就不小心犯了這個很低級的失誤，幸好資料有備份，不然這個年大概不太好過了。<br />
<br />
<br />
<br />
<br />
重點整理：<br />
<br />
我使用的環境是CentOS<br />
<br />
ln -s 目錄 &nbsp; &nbsp;是建立超連結<br />
<span style="color: red;">移除超連結請務必將/去掉，不然會變成遞迴刪除資料夾裡檔案</span><br />
<br />
CentOS 5的版本如果你加上/會提示你無法刪除<br />
但CentOS 6 如果你加上斜線/就真的都把檔案刪除了<br />
<br />
使用上請務必小心，<span style="color: red;">或是改用unlink也較為安全</span><br />
<br />
慘痛的教訓，希望自己不要再犯了，真的很該死。<br />
<br />
<br />
<br />
<br />
<br />
<div style="text-align: right;">
2017-01-25 &nbsp;11:25 &nbsp;David in Tainan<br />
<br />
<br />
<br /></div>
<style type="text/css">
p.p1 {margin: 0.0px 0.0px 0.0px 0.0px; font: 12.0px f4; back00; back0, 0, 0, 0.85)}
span.s1 {font-variant-ligatures: no-common-ligatures}
</style>


<style type="text/css">
p.p1 {margin: 0.0px 0.0px 0.0px 0.0px; font: 12.0px 21; back00; back0, 0, 0, 0.85)}
p.p2 {margin: 0.0px 0.0px 0.0px 0.0px; font: 12.0px f4; back00; back0, 0, 0, 0.85)}
span.s1 {font-variant-ligatures: no-common-liga1e}
span.s2 {font-variant-ligatures: no-common-ligaf4}
span.s3 {font-variant-ligatures: no-common-ligatures}
</style><style type="text/css">
p.p1 {margin: 0.0px 0.0px 0.0px 0.0px; font: 12.0px 21; back00; back0, 0, 0, 0.85)}
p.p2 {margin: 0.0px 0.0px 0.0px 0.0px; font: 12.0px f4; back00; back0, 0, 0, 0.85)}
span.s1 {font-variant-ligatures: no-common-liga1e}
span.s2 {font-variant-ligatures: no-common-ligaf4}
span.s3 {font-variant-ligatures: no-common-ligatures}
span.s4 {font-variant-ligatures: no-common-liga21}
</style></div>
