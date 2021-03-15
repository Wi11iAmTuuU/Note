# OCP筆記

COP集群中機器分為四種

```
bastion
bootstrap
--------------------
master node
worker(compute) node
```

常用指令區

```
> bastion服務控制

systemctl status dnsmasq 
systemctl status haproxy 
systemctl status httpd

systemctl restart dnsmasq 
systemctl restart haproxy 
systemctl restart httpd

systemctl enable dnsmasq 
systemctl enable haproxy 
systemctl enable httpd

> 取得node資訊
oc get no
oc get nodes
oc get nodes -o wide

> 查看目前登入的帳號
oc whoami

> 登入system:admin
oc login -u system:admin

> 登入
oc login

> 賦予user admin權限
oc adm policy add-cluster-role-to-user cluster-admin [userName] --rolebinding-name=cluster-admin

> 檢視pod
oc get pod
oc get pod -n [namespace]

> 檢視 pod 詳細描述
oc describe pod [podName]
oc describe pod [podName] -n [namespace]

> 刪除pod
oc delete pod [podName]
oc delete pod [podName] -n [namespace]

> 檢視deployment
oc get deploy
oc get deploy -n [namespace]

> 生成service
oc expose deploy/[deploymentName] --name=[serviceName]

> ssh連線上node
ssh core@[dns name]
範例: 
ssh core@bootstrap

> windows連結web主控台
console-openshift-console.apps.ocp.lab.com
```

### -----建立新的bastion與安裝新cluster-----

#### 設備要求 

```
> 申請 [OCP] RHCOS 4.5.6 
> 一個標準OCP共需申請6台機器 可選以下方式 
> 6台 = 1bootstrap + 3 master + 2 worker

> 1. 申請2個[OCP] RHCOS 4.5.6各包含1、5個執行個體
>    安裝完成後bootstrap可以獨立刪除 且易於管理
> 2. 申請3個[OCP] RHCOS 4.5.6各包含1、3、2個執行個體
>    worker cluster可選擇較低規格 節省VM資源

> 規格最低要求
> bootstrap CPU大小 4 core RAM大小 16 G 
> master    CPU大小 4 core RAM大小 16 G 
> worker    CPU大小 2 core RAM大小 8 G 

> 注:規格十分重要 如果規格不夠cluster會建不起來
```

申請一台VM機器做bastion

```
系統 : CentOS
CPU : 4C 
RAM : 8G
IP : 10.255.78.XXX
```

修改 bastion 的 hostname

```
> 修改
hostnamectl set-hostname bastion.ocp.[name].com
> 重啟生效
reboot
```

-----安裝功能----

#### 安裝與設定 dnsmasq

```
>安裝dnsmasq
yum install -y dnsmasq

> 關閉防火牆或者開放53port
systemctl  stop  firewalld
systemctl  disable  firewalld

> 暫時關閉selinux
setenforce 0

> 永久關閉selinux 
> 找到SELINUX= 將enforcing改為permissive
vi /etc/selinux/config

> 重啟生效
reboot

> 創建設定檔
touch /etc/dnsmasq.d/dns.conf
```
```
> 填入內容如下
> 修改所有domain
> 修改ipv4部分為自己要設定的IP
XX = basion IP
OO = boostrap IP
** = master IP X3
## = worker IP X2

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
#### 安裝與設定 haproxy

```
> 安裝haproxy
yum install haproxy 

vi /etc/haproxy/haproxy.cfg
> 透過vi編輯器 或mobaXterm打開
```
```
> 將以下內容複製至haproxy.cfg並根據提示修改
> 修改ipv4部分為自己要設定的IP
XX = basion IP
OO = boostrap IP
** = master IP X3
## = worker IP X2

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
#	server master0	10.255.83.**:80 check
#    server master1	10.255.83.**:80 check
#    server master2	10.255.83.**:80 check
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
#	server master0	10.255.83.**:443 check
#    server master1	10.255.83.**:443 check
#    server master2	10.255.83.**:443 check
    server compute0	10.255.83.##:443 check
    server compute1	10.255.83.##:443 check

```
#### 安裝與設定 httpd 

```
> 安裝httpd
yum install httpd

> 修改httpd.conf 
vi/etc/httpd/conf/httpd.conf 
```
```
> 將
Listen 80
> 改為
Listen 8080
```

#### 啟動功能

```
systemctl enable dnsmasq 
systemctl enable haproxy 
systemctl enable httpd

systemctl restart dnsmasq 
systemctl restart haproxy 
systemctl restart httpd

systemctl status dnsmasq 
systemctl status haproxy 
systemctl status httpd

```

#### 修改resolv.conf 

```
vi /etc/resolv.conf
> 修改內容為下 取代所有原有內容(註解掉或刪除)
```

```
search dynakumo.local ocp.[name].com
nameserver 10.255.78.XX
```

-----

#### 網絡要求 
```
> 此為防火牆設定 開通以下port 如不開可跳過

systemctl stop firewalld

> 所有機器到所有機器 port
> 9000--9999  
> 10250--10259
> 10256
> 4789
> 6081
> 9000--9999
> 30000--32767

> 所有機器到控制機
> 2379--2380
> 6443
> 
> Kubernetes API服務器
> 22623

> 應用程序入口負載均衡器
> 443
> 80
```

#### 生成SSH私鑰並將其添加到代理 

```
> 產生ssh key
ssh-keygen -t rsa -b 4096 -N ''

> ssh-agent作為後台任務啟動該過程
eval "$(ssh-agent -s)"

> 將您的SSH私鑰添加到ssh-agent
ssh-add /root/.ssh/id_rsa
```

#### 獲取安裝程序

![](https://i.imgur.com/KgjJxCx.png)
![](https://i.imgur.com/SkjNtgT.png)
![](https://i.imgur.com/5WNtl26.png)

```
#注意 下載時請右鍵複製網址 並找到4.5.6版本

> 下載 openshift-install-linux-4.5.6.tar.gz 並傳入 bastion /root/

> 下載Pull-Secret.txt 存於本機備用

> 下載 openshift-client-linux-4.5.6.tar.gz 並傳入 bastion /root/

> 下載 rhcos-4.5.6-x86_64-metal.x86_64.raw.gz 並傳入 bastion /root/
```

#### 安裝client( oc 指令工具 )
```
> 解壓縮client
tar xvzf /root/openshift-client-linux-4.5.6.tar.gz

> 查看PATH
echo $PATH

> 移動解壓縮的檔案
mv -f /root/oc /usr/local/bin
mv -f /root/kubectl /usr/local/bin
mv -f /root/README.md /usr/local/bin

> 查看是否有成功
ll /usr/local/bin
輸出> total 153516
輸出> -rwxr-xr-x. 2 root root 78595112 Aug 10 12:49 kubectl
輸出> -rwxr-xr-x. 2 root root 78595112 Aug 10 12:49 oc
輸出> -rw-r--r--. 1 root root      954 Aug 10 12:49 README.md

> 查看是否有成功
oc
輸出>OpenShift Client
輸出>
輸出>This client help...下略
```

#### 手動創建安裝配置文件

```
> 解壓縮openshift-install工具
tar xvf /root/openshift-install-linux-4.5.6.tar.gz

> 創建安裝目錄來儲存安裝文建
mkdir /root/ocp-data
```

install-config.yaml樣本

```
> 透過以下指令創建OCP的安裝設定文件 
touch /root/ocp-data/install-config.yaml

> 透過vi編輯器 或mobaXterm打開
vi /root/ocp-data/install-config.yaml

> 將以下內容複製至install-config.yaml 並根據提示修改
```

```
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
> 透過 openshift-install 工具建造manifests
/root/openshift-install create manifests --dir=/root/ocp-data/
```

#### 群集網絡操作員配置

```
> 修改 /root/ocp-data/manifests/cluster-scheduler-02-config.yml
mastersSchedulable 設為 False
```

#### 創建點火配置文件

```
> 創建ignition文件
  ignition憑證只有24小時，必須在24小時內將cluster建好
  /root/openshift-install create ignition-configs --dir=/root/ocp-data/

> 在目錄中生成以下文件：
> ├──AUTH 
> │├──kubeadmin密碼
> │└──kubeconfig 
> ├──bootstrap.ign 
> ├──master.ign 
> ├──metadata.json 
> └──worker.ign
> 將bootstrap.ign master.ign worker.ign 
> 與之前準備的image (rhcos.raw.gz或rhcos-metal.x86_64.raw.gz)
> 共四個檔案 移動至/var/www/html/
rm -f /var/www/html/bootstrap.ign 
rm -f /var/www/html/master.ign 
rm -f /var/www/html/worker.ign 
cp -f /root/ocp-data/bootstrap.ign /var/www/html/ 
cp -f /root/ocp-data/master.ign /var/www/html/ 
cp -f /root/ocp-data/worker.ign /var/www/html/ 
cp -f /root/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz /var/www/html/rhcos.raw.gz

> 修改權限來讓httpd可以存取
chmod 755 /var/www/ -R
```

使用ISO映像創建Red Hat Enterprise Linux CoreOS（RHCOS）機器

```
> 修改ipv4部分為自己要設定的IP
XX = basion IP
OO = boostrap IP
** = master IP X3
## = worker IP X2

> 連線[OCP] RHCOS 4.5.6 VM 開啟主控台
> 在安裝介面按下TAB
> 保留原有指令 輸入以下指令 沒有換行
> bootstrap範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://10.255.78.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://10.255.78.XX:8080/bootstrap.ign 
ip=10.255.83.OO::10.255.83.253:255.255.255.0:bootstrap.ocp.cat.com:ens192:none 
nameserver=10.255.78.XX

> master範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://10.255.78.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://10.255.78.XX:8080/master.ign 
ip=10.255.83.**::10.255.83.253:255.255.255.0:master0.ocp.cat.com:ens192:none 
nameserver=10.255.78.XX

> compute範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://10.255.78.XX:8080/rhcos.raw.gz 
coreos.inst.ignition_url=http://10.255.78.XX:8080/worker.ign 
ip=10.255.83.##::10.255.83.253:255.255.255.0:compute0.ocp.cat.com:ens192:none 
nameserver=10.255.78.XX

> 輸入完成後 按下enter運行安裝
> 等到RHCOS跑完
> 如安裝錯誤失敗 則會出現等待五分鐘後自動重啟 等待重啟後即可再次嘗試
```

#### 創建集群

```
/root/openshift-install --dir=/root/ocp-data/ wait-for bootstrap-complete --log-level=info

> 等待引導 第一階段約5分鐘左右 第二階段約10分鐘左右

> INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp.cat.com:6443...
> INFO API v1.18.3+002a51f up
> INFO Waiting up to 40m0s for bootstrapping to complete...
> INFO It is now safe to remove the bootstrap resources
> INFO Time elapsed: 10m13s

> 引導過程完成後，請從haproxy中卸下boostrap。
```

#### 登錄集群

```
>確實有在上一步移除bootstrap後
export KUBECONFIG=/root/ocp-data/auth/kubeconfig 

watch -n 1 oc whoami
> 等待輸出system:admin
```

#### 批准您機器的CSR

```
> 批准機器的CSR
oc get nodes
oc get csr
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
watch -n 1 oc get nodes

> 直到所有node都上線與ready為止
> 如等五分鐘以上仍看不到worker node
> 則ctrl+C後重新批准機器的CSR

> 直到所有node都上線與ready為止
watch -n 1 oc get nodes
```

#### 初始User配置

```
> 查看組件 直到所有組件都出現版本號為止 需要5分鐘左右
watch -n1 oc get clusteroperators

> 新增初始User
htpasswd -c -B -b /root/users.htpasswd [Name] [password]

> 將users.htpasswd轉成secret到OCP上
oc create secret generic htpass-secret --from-file=htpasswd=/root/users.htpasswd -n openshift-config

> 將帳密secret綁到登入器裡
touch /root/CR
```
```
> 輸入以下內容
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
oc apply -f /root/CR

> 升權限至cluster-admin
oc adm policy add-cluster-role-to-user cluster-admin [Name] --rolebinding-name=cluster-admin
```

安裝過程中刪除了映像註冊表

```
在不提供共享對象存儲的平台上，OpenShift Image Registry Operator將自身引導為Removed。這樣就可以 openshift-installer在這些平台類型上完成安裝。

安裝完畢後，必須編輯圖片註冊運營商配置切換managementState從Removed到Managed。

Prometheus控制台提供ImageRegistryRemoved警報，例如：

“圖像登記已被刪除。ImageStreamTags，BuildConfigs和 DeploymentConfigs其基準ImageStreamTags預期可能無法正常工作，請配置存儲和更新的配置，以Managed 通過編輯configs.imageregistry.operator.openshift.io狀態。”
```

映像註冊表存儲配置

```
該image-registry操作員最初不適用於不提供默認存儲的平台。安裝後，必須將註冊表配置為使用存儲，以便使註冊表操作員可用。

顯示了有關配置PersistentVolume的說明，這對於生產集群是必需的。在適用的地方，顯示了有關將空目錄配置為存儲位置的說明，該目錄僅適用於非生產群集。

提供了附加說明，以允許映像註冊表通過Recreate在升級過程中使用部署策略來使用塊存儲類型。

oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'

https://docs.openshift.com/container-platform/4.5/registry/configuring_registry_storage/configuring-registry-storage-vsphere.html
```

添加hosts來登入web-console

```
C:\Windows\System32\drivers\etc\hosts

10.255.78.XX console-openshift-console.apps.ocp.[name].com
10.255.78.XX oauth-openshift.apps.ocp.[name].com
10.255.78.XX api.ocp.[name].com
10.255.78.XX downloads-openshift-console.apps.ocp.[name].com

Browser 輸入 console-openshift-console.apps.ocp.[name].com
選擇my_htpasswd後登入
```



-----

詳細安裝說明
https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-bare-metal.html

redhat OCP主控台
https://cloud.redhat.com/openshift/

### -----配置步驟-----

####  新增節點

```
#申請虛擬機[OCP] RHCOS 4.5.6
https://prd-vra.dynakumo.local/vcac/#csp.cs.ui.catalog.list

# bastion上添加dns設定
/etc/dnsmsq.d/dns.conf

master node添加以下項目
host-record=masterX.ocp.lab.com,10.255.83.XX
host-record=etcd-X.ocp.lab.com,10.255.83.XX
srv-host=_etcd-server-ssl._tcp.ocp.lab.com,etcd-X.ocp.lab.com,2380,0,10
address=/masterX.ocp.lab.com/10.255.83.XX
address=/etcd-X.ocp.lab.com/10.255.83.XX

worker node添加以下項目
host-record=host-record=computeX.ocp.lab.com,10.255.83.XX
address=/computeX.ocp.lab.com/10.255.83.XX

# 重啟服務
systemctl restart dnsmasq

# 連線[OCP] RHCOS 4.5.6 VM 開啟主控台
# 在安裝介面按下TAB
# 保留原有指令 輸入以下指令 沒有換行
# bootstrap範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://10.255.78.75:8080/rhcos.raw.gz
coreos.inst.ignition_url=http://10.255.78.75:8080/bootstrap.ign 
ip=10.255.83.200::10.255.83.253:255.255.255.0:bootstrap.ocp.lab.com:ens192:none 
nameserver=10.255.78.75

# master範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://10.255.78.75:8080/rhcos.raw.gz
coreos.inst.ignition_url=http://10.255.78.75:8080/master.ign 
ip=10.255.83.73::10.255.83.253:255.255.255.0:master4.ocp.lab.com:ens192:none 
nameserver=10.255.78.75

# compute範例:
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://10.255.78.75:8080/rhcos.raw.gz
coreos.inst.ignition_url=http://10.255.78.75:8080/worker.ign 
ip=10.255.83.71::10.255.83.253:255.255.255.0:compute4.ocp.lab.com:ens192:none 
nameserver=10.255.78.75

# 輸入完成後 按下enter運行安裝
# 等到RHCOS跑完
# 如安裝錯誤失敗 則會出現等待五分鐘後自動重啟 等待重啟後即可再次嘗試

# 回到bastion上 登入admin
oc login -u admin -p @dmin123456

# 查看待處理的CSR
oc get csr

# 批准所有掛起的CSR
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

#批准node的CSR
#將計算機添加到群集時，
#將為添加的每台計算機生成兩個掛起的證書籤名請求（CSR）。
#您必須確認這些CSR已獲得批准，
#或在必要時自行批准。

#查看計算機列表
oc get nodes 


```

#### 刪除節點

```
# bastion上 登入admin
oc login -u admin -p @dmin123456

#查看計算機列表並記錄RHCOS計算計算機的節點名稱
oc get nodes -o wide
> NAME                   STATUS     ROLES          AGE   VERSION           INTERNAL-IP
> compute0.ocp.lab.com   Ready      <none>         43d   v1.18.3+002a51f   10.255.83.16   
> compute1.ocp.lab.com   Ready      <none>         43d   v1.18.3+002a51f   10.255.83.17   
> compute2.ocp.lab.com   Ready      worker         43d   v1.18.3+002a51f   10.255.83.18   
> compute3.ocp.lab.com   Ready      worker         43d   v1.18.3+002a51f   10.255.83.19   
> compute5.ocp.lab.com   Ready      worker         46m   v1.18.3+002a51f   10.255.83.71   
> infra0.ocp.lab.com     NotReady   infra,worker   44d   v1.18.3+002a51f   10.255.83.14   
> infra1.ocp.lab.com     Ready      infra,worker   43d   v1.18.3+002a51f   10.255.83.15   
> master0.ocp.lab.com    Ready      master         44d   v1.18.3+002a51f   10.255.83.11   
> master1.ocp.lab.com    Ready      master         44d   v1.18.3+002a51f   10.255.83.12   
> master2.ocp.lab.com    Ready      master         44d   v1.18.3+002a51f   10.255.83.13   
> master4.ocp.lab.com    Ready      master         46m   v1.18.3+002a51f   10.255.83.73   

#將節點標記為不可調度
oc adm cordon compute5.ocp.lab.com
> node/compute5.ocp.lab.com cordoned

#排空節點上的所有pod
oc adm drain compute5.ocp.lab.com --force --delete-local-data --ignore-daemonsets
> node/compute5.ocp.lab.com already cordoned
> WARNING: ignoring DaemonSet-managed Pods: openshift-cluster-node-tuning-operator/tuned-4ddsf, openshift-dns/dns-default-9998g, openshift-image-registry/node-ca-zpmnm, openshift-machine-config-operator/machine-config-daemon-t7hlq, openshift-monitoring/node-exporter-7xl48, openshift-multus/multus-wnptt, openshift-sdn/ovs-6rfgd, openshift-sdn/sdn-7trsr

#刪除節點
oc delete nodes compute5.ocp.lab.com
> node "compute5.ocp.lab.com" deleted

#查看列表 確認結果
oc get nodes -o wide
> NAME                   STATUS     ROLES          AGE   VERSION           INTERNAL-IP
> compute0.ocp.lab.com   Ready      <none>         43d   v1.18.3+002a51f   10.255.83.16   
> compute1.ocp.lab.com   Ready      <none>         43d   v1.18.3+002a51f   10.255.83.17   
> compute2.ocp.lab.com   Ready      worker         43d   v1.18.3+002a51f   10.255.83.18   
> compute3.ocp.lab.com   Ready      worker         43d   v1.18.3+002a51f   10.255.83.19   
> infra0.ocp.lab.com     NotReady   infra,worker   44d   v1.18.3+002a51f   10.255.83.14   
> infra1.ocp.lab.com     Ready      infra,worker   43d   v1.18.3+002a51f   10.255.83.15   
> master0.ocp.lab.com    Ready      master         44d   v1.18.3+002a51f   10.255.83.11   
> master1.ocp.lab.com    Ready      master         44d   v1.18.3+002a51f   10.255.83.12   
> master2.ocp.lab.com    Ready      master         44d   v1.18.3+002a51f   10.255.83.13   
> master4.ocp.lab.com    Ready      master         48m   v1.18.3+002a51f   10.255.83.73   

參考資料
https://docs.openshift.com/container-platform/4.5/post_installation_configuration/node-tasks.html
```

#### 增加更多user

```
> 只能用於以上方步驟建立過user的情況
> 新增新帳號密碼至dtpasswd檔案

htpasswd -bB /root/users.htpasswd [name] [password]

> replace secret

oc create secret generic htpass-secret --from-file=htpasswd=/root/users.htpasswd --dry-run -o yaml -n openshift-config | oc replace -f -


> 讓USER可以在CMD介面登入

/root/openshift-install --dir=/root/ocp-data/ wait-for install-complete 

> 升權限至cluster-admin
oc adm policy add-cluster-role-to-user cluster-admin [Name] --rolebinding-name=cluster-admin
```



### -----RBAC 權限管理-----

創建cluster role

```
oc create clusterrole <name> --verb=<verb> --resource=<resource>
```
```
# <name>，本地角色的名稱
# <verb>，以逗號分隔的動詞列表，用於該角色
# <resource>，該角色適用的資源
# 例:創建允許查看pod的role
oc create clusterrole podviewonly --verb=get --resource=pod
```

Local role binding

```
$ oc adm policy who-can <verb> <resource>

指示哪些用戶可以對資源執行操作。

$ oc adm policy add-role-to-user <role> <username>

將指定角色綁定到當前項目中的指定用戶。

$ oc adm policy remove-role-from-user <role> <username>

從當前項目中的指定用戶中刪除給定角色。

$ oc adm policy remove-user <username>

刪除指定用戶及其在當前項目中的所有角色。

$ oc adm policy add-role-to-group <role> <groupname>

將給定角色綁定到當前項目中的指定組。

$ oc adm policy remove-role-from-group <role> <groupname>

從當前項目的指定組中刪除給定角色。

$ oc adm policy remove-group <groupname>

刪除指定組及其在當前項目中的所有角色。
```

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/admin
  uid: b975de3d-9cf6-4108-b8dd-85cf9fa599d4
  resourceVersion: '23094'
  creationTimestamp: '2020-11-06T08:35:28Z'
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: 'true'
  managedFields:
    - manager: kube-apiserver
      operation: Update
      apiVersion: rbac.authorization.k8s.io/v1
      time: '2020-11-06T08:35:28Z'
      fieldsType: FieldsV1
      fieldsV1:
        'f:aggregationRule':
          .: {}
          'f:clusterRoleSelectors': {}
        'f:metadata':
          'f:annotations':
            .: {}
            'f:rbac.authorization.kubernetes.io/autoupdate': {}
          'f:labels':
            .: {}
            'f:kubernetes.io/bootstrapping': {}
    - manager: kube-controller-manager
      operation: Update
      apiVersion: rbac.authorization.k8s.io/v1
      time: '2020-11-06T08:54:00Z'
      fieldsType: FieldsV1
      fieldsV1:
        'f:rules': {}
rules:
  - verbs:
      - create
      - update
      - patch
      - delete
    apiGroups:
      - operators.coreos.com
    resources:
      - subscriptions
  - verbs:
      - delete
    apiGroups:
      - operators.coreos.com
    resources:
      - clusterserviceversions
      - catalogsources
      - installplans
      - subscriptions
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - operators.coreos.com
    resources:
      - clusterserviceversions
      - catalogsources
      - installplans
      - subscriptions
      - operatorgroups
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - packages.operators.coreos.com
    resources:
      - packagemanifests
      - packagemanifests/icon
  - verbs:
      - create
      - update
      - patch
      - delete
    apiGroups:
      - packages.operators.coreos.com
    resources:
      - packagemanifests
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
    resources:
      - secrets
      - serviceaccounts
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
      - image.openshift.io
    resources:
      - imagestreamimages
      - imagestreammappings
      - imagestreams
      - imagestreams/secrets
      - imagestreamtags
      - imagetags
  - verbs:
      - create
    apiGroups:
      - ''
      - image.openshift.io
    resources:
      - imagestreamimports
  - verbs:
      - get
      - update
    apiGroups:
      - ''
      - image.openshift.io
    resources:
      - imagestreams/layers
  - verbs:
      - get
    apiGroups:
      - ''
    resources:
      - namespaces
  - verbs:
      - get
    apiGroups:
      - ''
      - project.openshift.io
    resources:
      - projects
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
    resources:
      - pods/attach
      - pods/exec
      - pods/portforward
      - pods/proxy
      - secrets
      - services/proxy
  - verbs:
      - impersonate
    apiGroups:
      - ''
    resources:
      - serviceaccounts
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - ''
    resources:
      - pods
      - pods/attach
      - pods/exec
      - pods/portforward
      - pods/proxy
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - ''
    resources:
      - configmaps
      - endpoints
      - persistentvolumeclaims
      - replicationcontrollers
      - replicationcontrollers/scale
      - secrets
      - serviceaccounts
      - services
      - services/proxy
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - apps
    resources:
      - daemonsets
      - deployments
      - deployments/rollback
      - deployments/scale
      - replicasets
      - replicasets/scale
      - statefulsets
      - statefulsets/scale
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - autoscaling
    resources:
      - horizontalpodautoscalers
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - batch
    resources:
      - cronjobs
      - jobs
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - extensions
    resources:
      - daemonsets
      - deployments
      - deployments/rollback
      - deployments/scale
      - ingresses
      - networkpolicies
      - replicasets
      - replicasets/scale
      - replicationcontrollers/scale
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - policy
    resources:
      - poddisruptionbudgets
  - verbs:
      - create
      - delete
      - deletecollection
      - patch
      - update
    apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
      - networkpolicies
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - metrics.k8s.io
    resources:
      - pods
      - nodes
  - verbs:
      - create
    apiGroups:
      - ''
      - image.openshift.io
    resources:
      - imagestreams
  - verbs:
      - update
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - builds/details
  - verbs:
      - get
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - builds
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - buildconfigs
      - buildconfigs/webhooks
      - builds
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - builds/log
  - verbs:
      - create
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - buildconfigs/instantiate
      - buildconfigs/instantiatebinary
      - builds/clone
  - verbs:
      - edit
      - view
    apiGroups:
      - build.openshift.io
    resources:
      - jenkins
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
      - apps.openshift.io
    resources:
      - deploymentconfigs
      - deploymentconfigs/scale
  - verbs:
      - create
    apiGroups:
      - ''
      - apps.openshift.io
    resources:
      - deploymentconfigrollbacks
      - deploymentconfigs/instantiate
      - deploymentconfigs/rollback
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - apps.openshift.io
    resources:
      - deploymentconfigs/log
      - deploymentconfigs/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - image.openshift.io
    resources:
      - imagestreams/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - quota.openshift.io
    resources:
      - appliedclusterresourcequotas
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
      - route.openshift.io
    resources:
      - routes
  - verbs:
      - create
    apiGroups:
      - ''
      - route.openshift.io
    resources:
      - routes/custom-host
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - route.openshift.io
    resources:
      - routes/status
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
      - template.openshift.io
    resources:
      - processedtemplates
      - templateconfigs
      - templateinstances
      - templates
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - networkpolicies
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - buildlogs
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
    resources:
      - resourcequotausages
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - packages.operators.coreos.com
    resources:
      - packagemanifests
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - image.openshift.io
    resources:
      - imagestreamimages
      - imagestreammappings
      - imagestreams
      - imagestreamtags
      - imagetags
  - verbs:
      - get
    apiGroups:
      - ''
      - image.openshift.io
    resources:
      - imagestreams/layers
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
    resources:
      - configmaps
      - endpoints
      - persistentvolumeclaims
      - persistentvolumeclaims/status
      - pods
      - replicationcontrollers
      - replicationcontrollers/scale
      - serviceaccounts
      - services
      - services/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
    resources:
      - bindings
      - events
      - limitranges
      - namespaces/status
      - pods/log
      - pods/status
      - replicationcontrollers/status
      - resourcequotas
      - resourcequotas/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
    resources:
      - namespaces
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - apps
    resources:
      - controllerrevisions
      - daemonsets
      - daemonsets/status
      - deployments
      - deployments/scale
      - deployments/status
      - replicasets
      - replicasets/scale
      - replicasets/status
      - statefulsets
      - statefulsets/scale
      - statefulsets/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - autoscaling
    resources:
      - horizontalpodautoscalers
      - horizontalpodautoscalers/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - batch
    resources:
      - cronjobs
      - cronjobs/status
      - jobs
      - jobs/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - extensions
    resources:
      - daemonsets
      - daemonsets/status
      - deployments
      - deployments/scale
      - deployments/status
      - ingresses
      - ingresses/status
      - networkpolicies
      - replicasets
      - replicasets/scale
      - replicasets/status
      - replicationcontrollers/scale
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - policy
    resources:
      - poddisruptionbudgets
      - poddisruptionbudgets/status
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
      - ingresses/status
      - networkpolicies
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - buildconfigs
      - buildconfigs/webhooks
      - builds
  - verbs:
      - view
    apiGroups:
      - build.openshift.io
    resources:
      - jenkins
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - apps.openshift.io
    resources:
      - deploymentconfigs
      - deploymentconfigs/scale
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - route.openshift.io
    resources:
      - routes
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - template.openshift.io
    resources:
      - processedtemplates
      - templateconfigs
      - templateinstances
      - templates
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - build.openshift.io
    resources:
      - buildlogs
  - verbs:
      - '*'
    apiGroups:
      - packages.operators.coreos.com
    resources:
      - packagemanifests
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - ''
      - authorization.openshift.io
    resources:
      - rolebindings
      - roles
  - verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    apiGroups:
      - rbac.authorization.k8s.io
    resources:
      - rolebindings
      - roles
  - verbs:
      - create
    apiGroups:
      - ''
      - authorization.openshift.io
    resources:
      - localresourceaccessreviews
      - localsubjectaccessreviews
      - subjectrulesreviews
  - verbs:
      - create
    apiGroups:
      - authorization.k8s.io
    resources:
      - localsubjectaccessreviews
  - verbs:
      - delete
      - get
    apiGroups:
      - ''
      - project.openshift.io
    resources:
      - projects
  - verbs:
      - create
    apiGroups:
      - ''
      - authorization.openshift.io
    resources:
      - resourceaccessreviews
      - subjectaccessreviews
  - verbs:
      - create
    apiGroups:
      - ''
      - security.openshift.io
    resources:
      - podsecuritypolicyreviews
      - podsecuritypolicyselfsubjectreviews
      - podsecuritypolicysubjectreviews
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - ''
      - authorization.openshift.io
    resources:
      - rolebindingrestrictions
  - verbs:
      - admin
      - edit
      - view
    apiGroups:
      - build.openshift.io
    resources:
      - jenkins
  - verbs:
      - delete
      - get
      - patch
      - update
    apiGroups:
      - ''
      - project.openshift.io
    resources:
      - projects
  - verbs:
      - update
    apiGroups:
      - ''
      - route.openshift.io
    resources:
      - routes/status
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.authorization.k8s.io/aggregate-to-admin: 'true'

```



----------------------------------------------------------------------------------------------------

### -----參考資料-----

[網際網路] 認識網址與網域名稱（Domain Name, URL, DNS）
https://pjchender.github.io/2018/06/06/%E7%B6%B2%E9%9A%9B%E7%B6%B2%E8%B7%AF-%E8%AA%8D%E8%AD%98%E7%B6%B2%E5%9D%80%E8%88%87%E7%B6%B2%E5%9F%9F%E5%90%8D%E7%A8%B1%EF%BC%88domain-name-url-dns%EF%BC%89/

dnsmasq的安裝和配置
https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/36325/

安裝和配置 HAProxy
https://support.kaspersky.com/KWTS/6.1/zh-HantTW/167740.htm

鳥哥的 Linux 私房菜 第二十章、WWW 伺服器(Apache )
http://linux.vbird.org/linux_server/0360apache.php



#### Neli的筆記 -------------------------------------------------- 

(詳細安裝請見此)
https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-bare-metal.html

進入網站https://cloud.redhat.com/

OS下載
https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.5.6/
install client下載
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/

選擇(bare-matel)並下載
openshift-install-linux-4.5.6.tar.gz
openshift-client-linux-4.5.6.tar.gz
下載raw檔

安裝並調整參數(可參照檔案範例檔
dns:	dnsmasq
/etc/dnsmsq.d/dns.conf

loadbalancer:	haproxy
/etc/haproxy.cfg

http:	apache
/etc/conf.httpd.conf

創建檔案
install-config.yaml
參照範例 主要需修改的為baseDomain,metadata:name,pullSecret,sshKey
baseDomain為2層名稱 
例: 
  baseDomain    -> lab.com 
  metadata:name -> ocp

輸入指令
./openshift-install create manifests --dir=<installation_directory>

修改
ocp4/manifests/cluster-scheduler-02-config.yml
Locate the mastersSchedulable parameter and set its value to False.

輸入指令
./openshift-install create ignition-configs --dir=<installation_directory> 

於remote console中按下tab並輸入
coreos.inst.install_dev=sda 
coreos.inst.image_url=<image_URL> 
coreos.inst.ignition_url=http://example.com/config.ign 
ip=<dhcp or static IP address>   
bond=<bonded_interface> 
範例:  
coreos.inst.install_dev=sda coreos.inst.image_url=http://10.255.78.75:8080/rcho.raw.gz coreos.inst.ignition_url=http://10.255.78.75:8080/bootstrap.ign ip=10.255.83.200::10.255.83.253:255.255.255.0:bootstrap.ocp.lab.com:ens192:none nameserver=10.255.78.75

ssh連線上去
ssh core@xxx
範例: 
ssh core@bootstrap

Approve CSR
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

dns.conf移除bootstrap
haproxy移除bootstrap(按照之前的compute也要移除)

添加hosts(為了可以連上去)
C:\Windows\System32\drivers\etc\hosts
<bastion>	console-openshift-console.apps.ocp.lab.com
<bastion>	oauth-openshift.apps.ocp.lab.com

===================================================其他===================================================
(添加使用者)
參考網站: https://docs.openshift.com/container-platform/4.1/authentication/identity_providers/configuring-htpasswd-identity-provider.html

(添加label 用於taint 參考網站: https://access.redhat.com/solutions/5034771)
oc label node infra0.ocp.lab.com node-role.kubernetes.io/infra=
oc label node compute0.ocp.lab.com node-role.kubernetes.io/worker-
