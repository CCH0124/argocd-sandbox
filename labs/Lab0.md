## 建置 ArgoCD

首先，使用 K3d 建立給 ArgoCD 的 Kubernetes 集群。其環境，將 Ingress 資源關閉，因此此範例安裝 [Nginx Ingress](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/)。

```bash
k3d cluster create -c argocd-conf.yaml --servers-memory 6G --agents-memory 2G
```

透過 Helm 方式，安裝 ArgoCD，版本使用 `7.1.3`。

```bash
$ helm repo add argo https://argoproj.github.io/argo-helm
$ helm search repo argo-cd
NAME            CHART VERSION   APP VERSION     DESCRIPTION
argo/argo-cd    7.1.3           v2.11.3         A Helm chart for Argo CD, a declarative, GitOps...
$ helm install argo-cd argo/argo-cd --version 7.1.3 --namespace argo --create-namespace -f values.yaml
NAME: argo-cd
LAST DEPLOYED: Mon Feb 27 14:54:00 2023
NAMESPACE: argo
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argo-cd-argocd-server -n argo 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argo get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```

## Install Argo cli

安裝 Cli 工具，版本目前依照 ArgoCD 環境。

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v2.11.5/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

查看版本資訊:

```bash
$ argocd version
argocd: v2.11.5+c4b283c
  BuildDate: 2024-07-15T18:15:32Z
  GitCommit: c4b283ce0c092aeda00c78ae7b3b2d3b28e7feec
  GitTreeState: clean
  GoVersion: go1.21.12
```

安裝完上述後，接著 login 

```bash
$ argocd login argo.cch.com:8443
WARNING: server certificate had error: x509: certificate is valid for ingress.local, not argo.cch.com. Proceed insecurely (y/n)?
WARNING: server certificate had error: x509: certificate is valid for ingress.local, not argo.cch.com. Proceed insecurely (y/n)? y
WARN[0012] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web.
Username: admin
Password:
'admin:login' logged in successfully
Context 'argo.cch.com:8443' updated
```

`Password` 使用預設 argocd 創建的。

透過以下指令可以觀看目前管理的群集和當前使用的群集

```bash
$ argocd context
CURRENT  NAME               SERVER
*        argo.cch.com:8443  argo.cch.com:8443

$ argocd context argo.cch.com:8443 # 切換

$ argocd cluster list
WARN[0000] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web.
SERVER                          NAME        VERSION  STATUS   MESSAGE                                                  PROJECT
https://kubernetes.default.svc  in-cluster           Unknown  Cluster has no applications and is not being monitored.
```

另外，預設 ArgoCD 建立起來時管理的 Kubernetes 群集是 ArgoCD 被安裝到的 Kubernetes 群集。

```bash
$ argocd cluster list
SERVER                                  NAME             VERSION  STATUS   MESSAGE                                                  PROJECT
https://kubernetes.default.svc          in-cluster                Unknown  Cluster has no applications and is not being monitored.  
```


ArgoCD 會讀取我們預設 Kubernetes 的 `Service` 服務。

```bash
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   3h6m
```

## 變更密碼

```bash
$ argocd login argo.cch.com:8443 --grpc-web-root-path argocd --username admin
$ argocd account update-password --account admin --grpc-web-root-path argocd
WARN[0000] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web.
*** Enter password of currently logged in user (admin):
*** Enter new password for user admin:
*** Confirm new password for user admin:
Password updated
Context 'argo.cch.com:8443/argocd' updated
```

## 管理多個 Cluster

ArgoCD 要管理多個 Kubernetes 來說，目前已知會有兩種方式，透過 ArgoCD `secret` 資源和透過 CLI 方式，下面會演示兩種方式。

### CLI 與 KUBECONFIG

使用 Argo CLI 會需要使用 `KUBECONFIG` 讓 ArgoCD 來對集群進行管理。下面就開始演示。


首先，透過 K3d 環境建立 dev 環境，建立如下。建置後可藉由 `kubectl config get-contexts` 驗證，是否有建置 `k3d-dev-cluster`。

```bash
infra/k3d$ k3d cluster create -c dev-conf.yaml --servers-memory 2G --agents-memory 2G
$ kubectl config get-contexts 
CURRENT   NAME                                                                 CLUSTER                                                              AUTHINFO                                                             NAMESPACE
          k3d-argo-cluster                                                     k3d-argo-cluster                                                     admin@k3d-argo-cluster                                               
*         k3d-dev-cluster                                                      k3d-dev-cluster                                                      admin@k3d-dev-cluster                                                
          k3d-lab-cluster                                                      k3d-lab-cluster                                                      admin@k3d-lab-cluster                                       
```

那對於每個 K3d 所建立出來對集群的連線是由一些複雜網路技巧所組成，因此預設下 ArgoCD 無法透過 `KUBECONFIG` 內的連線位置進行存取。需透過容器中 IP。**這邊讓 K3d 建立的集群都共用同一個 docker 建立的網路**。

藉由以下方式獲取 `k3d-dev-cluster` 的 IP 位置。

```bash
// 對於 k3d-dev-cluster 他的 IP 位置是 172.18.0.6
$ docker exec k3d-dev-cluster-server-0 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:06  
          inet addr:172.18.0.6  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:15514 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8103 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:37337554 (35.6 MiB)  TX bytes:1535864 (1.4 MiB)
```

如下，這邊修改 `KUBECONFIG` 中 `k3d-dev-cluster` 的 API Server 連線位置，以讓 ArgoCD 能夠連線。

```
$ vi .kube/config

....
server: https://dev.cch.com:6446 # 替換成 https://172.18.0.6:6443
...
```

> .kube/config 是預設放置 Kubernetes 集群配置訊息的檔案

替換完成後即可透過 ArgoCD CLI 方式加入集群，先透過 `argocd login` 登入，再來使用 `argocd cluster add` 新增集群，其格式 
```bash
argocd cluster add --kubeconfig <path-of-kubeconfig-file> --kube-context string <cluster-context> --name <cluster-name>
```

```bash
$ argocd login argo.cch.com:8443 --grpc-web-root-path argocd --username admin
$ argocd  cluster add --kubeconfig ~/.kube/config --kube-context string k3d-dev-cluster --name k3d-dev-cluster
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `k3d-dev-cluster` with full cluster level privileges. Do you want to continue [y/N]? y
INFO[0001] ServiceAccount "argocd-manager" created in namespace "kube-system" 
INFO[0001] ClusterRole "argocd-manager-role" created    
INFO[0001] ClusterRoleBinding "argocd-manager-role-binding" created 
INFO[0006] Created bearer token secret for ServiceAccount "argocd-manager" 
Cluster 'https://172.18.0.6:6443' added
```

> 獲取 Kubernetes context 名稱可以使用 `kubectl config get-contexts -o name` 指令

藉由 `argocd` 指令驗證：

```bash
$ argocd  cluster list
SERVER                          NAME             VERSION  STATUS   MESSAGE                                                  PROJECT
https://172.18.0.6:6443         k3d-dev-cluster           Unknown  Cluster has no applications and is not being monitored.  
https://kubernetes.default.svc  in-cluster                Unknown  Cluster has no applications and is not being monitored.  
```


## ArgoCD 之 Secret 資源

第二種方式是使用 Secret 資源綁定 Kubernetes 集群存取資訊。這用另一種方式讓 ArgoCD 能夠加入要管理的 Kubernetes 集群，這邊會用 ServiceAccount 所綁定的 Token 進行綁定，而非來自 `KUBECONFIG` 中資訊。

首先建立一個 stage 環境。

```bash
$ k3d cluster create -c stag-conf.yaml --servers-memory 2G --agents-memory 2G
```

接著要至 `stage` 環境建立要給 ArgoCD 存取的 ServiceAccount 的資訊，記住 ServiceAccount 需要綁定 Token，要給 ArgoCD 使用。步驟如下：

1. 切換至 stage 環境
```bash
$ kubectl config use-context k3d-stage-cluster
```

2. 建立 ServiceAccount、ClusterRole 和 ClusterRoleBinding 的資源

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-admin-role
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-admin-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-admin-role
subjects:
- kind: ServiceAccount
  name: argocd-admin
  namespace: kube-system
```

3. 建立給 ServiceAccount 的 Token

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-admin-token
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: argocd-admin
type: kubernetes.io/service-account-token
```

4. 獲取 CA 與 Token

```bash
export CA=$(kubectl get -n kube-system secret/argocd-admin-token -o jsonpath='{.data.ca\.crt}')

export TOKEN=$(kubectl get -n kube-system secret/argocd-admin-token -o jsonpath='{.data.token}' | base64 --decode)
```

5. 至 ArgoCD 的 Server 位置，建置 Secret 資源

在建立 Cluster 資訊給 ArgoCD 時，每個 Secret 資源，必須有標籤 `argocd.argoproj.io/secret-type: cluster`。

```bash
cat <<EOF |  kubectl -n argo apply --context k3d-argo-cluster  -f -
apiVersion: v1
kind: Secret
metadata:
  name: stage-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: stage-cluster
  server: https://172.18.0.9:6443
  namespaces: "team-a, ingress"
  clusterResources: "true"
  config: |
    {
      "bearerToken": "$TOKEN",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "$CA"
      }
    }
EOF
```

這邊 `insecure` 設置為 false 表示不做 TLS 驗證，`serverName` 部分不做設定，實務上應當設定，因該藉由 `serverName` 來做 TLS 的驗證，他應當是 SNI 的值。`namespaces` 限制了兩個 `namespace`，因此這之外的資源，Argo CD 理論上是沒有辦法進行資源部署，會出現以下可能訊息。

```bash
ComparisonError
Failed to load live state: Namespace "team-c" for LimitRange "stage-limit-team-c" is not managed
```

關於配置的參數官方描述的很詳細，可參考官方資源[ArgoCD | clusters](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)。

透過 `argocd cluster` 指令，可以看到該從 K3d 建立的 k3d-stage-cluster 群集資源被 ArgoCD 管理。

```bash
$ argocd cluster list
SERVER                                  NAME             VERSION  STATUS   MESSAGE                                                  PROJECT
https://172.18.0.6:6443                 k3d-dev-cluster           Unknown  Cluster has no applications and is not being monitored.  
https://172.18.0.9:6443 (2 namespaces)  stage-cluster             Unknown  Cluster has no applications and is not being monitored.  
https://kubernetes.default.svc          in-cluster                Unknown  Cluster has no applications and is not being monitored. 
```

>相對要移除管理的集群可用 `argocd cluster rm [SERVER_NAME]`

