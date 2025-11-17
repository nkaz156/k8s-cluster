# System Infrastructure:

This is where services that a human user should never directly interact with live.

Troubleshooting:

Sometimes failed cilium pods get stuck and need to be deleted manually (normally during/after a reboot)
```bash
kubectl delete pods -n kube-system -l io.cilium/app=operator --field-selector status.phase!=Running
```