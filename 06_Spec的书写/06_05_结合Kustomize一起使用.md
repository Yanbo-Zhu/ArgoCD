

https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/



# 1 Patches

Patches are a way to kustomize resources using inline configurations in Argo CD applications. `patches` follow the same logic as the corresponding Kustomization. Any patches that target existing Kustomization file will be merged.

This Kustomize example sources manifests from the `/kustomize-guestbook` folder of the `argoproj/argocd-example-apps` repository, and patches the `Deployment` to use port `443` on the container.

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomize-inline-example
namespace: test1
resources:
  - https://github.com/argoproj/argocd-example-apps//kustomize-guestbook/
patches:
  - target:
      kind: Deployment
      name: guestbook-ui
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/ports/0/containerPort
        value: 443
```

This `Application` does the equivalent using the inline `kustomize.patches` configuration.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-inline-guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: test1
    server: https://kubernetes.default.svc
  project: default
  source:
    path: kustomize-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: master
    kustomize:
      patches:
        - target:
            kind: Deployment
            name: guestbook-ui
          patch: |-
            - op: replace
              path: /spec/template/spec/containers/0/ports/0/containerPort
              value: 443
```

The inline kustomize patches work well with `ApplicationSets`, too. Instead of maintaining a patch or overlay for each cluster, patches can now be done in the `Application` template and utilize attributes from the generators. For example, with [`external-dns`](https://github.com/kubernetes-sigs/external-dns/) to set the [`txt-owner-id`](https://github.com/kubernetes-sigs/external-dns/blob/e1adc9079b12774cccac051966b2c6a3f18f7872/docs/registry/registry.md?plain=1#L6) to the cluster name.

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: external-dns
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - clusters: {}
  template:
    metadata:
      name: 'external-dns'
    spec:
      project: default
      source:
        repoURL: https://github.com/kubernetes-sigs/external-dns/
        targetRevision: v0.14.0
        path: kustomize
        kustomize:
          patches:
          - target:
              kind: Deployment
              name: external-dns
            patch: |-
              - op: add
                path: /spec/template/spec/containers/0/args/3
                value: --txt-owner-id={{.name}}   # patch using attribute from generator
      destination:
        name: 'in-cluster'
        namespace: default
```

# 2 Components

Kustomize [components](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/components.md) encapsulate both resources and patches together. They provide a powerful way to modularize and reuse configuration in Kubernetes applications.

Outside of Argo CD, to utilize components, you must add the following to the `kustomization.yaml` that the Application references. For example:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
...
components:
- ../component
```

With support added for components in `v2.10.0`, you can now reference a component directly in the Application:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: application-kustomize-components
spec:
  ...
  source:
    path: examples/application-kustomize-components/base
    repoURL: https://github.com/my-user/my-repo
    targetRevision: main

    # This!
    kustomize:
      components:
        - ../component  # relative to the kustomization.yaml (`source.path`).
```


# 3 Custom Kustomize versions[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/#custom-kustomize-versions "Permanent link")

Argo CD supports using multiple Kustomize versions simultaneously and specifies required version per application. To add additional versions make sure required versions are [bundled](https://argo-cd.readthedocs.io/en/stable/operator-manual/custom_tools/) and then use `kustomize.path.<version>` fields of `argocd-cm` ConfigMap to register bundled additional versions.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
    kustomize.path.v3.5.1: /custom-tools/kustomize_3_5_1
    kustomize.path.v3.5.4: /custom-tools/kustomize_3_5_4
```

Once a new version is configured you can reference it in an Application spec as follows:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
spec:
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: kustomize-guestbook

    kustomize:
      version: v3.5.4
```

Additionally, the application kustomize version can be configured using the Parameters tab of the Application Details page, or using the following CLI command:

```
argocd app set <appName> --kustomize-version v3.5.4
```

# 4 Build Environment[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/#build-environment "Permanent link")

Kustomize apps have access to the [standard build environment](https://argo-cd.readthedocs.io/en/stable/user-guide/build-environment/) which can be used in combination with a [config managment plugin](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/) to alter the rendered manifests.

You can use these build environment variables in your Argo CD Application manifests. You can enable this by setting `.spec.source.kustomize.commonAnnotationsEnvsubst` to `true` in your Application manifest.

For example, the following Application manifest will set the `app-source` annotation to the name of the Application:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-app
  namespace: argocd
spec:
  project: default
  destination:
    namespace: demo
    server: https://kubernetes.default.svc
  source:
    path: kustomize-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
    kustomize:
      commonAnnotationsEnvsubst: true
      commonAnnotations:
        app-source: ${ARGOCD_APP_NAME}
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```











