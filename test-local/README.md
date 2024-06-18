# Test locally

How to test on a local educates kind cluster (with local registry) and kapp-controller:

```
educates admin cluster create --config kind-config.yaml
```

1.  Build the image and push it into your local registry:
    
    ```
    imgpkg --debug push -i localhost:5001/gitops-configurer:devel -f ../overlays
    ```

2.  Create the required [RBAC](./rbac.yaml).

    ```
    kubectl apply -f rbac.yaml
    ```

3.  Create your version of the configuration files in the [versions](./secret-versions.yaml), [common](./secret-common.yaml) and 
    [workshops](./secret-workshops.yaml) secrets and deploy them into the cluster:

    ```
    kubectl apply -f secret-versions.yaml
    kubectl apply -f secret-common.yaml
    kubectl apply -f secret-workshops.yaml
    ```

4.  Create the required [Gitops App definition](./crd-devel.yaml) and deploy it into your cluster.

    ```
    kubectl apply -f crd-devel.yaml
    ```

5. If you want to test any change in configuration, modify the appropriate secret and apply it into the cluster and wait for a reconciliation.
   If you don't want to wait, kick the reconciliation manually of the main gitops app:

   ```
   kctrl app kick -a workshops-gitops -n  package-installs
   ```
