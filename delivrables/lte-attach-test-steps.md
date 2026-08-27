# LTE Attach Test — Setup Steps

All the steps done to get a simulated UE to attach to the Magma AGW using srsRAN, and capture the registration flow. This assumes the AGW and orc8r are already deployed and the LTE module is installed.

## 1. Connect AGW to orc8r

The AGW needs to reach orc8r's bootstrapper and controller endpoints. In this setup, the AGW runs in VirtualBox and orc8r runs on Minikube in WSL, both on the same Windows machine.

Expose orc8r from WSL (leave running):

```bash
kubectl --namespace orc8r port-forward --address 0.0.0.0 svc/orc8r-nginx-proxy 7443:8443 7444:8444
```

Must use `--address 0.0.0.0`, otherwise it only listens on `127.0.0.1` and the AGW VM can't reach it.

Copy the real `rootCA.pem` from orc8r to the AGW VM (replacing the placeholder from the initial AGW install):

```bash
sudo cp /tmp/rootCA.pem /var/opt/magma/certs/rootCA.pem
```

Add hostname mappings in the AGW VM (`/etc/hosts`):

```
172.23.134.198 controller.magma.test bootstrapper-controller.magma.test
```

The IP is WSL's IP (`hostname -I` in WSL). It changes after every `wsl --shutdown`.

Update `control_proxy.yml` in the AGW VM:

```bash
cat << EOF | sudo tee /var/opt/magma/configs/control_proxy.yml
cloud_address: controller.magma.test
cloud_port: 7443
bootstrap_address: bootstrapper-controller.magma.test
bootstrap_port: 7444
fluentd_address: fluentd.magma.test
fluentd_port: 24224

rootca_cert: /var/opt/magma/certs/rootCA.pem
EOF
```

Restart the AGW and check the logs:

```bash
cd /var/opt/magma/docker
sudo docker-compose down && sudo docker-compose up -d
sudo docker logs magmad 2>&1 | grep -i "checkin"
```

Look for `Checkin Successful!`.

## 2. Create the LTE network

Done via the NMS dashboard (Traffic > Add Network) or the API. The network was created with MCC `001`, MNC `01`, TAC `1`.

If the LTE module was not installed when the network was first created, the network ends up as a generic network and the NMS can't display it. See the [LTE module install guide](install-lte-module.md) for the fix.

## 3. Register the gateway

Get the AGW identity:

```bash
sudo docker exec magmad show_gateway_info.py
```

In the NMS dashboard: Equipment > Add Gateway. Fill in the Hardware UUID and Challenge Key from the output above. The gateway name and ID are your choice.

Verify from the AGW logs:

```bash
sudo docker logs magmad 2>&1 | grep -i "checkin"
# Checkin Successful! Successfully sent states to the cloud!
```

## 4. Create an APN

In the NMS: Traffic > APNs > Add APN.

| Field                                | Value       |
| ------------------------------------ | ----------- |
| APN ID                               | `internet`  |
| Class ID                             | `9`         |
| ARP Priority Level                   | `15`        |
| Max Bandwidth UL/DL                  | `200000000` |
| Pre-emption Capability/Vulnerability | `Disabled`  |
| PDN Type                             | `IPv4`      |

## 5. Add a subscriber

Adding via the NMS returned a 500 error. Added via the API instead:

```bash
curl -sk --cert ${CERTS_DIR}/admin_operator.pem --key ${CERTS_DIR}/admin_operator.key.pem \
  -X POST https://localhost:9443/magma/v1/lte/test_network_lte/subscribers \
  -H "Content-Type: application/json" \
  -d '{
    "id": "IMSI001010000000001",
    "name": "test-ue",
    "lte": {
      "state": "ACTIVE",
      "auth_key": "00112233445566778899aabbccddeeff",
      "auth_opc": "63bfa50ee6523365ff14c1f45f88737d",
      "sub_profile": "default"
    },
    "active_apns": ["internet"]
  }'
```

The IMSI starts with `00101` matching the network's MCC/MNC. The auth key and OPC are test values that must match exactly in the srsRAN UE config.

Verify sync to AGW:

```bash
sudo docker exec subscriberdb /usr/local/bin/subscriber_cli.py get IMSI001010000000001
```

## 6. Install srsRAN 4G

Built from source on the AGW VM with ZMQ support:

```bash
sudo apt-get install -y cmake build-essential libfftw3-dev libmbedtls-dev \
  libboost-program-options-dev libconfig++-dev libsctp-dev libzmq3-dev

git clone https://github.com/srsRAN/srsRAN_4G.git
cd srsRAN_4G
mkdir build && cd build
cmake ../
make -j$(nproc)
sudo make install
sudo ldconfig
```

Verify ZMQ was detected: `grep -i zmq build/CMakeCache.txt` should show a path to `libzmq.so`.

Copy the example configs:

```bash
sudo mkdir -p /etc/srsran
sudo cp /usr/local/share/srsran/*.conf.example /etc/srsran/
cd /etc/srsran
for f in *.example; do sudo mv "$f" "${f%.example}"; done
```

## 7. Configure enb.conf

Changes from the defaults:

| Field         | Default                          | Changed to          | Why                                                      |
| ------------- | -------------------------------- | ------------------- | -------------------------------------------------------- |
| `mme_addr`    | `127.0.1.100`                    | `127.0.0.1`         | Default points to srsRAN's built-in EPC, not Magma's MME |
| `device_name` | `#device_name = zmq` (commented) | `device_name = zmq` | Enable ZMQ virtual radio                                 |
| `device_args` | commented                        | uncommented         | ZMQ TCP ports for the radio link                         |

MCC (`001`), MNC (`01`), and `n_prb` (`50`) were already correct in the defaults.

Issue: `rr.conf` had `tac = 0x0007` (decimal 7), but the LTE network uses TAC `1`. srsenb connected to the MME but got `S1 Setup Failure: unknown-PLMN` because of the TAC mismatch. Fixed with:

```bash
sudo sed -i 's|tac = 0x0007|tac = 0x0001|' /etc/srsran/rr.conf
```

## 8. Configure ue.conf

Changes from the defaults:

| Field          | Default                   | Changed to        | Why                                |
| -------------- | ------------------------- | ----------------- | ---------------------------------- |
| `imsi`         | `001010123456780`         | `001010000000001` | Must match the subscriber in orc8r |
| `device_name`  | commented                 | `zmq`             | Enable ZMQ                         |
| `device_args`  | commented                 | uncommented       | ZMQ TCP ports (reversed from eNB)  |
| `apn`          | `#apn = internetinternet` | `apn = internet`  | Must match the APN in orc8r        |
| `apn_protocol` | commented                 | `ipv4`            | Match the PDN type                 |

The `k` and `opc` were already matching the subscriber values by coincidence (same test values).

## 9. Fix MME SCTP binding (IPv6 issue)

srsenb failed to connect to the MME with `Failed to initiate S1 connection`.

The cause: Magma's `sctpd` was listening on `[::1]:36412` (IPv6 loopback only). srsRAN doesn't support IPv6 for SCTP, so it couldn't connect.

The config files (`mme.yml`, `spgw.yml`) have `s1ap_ipv6_enabled` and `s1_ipv6_enabled` flags, but changing them to `false` had no effect. The MME generates a libconfig `.conf` file at runtime (`/var/opt/magma/tmp/mme.conf`) and that's what the binary actually reads.

Fix: edit the generated file directly, then restart (not `docker-compose down` which regenerates it):

```bash
sudo docker exec oai_mme sed -i 's|MME_S1_IPV6_ENABLED.*=.*"True"|MME_S1_IPV6_ENABLED = "False"|' /var/opt/magma/tmp/mme.conf
sudo docker-compose restart sctpd oai_mme
```

After this, `ss -lnp | grep 36412` shows `[::ffff:127.0.0.1]:36412` — IPv4-mapped, reachable from srsRAN.

This fix does not survive `docker-compose down/up` (the .conf file is regenerated). Must be re-applied after every restart.

## 10. Fix GTP port 2152 conflict

srsenb failed with `Failed to bind on address 127.0.0.1, port 2152: Address already in use`.

The cause: Magma's OVS (Open vSwitch) creates a GTP tunnel port (`gtp0`) on `gtp_br0` that binds UDP port 2152 on `0.0.0.0` via a kernel module (`vport_gtp`). This blocks srsenb from binding its own GTP port on any address.

Fix: remove the OVS GTP port, stop pipelined, restart OVS, unload the kernel modules, then restart pipelined (needed for IP allocation):

```bash
sudo ovs-vsctl --if-exists del-port gtp_br0 gtp0
sudo docker-compose stop pipelined
sudo systemctl restart openvswitch-switch
sleep 3
sudo rmmod vport_gtp 2>/dev/null
sudo rmmod gtp 2>/dev/null
sudo docker-compose start pipelined
sleep 15
sudo ovs-vsctl --if-exists del-port gtp_br0 gtp0
sudo rmmod vport_gtp 2>/dev/null
sudo rmmod gtp 2>/dev/null
```

Verify with `ss -ulnp | grep 2152` — should be empty.

This fix does not survive a VM reboot. Must be re-applied after every restart.

## 11. Run the test

Three terminals to the AGW VM:

**Terminal 1 — capture:**

```bash
sudo tcpdump -i lo -w /tmp/lte_attach.pcap sctp
```

**Terminal 2 — eNB:**

```bash
sudo srsenb /etc/srsran/enb.conf
```

Wait for `==== eNodeB started ===` with no S1 error.

**Terminal 3 — UE:**

```bash
sudo srsue /etc/srsran/ue.conf
```

Wait for `Network attach successful. IP: 192.168.128.x`.

Stop all three with Ctrl+C (srsue first, then srsenb, then tcpdump). The pcap file can be opened in Wireshark with filter `s1ap || nas-eps`.

## 12. Startup script (after VM reboot)

All the runtime fixes (MME IPv6, GTP port) are lost on reboot. Run this to restore them:

```bash
cd /var/opt/magma/docker
sudo docker-compose up -d
sleep 20

# Fix MME IPv6
sudo docker exec oai_mme sed -i 's|MME_S1_IPV6_ENABLED.*=.*"True"|MME_S1_IPV6_ENABLED = "False"|' /var/opt/magma/tmp/mme.conf
sudo docker-compose restart sctpd oai_mme
sleep 5

# Free GTP port
sudo ovs-vsctl --if-exists del-port gtp_br0 gtp0
sudo docker-compose stop pipelined
sudo systemctl restart openvswitch-switch
sleep 3
sudo rmmod vport_gtp 2>/dev/null
sudo rmmod gtp 2>/dev/null
sudo docker-compose start pipelined
sleep 15
sudo ovs-vsctl --if-exists del-port gtp_br0 gtp0
sudo rmmod vport_gtp 2>/dev/null
sudo rmmod gtp 2>/dev/null
```
