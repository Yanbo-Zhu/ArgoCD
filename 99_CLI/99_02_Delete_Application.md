
https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/

Apps can be deleted with or without a cascade option. A **cascade delete**, deletes both the app and its resources, rather than only the app.

## 0.1 Deletion Using `argocd`

To perform a non-cascade delete:
```
argocd app delete APPNAME --cascade=false
```


To perform a cascade delete:
```
argocd app delete APPNAME --cascade
```

or

```
argocd app delete APPNAME
```

## 0.2 Deletion Using `kubectl`

To perform a non-cascade delete, make sure the finalizer is unset and then delete the app:

```
kubectl patch app APPNAME  -p '{"metadata": {"finalizers": null}}' --type merge
kubectl delete app APPNAME
```

To perform a cascade delete set the finalizer, e.g. using `kubectl patch`:

```
kubectl patch app APPNAME  -p '{"metadata": {"finalizers": ["resources-finalizer.argocd.argoproj.io"]}}' --type merge
kubectl delete app APPNAME
```

## 0.3 About The Deletion Finalizer[¶](https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/#about-the-deletion-finalizer "Permanent link")

```
metadata:
  finalizers:
    # The default behaviour is foreground cascading deletion
    - resources-finalizer.argocd.argoproj.io
    # Alternatively, you can use background cascading deletion
    # - resources-finalizer.argocd.argoproj.io/background
```

When deleting an Application with this finalizer, the Argo CD application controller will perform a cascading delete of the Application's resources.

Adding the finalizer enables cascading deletes when implementing [the App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#cascading-deletion).

The default propagation policy for cascading deletion is [foreground cascading deletion](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#foreground-deletion). Argo CD performs [background cascading deletion](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#background-deletion) when `resources-finalizer.argocd.argoproj.io/background` is set.

When you invoke `argocd app delete` with `--cascade`, the finalizer is added automatically. You can set the propagation policy with `--propagation-policy <foreground|background>`.







