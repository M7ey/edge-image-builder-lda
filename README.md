# edge-image-builder-lda

A repo to host some **EIB** (Edge Image Builder) + **Elemental** work.

## Prerequisites

- A Rancher management cluster with `kubectl` access
- `helm`
- `podman`

## Install Elemental on Rancher

Install the CRDs first, then the operator:

```bash
helm install --create-namespace -n cattle-elemental-system \
  elemental-operator-crds \
  oci://registry.suse.com/rancher/elemental-operator-crds-chart \
  --version 1.9.2

helm install -n cattle-elemental-system \
  elemental-operator \
  oci://registry.suse.com/rancher/elemental-operator-chart \
  --version 1.9.2
```

## Workflow

### 1. Create a MachineRegistration

Create the `MachineRegistration` in the `fleet-default` namespace (from the Elemental UI in Rancher or with a manifest).

### 2. Download the Elemental registration config

```bash
REGISTRATION_NAME=<eib-registration-name>
OUTPUT_DIR=<your-path>

REGISURL=$(kubectl get machineregistration "$REGISTRATION_NAME" \
  -n fleet-default \
  -o jsonpath='{.status.registrationURL}')
echo "Registration URL: $REGISURL"

curl -k "$REGISURL" -o "$OUTPUT_DIR/elemental_config.yaml"
```

> **Note:** `-k` skips TLS certificate verification, which is fine for a lab with self-signed certificates.

### 3. Build the ISO with EIB

Use the `eib-config.yaml` definition file to build the ISO. This definition is tailored to my Proxmox setup to simulate an Edge environment.

```bash
podman run --privileged --rm -it \
  -v <your-path>:/eib \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
  build --definition-file eib-config.yaml
```

> **Tip:** EIB expects the registration file at `elemental/elemental_config.yaml` inside the mounted directory.

### 4. Boot the machine

Boot the ISO on your edge environment. The machine registers itself and shows up as a `MachineInventory`.

### 5. Create a MachineInventorySelectorTemplate

Create the `MachineInventorySelectorTemplate` matching the label(s) of the machines you want to use.

### 6. Deploy a cluster

Deploy a cluster using the machines available in the inventory that match the selected label.
