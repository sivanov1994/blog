---
date: 2026-01-18T00:00:00+00:00
draft: false
title: "Migrating from K3s to Talos Linux"
description: "Installing Talos Linux to replace a K3s cluster with immutable infrastructure"
authors: ["Simeon Ivanov"]
tags: ["kubernetes", "homelab", "talos", "k3s"]
---

I ran K3s on Ubuntu for over a year. The main problem was configuration drift—SSH into a node to fix something, install a debug package, edit a config file directly. Six months later, the cluster works but I can't reproduce the setup.

Talos Linux is different. It's an operating system designed specifically for Kubernetes with no interactive shell and no SSH support. This greatly reduces the attack surface and eliminates the possibility of manual configuration changes. Every modification goes through a versioned configuration file applied via API.

This guide covers installing Talos Linux v1.11.5 on bare metal hardware for a single-node homelab cluster.

## Prerequisites

Before starting, you'll need:

- USB drive (16GB minimum)
- Bare metal machine with UEFI support
- Network connectivity during installation
- Workstation with `talosctl` installed

Install talosctl on your workstation:

```bash
curl -Lo /usr/local/bin/talosctl https://github.com/siderolabs/talos/releases/download/v1.11.5/talosctl-linux-amd64
chmod +x /usr/local/bin/talosctl
```

## Download ISO with iSCSI Extension

I'll be setting up Longhorn for external storage disks, which requires the image to include the `iscsi-tools` extension.

Visit the [Talos Image Factory](https://factory.talos.dev/) to generate a custom ISO. For bare metal installations with iSCSI support, download the pre-built image with the iSCSI schematic:

```bash
wget https://factory.talos.dev/image/c9078f9419961640c712a8bf2bb9174933dfcf1da383fd8ea2b7dc21493f8bac/v1.11.5/metal-amd64.iso
```

This schematic ID includes the `iscsi-tools` extension. You can verify the extensions included by checking the schematic details on the factory website.

## Create Bootable USB

First, identify your USB device. Be careful here—writing to the wrong device will destroy data.

```bash
lsblk
```

Look for your USB drive in the output. It's usually `/dev/sdb` or `/dev/sdc`, but verify the size matches your USB drive.

Write the ISO to the USB device:

```bash
sudo dd if=metal-amd64.iso of=/dev/sdb bs=4M status=progress oflag=sync
```

*Replace `/dev/sdb` with your actual USB device path. This will erase everything on the USB drive.*

## Boot and Generate Config

Insert the USB drive into your target machine and boot from it. You may need to enter the BIOS boot menu (usually F12, F11, or ESC during startup).

Talos will boot into maintenance mode and display its IP address on the screen:

```
Talos is running in maintenance mode
API endpoint: https://192.168.1.100:50000
Use 'talosctl' to configure the machine.
```

Note this IP address—you'll use it for the installation. The machine is now running Talos from RAM without touching the installed disk.

From your workstation, set up environment variables and generate the cluster configuration:

```bash
export NODE_IP=192.168.1.100  # Use the IP shown on screen
export CLUSTER_NAME=homelab

talosctl gen config $CLUSTER_NAME https://$NODE_IP:6443 \
  --install-disk /dev/nvme0n1
```

*Replace `/dev/nvme0n1` with your target installation disk. You can check available disks by running `talosctl get disks --insecure --nodes $NODE_IP` before generating the config.*

This command creates three files in your current directory:

- `controlplane.yaml` - Configuration for the control plane node
- `worker.yaml` - Configuration template for future worker nodes
- `talosconfig` - Authentication credentials for talosctl

### Configure Single-Node Cluster

By default, Kubernetes doesn't schedule application workloads on control plane nodes. For a single-node homelab cluster, you need to allow this.

Edit `controlplane.yaml` to uncomment the scheduling setting:

```bash
sed -i 's/# allowSchedulingOnControlPlanes: true/allowSchedulingOnControlPlanes: true/' controlplane.yaml
```

Alternatively, open the file in your editor and find the line `# allowSchedulingOnControlPlanes: true` under the `cluster:` section, then uncomment it by removing the `#`.

## Install to Disk

Now apply the configuration to install Talos to the disk. Note the `--insecure` flag—this is temporary and only used during initial installation when the machine doesn't have certificates yet.

```bash
talosctl apply-config --insecure \
  --nodes $NODE_IP \
  --file controlplane.yaml
```

Talos will:
1. Partition and format the installation disk
2. Install the Talos system image
3. Configure the node with your settings
4. Automatically reboot from the installed system

This takes about 2 minutes. You can remove the USB drive once the installation starts.

### Configure Secure Access

After the node reboots (wait about 2 minutes), configure talosctl to authenticate with the cluster:

```bash
export TALOSCONFIG=~/talos-config/talosconfig

talosctl config endpoint $NODE_IP
talosctl config node $NODE_IP
```

Verify the connection works:

```bash
talosctl version
```

You should see both client and server version information. The `--insecure` flag is no longer needed—all communication now uses mutual TLS authentication.

## Bootstrap Kubernetes

Initialize the Kubernetes cluster by bootstrapping etcd:

```bash
talosctl bootstrap
```

**Important:** Run this command only once. Running `bootstrap` multiple times will corrupt the etcd database and require a full reinstall.

The bootstrap process:
- Initializes the etcd database
- Generates cluster certificates
- Starts the Kubernetes control plane components
- Deploys CoreDNS and other system services

Wait 2-3 minutes for the control plane to start, then retrieve your kubeconfig:

```bash
talosctl kubeconfig
```

This saves the kubeconfig to `~/.kube/config` by default. You can now use `kubectl` to interact with the cluster:

```bash
kubectl get nodes
```

The node should transition to `Ready` status within a few minutes. If it stays `NotReady`, check the kubelet logs:

```bash
talosctl logs kubelet
```

## Verify iSCSI Extension

Confirm that the iSCSI tools extension loaded correctly:

```bash
talosctl get extensions
```

You should see output like:

```
NAME          VERSION
iscsi-tools   v0.2.0
```

You can also verify the iSCSI kernel modules are loaded:

```bash
talosctl read /proc/modules | grep iscsi
```

This confirms the image includes iSCSI support needed for storage solutions like Longhorn.

## Verify System Pods

Check that all Kubernetes system components are running:

```bash
kubectl get pods -n kube-system
```

You should see pods for:
- `coredns` - DNS service
- `kube-apiserver` - API server
- `kube-controller-manager` - Controller manager
- `kube-scheduler` - Scheduler
- `kube-proxy` - Network proxy
- CNI pods (Flannel by default)

All pods should show `Running` status.

## What Changed

**Before (K3s on Ubuntu):**
- SSH access to nodes for debugging
- Install packages with `apt`
- Edit config files directly on nodes
- Configuration drift accumulates over time
- Manual backup and documentation needed
- Security requires SSH hardening

**After (Talos):**
- No SSH daemon or shell access
- API-only management via `talosctl`
- All changes through versioned config files
- Configuration drift impossible
- System state fully described by config
- Minimal attack surface by design

The cluster is now completely reproducible. I can rebuild it exactly by applying the same `controlplane.yaml` configuration file.

## Next Steps

This installation gives you a working Kubernetes cluster, but there's more to set up:

- **Storage**: Deploy Longhorn for persistent volumes
- **GitOps**: Set up Flux CD for declarative application management
- **Ingress**: Configure Traefik for external access
- **Certificates**: Set up cert-manager for automatic TLS
- **Applications**: Migrate workloads from K3s

I'll cover these topics in future posts.

---

*Questions about Talos or migrating from K3s? Feel free to reach out.*
