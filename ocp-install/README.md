# Baremetal OCP Compact Cluster

3-node compact cluster (master nodes schedule workloads) deployed on HP EliteDesk G6 mini PCs.

|             |                               |
| ----------- | ----------------------------- |
| Cluster     | `ocp.clamie.au`                |
| API VIP     | `192.168.40.2`                |
| Ingress VIP | `192.168.40.3`                |
| Subnet      | `192.168.40.0/27`             |
| DNS / NTP   | `192.168.30.2` (idm.clamie.au) |

| Hostname  | IP            | MAC               |
| --------- | ------------- | ----------------- |
| node-01 | 192.168.40.11 | 00:e0:4c:68:0c:07 |
| node-02 | 192.168.40.12 | 00:e0:4c:68:0b:c9 |
| node-03 | 192.168.40.13 | 00:e0:4c:68:10:75 |

## Prerequisites

### 1. Pull secret

Replace `REDACTED` in [install-config.yaml](install-config.yaml) with your pull secret from
https://console.redhat.com/openshift/install before generating the ISO.

### 2. DNS records

Add the following records to IdM (`idm.clamie.au`) before starting the install:

```
api.ocp.clamie.au        A  192.168.40.2
api-int.ocp.clamie.au    A  192.168.40.2
*.apps.ocp.clamie.au     A  192.168.40.3
node-01.ocp.clamie.au  A  192.168.40.11
node-02.ocp.clamie.au  A  192.168.40.12
node-03.ocp.clamie.au  A  192.168.40.13
```

Create reverse zone and corresponding PTR records.

```bash
ipa dnszone-add 40.168.192.in-addr.arpa.

ipa dnsrecord-add 40.168.192.in-addr.arpa. 2  --ptr-rec api.ocp.clamie.au
ipa dnsrecord-add 40.168.192.in-addr.arpa. 3  --ptr-rec ingress.ocp.clamie.au
ipa dnsrecord-add 40.168.192.in-addr.arpa. 11 --ptr-rec node-01.ocp.clamie.au
ipa dnsrecord-add 40.168.192.in-addr.arpa. 12 --ptr-rec node-02.ocp.clamie.au
ipa dnsrecord-add 40.168.192.in-addr.arpa. 13 --ptr-rec node-03.ocp.clamie.au
```

Verify newly create records
```bash
# Forward
dig api.ocp.clamie.au
dig node-01.ocp.clamie.au

# Reverse
dig -x 192.168.40.2
dig -x 192.168.40.11
```

## Create `MachineConfig` object using Butane

The Butane configuration file found in `ocp-install/99-master-custom.bu` configures master 
nodes to add `nomodeset` to the kernel arguments. In addition, it configures 
`idm.clamie.au` as the NTP server.

To generate the `MachineConfig` use the command below. Adding it to the `./openshift` directory
will include the MachineConfig during the installation.

```bash
butane ./ocp-install/99-master-custom.bu -o ./ocp-install/openshift/99-master-custom.yaml
```

## Generating the ISO

> **Warning:** `openshift-install` consumes and deletes `install-config.yaml` and
> `agent-config.yaml` when generating the ISO. This directory serves as the source
> of truth — copy it to a working directory first.

```bash
# Copy to a fresh working directory
cp -r ocp-install /tmp/ocp-install

# Generate the ISO
openshift-install agent create image --dir /tmp/ocp-install
```

### Add kernel arguments to ISO for live boot display (nomodeset)

The `openshift/99-master-kernelargs-nomodeset.yaml` manifest applies `nomodeset` to the
installed system. To also apply it during the live ISO boot (needed for FlexIO HDMI on
HP EliteDesk G6), patch the ISO after generation:

```bash
coreos-installer iso kargs modify /tmp/ocp-install/agent.x86_64.iso --append nomodeset
```

If `coreos-installer` is not installed locally:

```bash
podman run --rm -v /tmp/ocp-install:/data:z \
  quay.io/coreos/coreos-installer:release \
  iso kargs modify /data/agent.x86_64.iso --append nomodeset
```

## Installation

1. Boot all three nodes from `agent.x86_64.iso` (USB or virtual media).
   - `rendezvousIP` is `192.168.40.11` (node-01) — boot this node first or simultaneously.
   - OS will be installed to `/dev/nvme0n1` on each node.

2. Monitor bootstrap completion:

```bash
openshift-install agent wait-for bootstrap-complete --dir /tmp/ocp-install --log-level=info
```

3. Monitor full install completion:

```bash
openshift-install agent wait-for install-complete --dir /tmp/ocp-install --log-level=info
```

The install typically takes 45-60 minutes. The kubeconfig is written to
`/tmp/ocp-install/auth/kubeconfig` on completion.

## Post-install access

```bash
export KUBECONFIG=/tmp/ocp-install/auth/kubeconfig
oc get nodes
oc get clusterversion
```
