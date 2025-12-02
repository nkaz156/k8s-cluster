[Docs README](./README.md)
# Talhelper Workflow:

Configure, apply, and start a cluster in under a minute, if you have secrets, a `talconfig.yaml`, and the prerequisite programs installed:

```bash
talhelper genconfig
talhelper gencommand apply --extra-flags --insecure | bash
talhelper gencommand bootstrap | bash
talhelper gencommand kubeconfig | bash
```

This depends on talhelper, Mozilla sops, and age, all of which can be easily installed using Homebrew. It assumes you already have a secret (don't use mine). Make sure sops is configured to use age (see the talhelper documentation: https://budimanjojo.github.io/talhelper/latest/guides/#configuring-sops-for-talhelper). Also make sure that the `SOPS_AGE_KEY_FILE` environment variable is set properly and you have a `.sops.yaml` file in your current working directory.
```bash
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
```
### Generating a secret:
To work, Talhelper needs a secret (`talsecret`). You have two options to get the secret:

1. If you have a cluster already, you can reuse its secret by running: 
    ```bash
    talosctl -n <controlplane-ip> get mc v1alpha1 -o jsonpath='{.spec}' > /tmp/machineconfig.yaml
    ```
    You can then run:
    ```bash
    talhelper gensecret -f /tmp/machineconfig.yaml > talsecret.sops.yaml && rm /tmp/machineconfig.yaml
    ```
    to get the secret into talhelper format. It will be unencrypted.

2. If you don't have a cluster yet (or want to start fully fresh), you can generate a new secret using the following workflow:
    ```bash
    talhelper gensecret > talsecret.sops.yaml
    ```
    **Note that the secret won't be encrypted yet.**

### Encrypt your secret:
```bash
sops -e -i talsecret.sops.yaml
```
This just encrypts the secret in place

### Make a `talconfig.yaml`:
There are plenty of good templates out there, but I based mine off of [Mircea Anton's](https://github.com/mirceanton/home-ops/tree/feat/cluster-bootstrap/kubernetes/bootstrap) setup and the [talhelper documentation](https://budimanjojo.github.io/talhelper/latest/guides/#example-talconfigyaml). My setup assumes you will install a replacement for the native kubernetes CNI and kube-proxy after bootstrapping the cluster. I use Cilium for this.

Customize the Talos image and return its schematic and version as an image URL that Talhelper can use:

> [!IMPORTANT]
> This schematic generation workflow has now been superseded by Talhelper's [URL generation](https://budimanjojo.github.io/talhelper/latest/guides/#adding-talos-extensions-and-kernel-arguments). 

```bash
TALOS_SCHEMATIC_ID=$( \
curl -X POST --data-binary @- https://factory.talos.dev/schematics <<EOF | jq -r '.id'
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
      - siderolabs/util-linux-tools
EOF
)
LATEST_TALOS_VERSION=$(curl -s https://factory.talos.dev/versions | jq -r '[.[] | select(test("^v[0-9]+\\.[0-9]+\\.[0-9]+$"))] | last')
echo "Schematic ID: $TALOS_SCHEMATIC_ID"
echo "Latest Version: $LATEST_TALOS_VERSION"
echo ""
echo "Add this to your talconfig.yaml:"
echo "talosImageURL: factory.talos.dev/installer/${TALOS_SCHEMATIC_ID}:${LATEST_TALOS_VERSION}"
```

This gets a Talos image schematic from the [Talos image factory](https://github.com/siderolabs/image-factory) with the options required to run [Longhorn storage](https://longhorn.io/docs/1.9.0/advanced-resources/os-distro-specific/talos-linux-support/) and returns it in a URL which Talos can use in its config. Note: You must use the 


### Generate config files, apply, and bootstrap:
```bash
talhelper genconfig
talhelper gencommand apply --extra-flags --insecure | bash
talhelper gencommand bootstrap | bash
talhelper gencommand kubeconfig | bash
```



## Useful commands reference:
```bash
talhelper gencommand apply --extra-flags --insecure | bash
```
- Generate commands to apply the talhelper-generated configs for each node to the respective nodes, and pipe the commands into bash to automatically apply them.  

```bash
talhelper gencommand upgrade
```
Generates cluster upgrade command based off the specified talos image


```bash
talhelper gencommand bootstrap | bash
```
- This is just another way to bootstrap the cluster without having to enter a node ip

```bash
talosctl config merge ./clusterconfig/talosconfig 
```
- Merges the cluster config into the main talos config file (~/.talos/config)

```bash
talosctl kubeconfig --nodes <control-plane-IP>
```
- Once the cluster is bootstrapped, target it with kubeconfig (and eveything which uses kubeconfig targeting info)

```bash
talhelper gencommand kubeconfig | bash
```
- Another way to get the cluster info into kubeconfig without specifying a node

```bash
talhelper gencommand apply --extra-flags --mode=reboot | bash
```
Apply the new config and reboot all nodes.

### Destroy the cluster and reset each node to factory stock:
```bash
talhelper gencommand reset --extra-flags --reboot --extra-flags --graceful=false --extra-flags --wait=false | bash
```
- Resets all nodes to stock. Nukes the cluster. Have backups if you care about what was on there obviously.



