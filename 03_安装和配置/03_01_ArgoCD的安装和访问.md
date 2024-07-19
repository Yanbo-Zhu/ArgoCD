
# 1 Requirements

- Installed [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) command-line tool.
- Have a [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) file (default location is `~/.kube/config`).
- CoreDNS. Can be enabled for microk8s by `microk8s enable dns && microk8s stop && microk8s start`


# 2 ArgoCD安装

## 2.1 通过 kubectl  (官方推荐)

```
1：创建一个命名空间存放argocd的Pod
[root@k8s-master ~]# kubectl create ns argocd
namespace/argocd created


2：通过官方命令部署（镜像在国外，需要梯子，或者加速也行）
$ kubectl apply -n argocd -f https://ghproxy.com/https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

~# kubectl wait --for=condition=Ready pods --all -n argocd --timeout 300s
pod/argocd-application-controller-0 condition met
pod/argocd-applicationset-controller-6dd887f766-gv2gk condition met
pod/argocd-dex-server-89774cfc6-v2dhc condition met
pod/argocd-notifications-controller-79c985586c-xkt7q condition met
pod/argocd-redis-74f98b85f-dvhjc condition met
pod/argocd-repo-server-9cb997c7b-vjzrn condition met
pod/argocd-server-749879d78d-hcc6v condition met

3：查看部署状态
[root@k8s-master ~]# kubectl get pod,svc -n argocd
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                     1/1     Running   0          60s
pod/argocd-applicationset-controller-79f97597cb-mwzld   1/1     Running   0          62s
pod/argocd-dex-server-6fd8b59f5b-zx76f                  1/1     Running   0          62s
pod/argocd-notifications-controller-5549f47758-2rgjk    1/1     Running   0          61s
pod/argocd-redis-79bdbdf78f-xhd8f                       1/1     Running   0          61s
pod/argocd-repo-server-5569c7b657-t5ftv                 1/1     Running   0          61s
pod/argocd-server-664b7c6878-9tjlh                      1/1     Running   0          61s


```


1 new namespace "argocd" 会被创造出来
The installation manifests include ClusterRoleBinding resources that reference argocd namespace. If you are installing Argo CD into a different namespace then make sure to update the namespace reference.

2 ClusterRoleBinding resources 
The installation manifests include ClusterRoleBinding resources that reference argocd namespace. If you are installing Argo CD into a different namespace then make sure to update the namespace reference.

---
以下的都可以省略 

安装客户端
```
 	
sudo curl -sSL -o /usr/local/bin/argo https://github.com/argoproj/argo/releases/download/v2.2.1/argo-linux-amd64
sudo chmod +x /usr/local/bin/argo
```


安装服务器

执行下面的命令，在K8S集群中安装控制器和UI： 

```
kubectl create ns argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/v2.2.1/manifests/install.yaml
```


你需要为argo使用的Service Account提供足够的权限：
```
# 默认帐户
kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=default:default
```

或者，在提交工作流时，指定运行工作流时使用的帐户：
`argo submit --serviceaccount [name]`



## 2.2 通过二进制文件 

我们也可以使用命令行客户端来访问 Argo CD，首先使用 curl 命令下载：
```
curl -LO https://github.com/argoproj/argo-cd/releases/download/v2.7.1/argocd-linux-amd64
```

然后使用 install 命令安装：
```
$ sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```



## 2.3 Kustomize
Argo CD 配置清单也可以使用 Kustomize 来部署，建议通过远程的 URL 来调用配置清单，使用 patch 来配置自定义选项。

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/v2.0.4/manifests/ha/install.yaml

```

For an example of this, see the [kustomization.yaml](https://github.com/argoproj/argoproj-deployments/blob/master/argocd/kustomization.yaml) used to deploy the [Argoproj CI/CD infrastructure](https://github.com/argoproj/argoproj-deployments#argoproj-deployments).


## 2.4 Helm

Argo CD 的 Helm Chart 目前由社区维护，地址： https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd。

https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#helm

argo-cd 5.5.22 · argoproj/argo (artifacthub.io)

```
helm repo add argo https://argoproj.github.io/argo-helm
helm install my-argo-cd argo/argo-cd --version 5.5.22
```

## 2.5 自定义安装 

Argo CD manifests 也可以使用自定义安装。建议将 manifests 作为远程资源包含在内，并使用 Kustomize 补丁应用其他自定义
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/v2.0.4/manifests/ha/install.yaml

```


# 3 two type of installations

https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/

## 3.1 多租户 Multi-Cluster

Argo CD 最常用的部署模式是多租户，一般如果组织内部包含多个应用研发团队，就会采用这种部署模式。用户可以使用可视化界面或者 argocd CLI 来访问 Argo CD。argocd CLI 必须先通过 `argocd login <server-host>` 来获取 Argo CD 的访问授权

```
$ argocd login SERVER [flags]

## Login to Argo CD using a username and password
$ argocd login cd.argoproj.io

## Login to Argo CD using SSO
$ argocd login cd.argoproj.io --sso

## Configure direct access using Kubernetes API server
$ argocd login cd.argoproj.io --core
```

多租户模式提供了两种不同的配置清单：

1 非高可用
推荐用于测试和演示环境，不推荐在生产环境下使用。有两种部署清单可供选择：
- [install.yaml](https://icloudnative.io/go/?target=aHR0cHM6Ly9naXRodWIuY29tL2FyZ29wcm9qL2FyZ28tY2QvYmxvYi9tYXN0ZXIvbWFuaWZlc3RzL2luc3RhbGwueWFtbA%3d%3d) - 标准的 Argo CD 部署清单，拥有集群管理员权限。可以使用 Argo CD 在其运行的集群内部署应用程序，也可以通过接入外部集群的凭证将应用部署到外部集群中。
- [namespace-install.yaml](https://icloudnative.io/go/?target=aHR0cHM6Ly9naXRodWIuY29tL2FyZ29wcm9qL2FyZ28tY2QvYmxvYi9tYXN0ZXIvbWFuaWZlc3RzL25hbWVzcGFjZS1pbnN0YWxsLnlhbWw%3d) - 这个部署清单只需要 namespace 级别的权限。如果你不需要在 Argo CD 运行的集群中部署应用，只需通过接入外部集群的凭证将应用部署到外部集群中，推荐使用此部署清单。还有一种花式玩法，你可以为每个团队分别部署单独的 Argo CD 实例，但是每个 Argo CD 实例都可以使用特殊的凭证（例如 `argocd cluster add <CONTEXT> --in-cluster --namespace <YOUR NAMESPACE>`）将应用部署到同一个集群中（即 `kubernetes.svc.default`，也就是内部集群）

> 注意：namespace-install.yaml 配置清单中并不包含 Argo CD 的 CRD，需要自己提前单独部署：kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable。



2 高可用
与非高可用部署清单包含的组件相同，但增强了高可用能力和弹性能力，推荐在生产环境中使用。
- [ha/install.yaml](https://icloudnative.io/go/?target=aHR0cHM6Ly9naXRodWIuY29tL2FyZ29wcm9qL2FyZ28tY2QvYmxvYi9tYXN0ZXIvbWFuaWZlc3RzL2hhL2luc3RhbGwueWFtbA%3d%3d) - 与上文提到的 install.yaml 的内容相同，但配置了相关组件的多个副本。
- [ha/namespace-install.yaml](https://icloudnative.io/go/?target=aHR0cHM6Ly9naXRodWIuY29tL2FyZ29wcm9qL2FyZ28tY2QvYmxvYi9tYXN0ZXIvbWFuaWZlc3RzL2hhL25hbWVzcGFjZS1pbnN0YWxsLnlhbWw%3d) - 与上文提到的 namespace-install.yaml 相同，但配置了相关组件的多个副本。


## 3.2 Core

If you are not interested in UI, SSO, and multi-cluster features, then you can install only the [core](https://argo-cd.readthedocs.io/en/stable/getting_started/operator-manual/core/#installing) Argo CD components.


The Argo CD Core installation is primarily used to deploy Argo CD in headless mode. This type of installation is most suitable for cluster administrators who independently use Argo CD and don't need multi-tenancy features. This installation includes fewer components and is easier to setup. The bundle does not include the API server or UI, and installs the lightweight (non-HA) version of each component.

Installation manifest is available at [core-install.yaml](https://github.com/argoproj/argo-cd/blob/master/manifests/core-install.yaml).

For more details about Argo CD Core please refer to the [official documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/core/)



Core 模式也就是最精简的部署模式，不包含 API Server 和可视化界面，只部署了每个组件的轻量级（非高可用）版本。
用户需要 Kubernetes 访问权限来管理 Argo CD，因此必须使用下面的命令来配置 argocd CLI：

```
$ kubectl config set-context --current --namespace=argocd # change current kube context to argocd namespace
$ argocd login --core
```

也可以使用命令 argocd admin dashboard 手动启用可视化界面。
具体的配置清单位于 Git 仓库中的 core-install.yaml。
https://github.com/argoproj/argo-cd/blob/master/manifests/core-install.yaml

除了直接通过原生的配置清单进行部署，Argo CD 还支持额外的配置清单管理工具。

Web UI 也可以使用，可以使用以下命令启动。
```
argocd admin dashboard
```

具体的 manifests 对应于仓库中的 core-install.yaml 。



# 4 安装完成后还需要配置证书 

This default installation will have a self-signed certificate and cannot be accessed without a bit of extra work. Do one of:

- Follow the [instructions to configure a certificate](https://argo-cd.readthedocs.io/en/stable/operator-manual/tls/) (and ensure that the client OS trusts it).
- Configure the client OS to trust the self signed certificate.
- Use the --insecure flag on all Argo CD CLI operations in this guide.


# 5 使用命令将namespace长期设置为argocd

Default namespace for `kubectl` config must be set to `argocd`. This is only needed for the following commands since the previous commands have -n argocd already: `kubectl config set-context --current --namespace=argocd`


# 6 Download Argo CD CLI 

Download the latest Argo CD version from [https://github.com/argoproj/argo-cd/releases/latest](https://github.com/argoproj/argo-cd/releases/latest). More detailed installation instructions can be found via the [CLI installation documentation](https://argo-cd.readthedocs.io/en/stable/cli_installation/).

Also available in Mac, Linux and WSL Homebrew:
```
brew install argocd
```


# 7 Access The Argo CD API Server


By default, the Argo CD API server is not exposed with an external IP. To access the API server, choose one of the following techniques to expose the Argo CD API server:



部署完成后，可以通过 Service argocd-server 来访问可视化界面

```
$ kubectl -n argocd get svc                                                             
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.105.250.212   <none>        7000/TCP,8080/TCP            5m10s
argocd-dex-server                         ClusterIP   10.108.88.97     <none>        5556/TCP,5557/TCP,5558/TCP   5m10s
argocd-metrics                            ClusterIP   10.103.11.245    <none>        8082/TCP                     5m10s
argocd-notifications-controller-metrics   ClusterIP   10.98.136.200    <none>        9001/TCP                     5m9s
argocd-redis                              ClusterIP   10.110.151.108   <none>        6379/TCP                     5m9s
argocd-repo-server                        ClusterIP   10.109.131.197   <none>        8081/TCP,8084/TCP            5m9s
argocd-server                             ClusterIP   10.98.23.255     <none>        80/TCP,443/TCP               5m9s
argocd-server-metrics                     ClusterIP   10.103.184.121   <none>        8083/TCP                     5m8s

```




## 7.1 方法1: 修改 service type 
Change the argocd-server service type to LoadBalancer:
Argo CD 部署好之后，默认情况下，API Server 从集群外是无法访问的，这是因为 API Server 的服务类型是 ClusterIP：
```
$ kubectl get svc argocd-server -n argocd
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
argocd-server   ClusterIP   10.111.209.6   <none>        80/TCP,443/TCP               23h
```


我们可以使用 kubectl patch 将其改为 NodePort 或 LoadBalancer：
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

修改之后，Kubernetes 会为 API Server 随机分配端口：
```
$ kubectl get svc argocd-server -n argocd
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
argocd-server   NodePort   10.111.209.6   <none>        80:32130/TCP,443:31205/TCP   23h
```

这时我们就可以通过 localhost:32130 或 localhost:31205 来访问 API Server 了：

---


```
# 我这里使用 NodePort
# 但是我们需要修改一下argocd-server的暴露方式为NodePort
[root@k8s-master argocd]# kubectl edit svc -n argocd argocd-server
......
  selector:
    app.kubernetes.io/name: argocd-server
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}

[root@k8s-master argocd]# kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   200.1.73.127    <none>        7000/TCP                     15m
argocd-dex-server                         ClusterIP   200.1.77.207    <none>        5556/TCP,5557/TCP,5558/TCP   15m
argocd-metrics                            ClusterIP   200.1.88.62     <none>        8082/TCP                     15m
argocd-notifications-controller-metrics   ClusterIP   200.1.185.23    <none>        9001/TCP                     15m
argocd-redis                              ClusterIP   200.1.5.40      <none>        6379/TCP                     15m
argocd-repo-server                        ClusterIP   200.1.249.26    <none>        8081/TCP,8084/TCP            15m
argocd-server                             NodePort    200.1.15.59     <none>        80:31715/TCP,443:30604/TCP   15m
argocd-server-metrics                     ClusterIP   200.1.216.113   <none>        8083/TCP                     15m

访问节点IP+30604即可
```

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423181139428-1569290379.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423181139428-1569290379.png)


## 7.2 方法2 使用 kubectl port-forward 命令进行端口转发

Kubectl port-forwarding can also be used to connect to the API server without exposing the service.
可以通过ingress或者nodeport方式暴露进行访问！
因为需要访问，我们可以通过NodePort或者Ingress暴露 argocd-server


如果你的客户端可以直连 Service IP，那就直接可以通过 argocd-server 的 Cluster IP 来访问。或者可以直接通过本地端口转发来访问：

除了修改服务类型，官方还提供了两种方法暴露 API Server：一种是 使用 Ingress 网关，另一种是使用 kubectl port-forward 命令进行端口转发：
```
kubectl port-forward svc argocd-server -n argocd 8080:443


$ kubectl port-forward svc/argocd-server -n argocd 8080:443
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

```

The API server can then be accessed using https://localhost:8080

## 7.3 方法3  Ingress

Follow the [ingress documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/) on how to configure Argo CD with ingress.



## 7.4 不构建AccessToArgoCDApiServer也可以使用CLI的方式 

The CLI environment must be able to communicate with the Argo CD API server. If it isn't directly accessible as described above in step 3, you can tell the CLI to access it using port forwarding through one of these mechanisms: 1) add `--port-forward-namespace argocd` flag to every CLI command; or 2) set `ARGOCD_OPTS` environment variable: `export ARGOCD_OPTS='--port-forward-namespace argocd'`.




# 8 查询登录密码

可以看到 API Server 需要登录才能访问，初始用户名为 admin，初始密码在部署时随机生成，并保存在 argocd-initial-admin-secret 这个 Secret 里：

The initial password for the `admin` account is auto-generated and stored as clear text in the field `password` in a secret named `argocd-initial-admin-secret` in your Argo CD installation namespace. You can simply retrieve this password using the `argocd` CLI:

```
argocd admin initial-password -n argocd

```

```
$ kubectl get secrets argocd-initial-admin-secret -n argocd -o yaml
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo


apiVersion: v1
data:
  password: SlRyZDYtdEpOT1JGcXI3QQ==
kind: Secret
metadata:
  creationTimestamp: "2023-05-04T00:14:19Z"
  name: argocd-initial-admin-secret
  namespace: argocd
  resourceVersion: "17363"
  uid: 0cce4b4a-ff9d-44b3-930d-48bc5530bef0
type: Opaque
```


密码以 BASE64 形式存储，可以使用下面的命令快速得到明文密码：
```
账号：admin

# 获取密码方式如下
[root@k8s-master argocd]# echo $(kubectl get secret -n argocd argocd-initial-admin-secret -o yaml | grep password | awk -F: '{print $2}') | base64 -d
密码：U8g9xqXAPIRz6Ds3
```



```

user：admin
passwd：
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```



# 9 修改默认密码

```
注：
修改密码前，先使用 argocd login 登录到 ArgoCD 服务端。
$ argocd account update-password
*** Enter password of currently logged in user (admin):
*** Enter new password for user admin:


或者 
$ argocd account update-password --account admin --current-password xxxx --new-password xxxx


```


输入用户名和密码登录成功后，进入 Argo CD 的应用管理页面：
[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423181358502-1739640554.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423181358502-1739640554.png)

## 9.1 删除 argocd-initial-admin-secret

You should delete the argocd-initial-admin-secret from the Argo CD namespace once you changed the password. The secret serves no other purpose than to store the initially generated password in clear and can safely be deleted at any time. 

It will be re-created on demand by Argo CD if a new admin password must be re-generated.




# 10 登录Argocd

对argoCD的操作需要先登录。
```
[root@node1 ~]# argocd login 10.233.37.63
WARNING: server certificate had error: x509: cannot validate certificate for 10.233.37.63 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? y
Username: admin
Password: 
'admin:login' logged in successfully
Context '10.233.37.63' updated
```
