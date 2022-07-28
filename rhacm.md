###### tags: `openshift`
# Red Hat Advanced Cluster Management
## 架構
![](https://i.imgur.com/di98ErT.png)
![](https://i.imgur.com/u503U7I.png)
* Managed Cluster
	* OpenShift 3.11
	* OpenShift 4.x
	* Public Cloud Kubernetes
	* EKS (Amazon Elastic Kubernetes Service)
	* AKS (Azure Kubernetes Service) 
	* GKS
	* IKS (IBM Kubernetes Service)
	* ROKS (RedHat Openshift on IBM Cloud)
* ACM 架構中將 Cluster 分為兩類
	* Hub : Hub 屬於管理平台本身，作為統一管理介面，與 Managed 透過 api 溝通
	* Managed
* 資源需求
	* CPU: 1core
	* RAM: 4G
* 單一管理中心架構
	* cluster只能被一個Hub控管
## 使用說明
### Install Hub
1. 使用operatorhub找到advanced cluster management for kubernetes並安裝
2. 安裝MultiClusterHub CRD
3. 點選route網址

![](https://i.imgur.com/r3S8Qrt.png)

### Import Cluster(Managed)
![](https://i.imgur.com/YdsHKtZ.png)
![](https://i.imgur.com/GFNjonO.png)
* 使用Command Line方式安裝
	* 提供一鍵複製Command Line
	* 到目標Cluster bastion下指令即可

![](https://i.imgur.com/hZnITJq.png)

#### 已知問題
* 如要地端控管公有雲環境
	* 安裝agent後，資料由公有雲傳回地端會有問題
	* cluster資料可能無法正常取得
	* 解決方案
		* 關閉防火牆(資安問題)
		* site to site

### Overview
#### Multi-Cluster Lifecycle Management
![](https://i.imgur.com/PmZfsQ8.png)
* 查看所有clusters摘要
    * 配置錯誤
    * Pod狀態
    * 資源量
    * Clusters issue


#### Multi-Cluster Observability
![](https://i.imgur.com/ok4y16H.png)
* Enhanced multi-cluster OCP metric aggregation with customized allowlist
    * Enhanced multi-cluster metric aggregation
    * Custom metrics and pre defined metrics
* 需要使用Object storage儲存thanos資料

### Deploy Application
* 部署檔案支援多種來源
	* Git Server
	* Helm Repository
	* Object Storage

![](https://i.imgur.com/LbTTv6N.png)
![](https://i.imgur.com/CV1D6ZC.png)
* 部屬cluster可透過Label控管同時部屬到多個cluster
* Time window 調整更新策略
	* 離峰時段更新

![](https://i.imgur.com/Z98gGb4.png)
* 部屬後提供介面管理

![](https://i.imgur.com/fTLMNK5.png)
* 也可在部屬時使用Ansible腳本
	* 須先到Credential設定 Ansible Tower 
* 搭配 Ansible Tower 進行 Automation 
	* Pre for LB
	* Post for notification

![](https://i.imgur.com/k6NnkLc.png)
* YAML檔在Git上

![](https://i.imgur.com/6qR6MQ4.png)

### policy (合規)
![](https://i.imgur.com/8K8S4nT.png)
![](https://i.imgur.com/j0OhvMR.png)
![](https://i.imgur.com/6G9nTxs.png)
* policy相關
	* AC - Access Control
	* AT - Awareness and Training
	* AU - Audit and Accountability
	* CA - Security Assessment and Authorization
	* CM - Configuration Management
	* CP - Contingency Planning
	* IA - Identification and Authentication
	* IR - Incident Response
	* MA - Maintenance
	* MP - Media Protection
	* PE - Physical and Environmental Protection
	* PL - Planning
	* PS - Personnel Security
	* RA - Risk Assessment
	* SA - System and Services Acquisition
	* SC - System and Communications Protection
	* SI - System and Information Integrity

## 設定
### Managed Clusters for OpenShift GitOps
1. 新增Cluster sets
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: all-openshift-clusters
  spec: {}
```
![](https://i.imgur.com/nfgUgzh.png)

2. 新增Namespace bindings
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSetBinding
metadata:
  name: all-openshift-clusters
  namespace: openshift-gitops
spec:
  clusterSet: all-openshift-clusters
```
![](https://i.imgur.com/u58BgDd.png)

3. 新增Placement(納管哪些cluster)
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: all-openshift-clusters
  namespace: openshift-gitops
spec:
  predicates:
  - requiredClusterSelector:
      labelSelector:
        matchExpressions:
        - key: vendor
          operator: "In"
          values:
          - OpenShift
```

4. 新增GitOpsCluster(啟用ArgoCD 並給予Placement)
```yaml
apiVersion: apps.open-cluster-management.io/v1alpha1
kind: GitOpsCluster
metadata:
  name: argo-acm-clusters
  namespace: openshift-gitops
spec:
  argoServer:
    cluster: local-cluster
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    name: all-openshift-clusters
    namespace: openshift-gitops
```

5. 設定完成 (application set 可使用ArgoCD)
![](https://i.imgur.com/9Eia0X4.png)
