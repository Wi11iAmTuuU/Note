# OCP4.6/4.7筆記

OCP集群中機器分為四種

```
bastion(操作機) X1
--------------------
bootstrap      X1
master node    X3
compute node   X2
```
設備要求
```
bootstrap CPU: 4 core RAM: 16 G 
master    CPU: 4 core RAM: 16 G 
worker    CPU: 2 core RAM: 8 G 
```

## -----建立bastion-----

### 修改 bastion 的 hostname
```
#修改hostname
hostnamectl set-hostname bastion.ocp.[name].com
#重啟
reboot
```

### 防火牆&selinux設定
```
#關閉防火牆
systemctl  stop  firewalld
systemctl  disable  firewalld

#暫時關閉selinux
setenforce 0

#永久關閉selinux 
#找到SELINUX= 將enforcing改為permissive
vi /etc/selinux/config

#重啟
reboot
```

### 安裝與設定 dnsmasq
```
#安裝dnsmasq
yum install -y dnsmasq

#建立設定檔
touch /etc/dnsmasq.d/dns.conf
```
```
#修改/etc/dnsmasq.d/dns.conf 
#填入內容如下
#修改所有domain&自己要設定的IP
# XX = basion   IP
# OO = boostrap IP
# ** = master   IP X3
# ## = worker   IP X2

domain=ocp.[name].com,10.255.83.0/24,local
server=192.168.101.1
server=192.168.101.2
host-record=bastion.ocp.[name].com,10.255.78.XX
host-record=bootstrap.ocp.[name].com,10.255.83.OO
host-record=master0.ocp.[name].com,10.255.83.**
host-record=master1.ocp.[name].com,10.255.83.**
host-record=master2.ocp.[name].com,10.255.83.**
host-record=compute0.ocp.[name].com,10.255.83.##
host-record=compute1.ocp.[name].com,10.255.83.##

### 
host-record=api.ocp.[name].com,10.255.78.XX
host-record=api-int.ocp.[name].com,10.255.78.XX

host-record=etcd-0.ocp.[name].com,10.255.83.**
host-record=etcd-1.ocp.[name].com,10.255.83.**
host-record=etcd-2.ocp.[name].com,10.255.83.**
srv-host=_etcd-server-ssl._tcp.ocp.[name].com,etcd-0.ocp.[name].com,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp.[name].com,etcd-1.ocp.[name].com,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp.[name].com,etcd-2.ocp.[name].com,2380,0,10
###


address=/apps.ocp.[name].com/10.255.78.XX
address=/.apps.ocp.[name].com/10.255.78.XX
address=/api.ocp.[name].com/10.255.78.XX
address=/api-int.ocp.[name].com/10.255.78.XX
address=/bastion.ocp.[name].com/10.255.78.XX
address=/bootstrap.ocp.[name].com/10.255.83.OO
address=/master0.ocp.[name].com/10.255.83.**
address=/master1.ocp.[name].com/10.255.83.**
address=/master2.ocp.[name].com/10.255.83.**
address=/etcd-0.ocp.[name].com/10.255.83.**
address=/etcd-1.ocp.[name].com/10.255.83.**
address=/etcd-2.ocp.[name].com/10.255.83.**
address=/compute0.ocp.[name].com/10.255.83.##
address=/compute1.ocp.[name].com/10.255.83.##
```
```
#啟用dnsmasq
systemctl enable dnsmasq 

#重啟dnsmasq
systemctl restart dnsmasq 

#查看dnsmasq狀態
systemctl status dnsmasq 
```

### 安裝與設定 haproxy
```
#安裝haproxy
yum install haproxy 
```

```
#修改/etc/haproxy/haproxy.cfg
#填入內容如下
#修改所有domain&自己要設定的IP
# XX = basion   IP
# OO = boostrap IP
# ** = master   IP X3
# ## = worker   IP X2

frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server bootstrap	10.255.83.OO:6443 check
    server master0	10.255.83.**:6443 check
    server master1	10.255.83.**:6443 check
    server master2	10.255.83.**:6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server bootstrap    10.255.83.OO:22623 check
    server master0	10.255.83.**:22623 check
    server master1	10.255.83.**:22623 check
    server master2	10.255.83.**:22623 check

frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server compute0	10.255.83.##:80 check
    server compute1	10.255.83.##:80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server compute0	10.255.83.##:443 check
    server compute1	10.255.83.##:443 check
```
```
#啟用haproxy
systemctl enable haproxy 

#重啟haproxy
systemctl restart haproxy 

#查看haproxy狀態
systemctl status haproxy
```

### 安裝與設定 httpd 
```
#安裝httpd
yum install httpd
```
```
#修改/etc/httpd/conf/httpd.conf 

#將
Listen 80
#改為
Listen 8080
```
```
#啟用httpd
systemctl enable httpd

#重啟httpd
systemctl restart httpd

#查看httpd狀態
systemctl status httpd
```

### 修改resolv.conf 
```
#修改/etc/resolv.conf
#修改內容為下 取代所有原有內容
search dynakumo.local ocp.[name].com
nameserver 10.255.78.XX
```

### 生成SSH私鑰並將其添加到代理 
```
#產生ssh key
ssh-keygen -t rsa -b 4096 -N ''
ssh-keygen -t ed25519 -N ''

#ssh-agent作為後台任務啟動該過程
eval "$(ssh-agent -s)"

#將您的SSH私鑰添加到ssh-agent
ssh-add /root/.ssh/id_rsa
ssh-add /root/.ssh/id_ed25519
```

## -----安裝新cluster-----
### 獲取安裝相關檔案
![](https://i.imgur.com/KgjJxCx.png)
![](https://i.imgur.com/SkjNtgT.png)
![](https://i.imgur.com/5WNtl26.png)
```
#注意 下載時請右鍵複製網址 並找到4.6.8版本

#下載 openshift-install-linux-4.6.8.tar.gz 並傳入 bastion /root/

#下載Pull-Secret.txt 存於本機備用

#下載 openshift-client-linux-4.6.8.tar.gz 並傳入 bastion /root/

#下載 rhcos-4.6.8-x86_64-metal.x86_64.raw.gz 並傳入 bastion /root/
```

### 安裝client(oc 指令工具)
```
#解壓縮client
tar zxvf /root/openshift-client-linux-4.5.6.tar.gz

#查看目前 $PATH 變數 
echo $PATH

#移動解壓縮的檔案
mv -f /root/oc /usr/local/bin
mv -f /root/kubectl /usr/local/bin
mv -f /root/README.md /usr/local/bin

# 查看是否有移動成功
ll /usr/local/bin
> total 145572
> -rwxr-xr-x 2 root root 74528312 Dec  5 11:12 kubectl
> -rwxr-xr-x 2 root root 74528312 Dec  5 11:12 oc
> -rw-r--r-- 1 root root      954 Dec  5 11:12 README.md

#測試oc指令
oc
> OpenShift Client
> ...
```

### 創建安裝配置文件
```
#解壓縮openshift-install工具
tar zxvf /root/openshift-install-linux-4.5.6.tar.gz

#創建安裝目錄
mkdir /root/ocp-data

#創建OCP的安裝設定文件
touch /root/ocp-data/install-config.yaml
```
```
#修改/root/ocp-data/install-config.yaml
#填入內容如下 並修改
apiVersion: v1
baseDomain: [name].com 
compute:
- hyperthreading: Enabled   
  name: worker
  replicas: 0 
controlPlane:
  hyperthreading: Enabled   
  name: master 
  replicas: 3 
metadata:
  name: ocp 
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths": ...}' 
sshKey: 'ssh-ed25519 AAAA...' 
```
```
> baseDomain : 需修改 需與Dnsmasq設定符合
> metadata: name: 可修改 也可維持
> compute: replicas: 必須為0
> controlPlane: replicas: master的數量
> 置換pullSecret ''中的內容為自己的 pullSecret 內容
> 置換sshKey ''中的內容為 /root/.ssh/id_rsa.pub
```
```
#透過 openshift-install 工具建造manifests
/root/openshift-install create manifests --dir=/root/ocp-data/
```

### cluster-scheduler配置
```
#修改 /root/ocp-data/manifests/cluster-scheduler-02-config.yml
#將mastersSchedulable 設為 False
```

### hybrid networking with OVN-Kubernetes配置 (option)
```
#修改 /root/ocp-data/manifests/cluster-network-03-config.yml
```
```
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  defaultNetwork:
    ovnKubernetesConfig:
      hybridOverlayConfig:
        hybridClusterNetwork: 
        - cidr: 10.132.0.0/14
          hostPrefix: 23
        hybridOverlayVXLANPort: 9898 
```

### 創建ignition文件
```
#創建ignition文件
#注意ignition憑證只有24小時，必須在24小時內將cluster建好(待確認)
/root/openshift-install create ignition-configs --dir=/root/ocp-data

#在目錄中生成以下文件：
# ├┬─AUTH 
# │├──kubeadmin密碼
# │└──kubeconfig 
# ├──bootstrap.ign 
# ├──master.ign 
# ├──metadata.json 
# └──worker.ign

#將bootstrap.ign master.ign worker.ign 
#與之前準備的image(rhcos-4.6.8-x86_64-metal.x86_64.raw.gz)
#共四個檔案 移動至/var/www/html/
rm -f /var/www/html/bootstrap.ign 
rm -f /var/www/html/master.ign 
rm -f /var/www/html/worker.ign 
cp -f /root/ocp-data/bootstrap.ign /var/www/html/ 
cp -f /root/ocp-data/master.ign /var/www/html/ 
cp -f /root/ocp-data/worker.ign /var/www/html/ 
cp -f /root/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz /var/www/html/rhcos.raw.gz

#修改權限來讓httpd可以存取
chmod 755 /var/www/ -R
```

### 配置Red Hat Enterprise Linux CoreOS（RHCOS）機器
![](https://i.imgur.com/rgildzw.png)
與4.5相似安裝方法
```
#開啟RHCOS 4.6.8 VM主控台
#在rhcos(live) 自動install前按下tab
#保留原有指令 輸入以下指令(不用換行)
#bootstrap範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://192.168.50.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.XX:8080/bootstrap.ign 
ip=192.168.50.OO::192.168.50.253:255.255.255.0:bootstrap.ocp.[name].com:ens192:none 
nameserver=192.168.50.XX

#master範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://192.168.50.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.XX:8080/master.ign 
ip=192.168.50.**::192.168.50.253:255.255.255.0:master0.ocp.[name].com:ens192:none 
nameserver=192.168.50.XX

#compute範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://192.168.50.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.XX:8080/worker.ign 
ip=192.168.50.##::192.168.50.253:255.255.255.0:compute0.ocp.[name].com:ens192:none 
nameserver=192.168.50.XX

#輸入完成後 按下enter運行安裝
#等到RHCOS跑完
#如有錯誤 則會出現等待五分鐘後自動重啟 等待重啟後即可再次嘗試
```

4.6版rhcos(live)安裝方法
```
#連線[OCP] RHCOS 4.6.8 VM 開啟主控台
#先設定網路相關(nmcli/nmtui)

node’s IP address
gateway address
The netmask
The hostname
The DNS server

#輸入以下指令 沒有換行
#bootstrap範例:
sudo coreos-installer install /dev/sda 
--ignition_url=http://10.255.78.XX:8080/bootstrap.ign
--image-url=http://10.255.78.XX:8080/rhcos.raw.gz 

#master範例:
sudo coreos-installer install /dev/sda 
--ignition_url=http://10.255.78.XX:8080/master.ign
--image-url=http://10.255.78.XX:8080/rhcos.raw.gz 

#compute範例:
sudo coreos-installer install /dev/sda 
--ignition_url=http://10.255.78.XX:8080/worker.ign
--image-url=http://10.255.78.XX:8080/rhcos.raw.gz 
```

### 創建集群監控boostrap創建集群過程
```
#監控boostrap創建集群過程
/root/openshift-install --dir=/root/ocp-data/ wait-for bootstrap-complete --log-level=info

#輸出範例
> INFO Waiting up to 30m0s for the Kubernetes API at > https://api.test.example.com:6443...
> INFO API v1.19.0 up
> INFO Waiting up to 30m0s for bootstrapping to complete...
> INFO It is now safe to remove the bootstrap resources

#boostrap完成後 將boostrap從haproxy移除
```

### 使用 CLI 登入到集群
```
#導出kubeadmin憑證
export KUBECONFIG=/root/ocp-data/auth/kubeconfig

#查看使用者 輸出為system:admin即可
oc whoami
```

### 批准所有機器的CSR
```
#查看node、csr及批准所有機器的CSR
oc get nodes
oc get csr
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

#等待所有node都ready即可
watch -n 1 oc get nodes
```

### 初始Operator配置
```
#查看組件 直到所有組件都出現版本號為止
watch -n5 oc get clusteroperators

#調整Image Registry Operator
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

### 初始User設定
```
#利用htpasswd新增User
htpasswd -c -B -b /root/users.htpasswd [Name] [password]

#將users.htpasswd轉成secret到OCP上
oc create secret generic htpass-secret --from-file=htpasswd=/root/users.htpasswd -n openshift-config

#將帳密secret綁到登入器裡
#創建密碼相關設定文件
touch /root/CR
```
```
#修改/root/CR內容如下
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: my_htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```
```
#將/root/CR設定進OCP
oc apply -f /root/CR

#提升user權限至cluster-admin
oc adm policy add-cluster-role-to-user cluster-admin [Name] --rolebinding-name=cluster-admin
```

### 添加hosts來登入web-console
```
C:\Windows\System32\drivers\etc\hosts

10.255.78.XX console-openshift-console.apps.ocp.[name].com
10.255.78.XX oauth-openshift.apps.ocp.[name].com
10.255.78.XX api.ocp.[name].com
10.255.78.XX downloads-openshift-console.apps.ocp.[name].com

#瀏覽器輸入console-openshift-console.apps.ocp.[name].com
選擇my_htpasswd後登入
```

## -----集群管理相關-----
### 新增節點
```
#bastion上添加dns設定
/etc/dnsmsq.d/dns.conf

#新增master node添加以下項目
host-record=masterX.ocp.lab.com,10.255.83.XX
host-record=etcd-X.ocp.lab.com,10.255.83.XX
srv-host=_etcd-server-ssl._tcp.ocp.lab.com,etcd-X.ocp.lab.com,2380,0,10
address=/masterX.ocp.lab.com/10.255.83.XX
address=/etcd-X.ocp.lab.com/10.255.83.XX

#新增worker node添加以下項目
host-record=host-record=computeX.ocp.lab.com,10.255.83.XX
address=/computeX.ocp.lab.com/10.255.83.XX

#重啟服務
systemctl restart dnsmasq
```
```
#bastion上添加haproxy設定
/etc/haproxy/haproxy.cfg

#新增master node添加以下項目
server masterX	10.255.83.XX:6443 check
server masterX	10.255.83.XX:22623 check

#新增worker node添加以下項目
server computeX	10.255.83.XX:80 check
server computeX	10.255.83.XX:443 check

#重啟服務
systemctl restart haproxy
```
```
#開啟RHCOS 4.6.8 VM主控台
#在rhcos(live) 自動install前按下tab
#保留原有指令 輸入以下指令(不用換行)
#bootstrap範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://192.168.50.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.XX:8080/bootstrap.ign 
ip=192.168.50.OO::192.168.50.253:255.255.255.0:bootstrap.ocp.[name].com:ens192:none 
nameserver=192.168.50.XX

#master範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://192.168.50.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.XX:8080/master.ign 
ip=192.168.50.**::192.168.50.253:255.255.255.0:master0.ocp.[name].com:ens192:none 
nameserver=192.168.50.XX

#compute範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://192.168.50.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.XX:8080/worker.ign 
ip=192.168.50.##::192.168.50.253:255.255.255.0:compute0.ocp.[name].com:ens192:none 
nameserver=192.168.50.XX

#輸入完成後 按下enter運行安裝
#等到RHCOS跑完
#如有錯誤 則會出現等待五分鐘後自動重啟 等待重啟後即可再次嘗試
```
```
# 回到bastion上 登入
oc login

##查看node、csr及批准所有機器的CSR
oc get nodes
oc get csr
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
watch -n 1 oc get nodes

#等待新增的node都ready即可
watch -n 1 oc get nodes
```
### 增加更多user
```
#新增新帳號密碼至dtpasswd檔案
htpasswd -bB /root/users.htpasswd [name] [password]

#替換OCP上secret
oc create secret generic htpass-secret --from-file=htpasswd=/root/users.htpasswd --dry-run -o yaml -n openshift-config | oc replace -f -

#提升權限至cluster-admin
oc adm policy add-cluster-role-to-user cluster-admin [Name] --rolebinding-name=cluster-admin
```

oc login時如遇到`Error from server (InternalError): Internal error occurred: unexpected response: 500`
```
#先刪除該帳號
oc delete user foo
> user "foo" deleted

#查看identities中是否還有該帳號
oc get identities
> NAME                    IDP NAME      IDP USER NAME   USER NAME   USER UID
> anypassword:developer   anypassword   developer       developer   ec2e120b-27ae-11e8-b202-507b9dac9a27
> anypassword:foo         anypassword   foo             foo         bcb48b19-27af-11e8-b202-507b9dac9a27

#有的話 刪除該帳號相關
oc delete identities/anypassword:foo
> identity "anypassword:foo" deleted

#重新登入
oc login -u foo
> Authentication required for https://127.0.0.1:8443 (openshift)
> Username: foo
> Password: 
> Login successful.
```

## coreos.inst boot options for ISO install
### 注意這些參數不用換行 用空白隔開就好
### bootstrap:
```
coreos.inst.install_dev=sda 
coreos.inst.insecure=yes
coreos.inst.image_url=http://192.168.50.21:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.21:8080/bootstrap.ign 
ip=192.168.50.201::192.168.50.253:255.255.255.0:bootstrap.ocp.olg.com:ens192:none 
nameserver=192.168.50.249
```

### master0:
```
coreos.inst.install_dev=sda 
coreos.inst.insecure=yes
coreos.inst.image_url=http://192.168.50.21:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.21:8080/master.ign 
ip=192.168.50.202::192.168.50.253:255.255.255.0:master0.ocp.olg.com:ens192:none 
nameserver=192.168.50.249
```

### master1:
```
coreos.inst.install_dev=sda 
coreos.inst.insecure=yes
coreos.inst.image_url=http://192.168.50.21:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.21:8080/master.ign 
ip=192.168.50.203::192.168.50.253:255.255.255.0:master0.ocp.olg.com:ens192:none 
nameserver=192.168.50.249
```

### master2:
```
coreos.inst.install_dev=sda 
coreos.inst.insecure=yes
coreos.inst.image_url=http://192.168.50.21:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.21:8080/master.ign 
ip=192.168.50.204::192.168.50.253:255.255.255.0:master0.ocp.olg.com:ens192:none 
nameserver=192.168.50.249
```

### compute0:
```
coreos.inst.install_dev=sda 
coreos.inst.insecure=yes
coreos.inst.image_url=http://192.168.50.21:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.21:8080/worker.ign 
ip=192.168.50.207::192.168.50.253:255.255.255.0:compute0.ocp.olg.com:ens192:none 
nameserver=192.168.50.249
```

### compute1:
```
coreos.inst.install_dev=sda 
coreos.inst.insecure=yes
coreos.inst.image_url=http://192.168.50.21:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://192.168.50.21:8080/worker.ign 
ip=192.168.50.208::192.168.50.253:255.255.255.0:compute0.ocp.olg.com:ens192:none 
nameserver=192.168.50.249
```

## coreos-installer安裝參數
### 注意這些參數不用換行 用空白隔開就好
### bootstrap:
```
sudo coreos-installer install /dev/sda 
--ignition-url=http://192.168.50.6:8080/bootstrap.ign
--image-url=http://192.168.50.6:8080/rhcos.raw.gz 
```

### master:
```
sudo coreos-installer install /dev/sda 
--ignition-url=http://192.168.50.6:8080/master.ign
--image-url=http://192.168.50.6:8080/rhcos.raw.gz 
```

### compute:
```
sudo coreos-installer install /dev/sda 
--ignition-url=http://192.168.50.6:8080/worker.ign
--image-url=http://192.168.50.6:8080/rhcos.raw.gz 
```

## Troubleshooting
![](https://i.imgur.com/UUh5FaM.png)
![](https://i.imgur.com/YdQ0Nan.png)
![](https://i.imgur.com/1TLcHiJ.png)
![](https://i.imgur.com/nJYkJm1.png)
