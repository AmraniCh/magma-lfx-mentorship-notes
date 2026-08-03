# Magma Access Gateway (AGW) Deployment with Docker

This guide is for deploying the Docker-based AGW on a VirtualBox VM. The official
guide assumes a bare-metal machine installed from a USB stick; this adapts those
steps to a VM workflow.

## Table of Contents

| Section | Description |
| --- | --- |
| [1. Environment Setup](#1-environment-setup) | VM configuration and OS installation |
| [1.1 Virtual Machine Configuration](#11-virtual-machine-configuration-virtualbox) | VirtualBox settings (adapters, resources) |
| [1.2 OS Installation](#12-os-installation) | Ubuntu 20.04 server install |
| [2. Deployment Steps](#2-deployment-steps) | The full AGW deployment |
| [2.1 Verify the two interfaces](#21-verify-the-two-interfaces) | Confirm both NICs are present |
| [2.2 SSH access from the Windows host](#22-ssh-access-from-the-windows-host) | Port forwarding and SSH setup |
| [2.3 AGW installation](#23-agw-installation) | Run the install script |
| [2.4 Reboot](#24-reboot) | Reboot and fix networking |
| [2.5 Configure the AGW](#25-configure-the-agw) | Point the AGW at an orc8r |
| [2.6 Start and verify](#26-start-and-verify) | Bring up the containers |
| [3. Final Result](#3-final-result) | Container status and interpretation |

## 1. Environment Setup

### 1.1 Virtual Machine Configuration (VirtualBox)

| Setting | Value |
| --- | --- |
| Name | `magma-agw` |
| OS | Ubuntu Server 20.04.6 LTS (Focal Fossa), 64-bit |
| RAM | 4 GB+ |
| CPU | 2+ cores |
| Disk | 40 GB (dynamically allocated) |
| Adapter 1 | NAT -- SGi side (internet + management) |
| Adapter 2 | Internal Network -- S1 side (isolated) |

The AGW requires **two ethernet interfaces** (`enp1s0`/SGi and `enp2s0`/S1), so
two network adapters must be configured before installing.

Issue: after the initial OS install, `ip address` may show only one interface
(`enp0s3`). This means only one network adapter was enabled in VirtualBox. Power
off the VM, enable **Adapter 2** (Attached to: Internal Network) in VM Settings,
then restart.

### 1.2 OS Installation

- Boot the VM from the `ubuntu-20.04.6-live-server-amd64.iso` image.
- Enable **OpenSSH server** during installation.
- Do not select any featured server snaps.

## 2. Deployment Steps

### 2.1 Verify the two interfaces

```bash
ip address
```

Expected output:

- `enp0s3` -- Adapter 1 (NAT), IP `10.0.2.15/24`, state UP
- `enp0s8` -- Adapter 2 (Internal Network), state DOWN, no IP (expected)

### 2.2 SSH access from the Windows host

Add port forwarding in VirtualBox (Adapter 1 > Advanced > Port Forwarding):

| Name | Protocol | Host Port | Guest Port |
| --- | --- | --- | --- |
| ssh | TCP | 2222 | 22 |

Connect from the host:

```bash
ssh -p 2222 amranich@127.0.0.1
```

Issue: SSH may fail with `Bad owner or permissions on C:\Users\.../.ssh/config`.
Windows has the `.ssh/config` file shared with extra users/groups; OpenSSH refuses
configs that are not restricted to the owner. Fix by restricting the file
permissions in PowerShell:

```powershell
$file = "$env:USERPROFILE\.ssh\config"
icacls $file /inheritance:r
icacls $file /grant:r "$($env:USERNAME):(R,W)"
```

Alternative quick workaround: `ssh -F none -p 2222 amranich@127.0.0.1`.

### 2.3 AGW installation

```bash
sudo -i
mkdir -p /var/opt/magma/certs
vim /var/opt/magma/certs/rootCA.pem      # placeholder cert (no orc8r deployed yet)
wget https://github.com/magma/magma/raw/v1.8/lte/gateway/deploy/agw_install_docker.sh
bash agw_install_docker.sh
```

The script installs Docker + Docker Compose, configures kernel/network settings,
renames the primary interface `enp0s3` to `eth0`, and prompts for a reboot.

### 2.4 Reboot

```bash
reboot
```

Issue: after rebooting, the VM has no network connectivity and SSH cannot connect.
The install script renamed `enp0s3` to `eth0`, but the Netplan config
(`/etc/netplan/00-installer-config.yaml`) still references the old name. That
interface no longer exists, so `eth0` never receives an IP via DHCP.

Diagnosis from the VirtualBox console:

```bash
ip a                                        # eth0 = DOWN, no IP
cat /etc/netplan/00-installer-config.yaml   # still references enp0s3
```

Fix: edit the Netplan file to use the new interface name `eth0`:

```yaml
network:
  ethernets:
    eth0:
      dhcp4: true
  version: 2
```

Then apply:

```bash
sudo netplan apply
ip a            # eth0 now UP with 10.0.2.15
```

SSH access is restored. The Magma-generated file `70-secondary-itf.yaml` (for the
second interface) is left unchanged.

### 2.5 Configure the AGW

Point the AGW at an Orchestrator. Replace the addresses below with your own orc8r
endpoints if deploying locally:

```bash
cat << EOF | sudo tee /var/opt/magma/configs/control_proxy.yml
cloud_address: controller.orc8r.magmacore.link
cloud_port: 443
bootstrap_address: bootstrapper-controller.orc8r.magmacore.link
bootstrap_port: 443
fluentd_address: fluentd.orc8r.magmacore.link
fluentd_port: 24224

rootca_cert: /var/opt/magma/certs/rootCA.pem
EOF
```

### 2.6 Start and verify

```bash
cd /var/opt/magma/docker
sudo docker-compose up -d
sudo docker ps
```

To get the gateway identity (needed when registering the AGW in orc8r):

```bash
sudo docker exec magmad show_gateway_info.py
```

Issue: if `magmad` is in a restart loop (expected without a running orc8r),
`show_gateway_info.py` fails. The Hardware ID can be read directly from disk:

```bash
cat /etc/snowflake
```

## 3. Final Result

The Docker AGW v1.8.0 deploys and starts successfully. Of 21 services, **17 run
healthy** and **4 remain in a restart loop**.

### 3.1 Healthy services (17)

`sessiond`, `directoryd`, `policydb`, `state`, `smsd`, `ctraced`, `redis`,
`td-agent-bit`, `connectiond`, `sctpd`, `subscriberdb`, `redirectd`, `mobilityd`,
`health`, `monitord`, `eventd`, `pipelined`

### 3.2 Restarting services (4)

| Service | Reason |
| --- | --- |
| `magmad` | Cannot reach an Orchestrator (no orc8r deployed) |
| `control_proxy` | Needs a real orc8r and a valid `rootCA.pem` for TLS bootstrap |
| `oai_mme` | MME (Mobility Management Entity) -- sensitive to network/config in a VM |
| `enodebd` | Manages the eNodeB radio -- no radio hardware connected |

These four services all depend on components that are intentionally absent in this
test setup: a deployed Orchestrator and a real radio (eNodeB). With a placeholder
`rootCA.pem`, the control plane cannot complete its secure bootstrap, which is the
expected behavior for a standalone AGW deployment.

Once the AGW is [connected to a running orc8r](orc8r-deployment-guide.md),
`magmad`, `control_proxy`, and `oai_mme` become healthy. Only `enodebd` continues
restarting (no physical radio).