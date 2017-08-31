---
published: true
layout: post
title: Docker Note (Nginx+uwsgi+Letsencrypt)
author: David
modified: 2017-08-31
categories: [雜七雜八 , Python]
tags: 
  - python
  - nginx
  - uwsgi
  - flash
  - letsencrypt
  - docker
  - container

  
comments: true
---
<h3>
<b>docker筆記</b></h3>
<div>
<br /></div>
<div>
nginx+uwsgi+flask+letsencrypt</div>
<div>
<br /></div>
<div>
Env: Python3</div>
<div>
<br /></div>
<div>
Line串接API使用，需執行crontab更新憑證(簽署期限為30天)</div>
<div>
<br /></div>
<div>
<br /></div>
<div>
<div>
note:</div>
<div>
<br /></div>
<div>
certbot-auto可單一clone此檔案就好</div>
<div>
<br /></div>
<div>
產生憑證： (docker裡執行)</div>
<div>
<span style="background-color: #fff2cc;">certbot-auto certonly --webroot --webroot-path=WEB_ROOT -d DOMAIN_NAME</span></div>
<div>
<br /></div>
<div>
更新憑證：&nbsp;</div>
<div>
<span style="background-color: #fff2cc;">docker exec -it CONTAINER_ID /opt/letsencrypt/certbot-auto renew --quiet --renew-hook "/etc/init.d/nginx restart"</span></div>
<div>
<br /></div>
<div>
啟動程序：&nbsp;</div>
<div>
<span style="background-color: #fff2cc;">docker run -p 80:80 -p 443:443 CONTAINER_ID /opt/start. &amp;</span></div>
</div>
<div>
<br /></div>
<div>
/opt/start.sh：</div>
<div>
<br /></div>
<div>
<span style="background-color: #f4cccc;">#!/bin/bash</span></div>
<div>
<span style="background-color: #f4cccc;">/bin/bash/uwsgi --socket 0.0.0.0:8080 --protocol=http -w PROGRAM_NAME</span></div>
<div>
<span style="background-color: #f4cccc;">/etc/init.d/nginx start</span></div>
<div>
<br /></div>
<div>
<br /></div>
<div>
備註：</div>
<div>
<br /></div>
<div>
create docker image:</div>
<div>
when quit docker use CTRL+P &amp; CTRL+Q</div>
<div>
<span style="background-color: #fff2cc;">docker ps</span> (searching docker container_id)</div>
<div>
<span style="background-color: #fff2cc;">docker commit -m "COMMENT" CONTAINER_ID your_account/container_name:tag</span></div>
<div>
<br /></div>
<div>
docker run container:</div>
<div>
<span style="background-color: #fff2cc;">docker run -p 80:80 -p 443:443 CONTAINER_ID PROGRAM</span> (multi port)</div>
<div>
<br /></div>
<div>
docker enter when container is running:</div>
<div>
<span style="background-color: #fff2cc;">docker exec -it CONTAINER_ID /bin/bash</span></div>
<div>
<br /></div>
<div>
docker remove images:</div>
<div>
<span style="background-color: #fff2cc;">docker rmi IMAGES_ID</span></div>
<div>
<br /></div>
<div>
docker delete container:</div>
<div>
<span style="background-color: #fff2cc;">docker rm CONTAINER_ID</span></div>
<div>
<br /></div>
<div>
未完</div>
<div>
<br /></div>
<div style="text-align: right;">
20170831 , David in Taipei</div>
<div style="text-align: right;">
<br /></div>
<div style="text-align: right;">
<br /></div>
