## ApplicationSet

`ApplicationSet` 是一個 Argo CD 的 CRD，他會對應至一個 `Applicationset` 的控制器，其對 `Application` 資源增加了自動化功能，優化了 Argo CD 的多集群的複雜部署。`ApplicationSet` 控制器提供的一系列方法還可用於允許開發人員，在無法存取 Argo CD 命名空間和在沒有叢集管理員干預的情況下獨立建立 `Application`。

會使用 `ApplicationSet` 場景可能會是
- 部署到多個 Kubernetes 叢集
- 部署到不同的 namespace
- 部署到單一 Kubernetes 叢集中的不同 `namespace`
- 從不同的 Git 儲存庫、資料夾或是分支進行部署

Argo CD 中的生成器(generators)透過提供整合到 `template` 欄位中的動態參數來自動建立 Argo CD 應用程式。這些參數用於定義應用程式及其配置，這應更輕鬆在多個環境中部署和管理應用程式。`ApplicationSet` 有以下產生器

- List
  - 允許從叢集名稱或 URL 值(Kubernetes API Server) 的固定清單配置將 Argo CD `Application` 資源定位到要的叢集
  - 從定義的鍵/值元素對的固定清單配置將 Argo CD `Application` 資源定位到要的叢集
- Cluster
  - 根據 Argo CD 中定義且被 Argo CD 管理的叢集列表將 Argo CD `Application` 資源定位到要的叢集
- Git 
  - 根據 Git 儲存庫中的檔案或基於 Git 儲存庫的目錄結構建立 `Application` 資源
- Matrix
  - 用於組合兩個單獨產生器的生成參數
- Merge
  - 合併產生器可用於合併兩個或多個生成器產生的參數。第二個生成器可以覆蓋第一個生成器的值
- SCM Provider
  - 使用 SCM 提供者（例如 GitHub）的 API 自動發現組織內的儲存庫
- Pull Request
  - 利用 SCMaaS 提供者（例如 GitHub）的 API 自動發現儲存庫中開放的拉取請求 (Pull Request)

更多 `ApplicationSet` 支援的產生器可參閱 [Argo CD 產生器文件](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/)。

## matrix 產生器範例
以下是一個範例，目標是在 dev 和 stage 的 Kubernetes 集群上將兩個 team 進行 `namespace` 的資源限制。這邊用到 `matrix` 的概念來實作。上面描述 `matrix` 產生器需要搭配兩個產生器進行整合，這邊範例會是 `clusters` 和 `list` 進行整合。


在 Argo CD 中，集群相關 `Secret` 資源資訊會儲存至部署 Argo CD 服務的 `namespace` 中。對於 `ApplicationSet` 使用這些 Secret 來產生參數來識別和定位可用叢集。下面這些值是可被識別並用於 `clusters` 產生器中。
- name
- nameNormalized ('name' but normalized to contain only lowercase alphanumeric characters, '-' or '.')
- server
- metadata.labels.<key>
- metadata.annotations.<key>

補充目前 `clusters` 產生器無法對 `annotations` 資源盡操作。

透過以下指令方式可以知道說，Argo CD 管理的群集資訊，預設上在新增集群 Secret 資源不會有額外的註解或是標籤，因此實踐上能增加註解或是標籤的值會比較好。

```bash
$ argocd cluster get k3d-dev-cluster -o yaml
annotations:
  env: dev
...
labels:
  env: stage
name: stage-cluster
namespaces:
- team-a
- team-b
- team-c
- ingress
server: https://172.18.0.5:6443
serverVersion: "1.29"
```

也因為有定義了註解，因次可以更靈活的使用 `clusters` 產生器。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ns-appset-matrix
  namespace: argo
spec:
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchExpressions:
                  - key: env
                    operator: In
                    values:
                      - dev
                      - stage
              values: # 定義額外的變數
                revision: argo-appset
                clusterName: '{{name}}' # name 會來自 secrets 中定義 cluster 的欄位 
          - list: # 定義一系列的變數，這邊就是以 team-b、team-c 為一個範例
              elements:
                - template: team-b
                - template: team-c
  template:
    metadata:
      name: '{{values.clusterName}}-{{template}}'
    spec:
      project: default
      source:
        repoURL:  https://github.com/CCH0124/helm-charts
        targetRevision: '{{values.revision}}'
        path: 'kustomizes/namespace-config/overlays/{{values.clusterName}}/{{template}}'
      destination:
        server: '{{server}}'
        namespace: '{{template}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
        managedNamespaceMetadata:
          labels:
            env: '{{values.clusterName}}'
          annotations:
            team: '{{template}}'
            manage-by: argocd  
```

從下面的 `ApplicationSet/ns-appset-matrix` 結果來看，確實 Argo CD 在 dev 和 stage 環境的集群建置了給 team-b、team-c 的 `namespace` 並且能夠配置資源限制的配置。

```bash
$ argocd app get argo/sre-app -o tree
Name:               argo/sre-app
Project:            sre-project
Server:             https://kubernetes.default.svc
Namespace:          argo
URL:                https://argo.cch.com/applications/sre-app
Source:
- Repo:             https://github.com/CCH0124/argocd-gitops-manifest.git
  Target:           HEAD
  Path:             .
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (7e0c070)
Health Status:      Healthy

KIND/NAME                             STATUS  HEALTH   MESSAGE
Secret/team-a-repo                    Synced
Secret/third-party-repo               Synced
AppProject/sre-project                Synced
AppProject/team-a-project             Synced
AppProject/third-party-project        Synced
Application/sre-app                   Synced           application.argoproj.io/sre-app configured
Secret/sre-repo                       Synced
Application/argocd                    Synced
Application/simple-app                Synced
Application/sync-wave-app             Synced
ApplicationSet/ns-appset-matrix       Synced  Healthy  applicationset.argoproj.io/ns-appset-matrix created
├─Application/k3d-dev-cluster-team-b
├─Application/k3d-dev-cluster-team-c
├─Application/stage-cluster-team-b
└─Application/stage-cluster-team-c
```

基本上可以想成是兩層迴圈運行

```
for i in [dev, stage]
  for k in [team-b, team-c]
```

好處是未來只要新增一個團隊至 `list` 產生器下的 `elements` 欄位，並在 `kustomizes/namespace-config/overlays/` 下目錄也配置對應團隊的 `namespace` 配置，這樣就可以再多集群靈活部署。同樣當新增一個 prd 集群給 team-b、team-c 使用時，也會同時建立相對應的 `namespace` 資源。

## clusters 產生器範例

這邊目標是希望 dev/stage 和 prd 環境能夠安裝 nginx ingress controller，因次設計了使用 `clusters` 產生器的 `ApplicationSet` 應用。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nginx-ingress-appset
  namespace: argo
spec:
  generators:
  - clusters: 
      selector:
        matchExpressions:
          - key: env
            operator: In
            values:
              - dev
              - stage
              - prd
  template:
    metadata:
      name: '{{name}}-ingress-controller'
    spec:
      project: sre-project
      sources:
      - repoURL: 'https://kubernetes.github.io/ingress-nginx'
        chart: ingress-nginx
        targetRevision: 4.10.1
        helm:
          releaseName: ingress-nginx
          version: v3
          valueFiles:
          - $values/third-party/ingress/values.yaml
      - repoURL: 'https://github.com/CCH0124/argocd-sandbox.git'
        targetRevision: HEAD
        ref: values
      destination:
        server: '{{server}}'
        namespace: 'ingress'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

因為 `project` 參照至 sre-project 因此要在該 `AppProject` 資源 `sourceRepos` 欄位新增下面兩個來源。

```yaml
  sourceRepos:
...
  - https://kubernetes.github.io/ingress-nginx
  - https://github.com/CCH0124/argocd-sandbox.git
```

最後在平台上可以看到，下面一個 `nginx-ingress-appset` 資源和兩個 `Application` 資源。

```bash
$ argocd app get argo/sre-app -o tree
Name:               argo/sre-app
Project:            sre-project
Server:             https://kubernetes.default.svc
Namespace:          argo
URL:                https://argo.cch.com/applications/sre-app
Source:
- Repo:             https://github.com/CCH0124/argocd-gitops-manifest.git
  Target:           HEAD
  Path:             .
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (c24720a)
Health Status:      Healthy

KIND/NAME                                         STATUS  HEALTH   MESSAGE
...
ApplicationSet/nginx-ingress-appset               Synced  Healthy
├─Application/k3d-dev-cluster-ingress-controller
└─Application/stage-cluster-ingress-controller
```

這樣好處是未來每新增一個集群，只要配置 `matchExpressions.values` 欄位就會自動安裝 nginx ingress controller。


## Git 產生器範例

前提 team-a 的 Helm charts 儲存庫如下

```bash
$ tree -d charts/
charts/
├── project-a
│   ├── charts
│   └── templates
│       └── tests
└── project-b
    ├── charts
    └── templates
        └── tests
```

下面是一個 git 產生器範例，其目標是希望 team-a 的 Helm-charts 儲存庫中的 charts 目錄下每個專案的 Helm charts 都能夠同步至 dev/stage/prd 三個環境。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-generator-app
  namespace: argo
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - matrix:
      generators:
      - clusters: 
          selector:
            matchExpressions:
              - key: env
                operator: In
                values:
                  - dev
                  - stage
                  - prd
          values:
            env: '{{index .metadata.annotations "env"}}' 
      - git:
          repoURL: https://github.com/CCH0124/helm-charts.git
          revision: argo-appset
          directories:
          - path: charts/*
  template:
    metadata:
      name: '{{.path.basename}}-{{.values.env}}'
    spec:
      project: team-a-project
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
          allowEmpty: true
        syncOptions:
          - ApplyOutOfSyncOnly=true
          - CreateNamespace=false
      source:
        repoURL: https://github.com/CCH0124/helm-charts.git
        targetRevision: 'argo-appset'
        path: 'charts/{{.path.basename}}'
        helm:
          valueFiles:
            - 'values-{{.values.env}}.yaml'
          ignoreMissingValueFiles: true
          releaseName: '{{.path.basename}}-{{.values.env}}'
      destination:
        name: '{{.name}}'
        namespace: 'team-a'
```

這邊要注意一點，`env: '{{index .metadata.annotations "env"}}'` 如果放在第二個產生器位置會有解析問題。

最後產生結果會像下面結果，產生四個，因為當前只有 dev/stage 的群集被建立。讀者可以嘗試建立 prd 環境來驗證是否會多出兩個 `Application` 資源。

```bash
$ argocd app get argo/sre-app -o tree
Name:               argo/sre-app
Project:            sre-project
Server:             https://kubernetes.default.svc
Namespace:          argo
URL:                https://argo.cch.com/applications/sre-app
Source:
- Repo:             https://github.com/CCH0124/argocd-gitops-manifest.git
  Target:           HEAD
  Path:             .
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (8f0f5c4)
Health Status:      Healthy

KIND/NAME                                         STATUS  HEALTH   MESSAGE
...
ApplicationSet/git-generator-app                  Synced  Healthy  applicationset.argoproj.io/git-generator-app configured
├─Application/project-a-dev
├─Application/project-a-stage
├─Application/project-b-dev
└─Application/project-b-stage
...
```

team-a-project 會儲存 helm-charts 儲存庫，因此需要至 `AppProject` 資源下新增新的儲存庫來源。

```yaml
  sourceRepos:
  ...
  - https://github.com/CCH0124/helm-charts.git
```

上述 Matrix 產生器結果將會產生以下的值

```bash
- name: dev
  path: charts/project-a
  path.basename: project-a

- name: dev
  path: charts/project-b
  path.basename: project-b

- name: stage
  path: charts/project-a
  path.basename: project-a

- name: stage
  path: charts/project-b
  path.basename: project-b

```

上面的好處是，team-a 管理專案 Helm charts 時使用單一儲存庫，因此只要開發新專案 project-c 時，專案只需至 helm-charts 儲存庫下 charts 新增一個 project-c 的 Helm `charts，而維運也無須再進行配置，ApplicationSet` 就能將新專案部署至要的集群，其結果會如下。

```bash
- name: dev
  path: charts/project-a
  path.basename: project-a

- name: dev
  path: charts/project-b
  path.basename: project-b

- name: dev
  path: charts/project-c
  path.basename: project-c

- name: stage
  path: charts/project-a
  path.basename: project-a

- name: stage
  path: charts/project-b
  path.basename: project-b

- name: stage
  path: charts/project-c
  path.basename: project-c
```

而 Helm chart 的儲存庫會如下。

```bash
$ tree -d charts/
charts/
├── project-a
│   ├── charts
│   └── templates
│       └── tests
├── project-b
│   ├── charts
│   └── templates
│       └── tests
└── project-c
    ├── charts
    └── templates
        └── tests

```
## 結論

前面 Application CRD 可以讓我們方便的使用宣告式方式建立資源，但是此方式如果在複雜環境像是多集群，會遇到重複配置的問題，這樣一來就是機器人了。而 Argo CD 提供更高階的整合 ApplicationSet 這使的整合更複雜環境更加靈活有彈性，但是產生器變成是另外要學習的一個工。

除此之外，在團隊中要讓 ApplicationSet 能夠完美運行，想必要做出很多設計，像是建置集群時是否要規範標籤或是註解或是對於專案 Git 儲存庫要使用單一儲存庫單一分支或是多分支，這都會影響 `ApplicationSet` 的設計，透過此文章能夠大致了解 `clusters`、`matrix` 和 `git` 產生器能夠帶來的效益。


## 參考資源

- https://medium.com/@mprzygrodzki/argocd-applicationsset-with-helm-72bb6362d494
- https://cloud.redhat.com/blog/configuring-openshift-cluster-with-applicationsets-using-helmkustomize-and-acm-policies
- https://kubebyexample.com/learning-paths/argo-cd/argo-cd-working-kustomize