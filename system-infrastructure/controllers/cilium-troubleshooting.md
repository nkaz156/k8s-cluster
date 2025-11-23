Check Cilium's view of services:
```bash
kubectl exec -n kube-system ds/cilium -- cilium service list
```
Check Cilium's view of LoadBalancers
```bash
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list
```