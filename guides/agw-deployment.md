# Magma Access Gateway (AGW) Deployment Report

**Mentee:** Shakir El Amrani
**Deliverable:** #2 — Magma Deployment Report
**Component:** Docker-based Access Gateway (AGW), Magma v1.8.0
**Environment:** Ubuntu Server 20.04.6 LTS on Oracle VirtualBox
**Source guide:** [Install Docker AGW — Magma Docs v1.8.0](https://magma.github.io/magma/docs/lte/deploy_install_docker)

---

## 1. Objective

Install the Magma Docker-based Access Gateway on a virtual machine, document the full process, and record the issues encountered along with their resolutions.

The official guide assumes a bare-metal machine installed from a USB stick. This deployment adapts those steps to an Oracle VirtualBox VM, since the Docker AGW guide does not document a VM workflow. **Each issue is documented inline, right after the step where the problem occurs**, so the procedure can be followed end-to-end with the relevant issue flagged at the exact point it appears.

---

## 2. Environment Setup

### 2.1 Virtual Machine Configuration (VirtualBox)

| Setting | Value |
| --- | --- |
| Name | `magma-agw` |
| OS | Ubuntu Server 20.04.6 LTS (Focal Fossa), 64-bit |
| RAM | 4 GB+ |
| CPU | 2+ cores |
| Disk | 40 GB (dynamically allocated) |
| Adapter 1 | NAT — SGi side (internet + management) |
| Adapter 2 | Internal Network — S1 side (isolated) |

The AGW requires **two ethernet interfaces** (`enp1s0`/SGi and `enp2s0`/S1), so two network adapters were configured.

> **⚠️ Issue — Only one network interface present.**
> After the initial OS install, `ip address` showed only one interface (`enp0s3`); the AGW requires two.
> **Cause:** only one network adapter was enabled in VirtualBox.
> **Fix:** power off the VM, enable **Adapter 2** (Attached to: Internal Network) in VM Settings → Network, then restart. `ip address` then shows the second interface `enp0s8`. (This is why the table above lists two adapters — configure both before installing.)

### 2.2 OS Installation

- Booted the VM from the `ubuntu-20.04.6-live-server-amd64.iso` image.
- Enabled **OpenSSH server** during installation (confirmed in the install log: `installing openssh-server`).
- No featured server snaps were selected.

---

## 3. Deployment Steps (with inline issues)

### 3.1 Verify the two interfaces (before install)

```bash
ip address
```

Output showed:
- `enp0s3` — Adapter 1 (NAT), IP `10.0.2.15/24`, state UP
- `enp0s8` — Adapter 2 (Internal Network), state DOWN, no IP (expected, nothing connected)

### 3.2 SSH access from the Windows host

Port forwarding was added in VirtualBox (Adapter 1 → Advanced → Port Forwarding):

| Name | Protocol | Host Port | Guest Port |
| --- | --- | --- | --- |
| ssh | TCP | 2222 | 22 |

Connected from the host:

```bash
ssh -p 2222 amranich@127.0.0.1
```

> **⚠️ Issue — SSH "Bad owner or permissions" on Windows.**
> Connecting via SSH failed with:
> ```
> Bad owner or permissions on C:\Users\AmraniCh/.ssh/config
> ```
> **Cause:** Windows had the `.ssh/config` file shared with extra users/groups; OpenSSH refuses configs that are not restricted to the owner.
> **Fix:** restrict the file permissions in PowerShell:
> ```powershell
> $file = "$env:USERPROFILE\.ssh\config"
> icacls $file /inheritance:r
> icacls $file /grant:r "$($env:USERNAME):(R,W)"
> ```
> (Alternative quick workaround: `ssh -F none -p 2222 amranich@127.0.0.1`.)

### 3.3 AGW installation

```bash
sudo -i
mkdir -p /var/opt/magma/certs
vim /var/opt/magma/certs/rootCA.pem      # placeholder cert (no orc8r deployed)
wget https://github.com/magma/magma/raw/v1.8/lte/gateway/deploy/agw_install_docker.sh
bash agw_install_docker.sh
```

The script installed Docker + Docker Compose, configured kernel/network settings, renamed the primary interface `enp0s3` to `eth0`, and prompted for a reboot.

### 3.4 Reboot

```bash
reboot
```

> **⚠️ Issue — Lost SSH / no IP after reboot (most important).**
> After running the install script and rebooting, the VM had no network connectivity and SSH could not connect.
> **Cause:** the install script renamed `enp0s3` to `eth0`, but the Netplan config (`/etc/netplan/00-installer-config.yaml`) still referenced the old name `enp0s3`. That interface no longer existed, so `eth0` never received an IP via DHCP.
> Diagnosis from the VirtualBox console:
> ```bash
> ip a                                        # eth0 = DOWN, no IP
> cat /etc/netplan/00-installer-config.yaml   # still referenced enp0s3
> ```
> **Fix:** edit the Netplan file to use the new interface name `eth0`:
> ```yaml
> network:
>   ethernets:
>     eth0:
>       dhcp4: true
>   version: 2
> ```
> Then apply it:
> ```bash
> sudo netplan apply
> ip a            # eth0 now UP with 10.0.2.15
> ```
> SSH access is restored. The Magma-generated file `70-secondary-itf.yaml` (for the second interface) is left unchanged.

### 3.5 Configure the AGW (after fixing networking)

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

### 3.6 Start and verify the AGW

```bash
cd /var/opt/magma/docker
sudo docker-compose up -d
docker ps
docker exec magmad show_gateway_info.py
sudo docker-compose up -d --force-recreate
```

> **⚠️ Issue — `show_gateway_info.py` failed.**
> ```
> Error response from daemon: Container ... magmad is restarting, wait until the container is running
> ```
> **Cause:** the `magmad` container was in a restart loop (see Section 4), so commands could not execute inside it.
> **Fix / workaround:** the Hardware ID is stored on disk and can be read directly:
> ```bash
> cat /etc/snowflake
> ```

---

## 4. Final Result

The Docker AGW v1.8.0 deployed and started successfully. Of 21 services, **17 run healthy** and **4 remain in a restart loop**.

### 4.1 Healthy services (17)

`sessiond`, `directoryd`, `policydb`, `state`, `smsd`, `ctraced`, `redis`, `td-agent-bit`, `connectiond`, `sctpd`, `subscriberdb`, `redirectd`, `mobilityd`, `health`, `monitord`, `eventd`, `pipelined`

### 4.2 Restarting services (4) — expected

| Service | Reason |
| --- | --- |
| `magmad` | Cannot reach an Orchestrator (no orc8r deployed) |
| `control_proxy` | Needs a real orc8r and a valid `rootCA.pem` for TLS bootstrap |
| `oai_mme` | MME (Mobility Management Entity) — sensitive to network/config in a VM |
| `enodebd` | Manages the eNodeB radio — no radio hardware connected |

### 4.3 Interpretation

The AGW core and data-plane services installed and run correctly. The four restarting services all depend on components that are intentionally absent in this test setup: a deployed **Orchestrator (orc8r)** and a real **radio (eNodeB)**. With a placeholder `rootCA.pem`, the control plane cannot complete its secure bootstrap with an orchestrator, which is the expected behavior for a standalone AGW deployment.

---

## 5. Known Limitations

- **No Orchestrator (orc8r):** the AGW cannot register/check in; control-plane containers restart.
- **No radio (eNodeB):** `enodebd` has no device to manage.
- **Placeholder certificate:** `rootCA.pem` is a placeholder, so no real TLS trust is established.
- **Virtualized environment:** the official Docker AGW guide documents a bare-metal/USB workflow; this deployment adapted it to VirtualBox.

---

## 6. Conclusion

The Magma Docker-based Access Gateway was successfully deployed on a VirtualBox Ubuntu 20.04 VM. The primary real-world issue was the interface rename (`enp0s3` → `eth0`) breaking the Netplan configuration after reboot, which was resolved by updating the Netplan file and re-applying it. The resulting AGW runs its core and data-plane services healthily; the remaining restart loops are the expected consequence of running a standalone AGW without an Orchestrator or radio hardware.
