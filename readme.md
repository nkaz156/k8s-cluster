# Introduction:
This repo provides declarative configuration management for my kubernetes cluster, which is still in its infancy, but some day will have a lot of cool services. Currently, I have my GitOps repos separated into this one, a private repo to hold sops-encrypted secrets, and a private repo which holds my talhelper configuration. 

I currently use Cilium as a CNI and kube-proxy replacement, and Flux to manage deployment of software. I also uses Cilium as a GatewayAPI provider and as a load balancer via L2 announcements (I'll get around to BGP sometime when I get a better router).

The goal is to have a pseudo-production cluster that can hum along for months on end without needing maintenance, while sipping power. I'm currently running on a set of three thin clients from Dell and Lenovo, each of which runs proxmox and hosts a Talos worker and control plane. At some point, I'd like to get more thin clients and move away from the additional resource demands of a hypervisor, but this setup works well for now, and it's very easy to change configurations or fix things I break along the way.

I am not a software professional or DevOps engineer, so while I try to keep everything in line with kubernetes/cybersecurity best practices and err towards overhardening, this is a complex project with many dependencies. I am currently not exposing anything to the internet and use a VPN ([Netbird](https://netbird.io/), it's amazing) to access hosted services remotely.

## Structure (In rollout order):
 - `clusters/production-ish` is the target directory for Flux. All other directories Flux watches are specified by kustomizations in `rollout.yaml`. The names of the kustomizations are prefixed by numbers in order to have them display neatly in the flux CLI.
 - `namespaces` holds any namespaces which contain various apps/app stacks
 - `repositories` holds all Flux repository sources
 - secrets are stored in a private repository, so `secrets-demo` is a non-functional minimal look into that repo. SOPS is clunky and doesn't handle secret rotation automatically, but it avoids any of the chicken and egg problems associated with self-hosting a vault instance and also minimizes cloud dependencies.
 - `crds` holds custom resource definitions which are needed for various system services, but not managed by helm releases. 
 - `system-infrastructure` holds the base-level deployments which run the cluster. Any configuration is declarative through this repo, so these should ideally only be interacted with through the monitoring stack. `system-infrastructure` is split between `configs` and `controllers`. Controllers houses the source helm release/deployment, configs houses the objects managed by the controllers (cilium LB IP pools, Cert manager issuers, etc.)
 - `user-infrastructure` houses software like sso providers, metrics dashboards, pgadmin, or other things an administrator may administrate. 
 - `apps` holds any other deployments which end users will use. Right now it's just a demo instance of nginx I set up to learn about deployments without helm.

## Inspirations:
 - anokfireball's [homelab-as-code](https://github.com/anokfireball/homelab-as-code) was a major help in structuring this repository along flux best practices, and in finding core services to use.
 - onedr0p's [cluster-template](https://github.com/onedr0p/cluster-template) and [home-ops](https://github.com/onedr0p/home-ops) has also informed my service selection and inspired me to use taskfiles or just to automate the bootstrap process.
 - [DaTosh Blog](https://blog.kammel.dev/post/k8s_home_lab_2025_01/) gives a simple, digestible breakdown of setting up flux


## Helpful resources:
- [Kubesearch](https://kubesearch.dev/) - search through all github repos tagged `k8s-at-home` or `kubesearch`
- [Artifacthub](https://artifacthub.io/) - large, public helm chart aggregator

## Lessons Learned:
- Set up the repo such that the path you give to flux only contains kustomizations. This is important to having control over flux's rollout order.
- Make sure the install sequencing installs deployments and their corresponding CRDs before it tries to create objects of the corresponding custom resources. Example: Flux got stuck trying to create a `ClusterIssuer` before installing `cert-manager`. This is what prompted me to restructure the repo into its current form. Before, I had all apps in flux's main watched path folder.
 - Just send it, it's GitOpsified for a reason. Disaster recovery is pretty simple

## Current bootstrap process (Non-automated)
### Deploying Cilium (from [Talos docs](https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium) and [Cilium docs](https://docs.cilium.io/en/stable/network/l2-announcements/)):

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

```bash
helm install \
    cilium \
    cilium/cilium \
    --version 1.18.3 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445\
    --set=gatewayAPI.enabled=true \
    --set=gatewayAPI.enableAlpn=true \
    --set=gatewayAPI.enableAppProtocol=true \
    --set l2announcements.enabled=true \
    --set k8sClientRateLimit.qps=10 \
    --set k8sClientRateLimit.burst=20 
```
**Make sure rate limits and lease duration specs are set appropriately to your number of services** ([Cilium docs entry](https://docs.cilium.io/en/stable/network/l2-announcements/#sizing-client-rate-limit)).

With the Cilium CLI installed 

### Fluxifying it (First start only):
Converting the cilium installation to flux HelmRelease format so that Flux can manage its updates from here on

1. Create the Helm repository source:
```bash
flux create source helm cilium \
--url=https://helm.cilium.io \
--interval=5m \
--export > ./flux-managed/infrastructure/cilium/repository.yaml
```

2. Get current Cilium Helm values:
```bash
helm get values cilium -n kube-system > /tmp/current-cilium-values.yaml
```

3. Create the `release.yaml`:
```bash
flux create helmrelease cilium \
    --interval=5m \
    --source=HelmRepository/cilium \
    --target-namespace=kube-system \
    --chart=cilium \
    --chart-version="1.18.3" \
    --values=/tmp/current-cilium-values.yaml \
    --export > ./flux-managed/infrastructure/cilium/release.yaml
```

This process also generalizes to other helm charts if you want to do an easy deployment with helm before committing the app to Github.

### Deploying Flux:
1. Create a fine-grained personal access token to give Flux access to the repo (https://github.com/settings/personal-access-tokens). 
This way we can use the principle of least privilege by only allowing Flux access to the specific repo.

    - Read-only access: Administration, Metadata
    - Read and write access: Contents

1. Bootstrap Flux. The `--token-auth` flag instructs Flux to use the fine-grained PAT we set up instead of an SSH key.
    ```bash
    flux bootstrap github \
        --token-auth \
        --owner nkaz156 \
        --personal \
        --repository k8s-cluster \
        --branch main \
        --path ./clusters/production-ish
    ```

1. You might have to clear the `GITHUB_TOKEN` environment variable to get flux to ask for a github token. You could also try setting it as an environment variable if that doesn't work.

```bash
unset GITHUB_TOKEN
```
```bash
export GITHUB_TOKEN=<your PAT>
```

### Testing Cilium:

With the Cilium CLI installed, run the following command:
The `cilium-test-1` namespace needs to be privileged to work properly, which is a deviation from talos' default spec, so we need to make the namespace privileged. Delete it after the test.
```bash
kubectl create namespace cilium-test-1
kubectl label namespace cilium-test-1 \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged

# Then run the test
cilium connectivity test
```

## Helpful commands quick reference:

Clear a namespace stuck on 'Terminating:'

```bash
kubectl get namespace longhorn-system -o json \
  | jq '.spec.finalizers = []' \
  | kubectl replace --raw /api/v1/namespaces/longhorn-system/finalize -f -
  ```

Remember to replace `longhorn-system` with whatever the name of the problem namespace is.

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
```
creates and removes a busybox pod that you're exec'd into to troubleshoot networking

Ethernet dropping fix for Lenovo M720q (eee on with incompatible switch):
ethtool - diagnosing an ethernet connection
Add `post-up /sbin/ethtool --set-eee $IFACE eee off` indented after the interface you want to disable eee on:
```
auto eno1
iface eno1 inet dhcp
    post-up /sbin/ethtool --set-eee $IFACE eee off
```

`yq` lets you programatically interact with `.yaml` files, for instance:
```bash
yq -i 'select(.apiVersion == "*.fluxcd.io/*") | (.spec.interval = "1h")' deploy-order.yaml
```
`yq` edits the file in place (`-i`), selecting all subfiles with the specified `.apiVersion`, then setting `.spec.interval` to 1 hour.

You can also pipe in other options, like setting the retryInterval and timeout here.
```bash
yq -i 'select(.apiVersion == "*.fluxcd.io/*") | (.spec.interval = "1h") | (.spec.retryInterval = "1m") | (.spec.timeout = "5m")' deploy-order.yaml
```

This is great, but if you want to update all of the retryIntervals across the repository, you can use find and xargs.

```bash
find . -type f -name "*.yaml" -print0
```
Recursively finds all files (`-type f`) in the current working directory (`.`) with a `.yaml` file extension (`-name "*.yaml"`) in a format suitable for `xargs -0` (`-print0`) - this prints null characters as delimiters. `-X` is also an option.