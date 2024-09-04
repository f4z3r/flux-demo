# Flux Demo

<!--toc:start-->
- [Flux Demo](#flux-demo)
  - [Setup](#setup)
    - [Generate a GitHub PAT](#generate-a-github-pat)
    - [Setup a Cluster with Flux](#setup-a-cluster-with-flux)
  - [Structure of the Repo](#structure-of-the-repo)
    - [Cluster Definitions](#cluster-definitions)
    - [Infrastructure Definition](#infrastructure-definition)
    - [Application Definitions](#application-definitions)
  - [Workflow](#workflow)
<!--toc:end-->

---

## Setup

### Generate a GitHub PAT

See the documentation: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens

For fined-grained control, grant the token Admin and content read/write permissions on the
repository.

### Setup a Cluster with Flux

Setup the required tooling with `devbox shell`, then

```bash
# install a cluster
kind create cluster -n demo
# set the token
export GITHUB_TOKEN='<redacted>'
# onboard flux
flux bootstrap github \
  --token-auth \
  --owner=f4z3r \
  --repository=flux-demo \
  --branch=main \
  --path=clusters/demo \
  --personal
```

<details>
<summary>Sample output from Flux installation</summary>

```
► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/f4z3r/flux-demo.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed component manifests to "main" ("158753158f3c760f741f22ed7f68bdee1b66e475")
► pushing component manifests to "https://github.com/f4z3r/flux-demo.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("f7a731c9e05b983c33012805a6be60b30d34505e")
► pushing sync manifests to "https://github.com/f4z3r/flux-demo.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

</details>

## Structure of the Repo

The repository is structured in three distinct parts:

- The cluster definitions
- The infrastructure definition
- The application definitions

### Cluster Definitions

The cluster definitions are located under `./clusters/` and typically consist of two parts. A file
defining how the infrastructure should be deployed, and a part that defines what applications need
to be deployed.

The infrastructure part is essentially a back-reference to the same repository and a templating of
the infrastructure for the cluster. The back-reference is needed in order to be able to freeze the
infrastructure to a specific version. This is needed as it is typically needed to ship the
infrastructure as a whole, and not individual controllers. Using the back-reference the "state of
the infrastructure" can be versioned, and deployed individually to clusters. The templating is of
course needed to inject cluster dependent variables into the infrastructure definition.

See [`./clusters/demo/infrastructure.yaml`](./clusters/demo/infrastructure.yaml) for an example
infrastructure deployment.

<details>
<summary>Back-Reference</summary>

The back reference is achieved using a `GitRepository` that defines the mono-repo:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/f4z3r/flux-demo.git
  ref:
    tag: v0.1.0
```

The idea here is to allow to pin the version of the infrastructure definition (see later section).
This allows to make fundamental changes to the infrastructure definition without affecting clusters.
The pinning is done by tagging the mono-repository. All tags essentially represent a state of the
infrastructure that is tested and approved.

The infrastructure engineering cluster used to test infrastructure changes of course references
`main`/`master` instead of a specific tag. People can thus introduce changes to the infrastructure
in parallel via PRs to `main`, and then use standard release management to shortly block merges to
`main` complete a full testing, and then tag that state with a new infrastructure version. That
version can then be rolled out to productive platforms as needed by bumping their tag references
within their back-references.

This setup makes the following assumptions in terms of best practices:
- The infrastructure is fully deployed to all clusters.
  - There is no mix-and-match of controllers deployed only to specific platforms.
  - Therefore, all platforms are fully uniform in terms of infrastructure.
  - The infrastructure is tested as a whole and is considered an atomic component, thus why it is
    versioned as a whole.
- Infrastructure changes need to be deployed individually to production clusters, hence why each
  cluster definition has its own back-reference.
- Infrastructure is only customized for each cluster, not modified in major ways.

<!-- TODO: add shortcomings of such an approach -->

</details>

<details>
<summary>Templating: variable injection</summary>

Even though the same infrastructure is deployed to all clusters, that same infrastructure needs to
be customized. Here it is not meant that some components are deployed only optionally, but that the
components themselves are configured. An example of this is the hostname that the ingress should
listen to, which of course would be different for each platform. This is achieved via a
`Kustomization`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 5m
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: infrastructure
  path: ./infrastructure/
  prune: true
  wait: true
  postBuild:
    substitute:
      meta_cluster_name: "demo"
      meta_network_zone: "dmz-123"
      meta_productive: true
      infrastructure_nginx_release_interval: "1m"
```

Note that the `Kustomization` references the infrastructure back-references and complements it with
variables substitution such as `meta_cluster_name` or `infrastructure_nginx_release_interval`. These
variables are used within the infrastructure definition and can be set here to customize some
infrastructure components as needed.

</details>

<!-- TODO: app deployment for cluster -->

### Infrastructure Definition

<!-- TODO -->

### Application Definitions

<!-- TODO -->

## Workflow

<!--
TODO:
  - deploy new app
  - test new infra
  - rollout new infra
  - onboard new cluster
-->
