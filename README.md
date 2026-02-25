# tt-device-plugin

Kubernetes Device Plugin for [Tenstorrent](https://tenstorrent.com) AI accelerators.

Exposes Tenstorrent NPU/AI accelerator cards to Kubernetes workloads via the standard [Device Plugin API](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/).
Forked from [squat/generic-device-plugin](https://github.com/squat/generic-device-plugin) and pre-configured for Tenstorrent hardware.

[![Build Status](https://github.com/squat/generic-device-plugin/workflows/CI/badge.svg)](https://github.com/squat/generic-device-plugin/actions?query=workflow%3ACI)

## Supported Hardware

| Generation | Model | Chips per card | Device file |
|---|---|---|---|
| Grayskull | e75, e150 | 1 | `/dev/tenstorrent/N` |
| Wormhole | n150 | 1 | `/dev/tenstorrent/N` |
| Wormhole | n300, n300s | 2 (L+R, Ethernet-linked) | `/dev/tenstorrent/N` |
| BlackHole | p100, p150 | 1 | `/dev/tenstorrent/N` |

All models expose devices via `/dev/tenstorrent/N`. Each device file represents one allocatable unit:
- **Single-chip cards** (e75, e150, n150, p100, p150): one device file = one chip.
- **Dual-chip cards** (n300, n300s): one device file = one card (both chips, L+R, accessible via the left chip's PCIe connection).

The plugin supports up to 8 devices per node (`/dev/tenstorrent/0–7`). Device files that do not exist on a node are automatically excluded from the advertised capacity.

## Kubernetes Resource

```
tenstorrent.com/device
```

## Quick Start

Deploy the DaemonSet to your cluster:

```shell
kubectl apply -f https://raw.githubusercontent.com/bricksum/tt-device-plugin/main/manifests/tenstorrent/daemonset.yaml
```

Verify that devices are registered on the node:

```shell
kubectl describe node <node-name> | grep tenstorrent
```

Expected output (example with 4 devices):

```
tenstorrent.com/device:  4
```

## Usage

Request Tenstorrent devices in a Pod spec using the `tenstorrent.com/device` resource:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tt-workload
spec:
  containers:
  - name: workload
    image: <your-image>
    resources:
      limits:
        tenstorrent.com/device: "1"
```

The requested device file is mounted into the container at the same path (`/dev/tenstorrent/N`).

### Examples

**Single device (1 card):**
```yaml
resources:
  limits:
    tenstorrent.com/device: "1"
```

**Multiple devices (e.g. 4 N300S cards for a mesh workload):**
```yaml
resources:
  limits:
    tenstorrent.com/device: "4"
```

**Node selector for Tenstorrent nodes** (recommended when the cluster has mixed hardware):
```yaml
spec:
  nodeSelector:
    tenstorrent.com/device: "true"   # set this label on Tenstorrent nodes
  containers:
  - name: workload
    resources:
      limits:
        tenstorrent.com/device: "1"
```

## How It Works

The plugin is a DaemonSet running in the `kube-system` namespace.
It uses [squat/generic-device-plugin](https://github.com/squat/generic-device-plugin) to watch `/dev/tenstorrent/0–7` and registers each existing device file as an allocatable unit under the `tenstorrent.com` domain via the Kubernetes Device Plugin gRPC API.

When a Pod requests `tenstorrent.com/device: N`, the scheduler selects a node with sufficient capacity, and the kubelet mounts the allocated device files into the container with the appropriate permissions.

## Requirements

- Kubernetes 1.26+
- [TT-KMD](https://github.com/tenstorrent/tt-kmd) kernel driver installed on the node
- Device files present at `/dev/tenstorrent/`

## Manifest

The DaemonSet manifest is at [`manifests/tenstorrent/daemonset.yaml`](manifests/tenstorrent/daemonset.yaml).

The container image used is [`squat/generic-device-plugin`](https://hub.docker.com/r/squat/generic-device-plugin) (upstream, unmodified). Only the configuration — device paths and resource domain — is customized in this fork.

## Upstream

This repository is a fork of [squat/generic-device-plugin](https://github.com/squat/generic-device-plugin).
For generic usage, advanced configuration options (`--device`, `--config`, USB devices, count, optional paths), and the full flag reference, see the upstream repository.
