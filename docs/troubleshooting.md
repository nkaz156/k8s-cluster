[Docs README](./README.md)
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