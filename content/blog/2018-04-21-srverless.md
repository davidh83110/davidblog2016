---
author: David
categories:
- AWS
- Serverless
comments: true
date: "2018-04-21T00:00:00Z"
modified: "2018-04-21"
published: true
title: 20180329 Serverless event NOTE
url: /技術簡介/aws/serverless/2018/04/21/srverless.html
images: ["https://day.ithome.com.tw/serverless/img/fb.png"]
---

**Serverless for Monitoring**
-

how to debug on serverless ? <br />
how's the security for using serverless ? <br />
hoe's the latency ? <br />
how's the cost ? <br />


**_"Less is More"_** <br />
we have to focus on other things not just only code when using serverless

<br />

Structure is Changing
-

During the application structure is changing to CaaS, we have to reduce the instances which are not for application layer as possible as we can, to avoid launching more and more instances which unnecessary to save cost.
<br />
**"Switching those instances to serverless is a good solution"**

<br />

What is Serverless
-
- Event driven
- No needs to hosting infrasturature by ourself
- High Reliable (hosting by third party)
- Launch by crontab not always online
- But not all of services suit serverless

<br />

What is the benefit for using Serverless
-
- Enchance effectivence
- No needs to manage instances including scaling
- Cost down (if you use it on correct way)
- Micro services

<br />

What is the shortcomings for using Serverless
-
- Lang (Node.js/Python/Go/C#..... but no Ruby!!)
- How to test, debug and collect log ?
- How to monitor your Lambda monitoring functions ?
- How to CI/CD and version control ?
- Should not launch for long time
- Depends on third party services (e.g. It's too difficult from using AWS Lambda to Azure Functions)
- Performance (high cold start time)

<br />

**"You buy a car just like using EC2, rent a car just like using Lambda"**

<br />


When should we using Serverless 
-
- lower requests count
- high networking traffic but in very short time 
(In EC2 case, we need to scale out first or auto scaling, but auto scaling is not fast at all to againest short time traffic)


-> Push notification to user <br />
-> IoT <br />
-> Data Analyze <br />
-> Chatbot <br />
-> Monitoring <br />

<br />

Monitoring by Serverless
-
- Latency (Domain name lookup / TCP TLS handshake)
- cURL is a excellent tool

![](https://i.imgur.com/0oVk4Vy.png)

![](https://i.imgur.com/TiNuUqz.png)


<br />

Common use case
-
![](https://i.imgur.com/3YIZLho.png)


![](https://i.imgur.com/f1UhVJU.png)


![](https://i.imgur.com/u2cbiTf.png)


![](https://i.imgur.com/EJlDq8n.png)


![](https://i.imgur.com/r0Gqqlc.png)


![](https://i.imgur.com/9sqkEO1.png)


![](https://i.imgur.com/r5oOfRr.png)


![](https://i.imgur.com/UYw6R8X.png)


![](https://i.imgur.com/xYVRnFi.png)



<br />

What can we do or improve something ?
-
- Nginx access log & Application production log -> EFK
- Nginx error log (log level: warn) -> cloudwatch -> Lambda -> Slack
- Don't Stream Access logs and others which Lambda doesn't need to save money
- Build Slack slash command for request count/latency/4xx error...etc count
- Slack App for alert massive error log
- Slack App for check merchant's shops SSL expiration day
- Version control Lambda function for AB testing

<br />


Reference
-
https://day.ithome.com.tw/serverless/ <br />

https://serverless.com/ <br />

Cliff <br />
https://github.com/clifflu?tab=repositories

Poga - Built his own serverless called Spacer <br />
https://github.com/poga/spacer

Rick Huang - 91APP tech manager <br />
https://rickhw.github.io/


<br />
<br />
<div style="text-align: right;">
2018-04-21 17:32 , David in Taipei</div>

<br />
<br />
<br />


