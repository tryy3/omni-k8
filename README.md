# Siderolabs Omni Example

This example shows how to manage a Talos Kubernetes cluster with Sidero Labs' Omni.
It deploys a Talos Kubernetes cluster using Omni, with the following tooling:

* Cilium for CNI & Hubble UI for observability
* ArgoCD for application management
* Rook Ceph for persistent volume management
* Prometheus & Grafana for monitoring

## Prereqs

An [Omni account](https://signup.siderolabs.io/), and some machines registered to it.
How the machines are started and joined to the Omni instance are not covered in this README, but [documentation is available](https://omni.siderolabs.com/tutorials/getting_started/).
With the default configuration, a minimum of 6 machines, 3 of which with additional block devices for persistent storage.

This example uses [Machine Classes](https://omni.siderolabs.com/how-to-guides/create-a-machine-class) called `omni-contrib-controlplane` and `omni-contrib-workers`.
How they are defined is entirely dependent on the infrastructure available, they would need to be configured on the Omni instance.

Lastly, `omnictl` the Omni CLI tool would also be needed.
See the [How-to](https://omni.siderolabs.com/how-to-guides/install-and-configure-omnictl) on how to obtain and configure it.

## Usage

Once the required machines are registered to Omni and machine classes have been configured, simply run

```bash
omnictl cluster template sync --file cluster-template.yaml
```

Omni will then being to allocate your machines, install Talos, and configure and bootstrap the cluster.

This setup makes use of the [Omni Workload Proxy](https://omni.siderolabs.com/how-to-guides/expose-an-http-service-from-a-cluster) feature,
which allows access to the HTTP front end services *without* the need of a separate external Ingress Controller or LoadBalancer.
Additionally, it leverages Omni's built-in authentication to protect the services, even those services that don't support authentication themselves.

## Applications

Applications are managed by ArgoCD, and are defined in the `apps` directory.
The first subdirectory defines the namespace and the second being the application name.
Applications can be made of Helm charts, Kustomize definitions, or just Kubernetes manifest files in YAML format.

## Extending

1. Commit the contents from the `omni` directory to a new repository
2. Configure ArgoCD to use that repository [bootstrap-app-set.yaml](apps/argocd/argocd/bootstrap-app-set.yaml)
3. Regenerate the ArgoCD bootstrap cluster manifest patch [argocd.yaml](infra/patches/argocd.yaml) (instructions can be found at the top of that file).
4. Commit and push these changes to a hosted git repository the Omni instance has access to.
5. Create a cluster with Omni as described above.

## Installation
### Kubectl
Install through package manager (`yay -S kubectl`)

### Omnictl / Talosctl
Download binary through dashboard and move to path (i.e. `/usr/bin/local/omnictl`)

#### Configure
https://docs.siderolabs.com/omni/getting-started/install-and-configure-omnictl
Download config from dashboard - move omniconfig.yaml to ~/.config/omni/config
`mv omniconfig.yaml ~/.config/omni/config`

The omniconfig.yaml should be renamed to config, so config is the yaml file and not a folder.

### kubectl
https://docs.siderolabs.com/omni/getting-started/use-kubectl-with-omni

Omni uses oidc to login, so you first need to install oidc-login plugin https://github.com/int128/kubelogin#getting-started
Easiest can be to download the plugin from github releases, extract the binary into path and rename to `kubectl-oidc_login`

After installation you can try the command `kubectl oidc-login` to make sure it's properly installed

#### configuration
Download from dashboard. To test if everything is working you can run `kubectl --kubeconfig ./talos-default-kubeconfig.yaml get nodes`

Move the config to a more centralised path for example `~/.config/kube` and add the file to your KUBECONFIG env
`export KUBECONFIG=$KUBECONFIG:/home/tryy3/.config/kube/talos-kubeconfig`
https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/

### Argocd
https://argo-cd.readthedocs.io/en/stable/getting_started/
Download from github releases and move to path `/usr/local/bin/argocd/`

To quickly access argocd locally you can port-forward the svc port.
`kubectl port-forward -n argocd svc/argocd-server 8080:443`
Then access via `http://localhost:8080`
