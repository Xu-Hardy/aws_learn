---
title: nginx返回客户端IP
tags: 
    - Nginx
    - Public IP
---


1. 在此文件中输入以下内容：

```nginx
server {
    listen 800 default_server;
    listen [::]:800 default_server;

    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }

    location /myip {
        default_type text/plain;
        return 200 $remote_addr;
    }
}
```

2. 检查Nginx配置是否有误：

```bash
sudo nginx -t
```

如果返回`syntax is okay`和`test is successful`，则配置没有问题。

3. 重启Nginx以应用更改：

```bash
sudo systemctl restart nginx
```

现在，当您访问`http://your_server_ip_or_domain/myip`时，它应该显示您的公网IP地址。