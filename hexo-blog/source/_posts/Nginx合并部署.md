---
title: nginx同端口多目录层级部署
date: 2018-01-12 17:41:21
tags: [nginx,反向代理]
---

# nginx同端口多目录层级部署

## 准备工作

### 系统环境

* 64位CentOS 7.2
* 64位JDK 1.8

### 工具

* tomcat服务器，版本：7.0.69   
* nginx服务器，版本：1.12.0

<!-- more -->

### 后端环境

后端需要准备好的包括    

* 合理用药系统服务，知识管理平台系统服务，用户中心服务     
* med.war，knowledgecenter.war，systemcenter.war

### 前端环境
前端需要准备好的包括    

* homecenter，knowledgecenter以及systemcenter前端包

## 部署
### 后端
1.  启动各个系统的服务（包括tomcat中的setenv.sh文件详情见3.4系统部署文档）
2.	将med.war，knowledgecenter.war以及systemcenter.war放到/mnt/yyspace/soft/tomcat（包名以及路径可自定义）/webapps 目录下
3.	进入到/mnt/yyspace/soft/tomcat（包名以及路径可自定义）/bin目录下，执行./startup.sh 命令

### 前端
1.	将前端包放到/mnt/yyspace/onestep目录下（包名以及路径可自定义）
2.	进入到/mnt/yyspace/soft/nginx-1.12.0（包名以及路径可自定义）/sbin目录下，执行./nginx 命令

# Nginx配置详解
## Ngixn默认配置文件
**部分注释块已删除**

```nginx
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
}
```

## 统一部署Nginx配置文件

```nginx
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
	server {
		listen       9999;
		server_name  Merger-deployment;
		#charset koi8-r;
		location /systemcenter/ {
			root /mnt/yyspace/onestep;
            rewrite ^/system-center/(?:login|management|system_management|data_management|drug_management|rule_management|rule-management|guest_management|knowledge_sharing|dictionary_management|dict-management|mxgraph)+(/.*)?$/systemcenter/index.html;
		}
				
		location /knowledgecenter/ {
			root /mnt/yyspace/onestep;
	        rewrite ^/knowledgecenter/(?:home|system_management|data-management|product-management|drug_management|rule_management|rule-management|guest_management|knowledge_sharing|dictionary_management|dict-management|mxgraph)+(/.*)$/knowledgecenter/index.html;
		}
		location /system/knowledge/ {
			proxy_pass http://10.1.2.136:8080/knowledgecenter/api/v1/;
			proxy_set_header Cookie $http_cookie;
		}

		location /system/ {
			proxy_pass http://10.1.2.136:8080/systemcenter/;
			proxy_set_header Cookie $http_cookie;
		}
				
		location /api/v1/ {
			proxy_pass http://10.1.2.136:8080/knowledgecenter/api/v1;
			proxy_set_header Cookie $http_cookie;
		}
				
		location /med/{
			proxy_pass http://10.1.2.136:8080/med/;
			proxy_set_header Cookie $http_cookie;
		}
		location / {
			root /mnt/yyspace/onestep/homecenter;
		}
	}
}
```

### 资源包路径定位
#### 用户中心
```nginx
location /systemcenter/ {
	root /mnt/yyspace/onestep;
    rewrite ^/system-center/(?:login|management|system_management|data_management|drug_management|rule_management|rule-management|guest_management|knowledge_sharing|dictionary_management|dict-management|mxgraph)+(/.*)?$/system-center/index.html;
}
```

**其中，root定位到/mnt/yyspace/onestep即可，会根据location** **/systemcenter/定位到用户中心的资源包**

#### 知识管理平台
```nginx
location /knowledgecenter/ {
	root /mnt/yyspace/onestep;
    rewrite ^/knowledgecenter/(?:home|system_management|data-management|product-management|drug_management|rule_management|rule-management|guest_management|knowledge_sharing|dictionary_management|dict-management|mxgraph)+(/.*)$  /knowledgecenter/index.html;
}
```

#### 统一登录页
```nginx
location / {
    root /mnt/yyspace/onestep/homecenter;
}
```
**因为统一登录页默认9999端口访问，所以需要定位到精确的文件夹**

### rewirte说明
在用户中心以及知识管理平台的root新增了rewrite，因为路由是以“/”来对模块进行区分，加载页面，为了防止手输地址，以及刷新，造成服务器无法访问（浏览器会默认去文件夹下加载资源），将路由重新写到对应的包下面的index.html中，走路由，保证前端能在刷新以及手输地址的情况下能正常访问。

### 反向代理说明
```nginx
location /system/ {
    proxy_pass http://10.1.2.136:8080/systemcenter/;
    proxy_set_header Cookie $http_cookie;
}
location /system/knowledge/ {
    proxy_pass http://10.1.2.136:8080/knowledgecenter/api/v1/;
    proxy_set_header Cookie $http_cookie;
}
location /api/v1/ {
    proxy_pass http://10.1.2.136:8080/knowledgecenter/api/v1/;
    proxy_set_header Cookie $http_cookie;
}
location /med/{
    proxy_pass http://10.1.2.136:8080/med/;
    proxy_set_header Cookie $http_cookie;
}
```

**location /system/**:对于包含/system/前缀的接口，转发到
proxy_pass http://10.1.2.136:8080/systemcenter/下
**注意**：在有目录层级情况下，proxy_pass 末尾“/”带上，转发之后的url不带system，
以http://10.1.2.136:9999/system/current 为例，会被转发至真实接口
http://10.1.2.136:8080/systemcenter/current 。
如果proxy_pass 末尾不带上“/”，以http://10.1.2.136:9999/system/current 为例，会被转发至真实接口http://10.1.2.136:8080/systemcentersystem/current，引发错误

**location /api/v1/**:对于包含/ api/v1/前缀的接口，转发到
proxy_pass http://10.1.2.136:8080/knowledgecenter/api/v1/下

**location /med/**:对于包含/ med/前缀的接口，转发到
proxy_pass http://10.1.2.136:8080/med/下

**location /system/knowledge/**:对于包含 /system/knowledge/ 前缀的接口，转发到proxy_pass http://10.1.2.136:8080/knowledgecenter/api/v1/下

### 注意点
1.	因为知识管理平台架构比较老，没有对url做提取，所以无法对url加统一前缀，/api/v1/这个标识需要给到知识管理平台
2.	对于用户中心以及统一登录页，对url进行了提取，放到json文件里，可以统一加上前缀（前缀与接口匹配规则对应，例如location /system/）。
用户中心是url.json文件，统一登录也是urlFile.json，在host后面加上前缀
例如下图：
 
3.	针对于用户中心对于调用知识管理平台的接口，有knowledge前缀，所以还需要对/system/knowledge/ 标识进行转发，转发到知识管理平台后端。
4.	针对于404 报错问题，首先排查，rewrite 的报名以及location的包名是否正确，第二步排查路由是否被rewrite正确重写，如果rewirte里面不包含路由，就添加
5.	修改nginx.conf文件之后，必须执行./nginx –s reload命令



