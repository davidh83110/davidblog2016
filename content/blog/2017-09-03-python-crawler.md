---
author: David
categories:
- Python
comments: true
date: "2017-09-03T00:00:00Z"
modified: "2017-09-03"
published: false
title: Python3 實作爬蟲
url: /雜七雜八/python/2017/08/31/docker-note.html
---
嗚 感冒好痛苦
<br />
心力交瘁的最近還是寫寫技術
<br />
Python實作爬蟲(crawler)
<br />
何謂爬蟲？ 其實就是擷取網頁而已
<br />
網頁擷取之後我們必須要過濾出我們要的資訊才算成功的爬蟲
<br />
這邊以 https://dinbendon.net 來作為範例<br />
目的為取出店家菜單<br />
<br />
Step1. 抓取網頁<br />
一般來說會使用requests的get方式來抓取網頁<br />
再以BeautifulSoup去抓取特定段落<br />
<br />
<br />
<br />
<br />
<br />

```
url = 'https://dinbendon.net/do/idine?shop=209534'
r = requests.get(url)
soup = BeautifulSoup(r.text, 'lxml')
print(soup)
```
<br />
會得到結果是網頁的前端code (省略)<br />
那前端前端code這麼多如何去找到我要的呢<br />
請善用瀏覽器 開發人員工具<br />
這邊我們找到價錢以及商品名稱是在tr裡的class=even以及class=odd這兩個段落<br />
<br />
將他抓取出來並放到MSSQL DB裡<br />
```
a = 0
    for i in soup.find_all('tr', {'class': 'even'}):
         c = i.text.split('\n')
         a += 1
         print(c)
         if a == 1:
            cursor.execute("""INSERT INTO  [dbo].[DIN_BEN_DON_PRODUCT] (Shop_ID , Product_Name , Product_Price) VALUES (%s , %s , %s) """,
                    (shop_id, c[2], c[5]))
            cnxn.commit()
            a = 0
            del c
    print('ok')

    a = 0
    for j in soup.find_all('tr' , {'class' : 'odd'}):
        c = j.text.split('\n')
        if '網友評價' not in c:
            if '地址' not in c :
                if '傳真' not in c:
                    if '店家詳細說明' not in c:
                        if '店家服務類型' not in c:
                            if '最後修改日' not in c:
                                a += 1
                                if a == 1:
                                    cursor.execute("""INSERT INTO  [dbo].[DIN_BEN_DON_PRODUCT] (Shop_ID , Product_Name , Product_Price) VALUES (%s , %s , %s) """,(shop_id, c[2], c[5]))
                                    cursor.execute("""DELETE FROM [dbo].[DIN_BEN_DON_PRODUCT] WHERE Product_Name = ''""")
                                    cnxn.commit()
                                    a = 0
                                    del c
    print('ok')
```

那抓取結果成功並存到DB的話就會print ok<br />
<br />
那店家資訊以此類推去抓取，重點依舊放在抓回來的資料處理<br />
可以發現大部分的動作都在處理資料切割<br />

```
def shop_info(url):
    r = requests.get(url)
    soup = BeautifulSoup(r.text, 'lxml')
    shop_content = soup.find_all('td' , {'class' : 'content'})
    columns = []

    for i in shop_content:
        columns.append(((i.text).replace('\n' , '')).replace('\t' , ''))

    info = []
    chose = (0 ,2, 3, 4, 6, 7)
    for j in chose:
        info.append(columns[j])
        print(info)
    cursor.execute("""INSERT INTO  [dbo].[DIN_BEN_DON] (Shop_ID , Shop_Name , Shop_Content , Shop_Address , Shop_TEL , Shop_City , Shop_Type) VALUES (%s , %s , %s , %s , %s , %s , %s )""",(shop_id, info[0], info[1], info[2], info[3], info[4], info[5]))
    cnxn.commit()
    #print(info[3])
    print("ok")
```
<br />
如果正確存到DB，會print ok <br />
<br />
附上完整code github上也有：<br />
```

import requests
from bs4 import BeautifulSoup
import pymssql
import re

cnxn = pymssql.connect(server='', user='',
                       password='',  database='')
cursor = cnxn.cursor()

url = 'https://dinbendon.net/do/idine?shop=209534'

print(url)
print(str(re.findall('shop=.+' , url)).replace('shop=' , ''))
shop_id = int(str(re.findall('shop=.+' , url)).replace('shop=' , '').replace('[\'' , '').replace('\']' , ''))
print(shop_id)

# product_name
def product_name_price(url):
    r = requests.get(url)
    soup = BeautifulSoup(r.text, 'lxml')
    print(url)
    print(soup)
    #### even
    a = 0
    for i in soup.find_all('tr', {'class': 'even'}):
         c = i.text.split('\n')
         a += 1
         print(c)
         if a == 1:
            cursor.execute("""INSERT INTO  [dbo].[DIN_BEN_DON_PRODUCT] (Shop_ID , Product_Name , Product_Price) VALUES (%s , %s , %s) """,
                    (shop_id, c[2], c[5]))
            cnxn.commit()
            a = 0
            del c
    print('ok')

    ### odd
    a = 0

    for j in soup.find_all('tr' , {'class' : 'odd'}):
        c = j.text.split('\n')
        if '網友評價' not in c:
            if '地址' not in c :
                if '傳真' not in c:
                    if '店家詳細說明' not in c:
                        if '店家服務類型' not in c:
                            if '最後修改日' not in c:
                                a += 1
                                if a == 1:
                                    cursor.execute("""INSERT INTO  [dbo].[DIN_BEN_DON_PRODUCT] (Shop_ID , Product_Name , Product_Price) VALUES (%s , %s , %s) """,(shop_id, c[2], c[5]))
                                    cursor.execute("""DELETE FROM [dbo].[DIN_BEN_DON_PRODUCT] WHERE Product_Name = ''""")
                                    cnxn.commit()
                                    a = 0
                                    del c
    print('ok')

# shop_info
def shop_info(url):
    r = requests.get(url)
    soup = BeautifulSoup(r.text, 'lxml')
    shop_content = soup.find_all('td' , {'class' : 'content'})
    columns = []

    for i in shop_content:
        columns.append(((i.text).replace('\n' , '')).replace('\t' , ''))

    info = []
    chose = (0 ,2, 3, 4, 6, 7)
    for j in chose:
        info.append(columns[j])
        print(info)
    cursor.execute("""INSERT INTO  [dbo].[DIN_BEN_DON] (Shop_ID , Shop_Name , Shop_Content , Shop_Address , Shop_TEL , Shop_City , Shop_Type) VALUES (%s , %s , %s , %s , %s , %s , %s )""",(shop_id, info[0], info[1], info[2], info[3], info[4], info[5]))
    cnxn.commit()
    #print(info[3])
    print("ok")


if __name__ == '__main__':
    # search = input("Enter the restaurant URL you want to search: ")
    # search = url
    # product_name_price(url)
    shop_info(url)

```
<br />
<div style="text-align:right">2017-09-03 19:50 , David in Taipei</div><br />
