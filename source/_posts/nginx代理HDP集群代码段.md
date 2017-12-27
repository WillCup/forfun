
title: nginx代理HDP集群代码段
date: 2017-06-07 19:43:46
tags: [youdaonote]
---

```

#user  nobody;
worker_processes  1;


events {
    worker_connections  1024;
}





http {
    include       mime.types;
    default_type  application/octet-stream;


upstream jobhis {
        server servicenode03.will.com:19888 fail_timeout=30s;
    }

    upstream azk {
        server schedule.will.com:8081 fail_timeout=30s;
    }

    upstream yarn {
        server datanode02.will.com:8088 fail_timeout=30s;
    }


    sendfile        on;
    keepalive_timeout  65;


    server {
        listen       81;
        server_name  localhost;

	subs_filter_types *;
#        subs_filter overwrite willhistory i;
        subs_filter datanode02.will.com:8088 $host:$server_port;
	subs_filter 10.2.19.(\d*) unknown_ip ir;
	subs_filter (data|name|service)node(\d*).will.com $1_$2.data.com ir;
#        subs_filter will.com will.data.com ir;


        #charset koi8-r;

        #access_log  logs/host.access.log  main;

	location /cluster/app {
                auth_request /auth;
                try_files $uri @yarn;
        }

        location /proxy {
                auth_request /auth;
                try_files $uri @yarn;
        }

        location /jobhistory {
               auth_request /auth;
               try_files $uri @jobhis;
		rewrite ^/jobhistory/logs/(data|service|name)_(\d*).data.com:(\d*)/(.*)$  /jobhistory/logs/$1node$2.will.com:$3/$4 break;
        }


	location / {
                try_files $uri @azk;
        }

	location @jobhis{
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_pass http://jobhis;
        }


        location = /auth {
           internal;
           proxy_pass http://10.2.19.62:8088/metamap/nginx_auth_test;
           proxy_set_header Content-Length "";
           proxy_set_header X-Original-URI $request_uri;
           proxy_pass_request_body off;
        }

        location /static {
                 try_files $uri @yarn;
        }

	location @yarn{
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_pass http://yarn;
        }

        location @azk{
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_pass http://azk;
	    subs_filter_types *;
	    subs_filter datanode(\d\d).will.com will.will.com ir;
	    subs_filter 10.2.19.(\d*) unknowip ir;
        }



        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

   
    }


   
}

```


