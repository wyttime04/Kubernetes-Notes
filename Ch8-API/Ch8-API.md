# CH8 Accessing pod metadata and other resources from applications

[TOC]


## Downward API
Downward API 能將 pod metadata 傳進 pod 內，可透過 **Environment variables** 或 **downwardAPI volume** 傳遞。

![](https://i.imgur.com/efB5kTu.png)

> **Service Account** 授權 pod 存取 downward API

### Available metadata
- `fieldRef`: store pod fields
    - `metadata.name`: the pod's name
    - `metadata.namespace`: the pod's namespace
    - `metadata.uid`: the pod's UID
    - `metadata.labels['<KEY>']`: the value of the pod's label
        - for example, `metadata.labels['mylabel']`
    - `metadata.annotations['<KEY>']`: the value of the pod's annotation
        - for example, `metadata.annotations['myannotation']`
    
- `resourceFieldRef`: store container fields
	- A Container's CPU limit
	- A Container's CPU request
	- A Container's memory limit
	- A Container's memory request
	- A Container's hugepages limit (provided that the `DownwardAPIHugePages` feature gate is enabled)
	- A Container's hugepages request (provided that the `DownwardAPIHugePages` feature gate is enabled)
	- A Container's ephemeral-storage limit
	- A Container's ephemeral-storage request

- downwardAPI volume `fieldRef`:
    - `metadata.labels`: all of the pod's labels, formatted as `label-key="escaped-label-value"` with one label per line
    - `metadata.annotations`: all of the pod's annotations, formatted as `annotation-key="escaped-annotation-value"` with one annotation per line

- environment variables:
	- `status.podIP`: the pod's IP address
	- `spec.serviceAccountName`: the pod's service account name
	- `spec.nodeName`: the name of the node to which the scheduler always attempts to schedule the pod
	- `status.hostIP`: the IP of the node to which the Pod is assigned



### Environment variables
透過 env expose metadata
- `fieldRef` -> `env[].valueFrom.fieldRef`
    - `fieldPath`
- `resourceFieldRef` ->  `env[].valueFrom.resourceFieldRef`
    - `resource`
    - `divisor`


**downward-api-env.yaml**

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    ...
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    ...
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        # A container’s CPU and memory requests and limits are referenced by using resourceFieldRef instead of fieldRef.
        resourceFieldRef:
          resource: requests.cpu
          divisor: 1m
          # For resource fields, you define a divisor to get the value in the unit you need.
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```

![](https://i.imgur.com/M7l7Dl4.png)


### downwardAPI volume
downwardAPI volume 透過檔案傳遞 metadata
- `fieldRef` -> `volumes.downwardAPI.items[].fieldRef`
    - `fieldPath`
- `resourceFieldRef` ->  `volumes.downwardAPI.items[].resourceFieldRef`
    - `containerName`: 指定 resource 來源 container
    - `resource`
    - `divisor`
  > `volumes.downwardAPI.items[].path`: 表示 filename

**downward-api-volume.yaml**
```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    ...
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
    ...
volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      ...
```

:::info
動態更新透過 symbolic link 鏈結 volume 中的檔案實現，更新 metadata 後會創建新的暫存目錄，之後再將 symbolic link 鏈結至新的目錄。
:::

:::info
如果使用 `subPath` 掛載 volume 將會無法接收 Downard API 更新
:::

![](https://i.imgur.com/xgxE5ov.png)


#### Updating labels and annotations
`labels` 和 `annotations` 可以在 pod 運行時隨時修改，如果透過 [Environment variables](#Environment-variables) 在創建 pod 寫入即無法再修改，所以只能透過 donwardAPI volume 傳遞。


---
## Kubernetes API server
Downward API 無法取得其他 pod 或 resource 資訊，必須和 k8s api server 溝通才能取得。

![](https://i.imgur.com/u9CTXlS.png)


**查詢 api server url**
```bash
$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443
```
**使用 curl insecure**
```bash
$ curl https://192.168.99.100:8443 -k
Unauthorized
```


### kubectl proxy
存取 k8s api server 需要取得授權，可以暫時使用 proxy server 在 local 執行。

```bash
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```
```bash
$ curl localhost:8001
{
"paths": [
    "/api",
    "/api/v1",
    ...
```

### explore api server
**Listing the API server’s REST endpoints**
```bash
$ curl http://localhost:8001
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/apps",
    "/apis/apps/v1beta1",
    ...
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v2alpha1",
    ...
```

**Listing endpoints under /apis/batch**
```bash
$ curl http://localhost:8001/apis/batch
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "batch",
  "versions": [
    # batch api group 版本
    {
      "groupVersion": "batch/v1",
      "version": "v1"
    },
    {
      "groupVersion": "batch/v2alpha1",
      "version": "v2alpha1"
    }
  ],
  "preferredVersion": {
    # 建議使用 v1
    "groupVersion": "batch/v1",
    "version": "v1"
  },
  "serverAddressByClientCIDRs": null
}
```

**Resource types in batch/v1**
```bash
$ curl http://localhost:8001/apis/batch/v1
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "batch/v1",
  "resources": [
    {
      "name": "jobs",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        # 對 resource 執行的動作
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ]
    },
    {
      "name": "jobs/status",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

**List of Jobs**
```bash
$ curl http://localhost:8001/apis/batch/v1/jobs
{
  "kind": "JobList",
  "apiVersion": "batch/v1",
  "metadata": {
    "selfLink": "/apis/batch/v1/jobs",
    "resourceVersion": "225162"
  },
  "items": [ {
     "metadata": {
       "name": "my-job",
       "namespace": "default",
       ...
```

**Retrieving a resource in a specific namespace by name**
```bash
$ curl http://localhost:8001/apis/batch/v1/namespaces/default/jobs/my-job
{
  "kind": "Job",
  "apiVersion": "batch/v1",
  "metadata": {
    "name": "my-job",
    "namespace": "default",
    ...
```
> same as `$ kubectl get job my-job -o json`

### talking to API server within a pod
- 可使用 k8s DNS 找到 API server
- 需要搭配 secrets 的 **ca.cert** 和 **token** (檔案放置在 `/var/run/secrets/kubernetes.io/serviceaccount/`)


1. attach container
    ```bash
    $ kubectl exec -it <POD_NAME> bash
    ```
2. kubernetes service
    ```bash
    # ip address
    > env | grep KUBERNETES_SERVICE
    KUBERNETES_SERVICE_PORT=443
    KUBERNETES_SERVICE_HOST=10.0.0.1
    KUBERNETES_SERVICE_PORT_HTTPS=443
    > curl https://10.0.0.1:443
    ```

    ```bash
    # dns
    > curl https://kubernetes
    curl: (60) SSL certificate problem: unable to get local issuer certificate
    ...
    If you'd like to turn off curl's verification of the certificate, use
      the -k (or --insecure) option.
    ```

3. get secrets
    ```bash
    > ls /var/run/secrets/kubernetes.io/serviceaccount/
    ca.crt    namespace    token
    ```

4. communicate with cert
    ```bash
    # with --cacert
    > curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes
    Unauthorized

    # or export to env
    > export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    > curl https://kubernetes
    ```

5. communicate authentication
    ```bash
    > export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    > TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
    > curl -H "Authorization: Bearer $TOKEN" https://kubernetes
    ```

:::info
Disabling role-based access control (RBAC)
如果 cluster 允許使用 RBAC，service account 會有部分功能受限制。
:::


#### get metadata in pod
有部分 metadata 儲存於 pod 內 `/var/run/secrets/kubernetes.io/`，如 namespace (`serviceaccount/namespace`)，可透過讀取檔案直接取的 metadata。

```bash
> cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

```bash
> NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
> curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods
```

#### pod talk to kubernetes
1. app 透過 **ca.cert** 驗證 API server
2. app 透過在 header 加入 `Authorization` bearer **token**
3. 如果要對 API object 操作 CRUD，需要傳遞 **namespace**，CRUD 對應 HTTP `POST`, `GET`, `PATCH`/`PUT`, `DELETE`


![](https://i.imgur.com/Htev6zm.png)



### ambassador containers
要處理 HTTPS, certificates, authentication tokens 步驟過於繁瑣，可以透過 proxy server 取代。建立一個 ambassador container (可使用 image `luksa/kubectl-proxy`) 作為與 Kubernetes API server 的橋樑。

![](https://i.imgur.com/vFChncj.png)


**curl-with-ambassador.yaml**
```yaml=
apiVersion: v1
kind: Pod
...
spec:
  containers:
  ...
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```


![](https://i.imgur.com/r59t8M4.png)


### client libraries
如果需要頻繁和 kubernetes API server 溝通或是需要擴充功能建議改用 client libraries

> [Client Libraries | Kubernetes](https://kubernetes.io/docs/reference/using-api/client-libraries/)

#### swagger UI
Kubernetes API server 可以匯出成 Swagger API，在啟用 kubernetes 時帶入參數 `--enable-swagger-ui=true`，(e.g. `minikube start --extra-config=apiserver.Features.Enable- SwaggerUI=true`)


---
## Question


## Ref
- [Expose Pod Information to Containers Through Files | Kubernetes](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
- [Kubernetes API Concepts | Kubernetes](https://kubernetes.io/docs/reference/using-api/api-concepts/)





 Using the Downward API to pass information into containers
 Exploring the Kubernetes REST API
 Leaving authentication and server verification to kubectl proxy
 Accessing the API server from within a container
 Understanding the ambassador container pattern
 Using Kubernetes client libraries