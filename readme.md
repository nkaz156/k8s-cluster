# Introduction:
This repo provides declarative configuration management for my kubernetes cluster, which is still in its infancy, but some day will have a lot of cool services. Currently, I have my GitOps repos separated into this one, a private repo to hold sops-encrypted secrets, and a private repo which holds my talhelper configuration. 

I currently use Cilium as a CNI and kube-proxy replacement, and Flux to manage deployment of software. I also uses Cilium as a GatewayAPI provider and as a load balancer via L2 announcements (I'll get around to BGP sometime when I get a better router).

The goal is to have a pseudo-production cluster that can hum along for months on end without needing maintenance, while sipping power. I'm currently running on a set of three thin clients from Dell and Lenovo, each of which runs proxmox and hosts a Talos worker and control plane. At some point, I'd like to get more thin clients and move away from the additional resource demands of a hypervisor, but this setup works well for now, and it's very easy to change configurations or fix things I break along the way.

I am not a software professional or DevOps engineer, so while I try to keep everything in line with kubernetes/cybersecurity best practices and err towards overhardening, this is a complex project with many dependencies. I am currently not exposing anything to the internet and use a VPN ([Netbird](https://netbird.io/), it's amazing) to access hosted services remotely.

## Prerequisites:
- [Mise](https://github.com/jdx/mise) manages all necessary CLI tools; additionally, it automatically sets environment variables when you `cd` into the repository. 
- I use homebrew to manage packages wherever possible (`brew install mise`), but all installation methods are listed on the [Mise website](https://mise.jdx.dev/installing-mise.html)
- You'll have to [activate mise for your shell](https://mise.jdx.dev/getting-started.html#activate-mise) and trust the repo before mise will install packages:
    ```bash
    mise trust
    mise install
    ```
## Bootstrap tools: 
I use Task to automate bootstrapping the cluster 




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
 - onedr0p's [cluster-template](https://github.com/onedr0p/cluster-template) and [home-ops](https://github.com/onedr0p/home-ops) has also informed my service selection and inspired me to implement taskfiles or just in the future to automate the bootstrap process.
 - [DaTosh Blog](https://blog.kammel.dev/post/k8s_home_lab_2025_01/) gives a simple, digestible breakdown of setting up flux


## Helpful resources:
- Budiman Jojo's [Talhelper](https://github.com/budimanjojo/talhelper) is a great way to rapidly bootstrap a Talos cluster and automate away a lot of work.
- [Kubesearch](https://kubesearch.dev/) - search through all github repos tagged `k8s-at-home` or `kubesearch`
- [Artifacthub](https://artifacthub.io/) - large, public helm chart aggregator
- [yq](https://github.com/mikefarah/yq) - makes working with multiple yaml files a lot easier
- [DeepWiki](https://deepwiki.org/) is a great tool to explore how anything in any git repo works

## Lessons Learned:
- Set up the repo such that the path you give to flux only contains kustomizations. This is important to having control over flux's rollout order.
- Make sure the install sequencing installs deployments and their corresponding CRDs before it tries to create objects of the corresponding custom resources. Example: Flux got stuck trying to create a `ClusterIssuer` before installing `cert-manager`. This is what prompted me to restructure the repo into its current form. Before, I had all apps in flux's main watched path folder.
 - Just send it, it's GitOpsified for a reason. Disaster recovery is pretty simple

# More Documentation:
[Docs directory:](./docs/README.md)