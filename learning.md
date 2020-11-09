# Kubernetes,Container學習筆記
> 記得不定期重新整理網頁[name=Albert Tu]
# Container 容器化
## 基本介紹
### Container
Container裡面包含了所有讓應用程式在各種不同環境都能順利執行的必要元素，包括：程式碼、執行時期環境、系統工具、系統程式庫、軟體相依性等等。
Container解決的核心問題是環境建置的問題，讓不同語言(技術)擁有獨立環境，避免掉不同技術互相帶來的影響。
容器虛擬化技術以共享Host OS Kernel的方式來執行App。
### VM VS Container
![](https://i.imgur.com/P8yQRuX.png)
Container是以App為單位的虛擬化技術，
VM是以作業系統為單位的虛擬化技術。
Container間是彼此隔離的，可以在一個OS擁有不同版本的環境建置，不像是VM為了避免衝突，為不同版本建置一台獨立的VM，造成不少浪費。
### 優點
* 輕便
容器佔用的資源比虛擬機少，通常只需幾秒鐘即可啟動。
* 彈性
容器具備高度彈性，不需要分配固定數量的資源。意味著容器能夠更有效地動態使用資源。當一個容器上的需求減少時，釋放額外的資源供其他容器使用。
* 密度
容器化允許創建密集的環境，其中主機的資源被充分利用但不被過度利用。與傳統虛擬機相比，容器化允許更密集的環境容器不需要託管自己的操作系統。
* 性能
當資源壓力很大時，使用容器的應用程式的性能會遠高於使用虛擬機。因為使用傳統的虛擬機，作業系統還必須滿足其自身的內存需求，從主機上獲取寶貴的記憶體。
* 維護效率
只有一個作業系統Kernel，作業系統級別的更新只需要執行一次，以使更改在所有容器中生效。這使得伺服器的操作和維護更加有效率。
### 為什麼要使用Container？
Container會在作業系統層級進行虛擬化，讓多個Container直接在作業系統Kernel之上執行。這表示Container遠比VM輕量，不但共用作業系統Kernel、啟動速度較快，而且使用的記憶體量也比啟動完整作業系統要少很多。
## 名詞介紹
### Image
Image包含了可執行程式碼、支援程式庫、運算要求條件，換句話說就是Container的prototype，每個Container建構時需要一個Image。
## 換句話說
Container容器化是將app的程式碼及相關要求Build成Image，要使用時再將Image建成Container作為佈署使用，提供一個獨立的環境，在不同階段時(例:開發、測試)提供環境的隔離。
## 應用(以Docker為例)
### dockerfile(檔案存local端)
```
FROM centos:7
MAINTAINER jack
RUN yum install -y wget
RUN cd /
ADD jdk-8u152-linux-x64.tar.gz /
RUN wget http://apache.stu.edu.tw/tomcat/tomcat-7/v7.0.82/bin/apache-tomcat-7.0.82.tar.gz
RUN tar zxvf apache-tomcat-7.0.82.tar.gz
ENV JAVA_HOME=/jdk1.8.0_152
ENV PATH=$PATH:/jdk1.8.0_152/bin
CMD ["/apache-tomcat-7.0.82/bin/catalina.sh", "run"]
```
**FROM**:使用到的Docker Image名稱
**COPY**:把local的檔案複製到Docker裡指定路徑
**WORKDIR**:指定工作目錄，用WORKDIR指定的工作目錄，會在build中的每一層中都存在
**RUN & CMD**:用於運行程式，但二者運行的時間點不同，CMD在docker run時運行，RUN是在docker build，推薦使用EXEC命令格式，例:`["yum","-y","install","vim"]`，CMD指令只有最後一個會執行
**MAINTAINER**:說明撰寫和維護這個Dockerfile的人是誰
**ADD**:功能與COPY相近，把Local的檔案複製到Image裡，如果是tar.gz檔複製進去Image時會順便自動解壓縮
**ENV**:定義了環境變量，那麼在後續的指令中，就可以使用這個環境變量
**ARG**:與ENV作用一致。不過作用域不一樣，ARG設置的環境變量僅對Dockerfile內有效

# Kubernetes
## 基本介紹
Kubernetes是用於自動部署、擴充和管理容器化的開源系統。
### 架構
![](https://i.imgur.com/cp4Z6Zx.png)
### 名詞說明
**Pod**
Kubernetes的最基本單位，一個Pod包含一個或多個Container，可以通過Kubernetes API手動管理，也可以委託給控制器來實現自動管理。
**Deployment**
創建並佈署一個或多個相同的Pod，若其中一個Pod發生意外，會由其他Pod接手，在yaml檔spec中需要加入replicas數量(要建立多少個相同的Pod)，並會建立replicationSet管理Pod數量。
**ReplicationSet**
當Cluster運行時如果Pod數量低於該數字，會自動增加pod，反之就會關掉Pod。
雖然ReplicaSet可以獨立使用，但一般還是建議使用Deployment來自動管理 ReplicaSet，這樣就無需擔心跟其他機制的不兼容問題。
**Service**
使用selector找尋相對應label並以部屬的Pod，運用設定的IP位址開放給內部或外部使用。
如果Service以找到相對應的Pod時，將建立一組相對應的endpoint。
**Ingress**
配置提供外部可訪問的URL、負載均衡、SSL、基於名稱的虛擬主機。
![](https://i.imgur.com/knEc3Ka.png)
需要安裝ingress-nginx，若使用helm安裝git上的ingress-nginx，需在ingress-nginx-controler上增加externalIPs。
[詳細內容請點擊此連結](https://hackmd.io/@williamtuuu/B16e68IFw)
**DaemonSet**
確保每個Node都有運行一個特定Pod，當新增Node時，會在新Node部屬特定Pod，也可限制在指定Node部屬特定Pod。
典型用法為建立log收集及監控等等，例如:
* storage daemon，例如：ceph, glusterd...
* 收集log用的daemon，例如：fluentd, logstash...
* 監控用的daemon，例如：Prometheus Node Exporter, collectd, Datadog...

**StatefulSet**
StatefulSet controller會為每一個pod產生一個固定的識別資訊，不會因為pod reschedule後有變動。
有符合特定條件，就需要使用StatefulSet，例如:
* 需要穩定&唯一的網路識別
* 需要穩定的persistent storage
* 佈署 & scale out 的時後，每個 pod 的產生都是有其順序且逐一慢慢完成的
* 進行更新操作時，也是與上面的需求相同

**Job**
建立一個或多個pod來執行工作，並確保指定數量的pod有成功的完成工作並終止，當Job被刪除時，所有相關的pod也都一併會被刪除。
**CronJob**
CronJob就是定時執行的job。
**Namespace**
提供獨立的命名空間，實現部分環境隔離
**NodePort**
通過每个Node上的IP和NodePort提供外部服務，Service可以指定的nodePort只有3000~32767。
**Etcd**
保存所有配置之狀態，其他組件在注意到儲存的變化之後，會變成相應的狀態，當Master發生故障時，也可以透過Etcd復原。
**API Server**
存在於Master Node之下，使用Kubernetes API和JSON over HTTP來提供了Kubernetes的內部和外部介面，負責與Worker Node進行溝通。
**Node**
部署Container的單機器，每個Node都必須具備Container的執行環境(runtime)
**Kubelet**
負責每個Node的執行狀態，Kubelet會監視pod的狀態，檢測到節點故障後，Replication Controller將在其他健康節點上啟動pod。
**Horizontal Pod Autoscaling**
依據監測到的CPU使用率自動的擴展Deployment和ReplicationSet。
**Container Runtime Interface(CRI)**
負責將指定image檔，創成Container的工具，例如:Docker、CRI-O
**Container Network Interface(CNI)**
負責分配容器上的IP位址，如Pod間通訊如果需經過Node，也要經由它處理，例如:Flannel
**Taint & Toleration**
taint設計讓pod不要被分派到特定Worker Node
Toleration設計讓pod分派到某個特定被Taint過的Worker Node上
### yaml檔說明
**apiVersion**
Kubernetes Api版本。
**kind**
對應apiVersion，定義這個yaml檔建立甚麼Object，例如:Pod、Deployment、Ingress等等。
**metadata**
提供Object附加資訊，Name和Label等等。
**label**
label有一定規範：由Key、Value組成，由英文、數字及＿.-組成，但必定使用英文(大小寫皆可)或數字開頭，不得超過63字。
**label selector**
equalily-base:使用=,==,!=,可使用,進行分隔
EX:environment = production, tier != fonted
set-base:使用in,notin,!(表示沒有該label的object)
EX:'environment in(prodction,qa), tier notin(fontes)'
**Annotation**
註解，沒有特定規範。
**spec**
詳細定義要這Object相關資訊。
**image**
Container的基礎，如果沒特別設定，預設在Docker Hub下載image。
**Volumes**
Pod中Container可以共同使用的資料。
**template**
統一Deployment建立Pod的設定，包括metadata以及Pod中的Container。
**nodeSelector**
要分配在哪個node用nodeSelector去找對應label。
基本上不會用到，因為kubernetes會自動分配Pod到不同的Node上，不太需要User自行分配。
**targetPort**
service用，指的是目標Pod的port，通常port跟targetPort會設定一樣。

### yaml檔範例
**Pod**
```
apiVersion: v1
kind: Pod
metadata:
 name: rss-site
 labels:
   app: web
spec:
 containers:
   – name: front-end
     image: nginx
     ports:
       – containerPort: 80
   – name: rss-reader
     image: nickchase/rss-php-nginx:v1
     ports:
       – containerPort: 88
```
**Deployment**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
**Service(NodePort)**
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
      nodePort: 30562
```
**Ingress**
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  defaultBackend:
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```
**DaemonSet**
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```
**StatefulSet**
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-gfs-storageclass"
      resources:
        requests:
          storage: 1Gi
```
### kubectl指令說明
**查看已部屬的Pod**
```
#查看基本資訊
$ kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
hello-minikube-7c77b68cff-ncqmn   1/1     Running   1          3d

#查看更多資訊
$ kubectl get pod -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP           NODE
hello-minikube-7c77b68cff-ncqmn   1/1     Running   1          3d    172.17.0.4   minikube
```
**查詢已部屬deployment**
```
#查看基本資訊
$ kubectl get deployments
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1         1         1            1           3d

#查看更多資訊
$ kubectl get deployments -o wide
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS       IMAGES                       SELECTOR
hello-minikube   1         1         1            1           3d    hello-minikube   k8s.gcr.io/echoserver:1.10   run=hello-minikube
```
**查詢已部屬service**
```
#查看基本資訊
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3d

#查看更多資訊
$ kubectl get services -o wide
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
hello-minikube   NodePort    10.97.219.147   <none>        8080:31472/TCP   3d    run=hello-minikube
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          5d    <none>
```
**取得更詳細(Node/Pod/Deployment)**
```
$ kubectl describe [kind] [name]

#例如 得Node詳細資訊 後面加上名稱
$ kubectl describe nodes my-node
```
**Scale(調整deployment replicas)**
```
kubectl scale deployment/'deployment name' --replicas='quantity'
```
**HPA使用方式**
```
#php-apache CPU使用率超過50%時 自動增長Pod數量 最多到10 最少要1個
kubectl autoscale deployment [name] --cpu-percent=50 --min=1 --max=10
```
也可撰寫yaml檔使用HPA
```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  #自動擴展目標
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  #Pod數量規範
  minReplicas: 1
  maxReplicas: 10
  #啟動自動擴展時間
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      current:
        averageUtilization: 0
        averageValue: 0
```
## 環境安裝(安裝kubeadm在CentOS上為例)
### 必備資源
docker(作為本次container)
Kubetl
Kubelet
Kubeadm
### Docker安裝
**Step 1 安裝依賴套件**
```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```       
**Step 2 新增 yum repository**
```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```     
**Step 3 利用 yum 安裝最新版本 Docker，若要指定版本則在指令後面加上 -<version>**
```
sudo yum install docker-ce
```
**Step 4 安裝完成後啟動Docker**
```
sudo systemctl start docker
```
**Step 5 確認Docker版本**
```
sudo docker version
```
**Step 6 查看Docker詳細狀態**
```
sudo docker info
```
**Step 7 測試Docker是否正確啟動**
```
sudo docker run hello-world
```
### kubelet kubectl kubeadm一起安裝
**關閉swap**
```
swapoff -a && sysctl -w vm.swappiness=0
sed '/swap.img/d' -i  /etc/fstab
```
**安裝**
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```
**設定kubermetes CRI(Docker)**
```
# 創建 /etc/docker
$ mkdir /etc/docker

# 設定 Docker daemon
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# 創建 /etc/systemd/system/docker.service.d
$ mkdir -p /etc/systemd/system/docker.service.d

# 重啟 Docker
$ systemctl daemon-reload
$ systemctl restart docker

# 開機就開啟的話
$ sudo systemctl enable docker
```
### 設定kubeadm
**下載kubeadm image**
```
$ kubeadm config images pull
```
**初始化kubeadm**
```
$ sudo kubeadm init
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
如果要用Flannel使用下方
```
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
如果要用Calico使用下方
```
$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
$ kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
#確認pod運行中
$ watch kubectl get pods -n calico-system
```
**新增worker node**
Master Node 使用kubeadm init 後會產生
開啟另一台VM輸入下面指令
```
$ sudo kubeadm join [token] [manster ip]:6443
```
範例:
```
kubeadm join 192.168.21.129:6443 --token 6u5fmv.l71op5r828pcg3sd --discovery-token-ca-cert-hash sha256:682bd0d4c11900532f06e50600fb86a2765b1a0de4c9514e0d8874b256183112
```
**查詢是否有node建立**
```
$ kubectl get nodes
```
**清除之前設定的kubeadm**
```
sudo kubeadm reset
```
### 錯誤處理

**ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables解決方式**
```
su
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
```
## 進階資訊
### Kubernetes CI/CD
**GitOps**
![](https://i.imgur.com/0HmoCX1.png)
1. CI/CD Pipeline 內不進行任何部署動作
2. 資源的描述狀態 (Yaml/Helm Chart) 放在 Git 裡面，Git 作為 Single Source of Truth的角色
3. Kubernetes 內部有一個 Controller 會定期去偵測 Git 的變化，並且把 Git 內的變動都更新到 Kubernetes 裡面

任何人如果想進行修改，只有一個辦法就是更新Git Repo，一旦Git Repo內描述的Yaml/Helm Chart有任何修改，Kubernetes Cluster內的Controlle 會負責將這些變動的差異性更新到Kubernetes Cluster內。

此外，GitOps的過程中，任何人都不應該直接對Kubernetes Cluster直接操作，因此也不需要將KUBECONFIG這個檔案給分享出去，因此安全性的隱憂也就迎刃而解.
**GitOps總結**
1. 不希望任何人/環境有辦法直接從外部去操作Kubernetes
2. 將所有描述Kubernetes資源狀態的檔案（Yaml/Helm Chart)用Git保存
3. Kubernetes內會安裝相對應的controller來監控Git Repo的更新並且將一切的更新都更新到kubernetes cluster內
4. 透過GitOps這種架構，Kubernetes Cluster本身不需要將API Server的存取方式給暴露出去，對於安全性來說也是減少了一些潛在的問題。
## 好用工具
### Dashboard
**基本介紹**
圖形化顯示kubernetes相關資訊
**使用步驟**
在kubernetes上安裝
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```
啟動
```
kubectl proxy
```
啟動後會產生Dashboard網址
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
**取得dashboard帳號**
```
#step1
kubectl create serviceaccount dashboard-admin-sa

#step2
kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa

#step3 取得dashboard名稱
kubectl get secrets

#step4 取得token
kubectl describe secret 'dashboard名稱'
```
### Helm 3
**基本介紹**
透過給參數的方式，去同時管理與設定Kubernetes中部署所需的yaml檔案
![](https://i.imgur.com/dvGLeXg.png)
**名詞說明**
**chart**: 將一個服務中各種元件的yaml檔打包成chart的集合。
**chart.yaml**: 定義chart的資訊，包括version及Name等等。
**value.yam**l: 定義chart的所有參數，參數會被帶入templates中。
**_helpers.tpl**: 放置模板的地方，自動生成時會包含Name、label及selectorLabel等等，可以自行更動或新增模板至此。
**Repository**: 存放chart的地方。
**Release**: 由chart產生的，一個chart可以被佈署多次，每個release name都不同且獨立運作的。
**指令說明**
`helm create [NAME]`: 創建基礎檔案
`helm search`: 在Helm Hub和Helm Repositories尋找charts
`helm install [NAME] [CHART] `: 部屬charts到Kubernetes
`helm list`: 列出指定Kubernetes Namespace中的releases
`helm show`: 取得chart相關資訊
`helm upgrade [RELEASE] [CHART] `: 用最新的chart更新release
`helm pull [CHART]`: 從Helm hub或Helm repository取得chart
`helm repo`: Add,update,index,list,或者remove chart repositories
`helm package [CHART]`: 把當前目錄中的chart封裝成tgz檔
`helm rollback [RELEASE] [REVISION]`: 更換為其他在package repository版本
**安裝步驟**
安裝Helm 3 (利用Script)
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

**使用步驟**
創建chart
```
helm create [chart-name]
```
會生成下列檔案
```
┌── charts/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```
進入資料夾輸入下方，進行一鍵部署
```
helm install .
```
也可封裝chart成tgz，在進行部屬
```
helm package .
```
部屬chart到kubernetes
--namespace:沒寫就會使用default
```
helm install [部屬名稱] [tgz 名稱] --namespace [名稱]
```
--generate-name: 由helm隨機分配名稱
```
helm install [tgz 名稱] --generate-name
```
**yaml檔介紹**
values.yaml
```
replicaCount: 2
 image:
   repository: hcwxd/blue-whale
 service:
   type: NodePort
   port: 80
 ingress:
   enabled: true
   hosts:
     - host: blue.demo.com
       paths: [/]
```
deployment.yaml
```
apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: {{ include "value-helm-demo.fullname" . }}
 spec:
   replicas: {{ .Values.replicaCount }}
   selector:
     matchLabels:
       app: {{ include "value-helm-demo.fullname" . }}
   template:
     metadata:
       labels:
         app: {{ include "value-helm-demo.fullname" . }}
     spec:
       containers:
         - name: {{ .Chart.Name }}
           image: '{{ .Values.image.repository }}'
           ports:
             - containerPort: 3000
```
service.yaml
```
apiVersion: v1
 kind: Service
 metadata:
   name: {{ include "value-helm-demo.fullname" . }}
 spec:
   type: {{ .Values.service.type }}
   ports:
     - port: {{ .Values.service.port }}
       targetPort: 3000
       protocol: TCP
   selector:
     app: {{ include "value-helm-demo.fullname" . }}
```
ingress.yaml
```
{{- if .Values.ingress.enabled -}}
 {{- $fullName := include "value-helm-demo.fullname" . -}}
 apiVersion: extensions/v1beta1
 kind: Ingress
 metadata:
   name: {{ $fullName }}
 spec:
   rules:
   {{- range .Values.ingress.hosts }}
     - host: {{ .host | quote }}
       http:
         paths:
         {{- range .paths }}
           - backend:
               serviceName: {{ $fullName }}
               servicePort: 80
         {{- end }}
   {{- end }}
 {{- end }}
```
Templates(模板)先定義，當要使用時include進yaml。
為了避免Templates的空格數發生異常，可以使用indent限制空格數。
```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.labels" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.labels" . | indent 2 }}
```
三項文件中的參數會由values.yaml自動填入
helm也有if 
```
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```
如果值為下面的幾種情況，則PIPELINE的結果為false： 
* Bool值為false 
* 數字為零 
* 空的String 
* nil（null） 
* 空的集合（map、slice、tuple、dict、array）

也有內建等運算元
```
{{ if eq .Values.course.python "django" }}web: true{{ end }}

and .Arg1 .Arg2 (Arg1 && Arg2, 及)
or  .Arg1 .Arg2 (Arg1 || Arg2, 或)
eq  .Arg1 .Arg2 (Arg1 == Arg2, 等於)
ne  .Arg1 .Arg2 (Arg1 != Arg2, 不等)
ge  .Arg1 .Arg2 (Arg1 >= Arg2, 大於等於)
gt  .Arg1 .Arg2 (Arg1 >  Arg2, 大於)
le  .Arg1 .Arg2 (Arg1 <= Arg2, 小於等於)
lt  .Arg1 .Arg2 (Arg1 <  Arg2, 小於)
not .Arg1       (!Arg1, 相反)
```
特別注意，因為if和end實際操作會產生空行，那我們需要一種方式去除空行
```
{{ if eq .Values.myvalue "hellworld" }}
result: true
{{ end }}
```
需要下列方式去除空行
```
{{- if eq .Values.myvalue "hellworld" }}
result: true
{{- end }}
```
# Prometheus
## 基本介紹
![](https://prometheus.io/assets/architecture.png)
### 名詞介紹
**Prometheus Server**：收集與儲存時間序列資料，並提供PromQL查詢語言支援。
**Client Library**：客戶端函式庫，提供語言開發來開發產生Metrics並曝露Prometheus Server。當Prometheus Server來Pull時，直接返回即時狀態的Metrics。
**Pushgateway**：主要用於臨時性Job推送。這類Job存在期間較短，有可能Prometheus來Pull時就消失，因此透過一個閘道來推送。適合用於服務層面的 Metrics。
**Exporter**：用來曝露已有第三方服務的Metrics給Prometheus Server，即以Client Library開發的HTTP server。
**AlertManager**：接收來至Prometheus Server的Alert event，並依據定義的Notification組態發送警報，ex:E-mail、Pagerduty、OpenGenie與Webhook等等。
### Metrics 類型
Prometheus Client 函式庫支援了四種主要 Metric 類型：
* **Counter**: 可被累加的 Metric，比如一個 HTTP Get 錯誤的出現次數。
* **Gauge**: 屬於瞬時、與時間無關的任意更動 Metric，如記憶體使用率。
* **Histogram**: 主要使用在表示一段時間範圍內的資料採樣。
* **Summary**： 類似 Histogram，用來表示一端時間範圍內的資料採樣總結。
## 安裝Prometheus+Grafana到kubernetes上
### 用Helm v3安裝
**將stable加入helm repo**
```
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com
```
**搜尋Prometheus**
```
$ helm search repo stable/prometheus
```
**安裝PrometheusOperator**
```
$ helm install --generate-name stable/prometheus-operator
```
**查詢是否安裝成功**
```
$ helm list
$ kubectl get pods
$ kubectl get service
```
**更改service prometheus-operator-159672-prometheus type為NodePort**
```
$ kubectl edit svc prometheus-operator-159672-prometheus
```
**更改service prometheus-operator-1595722742-grafana type為NodePort**
```
$ kubectl edit svc prometheus-operator-1595722742-grafana
```
**查看prometheus及grafana的service NodePort**
```
$ kubectl get service
```
**Grafana預設帳密**
```
username: admin
password: prom-operator
```

# 資料來源
* **Kubernetes官網**
https://kubernetes.io/
* **Kubernetes中文指南**
https://jimmysong.io/kubernetes-handbook/
* **Kubernetes Tutorial**
https://www.tutorialspoint.com/kubernetes/kubernetes_architecture.htm
* **KaiRen's Blog**
https://k2r2bai.com/tags/Kubernetes/
* **從Docker到Kubernetes進階**
https://www.qikqiak.com/k8s-book/
* **Docker安裝**
https://medium.com/ianyc/docker-%E5%9C%A8-centos7-%E4%B8%8A%E5%AE%89%E8%A3%9D-docker%E4%B8%A6%E7%B0%A1%E5%96%AE%E6%B8%AC%E8%A9%A6%E6%87%89%E7%94%A8-506a6e0767de
* **kubelet kubectl kubeadm一起安裝**
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
https://k2r2bai.com/2016/09/29/kubernetes/deploy/kubeadm/
* **Dockerfile教學**
https://zhuanlan.zhihu.com/p/143416842
* **Calico教學**
https://www.kubernetes.org.cn/4960.html
* **Helm教學**
(一)https://blog.csdn.net/weixin_36938307/article/details/105226395
(二)https://blog.csdn.net/weixin_36938307/article/details/105245770
* **Helm官方**
https://helm.sh/
* **Prometheus官方**
https://prometheus.io/
