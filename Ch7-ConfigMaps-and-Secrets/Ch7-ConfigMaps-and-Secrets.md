# CH7 ConfigMaps and Secrets: configuring applications

[TOC]

## 配置 container
部署服務、應用程式都需要配置設定，配置設定可能會根據服務對象、部署環境而有不同。

1. 透過 command-line 參數傳入
2. 寫入環境變數 env
3. 儲存在 config 檔透過 volume 掛載


---
## 透過 command-line 參數傳入
自定義 `entrypoint.sh` 處理參數傳入 container 後要執行的 executable。

### Docker ENTRYPOINT & CMD
- `ENTRYPOINT`: 定義啟動 container 後要使用的 executable
- `CMD`: 傳送參數給 executable

`docker run <image>`: 使用預設的參數啟動 container
`docker run <image> <arguments>`: 複寫 dockerfile 定義的 `CMD`

#### shell & exec
`ENTRYPOINT` & `CMD` 支援兩種格式
- `shell`: `ENTRYPOINT node app.js`
- `exec`: `ENTRYPOINT ["node", "app.js"]`

但使用 shell 會多一個**啟用 shell** 的 process
- `/bin/sh -c node app.js`: 使用 shell 執行指令
- `node app.js`: 實際執行 node

### 使用參數
**./entrypoint.sh**: 建立 `entrypoint.sh` 接收參數
```shell=
#!/bin/bash
MY_VAR=$1
echo MY_VAR is $MY_VAR
```

**Dockerfile**: 透過 `CMD` 傳入參數給 `/bin/entrypoint.sh`
```dockerfile=
FROM ubuntu:latest
ADD entrypoint.sh /bin/entrypoint.sh
ENTRYPOINT ["/bin/entrypoint.sh"]
CMD ["10"]
```

### k8s 複寫參數
| Docker     | Kubernetes | 用途                                     |
| ---------- | ---------- | ---------------------------------------- |
| ENTRYPOINT | command    | 定義啟動 container 後要使用的 executable |
| CMD        | args       | 傳送參數給 executable                    | 

**pod-args.yaml**
```yaml=
apiVersion: v1
kind: Pod
spec:
  containers:
  - image: <my_image>
    args: ["2"]    # MY_VAR
```

`args` 以不同格式傳入:
```yaml=
...
spec:
  containers:
  - image: <my_image>
    args:
    - string
    - "3"
```
:::info
`args` 數字一定要使用`"`，字串不必要。
:::


---
## 寫入環境變數 env
自定義 `entrypoint.sh` 調用環境變數後傳入 container 後要執行的 executable。

![](https://i.imgur.com/pgddjtK.png =300x)

### 使用環境變數
**./entrypoint.sh**: 調用環境變數 `$(VAR_NAME)`
```shell=
#!/bin/bash
echo FOO is $FOO. ABC is $ABC
```


**Dockerfile**: 透過 `ENV` 寫入
```dockerfile=
FROM ubuntu:latest
ADD entrypoint.sh /bin/entrypoint.sh
ENV FOO=BAR
ENV ABC=123
ENTRYPOINT ["/bin/entrypoint.sh"]
CMD ["10"]
```


### k8s 複寫 env
**pod-args.yaml**: 定義環境變數 
```yaml=
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: A
    image: <my_image>
    env:    # $FOO=FOOBAR, $ABC=567
    - name: FOO
      value: FOOBAR
    - name: ABC
      value: "567"
```

```yaml=
...
spec:
  containers:
  - name: A
    image: <my_image>
    env:    # $FIRST=1, $SECOND=12$(THIRD), $THIRD=123
    - name: FIRST
      value: "1"
    - name: SECOND
      value: "12$(THIRD)"
    - name: THIRD
      value: "$(FIRST)23"
```
:::info
env 可以調用前面定義過的，但如果順序錯誤則會無法代換！
:::


---
## ConfigMap
Decoupling(解耦) 將應用程式及其設定分開，更易於配置於不同部署情境。在 k8s 中在同樣的 pod 掛載不同 ConfigMap 就能達到部署不同情境的效果。

**Pod 掛載 ConfigMap**
![](https://i.imgur.com/As5djwE.png =600x)

**在不同情境使用不同 ConfigMap**
![](https://i.imgur.com/KkO63Cr.png =600x)


### Config & Commands
- `kubectl create configmap`
    - `--from-literal=<KEY>=<VALUE>`: 定義 ConfigMap entry，可給多個 entry。
    - `--from-file=<MY_FILE>`: 從檔案匯入，也可給 KEY
    - `--from-file=<MY/PATH/>`: 從資料夾匯入
    - `--from-env-file=<MY/PATH/>`: 從檔案匯入，會省略註解、空白行
    
    ![](https://i.imgur.com/7LqyfVy.png =600x)

- **configmap.yaml**
    ```yaml=
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: game-demo
    data:
      # simple value
      the_number: "123"
      the_string: "aaa"
      
      # file-like keys
      game.properties: |  # 換行符號
        enemy.types=aliens,monsters
        player.maximum-lives=5   
    ```
    - `immutable`: 不更新，減少 watch ConfigMap 的負擔

- `kubectl get configmap <CONFIGMAP_NAME> -o yaml`
- `kubectl edit configmap <CONFIGMAP_NAME>`

### pod 掛載 ConfigMap
創建好 ConfigMap 後可掛載在 pod 使用，若掛載時 ConfigMap 不存在則該 container 無法啟動，待 ConfigMap 建立完成 container 會自動啟動。

![](https://i.imgur.com/i2EJ7wI.png =600x)

:::info
若要忽略不存在 ConfigMap 可設定 `configMapKeyRef.optional: true`
:::

:::info
- name of a ConfigMap must be a valid [DNS subdomain name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names).
- key must consist of alphanumeric characters, -, _ or .
:::

#### env.valueFrom
`containers.env.valueFrom.configMapKeyRef` 個別定義 env 對應 configMap

```yaml=
apiVersion: v1
kind: pod
...
spec:
  containers:
  - image: alpine
    env:
    - name: <MY_VAR>
      valueFrom:
        configMapKeyRef:
          name: <CONFIGMAP_NAME>
          key: <ENTRY_KEY>
```

#### envFrom
`containers.env.envFrom` 將 ConfigMap 的值都匯入 container env

```yaml=
apiVersion: v1
kind: pod
...
spec:
  containers:
  - image: alpine
    envFrom:
    - prefix: <PREFIX_>  # optional, 給予 env prefix
      configMapRef:
        name: <CONFIGMAP_NAME>
```

:::info
使用 `envFrom` 會略過所有非法的 key 值，若有非法的 name 會記錄在 log (`REASON: InvalidVariableNames`)
```bash
$ kubectl get events

LASTSEEN FIRSTSEEN COUNT NAME          KIND  SUBOBJECT  TYPE      REASON                            SOURCE                MESSAGE
0s       0s        1     dapi-test-pod Pod              Warning   InvalidEnvironmentVariableNames   {kubelet, 127.0.0.1}  Keys [1badkey, 2alsobad] from the EnvFrom configMap default/myconfig were skipped since they are considered invalid environment variable names.
```
:::




### 透過 command-line 傳遞 ConfigMap 參數
掛載 ConfigMap 後也可複寫 `args` 傳遞給 `command`

![](https://i.imgur.com/ftsuFLo.png =600x)

```yaml=
apiVersion: v1
kind: pod
...
spec:
  containers:
  - image: alpine
    env:
    - name: MY_ENV_VAR
      valueFrom:
        configMapKeyRef:
          name: <CONFIGMAP_NAME>
          key: <ENTRY_KEY>
    args: ["$(MY_ENV_VAR)"]
```


### configMap volume
將匯入 ConfigMap entry 的檔案透過 volume 掛載進 pod 供 container 使用。

![](https://i.imgur.com/W3hbczO.png)

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    ...
    - name: config
      # nginx 可讀該目錄下所有 .conf 當作設定檔
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ...
  volumes:
  ...
  - name: config
    configMap:
      name: fortune-config
      defaultMode: "6600"    # -rw-rw------
```

- `volumes.configMap.defaultMode`: 更改掛載後的目錄權限


:::warning
在 Linux 系統下使用 volume 掛載目錄進 container，若原 container 該目錄下有檔案會被清除，僅留存 volume 的內容。
:::

#### volume 選擇單一 entry
可使用 `volumes.configMap.items` 選擇要使用的 ConfigMap entry 掛載到 pod，`key`: entry，`path`: 要儲存成的檔案名

```yaml=
...
spec:
  volumes:
  ...
  - name: config
    configMap:
      name: fortune-config
      items:    
      - key: my-nginx-config.conf
        path: gzip.conf
```

#### container 使用 volume subPath
透過 `containers.volumeMounts.subPath` 選擇要掛載進 container 的目錄

```yaml=
...
spec:
  containers:
  - image: some/image
    volumeMounts:
    - name: myvolume
      mountPath: /etc/someconfig.conf
      subPath: myconfig.conf
```


### 更新 config
使用 env, command-line arguments 當要更新設定檔就必須重啟 app 才可套用，但使用 ConfigMap 透過 volume 掛載則不需要重啟，k8s 支援發送 signal 給 container 通知已有檔案更新。


#### edit ConfigMap
更新後僅 container 內檔案目錄更新，若已啟用 server 如 NGINX，需要手動 reload。
```bash
kubectl edit configmap <CONFIGMAP_NAME>
```

#### after edit
configMap volume 是透過 symbolic links 實現，若有檔案更新，k8s 透過建立新目錄、重新建立檔案，將原目錄重新連結到新目錄。

:::info
如果只有 mount 單一檔案而非整個目錄，則該檔案不會連動更新。
:::

#### 更動 ConfigMap 的後果
- 更動 ConfigMap 後，新建立的 pod 會使用新版本的，但舊的 pod 會繼續使用舊的直到重啟，除非 app 支援 hot reload 能及時更新。
- 更動 ConfigMap 不一定能及時更新到 volume 檔案，有可能會耗時一分鐘。

---
## Secret
Secret 用於儲存機敏性資料，如憑證、私鑰。設定方式如同 ConfigMap。使用 Secret entry 透過 env 傳入 container、將 Secrets 倒入 volume 掛載到 container。

- Secret 只會在需要的 pod 所在的 node 上儲存。
- Secret 只會儲存在 memory 不會存到 physical storage。

| Resource  | 原則       | 儲存區                   | 加密 | 限制大小 | 顯示            | 用途                      |
| --------- | ---------- | ------------------------ | ---- | -------- | --------------- | ------------------------- |
| ConfigMap | app 設定檔 | physical storage, memory | x    | x        | plain-text      | server name, port         |
| Secret    | 機敏性資料 | memory                   | o    | 1MB      | Base64 encoding | certificate, access token |

:::warning
預設 Secret 儲存於 master node `etcd/`，未加密且能訪問 kube api 的人都可存取。
可使用以下方法增強 Secret 安全性：
- 為 Secret 啟用靜態加密。
- 啟用或配置 RBAC 規則，用以限制讀取和寫入 Secret。請注意，任何有權創建 Pod 的人都可以間接獲取 Secret。
- 在適當的情況下，還可以使用 RBAC 等機制來限制允許哪些主要人員能建立 Secret 或替換現有的 Secret。
:::


### Config & Commands

- `kubectl get secret`
- `kubectl describe secret`
- `kubectl edit secret <SECRET_NAME>`


### default token Secret
每個自己建立的 container 都有一個 default token Secret

```bash
$ kubectl describe pod <POD_NAME>
Name:         <POD_NAME>_...
...
Containers:
    <CONTAINER_NAME>:
        ...
        Mounts:
            /var/run/secrets/kubernetes.io/serviceaccount from <deafult_token> (ro)
```

![](https://i.imgur.com/xnuyFfZ.png =600x)


### create Secret
使用 openssl 簽一張憑證存入 Secret

- **簽憑證**
    ```bash
    $ openssl genrsa -out https.key 2048
    $ openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj
         /CN=www.kubia-example.com
    ```

- **建立 Secret**
    ```bash
    $ kubectl create secret generic <SECRET_NAME> --from-file=https.key --from-file=https.cert
    ```
    - subcommand: `generic`, `tls`, `docker-registry`
        - `docker-registry` Create a secret for use with a Docker registry
        - `generic`         Create a secret from a local file, directory, or literal value
        - `tls`             Create a TLS secret


- **secret.yaml**
    ```yaml=
    kind: Secret
    apiVersion: v1
    stringData:
        foo: plain text
    data:
        https.cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCekNDQ...
        https.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcE...
    ```
    - `stringData`: Secret 中儲存 non-binary 資料，**write-only** 使用 `kubectl get -o yaml` 不會顯示 `stringData` 欄位，會統一使用 Base64 編碼列在 `data` 下。



### secret volume
當 Secret 匯入至 secret volume 提供 container 掛載時，Secret 中儲存的值會 decode 匯出成檔案。

**以 Nginx 為例，使用 Secret 掛載網站憑證**
![](https://i.imgur.com/rz2LyHQ.png)
> `curl -k -v <https://site...>`，`-k` 不檢查憑證安全性，`-v` 查看網站憑證資訊。

**pod-secret.yaml**: pod 使用 secret volume
```yaml=
apiVersion: v1
kind: Pod
...
spec:
  containers:
  - name: ...
    volumeMounts:
    - name: <SECRET_VOLUME>
      mountPath: ...
      readOnly: true
  ...
volumes:
- name: <SECRET_VOLUME>
  secret:
    secretName: <MY_SECRET>
```
:::info
configMap volume, secret volume 皆可使用 `volumes.defaultMode` 設定掛載的檔案、目錄權限
:::

#### secret volume 儲存位置
secret volume 將 Secret 內的資料轉換成檔案儲存於 in-memory filesystem (tmpfs)，不會儲存到 physical storage。

**進入 container 查看 secret**
```bash
$ kubectl exec <POD> -c <CONTAINER> -- mount | grep certs
tmpfs on /etc/nginx/certs type tmpfs (ro,relatime)
```

### Pod 直接使用 Secret
`containers.env.valueFrom.secretKeyRef` 直接取用 Secret

```yaml=
apiVersion: v1
kind: pod
...
spec:
  containers:
  - name: ...
    ...
    env:
    - name: <ENV_VAR>
      valueFrom:
        secretKeyRef:
          name: <SECRET_NAME>
          key: <KEY>
```

:::warning
部分 app 出錯時會 dump 所有 env，這時的 Secret value 是無編碼的，所以要盡量避免讓 Secret 儲存在 env。
:::


### Secret docker-registry
用於登入授權 Docker Hub 帳號，可用於 pull private image

```bash
$ kubectl create secret docker-registry mydockerhubsecret \
  --docker-username=myusername --docker-password=mypassword \
  --docker-email=my.email@provider.com
```

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:
  - name: mydockerhubsecret
  containers:
  - name: ...
    image: username/private:tag
```


---
## Question
- 能同時定義 `containers.env` 和 `containers.envFrom` 嗎？ 有衝突時會優先參照哪個還是會定義順序？
    - `env` > `envFrom`
- Secret 為什麼要用 Base64？
    - 將檔案轉成 Base64 儲存進 JSON/YAML



## Ref
- [ConfigMaps | Kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets | Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Configure a Pod to Use a ConfigMap | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Kubernetes — ConfigMap. 簡單來說, 在我們的應用中, 當我們需要儲存一些較沒有敏感性的設定檔, 像是… | by Ray Lee | 李宗叡 | Learn or Die | Medium](https://medium.com/learn-or-die/kubernetes-configmap-aaad6ac09e18)
- [Day 16 - 系統設定就交給它吧：ConfigMap - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10193935)
- [kubernetes - When should I use envFrom for configmaps? - Stack Overflow](https://stackoverflow.com/questions/66352023/when-should-i-use-envfrom-for-configmaps)



---
 Changing the main process of a container
 Passing command-line options to the app
 Setting environment variables exposed to the app
 Configuring apps through ConfigMaps
 Passing sensitive information through Secrets