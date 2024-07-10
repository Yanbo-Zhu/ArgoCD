
我们也可以使用命令行客户端来访问 Argo CD，首先使用 curl 命令下载：


1 
使用 argocd version 查看 Argo CD 的版本信息，验证安装是否成功：
```
$ argocd version
argocd: v2.7.1+5e54351
  BuildDate: 2023-05-02T16:54:25Z
  GitCommit: 5e543518dbdb5384fa61c938ce3e045b4c5be325
  GitTreeState: clean
  GoVersion: go1.19.8
  Compiler: gc
  Platform: linux/amd64
FATA[0000] Argo CD server address unspecified
```


2
由于此时还没有配置服务端，所以 argocd version 只显示了 Argo CD 客户端的版本信息，没有服务端的版本信息。通过上一节的方法将 API Server 暴露之后，就可以使用 argocd login 连接服务端
```
$ argocd login localhost:32130
WARNING: server certificate had error: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password:
'admin:login' logged in successfully
Context 'localhost:32130' updated
```


登录成功后，再次查看版本：
```
$ argocd version
argocd: v2.7.1+5e54351
  BuildDate: 2023-05-02T16:54:25Z
  GitCommit: 5e543518dbdb5384fa61c938ce3e045b4c5be325
  GitTreeState: clean
  GoVersion: go1.19.8
  Compiler: gc
  Platform: linux/amd64
argocd-server: v2.7.1+5e54351.dirty
  BuildDate: 2023-05-02T16:35:40Z
  GitCommit: 5e543518dbdb5384fa61c938ce3e045b4c5be325
  GitTreeState: dirty
  GoVersion: go1.19.6
  Compiler: gc
  Platform: linux/amd64
  Kustomize Version: v5.0.1 2023-03-14T01:32:48Z
  Helm Version: v3.11.2+g912ebc1
  Kubectl Version: v0.24.2
  Jsonnet Version: v0.19.1
```
