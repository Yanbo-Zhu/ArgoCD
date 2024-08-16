

# 1 例子1: 通过 Web UI 部署应用


https://www.cnblogs.com/layzer/articles/ArgoCD_deploy_use.html


## 1.1 配置某个Repo

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203203997-293353688.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203203997-293353688.png)

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203232673-1382978628.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203232673-1382978628.png)

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203601448-950502624.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203601448-950502624.png)

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203626821-883605740.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203626821-883605740.png)


## 1.2 创建项目

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203716272-1126088758.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203716272-1126088758.png)

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203755163-646341050.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203755163-646341050.png)



[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203855448-78327339.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203855448-78327339.png)

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203926406-828448638.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423203926406-828448638.png)




![new-app.png](https://www.aneasystone.com/usr/uploads/2023/05/70599187.png "new-app.png")


- 通用配置
    - 应用名称：`guestbook`
    - 项目名称：`default`
    - 同步策略：`Manual`
- 源配置
    - Git 仓库地址：`https://github.com/argoproj/argocd-example-apps.git`
    - 分支：`HEAD`
    - 代码路径：`guestbook`
- 目标配置
    - 集群地址：`https://kubernetes.default.svc`
    - 命名空间：`default`

其他的选项暂时可以不用管，如果想了解具体内容可以参考官方文档 Sync Options，填写完成后，点击 CREATE 按钮即可创建应用：




----

准备项目yaml并上传到git仓库
```sh


[root@k8s-master flask]# cat demo.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      imagePullSecrets:
      - name: harbor
      containers:
      - name: demo
        image: registry.kubernetes-devops.cn/library/nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo
  namespace: demo
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: demo
    
[root@k8s-master flask]# git add .
[root@k8s-master flask]# git commit -m "demo"
[master eddda10] demo
 1 file changed, 9 insertions(+), 9 deletions(-)
 rename nginx.yaml => demo.yaml (77%)
[root@k8s-master flask]# git push origin master 
Username for 'http://10.0.0.10:31179': devops
Password for 'http://devops@10.0.0.10:31179': 
Counting objects: 4, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 509 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote: . Processing 1 references
remote: Processed 1 references in total
To http://10.0.0.10:31179/devops/flask.git
   d31d433..eddda10  master -> master

```


然后我们去ArgoCD去部署这个demo
因为刚刚填写的同步策略是手工同步，所以我们能看到应用的状态还是 OutOfSync，点击应用详情：
[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204356915-1888675531.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204356915-1888675531.png)

因为刚刚填写的同步策略是手工同步，所以我们能看到应用的状态还是 `OutOfSync`，点击应用详情：

![new-app-not-sync-detail.png](https://www.aneasystone.com/usr/uploads/2023/05/2723077529.png "new-app-not-sync-detail.png")

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204425645-1984148031.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204425645-1984148031.png)


可以看到 Argo CD 已经从 Git 仓库的代码中解析出应用所包含的 Kubernetes 资源了，guestbook 应用包含了一个 Deployment 和 一个 Service，点击 SYNC 按钮触发同步，Deployment 和 Service（以及它们关联的资源）被成功部署到 Kubernetes 集群中

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204450538-645431991.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204450538-645431991.png)

[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204510802-2120129897.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204510802-2120129897.png)
测试完成后，点击 DELETE 删除应用，该应用关联的资源将会被级联删除。

## 1.3 访问刚刚部署的应用 

我们查看一下部署情况，并看看部署之后是否可以访问

```sh
[root@k8s-master flask]# kubectl get pod,svc -n demo 
NAME                        READY   STATUS    RESTARTS   AGE
pod/demo-8645cf44c9-2pkv6   1/1     Running   0          84s

NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/demo   NodePort   200.1.73.45   <none>        80:30808/TCP   84s


# 测试访问
[root@k8s-master flask]# curl 10.0.0.10:30808 -I
HTTP/1.1 200 OK
Server: nginx/1.21.5
Date: Sat, 23 Apr 2022 12:46:29 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 28 Dec 2021 18:48:00 GMT
Connection: keep-alive
ETag: "61cb5be0-267"
Accept-Ranges: bytes
```


[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204657774-1780679496.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423204657774-1780679496.png)

## 1.4 变更一下manifest

这个时候我们去变更一下代码。（变更一下yaml）

```sh
这个时候我们去变更一下代码。（变更一下yaml）

[root@k8s-master flask]# cat demo.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      imagePullSecrets:
      - name: harbor
      containers:
      - name: demo
        image: registry.kubernetes-devops.cn/library/httpd:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo
  namespace: demo
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: demo
    
# 这里更换一个镜像，然后我们提交以下代码并在ArgoCD再次 SYNC一下
[root@k8s-master flask]# git add .
[root@k8s-master flask]# git commit -m "fix httpd"
[master 0d963aa] fix httpd
 1 file changed, 1 insertion(+), 1 deletion(-)
[root@k8s-master flask]# git push origin master 
Username for 'http://10.0.0.10:31179': devops
Password for 'http://devops@10.0.0.10:31179': 
Counting objects: 5, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 280 bytes | 0 bytes/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: . Processing 1 references
remote: Processed 1 references in total
To http://10.0.0.10:31179/devops/flask.git
   eb92f56..0d963aa  master -> master

```

![](image/Pasted%20image%2020240710151513.png)



 我们在增加新服务的时候我们可以看看pod的变化
```sh

[root@k8s-master flask]# kubectl get pod -n demo --watch
NAME                    READY   STATUS    RESTARTS   AGE
demo-8645cf44c9-jf4g5   1/1     Running   0          3m20s
demo-9f6c4b7f5-5mmqp    0/1     Pending   0          0s
demo-9f6c4b7f5-5mmqp    0/1     Pending   0          0s
demo-9f6c4b7f5-5mmqp    0/1     ContainerCreating   0          0s
demo-9f6c4b7f5-5mmqp    0/1     ContainerCreating   0          0s
demo-9f6c4b7f5-5mmqp    1/1     Running             0          2s
demo-8645cf44c9-jf4g5   1/1     Terminating         0          3m28s
demo-8645cf44c9-jf4g5   1/1     Terminating         0          3m28s
demo-8645cf44c9-jf4g5   0/1     Terminating         0          3m29s
demo-8645cf44c9-jf4g5   0/1     Terminating         0          3m29s
demo-8645cf44c9-jf4g5   0/1     Terminating         0          3m29s
# 这里可以看到，更新策略是先启动一个新的然后再删除老的，这里测试一下访问

[root@k8s-master flask]# kubectl get pod,svc -n demo 
NAME                       READY   STATUS    RESTARTS   AGE
pod/demo-9f6c4b7f5-5mmqp   1/1     Running   0          95s

NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/demo   NodePort   200.1.240.228   <none>        80:31086/TCP   5m1s
[root@k8s-master flask]# curl 10.0.0.10:31086
<html><body><h1>It works!</h1></body></html>
# 这里可以看到已经更新了
# 那么如果我们在这个yaml里面再增加一个pod呢？我们来实践一下，再次变更代码
[root@k8s-master flask]# cat demo.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      imagePullSecrets:
      - name: harbor
      containers:
      - name: demo
        image: registry.kubernetes-devops.cn/library/httpd:latest
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demos
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demos
  template:
    metadata:
      labels:
        app: demos
    spec:
      imagePullSecrets:
      - name: harbor
      containers:
      - name: demos
        image: registry.kubernetes-devops.cn/library/nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo
  namespace: demo
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: demo
---
apiVersion: v1
kind: Service
metadata:
  name: demos
  namespace: demo
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: demos
# 提交代码
[root@k8s-master flask]# git add .
[root@k8s-master flask]# git commit -m "add service"
[master 16ccff1] add service
 1 file changed, 38 insertions(+)
[root@k8s-master flask]# git push origin master 
Username for 'http://10.0.0.10:31179': devops
Password for 'http://devops@10.0.0.10:31179': 
Counting objects: 5, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 318 bytes | 0 bytes/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: . Processing 1 references
remote: Processed 1 references in total
To http://10.0.0.10:31179/devops/flask.git
   0d963aa..16ccff1  master -> master
```


[![](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423210857215-1975073797.png)](https://img2022.cnblogs.com/blog/2222036/202204/2222036-20220423210857215-1975073797.png)

在ArgoCD内再次SYNC，然后观察容器的更新
```

[root@k8s-master flask]# kubectl get pod -n demo --watch
NAME                   READY   STATUS    RESTARTS   AGE
demo-9f6c4b7f5-5mmqp   1/1     Running   0          6m43s
demos-7d56f6966c-brsvt   0/1     Pending   0          0s
demos-7d56f6966c-brsvt   0/1     Pending   0          0s
demos-7d56f6966c-brsvt   0/1     ContainerCreating   0          0s
demos-7d56f6966c-brsvt   0/1     ContainerCreating   0          0s
demos-7d56f6966c-brsvt   1/1     Running             0          1s
```


# 2 例子：用 Web UI部署

## 2.1 准备 Git 仓库

在 GitHub 上创建一个项目，取名为 [argocd-lab](https://icloudnative.io/go/?target=aHR0cHM6Ly9naXRodWIuY29tL3lhbmdjaHVhbnNoZW5nL2FyZ29jZC1sYWI%3d)，为了方便实验将仓库设置为公共仓库。在仓库中新建 dev 目录，在目录中创建两个 YAML 配置清单，分别是 `deployment.yaml` 和 `service.yaml`。

![](image/Pasted%20image%2020240710184043.png)


配置清单内容如下
```
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 2
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:latest
        ports:
        - containerPort: 80
        
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80

```


接下来在仓库根目录中创建一个 Application 的配置清单
```
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/yangchuansheng/argocd-lab.git
    targetRevision: HEAD
    path: dev
  destination: 
    server: https://kubernetes.default.svc
    namespace: myapp

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true

```

参数解释：
- **syncPolicy** : 指定自动同步策略和频率，不配置时需要手动触发同步。
- **syncOptions** : 定义同步方式。
    - **CreateNamespace=true** : 如果不存在这个 namespace，就会自动创建它。
- **automated** : 检测到实际状态与期望状态不一致时，采取的同步措施。
    - **selfHeal** : 当集群世纪状态不符合期望状态时，自动同步。
    - **prune** : 自动同步时，删除 Git 中不存在的资源。

Argo CD 默认情况下每 3 分钟会检测 Git 仓库一次，用于判断应用实际状态是否和 Git 中声明的期望状态一致，如果不一致，状态就转换为 OutOfSync。默认情况下并不会触发更新，除非通过 syncPolicy 配置了自动同步。

如果嫌周期性同步太慢了，也可以通过设置 Webhook 来使 Git 仓库更新时立即触发同步。具体的使用方式会放到后续的教程中，本文不再赘述


## 2.2 创建 Application
现在万事具备，只需要通过 application.yaml 创建 Application 即可。

$ kubectl apply -f application.yaml
application.argoproj.io/myapp-argo-application created

在 Argo CD 可视化界面中可以看到应用已经创建成功了。
点进去可以看到应用的同步详情和各个资源的健康状况。
![](image/Pasted%20image%2020240710184234.png)



如果你更新了 deployment.yaml 中的镜像，Argo CD 会自动检测到 Git 仓库中的更新，并且将集群中 Deployment 的镜像更新为 Git 仓库中最新设置的镜像版本。


# 3 Register A Cluster To Deploy Apps To (Optional)[¶](https://argo-cd.readthedocs.io/en/stable/getting_started/#5-register-a-cluster-to-deploy-apps-to-optional "Permanent link")

This step registers a cluster's credentials to Argo CD, and is only necessary when deploying a pplication  to an external cluster. When deploying internally (to the same cluster that Argo CD is running in), https://kubernetes.default.svc should be used as the application's K8s API server address.

First list all clusters contexts in your current kubeconfig:

```
kubectl config get-contexts -o name
```

Choose a context name from the list and supply it to `argocd cluster add CONTEXTNAME`. For example, for docker-desktop context, run:

```
argocd cluster add docker-desktop
```

The above command installs a ServiceAccount (`argocd-manager`), into the kube-system namespace of that kubectl context, and binds the service account to an admin-level ClusterRole. Argo CD uses this service account token to perform its management tasks (i.e. deploy/monitoring).

Note

The rules of the `argocd-manager-role` role can be modified such that it only has `create`, `update`, `patch`, `delete` privileges to a limited set of namespaces, groups, kinds. However `get`, `list`, `watch` privileges are required at the cluster-scope for Argo CD to function.

# 4 例子: 用 CLI部署应用

https://blog.csdn.net/chengyinwu/article/details/131957249

## 4.1 配置 ArgoCD 仓库访问权限（可选）

argocd login localhost:8080 --username admin --password=YeGXj5kU5B3aIg5h


```
$ argocd login 127.0.0.1:8080 --insecure    #更换地址
Username: admin
Password:
'admin:login' logged in successfully
```


添加示例应用仓库:
```
~# argocd repo add https://github.com/Hugh-yw/kubernetes-example.git --username $USERNAME  --password $PASSWORD
```

将 $USERNAME 替换为 GitHub 账户 ID，将 $PASSWORD 替换为 GitHub Personal Token，Token创建链接：https://github.com/settings/tokens/new


## 4.2 创建 ArgoCD 应用

ArgoCD 同时支持使用 Helm Chart、Kustomize 和 Manifest 来创建应用，本次实践以示例应用的 Helm Chart 为例。通过argocd app create命令来创建应用:

```sh
# First we need to set the current namespace to argocd running the following command:
kubectl config set-context --current --namespace=argocd

$ argocd app create example --sync-policy automated --repo https://github.com/Hugh-yw/kubernetes-example.git --revision main --path helm --dest-namespace gitops-example --dest-server https://kubernetes.default.svc --sync-option CreateNamespace=true

application 'example' created


# Create the example guestbook application with the following command:
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
```


```
–sync-policy 参数代表设置自动同步策略。automated 的含义是自动同步，也就是说当集群内的资源和 Git 仓库 Helm Chart 定义的资源有差异时，ArgoCD 会自动执行同步操作，实时确保集群资源和 Helm Chart 的一致性。
–repo 参数表示 Helm Chart 的仓库地址。这里的值是示例应用的仓库地址，注意需要替换成你实际的 Git 仓库地址。
–revision 参数表示需要跟踪的分支或者 Tag，这里我们让 ArgoCD 跟踪 main 分支的改动。
–path 参数表示 Helm Chart 的路径。在示例应用中，存放 Helm Chart 的目录是 helm 目录。
–dest-namespace 参数表示命名空间。这里指定了 gitops-example 命名空间，注意，这是一个不存在的命名空间，所以我们额外通过--sync-option参数来让 ArgoCD 自动创建这个命名空间。
最后，–dest-server 参数表示要部署的集群， https://kubernetes.default.svc 表示 ArgoCD 所在的集群。
```



## 4.3 检查 ArgoCD 同步状态

![在这里插入图片描述](https://img-blog.csdnimg.cn/ce4ff4b307a74092afaba1071baa1de2.png)


在应用详情页面，需要重点关注三个状态。

APP HEALTH： 应用整体的健康状态，它包含下面三个值。
    Progressing：处理中
    Healthy：健康状态
    Degraded：宕机
CURRENT SYNC STATUS： 应用定义和集群对象的差异状态，也包含下面三个值。
    Synced：完全同步
    OutOfSync：存在差异
    Unknown：未知
LAST SYNC RESULT： 最后一次同步到 Git 仓库的信息，包括 Commit ID 和提交者信息。


## 4.4 访问应用

当应用健康状态变为 Healthy 之后，我们就可以访问应用了。

```
~# kubectl get ingressroute -n gitops-example 
NAME               AGE
frontend-service   70m
```

访问应用链接：http://frontend.demo.com


## 4.5 连接 GitOps 工作流


在完成 ArgoCD 的应用配置之后，我们就已经将示例应用的 Helm Chart 定义和集群资源关联起来了，但整个 GitOps 工作流还缺少非常重要的一部分，就是上面提到的自动更新 Helm Chart values.yaml 文件镜像版本的部分，在下面这张示意图中用“❌”把这个环节标记了出来。
![在这里插入图片描述](https://img-blog.csdnimg.cn/264a70ff319a4273a5321d8632f5740f.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/264a70ff319a4273a5321d8632f5740f.png)

在这部分工作流没有打通之前，提交的新代码虽然会构建出新的镜像，但是 Helm Chart 定义的镜像版本并不会产生变化， 这会导致 ArgoCD 不能自动更新集群内工作负载的镜像版本。
要解决这个问题，我们还需要在 GitHub Action 中添加自动修改 Helm Chart 并重新推送到仓库操作。
接下来，我们修改示例应用的.github/workflows/build.yaml文件，在“Build frontend and push”阶段后面添加一个新的阶段，代码如下：


```yaml
- name: Update helm values.yaml
  uses: fjogeleit/yaml-update-action@main
  with:
    valueFile: 'helm/values.yaml'
    commitChange: true
    branch: main
    message: 'Update Image Version to ${{ steps.vars.outputs.sha_short }}'
    changes: |
      {
        "backend.tag": "${{ steps.vars.outputs.sha_short }}",
        "frontend.tag": "${{ steps.vars.outputs.sha_short }}"
      }
```

使用了 GitHub Action 中yaml-update-action插件来修改 values.yaml 文件并把它推送到仓库。如果你是使用 GitLab 或者 Tekton 构建镜像，可以调用 jq 命令行工具来修改 YAML 文件，再使用 git 命令行将变更推送到仓库。
到这里， 一个完整的 GitOps 工作流就建立好了。


## 4.6 体验 GitOps 工作流

尝试修改 frontend/src/App.js 文件，例如修改文件第 49 行的“Hi! I am a geekbang”。修改完成后， 将代码推送到 GitHub 仓库 main 分支，此时，GitHub Action 会自动构建镜像，并且还会更新代码仓库中 Helm values.yaml 文件的镜像版本。


这里遇到一点小插曲，在GitHub Action自动构建镜像时报异常：
>HttpError: Resource not accessible by integration

参考解决资料：https://www.drixn.com/3074.html

ArgoCD 默认每 3 分钟会拉取仓库检查是否有新的提交，你也可以在 ArgoCD 控制台手动点击 Sync 按钮来触发同步。

![在这里插入图片描述](https://img-blog.csdnimg.cn/bbb5e8886dc14b3dabefae0dcfb292f6.png)

ArgoCD 同步完成后，我们可以在“LAST SYNC RESULT”一栏中看到 GitHub Action 修改 values.yaml 的提交记录，当应用状态为 Healthy 时，我们就可以访问新的应用版本了。


# 5 例子: 用YAML 文件声明式地创建 Argo CD 应用


我们还可以通过 Argo CD 提供的命令行工具来部署应用，经过上一节的步骤，我们已经登录了 API Server，我们只需要执行下面的 argocd app create 命令即可创建 guestbook 应用：
```
$ argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
application 'guestbook' created
```


也可以使用 YAML 文件声明式地创建 Argo CD 应用：
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```


我们创建一个 YAML 文件，包含以上内容，然后执行 kubectl apply -f 即可，和 Web UI 操作类似，刚创建的应用处于 OutOfSync 状态，我们可以使用 argocd app get 命令查询应用详情进行确认：
```
$ argocd app get guestbook
Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:32130/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from  (53e28ff)
Health Status:      Missing
 
GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  OutOfSync  Missing
apps   Deployment  default    guestbook-ui  OutOfSync  Missing
```


接着我们执行 argocd app sync 命令，手工触发同步：
```
$ argocd app sync guestbook
TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS    HEALTH        HOOK  MESSAGE
2023-05-06T08:22:53+08:00            Service     default          guestbook-ui  OutOfSync  Missing
2023-05-06T08:22:53+08:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing
2023-05-06T08:22:53+08:00            Service     default          guestbook-ui    Synced  Healthy
2023-05-06T08:22:54+08:00            Service     default          guestbook-ui    Synced   Healthy              service/guestbook-ui created
2023-05-06T08:22:54+08:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing              deployment.apps/guestbook-ui created
2023-05-06T08:22:54+08:00   apps  Deployment     default          guestbook-ui    Synced  Progressing              deployment.apps/guestbook-ui created
 
Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:32130/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to  (53e28ff)
Health Status:      Progressing
 
Operation:          Sync
Sync Revision:      53e28ff20cc530b9ada2173fbbd64d48338583ba
Phase:              Succeeded
Start:              2023-05-06 08:22:53 +0800 CST
Finished:           2023-05-06 08:22:54 +0800 CST
Duration:           1s
Message:            successfully synced (all tasks run)
 
GROUP  KIND        NAMESPACE  NAME          STATUS  HEALTH       HOOK  MESSAGE
       Service     default    guestbook-ui  Synced  Healthy            service/guestbook-ui created
```


等待一段时间后，应用关联资源就部署完成了：

```
$ kubectl get all
NAME                               READY   STATUS    RESTARTS   AGE
pod/guestbook-ui-b848d5d9d-rtzwf   1/1     Running   0          67s
 
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/guestbook-ui   ClusterIP   10.107.51.212   <none>        80/TCP    67s
service/kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP   30d
 
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/guestbook-ui   1/1     1            1           67s
 
NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/guestbook-ui-b848d5d9d   1         1         1       67s
```


测试完成后，执行 argocd app delete 命令删除应用，该应用关联的资源将会被级联删除：
```
$ argocd app delete guestbook
Are you sure you want to delete 'guestbook' and all its resources? [y/n] y
application 'guestbook' deleted
```




# 6 Sync (Deploy) The Application


## 6.1 Syncing via CLI[¶](https://argo-cd.readthedocs.io/en/stable/getting_started/#syncing-via-cli "Permanent link")

Once the guestbook application is created, you can now view its status:

```
$ argocd app get guestbook
Name:               guestbook
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://10.97.164.88/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
Sync Policy:        <none>
Sync Status:        OutOfSync from  (1ff8a67)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH
apps   Deployment  default    guestbook-ui  OutOfSync  Missing
       Service     default    guestbook-ui  OutOfSync  Missing
```

The application status is initially in `OutOfSync` state since the application has yet to be deployed, and no Kubernetes resources have been created. To sync (deploy) the application, run:

```
argocd app sync guestbook
```

This command retrieves the manifests from the repository and performs a `kubectl apply` of the manifests. The guestbook app is now running and you can now view its resource components, logs, events, and assessed health status.

## 6.2 Syncing via UI[¶](https://argo-cd.readthedocs.io/en/stable/getting_started/#syncing-via-ui "Permanent link")

![guestbook app](https://argo-cd.readthedocs.io/en/stable/assets/guestbook-app.png) ![view app](https://argo-cd.readthedocs.io/en/stable/assets/guestbook-tree.png)


