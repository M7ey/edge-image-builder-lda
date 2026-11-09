this example contains an EIB config file with helm, multiple nodes and multiple apps deployment.
it refers to an OCI helm registry with a simple nginx configuration inside it.
See the comments to have more details.
This Example gives a full airgapp example

## Important folders

| Folder / file | Content |
| --- | --- |
| `network/` | Networking configuration of each node. Each file is named after the node hostname (`node1.yaml`, `node2.yaml`, `node3.yaml`), and the MAC address inside lets each machine find its own configuration. |
| `kubernetes/manifests/` | Kubernetes manifest (`apps-pool.yaml`) with an IP pool used by MetalLB. When requesting a service with an external IP (type `LoadBalancer`), MetalLB gives an IP from the pool. This pool is separate from the `api-ip` pool created by EIB for the API VIP. |
| `kubernetes/helm/values/` | All the values files used by helm deployments. |
| `kubernetes/config/server.yaml` | RKE2 configuration applied to all server nodes (cluster token, etcd snapshots). |

```text
eib/
├── definition.yaml
├── base-images/
├── network/
│   ├── node1.yaml
│   ├── node2.yaml
│   └── node3.yaml
└── kubernetes/
    ├── config/
    │   └── server.yaml
    ├── manifests/
    │   └── apps-pool.yaml
    └── helm/
        └── values/
            └── demo-nginx.yaml
```

## Edge Image Builder definition file

### `kubernetes` section

- Version of RKE2.
- `apiVIP`: served by MetalLB and the Endpoint Copier Operator, automatically added by EIB for multi-node clusters.
- List of nodes, with one initializer node.

### `helm` section

Defines all helm charts embedded in the ISO:

- `suse-storage`
- `system-upgrade-controller`
- `demo-nginx`

Every element contains a name, a version, a repository name, its target namespace and also a values file when needed.

### `repositories` section

Three repositories here:

| Name | Usage |
| --- | --- |
| `gitlab` | `demo-nginx` chart (OCI registry of the GitLab project) |
| `appco` | `suse-storage` chart (SUSE Application Collection) |
| `rancher-charts` | `system-upgrade-controller` chart |

### `embeddedArtifactRegistry` section

Used to pull the required images during the image build and host them in the ISO. EIB downloads the images during the build and serves them on the nodes through the embedded artifact registry, so nothing is pulled from the internet on site.

This section has two parts:

- **`registries`**: the authentication part, only for registries that need credentials. `dp.apps.rancher.io` is the only one used here, I am using the app catalog from SUSE.
- **`images`**: manual list of images not declared in the charts (pulled at runtime by Longhorn, `kuberlr-kubectl` for SUC). Images referenced in the charts (e.g. nginx) are detected and embedded automatically.

## Secrets

Credentials (appco, gitlab) are not stored in the definition file. They are injected with `envsubst` from a `secrets.env` file kept out of Git.

```bash
source secrets.env
envsubst < definition.tpl.yaml > definition.yaml
podman run --privileged --rm -it -v $PWD:/eib \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
  build --definition-file definition.yaml
```

Classic build can be done with:
```bash
podman run --privileged --rm -it -v YOURPATH:/eib registry.suse.com/edge/3.7/edge-image-builder:1.3.4 build --definition-file definition.yaml
```
