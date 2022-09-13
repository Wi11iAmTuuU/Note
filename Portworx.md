###### tags: `openshift`
# Portworx with Openshift 
## 架構
![](https://i.imgur.com/qyhUeeQ.png)

## 相關設定
### 安裝 Grafana
1. 部屬grafana-dashboard-config configmap
下載[grafana-dashboard-config.yaml](https://2.1.docs.portworx.com/samples/k8s/pxc/grafana-dashboard-config.yaml)
```shell
$ kubectl -n kube-system create configmap grafana-dashboard-config --from-file=grafana-dashboard-config.yaml
```

2. 部屬grafana-source-config configmap
下載[grafana-datasource.yaml](https://2.1.docs.portworx.com/samples/k8s/pxc/grafana-datasource.yaml)
```shell
$ kubectl -n kube-system create configmap grafana-source-config --from-file=grafana-datasource.yaml
```

3. 部屬grafana-dashboards configmap(Grafana templates)
```shell
$ curl "https://docs.portworx.com/samples/k8s/pxc/portworx-cluster-dashboard.json" -o portworx-cluster-dashboard.json 
$ curl "https://docs.portworx.com/samples/k8s/pxc/portworx-node-dashboard.json" -o portworx-node-dashboard.json
$ curl "https://docs.portworx.com/samples/k8s/pxc/portworx-volume-dashboard.json" -o portworx-volume-dashboard.json 
$ curl "https://docs.portworx.com/samples/k8s/pxc/portworx-performance-dashboard.json" -o portworx-performance-dashboard.json
$ curl "https://docs.portworx.com/samples/k8s/pxc/portworx-etcd-dashboard.json" -o portworx-etcd-dashboard.json
$ kubectl -n kube-system create configmap grafana-dashboards --from-file=portworx-cluster-dashboard.json --from-file=portworx-performance-dashboard.json --from-file=portworx-node-dashboard.json --from-file=portworx-volume-dashboard.json --from-file=portworx-etcd-dashboard.json
```

4. 部屬Grafana
下載[grafana.yaml](https://2.1.docs.portworx.com/samples/k8s/pxc/grafana.yaml)
```shell
$ kubectl apply -f grafana.yaml
```

5. 部屬route
```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: portworx-grafana
  namespace: kube-system
  labels:
    app: grafana
spec:
  host: portworx-grafana-kube-system.apps.ocp.olg.com
  to:
    kind: Service
    name: grafana
    weight: 100
  port:
    targetPort: 3000
  wildcardPolicy: None
```

![](https://i.imgur.com/NDIWx88.png)

![](https://i.imgur.com/GQmlCJq.png)

![](https://i.imgur.com/JRSImRm.png)

![](https://i.imgur.com/eGqxieT.png)


### 進Pod操作pxctl
```shell
$ PX_POD=$(oc get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
$ oc rsh $PX_POD -n kube-system
```
* 偷懶行為
```shell
sh-4.4# cp /opt/pwx/bin/pxctl /usr/local/bin/
```
```shell
sh-4.4# pxctl
px cli

Usage:
  pxctl [command]

Available Commands:
  alerts         px alerts
  auth           pxctl auth
  clouddrive     Manage cloud drives
  cloudmigrate   Migrate volumes across clusters
  cloudsnap      Backup and restore snapshots to/from cloud
  cluster        Manage the cluster
  context        pxctl context
  credentials    Manage credentials for cloud providers
  eula           Show license agreement
  help           Help about any command
  license        Manage licenses
  role           pxctl role
  sched-policy   Manage schedule policies
  secrets        Manage Secrets. Supported secret stores AWS KMS | Vault | DCOS Secrets | IBM Key Protect | Kubernetes Secrets | Google Cloud KMS
  service        Service mode utilities
  status         Show status summary
  storage-policy Manage storage policies for creating volumes
  upgrade        Upgrade PX
  volume         Manage volumes

Flags:
      --ca string            path to root certificate for ssl usage
      --cert string          path to client certificate for ssl usage
      --color                output with color coding
      --config string        config file (default is $HOME/.pxctl.yaml)
      --context string       context name that overrides the current auth context
  -h, --help                 help for pxctl
  -j, --json                 output in json
      --key string           path to client key for ssl usage
      --output-type string   use "wide" to show more details
      --raw                  raw CLI output for instrumentation
      --ssl                  ssl enabled for portworx
  -v, --version              print version and exit

Use "pxctl [command] --help" for more information about a command.
```

### CSI Storage Class
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-io-priority-high
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "1"
  snap_interval:   "70"
  priority_io:  "high"
```
* [**相關參數連結**](https://docs.portworx.com/operations/operate-kubernetes/storage-operations/create-pvcs/dynamic-provisioning/)
* **fs**: none/xfs/ext4 (default: **ext4**)
* **block_size**: block size in KB (default: **32**)
* **repl**: 同步的replicas數量(1~3) (default: **"1"**)
* **priority_io**: 存儲優先度 high/medium/low (default: **low**)
* **snap_interval**: 多久觸發一次(單位: 分鐘),0是禁用 (default: **"0"**)
* **ephemeral**: 是否啟用臨時Volume `true/false` (default **"false"**)
* **proxy_endpoint**: NFS proxy網址,格式`<protocol>://<endpoint>` EX:`nfs://192.168.50.11`
* **proxy_nfs_exportpath**: NFS export位置 EX:`/OCPNfs/pv15`

### 靜態設定相關
==※PV名稱必須與portworx volume名稱相同==
![](https://i.imgur.com/EculMkS.png)

### 移除PortWorx
#### 使用Operator安裝
```shell
$ kubectl edit -n kube-system storagecluster <storagecluster_name>
```
* 在spec中新增deleteStrategy部分才能完整移除PortWorx
* type有二種`Uninstall`/`UninstallAndWipe`
    * `Uninstall`: 移除PortWorx
    * `UninstallAndWipe`: 移除PortWorx並永久刪除所有檔案，包括Portworx metadata
```yaml
apiVersion: core.libopenstorage.org/v1
kind: StorageCluster
metadata:
  name: portworx
  namespace: kube-system
spec:       
  deleteStrategy:
    type: Uninstall
```
```shell
$ kubectl delete StorageCluster <your-storagecluster> -n kube-system
```

## 測試
### 靜態測試(基本)
```shell
$ pxctl volume create pv-portworx-static --size=2
```
![](https://i.imgur.com/GpmNluX.png)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-portworx-static
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  portworxVolume:
    volumeID: '681202150650744391'
```
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-portworx-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  volumeName: pv-portworx-static
```
```yaml
apiVersion: v1
kind: Pod
metadata:
   name: nginx-portworx-static
spec:
  containers:
  - image: nginx
    name: nginx-portworx-static
    volumeMounts:
    - mountPath: /test-portworx-volume
      name: portworx
  volumes:
  - name: portworx
    persistentVolumeClaim:
      claimName: pvc-portworx-static
```
![](https://i.imgur.com/oiC9DzY.png)
![](https://i.imgur.com/x69DzNt.png)

### 靜態測試(NFS)
```shell
$ pxctl volume create pv-portworx-nfs-static --size=2 --proxy_endpoint=nfs://192.168.50.185 --proxy_nfs_exportpath=/volume1/ocp-portworx-test01 --mount_options='vers=4.0'
```
![](https://i.imgur.com/eQwJfd0.png)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-portworx-nfs-static
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  portworxVolume:
    volumeID: '793922988093500412'
```
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-portworx-nfs-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  volumeName: pv-portworx-nfs-static
```
```yaml
apiVersion: v1
kind: Pod
metadata:
   name: nginx-portworx-nfs-static
spec:
  containers:
  - image: nginx
    name: nginx-portworx-static
    volumeMounts:
    - mountPath: /test-portworx-volume
      name: portworx
  volumes:
  - name: portworx
    persistentVolumeClaim:
      claimName: pvc-portworx-nfs-static
```
![](https://i.imgur.com/YRsPYXx.png)
![](https://i.imgur.com/JAaF15C.png)

### CSI測試(基本)
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: portworx-sc-test
provisioner: kubernetes.io/portworx-volume
parameters:
  ephemeral: 'true'
  repl: '1'
```
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
  namespace: pocapp01
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: portworx-sc-test
  volumeMode: Filesystem
```
![](https://i.imgur.com/5g1PU3h.png)
![](https://i.imgur.com/FUl0jVL.png)
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: portworx-postgres
  namespace: pocapp01
spec:
  volumes:
    - name: portworx
      persistentVolumeClaim:
        claimName: test-pvc
  securityContext:
    allowPrivilegeEscalation: false
    runAsUser: 0
  containers:
    - name: postgresql
      image: postgres:10.15-alpine
      env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: postgres12345
      ports:
        - name: postgresql
          containerPort: 5432
          protocol: TCP
      volumeMounts:
        - name: portworx
          mountPath: /var/lib/pgsql/data
```
![](https://i.imgur.com/I59nc6P.png)
![](https://i.imgur.com/xEyqMWq.png)


### CSI測試(NFS proxy)
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: portworx-proxy-nfs
provisioner: kubernetes.io/portworx-volume
parameters:
  mount_options: vers=4.0
  proxy_endpoint: 'nfs://192.168.50.11'
  proxy_nfs_exportpath: /OCPNfs/pv15
```
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc-nfs
  namespace: pocapp01
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: portworx-proxy-nfs
  volumeMode: Filesystem
```
![](https://i.imgur.com/wWRtbwR.png)
![](https://i.imgur.com/p7Cfd8j.png)
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: portworx-postgres-nfs
  namespace: pocapp01
spec:
  volumes:
    - name: portworx
      persistentVolumeClaim:
        claimName: test-pvc-nfs
  securityContext:
    allowPrivilegeEscalation: false
    runAsUser: 0
  containers:
    - name: postgresql
      image: postgres:10.15-alpine
      env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: postgres12345
      ports:
        - name: postgresql
          containerPort: 5432
          protocol: TCP
      volumeMounts:
        - name: portworx
          mountPath: /var/lib/pgsql/data
```
![](https://i.imgur.com/hGRg2Sx.png)
![](https://i.imgur.com/xLHDxM0.png)

### POD Volume Enter Maintenance Mode
* StorageClass
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: portworx-csi-sc
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: '1'
```

* PVC
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: portworx-csi-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: portworx-csi-sc
  volumeMode: Filesystem
```

* Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portworx-busybox-deploy
  labels:
    name: portworx-busybox-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portworx-busybox-deploy
  template:
    metadata:
      labels:
        app: portworx-busybox-deploy
    spec:
      containers:
      - image: busybox
        name: busybox-portworx
        command: ["tail","-f","/dev/null"]
        volumeMounts:
        - mountPath: /test-portworx-volume
          name: portworx
      volumes:
      - name: portworx
        persistentVolumeClaim:
          claimName: portworx-csi-pvc
```

---
#### **第一次測試**
![](https://i.imgur.com/sLmkrbc.png)
* 進入維護模式
![](https://i.imgur.com/oG2Z7sB.png)
* Pod依舊被砍掉
* 且deployment新pod無法啟用
![](https://i.imgur.com/flho57c.png)
* volume跑到compute2
![](https://i.imgur.com/z5evu9e.png)
* 離開維護模式
![](https://i.imgur.com/egP9Eqs.png)
* 新Pod測試存取 舊資料依舊存在
![](https://i.imgur.com/7MXVCvN.png)

---
#### **第二次測試**
* Portworx Volume現況
![](https://i.imgur.com/4jw89KM.png)
* 存取資料測試
![](https://i.imgur.com/JU9Cfzb.png)
* 將208(compute3)進維護模式
![](https://i.imgur.com/3urrIZQ.png)
* Pod倒了
![](https://i.imgur.com/mTS6EY5.png)
* Portworx似乎無法復原Volume
![](https://i.imgur.com/tCjijaq.png)
* 將208(compute3)離開維護模式
![](https://i.imgur.com/s11IcZt.png)
* Volume未搬移到其他Node上
![](https://i.imgur.com/40pZhvS.png)
* 測試存取，資料依舊存在
![](https://i.imgur.com/DDpbbbD.png)

---
#### **第三次測試**
* Portworx Volume現況
![](https://i.imgur.com/9SpMPCr.png)
* 存取資料測試
![](https://i.imgur.com/rBLgSqc.png)
* 將208(compute3)進維護模式
![](https://i.imgur.com/a9UyBvy.png)
* Pod倒了 也無法重新生成 
![](https://i.imgur.com/MFc6uJC.png)
* 將208(compute3)離開維護模式
![](https://i.imgur.com/8TXwrhf.png)
* Volume未搬移到其他Node上
![](https://i.imgur.com/r1pCgxN.png)
* 測試存取，資料依舊存在
![](https://i.imgur.com/5oWn49z.png)


#### 測試結果
* 第一次測試
> * Node進入維護模式後，Volume無法正常存取，K8s會將Pod Kill掉
> * 新起的Pod也無法Attach到Volume造成無法啟用
> * Volume Attach另一個Node
> * Volume正常後，Pod工作恢復正常，資料也未消失

---
* 第二次測試
> * Node進入維護模式後，Volume無法正常存取，K8s會將Pod Kill掉
> * 新起的Pod因`failed filter with extender at URL http://stork-service.kube-system:8099/filter, code 400`導致狀態Pending
> * pod event顯示`No online node found with volume replica` 找不到volume replication，也無法將資料從208轉移到206
> * 208(compute3)離開維護模式
> * Pod工作恢復正常，資料也未消失

---
* 第三次測試
> * 結果與第二次測試結果並無不同

### POD Volume Storage Class : repl: “2
* StorageClass
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: portworx-csi-sc2
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: '2'
```

* PVC
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: portworx-csi-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: portworx-csi-sc2
  volumeMode: Filesystem
```

* Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portworx-busybox-deploy
  labels:
    name: portworx-busybox-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portworx-busybox-deploy
  template:
    metadata:
      labels:
        app: portworx-busybox-deploy
    spec:
      containers:
      - image: busybox
        name: busybox-portworx
        command: ["tail","-f","/dev/null"]
        volumeMounts:
        - mountPath: /test-portworx-volume
          name: portworx
      volumes:
      - name: portworx
        persistentVolumeClaim:
          claimName: portworx-csi-pvc
```

---
#### **第一次測試**
* Portworx Volume現況(與`repl: 1`差異在`HA`欄位為2)
![](https://i.imgur.com/OOHIhBZ.png)
* 查看Volume詳細內容 `Replica sets on nodes`及`Replication Status`皆有數據
![](https://i.imgur.com/2eF9OBF.png)
* 存取資料測試
![](https://i.imgur.com/oqLMmHO.png)
* 將208(compute3)進維護模式
![](https://i.imgur.com/a9UyBvy.png)
* Pod倒了並重新長出
![](https://i.imgur.com/RIJRNiu.png)
* Volume自動搬移到206(compute2)上
![](https://i.imgur.com/LJuCNVJ.png)
* 測試新Pod 就資料依舊存在
![](https://i.imgur.com/V00p7Ax.png)
