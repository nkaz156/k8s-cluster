# Cluster GitOps repo:
This repo is for an already bootstrapped cluster. It uses Cilium as a CNI and kube-proxy replacement, and Flux to manage deployment of software. It also uses Cilium as a gatewayAPI provider and as a load balancer via L2 announcements (I'll get around to BGP sometime when I get a better router).

The goal is to have a pseudo-production cluster that can hum along for months on end without needing maintenance, with only the necessary compromises to run on cheap hardware.

Helpful resources:
- [Kubesearch](https://kubesearch.dev/)




## Deploying Cilium (from [Talos docs](https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium) and [Cilium docs](https://docs.cilium.io/en/stable/network/l2-announcements/)):

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

## Fluxifying it:
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

## Deploying Flux:
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
        --path ./flux-managed 
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


## Credits:
 - https://blog.kammel.dev/post/k8s_home_lab_2025_01/
