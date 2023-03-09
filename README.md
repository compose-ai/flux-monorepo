# flux2-kustomize-helm-example

[![test](https://github.com/fluxcd/flux2-kustomize-helm-example/workflows/test/badge.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/actions)
[![e2e](https://github.com/fluxcd/flux2-kustomize-helm-example/workflows/e2e/badge.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/actions)
[![license](https://img.shields.io/github/license/fluxcd/flux2-kustomize-helm-example.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/blob/main/LICENSE)

For this example we assume a scenario with two clusters: stage and prod.
The end goal is to leverage Flux and Kustomize to manage both clusters while minimizing duplicated declarations.

We will configure Flux to install, test and upgrade a demo app using
`HelmRepository` and `HelmRelease` custom resources.
Flux will monitor the Helm repository, and it will automatically
upgrade the Helm releases to their latest chart version based on semver ranges.

![flux-ui-apps.png](.github/screens/flux-ui-apps.png)

On each cluster, we'll install [Weave GitOps](https://docs.gitops.weave.works/) (an OSS UI for Flux)
to visualise and monitor the workloads managed by Flux.

## Prerequisites

You will need a Kubernetes cluster version 1.21 or newer.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on MacOS or Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **infrastructure** dir contains common infra tools like weave-gitops
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── prod 
│   └── stage
├── infrastructure
│   ├── configs
│   └── controllers
└── clusters
    ├── prod
    └── stage
```

### Applications

The apps configuration is structured into:

- **apps/base/** dir contains namespaces and Helm release definitions
- **apps/prod/** dir contains the prod Helm release values
- **apps/stage/** dir contains the stage values

```
./apps/
├── base
│   └── podinfo
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── release.yaml
│       └── repository.yaml
├── prod
│   ├── kustomization.yaml
│   └── podinfo-patch.yaml
└── stage
    ├── kustomization.yaml
    └── podinfo-patch.yaml
```

In **apps/base/podinfo/** dir we have a Flux `HelmRelease` with common values for both clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  releaseName: podinfo
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 50m
  values:
    ingress:
      enabled: true
      className: nginx
```

In **apps/stage/** dir we have a Kustomize patch with the stage specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0-alpha"
  test:
    enable: true
  values:
    ingress:
      hosts:
        - host: podinfo.stage
```

Note that with ` version: ">=1.0.0-alpha"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest chart version including alpha, beta and pre-releases.

In **apps/prod/** dir we have a Kustomize patch with the prod specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0"
  values:
    ingress:
      hosts:
        - host: podinfo.prod
```

Note that with ` version: ">=1.0.0"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest stable chart version (alpha, beta and pre-releases will be ignored).

### Infrastructure

The infrastructure is structured into:

- **infrastructure/controllers/** dir contains namespaces and Helm release definitions for Kubernetes controllers
- **infrastructure/configs/** dir contains Kubernetes custom resources such as cert issuers and networks policies

```
./infrastructure/
├── configs
│   ├── network-policies.yaml
│   └── kustomization.yaml
└── controllers
    ├── weave-gitops.yaml
    └── kustomization.yaml
```

In **clusters/prod/infrastructure.yaml** we replace the Let's Encrypt server value to point to the prod API:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: infra-configs
  namespace: flux-system
spec:
  # ...omitted for brevity
  dependsOn:
    - name: infra-controllers
  patches:
    - patch: |
        - op: replace
          path: /spec/acme/server
          value: https://acme-v02.api.letsencrypt.org/directory
      target:
        kind: ClusterIssuer
        name: letsencrypt
```

Note that with `dependsOn` we tell Flux to first install or upgrade the controllers and only then the configs.
This ensures that the Kubernetes CRDs are registered on the cluster, before Flux applies any custom resources.

## Bootstrap stage and prod

The clusters dir contains the Flux configuration:

```
./clusters/
├── prod
│   ├── apps.yaml
│   └── infrastructure.yaml
└── stage
    ├── apps.yaml
    └── infrastructure.yaml
```

In **clusters/stage/** dir we have the Flux Kustomization definitions, for example:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infra-configs
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/stage
  prune: true
  wait: true
```

Note that with `path: ./apps/stage` we configure Flux to sync the stage Kustomize overlay and 
with `dependsOn` we tell Flux to create the infrastructure items before deploying the apps.

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your stage cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your stage cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=stage \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/stage
```

The bootstrap command commits the manifests for the Flux components in `clusters/stage/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being installed on stage:

```console
$ watch flux get helmreleases --all-namespaces

NAMESPACE    	NAME         	REVISION	SUSPENDED	READY	MESSAGE 
flux-system  	weave-gitops 	4.0.12   	False    	True 	Release reconciliation succeeded
podinfo      	podinfo      	6.3.0   	False    	True 	Release reconciliation succeeded
```

Bootstrap Flux on prod by setting the context and path to your prod cluster:

```sh
flux bootstrap github \
    --context=prod \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/prod
```

Watch the prod reconciliation:

```console
$ flux get kustomizations --watch

NAME             	REVISION     	SUSPENDED	READY	MESSAGE                         
apps             	main/696182e	False    	True 	Applied revision: main/696182e	
flux-system      	main/696182e	False    	True 	Applied revision: main/696182e	
infra-configs    	main/696182e	False    	True 	Applied revision: main/696182e	
infra-controllers	main/696182e	False    	True 	Applied revision: main/696182e	
```

### Access the Flux UI

To access the Flux UI on a cluster, first start port forwarding with:

```sh
kubectl -n flux-system port-forward svc/weave-gitops 9001:9001
```

Navigate to http://localhost:9001 and login using the username `admin` and the password `flux`.

[Weave GitOps](https://docs.gitops.weave.works/) provides insights into your application deployments,
and makes continuous delivery with Flux easier to adopt and scale across your teams.
The GUI provides a guided experience to build understanding and simplify getting started for new users;
they can easily discover the relationship between Flux objects and navigate to deeper levels of information as required.

![flux-ui-depends-on](.github/screens/flux-ui-depends-on.png)

You can change the admin password bcrypt hash in **infrastructure/controllers/weave-gitops.yaml**:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: weave-gitops
  namespace: flux-system
spec:
  # ...omitted for brevity
  values:
    adminUser:
      create: true
      username: admin
      # bcrypt hash for password "flux"
      passwordHash: "$2a$10$P/tHQ1DNFXdvX0zRGA8LPeSOyb0JXq9rP3fZ4W8HGTpLV7qHDlWhe"
```

To generate a bcrypt hash please see Weave GitOps
[documentation](https://docs.gitops.weave.works/docs/configuration/securing-access-to-the-dashboard/#login-via-a-cluster-user-account). 

Note that on prod systems it is recommended to expose Weave GitOps over TLS with an ingress controller and
to enable OIDC authentication for your organisation members.
To configure OIDC with Dex and GitHub please see this [guide](https://docs.gitops.weave.works/docs/guides/setting-up-dex/).

## Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```sh
mkdir -p clusters/dev
```

Copy the sync manifests from stage:

```sh
cp clusters/stage/infrastructure.yaml clusters/dev
cp clusters/stage/apps.yaml clusters/dev
```

You could create a dev overlay inside `apps`, make sure
to change the `spec.path` inside `clusters/dev/apps.yaml` to `path: ./apps/dev`. 

Push the changes to the main branch:

```sh
git add -A && git commit -m "add dev cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev
```

## Identical environments

If you want to spin up an identical environment, you can bootstrap a cluster
e.g. `prod-clone` and reuse the `prod` definitions.

Bootstrap the `prod-clone` cluster:

```sh
flux bootstrap github \
    --context=prod-clone \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/prod-clone
```

Pull the changes locally:

```sh
git pull origin main
```

Create a `kustomization.yaml` inside the `clusters/prod-clone` dir:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - ../prod/infrastructure.yaml
  - ../prod/apps.yaml
```

Note that besides the `flux-system` kustomize overlay, we also include
the `infrastructure` and `apps` manifests from the prod dir.

Push the changes to the main branch:

```sh
git add -A && git commit -m "add prod clone" && git push
```

Tell Flux to deploy the prod workloads on the `prod-clone` cluster:

```sh
flux reconcile kustomization flux-system \
    --context=prod-clone \
    --with-source 
```

## Testing

Any change to the Kubernetes manifests or to the repository structure should be validated in CI before
a pull requests is merged into the main branch and synced on the cluster.

This repository contains the following GitHub CI workflows:

* the [test](./.github/workflows/test.yaml) workflow validates the Kubernetes manifests and Kustomize overlays with [kubeconform](https://github.com/yannh/kubeconform)
* the [e2e](./.github/workflows/e2e.yaml) workflow starts a Kubernetes cluster in CI and tests the stage setup by running Flux in Kubernetes Kind
