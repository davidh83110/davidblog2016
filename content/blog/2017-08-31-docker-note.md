---
author: David
categories:
- jibber-jabber
comments: true
date: "2017-08-31T00:00:00Z"
modified: "2017-08-31"
published: true
title: Docker Note (Nginx+uwsgi+Letsencrypt)
url: /雜七雜八/python/2017/08/31/docker-note.html
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
<span style="background-color: #f4cccc;">/etc/init.d/nginx start</span><br />
<span style="background-color: #f4cccc;"><br /></span>
<span style="background-color: #f4cccc;"><br /></span>
<span style="background-color: #f4cccc;">Nginx Config:</span><br />
<span style="background-color: #f4cccc;"><br /></span>
原理是使用proxy_pass來串接uwsgi<br />
而我強制轉換為HTTPS，配置方式如下，只列出重點配置<br />
<br />
<br />
server {<br />
&nbsp; listen 80;<br />
&nbsp; server_name DOMAIN.COM.TW;<br />
&nbsp;<br />
&nbsp; return 301 https://$server_name$request_uri;<br />
&nbsp; }<br />
<br />
server {<br />
&nbsp; listen 443;<br />
&nbsp; server_name DOMAIN.COM.TW;<br />
&nbsp; ssl_certificate /etc/letsencrypt/live/blog.gtwang.org/fullchain.pem;<br />
&nbsp; ssl_certificate_key /etc/letsencrypt/live/blog.gtwang.org/privkey.pem;<br />
&nbsp; ssl_protocols TLSv1 TLSv1.1 TLSv1.2; <br />
<br />
&nbsp; proxy_pass http://127.0.0.1:8080;<br />
&nbsp; }</div>
<div>
<br />
<br />
這邊只使用到TLSv1.2而不使用最新的1.3原因是因為1.3存在漏洞<br />
我們使用proxy_pass代理uwsgi的服務<br />
<br />
使用letsencrypt前別忘了在nginx根目錄裡建立.well-known的資料夾以作為webroot認證<br />
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
<br /></div>
<div>
<br /></div>
<div style="text-align: right;">
20170831 , David in Taipei</div>
<div style="text-align: right;">
<br /></div>
<div style="text-align: right;">
<br /></div>

