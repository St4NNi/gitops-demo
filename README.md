# Advanced Kubernetes (CLUM-2024)
## GitOps Hands-On for the de.NBI Cloud Usermeeting 2024
### Step 0: Preparations
Demonstrations of gitops using fluxcd.



#### Installing flux cd cli:

https://fluxcd.io/flux/installation/


#### Adding auto-complete for your shell:

`. <(flux completion bash)`

`zsh`, `fish` and `powershell` will also work!


#### Bootstrapping a cluster:

Create a `GITHUB_TOKEN`, [personal access token (PAT)}(https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens).

Make sure that your PAT is exported in your shell:

`> export GITHUB_TOKEN=<gh-token>` 


Make sure that you have configured the correct kubectl config for the cluster.

```
flux bootstrap github 
--token-auth
--owner=<org>
--repository=<repo-name>
--path=<path-in-repo>
--branch=main
```


### Step 1: Flux resources

Flux has a few main resources that are used to manage your applications.


#### Repositories

Git:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app
spec:
  url: https://<host>/<org>/app
  ref:
    branch: main
```
Specifies another git repo. Can have optional authentication.


```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: apps
spec:
  url: https://<host>/<org>/charts
```

Repository referencing Helm charts.


#### References

```yaml

apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: app
spec:
  chart:
    spec:
      chart: app
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: apps
  values:
    replicas: 2
```


```yaml

apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infra-configs
spec:
  interval: 1h
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: app
  path: ./clusters/dev/giessen/configs/infra # Subpath in repo
  prune: false # Should resources be deleted
```

Create a new subfolder in the `projects` directory for your name. Reference it in the deployments folder via a `Kustomization`.
Create a `kustomization.yaml` and a yaml containing an nginx deployment as well as an associated service. See [here](https://github.com/ag-computational-bio/kubernetes-course) for example yamls.  


### Step 2: Secrets

#### Secrets using SOPS




