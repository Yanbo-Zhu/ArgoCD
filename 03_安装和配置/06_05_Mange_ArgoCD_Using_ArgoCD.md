

# 1 她告诉 argocd 本身, 哪里能找到 spec of application or spec of xx 的位置 

https://git.ivu-ag.com/projects/SYSPLAT/repos/e20-main/browse/argocd/argocd-config.yaml

她告诉 argoCD 本身, 哪里能找到 spec of application or spec of xx 的位置 

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-config
  labels:
    app.kubernetes.io/instance: argocd
spec:
  project: default
  source:
    repoURL: ssh://git@git.ivu-ag.com:7999/sysplat/e20-main.git
    targetRevision: main
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true

```


# 2 ArgoCD_instance的manifest

https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#manage-argo-cd-using-argo-cd
Argo CD is able to install and manage itself since all settings are represented by Kubernetes manifests. 


## 2.1 Application manifest 

The live example of self managed Argo CD config is available at [https://cd.apps.argoproj.io](https://cd.apps.argoproj.io) 

The suggested way is to create [Kustomize](https://github.com/kubernetes-sigs/kustomize) based application which uses base Argo CD manifests from [https://github.com/argoproj/argo-cd](https://github.com/argoproj/argo-cd/tree/stable/manifests) and apply required changes on top.

The live example of self managed Argo CD config is available at [https://cd.apps.argoproj.io](https://cd.apps.argoproj.io) and with configuration stored at [argoproj/argoproj-deployments](https://github.com/argoproj/argoproj-deployments/tree/master/argocd).


1 
使用 kustomize 去 deployen Argo CD manifests 
https://github.com/argoproj/argoproj-deployments/tree/master/argocd

Example of `kustomization.yaml`:

```
# additional resources like ingress rules, cluster and repository secrets.
resources:
- github.com/argoproj/argo-cd//manifests/cluster-install?ref=stable
- clusters-secrets.yaml
- repos-secrets.yaml

# changes to config maps
patches:
- path: overlays/argo-cd-cm.yaml
```


argoproj-deployments/argocd/kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd

resources:
- base/argo-cd-issuer.yaml  # 额外的配置
- base/argo-cd-certificate.yaml
- base/argo-cd-ui-ingress.yaml
- base/rollouts-extension.yaml
- https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/ha/install.yaml   # 这里是用到的 Argo CD Installation Manifests

components:
- https://github.com/argoproj-labs/argocd-extensions/manifests

patches:
- path: overlays/production/argo-cd-cm.yaml
- path: overlays/production/argocd-server-service.yaml
- path: overlays/production/argocd-notifications-controller-deploy.yaml
- path: overlays/production/argocd-notifications-cm.yaml
- path: overlays/production/argocd-cmd-params-cm.yaml
- path: overlays/production/argocd-rbac-cm.yaml
- path: https://raw.githubusercontent.com/argoproj/argo-cd/master/notifications_catalog/install.yaml

images:
- name: quay.io/argoproj/argocd
  newName: ghcr.io/argoproj/argo-cd/argocd
  newTag: 2.13.0-ea725a9c
```



2  Argo CD Installation Manifests
用这个Manifest就可以 install argocd 

https://github.com/argoproj/argo-cd/tree/stable/manifests
里面的 install.yaml 最重要 




## 2.2 IVU的例子

1 
Application spec 
https://git.ivu-ag.com/projects/SYSPLAT/repos/e20-main/browse/argocd/argocd-app.yaml
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-app
  labels:
    app.kubernetes.io/instance: argocd
spec:
  project: default
  source:
    repoURL: ssh://git@git.ivu-ag.com:7999/ptpdeli/k8s-core-components.git
    targetRevision: main
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```


2 
manifest to depoly this argocd instance 
https://git.ivu-ag.com/projects/PTPDELI/repos/k8s-core-components/browse/argocd/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd

generatorOptions:
  disableNameSuffixHash: true
  labels:
    grafana_dashboard: "1"

resources:
  - https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.12/manifests/install.yaml
  - servicemonitors/argocd-applicationset-controller-metrics-smon.yaml
  - servicemonitors/argocd-metrics-smon.yaml
  - servicemonitors/argocd-notifications-controller-smon.yaml
  - servicemonitors/argocd-redis-haproxy-metrics-smon.yaml
  - servicemonitors/argocd-repo-server-metrics-smon.yaml
  - servicemonitors/argocd-server-metrics-smon.yaml

patches:
  - path: argocd-ssh-known-hosts.yaml
  - path: argocd-tls-certs.yaml
  - path: argocd-cm.yaml

configMapGenerator:
  - files:
      - generator/argocd-dashboard.json
    name: argocd-dashboard
```





