---
author: David
categories:
- Python
comments: true
date: "2018-07-21T00:00:00Z"
modified: "2018-07-21"
published: true
title: How to Import data to Elasticsearch (Nginx Access logs)
url: /技術簡介/系統建置/aws/python/2018/07/21/import-els.html
images: ["https://static001.infoq.cn/resource/image/48/06/48fb5f5d37a9efd331763e97c5368306.png"]
---

因為統計流量的需求，需要將nginx access log放進elasticsearch方便做統計。 <br />
對ELS不是很熟的我，也花了一點時間釐清觀念，就寫進來當作notes吧。 <br />
也希望對剛接觸ELS的朋友有一些幫助。<br />

<br />

## Preparation

- Elasticsearch Server == 6.3.0 (host by AWS)
- Python == 3.6
- elasticsearch python module == 6.3.0

<br />
<br />

## Step 1. Install Elasticsearch
---
Because we use ELS which is host by AWS, <br />
so basically we just need to take care of instance type and count.<br />

And why I use AWS ELS ?<br />
Because I just wanna get the rid of instance management lol<br />
<br />

### Intruction Elasticsearch Architecture
ELS is totally using JAVA to develop and of course it's open source.<br />
In other word, you can install it on your own machine if you want.<br />
btw, install by docker is also a good way on development environment.<br />

`docker pull elasticsearch` <br />
very simple and easy way to isntall for testing, but not refer to do this on production environment. (Don't pull it on Macbook it will be slow, I've tried it lol) <br />
<br />
![Alt text](http://www.uml.org.cn/bigdata/images/2018012633.jpg)

This is simple ELS structure and those Shard/Replica is ELS `node`. <br />

<br />
<br />

## Architecture
---
- Index <br />
  index is some kind of `Table` on RDBMS <br />

- Shard <br />
  All indeics on ELS will be split to a lot of shards <br />
  A shard can be a piece of all data, or a `REPLICA` (backup) of data <br />

<br />
okay, lets talk about what we should notice about AWS ELS

<br />
<br />

## AWS instance type 
---
  default type is `m4.large`, the smallest type is `t2.small` <br />
  but I think using `m4.large` for store about 1G data is too waste cuz `m4` really very expensive<br />
<br />
  In my case, I forgot to change the default type so I spent NTD$ 7000 for it, I changed the type to `t2.small` after I checked billing T_T <br />
  <br />
  so please change instance type if you don't need to much resouces.<br />
  
  <br />
  <br />

## Node count
---
  ELS is composed of a lot of nodes<br />
  And the `volume size` is decide of your node is `REPLICA` or `SHARD` <br />
  <br />
  What is `REPLICA` and `SHARD` ? <br />
  REPLICA is some kind of backup, it's totally a copy of your master. <br />
  so the volume size cannot be superimpose. <br />
   <br />
  In the other hand, every single shard is one of your master. <br />
  So the volume size is superimposed. <br />
  
  <br />
  <br />

## Notes
---
  AWS will install not only ELS but also `Kibana` into. <br />
  So we can query data from kibana without any additional setting. <br />

<br />
<br />

## Step 2. Import Access log in Python
---
```py
data_es = { 
    "message": line,
    "@timestamp": log_formated_time,
    "tag": log_tag
} 

es.index( index="imported", doc_type="imported-python", body=data_es )
```
<br />
<br />

The only one thing we need to take care is this request body and `es.index` to create a new index. <br />
And the format of time stamp is `%Y-%m-%dT%H:%M:%S.000+0800`, this rule must to be allow or it will cannot be imported. <br />
<br />
In my case, I create a new index to store all data I wanna import to. <br />
This index like a `table` on MySQL/MSSQL. <br />
So if you delete a index, the data insode will be delete in the same time.<br />

<br />
<br />

okay, lets take a look of complete code below. <br />
If you think it's too hard to read, can read it on my Github. <br />
<br />
https://github.com/davidh83110/import-accesslog-to-elasticsearch <br />

<br />
<br />

```py
ELS_ENDPOINT = 'localhost:9200'
```

```py
import endpoint
from elasticsearch import Elasticsearch
from datetime import datetime
import time
import re

es = Elasticsearch([endpoint.ELS_ENDPOINT])
log_tag = 'imported-python'

def time_to_format(log_timestamp):
    log_formated_time = time.strftime('%Y-%m-%dT%H:%M:%S.000+0800', time.localtime(log_timestamp))

    return log_formated_time

def clean_time(log_time):
    log_timestamp_change = datetime.strptime(log_time, '%d/%b/%Y:%H:%M:%S%z').timetuple()
    log_timezone = "".join(re.findall('\+\d\d\d\d', log_time))

    if log_timezone == '+0000':
        to_be_add_timestamp = 28800*2

    elif log_timezone == '+0800':
        to_be_add_timestamp = 28800

    log_timestamp = int(time.mktime(log_timestamp_change))+to_be_add_timestamp
    log_formated_time = time_to_format(log_timestamp)

    return log_formated_time
    
def handler(log_path):
    with open(log_path, 'r') as fp:
        for line in fp.readlines():
            try:
                log_time = str(re.findall('- \[.+\+\d\d\d\d', line)).replace('[', '').replace(']', '').replace('-', '').replace(' ', '').replace('\'', '')
                log_formated_time = clean_time(log_time)

                data_es = { 
                    "message": line,
                    "@timestamp": log_formated_time,
                    "tag": log_tag
                } 

                es.index( index="imported", doc_type="imported-python", body=data_es )
                
            except:
                print("import successed "+line)
        else:
            print("import failed ")
    fp.close()

if __name__ == '__main__':
    handler('/var/log/nginx/access.log')
    
```

<br />
<br />

Not Sure if this a good way to import data, but if you need still can use it. <br />
Welcome to e-mail me or leave some comments if anything wrong or need more details. <br />
<br />
Thanks for watching. <br />

<br />
<br />
<br />
<div style="text-align: right;">
2018-07-21 22:17 , David in Taipei</div>

<br />
<br />
<br />


