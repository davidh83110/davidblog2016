---
author: David
categories:
- jibber-jabber
comments: true
date: "2018-09-25T00:00:00Z"
modified: "2018-09-25"
published: true
title: MacOS Mojave Upgrade Issues and Solutions / MacOS Mojave 更新災情及解決方法
url: /技術簡介/2018/09/25/macos-mojave-bugs.html
images: ["http://farm2.staticflickr.com/1913/43093400610_b4941d7a83_b.jpg"]
---

MacOS Mojave 於今日2018/09/25正式上線提供正式版給全球用戶下載

我就打了頭陣搶先更新，沒想到更新完遇到一連串的問題

慶幸的是有找到解決的方法，就讓我來分享一下吧！
<br />
<br />
先說整個更新的過程，花費約30min。

更新之後原本可以選的暗色系狀態欄已經消失了，只能選淺色或者暗色主題

而這個暗色，一開始會很不習慣，因為實在是太黑了！！

有多黑呢？

![Alt text](http://farm2.staticflickr.com/1960/44856266682_357b9002bf_b.jpg)

大概就是這樣

而更新後出現的問題我整理一下，大概如下面幾點。
<br />
<br />
<br />

1. 系統字體渲染效果很模糊也很細，中文更加嚴重，很像windows的字體<br />
    是真的會模糊到看不清楚那種，解法如下：<br />
    打開終端機(Option+Space打開Spotlight搜尋 -> 輸入Terminal -> Enter)，輸入下列指令：<br />
    `defaults write -g CGFontRenderingFontSmoothingDisabled -bool NO`
<br />
    輸入完按Enter後把終端機關掉，登出系統再登入，字體就會恢復正常。
    <br />
    <br />
    
2. git 會壞掉  
    ![Alt text](http://farm2.staticflickr.com/1958/44186441644_ddf0de2b4c_b.jpg)

    原因是xcode需要安裝一些東西，在Terminal執行下列指令即可：<br />
    `xcode-select --install`<br />
    git 就會復活了！！<br /><br />

3. homebrew也壞掉
    ![Alt text](http://farm2.staticflickr.com/1951/44906414701_416be9819a_b.jpg)<br />
    因為在MacOS裡，xcode是很重要的東西，dependency很重，所以連帶會有問題<br />
    解答他也給你了，權限問題，執行下列指令即可<br />
    `sudo xcodebuild -license accept`<br /><br />

4. bashrc清空
    眼尖的朋友就會發現，我的command line提示列是default狀態，因為被他媽的清空了
    
    幹，還我alias阿～～～
    
    這問題我記得每次大改版都會清掉，就只能重寫囉
    
    用zshrc / iTerm的朋友應該不會遇到這問題，所以不用擔心
    
<br />
    
目前已知的問題大概是這樣，如果有遇到其他問題可以在下方留言幫忙補充。

<br />
### 感謝補充：更新MacOS之前，先去AppStore更新Terminal以及XCode就可以避免所有問題囉（除了字體）


<br />
問題講完了，講講這次改版的特色

1. 當然是暗色主題，習慣後其實還挺不錯的，對眼睛也比較舒服

    基本上只有系統內建的會是暗色，像是Chrome/Line這一些額外安裝的就沒有了。

    ![Alt text](http://farm2.staticflickr.com/1945/31033483818_45bdf34624_b.jpg)


2. 桌面堆疊整理，這個其實滿實用的，看起來乾淨很多 (桌面點右鍵 -> 堆疊整理)


3. 螢幕錄影，截圖工具 (CMD + Shift + 5)<br />

    這個其實也挺不錯的，有點像iPhone的截圖方式。<br />
    
    ![Alt text](http://farm2.staticflickr.com/1919/44186640124_c3ab396e0e_b.jpg)


其他就是AppStore變得更快了，介面也更新。  
整體來說，改版後似乎速度有加快那麼一點點，但看得出來其實存在一些bug  
如果不想嚐新的人，其實可以不用急著更新，沒太大的改變  

順帶一提，軟體相容性也不錯，沒有太大問題。

<br />
<br />
因為弄了一整個下午，所以久違發文紀念過程，幹。


<br />
<div style="text-align: right;">
2018-09-25 20:40 , David in Taipei</div>

<br />
<br />
<br />



