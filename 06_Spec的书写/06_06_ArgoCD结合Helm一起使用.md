
https://argo-cd.readthedocs.io/en/stable/user-guide/helm/
# 1 example 

Order of precedence is `parameters > valuesObject > values > valueFiles > helm repository values.yaml`


The lifecycle of the application is handled by Argo CD instead of Helm. Here is an example: 
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  project: default
  source:
    chart: sealed-secrets
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    targetRevision: 1.16.1
    helm:
      releaseName: sealed-secrets
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kubeseal
```



Another example using a public OCI helm chart:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
spec:
  project: default
  source:
    chart: nginx
    repoURL: registry-1.docker.io/bitnamicharts  # note: the oci:// syntax is not included.
    targetRevision: 15.9.0
  destination:
    name: "in-cluster"
    namespace: nginx
```




# 2 Values Files

Helm has the ability to use a different, or even multiple "values.yaml" files to derive its parameters from. Alternate or multiple values file(s), can be specified using the `--values` flag. The flag can be repeated to support multiple values files:

```
argocd app set helm-guestbook --values values-production.yaml
```


In the declarative syntax:

```
source:
  helm:
    valueFiles:
    - values-production.yaml
```




# 3 Values[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#values "Permanent link")

Argo CD supports the equivalent of a values file directly in the Application manifest using the `source.helm.valuesObject` key.

```
source:
  helm:
    valuesObject:
      ingress:
        enabled: true
        path: /
        hosts:
          - mydomain.example.com
        annotations:
          kubernetes.io/ingress.class: nginx
          kubernetes.io/tls-acme: "true"
        labels: {}
        tls:
          - secretName: mydomain-tls
            hosts:
              - mydomain.example.com
```

Alternatively, values can be passed in as a string using the `source.helm.values` key.

```
source:
  helm:
    values: |
      ingress:
        enabled: true
        path: /
        hosts:
          - mydomain.example.com
        annotations:
          kubernetes.io/ingress.class: nginx
          kubernetes.io/tls-acme: "true"
        labels: {}
        tls:
          - secretName: mydomain-tls
            hosts:
              - mydomain.example.com
```


# 4 Helm Parameters


Helm has the ability to set parameter values, which override any values in a `values.yaml`. For example, `service.type` is a common parameter which is exposed in a Helm chart:

```
helm template . --set service.type=LoadBalancer
```

Similarly, Argo CD can override values in the `values.yaml` parameters using `argocd app set` command, in the form of `-p PARAM=VALUE`. For example:

```
argocd app set helm-guestbook -p service.type=LoadBalancer
```

In the declarative syntax:

```
source:
  helm:
    parameters:
    - name: "service.type"
      value: LoadBalancer
```



# 5 Helm Value Precedence

Values injections have the following order of precedence `parameters > valuesObject > values > valueFiles > helm repository values.yaml` Or rather

```
    lowest  -> valueFiles
            -> values
            -> valuesObject
    highest -> parameters
```

so values/valuesObject trumps valueFiles, and parameters trump both.

Precedence of valueFiles themselves is the order they are defined in

```
if we have

valueFiles:
  - values-file-2.yaml
  - values-file-1.yaml

the last values-file i.e. values-file-1.yaml will trump the first
```

When multiple of the same key are found the last one wins i.e

```
e.g. if we only have values-file-1.yaml and it contains

param1: value1
param1: value3000

we get param1=value3000
```

```
parameters:
  - name: "param1"
    value: value2
  - name: "param1"
    value: value1

the result will be param1=value1
```

```
values: |
  param1: value2
  param1: value5

the result will be param1=value5
```

> The list of parameters seen in the ui is not what is used for resources, rather it is the values/valuesObject merged with parameters (see [this issue](https://github.com/argoproj/argo-cd/issues/9213) incase it has been resolved) As a workaround using parameters instead of values/valuesObject will provide a better overview of what will be used for resources



# 6 Helm Release Name[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-release-name "Permanent link")

By default, the Helm release name is equal to the Application name to which it belongs. Sometimes, especially on a centralised Argo CD, you may want to override that name, and it is possible with the `release-name` flag on the cli:

```
argocd app set helm-guestbook --release-name myRelease
```

or using the releaseName for yaml:

```
source:
    helm:
      releaseName: myRelease
```



# 7 Helm Hooks


Helm hooks are similar to [Argo CD hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/). In Helm, a hook is any normal Kubernetes resource annotated with the `helm.sh/hook` annotation.

Argo CD supports many (most?) Helm hooks by mapping the Helm annotations onto Argo CD's own hook annotations:

|Helm Annotation|Notes|
|---|---|
|`helm.sh/hook: crd-install`|Supported as equivalent to `argocd.argoproj.io/hook: PreSync`.|
|`helm.sh/hook: pre-delete`|Not supported. In Helm stable there are 3 cases used to clean up CRDs and 3 to clean-up jobs.|
|`helm.sh/hook: pre-rollback`|Not supported. Never used in Helm stable.|
|`helm.sh/hook: pre-install`|Supported as equivalent to `argocd.argoproj.io/hook: PreSync`.|
|`helm.sh/hook: pre-upgrade`|Supported as equivalent to `argocd.argoproj.io/hook: PreSync`.|
|`helm.sh/hook: post-upgrade`|Supported as equivalent to `argocd.argoproj.io/hook: PostSync`.|
|`helm.sh/hook: post-install`|Supported as equivalent to `argocd.argoproj.io/hook: PostSync`.|
|`helm.sh/hook: post-delete`|Supported as equivalent to `argocd.argoproj.io/hook: PostDelete`.|
|`helm.sh/hook: post-rollback`|Not supported. Never used in Helm stable.|
|`helm.sh/hook: test-success`|Not supported. No equivalent in Argo CD.|
|`helm.sh/hook: test-failure`|Not supported. No equivalent in Argo CD.|
|`helm.sh/hook-delete-policy`|Supported. See also `argocd.argoproj.io/hook-delete-policy`).|
|`helm.sh/hook-delete-timeout`|Not supported. Never used in Helm stable|
|`helm.sh/hook-weight`|Supported as equivalent to `argocd.argoproj.io/sync-wave`.|
|`helm.sh/resource-policy: keep`|Supported as equivalent to `argocd.argoproj.io/sync-options: Delete=false`.|

Unsupported hooks are ignored. In Argo CD, hooks are created by using `kubectl apply`, rather than `kubectl create`. This means that if the hook is named and already exists, it will not change unless you have annotated it with `before-hook-creation`.


Helm hooks + ArgoCD hooks:
If you define any Argo CD hooks, _all_ Helm hooks will be ignored.


'install' vs 'upgrade' vs 'sync': 
Argo CD cannot know if it is running a first-time "install" or an "upgrade" - every operation is a "sync'. This means that, by default, apps that have `pre-install` and `pre-upgrade` will have those hooks run at the same time.


## 7.1 Hook Tips[

- Make your hook idempotent.
- Annotate `crd-install` with `hook-weight: "-2"` to make sure it runs to success before any install or upgrade hooks.
- Annotate `pre-install` and `post-install` with `hook-weight: "-1"`. This will make sure it runs to success before any upgrade hooks.
- Annotate `pre-upgrade` and `post-upgrade` with `hook-delete-policy: before-hook-creation` to make sure it runs on every sync.

Read more about [Argo hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/) and [Helm hooks](https://helm.sh/docs/topics/charts_hooks/).


# 8 Random Data[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#random-data "Permanent link")

Helm templating has the ability to generate random data during chart rendering via the `randAlphaNum` function. Many helm charts from the [charts repository](https://github.com/helm/charts) make use of this feature. 

For example, the following is the secret for the [redis helm chart](https://github.com/helm/charts/blob/master/stable/redis/templates/secret.yaml):

```
data:
  {{- if .Values.password }}
  redis-password: {{ .Values.password | b64enc | quote }}
  {{- else }}
  redis-password: {{ randAlphaNum 10 | b64enc | quote }}
  {{- end }}
```


采用随机生成的 password 会造成 每次 git comprision 比较的不相等 
The Argo CD application controller periodically compares Git state against the live state, running the `helm template <CHART>` command to generate the helm manifests. Because the random value is regenerated every time the comparison is made, any application which makes use of the `randAlphaNum` function will always be in an `OutOfSync` state. This can be mitigated by explicitly setting a value in the values.yaml or using `argocd app set` command to overide the value such that the value is stable between each comparison. For example:

```
argocd app set redis -p password=abc123
```


# 9 Build Environment[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#build-environment "Permanent link")

Helm apps have access to the [standard build environment](https://argo-cd.readthedocs.io/en/stable/user-guide/build-environment/) via substitution as parameters.

E.g. via the CLI:
```
argocd app create APPNAME \
  --helm-set-string 'app=${ARGOCD_APP_NAME}'
```

Or via declarative syntax:
```
  spec:
    source:
      helm:
        parameters:
        - name: app
          value: $ARGOCD_APP_NAME
```


It's also possible to use build environment variables for the Helm values file path:
```
  spec:
    source:
      helm:
        valueFiles:
        - values.yaml
        - myprotocol://somepath/$ARGOCD_APP_NAME/$ARGOCD_APP_REVISION
```



# 10 Helm plugins[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-plugins "Permanent link")

Argo CD is un-opinionated on what cloud provider you use and what kind of Helm plugins you are using, that's why there are no plugins delivered with the ArgoCD image.

But sometimes you want to use a custom plugin. Perhaps you would like to use Google Cloud Storage or Amazon S3 storage to save the Helm charts, for example: https://github.com/hayorov/helm-gcs where you can use `gs://` protocol for Helm chart repository access. There are two ways to install custom plugins; you can modify the ArgoCD container image, or you can use a Kubernetes `initContainer`.


## 10.1 Modifying the ArgoCD container image[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#modifying-the-argocd-container-image "Permanent link")

One way to use this plugin is to prepare your own ArgoCD image where it is included.

Example `Dockerfile`:

```
FROM argoproj/argocd:v1.5.7

USER root
RUN apt-get update && \
    apt-get install -y \
        curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER argocd

ARG GCS_PLUGIN_VERSION="0.3.5"
ARG GCS_PLUGIN_REPO="https://github.com/hayorov/helm-gcs.git"

RUN helm plugin install ${GCS_PLUGIN_REPO} --version ${GCS_PLUGIN_VERSION}

ENV HELM_PLUGINS="/home/argocd/.local/share/helm/plugins/"
```

You have to remember about `HELM_PLUGINS` environment property - this is required for plugins to work correctly.

After that you have to use your custom image for ArgoCD installation.

## 10.2 Using `initContainers`[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#using-initcontainers "Permanent link")

Another option is to install Helm plugins via Kubernetes `initContainers`. Some users find this pattern preferable to maintaining their own version of the ArgoCD container image.

Below is an example of how to add Helm plugins when installing ArgoCD with the [official ArgoCD helm chart](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd):

```
repoServer:
  volumes:
    - name: gcp-credentials
      secret:
        secretName: my-gcp-credentials
  volumeMounts:
    - name: gcp-credentials
      mountPath: /gcp
  env:
    - name: HELM_CACHE_HOME
      value: /helm-working-dir
    - name: HELM_CONFIG_HOME
      value: /helm-working-dir
    - name: HELM_DATA_HOME
      value: /helm-working-dir
  initContainers:
    - name: helm-gcp-authentication
      image: alpine/helm:3.8.1
      volumeMounts:
        - name: helm-working-dir
          mountPath: /helm-working-dir
        - name: gcp-credentials
          mountPath: /gcp
      env:
        - name: HELM_CACHE_HOME
          value: /helm-working-dir
        - name: HELM_CONFIG_HOME
          value: /helm-working-dir
        - name: HELM_DATA_HOME
          value: /helm-working-dir
      command: [ "/bin/sh", "-c" ]
      args:
        - apk --no-cache add curl;
          helm plugin install https://github.com/hayorov/helm-gcs.git;
          helm repo add my-gcs-repo gs://my-private-helm-gcs-repository;
          chmod -R 777 $HELM_DATA_HOME;
```



# 11 Helm Version[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-version "Permanent link")

Argo CD will assume that the Helm chart is v3 (even if the apiVersion field in the chart is Helm v2), unless v2 is explicitly specified within the Argo CD Application (see below).

If needed, it is possible to specifically set the Helm version to template with by setting the `helm-version` flag on the cli (either v2 or v3):

```
argocd app set helm-guestbook --helm-version v3
```

Or using declarative syntax:

```
spec:
  source:
    helm:
      version: v3
```

# 12 Helm `--pass-credentials`[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-pass-credentials "Permanent link")

Helm, [starting with v3.6.1](https://github.com/helm/helm/releases/tag/v3.6.1), prevents sending repository credentials to download charts that are being served from a different domain than the repository.

If needed, it is possible to opt into passing credentials for all domains by setting the `helm-pass-credentials` flag on the cli:

```
argocd app set helm-guestbook --helm-pass-credentials
```

Or using declarative syntax:

```
spec:
  source:
    helm:
      passCredentials: true
```

# 13 Helm `--skip-crds`[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-skip-crds "Permanent link")

Helm installs custom resource definitions in the `crds` folder by default if they are not existing. See the [CRD best practices](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/) for details.

If needed, it is possible to skip the CRD installation step with the `helm-skip-crds` flag on the cli:

```
argocd app set helm-guestbook --helm-skip-crds
```

Or using declarative syntax:

```
spec:
  source:
    helm:
      skipCrds: true
```






