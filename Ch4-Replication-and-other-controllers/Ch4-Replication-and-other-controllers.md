# CH4 - Replication and other controllers: deploying managed pods

## liveness probe
確保 pods 狀態

### 檢測機制
- **HTTP GET**: HTTP GET response **status code**
- **TCP Socket**: establish **TCP connection** to the port
- **Exec**: command **exit status** code (*code 0 means success*)

### 查看重啟次數及狀態
- `kubectl get po <Pod_Name>`

    ```
     $ kubectl get po <Pod_Name>
        NAME         READY     STATUS    RESTARTS   AGE
        <Pod_Name>   1/1       Running   1          2m
    ```
- `kubectl logs <Pod_Name> --previous`
- `kubectl describe po <Pod_Name>`
    ![](https://i.imgur.com/EBrxhHb.png)
    ![](https://i.imgur.com/RWuSdKZ.png)
    > Exit Code: 137 (128+x, x=process signal number, e.g. `SIGKILL`=9)
    > [signal(7) — Linux manual page](https://man7.org/linux/man-pages/man7/signal.7.html)
    > ![](https://i.imgur.com/NtUIEIJ.png)

### initialDelaySeconds
啟動 container 後延遲檢測

```yaml=
...
spec:
    containers:
      - image: <Image_Name>
        name: <Container_Name>
        livenessProbe:
            httpGet:
                path: /
                port: 8080
            initialDelaySeconds: 15
```
    
### 原則
- 檢測目標: 確保本身運作正常，無關聯其他服務。(e.g. web frontend 因為連動 backend 僅重啟 frontend container 無法解決網站問題，在前端僅檢測前端頁面如 `/heath` 運作正常)
- 輕量化: 盡量保持檢測時間在1秒內，確保不會佔用太多資源。
- 注意無限重啟: 如果應對 container 錯誤的手段只有重啟沒有回復方法會導致 container 不斷重啟。


---

## ReplicationControllers
當 nodeA 被中止，有 ReplicationController 的 pods 會在其他可使用的 node 上創建新的 pods 並繼續納管在 ReplicationController 下。

### Operation

- **label selector**: 篩選納管的 pods
- **replica count**: pods 需求量
- **pod template**: replicas pod 底版

![](https://i.imgur.com/PvMoTSi.png)
![](https://i.imgur.com/rQhgw2F.png)

> ReplicationControllers 只管 pod 存不存在，不管 pod 的內容物，只負責確保 pod 數量，數量不足會複製新的 pod ，數量過多則移除 pod。

### Config & Commands

- **kubia-rc.yaml**
    ```yaml=
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: <RC_Name>
    spec:
      replicas: 3
      selector:
        app: <Label_Name>
    template:
      metadata:
        labels:
          app: <Label_Name>
      spec:
        containers:
        - name: <Container_Name>
          image: <Image_Name>
          ports:
          - containerPort: 8080
    ```
    
- `kubectl get rc`

    ```
    $ kubectl get rc
    NAME      DESIRED   CURRENT   READY     AGE
    <RC_Name> 3         3         2         3m
    ```
    
- `kubectl describe rc <RC_Name>`
- `kubectl edit rc <RC_Name>`: 修改 RC 設定檔, template, scale, etc.
- `kubectl scale rc <RC_Name> --replicas=10`
- `kubectl delete rc <RC_Name> --cascade=false`: 刪除 RC 但不刪除 pods


---

## ReplicaSets
支援以表達式篩選 label

### Config & Commands

- **replicaset.yaml**

    ```yaml=
    apiVersion: apps/v1beta2
    kind: ReplicaSet
    ...
    spec:
      replicas: 3
      selector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - kubia
    ...
    ```
    
    > `apiVersion`: 使用 `kubectl api-versions` 列出來自不同 group 釋出的版本
    > ![](https://i.imgur.com/Zp6YGG4.png)

- `kubectl get <RS_Name>`
- `kubectl describe rs`
- `kubectl delete rs <RS_Name>`

### Label Selector
如果同時定義 `matchLabels` 和 `matchExpressions`，代表 pod 必須兩種都符合

**matchExpressions**
- logical **AND** (&&) operator: `,`
- **Equality-based requirement**: `=`, `==`, `!=`
- **Set-based requirement**: `In`, `NotIn`, `Exists`, `DoesNotExist`

> [Labels and Selectors | Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)


---

## DaemonSets
與 ReplicaSets 散佈 pods 不同，DaemonSets 將 pod 放置於指定的 nodes，可用於監測主機、備份資料等系統服務。

### Config & Commands

- **daemonset.yaml**

    ```yaml=
    apiVersion: apps/v1beta2
    kind: DaemonSet
    metadata:
      name: ssd-monitor
    spec:
      selector:
        matchLabels:
          app: ssd-monitor
      template:
        metadata:
          labels:
            app: ssd-monitor
        spec:
          nodeSelector:
            disk: ssd
          containers:
            - name: main
              image: luksa/ssd-monitor
    ```
    
- `kubectl get ds`
- `kubectl label node <Node_Name> <Label>=<Value>`: label the nodes

---

## Jobs
用於執行臨時性的任務

### Config & Commands

- **job.yaml**

    ```yaml=
    apiVersion: batch/v1
    kind: Job
    ...
    spec:
      template:
        ...
        spec:
          restartPolicy: OnFailure    #執行成功後停止 (default=Always)
          containers:
          ...
    ```

- `kubectl get jobs`
- `kubectl logs <Pod_Name>`

### Multiple Pods

- **job.yaml**
    ```yaml=
    ...
    spec:
      ...
      completions: 5
      parallelism: 2
      template:
      ...
    ```
    
    - `completions`: Job 需要完成次數
    - `parallelism`: 同時運行 Job 數量
    - `activeDeadlineSeconds`: 限制 Job 運行時間
    - `backoffLimit`: 限制 Job 嘗試執行失敗次數

- `kubectl scale job <Job_Name> --replicas 3`: 修改`parallelism`

---

## CronJob
作用類似 linux `crontab`, windows `schtasks` 排程服務

### Config & Commands

- **cronjob.yaml**

    ```yaml=
    apiVersion: batch/v1beta1
    kind: CronJob
    ...
    spec:
      schedule: "0,15,30,45 * * * *"
      jobTemplate:
        spec:
          ...<job>
    ```
    
    > `startingDeadlineSeconds`

---

## Question
- *liveness probe* 檢測機制依據 pod 屬性挑選，網頁可使用 `HTTP GET` 依據網頁回傳判定，能夠參照 http status code，那選擇 `Exec` 自行撰寫的程式有什麼規則可以參照，能否由此方法從 exit code 得到更多資訊？
- *ReplicationControllers* 發現 pod 過多時會移除 pod，會以什麼原則挑選要移除的 pod？
    - 最後建立的優先砍
- yaml 設定檔中的 `apiVersion` 能指定api 版本，從不同 group 釋出的 api 版本差異？ 哪個版本比較適合本地自架的？


- 既然 liveness probe 是偵測 container 的健康度的方法，那有什麼方法可以偵測整個 pod的健康度？
    - liveness probe 會重啟 container，pod 健康度??

---

## Ref
- [signal(7) — Linux manual page](https://man7.org/linux/man-pages/man7/signal.7.html)
- [Kubernetes | kubectl api-versions](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_api-versions/)
- [Labels and Selectors | Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

