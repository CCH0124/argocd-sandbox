## RBAC and User
1. Create Project
2. Create User
3. Create RBAC Policy
4. Verify the Users
5. Using can-i

RBAC 功能可以限制對 ArgoCD 資源的存取。ArgoCD 本身設計沒有使用者管理系統，只有一個內建的 `admin` 超級使用者。另外 RBAC 需要搭配 SSO 或是地端使用者設定，這樣才能使得使用者綁定到 RBAC 角色。可以定義 RBAC 配置的兩個主要組件：
- 在 ConfigMap 配置 RBAC
- AppProject CRD

當使用這被 Argo CD 通過身分驗證時將被賦予 `policy.default` 中指定的角色。對於匿名存取即無須被驗證，對於是否可匿名存取可針對參數 `users.anonymous.enabled` 進行設定。當然 `policy.default` 絕對不是最佳實踐，而是要而外設計角色並附加於 `policy.default` 上。

ArgoCD 有兩個預先定義的角色

- `role:readonly` 對所有資源的存取權限是唯讀
- `role:admin`- 不受限制存取所有資源

而 RBAC 配置允許定義角色(role)、群組(group)和 Policy。
1. `group` 允許將經過身份驗證的使用者或群組指派給內部角色

結構:

`g, <user/group>, <role>`

`<user/group>` 被指派角色的實體。可以是本機的使用這或是 SSO 進行身分驗證的使用者。

`<role>` 實體被指派的內部角色。

2. Policy 允許向實體分配權限。

結構:

`p, <role/user/group>, <resource>, <action>, <object>, <effect>`

`<role/user/group>` 指派權限至某個實體
`<resource>` 要操作的資源類型
`<action>` 對資源執行的操作
`<object>` 表示資源的標識符，格式依資源類型而定
`<effect>` 決定是否允許或拒絕對目標物件的存取，值為 `allow` 或 `deny`


下表總結了所有可能的資源以及對每個資源有效的操作。

|Resource\Action	|get	|create	|update	|delete	|sync	|action	|override	|invoke|
|---|---|---|---|---|---|---|---|
|applications|	✅	|✅	|✅	|✅	|✅	|✅	|✅	|❌|
|applicationsets|	✅	|✅	|✅	|✅	|❌	|❌	|❌	|❌
|clusters	|✅	|✅	|✅	|✅|	❌|	❌|	❌|	❌|
|projects	|✅|	✅|	✅|	✅|	❌|	❌|	❌|	❌|
|repositories|	✅|	✅|	✅|	✅|	❌|	❌|	❌|	❌|
|accounts	|✅|	❌|	✅|	❌|	❌|	❌|	❌|	❌|
|certificates	|✅|	✅|	❌|	✅|	❌|	❌|	❌|	❌|
|gpgkeys	|✅|	✅|	❌|	✅|	❌|	❌|	❌|	❌|
|logs	|✅	|❌|	❌|	❌|	❌|	❌|	❌|	❌|
|exec	|❌|	✅|	❌|	❌|	❌|	❌|	❌|	❌|
|extensions|	❌|	❌|	❌|	❌|	❌|	❌|	❌|	✅|

先以官方的範例來看

1. 要授予 example-user 存取權但僅刪除 prod-app Application 資源中的 Pod 資源，Policy 設計可以是

```bash
p, example-user, applications, delete/*/Pod/*, default/prod-app, allow
```

2. 要授予使用者更新應用程式所有資源的存取權限，而不是應用程式本身。因此無法對 Application 資源進行刪除等。

```bash
p, example-user, applications, update/*, default/prod-app, allow
```

3. 如果要明確拒絕刪除應用程序，但允許用戶刪除 Pod 資源

```bash
p, example-user, applications, delete, default/prod-app, deny
p, example-user, applications, delete/*/Pod/*, default/prod-app, allow
```

4. 將 Policy 指派給使用者

```bash
p, my-local-user, applications, sync, my-project/*, allow
```

5. 指派角色給使用者

```bash
g, my-local-user, role:admin
```

`<action>` 欄位可以是 `<action>/<group>/<kind>/<ns>/<name>` 的設計。


## Create Project

前面的實驗有建置過下面的資源，接下來會基於此資源進行相關的 RBAC 設定。

```yaml
# /developer/team-a/project/project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a-project
  namespace: argo
  # Finalizer that ensures that project is not deleted until it is not referenced by any application
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Project for team-a
  sourceRepos:
  - https://github.com/CCH0124/K8s-with-quarkus.git
  destinations:
  - namespace: team-a
    name: stage-cluster
  - namespace: team-a
    name: k3d-dev-cluster
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: 'apps'
    kind: ReplicaSet
  - group: '*'
    kind: ServiceAccount
  - group: '*'
    kind: Service
  - group: '*'
    kind: Pod
  - group: 'batch'
    kind: Job
  permitOnlyProjectScopedClusters: false
```

## Local Users/Accounts and RBAC

以下也許是地端使用者整合的場景

1. 自動化管理 Argo CD 的令牌(token)，可以建立具有有限權限的 API 帳號並生綁定令牌。該令牌可用於自動創建 Application、AppProject 等資源
2. 小型團隊，對於小團隊當使用 SSO 整合時可能是殺雞用牛刀的效果，因此可以使用 Argo CD 本地使用者功能建議管理。但，本地使用者缺乏較高抽象層級功能，如群組、登錄歷史紀錄等。如果需要這些功能，則建議使用 SSO

對於建置的帳號，該帳號可能有兩種類型的權限
- `apiKey` 允許產生用於存取 API 的令牌
- `login` 允許從 UI 登入

建立使用者帳號格式為 `accounts.<username>: <capabilities>`。

現在假設有 `dev`、`pm` 和 `op` 三種角色，每個角色都有自己對應的權限。

1. dev

視為開發者，對於 `team-a-project` 下的 `Application` 資源可以進行任何操作，對於 `AppProject`、`repositories` 和 `clusters` 只有讀的權限，而 `applicationsets` 將被拒絕任何操作。

2. pm

視為專案管理者，對任何訊應當都為讀取權限，像是 `applications`、`projects` 和 `repositories`。而 `clusters` 和  `applicationsets` 將被拒絕任何操作。

3. op

視為維運者，其權限相比 dev 或 pm 來的大因此權限基本上都放行像是 `exec` 和 `accounts` 等資源。

Argo CD 使用 Helm Chart 佈署，因此可以如下進行 User 和 RBAC 設定部分。

```yaml
        configs:
          cm:
            ...
            exec.enabled: true
            admin.enabled: true
            exec.shells: "bash,sh"
            accounts.devuser: login
            accounts.pmuser: login
            accounts.opsuser: login
            accounts.devuser1: login
            accounts.devuser1.enabled: "false"
          rbac:
            create: true
            policy.default: role:readonly
            policy.csv: |
                p, role:dev, applications, *,  team-a-project/*, allow
                p, role:dev, projects, get,  team-a-project, allow
                p, role:dev, repositories, get,  *, allow
                p, role:dev, clusters, get,  *, allow
                g, devuser, role:dev
                
                p, role:pm, applications, get,  */*, allow
                p, role:pm, projects, get,  *, allow
                p, role:pm, repositories, get,  *, allow
                p, role:pm, clusters, get,  *, allow
                g, pmuser, role:pm

                p, role:op, applications, *,  */*, allow
                p, role:op, projects, *,  *, allow
                p, role:op, repositories, *,  *, allow
                p, role:op, clusters, *,  *, allow
                p, role:op, accounts, *,  *, allow
                p, role:op, certificates, *,  *, allow
                p, role:op, gpgkeys, *,  *, allow
                p, role:op, exec, create, */*, allow
                g, opsuser, role:op
```

比較特別的配置是 `exec.enabled` 預設是關閉，其用於可以針對 Pod 資源進行遠端連線操作。這邊將設定為 `true` 且只有 `opuser` 能夠操做。

基本上在 `configs.cm` 下透過 `accounts.<username>: <capabilities>` 格式設定使用者帳號後 Argo CD 會建置帳號，如下透過 `argocd account list` 可以列出當前有的使用者帳號。

```bash
$ argocd account list
NAME      ENABLED  CAPABILITIES
admin     true     login
devuser   true     login
devuser1  false    login
opsuser   true     login
pmuser    true     login
```

這些建置的帳號預設沒有密碼，也無法正常登入因此藉由以下指令格式給予密碼。

```bash
argocd account update-password --account <new-username> --new-password <new-password>
```

如果是令牌，可以使用以下格式

```bash
argocd account generate-token --account <username>
```

### 驗證 devuser1

devuser1 是無法被登入可以想像是離開專案的開發者，因此其登入帳號被進行控管。
```yaml
            accounts.devuser1: login
            accounts.devuser1.enabled: "false"
```

從平台進行登入時理當會看到此訊息 `Account devuser1 is disabled`。

### 驗證 devuser RBAC

使用 CLI 方式使用 devuser 登入

```bash
argocd login argo.cch.com:8443 --grpc-web-root-path argocd --username devuser
```

對於 devuser1 其權限如下

```yaml
p, role:dev, applications, *,  team-a-project/*, allow
p, role:dev, projects, get,  team-a-project, allow
p, role:dev, repositories, get,  *, allow
p, role:dev, clusters, get,  *, allow
g, devuser, role:dev    
```

對於 team-a-project `AppProject` 資源下的 Application 資源都可以進行操作但被限制於 `team-a-project` 下；但對於 projects、repositories、clusters 都只能獲取，不能夠新增或更新。這邊對於應用程式佈署我們限制於 `team-a-project` 同時只有 dev/stage 叢集能佈署，只要是其它叢集都會被拒絕。而 `exec` 此功能也是被拒絕，因此在平台上是無法進行遠端操作。

使用 devuser 嘗試對 opuser 管理的 sre-project 資源下進行 `sync` 操作，會出現 `PermissionDenied` 訊息。

```bash
$ argocd app sync argo/sre-app
FATA[0000] rpc error: code = PermissionDenied desc = permission denied: applications, sync, sre-project/sre-app, sub: devuser, iat: 2024-08-11T04:49:48Z
```

同理對 `repositories` 進行新增時也會有權限問題。

```bash
$ argocd repo add --name app https://github.com/CCH0124/K8s-with-quarkus.git --type git --project default
FATA[0000] rpc error: code = PermissionDenied desc = permission denied: repositories, create, default/https://github.com/CCH0124/K8s-with-quarkus.git, sub: devuser, iat: 2024-08-11T04:49:48Z
```

但對 team-a-project 下的 simple-app `Application` 資源執行 `sync` 是放行，在權限設計上是允許。

```bash
$ argocd app sync argo/simple-app
```

當然最快驗證可以使用 `can-i` 方式類似於 Kubernetes `kubectl auth can-i`。

```bash
$ argocd account can-i sync applications 'team-a-project/*'
yes
$ argocd account can-i sync applications 'sre-project/*'
no
$ argocd account can-i delete applications 'sre-project/*'
no
$ argocd account can-i delete clusters '*'
no
$ argocd account can-i get clusters '*'
yes
$ argocd account can-i get exec 'team-a-project/*'
no
$ argocd account can-i create exec 'sre-project/*'
no
$ argocd account can-i create exec 'team-a-project/*'
no
```

pmuser 部分讀者可以嘗試驗證看看，對於 pmuser 可以讀取資源，但基本沒權限建立像是 `AppProject`、`repositories`、`clusters` 等資源。


### 驗證 opsuser RBAC

接著針對 `opsuser` 進行驗證。

登出 devuser 並使用 opsuser 登入。

```bash
argocd logout argo.cch.com:8443 --grpc-web-root-path argocd
argocd login argo.cch.com:8443 --grpc-web-root-path argocd --username opsuser
```

對於 opsuser 可以對 `account` 資源進行變更密碼等操作；`exec` 可以對資源進行遠端，剩下資源也可以進行刪除、建立或更新等操作。

```bash
$ argocd account can-i sync applications 'team-a-project/*'
yes
$ argocd account can-i sync applications 'sre-project/*'
yes
$ argocd account can-i delete applications 'team-a-project/*'
yes
$ argocd account can-i create clusters '*'
yes
$ argocd account update-password --account pmuser --new-password a1234567890
*** Enter password of currently logged in user (opsuser):
Password updated
$ argocd account can-i create exec 'team-a-project/*'
yes
$ argocd account can-i create exec 'sre-project/*'
yes
```

## 結論

Argo CD 可以讓登入功能串接第三方 SSO 平台或是對於簡單團隊可以使用地端簡易配置 RBAC 方式進行使用者管理。對於本篇分享是對於地端配置進行簡易的設計，並理解 Argo CD 平台 RBAC 的結構以及可以如何進行細粒度上的資源控管。對於第三方 SSO 平台整合會相較於地端管理方式前期配置較複雜，這部分可以參考官方[使用者管理](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/)。

Argo CD 本身也可以針對登入失敗的政策進行配置像是，這些設定都是正向且是有好的配置
- ARGOCD_SESSION_FAILURE_MAX_FAIL_COUNT 失敗幾次進行拒絕登入
- ARGOCD_SESSION_FAILURE_WINDOW_SECONDS 等待多久後可再次進行登入
- ARGOCD_MAX_CONCURRENT_LOGIN_REQUESTS_COUNT 限制並發登入請求的最大數量

最後可以從本篇文章知道如何使用 `can-i` 對 RBAC 進行驗證。