# CH9 Deployments: updating applications declaratively
[TOC]

## Deployments
根據前幾章介紹已經創建多種 resource，但要如何更新這些不同的 resource?
- 砍掉舊的 pod 再建立新的
- 使用 [ReplicationControllers](/HJCWOUwZTomCV8EDdXEbdQ#ReplicationControllers) 或 [ReplicaSets](/HJCWOUwZTomCV8EDdXEbdQ#ReplicaSets) 更新 pod template，再手動調控新舊 pod

Deployments 在 ReplicaSets 上附加了一層，用於實現更新 app。

![](https://i.imgur.com/nEqC3bR.png =400x)


---
## Updating applications running in pods
Pod 基於 ReplicaSet，Client 透過 Service 存取 Pod，若要更新 Application 必須重建新Image 版本的 Pod。該如何處理 downtime 或新舊版本同時存在的過渡期?

![](https://i.imgur.com/uGAIyVj.png)


### with downtime
直接修改 ReplicaSets template
- 優點
    - 步驟簡單容易更新
- 缺點
    - 更新版本有 downtime


![](https://i.imgur.com/wzOWhfe.png)


### without downtime
先準備好新版本的 Pod，確認可運行後移除舊的 Pod，會消耗較多硬體資源，也必須處理新舊版本差異性
1. **blue-green deployment**: 準備兩個環境，一個線上使用，另一個備用測試，確認環境無異常就可以切換新環境。

    - 優點
        - 沒有 downtime
        - 線上只會有一個版本
    - 缺點
        - 消耗多一倍的資源
    ![](https://i.imgur.com/bFrZLlh.png)

2. **rolling update**: 逐一替換舊版應用程式，直到線上皆為新版為止。
    - 優點
        - 沒有 downtime
        - 不需花費太多資源
    - 缺點
        - 費時費力易出錯

    ![](https://i.imgur.com/OuKoVsF.png)

---

## Performing an automatic rolling update with a ReplicationController
使用 kubectl 工具協助處理，此方法雖然已過時，但能從此開始進入部署自動化

:::info
`kubectl` 從 v1.11 開始已不支援 `rolling update`，故無法重現此章節的步驟，目前被 `rollout` 取代。
`rollout` 為 Deployment 的管理工具。

> [Remove deprecated command `kubectl rolling-update` · Issue #88051 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/issues/88051)
:::

### steps
#### 1. files
**app.js**
```javascript=
const http = require('http');
const os = require('os');
console.log("Kubia server starting...");
var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  // resp version
  response.end("This is v1 running in pod " + os.hostname() + "\n");
};
var www = http.createServer(handler);
www.listen(8080);
```

**Dockerfile**
```dockerfile=
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

```bash
$ docker build -t kubia:v1 .
```

**kubia-rc-and-service-v1.yaml**
```yaml=
apiVersion: v1
kind: ReplicationController
metadata:
  name: demo-kubia-v1
spec:
  replicas: 3
  template:
    metadata:
      name: demo-kubia
      labels:
        app: demo-kubia
    spec:
      containers:
      - name: demo-nodejs
        image: demo-kubia:v1
        ports:
        - containerPort: 8080
        imagePullPolicy: Never
---
apiVersion: v1
kind: Service
metadata:
  name: demo-kubia
spec:
  type: NodePort
  selector:
    app: demo-kubia
  ports:
  - port: 80
    targetPort: 8080
```
> 若不支援 `LoadBalancer` 可用 `NodePort`

> `imagePullPolicy`: IfNotPresent, Always, Never


#### 2. 執行 `kubectl rolling-update`
```bash
# kubectl rolling-update OLD_CONTROLLER_NAME ([NEW_CONTROLLER_NAME] --image=NEW_CONTAINER_IMAGE | -f NEW_CONTROLLER_SPEC)
$ kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2
Created kubia-v2
Scaling up kubia-v2 from 0 to 3, scaling down kubia-v1 from 3 to 0 (keep 3
     pods available, don't exceed 4 pods)
...
```
![](https://i.imgur.com/taBkTIc.png)

#### 3. 自動建立新的 RC `kubia-v2`
![](https://i.imgur.com/MqvxfGs.png)


#### 4. 自動更新 Pod `label`, RC `selector`
![](https://i.imgur.com/aW3OsiM.png)
![](https://i.imgur.com/tCbtbWw.png)
![](https://i.imgur.com/77UGlEn.png)


#### 5. Scale down RC v1, Scale up RC v2
![](https://i.imgur.com/8riW9SB.png)
![](https://i.imgur.com/GmaPO98.png)


### why obsolete
- 會自動修改使用者創建的物件，Pod `label`, RC `selector`，若 server 為共同使用，可能有誤以為是別人操作的隱憂
    - `kubectl rolling-update` verbose logging `--v` 可以看到更詳細的 log
        
        `$ kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2 --v 6`
        
    - 從 log 查看 kubectl 發送 requests 到 Kubernetes API server
        
        `PUT /api/v1/namespaces/default/replicationcontrollers/kubia-v1`

- 因為 kubectl 由 client 執行，若操作上與 server 失去連線可能會造成 Pod 與 RC 間的關聯異常


---

## Using Deployments for updating apps declaratively
Deployments 基於 ReplicaSets，向上加一層管理層

![](https://i.imgur.com/PVCmzQs.png)


**declarative**

![](https://i.imgur.com/FnCrwHx.png =500x)

> [Buzz Word 1 : Declarative vs. Imperative - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10233761)

### create deployments

**kubia-deployment-v1.yaml**
```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  # remove version in name
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: demo-kubia
      labels:
        app: demo-kubia
    spec:
      containers:
      - name: demo-nodejs
        image: demo-kubia:v1
        ports:
        - containerPort: 8080
        imagePullPolicy: Never
---
apiVersion: v1
kind: Service
metadata:
  name: demo-kubia
spec:
  type: NodePort
  selector:
    app: demo-kubia
  ports:
  - port: 80
    targetPort: 8080
```

1. `kubectl create -f kubia-deployment-v1.yaml --record`
    - `--record`: 紀錄版本資訊
        ```
        Flag --record has been deprecated, --record will be removed in the future
        ```

2. `kubectl rollout status deployment kubia`
    - 查看 deployment 發佈狀態

3. 檢查 Pod, RS: Deployment 給 RS hash value, RS 給 Pod unique id
    - Pod-template-hash label: 是針對 Pod template 進行 hash，故 template 設定相同則 hash value 也會相同
    ![](https://i.imgur.com/L0Q70qb.png)
    ![](https://i.imgur.com/99F1UMa.png)

### update deployments

#### Strategy
`spec.Strategy`
1. `RollingUpdate`: 逐一替換舊版應用程式，直到線上皆為新版為止
2. `Recreate`: 直接修改 ReplicaSets template

#### minReadySeconds
`spec.minReadySeconds`: Deployment Pod 最短可用狀態時間

#### update pod template
以 `RollingUpdate` 為例，修改 pod template image 觸發更新
```bash
$ kubectl set image deployment kubia nodejs=luksa/kubia:v2
deployment "kubia" image updated
```

![](https://i.imgur.com/RsBWM9k.png)

![](https://i.imgur.com/MMrD0Ib.png)
> 完成更新後會發現舊 RS 還存留


:::warning
若 Deployment 中的 Pod 有 reference ConfigMap 或 Secret，修改 ConfigMap 或 Secret 不會觸發 update Deployment，可透過修改 pod template 強制 update
:::


:::info
**Method to modify resource**

| Method          | What it does                                                                                       | Example                                                                                                                                 |
| --------------- | -------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| kubectl edit    | Edit a resource on the server, it will open the object’s manifest in your default editor.          | `kubectl edit deployment kubia`                                                                                                         |
| kubectl patch   | Modifies individual properties of an object.                                                       | `kubectl patch deployment kubia -p '{"spec": {"template": {"spec": {"containers": [{"name": "nodejs", "image": "luksa/kubia:v2"}]}}}}'` |
| kubectl apply   | Apply a configuration to a resource, the file needs to contain the full definition of the resource | `kubectl apply -f kubia-deployment-v2.yaml`                                                                                             |
| kubectl replace | Replace a resource, the object must exist                                                          | `kubectl replace -f kubia-deployment-v2.yaml`                                                                                           |
| kubectl set     | Set specific features on objects                                                                   | `kubectl set image deployment kubia nodejs=luksa/kubia:v2`                                                                              |
:::


### Rolling back a deployment
回溯 apps 版本

#### deploy broken version
```bash
$ kubectl set image deployment kubia nodejs=luksa/kubia:v3
deployment "kubia" image updated

$ kubectl rollout status deployment kubia
Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for rollout to finish: 1 old replicas are pending termination...
deployment "kubia" successfully rolled out
```


#### rollout undo
```bash
$ kubectl rollout undo deployment kubia
deployment "kubia" rolled back
```
- `--to-revision=1`: 指定 rollback 版本

:::info
若有其他 rollout process 正在執行 `rollout undo` 可撤銷原 process，在撤銷的 process 若已有建立的 resource 也會被移除或取代。
:::

:::danger
**請勿自行刪除舊版的 RS！**

Deployment 會依照 `.spec.revisionHistoryLimit` 數量留存舊版本的 RS，RS 會存留 Pod template，Deployment 能藉此回復 application 版本。

![](https://i.imgur.com/QbNOXk7.png)
:::

> `extensions/v1beta1` 無預設 revisionHistoryLimit，`apps/v1beta2` 預設為 10，若訂為 0 會無法回溯版本。

#### rollout history

```bash
$ kubectl rollout history deployment kubia
deployments "kubia":
REVISION    CHANGE-CAUSE
2           kubectl set image deployment kubia nodejs=luksa/kubia:v2
3           kubectl set image deployment kubia nodejs=luksa/kubia:v3
```




### Controlling the rate of the rollout
rollingUpdate properties
```yaml=
...
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

- `maxSurge`: 與機器資源有關，設定可創建的 Pod 數量的最大上限，可以是絕對數字或百分比。(default 25%)
- `maxUnavailable`: 與服務負載有關，設定更新過程中不可用的 Pod 數量上限，可以是絕對數字或百分比。(default 25%)

在計算 Deployment availableReplicas 時會排除 Terminating 的 Pod，availableReplicas 數量必須介在 `replicas - maxUnavailable` ~ ` replicas + maxSurge`，不可用的 Pod 會在 `terminationGracePeriodSeconds` 後自動終止。

![](https://i.imgur.com/boR2feL.png)

![](https://i.imgur.com/OJaE90E.png)



### Pausing the rollout process
暫停 rollout process 可查看 update 狀態，若使用 RollingUpdate 可用以確認新版本環境無誤再繼續 update

:::info
`rollout undo` 不可在 Deployment paused 期間操作，若發現新版異常須先 resume 再 undo。
:::

#### rollout pause
停止更新作業，可以用來確認新版狀態

```bash
$ kubectl set image deployment kubia nodejs=luksa/kubia:v4
deployment "kubia" image updated
        
$ kubectl rollout pause deployment kubia
deployment "kubia" paused
```

#### rollout resume
繼續更新作業

```bash
$ kubectl rollout resume deployment kubia
deployment "kubia" resumed
```


#### Canary deployments
部署新版本進行測試，僅有部分 client 會存取到新版本，確認新版無異常之後再繼續更新版本，線上會同時有新舊版本。

以手動部署的 2 種方法:
1. 在 RollingUpdate 期間藉由 `rollout pause` 停止更新，藉以實現 Canary deployments。
2. 要更新版本時新增另一個 Deployment，手動調控新舊版本的 replicas。

##### kubectl wait
可使用 `kubectl wait` 控制等待部署完成，但 `kubectl wait` 目前還是 Experimental
```bash
$ kubectl wait deployment.apps/demo-kubia \
  --timeout=10m \
  --for=jsonpath='{.status.updatedReplicas}'=2;
```

:::info
- 在更新 Deployment 時 `.status.updatedReplicas` 暫時無法使用
    ```bash
    kubectl apply -f demo-kubia-deploy.yaml; \
    kubectl wait deployment.apps/demo-kubia \
        --timeout=10m \
        --for=jsonpath='{.status.updatedReplicas}'=1; \
    kubectl rollout pause deployment demo-kubia;
    ```
    ![](https://i.imgur.com/EXRwA7c.png)
    ```
    error: updatedReplicas is not found
    ```

- 等待 Deployment `condition=Available`
    ```bash
    kubectl apply -f demo-kubia-deploy.yaml; \
    kubectl wait deployment.apps/demo-kubia \
        --timeout=5s \
        --for=condition=Available=True; \
    kubectl wait deployment.apps/demo-kubia \
        --timeout=10m \
        --for=jsonpath='{.status.updatedReplicas}'=1; \
    kubectl rollout pause deployment demo-kubia;
    ```
    ![](https://i.imgur.com/vMsDQFG.png)
> [kubectl-commands#wait | Kubernetes](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#wait)
:::




### Blocking rollouts of bad versions
搭配 [readiness probes](/0RQHdSBsR7icM4HD55KmtA#readiness-probes) 防止 client 存取到無法使用的 Pod

- **kubia-deployment-v3-with-readinesscheck.yaml**

    `kubectl apply -f kubia-deployment-v3-with-readinesscheck.yaml`
    
    ```yaml=
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-kubia
    spec:
      # To keep the desired replica count unchanged 
      # when updating a Deployment with kubectl apply, 
      # don’t include the replicas field in the YAML.
      replicas: 3
      # If you only define the readiness probe without setting minReadySeconds properly,
      # new pods are considered available immediately 
      # when the first invocation of the readiness probe succeeds.
      #  If the readiness probe starts failing shortly after, 
      #  the bad version is rolled out across all pods. 
      #  Therefore, you should set minReadySeconds appropriately.
      minReadySeconds: 10
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 0
        type: RollingUpdate
      template:
        metadata:
          name: demo-kubia
          labels:
            app: demo-kubia
        spec:
          containers:
          - name: demo-nodejs
            image: demo-kubia:v3
            ports:
            - containerPort: 8080
            imagePullPolicy: Never
            readinessProbe:
              periodSeconds: 1
              httpGet:
                path: /
                port: 8080
    ```

- 查看 rollout status，已建立 1 個 replicas

    ![](https://i.imgur.com/vuKSUj0.png)

- 無法存取到 v3

    ![](https://i.imgur.com/asJqeMC.png =400x)
    ![](https://i.imgur.com/f6XCGhD.png)

- Pod not READY

    ![](https://i.imgur.com/ldiL7rC.png =500x)

- deploy failed
    更新超出 `progressDeadlineSeconds` 部署失敗，contdition=Progressing 會顯示結果 ProgressDeadlineExceeded
    
    ![](https://i.imgur.com/SeA0zXg.png)



### Commands 整理
```
kubectl create -f demo-kubia-deploy.yaml --record
kubectl apply -f demo-kubia-deploy.yaml

kubectl get all --show-labels
kubectl describe deploy demo-kubia
kubectl rollout history deploy demo-kubia
kubectl rollout status deployment demo-kubia

kubectl rollout pause deployment demo-kubia
kubectl rollout resume deployment demo-kubia
```


---
## Ref
- [透過 Kubernetes Deployments 實現滾動升級 | Kubernetes - tachingchen.com](https://tachingchen.com/tw/blog/kubernetes-rolling-update-with-deployment/)
- [[Kubernetes] Deployment Overview | 小信豬的原始部落](https://godleon.github.io/blog/Kubernetes/k8s-Deployment-Overview/)
- [什麼是藍綠部署、金絲雀部署、滾動部署、紅黑部署、AB測試？_kaisesai - MdEditor](https://www.gushiciku.cn/pl/gUOs/zh-tw)
- [Images | Kubernetes](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy)
- [Deployment | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment)




---

## Troubleshooting

### 無法部署 pod 到 master node
1. `kubectl get all` 發現 pod Pending
    
    ![](https://i.imgur.com/8GpB2Hh.png)
    
2. `kubectl describe pod <Pod_Name>` 查看 events

    `0/1 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.`
    
    ![](https://i.imgur.com/mxVpmNR.png)
    
3. `kubectl taint nodes 412cisco5 node-role.kubernetes.io/master-`: 讓 master 也能部署 pod
    
    before:
    ![](https://i.imgur.com/JfkCinQ.png)
    
    ![](https://i.imgur.com/Dp25Bm2.png)
    
    after:
    ![](https://i.imgur.com/L0Qf565.png)
    
    ![](https://i.imgur.com/5nMRqY7.png)

> [Taint and Toleration | Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
> [Kubernetes 兩步安裝一次上手 - tachingchen.com](https://tachingchen.com/tw/blog/kubernetes-installation-with-kubeadm/#troubleshooting)



This chapter covers
 Replacing pods with newer versions
 Updating managed pods
 Updating pods declaratively using Deployment resources
 Performing rolling updates
 Automatically blocking rollouts of bad versions
 Controlling the rate of the rollout
 Reverting pods to a previous version
