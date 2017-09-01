django channels部署
=========================== 

channels是django新出来的一个"杀手级"功能，可以使用它来替换celery后台任务和处理websocket.官方文档中目前还没有集成到django中，目前以一个第三方的库来支持，但是后面的django肯定会把它作为内置的功能。

channels的部署方案在官方文档里面已经有提示，这里使用uwsgi + nginx + daphne的方式，原来的正常的http请求和静态文件的请求还是走uwsgi+nginx的模式，websocket的请求走nginx + daphne + worker的形式。这样子不会影响之前的架构，部署起来成本很低。

另外，官方文档在部署方法里面提到了如何在后台启动channels，但是没有给出具体的部署方案，只是简单的提到了一些方法，这些需要你自己去尝试。

##以upstart的方式启动daphne interface 和worker

###启动daphne interface server
`cd /etc/init && vim daphne-interface.conf`

编辑内容如下:
```
start on runlevel [1] 
stop on runlevel [016]

respawn

script
    cd /data/www/myproject
    export DJANGO_SETTINGS_MODULE="myproject.settings"
    exec /data/code/cy_devops/bin/daphne -b 0.0.0.0 -p 8001 --access-log /var/log/daphne-interface.log  myproject.asgi:channel_layer
end script
```

启动daphne-interface
`initctl start daphne-interface`

###启动daphne worker
`cd /etc/init && vim daphne-worker.conf`

编辑内容如下：
```
start on runlevel [1] 
stop on runlevel [016]

respawn

script
    cd /data/www/myproject
    export DJANGO_SETTINGS_MODULE="myproject.settings"
    exec /data/code/cy_devops/bin/python manage.py runworker --threads 2  -v2 
end script
```

启动daphne worker
`initctl start daphne-worker`

##配置nginx支持websocket
此处的难点是反向代理 websockets，因为 nginx 默认不识别 websocket 协议。为了能正确指定协议头，只能将所有 websocket 请求路由到某一子路径下**/ws**。
增加内容如下:
```
location /ws {
        proxy_pass http://0.0.0.0:8001;    # 转发到daphne-worker
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_redirect     off;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Host $server_name;
        proxy_read_timeout  36000s;
        proxy_send_timeout  36000s;
    }
```
