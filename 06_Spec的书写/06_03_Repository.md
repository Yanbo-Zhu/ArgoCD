

https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#repositories

Repository details are stored in secrets. To configure a repo, create a secret which contains repository details. Consider using [bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) to store an encrypted secret definition as a Kubernetes manifest. 

Each repository must have a `url` field and, depending on whether you connect using HTTPS, SSH, or GitHub App, `username` and `password` (for HTTPS), `sshPrivateKey` (for SSH), or `githubAppPrivateKey` (for GitHub App).


Example for HTTPS:
```
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/argoproj/private-repo
  password: my-password
  username: my-username

```

Example for SSH:
```
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:argoproj/my-private-repository.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----

```


# 1 Helm Chart Repositories

Non standard Helm Chart repositories have to be registered explicitly. Each repository must have `url`, `type` and `name` fields. For private Helm repos you may need to configure access credentials and HTTPS settings using `username`, `password`, `tlsClientCertData` and `tlsClientCertKey` fields.


```
apiVersion: v1
kind: Secret
metadata:
  name: istio
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  name: istio.io
  url: https://storage.googleapis.com/istio-prerelease/daily-build/master-latest-daily/charts
  type: helm
---
apiVersion: v1
kind: Secret
metadata:
  name: argo-helm
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  name: argo
  url: https://argoproj.github.io/argo-helm
  type: helm
  username: my-username
  password: my-password
  tlsClientCertData: ...
  tlsClientCertKey: ...

```





