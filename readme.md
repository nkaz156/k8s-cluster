# Cluster GitOps repo:
This repo is for an already bootstrapped cluster. It uses Cilium as a CNI and kube-proxy replacement, and Flux to manage deployment of software. It also uses Cilium as a gatewayAPI provider and as a load balancer via l2 announcements (I'll get around to BGP sometime when I get a better router).

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

## Deploying Flux:
```bash
flux bootstrap github \
    --token-auth \
    --owner nkaz156 \
    --personal \
    --repository k8s-cluster \
    --branch main \
    --path ./flux-managed 
```

