## Operations

在 gitops 中，會希望比對遠方的物件(Helm Chart、yaml...)，那這些物件會儲存在版控系統中。因此 Argo 有以下概念

- Repositories
  - 目標同步的位置
- Project
  - 邏輯上對應用程式分群
  - RBAC 使用者可以做哪些事情



### [Sync Options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- Prune
- Dry-run
- Apply Only
  - Skip pre/post sync hooks
- Force

### [Sync Hook](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/)

- PreSync
  - DB schema migration
- Sync
- Skip
- PostSync
- SyncFail	

|Hook| Description |Use case|
|---|---|---|
|PreSync| Executes prior to the application of the manifests| Database migrations|
|Sync| Executes at the same time as manifests| Complex rolling update strategies like canary releases or dark launches|
|PostSync |Executes after all Sync hooks have completed and were successful (healthy)|Run tests to validate deployment was correctly done|
|SyncFail |Executes when the sync operation fails| Rollback operations in case of failure| 
|Skip| Skip the application of the manifest| When manual steps are required to deploy the application (i.e., releasing public traffic to new version)|

### [Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
有以下兩個

- Application
- Resource
  - Kubernetes 資源(Deployment/Service/PVC)

### State
- Target state
  - 所需的期望狀態，由 Git 儲存庫中的檔案表示
- Live state
  - 應用程式當前狀態，像是部署了哪些 Pods 等當前狀態
- Sync status
  - 表示當前狀態是否與目標狀態一致，部署的物件是否與 Git 管控的內容所描述的一樣

##  [Declarative ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
對於 ArgoCD 中 application、projects 或一些同步設定等，都可以透過宣告式方式來進行。這對於管理方面來說都會是一個很好的方式。

## [User Management](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/)
- [Local users](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#local-usersaccounts-v15)
  - 小團隊
  - 對 Argocd 進行存取像是 CICD 流程中
  - 每個使用者都可以有 `apiKey` 或是 `login` 的能力
  - 沒有 Group 等概念
- [SSO](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#sso)

## 權限控管
### [Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
使用 `AppProject` 資源。對應用程式上的邏輯群組概念。
- 可以被佈署的範圍
  - 一個 Repo
- 佈署的目標
  - 集群或是 `namespace`
- 可被佈署的物件
  - CRDS、DaemonSet 等
- 整合 RBAC

預設 Project 如下
```yaml
spec:
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
```
### [RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
使用 `ConfigMap` 配置。RBAC 功能可以限制對 Argo CD 資源的訪問。需要 SSO 配置或一個或多個本地用戶設置。

**其基本內置角色有以下**

- role:readonly - read-only access to all resources
- role:admin - unrestricted access to all resources

**權限結構**
- `p, <role/user/group>, <resource>, <action>, <object>`
- `p, <role/user/group>, <resource>, <action>, <appproject>/<object>`

`P` 表示 policy；`Resources` 包含以下
- clusters 
- projects 
- applications
- applicationsets
- repositories
- certificates
- accounts
- gpgkeys
- logs
- exec

`Actions` 包含以下
- get 
- create
- update
- delete
- sync
- override
- `action/<group/kind/action-name>`
