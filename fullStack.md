# 前後端分離小專案(將部屬上kubernetes)
## 主題

## 前端
### 前端框架
vue.js
### 網頁架構

---
## 後端
### 後端框架
Django
### 使用語言
python
### API說明

---
## 注意
使用nginx架設前端，若要連接後端則要修改default.conf
```
upstream [後端service名稱]{
     server [後端service名稱];
}

server {
    listen       80;
    server_name  localhost;
    location / {
         root   /usr/share/nginx/html;
         index  index.html index.htm;
    }

    location /api {
        proxy_pass http://[後端service名稱];
    }

    error_page   500 502 503 504  /50x.html;
        location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```