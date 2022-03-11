# 環境建置
## 1. 時區修改
```shell
$ date
$ timedatectl list-timezones | grep Taipei
$ timedatectl set-timezone Asia/Taipei
$ date
$ ls -l /etc/localtime
```

## 2. httpd安裝
#### 安裝httpd
```shell
$ yum install httpd
```

#### 調整selinux & firewall
```shell
$ semanage port -a -t http_port_t -p tcp 18080 

$ firewall-cmd --add-port=18080/tcp --permanent
$ firewall-cmd --reload
$ firewall-cmd --list-ports
```

#### 修改/etc/httpd/conf/httpd.conf
```shell
$ vi /etc/httpd/conf/httpd.conf
```
```
#將
Listen 80
#改為
Listen 18080
```

#### 啟動httpd
```shell
#設定開機及啟用httpd
$ systemctl enable httpd

#重啟httpd
$ systemctl restart httpd

#查看httpd狀態
$ systemctl status httpd
```

## 3. 安裝後端service
#### 調整selinux & firewall
```shell
$ semanage port -a -t http_port_t -p tcp 10001

$ firewall-cmd --add-port=10001/tcp --permanent
$ firewall-cmd --reload
$ firewall-cmd --list-ports
```

#### 檔案放置
```shell
$ mkdir /usr/local/ctbc-bankend
$ cp ctbc-emap-backend.jar /usr/local/ctbc-bankend/
$ cp ctbc.service /usr/lib/systemd/system/
```

#### 啟動服務
```shell
#重新加载服務配置
$ systemctl daemon-reload

#啟動服務
$ systemctl start ctbc.service

#查看服務狀態
$ systemctl status ctbc.service
```