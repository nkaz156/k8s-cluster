# Secrets workflow:

I store my secrets in a private git repo (encrypted with SOPS and age) and recommend you do the same. This repo just provides a look into what that git repo looks like. Flux is not pointed at it and doesn't know about its existence. The age key here is just a demo key. The `kustomization` pointing to my secrets repo is in `rollout.yaml` and the `GitRepository` source is in `repositories/`

**WARNING:** Keep a `.gitignore` file in your secrets repo to avoid committing unencrypted secrets - mine is below. It ignores everything, then exempts everything ending in `.sops.yaml`. This is critical even in a private repo. I only have an unencrypted secret here as a demo.
```.gitignore
*
!*.sops.yaml
```

You can encrypt a secret by running:
```bash
sops -e example-secret.yaml > example-secret.sops.yaml
```
Currently I just do this per-secret every time I create a new one


Here's how I set this up:

## 1. Create a Separate GitRepository Source

Create a `GitRepository` resource that points to your private secrets repo:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: secrets-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/your-org/secrets-repo
  ref:
    branch: main
  secretRef:
    name: secrets-repo-auth  # Git credentials for private repo
```

## 2. Create Authentication Secret

You'll need to provide credentials for the private repo:

```bash
export HISTCONTROL=ignorespace
 export GITHUB_SECRET_REPO_TOKEN=<your PAT>
flux create secret git secrets-repo-auth \
  --url=https://github.com/your-org/secrets-repo \
  --username=flux \
  --password=$GITHUB_SECRET_REPO_TOKEN
unset GITHUB_SECRET_REPO_TOKEN
```
Note that the username field can be whatever since Github only authenticates with a PAT over HTTPS. Use a fine-grained PAT with the same privileges as your main flux token. Setting the `HISTCONTROL` environment variable to `ignorespace` prevents your PAT from ending up in your bash history.


Or for SSH:

```bash
flux create secret git secrets-repo-auth \
  --url=ssh://git@github.com/your-org/secrets-repo \
  --private-key-file=/path/to/private/key
```

## 3. Create a Kustomization Pointing to the Secrets Repo

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: secrets
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: secrets-repo  # References your private repo
  path: ./clusters/production  # Path within secrets repo
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age  # Your age key secret
```

## 4. Structure Your Repos

**Public repo** (flux-repo):
- Infrastructure manifests
- Kustomizations
- HelmReleases
- Non-sensitive configuration

**Private repo** (secrets-repo):
- SOPS-encrypted secrets
- `.sops.yaml` configuration
- Age public key in the repo (private key stays in cluster)

## Important Notes

- The age **private key** should be stored as a Kubernetes secret in your cluster (usually created during Flux bootstrap). Here's an example:

```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=keys.txt
```
Where `keys.txt` is the file containing your Age private key on your local machine and `age.agekey` is the name of the file on the k8s cluster. The file on the cluster is required to be `age.agekey` exactly for flux to work.

- The age **public key** can be in your private repo's `.sops.yaml`
- Both Kustomizations can reference the same `sops-age` secret for decryption
- You can have multiple Kustomizations from the same secrets repo for different paths/environments

This approach gives you the flexibility of a public GitOps repo while keeping your encrypted secrets in a separate, access-controlled repository.

## Credits:

Thanks to [Claude](claude.ai) for a very solid structure for this