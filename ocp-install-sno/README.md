# Single Node OpenShift — Agent-Based Install

SNO cluster with FIPS and TPM v2 boot disk encryption enabled.

## Prerequisites

- `openshift-install-fips` binary matching the target OCP version
- `nmstate` RPM for the `nmstatectl` command
- A valid pull secret from [console.redhat.com](https://console.redhat.com/openshift/install/pull-secret) (if internet connected)
- SSH public key for node access
- Target node with a TPM 2.0 chip

## Configuration

### 1. Replace Credentials and Network Configuration

Before generating the ISO, update [install-config.yaml](install-config.yaml) with your pull secret and SSH key:

```yaml
pullSecret: 'REPLACE_ME'
sshKey: 'REPLACE_ME'
```

Update [agent-config.yaml](agent-config.yaml) with the correct MAC address and rendezvous IP for the target node.

### 2. Duplicate Configuration Directory

Create a copy of the working configuration directory (the installer consumes and removes them):

```bash
cp -r ocp-install-sno ~/ocp-install-sno
```

### 3. Generate and Modify Cluster Manifest

```bash
openshift-install-fips agent create cluster-manifests --dir ~/ocp-install-sno
```

Modify `~/ocp-install-sno/cluster-manifests/agent-cluster-install.yaml`

```yaml
spec:
  ...
  diskEncryption:
    enableOn: all        # Options: all, masters, workers, none
    mode: tpmv2
```

> **Note:** FIPS is already set to `true` in `install-config.yaml`. The node must have a TPM 2.0 chip accessible during boot.

### 4. Generate ISO and Boot Nodes

```bash
openshift-install-fips agent create image --dir ~/ocp-install-sno
```

Write the ISO to a USB drive or mount it via your platform's virtual media (IPMI/iDRAC/iLO):

Boot the target node from the ISO. The agent will configure networking, apply FIPS mode, set up TPM-backed disk encryption, and begin the OCP install automatically.

### Monitor Installation

```bash
openshift-install-fips agent wait-for bootstrap-complete --dir ~/ocp-install-sno --log-level=info
openshift-install-fips agent wait-for install-complete   --dir ~/ocp-install-sno --log-level=info
```

The `kubeconfig` is written to `~/ocp-install-sno/auth/kubeconfig` on completion.

---

## Validation

### FIPS

```bash
oc get nodes

oc debug node/<node_name>

chroot /host

cat /proc/sys/crypto/fips_enabled
```

### Disk Encryption (TPM)

```bash
oc debug node/<node_name>

chroot /host

# Confirm root device is LUKS-encrypted
lsblk -o NAME,TYPE,FSTYPE

# Verify Clevis TPM binding
clevis luks list -d /dev/vda4
```
