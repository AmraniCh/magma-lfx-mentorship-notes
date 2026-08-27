# Installing the LTE Module for orc8r on Minikube

**Context:** After deploying orc8r v1.8.0 on Minikube following the official dev guide, the NMS dashboard shows "Not Found" when creating an LTE network, and the API endpoint `/magma/v1/lte` returns a 404. This happens because the base orc8r Helm chart only deploys core services. The LTE-specific module must be installed separately.

---

## 1. The Problem

The base orc8r Helm chart (at `orc8r/cloud/helm/orc8r/`) deploys only the core orchestrator services:

```
accessd, alertmanager, analytics, bootstrapper, certifier, configurator,
ctraced, device, directoryd, dispatcher, eventd, metricsd, nginx,
obsidian, orchestrator, prometheus, service-registry, state, streamer, tenants
```

The LTE-specific services (lte, ha, policydb, subscriberdb, subscriberdb_cache, smsd, nprobe) are **not included**. Without them:

- The `/magma/v1/lte` API endpoint does not exist (returns 404).
- The NMS dashboard cannot create or display LTE networks (shows "Not Found").
- A generic network can be created via `/magma/v1/networks`, but it has no LTE configuration, so the NMS ignores it.

The official Minikube guide does not mention this separate installation step.

---

## 2. Where the LTE Chart Lives

The LTE module chart is at:

```
${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r/
```

It deploys 7 pods that register the LTE API handlers, subscriber management, policy, and SMS endpoints.

---

## 3. Installation Steps

### 3.1 Create the values file

Copy the example values file and substitute the registry and tag:

```bash
cp ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r/examples/minikube.values.yaml \
   ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml

sed -i 's|IMAGE_REGISTRY_URL|linuxfoundation.jfrog.io/magma-docker|g' \
  ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml

sed -i 's|IMAGE_TAG|1.8.0|g' \
  ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml
```

### 3.2 Add the missing orc8r_domain_name field

The LTE chart's orc8rlib templates reference `controller.image.env.orc8r_domain_name`, but the example values file does not include it. Without this field, `helm install` fails with:

```
nil pointer evaluating interface {}.orc8r_domain_name
```

Fix by adding the `env` block under `controller.image` (after the `pullPolicy` line):

```bash
sed -i '/^  image:/,/pullPolicy/{ /pullPolicy/a\    env:\n      orc8r_domain_name: magma.test
}' ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml
```

Verify it looks right:

```bash
grep -A8 "image:" ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml
```

Expected output:

```yaml
  image:
    repository: linuxfoundation.jfrog.io/magma-docker/controller
    tag: 1.8.0
    pullPolicy: IfNotPresent
    env:
      orc8r_domain_name: magma.test
```

### 3.3 Fix deprecated Kubernetes API versions

Same issue as the base orc8r chart: Kubernetes 1.28 removed the beta API versions that the v1.8 chart still uses.

```bash
cd ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r
grep -rl "policy/v1beta1" . | xargs -r sed -i 's|policy/v1beta1|policy/v1|g'
grep -rl "rbac.authorization.k8s.io/v1beta1" . | xargs -r sed -i 's|rbac.authorization.k8s.io/v1beta1|rbac.authorization.k8s.io/v1|g'
```

### 3.4 Install the chart

```bash
cd ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r
helm dep update
helm upgrade --install --namespace orc8r \
  --values ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml \
  lte-orc8r .
```

Expected output: `STATUS: deployed`.

### 3.5 Verify the pods

```bash
kubectl --namespace orc8r get pods | grep -E "lte|ha|policydb|subscriberdb|smsd|nprobe"
```

Expected: 7 new pods, all `Running`:

```
orc8r-ha-...                  1/1     Running
orc8r-lte-...                 1/1     Running
orc8r-nprobe-...              1/1     Running
orc8r-policydb-...            1/1     Running
orc8r-smsd-...                1/1     Running
orc8r-subscriberdb-...        1/1     Running
orc8r-subscriberdb-cache-...  1/1     Running
```

Total pod count goes from 28 (base orc8r) to 35.

---

## 4. Post-Install: Fix Pre-Existing Generic Networks

If you already created a network via the NMS before installing the LTE module, that network was registered as a generic network (no LTE configuration). It will not appear in the NMS LTE dashboard. To fix this:

**Delete the old generic network:**

```bash
curl -sk --cert ${CERTS_DIR}/admin_operator.pem --key ${CERTS_DIR}/admin_operator.key.pem \
  -X DELETE https://localhost:9443/magma/v1/networks/<network_id>
```

**Recreate it as a proper LTE network** (via the NMS dashboard or API):

```bash
curl -sk --cert ${CERTS_DIR}/admin_operator.pem --key ${CERTS_DIR}/admin_operator.key.pem \
  -X POST https://localhost:9443/magma/v1/lte \
  -H "Content-Type: application/json" \
  -d '{
    "id": "test_network_lte",
    "name": "Test LTE Network",
    "description": "LTE network for AGW testing",
    "dns": {"enable_caching": false, "local_ttl": 0},
    "cellular": {
      "epc": {"mcc": "001", "mnc": "01", "tac": 1},
      "ran": {"bandwidth_mhz": 20}
    }
  }'
```

Or simply create a new LTE network from the NMS dashboard (it works now that the LTE module is running).

---

## 5. Verification

Confirm the LTE API endpoint now works:

```bash
curl -sk --cert ${CERTS_DIR}/admin_operator.pem --key ${CERTS_DIR}/admin_operator.key.pem \
  https://localhost:9443/magma/v1/lte/test_network_lte
```

This should return the full LTE network configuration (cellular, dns, etc.) instead of "Not Found".

---

## 6. Summary of Issues and Fixes

| Issue                                                                                             | Cause                                                                                                  | Fix                                                                        |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| NMS shows "Not Found" when creating a network                                                     | LTE module not deployed; `/magma/v1/lte` endpoint does not exist                                       | Install the `lte-orc8r` Helm chart separately                              |
| `helm install` fails with `nil pointer evaluating interface {}.orc8r_domain_name`                 | Example values file is missing the `controller.image.env.orc8r_domain_name` field                      | Add `env: orc8r_domain_name: magma.test` under `controller.image`          |
| `helm install` fails with `no matches for kind "PodDisruptionBudget" in version "policy/v1beta1"` | Kubernetes 1.28 removed beta API versions                                                              | Replace `policy/v1beta1` with `policy/v1` in the chart templates           |
| Network exists in API but not in NMS                                                              | Network was created as generic (before LTE module existed), so NMS cannot query it via `/magma/v1/lte` | Delete the generic network and recreate it through the LTE endpoint or NMS |
