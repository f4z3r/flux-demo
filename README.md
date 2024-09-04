# Flux Demo

## Setup

### Generate a GitHub PAT

See the documentation: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens

For fined-grained control, grant the token Admin and content read/write permissions on the
repository.

### Setup a cluster with Flux

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
