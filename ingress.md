# Ingress
## 名詞說明
Ingress Class: 當單一叢集擁有多個ingress controller時，用來定義controller名字，方便啟用ingress時，連接正確的ingress controller。
pathTpe: 
* ImplementationSpecific: 根據IngressClass
* Exact: URL必須相同
* Prefix: 以/分隔URL，只要/前的路徑有匹配即可

Deployment: 如果想動態改變Ingress controller replicas數量，缺點若replica數量開不夠造成有些node上沒有controller會影響到有些pod無法被連到。
DaemonSet: 在每個node上部署Ingress controller

## NGINX-ingress/NGINX-ingress plus差異

|                                                                               | nginxinc/kubernetes-ingress with NGINX | nginxinc/kubernetes-ingress with NGINX Plus |
| ----------------------------------------------------------------------------- |:--------------------------------------:|:-------------------------------------------:|
| NGINX 版本                                                                    |              **NGINX主版本**               |                 **NGINX Plus**                  |
| 單一host合併ingress rule                                                      |      通過Mergeable Ingresses支援       |         通過Mergeable Ingresses支援         |
| TCP/UDP                                                                       |            通過客製資源支援            |              通過客製資源支援               |
| Websocket                                                                     |           通過annotation支援           |             通過annotation支援              |
| TCP SSL Passthrough                                                           |            通過客製資源支援            |              通過客製資源支援               |
| HTTP load balancing                                                           |  VirtualServer and VirtualServerRoute  |    VirtualServer and VirtualServerRoute     |
| TCP/UDP load balancing                                                        |            TransportServer             |               TransportServer               |
| TCP SSL Passthrough load balancing                                            |            TransportServer             |               TransportServer               |
| Reporting the IP address(es) of the Ingress controller into Ingress resources |                  支援                  |                    支援                     |
| Prometheus exporter                                                           |                  支援                  |                    支援                     |
| Helm charts                                                                   |                  支援                  |                    支援                     |
| Extended Status                                                               |                 不支援                 |                    支援                     |
| 動態重新配置endpoints                                                         |                 不支援                 |                    支援                     |
| JSON Web Tokens驗證                                                           |                 不支援                 |                    支援                     |
| Session persistence                                                           |                 不支援                 |                    支援                     | 
| 即時監控                                                                      |                 不支援                 |                    支援                     |
## 名詞說明
**VirtualServer:** 用來定義load balance構造
```
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: cafe
  namespace: cafe-ns
spec:
  host: cafe.example.com
  upstreams:
  - name: tea
    service: tea-svc
    port: 80
  routes:
  - path: /tea
    action:
      pass: tea
  - path: /coffee
    route: coffee-ns/coffee
```
**VirtualServerRoute:** 提供route給VirtualServer
```
apiVersion: k8s.nginx.org/v1
kind: VirtualServerRoute
metadata:
  name: coffee
  namespace: coffee-ns
spec:
  host: cafe.example.com
  upstreams:
  - name: latte
    service: latte-svc
    port: 80
  - name: espresso
    service: espresso-svc
    port: 80
  subroutes:
  - path: /coffee/latte
    action:
      pass: latte
  - path: /coffee/espresso
    action:
      pass: espresso
```

## 安裝
### 手動安裝
**預先處理**
安裝NGINX Ingress controller:使用DockerHub上的nginx/nginx-ingress image
安裝NGINX Plus Ingress controller:建構自己的image並推到私人的Docker registry，方法起參考此[連結](https://docs.nginx.com/nginx-ingress-controller/installation/building-ingress-controller-image/)

下載nginxinc/kubernetes-ingress並切換到版本v1.9.0
```
$ git clone https://github.com/nginxinc/kubernetes-ingress/
$ cd kubernetes-ingress/deployments
$ git checkout v1.9.0
```
**1. 配置rbac**
創建namespace & service account
```
$ kubectl apply -f common/ns-and-sa.yaml
```
創建cluster role and cluster role binding for the service account
```
$ kubectl apply -f rbac/rbac.yaml
```
創建App Protect role and role binding
```
$ kubectl apply -f rbac/ap-rbac.yaml
```
**2. 創建Common Resources**
創建secret(包含TLS認證及key)給NGINX的default server
```
$ kubectl apply -f common/default-server-secret.yaml
```
創建config map客製化NGINX configuration
```
$ kubectl apply -f common/nginx-config.yaml
```
創建IngressClass(for Kubernetes >= 1.18)
```
$ kubectl apply -f common/ingress-class.yaml
```
**創建客製化資源**
創建客製化資源定義給VirtualServer, VirtualServerRoute, TransportServer 及 Policy
```
$ kubectl apply -f common/vs-definition.yaml
$ kubectl apply -f common/vsr-definition.yaml
$ kubectl apply -f common/ts-definition.yaml
$ kubectl apply -f common/policy-definition.yaml
```
創建客製化資源定義給GlobalConfiguration
```
$ kubectl apply -f common/gc-definition.yaml
```
創建GlobalConfiguration
```
$ kubectl apply -f common/global-configuration.yaml
```
**NGINX App Protect資源**
創建客製化資源定義給APPolicy及APLogConf
```
$ kubectl apply -f common/ap-logconf-definition.yaml 
$ kubectl apply -f common/ap-policy-definition.yaml 
```
**3. 部屬Ingress Controller**
**3.1 執行Ingress Controller**
**使用Deployment**
使用NGINX:
```
$ kubectl apply -f deployment/nginx-ingress.yaml
```
使用NGINX Plus:
```
$ kubectl apply -f deployment/nginx-plus-ingress.yaml
```
**使用Daemon-set**
使用NGINX:
```
$ kubectl apply -f daemon-set/nginx-ingress.yaml
```
使用NGINX Plus:
```
$ kubectl apply -f daemon-set/nginx-plus-ingress.yaml
```
**3.2 確認Ingress Controller執行中**
```
$ kubectl get pods --namespace=nginx-ingress
```
**4.1 創建Service給Ingress Controller Pods**
使用NodePort
```
$ kubectl create -f service/nodeport.yaml
```
使用LoadBalancer
```
$ kubectl apply -f service/loadbalancer.yaml
```

### 經由helm3
```
$ helm repo add nginx-stable https://helm.nginx.com/stable
$ helm repo update
```
NGINX:
```
$ helm install [名稱] nginx-stable/nginx-ingress
```
NGINX Plus:
```
$ helm install [名稱] nginx-stable/nginx-ingress --set controller.image.repository=myregistry.example.com/nginx-plus-ingress --set controller.nginxplus=true
```


# 資料來源
* NGINX-ingress doc 
https://docs.nginx.com/nginx-ingress-controller/
