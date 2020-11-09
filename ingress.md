# Ingress
# nginx-ingress/nginx-ingress plus
## 差異


|   | nginxinc/kubernetes-ingress with NGINX | nginxinc/kubernetes-ingress with NGINX Plus |
| -------- | -------- | -------- |
| NGINX 版本 | NGINX主版本 | NGINX Plus |
| 單一host合併ingress rule | 通過Mergeable Ingresses支援 | 通過Mergeable Ingresses支援 |
| TCP/UDP | 通過客製資源支援 | 通過客製資源支援 |
| Websocket | 通過annotation支援 | 通過annotation支援 |
| TCP SSL Passthrough | 通過客製資源支援 | 通過客製資源支援 |
| JSON Web Tokens驗證 | 不支援 | 支援 |
| Session persistence | 不支援 | 支援 |
| HTTP load balancing | VirtualServer and VirtualServerRoute | VirtualServer and VirtualServerRoute |
| TCP/UDP load balancing | TransportServer | TransportServer |
| TCP SSL Passthrough load balancing | TransportServer | TransportServer |
| Reporting the IP address(es) of the Ingress controller into Ingress resources | 支援 | 支援 |
| Extended Status | 不支援 | 支援 |
| 動態重新配置endpoints | 不支援 | 支援 |

## 安裝(經由helm3)
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
