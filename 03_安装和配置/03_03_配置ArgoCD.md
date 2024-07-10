
https://github.com/devops-ws/argo-cd-guide

# 1 同步策略


Argo CD 可以指定 Git 仓库中的特定目录，已经一些通用配置：
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample
spec:
  source:
    repoURL: https://github.com/devops-ws/learn-pipeline-go
    targetRevision: HEAD
    path: kustomize                           # 指定父目录
    directory:
      recurse: true                           # 支持遍历子目录
      exclude: '{config.json,env-usw2/*}'     # 忽略部分
      include: '*.yaml'                       # 包含所有 YAML 文件
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```


设置同步策略：
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample
spec:
  syncPolicy:
    syncOptions:
    - CreateNamespace=true      # 自动创建命名空间
    automated:
      prune: true               # Git 库中删除的资源，也会在集群中删除
      selfHeal: true
```



# 2 Git仓库

Argo CD 支持 HTTPS、SSH 协议的 Git 仓库，下面的例子中使用的是 SSH 协议：
```
apiVersion: v1
data:
  insecure: dHJ1ZQ==
  project: ZGVmYXVsdA==
  sshPrivateKey: eW91ci12YWx1ZQ==
  type: Z2l0
  url: Z2l0QGdpdGh1Yi5jb206ZGV2b3BzLXdzL2FyZ28tY2QtZ3VpZGUuZ2l0
kind: Secret
metadata:
  labels:
    argocd.argoproj.io/secret-type: repository # 表明这会被 Argo CD 识别为仓库（Git、Helm 等）
  name: repo-1108037796 # 以 repo- 为前缀的名称，通过 UI 创建时会自动生成
  namespace: argocd
type: Opaque
```


# 3 Helm 仓库
截至 v2.5.2 Argo CD 界面还支持添加 Helm 类型的仓库，可以通过命令行或者 YAML 的方式来添加。

```
apiVersion: v1
data:
  enableOCI: dHJ1ZQ==
  name: c2staGVsbQ==
  project: ZGVmYXVsdA==
  type: aGVsbQ==
  url: cmVnaXN0cnktMS5kb2NrZXIuaW8=
kind: Secret
metadata:
  labels:
    argocd.argoproj.io/secret-type: repository
  name: repo-skywalking-helm
  namespace: argocd
type: Opaque
```

OCI 类型的 Helm 仓库安装示例请查看 examples/skywalking


# 4 配置管理插件


配置管理工具（Config Management Plugin，CMP）使得 Argo CD 可以支持 Helm、Kustomize 以外的（可转化为 Kubernetes 资源）格式。

例如：我们可以将 [GitHub Actions 的配置文件转为 Argo Workflows](https://github.com/LinuxSuRen/github-action-workflow/) 的文件，从而实现在不了解 Argo Workflows 的 `WorkflowTemplate` 写法的前提下，也可以把 Argo Workflows 作为 CI 工具。

> 下面的例子中需要用到 Argo Workflows，请自行安装，或查看[这篇中文教程](https://github.com/LinuxSuRen/argo-workflows-guide)。

我们只需要将插件作为 sidecar 添加到 `argocd-repo-server` 即可。下面是 sidecar 的配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      containers:
      - args:
        - --loglevel
        - debug
        command:
        - /var/run/argocd/argocd-cmp-server
        image: ghcr.io/linuxsuren/github-action-workflow:master
        imagePullPolicy: IfNotPresent
        name: tool
        resources: {}
        securityContext:
          runAsNonRoot: true
          runAsUser: 999
        volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
```

然后，再添加如下 Argo CD Application 后，我们就可以看到已经有多个 Argo Workflows 被创建出来了。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: yaml-readme
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    path: .github/workflows/                            # It will generate multiple Argo CD application manifests 
                                                        # base on YAML files from this directory.
                                                        # Please make sure the path ends with slash.
    plugin: {}                                          # Argo CD will choose the corresponding CMP automatically
    repoURL: https://gitee.com/linuxsuren/yaml-readme   # a sample project for discovering manifests
    targetRevision: HEAD
  syncPolicy:
    automated:
      selfHeal: true
```

由于用到 PVC 作为 Pod 之间的共享存储，我们还需要安装对应的依赖。如果是测试环境，可以安装 [OpenEBS](https://openebs.io/docs/user-guides/installation)。并设置其中的 为[默认存储卷](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/change-default-storage-class/)：

```shell
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

如果需要用到 Git 凭据的话，可以通过下面的命令拿到：

```shell
kubectl create secret generic git-secret --from-file=id_rsa=/root/.ssh/id_rsa --from-file=known_hosts=/root/.ssh/known_hosts --dry-run=client -oyaml
```

这一点对于 Argo Workflows 落地为持续集成（CI）工具时，非常有帮助。如果您觉得 GitHub Actions 的语法足够清晰，那么，可以直接使用上面的插件。或者，您希望能定义出更简单的 YAML，也可以自行实现插件。插件的核心逻辑就是将目标文件（集）转为 Kubernetes 的 YAML 文件，在这里就是 `WorkflowTemplate`。

如果再发散性地思考下，我们也可以通过自定义格式的 YAML（或 JSON 等任意格式）文件转为 Jenkins 可以识别的 Jenkinsfile，或其他持续集成工具的配置文件格式。

# 5 凭据管理


可以通过下面的命令，生成一个加密后的 Secret：

```shell
kubectl create secret generic test --from-literal=username=admin --from-literal=password=admin --dry-run=client -oyaml -n default | kubeseal -oyaml
```

下面是生成 Docker 认证信息的命令：

```shell
kubectl create secret docker-registry harbor --docker-server='10.121.218.184:30002' \
  --docker-username=admin --docker-password=password \
  --dry-run=client -oyaml -n default | kubeseal -oyaml
```

# 6 单点登录


Argo CD [内置了 Dex 服务](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#dex)，我们可以参考如下的配置来对接外部身份认证服务：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://10.121.218.184:31392 # argo-cd server 的地址
  dex.config: |
    logger:
      level: debug
    connectors:
      - type: gitlab
        id: gitlab
        name: GitLab
        config:
          baseURL: http://10.121.218.82:6080
          clientID: b9119ac2313f62625d8b1e9648f7b10b9dad9c5198f19e5df731b09ffa5d008d
          clientSecret: a0c1bef745da758609acceb5beba3c0104f04c3b0a491aee7c7c479ed3e26309
          redirectURI: https://10.121.218.184:31392/api/dex/callback
          groups:
          - dev               # 只允许 dev 用户组
          useLoginAsID: false
```

```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # 只允许 dev 组的用户查看 application
    p, role:org-readonly, applications, get, default/*, allow
    g, dev, role:org-readonly           # 假如用户组名为 dev
  policy.default: role:org-readonly
  scopes: '[groups, email]' 
```

对于通用的 OAuth 认证，可以访问下面地址获取相关信息：

`https://10.121.218.184:31392/api/dex/.well-known/openid-configuration`

# 7 多集群

```
#!title: Create New Cluster
cat <<EOF | kubectl apply -n argocd -f -
apiVersion: v1
data:
  config: eyJ0bHNDbGllbnRDb25maWciOnsiaW5zZWN1cmUiOnRydWUsImNhRGF0YSI6IkxTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENrMUpTVU0xZWtORFFXTXJaMEYzU1VKQlowbENRVVJCVGtKbmEzRm9hMmxIT1hjd1FrRlJjMFpCUkVGV1RWSk5kMFZSV1VSV1VWRkVSWGR3Y21SWFNtd0tZMjAxYkdSSFZucE5RalJZUkZSSmVVMVVTWGRPVkVGNlRXcFJNRTFXYjFoRVZFMTVUVlJKZDAxcVFYcE5hbEV3VFZadmQwWlVSVlJOUWtWSFFURlZSUXBCZUUxTFlUTldhVnBZU25WYVdGSnNZM3BEUTBGVFNYZEVVVmxLUzI5YVNXaDJZMDVCVVVWQ1FsRkJSR2RuUlZCQlJFTkRRVkZ2UTJkblJVSkJUVXBzQ21OdVNHc3pORkVyVUhWdVRFeG9PV1pIZVdoaGVVRXdXbEoyYmxncmNqSkZXa2xZTTJzeWFTOXhSVTExWnpWRlRXMTVWMWxuU0d4YU4xQnZNVWR5UkZBS2VsZFdXbGx4Tm1sblozSmFOWGN3Um1JMWRtVkNjRmhDTUhOWFJVRXpiVVpOUVdFNEszWlhhRmxvTDI5UU1uSldWMEkxZFdwQ2NIa3piMDFNWWpGamNRcGtSWFoxUjBsQ1IxTlFSV0YwVUZsbmJYbHZjbXRHUkZKWksxZFlOVVpyWVUwNU5VUTNWbE14Wm1seE5qWndOR0oyTm5wbFNHSjNUM1prYWk4clZVMWxDbWxTWmpGR01WQXpWSGxqUVcxdFR6Vmthall2UTBWdE5tWkhURXBKYUZWUVNqWldRbE51V0c5MldESmtVekZUZDJ4dU56UlBZbU5PTlVZekt6VllPRmtLVm1wSU9UazJiQzgyWlcxVFIzZGlUMEpFVGxGTVlrUkJVR1JSTm14bU5GUlNPRE5EZEdGdVMzSlFha1l5VGpCV1ZGTTNUV1JoVUhRNVMwWnJjRlZ3YVFwcE9XNXViWFI1VWpWUVdWQXhjREl6VGxORlEwRjNSVUZCWVU1RFRVVkJkMFJuV1VSV1VqQlFRVkZJTDBKQlVVUkJaMHRyVFVFNFIwRXhWV1JGZDBWQ0NpOTNVVVpOUVUxQ1FXWTRkMGhSV1VSV1VqQlBRa0paUlVaRFMxbFhTR3N6VVVGS1ZtMUJiV1E1UzJsQlRsWTBiMDQzYjBkTlFUQkhRMU54UjFOSllqTUtSRkZGUWtOM1ZVRkJORWxDUVZGRGRFbDNjVlJRU0VjeFIxTlBZMGNyYlhrdmJXSlVWM2d3YmpKMFRYaEhUbnBWTjB4RGIzY3JhblJsTm5FdmJuRkVZZ3AxTDNOd1ZXb3JabXRVZGtkWVRuSmpWbTlYVW5sTVZqVlNhbWN5Y0VwTFdYQmpRMlExYXpReGVEVkhXbEJZTUZKbWVXeHFPRVJKYlhOaGNqRXJaRGxhQ20xTE9YRndSM0k0VEVRNGRrdFZUMVJCTW1seU5tUnRVMDlyV1U1WlNHNWtOM015Uml0b1JYY3pRMWc1YXpacGMwaFlSVGxCZW05WWVVODRZelV3Um1zS1NXOHZkMmhRYVhSUVpIQmphbE5WV0hjNFVVUmlTMk5RT0Zwdk4zZGllRTFyTDBsc2NtWmFRVVJ2UVdZMVRERnhNRmRQWW14RFZWSlZlVTVrTTNkd1RncFVlR2QzZDBNM1NVWmhiUzl0SzNSM1MwVnNjVlZOWm1WRFNGVkljemR2Tm1aS2RtdGFZa1JzYVV4eWMyeEJlVmRzUldzMFZVdFFhVkoxWkVOVVVrdE5DamRHVDBKcmMwbEtPRlU1ZHl0TmRsbEpjMGc1TURFNFoyNDNiVU5sZVVKSE1UbHNhQW90TFMwdExVVk9SQ0JEUlZKVVNVWkpRMEZVUlMwdExTMHRDZz09IiwiY2VydERhdGEiOiJMUzB0TFMxQ1JVZEpUaUJEUlZKVVNVWkpRMEZVUlMwdExTMHRDazFKU1VSSlZFTkRRV2R0WjBGM1NVSkJaMGxKWkVwclNISnlibTFvVXpCM1JGRlpTa3R2V2tsb2RtTk9RVkZGVEVKUlFYZEdWRVZVVFVKRlIwRXhWVVVLUVhoTlMyRXpWbWxhV0VwMVdsaFNiR042UVdWR2R6QjVUV3BGZVUxRVZYZE5la2t3VGtSR1lVWjNNSGxOZWtWNVRVUlZkMDE2U1RCT1JGSmhUVVJSZUFwR2VrRldRbWRPVmtKQmIxUkViazQxWXpOU2JHSlVjSFJaV0U0d1dsaEtlazFTYTNkR2QxbEVWbEZSUkVWNFFuSmtWMHBzWTIwMWJHUkhWbnBNVjBackNtSlhiSFZOU1VsQ1NXcEJUa0puYTNGb2EybEhPWGN3UWtGUlJVWkJRVTlEUVZFNFFVMUpTVUpEWjB0RFFWRkZRVFZDWkhVelpFeGlMMngyWjFwNmVUZ0tSVTFCUTBKM1VqVTFaR2RYU2tjMFVHczNTV1J1TVRkQk1Ib3hVbWRJTlhCS1dETjZObFV2ZDJWQ2VrdEtTbXhUYlhaaWF6VjNNV1FyV1VzMk5HcEJhZ3BzTDJsT1NtMXVkVlJzYlZnNVprRjZUbkpLWm5CemMxQnFhbFUyVlVkSmFGVndTa0ZTTkhnNGNpOXpkblY1UldGaWJuQlhlRzR5VmxGeWRtNXJiMG92Q2xwWlpFTklRMWR5Y0Vadk5IZEVhSFJGVUZaSVdrOTRTVUl5VjBOTFRGTnRibnBrZUhRMFJIUkJZbmQxWXpsRGFXUnVPRTFvTW1SM1dYWXdNR3hWTlRRS05Ua3haa05uU21zeU9VUkdWU3MxYzJwUk16QnBSa3RKVVhwbGVEazNkQ3RhVFdsYVpqRndWVzFpVEhBMWVqWXJLMUJ1TUZGNk56QXdZa3RyVGt0ak5BcFhWMUpIY0ZGTGMyUlZiR3gwTDNwYWFuSjZaaTl6Wm5wUmFVbFFielJKV0VvMGVGQlZlbXQyVEhWMVVHVjJVbEpOVlhVclV6VnVaMFZKZWxwamVGVmtDa0pXWXpSRGQwbEVRVkZCUW04eFdYZFdSRUZQUW1kT1ZraFJPRUpCWmpoRlFrRk5RMEpoUVhkRmQxbEVWbEl3YkVKQmQzZERaMWxKUzNkWlFrSlJWVWdLUVhkSmQwUkJXVVJXVWpCVVFWRklMMEpCU1hkQlJFRm1RbWRPVmtoVFRVVkhSRUZYWjBKUmFXMUdhRFZPTUVGRFZscG5TbTVtVTI5blJGWmxTMFJsTmdwQ2FrRk9RbWRyY1docmFVYzVkekJDUVZGelJrRkJUME5CVVVWQmJtZzBSemhJV1d4bGMyazVTR2haZGxReE5ETmxOU3RCY0hSTFNGVjJaMlEyUnpod0NtNHZXSEZ2V0dFdmFHb3diamxZTW1GU1RrSXpNVnBaWVc4ME1qZHpibFZKWVc5UlpXMTZaVWhLZEVaUlRrODRWMU5YYjBRclZsQkVORGhsWVZOMk4yUUtPR3hvTTBkNFNsaFhkVWxpV1ZZMWQyTnBOM2xRYnpsR1dXOTRRMnhNWlROVE5HeFdiVVYwVERkdFZqVlBVbnBxVVU5WlFVMHZkV1pIVVdsdWQyOTRUUW95U1ZKc1ZsVjBjRVZ5VmxSUGMwZDBRMGR3WjJsRGN6Y3hTVUpLVTJnemNUZzNjbXRLYm1abU1XSnlZa1owZHpSdk5GSkhlVFlyTVRGQmVFVjJNak0wQ21wSVR6VjJha3RwTTBJd1RubHNSMlkwWW1WSWVtWkNSVTFSTDNWRVIyZHBibVYzV0hKVFVFWlZRbXRXUmsxVlFWWlhiVmxNZG1WemJXbG1aMWMwTVdVS2NYaFZWa3R6SzJ0RlJHUkhaMGRMUVRCa2VFUkdUbUV5TDFaSWVEQk1jbHBvUVZvNGIwRjFhM2hxYW1kMWRtcFNTMUU5UFFvdExTMHRMVVZPUkNCRFJWSlVTVVpKUTBGVVJTMHRMUzB0Q2c9PSIsImtleURhdGEiOiJMUzB0TFMxQ1JVZEpUaUJTVTBFZ1VGSkpWa0ZVUlNCTFJWa3RMUzB0TFFwTlNVbEZiM2RKUWtGQlMwTkJVVVZCTlVKa2RUTmtUR0l2YkhablducDVPRVZOUVVOQ2QxSTFOV1JuVjBwSE5GQnJOMGxrYmpFM1FUQjZNVkpuU0RWd0NrcFlNM28yVlM5M1pVSjZTMHBLYkZOdGRtSnJOWGN4WkN0WlN6WTBha0ZxYkM5cFRrcHRiblZVYkcxWU9XWkJlazV5U21ad2MzTlFhbXBWTmxWSFNXZ0tWWEJLUVZJMGVEaHlMM04yZFhsRllXSnVjRmQ0YmpKV1VYSjJibXR2U2k5YVdXUkRTRU5YY25CR2J6UjNSR2gwUlZCV1NGcFBlRWxDTWxkRFMweFRiUXB1ZW1SNGREUkVkRUZpZDNWak9VTnBaRzQ0VFdneVpIZFpkakF3YkZVMU5EVTVNV1pEWjBwck1qbEVSbFVyTlhOcVVUTXdhVVpMU1ZGNlpYZzVOM1FyQ2xwTmFWcG1NWEJWYldKTWNEVjZOaXNyVUc0d1VYbzNNREJpUzJ0T1MyTTBWMWRTUjNCUlMzTmtWV3hzZEM5NldtcHllbVl2YzJaNlVXbEpVRzgwU1ZnS1NqUjRVRlY2YTNaTWRYVlFaWFpTVWsxVmRTdFROVzVuUlVsNldtTjRWV1JDVm1NMFEzZEpSRUZSUVVKQmIwbENRVkZEZVRsdlVHaG1iR3h2VTFCU1dRcGlSRzUyVG05blozTXlNV3hZZEdRMGRTOWFlVE42VlVrek5VdFNOamRIZDBSUU9EVkZTWE0xVXpaVFZYbGFiamxtTWs1c0wxVlFOakpsTUdKSlYzazJDbTVwVXk5VU5rTkpVRWRPYW1wRU5rVkdTRkpMWmxWbk5XMVhabUZJTXpscGNFdDNZVzF0TDJVNE5tRTVaVmQxYnl0eU9DOVpZbEZIVWpOREsycExjVlVLZEU1c05tTjBZak5QVFU1cmVpOXNjRTFLZVRaNWIyazJPR1l4VGpsMVlXSlVZVmRFV1hRMk9HZFZSR3h1VkVoNlVuSkhObE5wYkdwaVFscEVOVXc0Y3dwM01WcHBTRVZWVDBnM1pXTktZWHBOZGxRMVUxcHlTV1ZqYW5aNVNrNVhWbGxhTWxRclozTXJWMVpLU1hsV1dHNXljVzUxUm5oUk9HSm1kRWt6UlRoU0NrY3plbmRDU1VaUWJDOW1TRmd2VEVkUWRDczFUMVJMWlhadVlrSjZRVU01YWtrdk1rOTZObFpLWjB0VWNraFJlVkJITXpRMFpWbEJRMmhzVFhRM2QyY0thaXRRV2sxaUwxSkJiMGRDUVU5VU9VVkZWMEo1VGpoQ2JWUk5kSGxGT1hBNFNsUkJWRE5oTVZGQ1JtSkhkR1JCZEZNMVowOWhhbWRuZVRSVldDdE9XUXBaVkdsSFZFVnJWamxUU25abE0wWk5WVUpoWkhkM1RFUlNia3cyVFdOdE1rVklNazFrTTFkeE1DdEVNMlppWVVoa05WQjVWQzlhZVZJMFRUY3hVRGR1Q25oak4yUk1hMDV0ZGpKMmMwTnRSMVpUYVZwd1dGZEpWV2QwUkVKNldIZHNUMm92ZEV4WVJrWk1RVUY1YkZkV1QwNXNUVTB3TUVKS1FXOUhRa0ZRTnk4S1UwVllXVGxsWkVFNWQyMXphVVZZWVU5UlJYUkNUSGxsTW14VFVHOHlSMUJTWlVaclRrVlhVMDVOVERsVmJYUnJaSEZyUlhBNE9FcHJNSGhVVWxZek9BcHhkbWxvUmxJeVpHdHJOakpNZGs4eFozWjBWMkpvVXpoSGJXYzVTMWRKTURKUllXUjJkMVZTU2xGdFJIWkZWa3BtYTJWS1JsRnRTWE5IY1dWak5IbzBDbVZKZEVSRlJUbFJhaTlrVEVGVGRrTXdRbGxpZW5Sc2VrSlNNWFZ3ZFZKdVMyWTBSMnRTTW5wQmIwZEJTazlYYUN0YVJYZEVURGN4VFVsdWRpOU9kbFlLUzBOTVZYRjNkMHBvYzBwdVZVMW1ZMkZrZVZoaVpEWXZVa2N5UlVKa016TjZSMUJZV25VNWFUQkhiVzFQYkhSU2FrWk9abGRPUzJWU01tbEtTRlJrTkFvNFRVaDRabU5TU1RNM1kwSlRjV2RLV0VreGRURlJZMVV2ZEVKc1VXRXlWemhtTkhoMGFYRlpURmwxWWtsS05IUnNTVXhzZVZOblJUZENOVTlRWmt4Q0NuTkJkRWhvZW1wbFNHbHZTV0ZKY0hoMGJrRmhiV2RGUTJkWlFsQm1ObU0wYmpOMVMzSlhXbGhVWTB4MWFFTndhR2N6WlVkc2NpOWhkbE14YVhKU2FFMEtVRTFHVUV3eFdHcDBUR0ZPV2t4VVdqTlBiVEJWYm1ad2R6aDRUV3RSYkRodGFuWk1SQ3RWWkZKUk9DdFRPR2xoVFRCbUsycDFXbk4xVTFNNWVWWjNad3BCS3pSYU1XczJSblZPZUhCbVdrRjRTRlIyUW5wVGRYcEZaRTVWYWl0TFJrTkdUamxhZEVJM1pVcGlWVE5rWldsRVQyeG5VV04yUjJOVmFqSjZTRXczQ2tRelJVOXZVVXRDWjBOWlYxcE1NRVJ4VURWRU5HUk9NMngxVG5OblFreGtUMkppTmpSUmQwVlBUM3BzVGtNeWRuRk5iak5aVFM5UFFqRndUVGNyVFdvS1FVTk1TRVV6VFZGM2NITldiRUV6ZVV0VFdGUklXVlUySzJsa01taElVVE15U2paRVdVaG5SRFIzZFVsQlIyRTFOVEpwVlRoWWFUZGFPVlZwZW1Od1NRcElZazF1TkZndlEweGplV3dyUWxKdmEzaGpXbWw2VERCR1RUZDRkMGwyV1RWUlEzVm9OazV4U0haNlVHVlhaamQ2WTNWcUNpMHRMUzB0UlU1RUlGSlRRU0JRVWtsV1FWUkZJRXRGV1MwdExTMHRDZz09In19
  name: ZGV2
  server: aHR0cHM6Ly8xOTIuMTY4LjEyMy4yNDU6NjQ0Mw==
kind: Secret
metadata:
  annotations:
    managed-by: argocd.argoproj.io
  labels:
    argocd.argoproj.io/secret-type: cluster
  name: cluster-dev
  namespace: argocd
type: Opaque
EOF


```



```
{
  "username": "",
  "password": "",
  "bearerToken": "token", # 必须字段
  "tlsClientConfig": {
    "insecure": true,
    "caData": "sample",
    "certData": "sample",
    "keyData": "sample"
  }
}
```


```
cat <<EOF | kubectl apply -n argocd -f -
apiVersion: v1
kind: Secret
metadata:
  name: cluster-dev
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: dev
  server: https://192.168.123.245:6443
  config: |
    {
      "bearerToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6ImdwQ20yZjZlTzRKeGdNNVRTUlRKak5DWTlFUVpDT09sSTBEWHBPckNUUkEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJjbHVzdGVyLXRva2VuLWxzN2NkIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImNsdXN0ZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2ZGZiMjVhOC1iZDU4LTQ2YzQtOWUwMy00NmExZDViZDBhYmMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06Y2x1c3RlciJ9.wDmF85TPCLCtBvj7OQDNP6ZXPuS96CI0ajB2iRKTMP9yTy5xqcffWko4ajEYknha5hr4u1kj-PaVS92dAmeG7h8dibgBALPL3uwACC1aZ7OZbWDVXHBW8ttBP-VYfMC07K0GPbUgaAEE8OdZ4HEtntpHOshiJqIVRBkT4QrGmjdGmbuJ3HAq6SQL4n0gQ09C5lKzk3e1VTQOJtG25mqj0-CSmlydDsM49AUTD3Sd2Bz6Fdy9uFjufwxLySSnNka4mkHWzdsrQ3istYSJaZZtQwnY4C4ZRqyV-hqMDOl8jQND5HivjGez-DgdoffrW_tK6kCZAmIl9gal44yhE7hgaA",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1USXdOVEF6TWpRME1Wb1hEVE15TVRJd01qQXpNalEwTVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTUpsCmNuSGszNFErUHVuTExoOWZHeWhheUEwWlJ2blgrcjJFWklYM2syaS9xRU11ZzVFTW15V1lnSGxaN1BvMUdyRFAKeldWWllxNmlnZ3JaNXcwRmI1dmVCcFhCMHNXRUEzbUZNQWE4K3ZXaFloL29QMnJWV0I1dWpCcHkzb01MYjFjcQpkRXZ1R0lCR1NQRWF0UFlnbXlvcmtGRFJZK1dYNUZrYU05NUQ3VlMxZmlxNjZwNGJ2NnplSGJ3T3Zkai8rVU1lCmlSZjFGMVAzVHljQW1tTzVkajYvQ0VtNmZHTEpJaFVQSjZWQlNuWG92WDJkUzFTd2xuNzRPYmNONUYzKzVYOFkKVmpIOTk2bC82ZW1TR3diT0JETlFMYkRBUGRRNmxmNFRSODNDdGFuS3JQakYyTjBWVFM3TWRhUHQ5S0ZrcFVwaQppOW5ubXR5UjVQWVAxcDIzTlNFQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZDS1lXSGszUUFKVm1BbWQ5S2lBTlY0b043b0dNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFDdEl3cVRQSEcxR1NPY0crbXkvbWJUV3gwbjJ0TXhHTnpVN0xDb3cranRlNnEvbnFEYgp1L3NwVWorZmtUdkdYTnJjVm9XUnlMVjVSamcycEpLWXBjQ2Q1azQxeDVHWlBYMFJmeWxqOERJbXNhcjErZDlaCm1LOXFwR3I4TEQ4dktVT1RBMmlyNmRtU09rWU5ZSG5kN3MyRitoRXczQ1g5azZpc0hYRTlBem9YeU84YzUwRmsKSW8vd2hQaXRQZHBjalNVWHc4UURiS2NQOFpvN3dieE1rL0lscmZaQURvQWY1TDFxMFdPYmxDVVJVeU5kM3dwTgpUeGd3d0M3SUZhbS9tK3R3S0VscVVNZmVDSFVIczdvNmZKdmtaYkRsaUxyc2xBeVdsRWs0VUtQaVJ1ZENUUktNCjdGT0Jrc0lKOFU5dytNdllJc0g5MDE4Z243bUNleUJHMTlsaAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg=="
      }
    }
EOF
```


详情请查看 [https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)

# 8 自动更新镜像


借助 [argocd-image-updater](https://github.com/argoproj-labs/argocd-image-updater) 可以自动更新应用的镜像，参考配置如下：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: server
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: 10.121.218.184:30002/al-cloud/console:master
    argocd-image-updater.argoproj.io/update-strategy: digest
```

# 9 通知

Argo CD 内置了[消息通知模块](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/)，只要添加下面的 ConfigMap 即可，这里以发送消息到钉钉为例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  subscriptions: |
    - recipients:
      - status
      triggers:
      - sync-operation-change
  defaultTriggers: |
    - sync-operation-change
  trigger.sync-operation-change: |
    - when: app.status.operationState.phase in ['Succeeded', 'Running', 'Error', 'Failed']
      oncePer: app.status.sync.revision
      send: [status]
  template.status: |
    webhook:
      status:
        method: POST
        path: /send?access_token=token
        body: |
          {"msgtype": "text", "text": {"content": "Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}."}}
    message: |
      Application {{.app.metadata.name}} sync is {{.app.status.sync.status}}.
      Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
  service.webhook.status: |
    url: https://oapi.dingtalk.com/robot
    headers:
    - name: Content-Type
      value: application/json
```



# 4 备份与恢复

官方的方法简单粗暴，使用会直接报错；需要给出argoCD部署所在集群的config文件和所在的命名空间。导出的内容为所有资源信息的`yaml 资源配置清单`。

```bash
# 备份    export
# 恢复    import

argocd admin export --kubeconfig .kube/config -n argo > argocd_$(date +%F).conf_bak

# 恢复
argocd admin import --kubeconfig .kube/config -n argo < argocd_$(date +%F).conf_bak
```

# 5 用户管理

## 5.1 查看用户

```bash
[root@node1 ~]# argocd account list
NAME      ENABLED  CAPABILITIES
admin     true     login
dev-user  true     login
pre-user  true     login
system    true     login

# login 允许从UI登录
```

## 5.2 添加用户

在argoCD的命名空间里面直接编辑cm。

```bash
[root@node1 ~]# kubectl get cm -n argo
NAME                        DATA   AGE
argocd-cm                   7      7d21h    # 对用户的创建和删除
argocd-gpg-keys-cm          0      7d21h
argocd-notifications-cm     1      7d21h
argocd-rbac-cm              2      7d21h    # 用户对资源的鉴权
argocd-ssh-known-hosts-cm   1      7d21h
argocd-tls-certs-cm         0      7d21h
kube-root-ca.crt            1      7d21h
```

`argocd-cm`包含对admin的管理和普通用户的创建。下面的内容一共创建了三个`accounts`。非admin用户即使在rbac中全部放行了全部`resource`的权限但是还是无法操作全部的资源。

```bash
[root@node1 ~]# kubectl -n argo edit cm argocd-cm 
apiVersion: v1
data:
  accounts.dev-user: login
  accounts.pre-user: login
  accounts.system: login
  admin.enabled: "false"
  application.instanceLabelKey: argocd.argoproj.io/instance
  exec.enabled: "false"     # 命令终端
  server.rbac.log.enforce.enable: "false"
  
admin.enabled: "false"      # 是否启用admin用户
accounts.pre-user: login    # 新建pre-user的用户，允许从UI登录
```

## 5.3 用户的鉴权

参考链接：[https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)

目前现有环境中有三个用户`admin`用户在环境配置好了之后直接禁用掉，然后运维人员使用`system`用户来操作集群的资源，`pre-user`限制特定的资源来给用户使用。

```bash
[root@node1 ~]# argocd account list
NAME      ENABLED  CAPABILITIES
admin     true     login
pre-user  true     login
system    true     login
```

```bash
[root@node1 ~]# kubectl -n argo edit cm argocd-rbac-cm
apiVersion: v1
data:
  policy.csv: |
    p, system, *, *, */*, allow
    p, pre-user, *, *, argo-ng-kboss/*, allow
    p, pre-user, clusters, *, argo-ng-kboss/*, deny
    
p           # policy
dev-user    # 用户
*           # 集群
*           # 动作
*/*         # 项目/app
allow       # 允许 拒绝/deny

# 如果用户没有在组中或者是关联特定的role，默认使用这条策略，对资源只有只读的权限且无法修改资源。
policy.default: role:readonly

# 此策略展示了test role对test 环境的应用权限。
p, role:test-admins, applications, create, test-*/*, allow
p, role:test-admins, applications, delete, test-*/*, allow
p, role:test-admins, applications, get, test-*/*, allow
p, role:test-admins, applications, override, test-*/*, allow
p, role:test-admins, applications, sync, test-*/*, allow
p, role:test-admins, applications, update, test-*/*, allow
p, role:test-admins, logs, get, test-*/*, allow
p, role:test-admins, exec, create, test-*/*, allow
p, role:test-admins, projects, get, test-*, allow
g, test-admin, role:test-admins

# g test-admin用户 绑定test-admins的role。
```

里面的`clusters`字段指对argoCD中集群功能的管理，无法做到用户对单一集群的鉴权。`*` 表示支持正则匹配。

资源：`clusters``projects``applications``repositories``certificates``accounts``gpgkeys``logs``exec`

动作：`get``create``update``delete``sync``override``action/<group/kind/action-name>`

## 5.4 修改用户密码

默认输入admin用户的秘密码就可以更改其它用的权限配置。根据交互的提示信息输入即可。

```bash
# argocd account update-password --account dev-user

--account   # 指定用户，默认是修改当前登录用户

system
system.1

pre-user
pre-user

dev-user
dev-user
```


# 6 argoCD 时间问题

[官方文档]([模板 - Argo CD - 声明性 GitOps CD 用于 Kubernetes (argo-cd.readthedocs.io)](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/templates/#change-the-timezone))

需要给每个pod的控制器添加环境变量，需要修改deployment 和 sts 控制器的env

```bash
# kubectl get all -n argo
deployment.apps/argo-cd-argocd-applicationset-controller
deployment.apps/argo-cd-argocd-dex-server
deployment.apps/argo-cd-argocd-notifications-controller
deployment.apps/argo-cd-argocd-redis
deployment.apps/argo-cd-argocd-repo-server
deployment.apps/argo-cd-argocd-server

statefulset.apps/argo-cd-argocd-application-controller

# 添加容器的环境变量
env:
- name: LC_TIME
  value: POSIX
- name: TZ
  value: Asia/Shanghai
```

# 7 打开终端功能

```bash
# kubectl edit cm -n argo argocd-cm
exec.enabled "true"

# kubectl edit clusterrole argo-cd-argocd-server
--------参考/添加到末尾-------
- apiGroups:
  - ""
  resources:
  - pods/exec
  verbs:
  - create
  
# 添加 RBAC 规则以允许用户访问资源，即createexec
p, role:myrole, exec, create, */*, allow
```

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155630264-747412506.png)

# 8 远程仓库管理

argoCD 的远程仓库，如果创建了同名的远程仓库会出现无法删除的情况，仓库配置信息是存储在部署的命名空间下使用`secret`来存储的。手动的在UI删除会直接报错。需要手动的在k8s集群中删除。

![](https://img2022.cnblogs.com/blog/1606824/202211/1606824-20221106155638952-1757746465.png)

```bash
# kubectl get secrets -n argo
repo-1884816892  
repo-3508549329  
repo-507453937   
repo-533094657   

# kubectl get secrets repo-1015242351 -n argo -o yaml
apiVersion: v1
data:
  name: YXJnby10ZXN0
  project: YXJnby10ZXN0

# 通过base64 -d 解码来查看原始的内容
# echo YXJnby10ZXN0 | base64 -d
argo-test

# 删除远程仓库
# kubectl delete secrets repo-1015242351 -n argo
```

# 9 GitlabCI 实现对argoCD模板的更新.

通过在CI中运行stage运行脚本的方式，就可以对app的模板进行更新，而不用修改模板本身，这样实现了统一的部署模板。

```bash
#!/bin/bash

INSTANCE_ID=`uuid | base64 | cut -c1-21`
TIME_STAMP=`date +%s`

# 生成修改的变量,前端的加上front的路径前缀。
APP_NAME=front-${CI_PROJECT_NAME}

argocd login 192.168.21.101:30594 --username cisystem --password $GIT_CD_PASSWORD --insecure

case $ENVIRONMENT in
pre)
    argocd app set pre-$APP_NAME --helm-set envs.default.BUILD_AT=$TIME_STAMP \
--helm-set image.branch=$CI_COMMIT_BRANCH \
--helm-set envs.default.COMMIT_ID=$CI_COMMIT_SHA \
--helm-set image.tag=$CI_PIPELINE_ID ;;
dev)
    argocd app set dev-$APP_NAME --helm-set envs.default.BUILD_AT=$TIME_STAMP \
--helm-set image.branch=$CI_COMMIT_BRANCH \
--helm-set envs.default.COMMIT_ID=$CI_COMMIT_SHA \
--helm-set image.tag=$CI_PIPELINE_ID ;;
test)
    argocd app set test-$APP_NAME --helm-set envs.default.BUILD_AT=$TIME_STAMP \
--helm-set image.branch=$CI_COMMIT_BRANCH \
--helm-set envs.default.COMMIT_ID=$CI_COMMIT_SHA \
--helm-set image.tag=$CI_PIPELINE_ID ;;
*)
    echo -e "\033[31m 请给变量ENVT赋值 pre dev test !!! 对应要发布到的环境中 master --> pre \033[0m" 
    argocd logout 192.168.21.101:30594
    exit 1 ;;
esac

# 退出登录
argocd logout 192.168.21.101:30594

echo -e "\033[31m BUILD_AT:    ---------------------------------------------------------- $TIME_STAMP \033[0m"
echo -e "\033[31m COMMIT_ID:    --------------------------------------------------------- ${CI_COMMIT_SHA} \033[0m"
echo -e "\033[31m CI_PROJECT_NAME:    --------------------------------------------------- $CI_PROJECT_NAME \033[0m"
echo -e "\033[31m CI_PROJECT_PATH:    --------------------------------------------------- $CI_PROJECT_PATH \033[0m"
echo -e "\033[31m CI_PROJECT_NAMESPACE:    ---------------------------------------------- $CI_PROJECT_NAMESPACE \033[0m"
echo -e "\033[31m CI_PIPELINE_ID:     --------------------------------------------------- $CI_PIPELINE_ID \033[0m"
```

# 10 模板中生成随机数的问题

[https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#random-data](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#random-data)

