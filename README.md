# Educates Workshop GitOps Configurer

The Educates Workshop GitOps Configurer project includes a set of tools
to enable an "Educates Operator's" provisioning of workshops and associated
training portals via a GitOps approach, and only expose the minimum necessary
configuration for the operator.

The Educates Operator can configure environments to host a set of workshops
and training portals that are sourced by one or more Educates clusters.

Workshops are organized around the concept of "workshop bundle",
where the bundles is a packages set of workshop definitions.
Workshop bundles are typically authored and managed from the same Github
repositories, but do not necessarily need to be.

Workshop bundles are packaged as OCI container images that are sourced
via a GitOps reconciler,
along with associated training portal configuration from the workshop
configuration github repository.

The project leverages Carvel tooling to accomplish two concerns:

- _GitOps reconcilation_ - use the _Kapp controller_ to monitor for specific
  OCI and Github artifacts and versions

- _Simplified configuration through overlays_ - the educates operator
  is not concerned with specifics of authoring either workshop definitions
  or training portal configuration.
  The operator only needs to be concerned with operational configuration.
  Carvel `ytt` is used to manage overlay templates to take simplified
  operator configuration and configure training portal resources under the hood.

## Educates Operator concerns

### Configure a new workshop configuration repository

- new github repo
- create credentials for access
- create `environments` directory at root

### Configure a new environment configuration

- create a new directory with appropriate name for environment
  under `environments` directory.

- following _Configure_ sections outline how to add
  common training portal and workshop bundle configurations

### Configure workshop bundle versions (this may be elevated above educates operator to prod)

Add the following to `versions.yaml`

General:

- mode (This can be one of `one_app` or `app_per_bundle`)
- reconcilation interval (default 3min)
- overlays version (aligns to educates minor version?)

Workshop bundle:

- names
- package coordinates
- versions/references
- pkg credentials to access

### Configure training portal common configuration

Add following to `common.yaml` file to configure:

- themes
- ancestors
- sessions configuration
- registration
- credentials

### Configure workshop session characteristics

for each training portal

- training portal name
- workshop names associated with workshop bundle(s)
- defaults, or configurations by specific workshop:
  - capacity
  - initial
  - reserved
  - expires
  - overtime
  - deadline
  - orphaned
  - overdue
  - refresh

## Platform Operator concerns

### Deploy workshop configuration as gitops in an Educates Cluster

**NOTE** THIS NEEDS TO BE REWRITEN AS WE DON'T USE SCRIPTS TO INSTALL/UNINSTALL IN CLUSTER

If manually deploying to an educates cluster,
do following steps:

1.  Ensure you have an educates cluster, and KUBECONFIG set to point to its config

1.  Configure the following environment variables in shell or runner:

    ```
    export KUBECONFIG=<path to KUBECONFIG>
    export WORKSHOP_CONFIG_REPO_URL="https://github.com/<org/repo>" # destination repository with workshop training portal configuration overlays
    export WORKSHOP_CONFIG_REPO_REF="origin/main" # branch/tag reference at workshop config repository that will be reconciled to cluster - defaults to origin/main
    export WORKSHOP_OVERLAYS_PKG_URL="vmware-tanzu-learning/educates-workshop-gitops-configurer" # url of overlays package - defaults to vmware-tanzu-learning for prod releases, you may want to override for local development.
    export ENVIRONMENT="<environment name>" # directory at workshop configuration reposistory from where workshop bundle configuration will be sourced
    export GITHUB_USER="<>" # github user whose GITHUB_TOKEN has OCI package read permissions to the workshop config repo
    export GITHUB_TOKEN="<>" # GITHUB_USER's token that OCI package read permissions to the workshop config repo
    export EDUCATES_VERSION="latest" # defaults to "latest", although you should use the appropriate semver when implemented.
    ```

    example for spring academy tenant staging environment with defaults

    ```
    export WORKSHOP_CONFIG_REPO_URL="https://github.com/vmware-tanzu-learning/spring-academy-educates-workshops-config>"
    export ENVIRONMENT="staging"
    export GITHUB_USER="<redacted>"
    export GITHUB_TOKEN="<redacted>"
    export KUBECONFIG="<redacted>"
    ```

1.  Run the install script:

    ```bash
    install/install.sh
    ```

1.  Verify the application and workshop configuration has reconciled:

    - via `kapp`:

      ```bash
      kapp list -n package-installs
      ```

      you should see similar to follows:

      ```no-highlight
      Name                  Namespaces                                                      Lcs   Lca
      workshops-gitops.app  (cluster),                                                      true  22s
                          workshops-course-spring-academy-sample,workshops-spring-guides

      Lcs: Last Change Successful
      Lca: Last Change Age
      ```

      This shows a coarse grain status of all the gitops configuration from your workshop configuration repo.
      Any failures across any workshop bundles will show a failure here.

    - via `kubectl`:

      ```bash
      kubectl --kubeconfig ${KUBECONFIG} get apps.kappctrl.k14s.io -a
      ```

      you should see similar to follows:

      ```no-highlight
      NAMESPACE                                NAME                                     DESCRIPTION           SINCE-DEPLOY   AGE
      educates-package                         educates-cluster-essentials              Reconcile succeeded   27m            27m
      educates-package                         educates-training-platform               Reconcile succeeded   25m            27m
      package-installs                         workshops-gitops                         Reconcile succeeded   103s           4m56s
      workshops-course-spring-academy-sample   workshops-course-spring-academy-sample   Reconcile succeeded   101s           4m54s
      workshops-spring-guides                  workshops-spring-guides                  Reconcile succeeded   101s           4m54s
      ```

      Notice listing apps across all namespaces allows you to see the state of the individual workshop bundle app deployments.
      You can use this to troubleshoot specific bundle failures.

### Remove workshop gitops configuration

Run the following script:

```bash
scripts/uninstall.sh
```

It will remove the carvel application and dependent config and security config.

## Educates Workshop GitOps Configurer Developer concerns

### Local development/testing of application overlays

This will produce a list of Carvel Apps with the required k8s credentials and configuration for courses, workshops, ....

```
ytt -v environment=test \
    -v mode=app_per_bundle \
    --data-values-file test/gitops-app/versions.yaml \
    -f overlays/gitops-app/src/bundle/config
```

```
ytt -v environment=test \
    -v mode=one_app \
    --data-values-file test/gitops-app/versions.yaml \
    -f overlays/gitops-app/src/bundle/config
```

### Local development/testing of workshop configuration overlays

This will produce a list of workshops, a trainingportal based on config and kbld/kapp configurations.

```
ytt -v name=workshop-bundle-colours \
    -v mode=app_per_bundle \
    --data-values-file test/portal-app/config \
    -f test/portal-app/workshops/workshop-bundle-colours \
    -f overlays/portal-app/src/bundle/config

ytt -v name=workshop-bundle-animals \
    -v mode=app_per_bundle \
    --data-values-file test/portal-app/config \
    -f test/portal-app/workshops/workshop-bundle-animals \
    -f overlays/portal-app/src/bundle/config
```

```
ytt -v name=global \
    -v mode=one_app \
    --data-values-file test/portal-app/config \
    -f test/portal-app/workshops/workshop-bundle-animals \
    -f overlays/portal-app/src/bundle/config
```

## TODO

- implement installation in terraform provisioning

- clean up naming to make maintenance and operator usage more intuitive.

- implement semantic versioning / tagged releases in workflow,
  ideally keep aligned with educates versions,
  environment versions,
  especially with training portal and workshop crd changes.

- troubleshooting guide(here in readme?).

  - missing secrets for workshop bundles
  - incorrect workshop bundles coordinates.

- implement external secrets handling
  (external secrets operator with either Cerberus of AWS secrets manager)

### nice to haves

- workshop configuration repository template or archetype generator.
