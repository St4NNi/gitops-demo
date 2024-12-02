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

Imperative:
```yaml
flux create source git <name> \
--url=<url> \
--branch=main
```

Declarative:
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

Installing sops:

https://github.com/getsops/sops/releases

Generate a gpg key:

```bash
export KEY_NAME="cluster0.yourdomain.com"
export KEY_COMMENT="flux secrets"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: ${KEY_COMMENT}
Name-Real: ${KEY_NAME}
EOF
```

Retrieve the fingerprint:

```bash

gpg --list-secret-keys "${KEY_NAME}"

sec   rsa4096 2020-09-06 [SC]
      1F3D1CED2F865F5E59CA564553241F147E7C5FA4
```

Store fingerprint as env-var:

```bash
export KEY_FP=1F3D1CED2F865F5E59CA564553241F147E7C5FA4
```

Create a secret in the k8s cluster:

```bash

gpg --export-secret-keys --armor "${KEY_FP}" |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin
```

Configure the key in the git directory:


```bash

cat <<EOF > ./clusters/cluster0/.sops.yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    pgp: ${KEY_FP}
EOF
```

Encrypt secrets using OpenPGP:

```bash

kubectl -n default create secret generic basic-auth \
--from-literal=user=admin \
--from-literal=password=change-me \
--dry-run=client \
-o yaml > basic-auth.yaml
```

```bash
sops --encrypt --in-place basic-auth.yaml
```

Reference the secret in your Kustomization

```bash

apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-secrets
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: my-secrets
  path: ./
  prune: true
  decryption:
    provider: sops # SOPS
    secretRef:
      name: sops-gpg # Secret name
```
