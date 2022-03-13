# CH3 - Pods: running containers in Kubernetes

## pod
- pod 為 k8s 最小單位，依照服務劃分不同 pod，pod 內可有多個 container，通常只會放單一一個 container，而 container 通常只會運行單一 process。

- container 有獨立的作業環境，在外層的 pod 則讓這些 container 有獨立的網路環境，pod 透過 ip 定址可以和其他 pod 互聯，也因為 flat network 不同 node 也可以此方式互聯，如同 LAN。

- 可以透過 label 定義 pod 性質，e.g. `app=my_website`, `web=frontend`, `web=backend`，之後可以 label 指令查詢。

 

## question

- docker 本身的 container 就能以 container 名稱當作 dns 連線，也能定義 container 專屬的 network，類型有 bridge, host, overlay, etc，不一定需要 pod 才能切割網路，pod 這層又多加了哪些功能？

- docker-compose 可以同時啟用多個關聯的 container，但 docker-compose 對於 container 啟用後較難管理，k8s 有提供哪些更好管理多個 container 的功能？

 

## ref

- [docker network drivers](https://docs.docker.com/network/#network-drivers)
- [docker compose](https://docs.docker.com/compose/)