---
published: true
layout: post
title: MongoDB 簡介與Replica Set架設以及常見雷點 / MongoDB Introduce ,Replica Set Setup and Error Soultions
author: David
modified: 2018-12-01
categories: [系統建置, 技術簡介]

  
comments: true
---

![Alt text](https://cdn-images-1.medium.com/max/2000/1*Ce0gUe0LbnhL7ebnDGTp5w.png)
<br />
由於敝司 Shopline 使用的 Mongodb 的 Shard 以及 Configsvr 版本不一致，此次升級主要會將版本一致升級到3.2, 並全部採用 Replica Set.

踩過的雷很多，該碰到的問題也都碰到了，卻只花了短短的一個多禮拜，果然壓力使人迅速成長，也見證了一句話，「所謂大神，不過就是雷踩得比較多」。

MongoDB 為近年來很火紅的No SQL Databese, 在新版本中也快採用近年多數資料庫使用的 Replica Set (Primary and Sceondary) 架構, 取代舊有的 Shard Cluster (Master and Slave). 進而增加許多優勢，像是自動故障恢復、讀寫控制等等。

本篇文章以 mongodb-org 3.0 以及 3.2 為主, 這兩個版本最大的差異就是在 Replica Set.

升級上官方也提供非常詳盡的documents, 不得不稱讚mongodb的文件，真的很完整。
<br />
<br />

This post is mainly for describing the processes and experiences to upgrade Mongodb from 3.0 to 3.2 on Shopline which is our company.

There's a big difference of Mongodb 3.0 and 3.2, is the Replica Set. 

Replica Set is a is a new structure of databases, using Primary and Secondary instead of Master and Slave and this structure has a lot of advantages such like auto recovery from an unexpected shutdown and split read and write nodes.

<br />
<br />

---

## Environment

Mongodb-org 3.0 -> 3.2 <br />
Amazon Linux 1 

<br />
<br />

---

## MongoDB Introduction
<br />
<br />

### Main Structure
![Alt text](http://blogs.bmc.com/wp-content/uploads/2017/10/shard-cluster-full.png)

`Shard`: The place where is to store data. 

`Replica Set`: Normally it will be a primary, secondary and a arbiter in a replica set, whatever shard server or config server.

`Config Server`: The bridge which is mongos router communicate with shard server.

`Mongos`: A mongo client agent and shell, usually install with application.
<br />
<br />

MongoDB 主要的架構大概就是這樣，會有Shard/Configsvr/mongos

一般來說Shard會存放資料，mongos是client端，負責去連接資料庫，而configsvr就是那座橋樑，讓mongos client知道該怎麼走到正確的shard去讀寫資料。

<br />
<br />

### Shard / Configsvr Replica Set

![Alt text](https://www.simplilearn.com/ice9/free_resources_article_thumb/Replica-Sets-Members.JPG)

`Primary`: The controller of a Shard, receive all requests to a database when writing and reading, it is the only one can be read and write node in a normal situation. Also has the responsibility to store data into a database.

`Secondary`: Normally secondary doesn't allow to receive any request, the only job is to replicate primary's data to store in another place. And can be promoted to a primary node when the original primary is down in unexpected or necessary.

`Arbiter`: The main function is to allocate primary and secondary nodes, it will use "vote" to decide which secondary will be promoted to primary when needed. Arbiter won't store any data inside.

`Heartbeat`: No need to explain, health check.
<br />
<br />

這是一般的 Shard Replica Set 架構，通常包含 Primary / Secondary / Arbiter.

Primary 主要接收所有讀與寫的請求以及儲存資料，Secondary 一般來說不會接受任何讀寫請求，只做一件事，就是複製 Primary 的資料來儲存，以便在必要或意外時可以被提升為 Primary 節點。

而 Arbiter 不會存儲任何資料，主要功用是調配 Primary and Secondary，當 Primary Down的時候，可以投票決定哪一個 Secondary 將被升級為 Primary。

<br />
<br />

### This is the Configsvr replica set structure.
![Alt text](https://docs.mongodb.com/manual/_images/replica-set-primary-with-two-secondaries.bakedsvg.svg)

In confinsvr replica set, the only difference is no Arbiter.
<br />
<br />
<br />

---

## Install MongoDB-org 3.0 on Amazon Linux
<br />

`yum` doesn't have mongodb-org inside the package pools, we have to add repo manually.

```
cat >> /etc/yum.repos.d/mongodb-org-3.0.repo <<EOF
[mongodb-org-3.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2013.03/mongodb-org/3.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.0.asc
EOF
```
Then
`> yum install mongodb-org -y`
<br />

If we wanna change to another version, just modify the value on repo file.
<br />
<br />
<br />

---

## Setup a Replica Set of a complete MongoDB
<br />

`Shard` : 3 mongod servers, usually use `27018` port (version 3.2)<br />
`Configsvr` : 3 mongod servers, usually use `27019` port<br />
`mongos `: 1 mongo-shell, usually use `27017` mongodb default route port<br />
<br />
Please use the same version, mongodb-org 3.0 in all servers.<br />
<br />
<br />
<br />

---

## Shard Servers config
<br />

In 3 shard servers, please use the same config file.


mongod-32.conf
```
## mongod-32.conf

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# Where and how to store data.
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile

# network interfaces
net:
  port: 27018
  bindIp: 0.0.0.0  # Listen to local interface only, comment to listen on all interfaces.

replication:
  replSetName: shard1
sharding:
  clusterRole: shardsvr
```
<br />
<br />

---

## Start your mongod
<br />

`service mongod start` or `mongod -f /etc/mongod.conf` to start mongod servers.<br />
<br />
<br />

Feel free to use `ps aux | grep mongo` or `service mongod status` to check if your mongod really started.<br />

If you get any issues when starting mongod, please go to `/var/log/mongod.log` to check error messages. <br />
<br />

** And please notice the permissions of those directories which is define in mongod.conf.<br />
<br />
<br />
請使用上述啟動指令來啟動你的 mongodb，並檢查他是否真的被啟動了。如果再啟動過程出現 Failed，請到 log 目錄檢查錯誤。常常因為目錄權限問題而失敗，或者是目錄不存在。請小心檢查，若不存在請建立目錄並給予權限。
<br />
<br />
<br />

---


## Setup Shard Replica mode
<br />
<br />

After mongodb is started, please use mongo-shell to configure our shard to a replica set.<br />

Let's do it !!!  The commands are as following below.<br />

shard - primary : `> mongo localhost:27018`
```
> mongo> rs.initiate()  // to enable replica mode
> mongo> rs.status()  // check the replica mode if really enabled
> mongo> rs.add({host: "SECONDARY-IP:27018", priority: 0.5})  // add secondary to replica set
> mongo> rs.addArb("ARBITER-IP:27018")  // add arbiter to replica set
```
<br />
<br />
Congratulations !! Shard Replica Set has already set up successfully if you see primary, secondary, arbiter when you execute `rs.status()`.

```
shard1:PRIMARY> rs.status()
{
	"set" : "shard1",
    ...
	"members" : [
		{
			"_id" : 5,
			"name" : "ip-10-0-0-57:27018",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
            ...
		},
		{
			"_id" : 8,
			"name" : "ip-10-0-0-166:27018",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
	        ...
		},
		{
			"_id" : 9,
			"name" : "ip-10-0-0-122:27018",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			...
		}
	],
	"ok" : 1
}
```
<br />
<br />
<br />

---

## Set up a mongos client agent
<br />

Actually, mongos is mongo-shell, one of a package of mongodb-org.
So when we install mongodb-org, mongos was be install as well.
<br />

mongos.conf
```
#config servers
configdb=CONFIGSVR-IP-1,CONFIGSVR-IP-2,CONFIGSVR-IP-3
#where to log
logpath=/var/log/mongo
#log overwritten or appended to
logappend=true
#fork and run in background
fork=true
#override port
port=27017
```

`> mongos -f /etc/mongod.conf` to start mongos and keep it right there.
<br />
<br />

we will use mongos to configure configsvr later.
<br />
<br />
<br />

---

## Set up configsvr 
<br />

The steps of installation configsvr are the same with shards.<br />

But the config file is different, please set up as below.<br />
<br />

```
# mongod.conf
#where to log
logpath=/mnt/log/mongodb/mongod.log
logappend=true
configsvr=true
# fork and run in background
fork=true
#port=27019
dbpath=/mnt/mongo
# location of pidfile
pidfilepath=/var/run/mongodb/mongod.pid
```

Please running 3 configsvr with the same config file.
<br />
<br />

And go inside with mongo-shell into `mongos` which we just set up.
`> mongo` // default use 27017, so no need to assign port.
<br />

The steps are as following: 
1. add shard with primary
2. create db and collection
3. insert data to database.collection
4. create index for collection
5. enable sharding
6. shard collection
<br />
<br />

In mongos
```
mongo> sh.addShard("shard1/PRIMARY-SHARD-IP:27018")   //add one of shard servers, then others will be add automatically
mongo> use test   //to create a test database
mongo> db.createCollection("users")   //create a collection on test db
mongo> db.users.insert({"name": "david", "sex": "male"})
mongo> db.users.ensureIndex({_id: 1}, {background: true})
mongo> sh.enableSharding("test.users")   //"test.users" = "<database>.<collection>"
mongo> sh.shardCollection("test.users", {_id: "hashed"})   //use hashed to shard collection
mongo> sh.status()   //to check if sharding enabled
```
<br />

Then `mongo> db.users.find()` to find data if exist
And please insert another data to test whether succeed.
<br />
<br />
<br />

---

## Upgrade MongoDB from 3.0 to 3.2

If wanna upgrade configsvr to 3.2, we have some notes to be remember.<br /><br />

3.0 and 3.2 use different `storage engine`<br />
    3.0 -> MMAPv1<br />
    3.2 -> wiredTiger<br />
<br />
<br />
Cluster mode and Replica set<br />
    3.0 -> cluster mode, sccc<br />
    3.2 -> replica set, csrs<br />
<br />
<br />

So, the step is as following below.<br />
<br />

a. disable balancer with `sh.stopBalancer()` and change one of three 3.0 configsvr version to 3.2 and set `configsvrMode: sccc` and `storage engine: mmapv1`
<br />

=> now 3 configsvr, one for 3.2 and two for 3.0<br />
<br />
<br />

b. use mongo-shell into 3.2 that one to initiate rs mode. `rs.initiate()`<br />
<br />
=> waiting for the data synced to 3.2 that one.<br />
<br />
<br />

c. launch another 3 configsvr with 3.2 mongod and start with `wiredTiger` storagr engine.<br />
<br />
<br />

d. go into the `mmapv1` that one and `rs.add("config1/IP")` to add those 3 new servers.<br />
<br />
=> now we have four 3.2 configsvr and two 3.0.<br />
<br />
<br />

e. `rs.stepDown()` in `mmapv1` that 3.2 one and primary node will change to another server.<br />
<br />
<br />

f. go to new primary and `rs.remove("NNAPv1-OLD-PRIMARY")` and enable balancer back with `rs.enableBalancer()`<br />
<br />
<br />

g. shut down old two 3.0 configsvr<br />
<br />
<br />

h. using 3.2 mongos to connect to new replica set configsvr and check status.<br />
<br />
<br />
<br />


---


## The Errors I Encountered when Upgrading to 3.2
<br />

a. `Can 3.0 mongos connect to 3.2 configsvr ?`<br />
    Absolutely not, 3.0 doesn't support replica mode<br />
    <br />
    mongos 3.0 是不允許連接到 3.2 或以上版本的 configsvr的。<br />
    <br />
    <br />

b. `"setShardVersion failed" when mongos configDB endpoint has been changed`<br />
```
mongos> db.users.find()
Error: error: {
    "$err" : "setShardVersion failed shard: shard1:shard1/ip-10-0-0-122:27018,ip-10-0-0-166:27018 { configdb: { stored: \"10.0.0.45:27019,10.0.0.45:27020,10.0.0.45:27021\", given: \"ip-10-0-0-45:27019,ip-10-0-0-45:27020,ip-10-0-0-45:27021\" }, ok: 0.0, errmsg: \"mongos specified a different config database string : stored : 10.0.0.45:27019,10.0.0.45:27020,10.0.0.45:27021 vs given : ip-10-0-0-45:27019,ip-10-0-0...\", $gleStats: { lastOpTime: Timestamp 0|0, electionId: ObjectId('5bffb069a4b240083f2691cb') } }",
    "code" : 10429,
    "shard" : "shard1"
}
```
<br />
cause the shard servers will cache mongos `configDB string`.<br />
If you wanna solve this issue, just `restart shard servers` then everything works well.<br />
<br />
因為 shard server 會 cache mongos 的 configDB string，所以如果遇到這個問題，重啟 Shard servers 就可以解決了。（記得先 StepDown Primary 再重啟）<br />
<br />
<br />

c. `Can 3.0 configsvr connect to 3.2 or 3.4up shard servers ?`<br />
    Absolutely the answer is NO.<br />
<br />
    3.0 的 configsvr 是不能連接到 3.0 以上版本的 Shard Server 的，因為不支援 Replica Mode。<br />
<br />
<br />

d. `How to separate data to two shards equally ?`<br />
    when `shardCollection()`, please use "hashed" on index, then will be equally separated.<br />
    <br />
    使用 hashed 雜湊就能使資料平均分配在不同的 Shard Server。<br />
    <br />
    <br />

e. `Can 3.0 mongod upgrade to 3.4 and skip 3.2 directly ?`<br />
    Sorry, the answer is NO, you must to upgrade to 3.2 first then go to 3.4.<br />
    <br />
    mongodb 不允許跳版本升級，3.0 只允許升級到 3.2 再到 3.4。<br />
    <br />
    <br />

f. `How to STOP a shard server safely ?`<br />
    Please use `rs.stepDown(120)` and `db.shutdownServer()` in mongo-shell.<br />
    Then exit mongo-shell and stop or kill mongod process.<br />
    <br />
    使用正確的指令停止一台 mongodb server。<br />
    <br />
    <br />

g. `How to change priority on Shard server ?`<br />
    ```
    var cfg = rs.conf()
    cfg.members[ID].priority = 0.5
    rs.reconfig(cfg)
    ```
    // The ID can use `rs.status()` to lookup
    <br />
    使用上列指令去更改 mongodb 的優先度。<br />
    <br />
    <br />
    
h. `How to failover the primary temporary ?`<br />
    `rs.stepDown(120)` on Primary node // The 120 is the Time by second
    Then it will auto failover to another node.<br />
    <br />
    暫時的停止一台 Shard Server 的運作，也可以運來測試主副節點切換是否正常運作。<br />
    <br />
    <br />
    
---

<br />
<br />

以上就寫到這裡了，如果遇到其他的問題歡迎在下面留言或來信發問。<br />
對於過程也歡迎提出意見，謝謝。<br />
<br />
<br />

Feel free to mail me or leave some commemts if you have any questions.<br />
I will reply all questions if I can help. <br />
<br />
<br />
<br />
Thanks for reading.










<br />
<br />
<br />
<div style="text-align: right;">
2018-12-01 04:05 , David in Taipei</div>

<br />
<br />
<br />