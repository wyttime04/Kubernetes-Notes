# CH6 Volumes: attaching disk storage to containers
[toc]

## Volumes
pod 裡的 container 可能隨時重啟或重建，為了讓檔案目錄保留能使用 volume 將 container 內的檔案目錄掛到 pod 層級，pod 內的各 container 也能共用 volume。

![](https://i.imgur.com/rmJaXLK.png)

---
## docker volume
- volume
- bind mount
- tmpfs mount

![](https://i.imgur.com/zvGWcjs.png)

---

## k8s volume
volume 類型分為 **Ephemeral Volume**, **Projected Volume**, **Projected Volume**

- `emptyDir`: 建立空目錄儲存暫時性資料
- `hostPath`: mount node 的目錄到 pod 內
- `local`: mount node 的 local storage device 到 pod 內
- `gitRepo` (deprecated): pod 初始化時 clone git repository
- `nfs`: mount 外部 NFS (Network File System) share
- `awsElasticBlockStore`, `azureDisk`, `azureFile`, `gcePersistentDisk`: cloud storage
- `cinder`, `cephfs`, `iscsi`, `flocker` (deprecated), `glusterfs`, `quobyte` (deprecated), `rbd`, `flexVolume`, `vsphereVolume`: network storage
- `configMap`, `secret`, `downwardAPI`: 特殊 volume 用於匯出 k8s 內 resource, cluster 資訊
- `persistentVolumeClaim`: 預先建立或動態配置的儲存區，e.g. GCE PersistentDisk, iSCSI volume

:::warning
在 Linux 系統下使用 volume 掛載目錄進 container，若原 container 該目錄下有檔案會被清除，僅留存 volume 的內容。
:::

---
## Ephemeral Volume
生命週期短暫相依於 pod，可用於儲存配置檔或 app 密鑰，或讓 container 儲存龐大的暫時性資料使用。

**類型**
- `emptyDir`: 建立空目錄儲存暫時性資料
- `gitRepo` (deprecated): pod 初始化時 clone git repository
- `configMap`, `secret`, `downwardAPI`: 特殊 volume 用於匯出 k8s 內 resource, cluster 資訊

:::info
`emptyDir`, `configMap`, `downwardAPI`, `secret` 屬於  local ephemeral storage，由各 node 的 kubelet 管理
:::


#### Config & Commands
- **fortune-pod.yaml**: pod with volume

    ```yaml=
    apiVersion: v1
    kind: Pod
    ...
    spec:
        containers:
            - image: luksa/fortune
              name: html-generator
              volumeMounts:
              - name: <volume_name>
                mountPath: /var/htdocs
            - image: nginx:alpine
              name: web-server
              volumeMounts:
              - name: <volume_name>
                mountPath: /usr/share/nginx/html
                readOnly: true
              ...
    volumes:
    - name: <volume_name>
      emptyDir: {}
    ```

### emptyDir
創建空目錄儲存暫時性資料，`.volumes[*].emptyDir.medium` 可設定儲存區為 memory

```yaml=
...
volumes:
- name: <volume_name>
  emptyDir: {}
```

```yaml=
...
volumes:
  - name: <volume_name>
    emptyDir:
      medium: Memory    # tmpfs filesystem
```

### gitRepo (deprecated)
在 container 建立前預先準備 git repositroy 的 volume
![](https://i.imgur.com/N8KwiCS.png)

:::info
`gitRepo` volume 無法自動同步，只能等待 pod 重建時才會再次建立新的 volume。
:::

:::warning
官網文件標示已棄用，若要預先建立 `gitRepo` volume 建議使用 `InitContainer` 掛載 `emptyDir` 再執行 `git clone` 將 repo 拉進 volume。
> [gitRepo | Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/#gitrepo)
:::

> `InitContainer` example
> ```yaml=
> ...
> spec:
>   contianers:
>   - image: ...
>     ...
>     volumeMounts:
>       - mountPath: /server/data
>         name: git-data
>   initContainers:
>   - image: alpine/git
>     args:
>       - clone
>       - --
>       - https://gitlab.url/.../.git
>       - /data
>     volumeMounts:
>       - mountPath: /data
>         name: git-data
> volume:
> - name: git-data
>   emptyDir: {}
> ```

> **pod life cycle**
> ![](https://i.imgur.com/fk3a9qV.png)
> ```yaml=
> ...
> kind: pod
> ...
> status:
>   conditions:
>     - type: Ready  # PodScheduled, ContainersReady, Initialized, Ready
>       status: "False"  # "True", "False", "Unknown"
>       ...
> ```
> [Pod conditions | Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions)



> `sidecar` container:
> A sidecar container is a container that aug- ments the operation of the main container of the pod.
> more resoruce cost, more network hop
> ![](https://i.imgur.com/EKDO3HF.png)

> use `git sync`


---
## Persistent Volume
PV 獨立於 pod 外，由 admin 配置一塊儲存區 (**PersistentVolume**, PV)， user 能夠請求使用 (**PersistentVolumeClaim**, PVC)，pod 再設定掛載該 PVC。

**PV 類型**
- `hostPath`: mount node 的目錄到 pod 內
- `local`: mount node 的 local storage device 到 pod 內
- `nfs`: mount 外部 NFS (Network File System) share
- `awsElasticBlockStore`, `azureDisk`, `azureFile`, `gcePersistentDisk`: cloud storage
- `cinder`, `cephfs`, `iscsi`, `flocker` (deprecated), `glusterfs`, `quobyte` (deprecated), `rbd`, `flexVolume`, `vsphereVolume`: network storage

**admin PV & user PVC**
![admin PV & user PVC](https://i.imgur.com/fV12sd8.png)

**PV & PVC**
PV 不隸屬於任何 namespace
![PV & PVC](https://i.imgur.com/phGMYgU.png)

**underlying persistent volume V.S PV + PVC**
直接掛載 persistent volume 及使用 PV+PVC 差異，配置 app 的人員不需要配置實體儲存區、煩惱如何介接遠端儲存庫。
![underlying persistent volume V.S PV + PVC](https://i.imgur.com/rMO5k46.png)


### Config & Commands
- **underlying-persistent-volume.yaml**
    ```yaml=
    apiVersion: v1
    kind: Pod
    metadata:
      name: mongodb
    spec:
      volumes:
      - name: <volume_name>
        hostPath:    # use hostPath
          # directory location on host
          path: /data
          # this field is optional
          type: Directory
      containers:
      - image: mongo
        name: mongodb
        volumeMounts:
        - name: <volume_name>
          mountPath: /data/db
        ports:
        - containerPort: 27017
          protocol: TCP
    ```

- **pv.yaml**
    ```yaml=
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: <pv_name>
    spec:
      capacity:
        storage: 1Gi       # define pv size
      accessModes:
        - ReadWriteOnce
        - ReadOnlyMany
        persistentVolumeReclaimPolicy: Retain
        gcePersistentDisk:    # pv type
          pdName: mongodb
          fsType: ext4
    ```
     
    - **accessModes**: 存取權限
        - `RWO`, `ReadWriteOnce`: 限同一 node 以 read-write 方式 mount volume。
        - `ROX`, `ReadOnlyMany`: 多個不同 node 能以 read-only 方式 mount volume。
        - `RWX`, `ReadWriteMany`: 多個不同 node 能以 read-write 方式 mount volume。
        - `RWOP`, `ReadWriteOncePod`: 限定一個 pod 以 read-write 方式 mount volume。僅支援 `CSI` 和 Kubernetes v1.22^
        :::warning
        實際 mount volume 到 pod 時只能選其中一種存取權限
        :::

    - **persistentVolumeReclaimPolicy**: 當 claim 被移除時要回收釋出的空間
        - `Retain`: 手動處理
        - `Recycle`: 自動回收 `rm -rf /thevolume/*`，支援 `NFS`、`hostPath`
        - `Delete`: 關聯的存儲區，支援 AWS EBS、GCE PD、Azure Disk 、 OpenStack Cinder 

        ![](https://i.imgur.com/jw7iYH8.png)


- `kubectl get pv`
    ```
    $ kubectl get pv
    NAME         CAPACITY   RECLAIMPOLICY   ACCESSMODES   STATUS      CLAIM
    mongodb-pv   1Gi        Retain          RWO,ROX       Available
    ```

- **pvc.yaml**
    ```yaml=
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: <pvc_name>
    spec:
      storageClassName: ""    # 指定 storageClass
      resources:
        requests:
          storage: 1Gi    # request storage size
      accessModes:        # 對應 pv accessModes
      - ReadWriteOnce
    ```

- `kubectl get pvc`
    ```
    $ kubectl get pvc
    NAME          STATUS    VOLUME       CAPACITY   ACCESSMODES   AGE
    mongodb-pvc   Bound     mongodb-pv   1Gi        RWO,ROX       3s
    ```
    
- **pod-pvc.yaml**
    ```yaml=
    ...
    spec:
      volumes:
      - name: <volume_name>
        persistentVolumeClaim:
          claimName: <pvc_name>
    ```
   


### hostPath
mount node 的目錄到 pod 內
![](https://i.imgur.com/ww4p5eX.png)

**hostPath type**: `.spec.volumes[*].hostPath.type`
- `default`: 掛載 hostPath volume 之前不會執行任何檢查。
- `DirectoryOrCreate`: 如果給定路徑不存在任何內容，則會創建一個空目錄，權限設置為 0755，與 Kubelet 具有相同的 group 和所有權。
- `Directory`: 目錄必須存在於給定路徑
- `FileOrCreate`:	如果給定路徑不存在任何內容，則會創建一個空檔案，權限設置為 0644，與 Kubelet 具有相同的 group 和所有權。
- `File`: 文件必須存在於給定路徑
- `Socket`: UNIX socket 必須存在於給定路徑
- `CharDevice`: character device 必須存在於給定路徑
- `BlockDevice`: block device 必須存在於給定路徑

**underlying-persistent-volume.yaml**
```yaml=
...
spec:
  volumes:
  - name: mydir
    hostPath:
      # Ensure the file directory is created.
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
  containers:
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
```

### local
mount node 的 local storage device (e.g. disk, partition) 到 pod 內，`local` 只能建立靜態的 PV

:::info
如果不使用 external static provisioner 來管理卷生命週期，則 local PV 需要手動清理和刪除。
:::

**underlying-persistent-volume.yaml**
```yaml=
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage    # 需定義 storageClass
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

### nfs
mount 外部 NFS (Network File System) share

**underlying-persistent-volume.yaml**
```yaml=
volumes:
- name: mongodb-data
  nfs:    # This volume is backed by an NFS share.
    server: 1.2.3.4    # The IP of the NFS server
    path: /some/path    # The path exported by the server
```

**nfs-pv.yaml** + **nfs-pvc**
```yaml=
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 1Mi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.default.svc.cluster.local
    path: "/"
  mountOptions:
    - nfsvers=4.2
```

```yaml=
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1Mi
  volumeName: nfs
```

#### Out-of-tree volume plugins
讓 storage 廠商自行整合 k8s 免除需要整合進 k8s 專案的審核
- csi
- flexVolume (deprecated)


---
## Projected Volume
將已存在的 volume 映射到對應的目錄，所有 resource 必須在同一個 namespace 才可使用。

**類型**
- `configMap`, `secret`, `downwardAPI`
- serviceAccountToken

:::info
container 若使用 Projected Volume 做為 subPath 不會更新 volume resource
:::


---
## StorageClass
StorageClass 為 admin 提供分類儲存區的方法，可用於動態配置 storage。StorageClass 不隸屬於任何 namespace。

admin 透過 PV provisioner 配置 PV 後，使用 storageClass 標註 provisioner。
user 建立綁定 storageClass 的 PVC，k8s 查找 provisioner 從對應的 PV 取得實體儲存區，user 建立 pod 時綁定 volume 至剛剛從 PVC 得到的儲存區。

![dynamic provisioning](https://i.imgur.com/TFjzpeT.png)

### Config & Commands
- **storageclass.yaml**
    ```yaml=
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: <storageclass_name>
    # The volume plugin to use for provisioning the PersistentVolume
    provisioner: kubernetes.io/aws-ebs
    # The parameters passed to the provisioner
    parameters:
      type: gp3
    reclaimPolicy: Retain
    allowVolumeExpansion: true
    mountOptions:
      - debug
    volumeBindingMode: Immediate
    ```

- **pvc-storageclass**
    ```yaml=
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: <pvc_name>
    spec:
      storageClassName: <storageclass_name>    # 指定 storageClass
      resources:
        requests:
          storage: 1Gi    # request storage size
      accessModes:        # 對應 pv accessModes
      - ReadWriteOnce
    ```
    - `storageClassName`: 如果未定義 `storageClassName`，則會不挑選已存在的 PV 直接使用 `default` storageClass 建立新的 PV。若給予空字串 `storageClassName: ""`，則會使用 pre-provisioned PV
    
    :::warning
    如果 PVC 指定的 storageClass 不存在會無法分配 storage 並出現 `ProvisioningFailed` 錯誤
    :::

- `kubectl get sc`, `kubectl get sc standard -o yaml`
    ```
    $ kubectl get sc
    NAME                    TYPE
    <storageclass_name>     k8s.io/minikube-hostpath
    standard (default)      k8s.io/minikube-hostpath
    ```
    
    ![](https://i.imgur.com/ogbQps4.png)



---
## Question
- 若在建立 docker container 就有使用 bind mount 到 pod 內的目錄 `/var/myhostdocs`，又在 k8s 再次使用 `hostPath` 建立 volume path 同樣到目錄 `/var/myhostdocs`，pod 的目錄會依照哪個為主？
- 以 `hostPath` 建立 volume，在 volume 建立檔案 `test.txt` 存取權限定義同 group `test-group` 能讀寫，那在 container 內建立同樣的 group `test-group` 是否能存取？
    - gid 一樣就可以，但要注意 user 權限，在 image 設定好 default user

## Ref
- [Storage | Docker](https://docs.docker.com/storage/)
- [examples/staging/volumes at master．kubernetes/examples | github](https://github.com/kubernetes/examples/tree/master/staging/volumes)
- [Volumes | Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes)


 Creating multi-container pods
 Creating a volume to share disk storage between
containers
 Using a Git repository inside a pod
 Attaching persistent storage such as a GCE Persistent Disk to pods
 Using pre-provisioned persistent storage
 Dynamic provisioning of persistent storage



#### Config & Commands
- `kubectl describe pod`, `kubectl get pod`: 查看 pod volumes
    ![](https://i.imgur.com/RgcT1x6.png)

    ```
    $ kubectl describe pod calico-node-l69sd --namespace kube-system
    Name:                 calico-node-l69sd
    Namespace:            kube-system
    Priority:             2000001000
    Priority Class Name:  system-node-critical
    Node:                 <hostname>/<host_ip>
    Start Time:           Sun, 13 Mar 2022 11:14:59 +0000
    Labels:               controller-revision-hash=8486f8985b
                          k8s-app=calico-node
                          pod-template-generation=1
    Annotations:          <none>
    Status:               Running
    IP:                   <host_ip>
    IPs:
      IP:           <host_ip>
    ...
    ...
    Volumes:
      lib-modules:
        Type:          HostPath (bare host directory volume)
        Path:          /lib/modules
        HostPathType:  
      var-run-calico:
        Type:          HostPath (bare host directory volume)
        Path:          /var/run/calico
        HostPathType:  
      var-lib-calico:
        Type:          HostPath (bare host directory volume)
        Path:          /var/lib/calico
        HostPathType:  
      xtables-lock:
        Type:          HostPath (bare host directory volume)
        Path:          /run/xtables.lock
        HostPathType:  FileOrCreate
      sysfs:
        Type:          HostPath (bare host directory volume)
        Path:          /sys/fs/
        HostPathType:  DirectoryOrCreate
      cni-bin-dir:
        Type:          HostPath (bare host directory volume)
        Path:          /opt/cni/bin
        HostPathType:  
      cni-net-dir:
        Type:          HostPath (bare host directory volume)
        Path:          /etc/cni/net.d
        HostPathType:  
      cni-log-dir:
        Type:          HostPath (bare host directory volume)
        Path:          /var/log/calico/cni
        HostPathType:  
      host-local-net-dir:
        Type:          HostPath (bare host directory volume)
        Path:          /var/lib/cni/networks
        HostPathType:  
      policysync:
        Type:          HostPath (bare host directory volume)
        Path:          /var/run/nodeagent
        HostPathType:  DirectoryOrCreate
      flexvol-driver-host:
        Type:          HostPath (bare host directory volume)
        Path:          /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
        HostPathType:  DirectoryOrCreate
      kube-api-access-vs4rn:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        ConfigMapOptional:       <nil>
        DownwardAPI:             true
    ```
    ```
    $ kubectl get pods calico-node-l69sd --namespace kube-system -o json
    {
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {
            "creationTimestamp": "2022-03-13T11:14:59Z",
            "generateName": "calico-node-",
            "labels": {
                "controller-revision-hash": "8486f8985b",
                "k8s-app": "calico-node",
                "pod-template-generation": "1"
            },
            "name": "calico-node-l69sd",
            "namespace": "kube-system",
            ...
        },
        "spec": {
            "volumes": [
                ...,
                {
                    "hostPath": {
                        "path": "/var/lib/cni/networks",
                        "type": ""
                    },
                    "name": "host-local-net-dir"
                },
                {
                    "hostPath": {
                        "path": "/var/run/nodeagent",
                        "type": "DirectoryOrCreate"
                    },
                    "name": "policysync"
                },
                {
                    "hostPath": {
                        "path": "/usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds",
                        "type": "DirectoryOrCreate"
                    },
                    "name": "flexvol-driver-host"
                },
                {
                    "name": "kube-api-access-vs4rn",
                    "projected": {
                        "defaultMode": 420,
                        "sources": [
                            {
                                "serviceAccountToken": {
                                    "expirationSeconds": 3607,
                                    "path": "token"
                                }
                            },
                            {
                                "configMap": {
                                    "items": [
                                        {
                                            "key": "ca.crt",
                                            "path": "ca.crt"
                                        }
                                    ],
                                    "name": "kube-root-ca.crt"
                                }
                            },
                            {
                                "downwardAPI": {
                                    "items": [
                                        {
                                            "fieldRef": {
                                                "apiVersion": "v1",
                                                "fieldPath": "metadata.namespace"
                                            },
                                            "path": "namespace"
                                        }
                                    ]
                                }
                            }
                        ]
                    }
                }
            ]
    }
    ```