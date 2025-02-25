# Home Integration Cluster

At a high level this project makes use of [makejinja](https://github.com/mirkolenz/makejinja) to read in a [configuration file](./config.sample.yaml) which renders out templates to install and manage a cluster.

- [Talos Linux](https://github.com/siderolabs/talos)

    - **Components:** [argo](https://github.com/argoproj/argo-cd), [cilium](https://github.com/cilium/cilium), [cert-manager](https://github.com/cert-manager/cert-manager), [spegel](https://github.com/spegel-org/spegel), [reloader](https://github.com/stakater/Reloader), and [openebs](https://github.com/openebs/openebs)
    - Dev env managed w/ [mise](https://mise.jdx.dev/)
    - Workflow automation w/ [GitHub Actions](https://github.com/features/actions)
    - Dependency automation w/ [Renovate](https://www.mend.io/renovate)


### Stage 1: Machine Preparation

1. Create ISO from [Talos Linux Image Factory](https://factory.talos.dev). Be sure to only choose the **bare-minimum system extensions** as some might require additional configuration and prevent Talos from booting without it. You can always add system extensions after Talos is installed and working.

2. Download a Talos Linux ISO file. Make sure to note the **schematic ID**.

3. Build cluster nodes and boot the Talos ISO.

### Stage 2: Local Workstation

1. Create a new **public** repository by cloning this repo.

2. Use `git clone` to download **the repo you just created** to your local workstation and `cd` into it.

3. **Install** and **activate** [mise](https://mise.jdx.dev/) for the **required** CLI tools:

    ```sh
    mise trust
    mise install
    mise run deps
    ```

### Stage 3: Template Configuration

1. Generate the `config.yaml` from the [config.sample.yaml](./config.sample.yaml) configuration file:

    ```sh
    task init
    ```

2. Fill out the `config.yaml` configuration file using the comments in that file as a guide.

3. Template out all the configuration files:

    ```sh
    task configure
    ```

4. Push your changes to git:

   📍 _**Verify** all the `./kubernetes/**/*.sops.*` files are **encrypted** with SOPS_

    ```sh
    git add -A
    git commit -m "chore: initial commit :rocket:"
    git push
    ```

### Stage 4: Bootstrap Talos, Kubernetes, and Argo

1. Install Talos:

   📍 _It might take a while for the cluster to be setup (10+ minutes is normal). During which time you will see a variety of error messages like: "couldn't get current server API group list," "error: no matching resources found", etc. 'Ready' will remain "False" as no CNI is deployed yet. **This is a normal.** If this step gets interrupted, e.g. by pressing <kbd>Ctrl</kbd> + <kbd>C</kbd>, you likely will need to [reset the cluster](#-reset) before trying again_

    ```sh
    task bootstrap:talos
    ```

2. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: add talhelper encrypted secret :lock:"
    git push
    ```

3. Install cilium, coredns, spegel, argo and sync the cluster to the repository state:

    ```sh
    task bootstrap:apps
    ```

4. Watch the rollout of your cluster happen:

   📍 _Depending on the features you choose a successful rollout will include pods being deployed into the **cert-manager, argo-system, network and openebs-system** namespaces_

    ```sh
    watch kubectl get pods --all-namespaces
    ```

## 📣 Argo w/ Cloudflare post installation

### 🌐 Public DNS

> [!TIP]
> Use the `external` ingress class to make applications public to the internet.

The `external-dns` application created in the `networking` namespace will handle creating public DNS records. By default, `echo-server` and the `argo` are the only subdomains reachable from the public internet. In order to make additional applications public you must set set the correct ingress class name and ingress annotations like in the HelmRelease for `echo-server`.

### 🏠 Home DNS

> [!TIP]
> Use the `internal` ingress class to make applications private to your network. If you're having trouble with internal DNS resolution check out [this](https://github.com/onedr0p/cluster-template/discussions/719) GitHub discussion.

`k8s_gateway` will provide DNS resolution to external Kubernetes resources (i.e. points of entry to the cluster) from any device that uses your home DNS server. For this to work, your home DNS server must be configured to forward DNS queries for `${cloudflare.domain}` to `${cloudflare.gateway_vip}` instead of the upstream DNS server(s) it normally uses. This is a form of **split DNS** (aka split-horizon DNS / conditional forwarding).

... Nothing working? That is expected, this is DNS after all!

### 📜 Certificates

By default this template will deploy a wildcard certificate using the Let's Encrypt **staging environment**, which prevents you from getting rate-limited by the Let's Encrypt production servers if your cluster doesn't deploy properly (for example due to a misconfiguration). Once you are sure you will keep the cluster up for more than a few hours be sure to switch to the production servers as outlined in `config.yaml`.

### 🪝 Github Webhook

By default Argo will periodically check your git repository for changes. In order to have Argo reconcile on `git push` you must configure Github to send `push` events to Argo.

> [!IMPORTANT]
> This will only work after you have switched over certificates to the Let's Encrypt Production servers.

1. Full URL with the webhook path appended:
    ```text
    https://argo.${cloudflare.domain}/api/webhook
    ```

2. Navigate to the settings of your repository on Github, under "Settings/Webhooks" press the "Add webhook" button. Fill in the webhook URL and your `${github.webhook_token}` secret in `config.yaml`, Content type: `application/json`, Events: Choose Just the push event, and save.

## 💥 Reset

There might be a situation where you want to destroy your Kubernetes cluster. The following command will reset your nodes back to maintenance mode, append `--force` to completely format your the Talos installation. Either way the nodes should reboot after the command has sucessfully ran.

```sh
task talos:reset # --force
```

## 🛠️ Talos and Kubernetes Maintenance

### ⚙️ Updating Talos node configuration

> [!IMPORTANT]
> Ensure you have updated `talconfig.yaml` and any patches with your updated configuration. In some cases you **not only need to apply the configuration but also upgrade talos** to apply new configuration.

```sh
# (Re)generate the Talos config
task talos:generate-config
# Apply the config to the node
task talos:apply-node IP=? MODE=?
# e.g. task talos:apply-node IP=10.10.10.10 MODE=auto
```

### ⬆️ Updating Talos and Kubernetes versions

> [!IMPORTANT]
> Ensure the `talosVersion` and `kubernetesVersion` in `talconfig.yaml` are up-to-date with the version you wish to upgrade to.

```sh
# Upgrade node to a newer Talos version
task talos:upgrade-node IP=?
# e.g. task talos:upgrade-node IP=10.10.10.10
```

```sh
# Upgrade cluster to a newer Kubernetes version
task talos:upgrade-k8s
# e.g. task talos:upgrade-k8s
```

## 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Argo itself! Merging a PR will cause Argo to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](./.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state. Most of these steps do not include a way to fix the problem as the problem could be one of many different things.

1. Verify all Argo Applications  are up-to-date and in a ready state.

    ```sh
      kubectl get applications -n argo-system
    ```

2. Do you see the pod of the workload you are debugging?

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

3. Check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    ```

4. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

5. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on a NFS server. If you are unable to figure out your problem see the support sections below.

## 🧹 Tidy up

Once your cluster is fully configured and you no longer need to run `task configure`, it's a good idea to clean up the repository by removing the [templates](./templates) directory and any files related to the templating process. This will help eliminate unnecessary clutter from the upstream template repository and resolve any "duplicate registry" warnings from Renovate.

1. Tidy up your repository:

    ```sh
    task template:tidy
    ```

2. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: tidy up :broom:"
    git push
    ```

## 👉 Community Support

- Make a post in this repository's Github [Discussions](https://github.com/ajaykumar4/cluster-template/discussions).
- Start a thread in the `#support` or `#cluster-template` channels in the [Home Operations](https://discord.gg/home-operations) Discord server.

## ❔ What's next

The cluster is your oyster (or something like that). Below are some optional considerations you might want to review.

### DNS

Instead of using [k8s_gateway](https://github.com/ori-edge/k8s_gateway) to provide DNS for your applications you might want to check out [external-dns](https://github.com/kubernetes-sigs/external-dns), it has wide support for many different providers such as [Pi-hole](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/pihole.md), [UniFi](https://github.com/kashalls/external-dns-unifi-webhook), [Adguard Home](https://github.com/muhlba91/external-dns-provider-adguard), [Bind](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/rfc2136.md) and more.

### Storage

The included CSI (openebs in local-hostpath mode) is a great start for storage but soon you might find you need more features like replicated block storage, or to connect to a NFS/SMB/iSCSI server. If you need any of those features be sure to check out the projects like [rook-ceph](https://github.com/rook/rook), [longhorn](https://github.com/longhorn/longhorn), [openebs](https://github.com/openebs/openebs), [democratic-csi](https://github.com/democratic-csi/democratic-csi), [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs),
and [synology-csi](https://github.com/SynologyOpenSource/synology-csi).

### Community Repositories

Community member [@whazor](https://github.com/whazor) created [Kubesearch](https://kubesearch.dev) to allow searching Argo Helm Releases across Github and Gitlab repositories with the `kubesearch` topic.

## 🙌 Related Projects

If this repo is too hot to handle or too cold to hold check out these following projects.

- [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) - _A template for deploying a Talos Kubernetes cluster including Flux for GitOps._
- [khuedoan/homelab](https://github.com/khuedoan/homelab) - _Fully automated homelab from empty disk to running services with a single command._
- [ricsanfre/pi-cluster](https://github.com/ricsanfre/pi-cluster) - _Pi Kubernetes Cluster. Homelab kubernetes cluster automated with Ansible and FluxCD_
- [techno-tim/k3s-ansible](https://github.com/techno-tim/k3s-ansible) - _The easiest way to bootstrap a self-hosted High Availability Kubernetes cluster. A fully automated HA k3s etcd install with kube-vip, MetalLB, and more. Build. Destroy. Repeat._

## ⭐ Stargazers

<div align="center">

<a href="https://star-history.com/#ajaykumar4/cluster-template&Date">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=ajaykumar4/cluster-template&type=Date&theme=dark" />
    <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=ajaykumar4/cluster-template&type=Date" />
    <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=ajaykumar4/cluster-template&type=Date" />
  </picture>
</a>

</div>

## 🤝 Thanks [onedr0p](https://github.com/onedr0p)
