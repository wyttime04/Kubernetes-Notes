# CH5 Services: enabling clients to discover and talk to pods

[toc]

## pods...

### Pods problems
ReplicaSets 確保 pod 的運行，但要如何調配 pod 對外及 pod 之間的連線？
- pods 需要回應外部請求
- pods 需要找到彼此位置
    - pod 生命週期短，可能因為調配資源而被移除
    - pod 在啟動前才會被分配到 ip，在此之前存在的 pod 無法得知後來的 pod
    - Horizontal scaling 每個 pod 都有自己的 ip，但對外應該以一個 ip 來存取服務


### solve by Service
Service 統一接收來自外部的請求，再分配給下層的 pods
![](https://i.imgur.com/BLvjygA.png)


---

## Config & Commands
Service 同 ReplicationControllers 以 label 斷定底下有哪些 pod

- **kubia-svc.yaml**

    ```yaml=
    apiVersion: v1
    kind: Service
    metadata:
      name: <Service_Name>
    spec:
      type: ClusterIP
      ports:
      - port: <Service_Port>
        targetPort: <Container_Port>
      selector:
        app: <Service_Name>
    ```
    
- `kubectl get svc`

    ```
    $ kubectl get svc
    NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
    kubernetes   10.111.240.1     <none>        443/TCP   30d
    kubia        10.111.249.153   <none>        80/TCP    6m
    ```
    
- `kubectl exec <Container_Name> -- curl -s http://10.111.249.153`: 在 container 中執行指令
    
    ![](https://i.imgur.com/UeQQ72c.png)
    
    :::info
    雙減號 (`--`) 是為了傳遞後方指令參數到 container，否則視同 kubectl 的參數
    :::
    :::info
    `kubectl exec` 功能同 `docker exec`
    [kubectl exec | Kubernetes](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_exec/) & [docker exec | Docker](https://docs.docker.com/engine/reference/commandline/exec/)
    :::

- `kubectl describe svc <Service_Name>`
- `kubectl get endpoints <Service_Name>`
    ```
    $ kubectl get endpoints kubia
    NAME    ENDPOINTS                                         AGE
    kubia   10.108.1.4:8080,10.108.2.5:8080,10.108.2.6:8080   1h
    ```


### Service Types
`spec.type`
- `ClusterIP`: cluster 內部使用

- `NodePort`: 發布 Service 供 cluster 外部使用

- `LoadBalancer`: 發布 Service 供 cluster 外部使用，但僅限雲端

- `ExternalName`: [Service without selector](#Service-without-selector)，回應 CNAME record 給 `externalName` DNS

    :::info
    `ExternalName` 版本需求 kube-dns version 1.7^, CoreDNS version 0.0.8^
    ::: 


### Session Affinity
Service 能以 TCP/UDP 封包辨別 ClientIP，讓請求導向同一個 pod

```yaml=
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP    # default to None
...
```

:::info
k8s 只能處理到 layer 4 無法使用 layer 7 HTTP 中的 cookie
:::
> [OSI model | Wikipedia](https://en.wikipedia.org/wiki/OSI_model)


### Ports
定義 ports
#### Multiple Ports
若要 expose 多個 port 需要為每個 port 命名
```yaml=
apiVersion: v1
kind: Service
metadata:
  name: kubia
  spec:
    ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
    selector:
      app: kubia
```

#### Named Port
可在 pod 定義 Named Port 讓 Service 能直接對應
**Pod**
```yaml=
kind: Pod
spec:
  containers:
  - name: kubia
    ports:
    - name: http  # <Container_Port_Name>
      containerPort: 8080
    - name: https  # <Container_Port_Name>
      containerPort: 8443
```

**Service**
```yaml=
apiVersion: v1
kind: Service
spec:
  ports:
  - name: http
    port: 80
    targetPort: http  # <Container_Port_Name>
  - name: https
    port: 443
    targetPort: https  # <Container_Port_Name>
```

### Discovering services
client 連接 service 底下的 pod 的方法

#### ENV
service 底下的 pod 透過 env 儲存 service 相關設定，如 service host ip, port

```
$ kubectl exec kubia-3inly env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubia-3inly
KUBERNETES_SERVICE_HOST=10.111.240.1 
KUBERNETES_SERVICE_PORT=443
... 
KUBIA_SERVICE_HOST=10.111.249.153 
KUBIA_SERVICE_PORT=80
...
```

#### DNS
k8s 提供 `kube-dns` 服務，pod 一律先從 k8s 內部的 DNS 開始查找，可至 container `/etc/resolv.conf` 檔案查看對應。

:::info
`spec.dnsPolicy`: `Default`, `ClusterFirst`, `ClusterFirstWithHostNet`, `None`，設定 pod 使用 k8s 內部的 DNS

:::
> [Pod's DNS Policy | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)

#### FQDN
pod 可使用 `svc.cluster.local` 存取 service，若在同一個 cluster 可省略

```
curl http://kubia.default.svc.cluster.local
curl http://kubia.default
curl http://kubia
```

:::warning
無法使用 `ping` 連到 service
:::

> pod 為實體 ip，service 為虛擬 ip。
> kube-proxy 透過 `iptables` 定義需要將虛擬 ip 轉派到哪個 pod 的實體 ip。
> [Service IP addresses | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/#ips-and-vips)


---

## 連接 cluster 外部服務
![](https://i.imgur.com/XXKCNoV.png)

### Service without selector
Service 通常以 Pod 作為 Endpoint，但也能以其他類型的服務串接。

1. 建立無 selector 的 Service
    **external-service-externalname.yaml**
    ```yaml=
    apiVersion: v1
    kind: Service
    metadata:
      name: external-service
    spec:
      type: ExternalName
      # externalName: alias for an external service
      externalName: someapi.somecompany.com
      ports:
      - port: 80
    ```

2. 手動建立 Endpoints
    **external-service-endpoints.yaml**
    ```yaml=
    apiVersion: v1
    kind: Endpoints
    metadata:
      name: external-service
    subsets:
      - addresses:
        - ip: 11.11.11.11
        - ip: 22.22.22.22
        ports:
        - port: 80
    ```

    :::warning
    Endpoint ip 不能為 **loopback** (127.0.0.0/8 for IPv4, ::1/128 for IPv6), or **link-local** (169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6)
    :::
    > [Services without selectors | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)

---

## 發布 Service 到 cluster 外

1. Service type **NodePort**
2. Service type **LoadBalancer**
3. resource **Ingress**

### NodePort
- k8s control plane 配置一個對外 port (`.spec.ports[*].nodePort`)
- port 限定範圍 `30000-32767` (`--service-node-port-range`)，
- 配置完成能透過所有 Node 對此 Service 進行存取，若要指定單一 Node 使用 kube-proxy `--nodeport-addresses`
- 存取方式 `<NodeIP>:spec.ports[*].nodePort`, `.spec.clusterIP:spec.ports[*].port`

![](https://i.imgur.com/hZ0GZvi.png)


### LoadBalancer
有提供 loadbalancer 的雲使用
![](https://i.imgur.com/4916URf.png)


#### external connections
可設定外部連線僅限存取進入的 Node 下的 pod，以避免多餘的 Network Hops

```yaml
spec:
  externalTrafficPolicy: Local
  ...
```
![](https://i.imgur.com/PZtHyxr.png)

:::danger
如果 client 從不同 Node 進入會造成 Source Network Address Translation (SNAT)，pod 無法取得真正的 ClientIP
:::


---

## Ingress
- Ingress 對外僅需要一個 port
- Ingress 運行在 layer 7 (application layer) 可執行 HTTP cookie 等功能
![](https://i.imgur.com/NT28VVq.png)


### Config & Commands
- **kubia-ingress.yaml**

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: kubia
    spec:
      rules:
      - host: kubia.example.com  # domain name
        http: paths:
          - path: /
            pathType: Prefix
            backend:
              serviceName: kubia-nodeport
              servicePort: 80
    ```
    
    > [Ingress | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/)

- `kubectl get ingresses`

    ```
    $ kubectl get ingresses
    NAME      HOSTS               ADDRESS          PORTS     AGE
    kubia     kubia.example.com   192.168.99.100   80        29m
    ```
    
![](https://i.imgur.com/yhyxfrv.png)


### rules and paths
```yaml=
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: /kubia
        backend:
          serviceName: kubia
          servicePort: 80
      - path: /foo
        pathType: Prefix
        backend:
          serviceName: bar
          servicePort: 80
  - host: bar.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: bar
          servicePort: 80
```


### pathType
- `ImplementationSpecific`: 依據 IngressClass 設定，IngressClass 用於多個 controller 時

- `Exact`: 完全匹配 path，`path: /aaa` 無法匹配 `/aaa/`

- `Prefix`: 前綴，如 `/api`



### Ingress with TLS
client 與 controller 之間需要加密，controller 與 pod 間則不需要。
k8s 需要 certificate 和 private key (resource Secret)

```
$ openssl genrsa -out tls.key 2048
$ openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj \
    /CN=kubia.example.com
$ kubectl create secret tls <Secret_Name> --cert=tls.cert --key=tls.key
secret "<Secret_Name>" created
```

```yaml=
...
spec:
tls:
- hosts:
    - kubia.example.com
    secretName: <Secret_Name>
...
```


---

## readiness probes

| Resource         | 功能                           | pod failed           |
| ---------------- | ------------------------------ | -------------------- |
| liveness probes  | 判斷 pod healthy               | 重啟、移除、建立 pod |
| readiness probes | 判斷 pod 能接受 client request | X                    |

### 檢測機制
- **HTTP GET**: HTTP GET response **status code**
- **TCP Socket**: establish **TCP connection** to the port
- **Exec**: command **exit status** code (*code 0 means success*)


### Config & Commands
- **kubia-rc-readinessprobe.yaml**

    ```yaml=
    apiVersion: v1
    kind: ReplicationController
    ...
    spec:
      ...
      template:
        ... 
        spec:
          containers:
          - name: kubia
            image: luksa/kubia
            readinessProbe:
              exec:
                command:
                - ls
                - /var/ready
    ```

### 其他注意事項
- pod 盡可能加上 readiness probes，以防對未準備完成的服務分配請求
- 不需要在 readiness probes 終止 pod


---

## Headless Service

### with selectors
透過 selector 自動建立 Endpoints，當存取 Service 時直接回應 A record (IP addresses)，client 可以使用 service domain name 直接存取到 pod

```yaml=
apiVersion: v1
kind: Service
metadata:
  name: kubia-headless
spec:
  clusterIP: None    # set to None
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

#### Discovering pods through DNS
container image `tutum/dnsutils`
```
$ kubectl run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 \ 
    --command -- sleep infinity
pod "dnsutils" created
```

```
$ kubectl exec dnsutils nslookup kubia-headless
...
Name:    kubia-headless.default.svc.cluster.local
Address: 10.108.1.4
Name:    kubia-headless.default.svc.cluster.local
Address: 10.108.2.5
```
```
$ kubectl exec dnsutils nslookup kubia
...
Name:    kubia.default.svc.cluster.local
Address: 10.111.249.153
```


### without selector
用於存取外部服務 [Service without selector](#Service-without-selector)

---

## JSONPath
[JSONPath Support | Kubernetes](https://kubernetes.io/docs/reference/kubectl/jsonpath/)


---

## Question
- Ingress 如果使用 `path: /`, `pathType: Prefix` 作為網頁前端， `path: /api`, `pathType: Prefix` 作為 api 是否可行？
    - `Exact` 先配對，之後以最長 `Prefix` 的優先配對

---

## Ref
- [Service | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service)
- [Ingress | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [[Kubernetes] Service Overview | 小信豬的原始部落](https://godleon.github.io/blog/Kubernetes/k8s-Service-Overview/)



This chapter covers
 Creating Service resources to expose a group of pods at a single address
 Discovering services in the cluster
 Exposing services to external clients
 Connecting to external services from inside the cluster
 Controlling whether a pod is ready to be part of the service or not
 Troubleshooting services
