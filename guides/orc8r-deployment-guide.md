# Magma orc8r Deployment with Minikube

This guide is for deploying orc8r with Minikube for local developpement.

## Table of Contents

| Section | Description |
| --- | --- |
| [1. Installing Prerequisites](#1-installing-prerequisites) | Install Docker, kubectl, Minikube, Helm |
| [1.1 Docker](#11-docker) | Install Docker and configure permissions |
| [1.2 kubectl](#12-kubectl) | Install kubectl v1.28.0 (pinned to cluster) |
| [1.3 Minikube](#13-minikube) | Install Minikube |
| [1.4 Helm 3](#14-helm-3) | Install Helm 3 |
| [1.5 Verify all prerequisites](#15-verify-all-prerequisites) | Confirm all tools are installed |
| [2. Deployment Steps](#2-deployment-steps) | The full orc8r deployment |
| [2.1 Clone Magma & Set MAGMA_ROOT](#21-clone-magma--set-magma_root-environment-variable) | Clone the repo and set the environment |
| [2.2 Start the Kubernetes cluster](#22-start-the-kubernetes-cluster) | Start Minikube (K8s v1.28.0) |
| [2.3 Install PostgreSQL](#23-install-postgresql) | Install the database |
| [2.4 Generate secrets](#24-generate-secrets) | Build images and generate certificates |
| [2.5 Apply secrets](#25-apply-secrets) | Load certificates into the cluster |
| [2.6 Create the values file](#26-create-the-values-file) | Configure the chart values |
| [2.7 Install the orc8r chart](#27-install-the-orc8r-chart) | Install the base orc8r |
| [2.8 Create the admin user and verify the API](#28-create-the-admin-user-and-verify-the-api) | Create admin, test the API |
| [2.9 Access the NMS web dashboard](#29-access-the-nms-web-dashboard) | Log in to the NMS |
| [3. Install the LTE module](#3-install-the-lte-module-optional) | Install the separate lte-orc8r chart |
| [3.1 Create the LTE values file](#31-create-the-lte-values-file) | Configure the LTE chart values |
| [3.2 Add the domain name field](#32-add-the-domain-name-field) | Add required orc8r_domain_name |
| [3.3 Patch the removed Kubernetes API versions](#33-patch-the-removed-kubernetes-api-versions) | Fix policy/v1beta1 |
| [3.4 Install the LTE chart](#34-install-the-lte-chart) | Install lte-orc8r |
| [3.5 Verify](#35-verify) | Confirm the 7 LTE pods are running |

## 1. Installing Prerequisites

* Docker
* kubectl
* Minikube
* Helm

> the Magma guide starts at minikube start, assuming Docker, Minikube, kubectl, and Helm are already installed

### 1.1 Docker

```bash
# Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

#### Verify Docker is running

```bash
sudo service docker status
```

#### Verify no permission error is showing

```bash
docker ps
```

### 1.2 kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

* from the official docs of installing `kubectl`: **Official policy: kubectl must be within ±1 minor version of the cluster.**
* used kubernities 1.28 because Minikube refuse to work with old versions (1.20 in the official magma docs)

```bash
# download the binary
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"

# validate the binary (optional)
curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check # should show 'kubectl: OK'

# install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# verify the version
kubectl version --client

# remove the binary & checksum file
rm kubectl kubectl.sha256
```

### 1.3 Minikube

https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
# verify with
minikube version
```

### 1.4 Helm 3

https://helm.sh/docs/v3/intro/install

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh
# verify with
helm version
```

### 1.5 Verify all prerequisites

```bash
docker --version && minikube version && kubectl version --client && helm version
```

## 2. Deployment Steps

### 2.1 Clone Magma & Set MAGMA_ROOT environment variable

```bash
git clone https://github.com/magma/magma.git ~/magma
export MAGMA_ROOT=PATH_TO_YOUR_MAGMA_CLONE
# OR it better to preserve it in ~/.bashrc
echo "export MAGMA_ROOT=$HOME/magma" >> ~/.bashrc
```

### 2.2 Start the Kubernetes cluster

* Changed `--kubernetes-version='v1.20.2'` from the official docs to `v1.28.0`:
  modern Minikube rejects v1.20.2 (oldest supported: v1.28.0), and forcing it with
  `--force` produces a broken cluster.

```bash
# make sure that your machine has the specified number of CPU cores (8 cores in our case here), the same for memory
minikube start --cni=bridge --driver=docker --memory=8gb --cpus=8 \
  --kubernetes-version='v1.28.0' --mount \
  --mount-string "${MAGMA_ROOT}/orc8r/cloud/docker/metrics-configs:/configs"
```

### 2.3 Install PostgreSQL

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install \
    --create-namespace \
    --namespace orc8r \
    --set global.postgresql.auth.postgresPassword=postgres,global.postgresql.auth.database=magma,fullnameOverride=postgresql \
    postgresql \
    bitnami/postgresql

# Verify postgresql-0 is Running.
kubectl --namespace orc8r get pods
```

### 2.4 Generate secrets

* The certificate generation step builds all orc8r Docker images including the `fluentd`. The fluentd image build fails on a modern system because its base image (`fluent/fluentd:v1.14.6`, from 2022) ships **Ruby 2.7.5**, but the `gem install` step inside it fetches today's newest versions of two dependencies (`multi_json` and `excon`), which now require **Ruby >= 3.x**.

* The command below fixes this by inserting two lines into both fluentd Dockerfiles that pre-install the last Ruby-2.7-compatible versions of those gems, so the dependency resolver finds them already installed and does not fetch the incompatible new ones.

```bash
for f in ${MAGMA_ROOT}/orc8r/cloud/docker/fluentd/Dockerfile ${MAGMA_ROOT}/orc8r/cloud/docker/fluentd_forward/Dockerfile; do
  sed -i '/^USER root/a RUN gem install multi_json -v 1.15.0 --no-document\nRUN gem install excon -v 1.2.5 --no-document' "$f"
done
```

Without this patch, the build fails with:
`multi_json requires Ruby version >= 3.2. The current ruby version is 2.7.5`
(and after fixing only multi_json, the same error appears for `excon`).

#### Build for generating the secrets

```bash
export CERTS_DIR=${MAGMA_ROOT}/.cache/test_certs
cd ${MAGMA_ROOT}/orc8r/cloud/docker && ./build.py && ./run.py && sleep 30 && docker-compose down && ls -l ${CERTS_DIR} && cd -
```

### 2.5 Apply secrets

Before applying the certificates, grant ownership of the cert files to the current user: 

```bash
export CERTS_DIR=${MAGMA_ROOT}/.cache/test_certs
sudo chown -R $USER:$USER ${CERTS_DIR}
```

and then: 

```bash
export IMAGE_REGISTRY_URL=linuxfoundation.jfrog.io/magma-docker  # or replace with your registry
export IMAGE_REGISTRY_USERNAME=''
export IMAGE_REGISTRY_PASSWORD=''

cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r
helm template orc8r charts/secrets \
  --namespace orc8r \
  --set-string secret.certs.enabled=true \
  --set-file 'secret.certs.files.rootCA\.pem'=${CERTS_DIR}/rootCA.pem \
  --set-file 'secret.certs.files.bootstrapper\.key'=${CERTS_DIR}/bootstrapper.key \
  --set-file 'secret.certs.files.controller\.crt'=${CERTS_DIR}/controller.crt \
  --set-file 'secret.certs.files.controller\.key'=${CERTS_DIR}/controller.key \
  --set-file 'secret.certs.files.admin_operator\.pem'=${CERTS_DIR}/admin_operator.pem \
  --set-file 'secret.certs.files.admin_operator\.key\.pem'=${CERTS_DIR}/admin_operator.key.pem \
  --set-file 'secret.certs.files.certifier\.pem'=${CERTS_DIR}/certifier.pem \
  --set-file 'secret.certs.files.certifier\.key'=${CERTS_DIR}/certifier.key \
  --set-file 'secret.certs.files.nms_nginx\.pem'=${CERTS_DIR}/controller.crt \
  --set-file 'secret.certs.files.nms_nginx\.key\.pem'=${CERTS_DIR}/controller.key \
  --set=docker.registry=${IMAGE_REGISTRY_URL} \
  --set=docker.username=${IMAGE_REGISTRY_USERNAME} \
  --set=docker.password=${IMAGE_REGISTRY_PASSWORD} |
  kubectl apply -f -
```

### 2.6 Create the values file

```bash
cp ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/examples/minikube.values.yaml \
   ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r.values.yaml
cd ${MAGMA_ROOT}/orc8r/cloud/helm
sed -i 's|IMAGE_REGISTRY_URL|linuxfoundation.jfrog.io/magma-docker|g' orc8r.values.yaml
sed -i 's|IMAGE_TAG|1.8.0|g' orc8r.values.yaml
```

Verify with:

```bash
grep -E "repository|tag" orc8r.values.yaml | head
```

### 2.7 Install the orc8r chart

* Kubernetes 1.28 removed the beta APIs `policy/v1beta1` and `rbac.authorization.k8s.io/v1beta1`, which the v1.8 chart still uses. 
* Fixed by updating them to the stable versions across the chart.

```bash
cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r
grep -rl "policy/v1beta1" . | xargs sed -i 's|policy/v1beta1|policy/v1|g'
grep -rl "rbac.authorization.k8s.io/v1beta1" . | xargs sed -i 's|rbac.authorization.k8s.io/v1beta1|rbac.authorization.k8s.io/v1|g'
```

and then:

```bash
cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r
helm dep update
helm upgrade --install --namespace orc8r --values ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r.values.yaml orc8r .
```

After installing watch the pods come up:

```bash
kubectl --namespace orc8r get pods
```

Issue: the `nms-nginx-proxy` pod stays in `CrashLoopBackOff`. To fix (modern nginx removed the `ssl on;` directive the chart still uses):

```bash
cd ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r/charts/nms/templates/etc
sed -i 's|listen 443;|listen 443 ssl;|g' _nginx_proxy_ssl.conf.tpl _certs_nginx_proxy_ssl.conf.tpl
sed -i '/ssl on;/d' _nginx_proxy_ssl.conf.tpl _certs_nginx_proxy_ssl.conf.tpl
```

and then: 

```bash
helm upgrade --install --namespace orc8r --values ${MAGMA_ROOT}/orc8r/cloud/helm/orc8r.values.yaml orc8r .
# get the container name for 'nginx-proxy' and replace it 
kubectl --namespace orc8r get pods
kubectl --namespace orc8r delete pod <nms-nginx-proxy-POD-NAME>
```

### 2.8 Create the admin user and verify the API

```bash
# Create an Orc8r admin user
kubectl exec -it --namespace orc8r deploy/orc8r-orchestrator -- \
  /var/opt/magma/bin/accessc add-existing -admin -cert /var/opt/magma/certs/admin_operator.pem admin_operator

# Verify (two terminals)
# Terminal 1 (leave running):
kubectl --namespace orc8r port-forward svc/orc8r-nginx-proxy 7443:8443 7444:8444 9443:443

# Terminal 2:
# change PATH_TO_YOUR_MAGMA_CLONE by your magma clone path
export MAGMA_ROOT=~/magma
export CERTS_DIR=${MAGMA_ROOT}/.cache/test_certs
curl --insecure --cert ${CERTS_DIR}/admin_operator.pem --key ${CERTS_DIR}/admin_operator.key.pem https://localhost:9443
# should output: "hello"
```

### 2.9 Access the NMS web dashboard

```bash
# Create the NMS admin in the magma-test organization (must match the URL subdomain)
kubectl --namespace orc8r exec -it deploy/nms-magmalte -- \
  yarn setAdminPassword magma-test admin@magma.test password1234
```

and then:

```bash
kubectl --namespace orc8r port-forward --address 0.0.0.0 svc/nginx-proxy 8081:443
```

Open: `https://magma-test.localhost:8081` OR `https://SERVER_IP:8081`

Login: 

* `admin@magma.test`
* `password1234`


## 3. Install the LTE module (optional)

The base `orc8r` chart does not include LTE functionality. Without the `lte-orc8r`
chart, creating an LTE network in the NMS fails with a "Not Found" error. Install
this chart if you plan to manage LTE networks, subscribers, or gateways.

### 3.1 Create the LTE values file

```bash
cp ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r/examples/minikube.values.yaml \
   ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml

sed -i 's|IMAGE_REGISTRY_URL|linuxfoundation.jfrog.io/magma-docker|g' ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml
sed -i 's|IMAGE_TAG|1.8.0|g' ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml
```

### 3.2 Add the domain name field

The orc8rlib templates require `orc8r_domain_name`. Add it under the image block in
`lte-orc8r.values.yaml` (use the same domain as the base orc8r install):

```yaml
image:
  env:
    orc8r_domain_name: "magma.test"
```

### 3.3 Patch the removed Kubernetes API versions

Same fix as the base chart (Kubernetes 1.25+ removed `policy/v1beta1`):

```bash
cd ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r
grep -rl "policy/v1beta1" . | xargs -r sed -i 's|policy/v1beta1|policy/v1|g'
grep -rl "rbac.authorization.k8s.io/v1beta1" . | xargs -r sed -i 's|rbac.authorization.k8s.io/v1beta1|rbac.authorization.k8s.io/v1|g'
```

### 3.4 Install the LTE chart

```bash
cd ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r
helm dep update
helm upgrade --install --namespace orc8r \
  --values ${MAGMA_ROOT}/lte/cloud/helm/lte-orc8r.values.yaml lte-orc8r .
```

### 3.5 Verify

```bash
kubectl --namespace orc8r get pods
```

Seven new pods should appear (`orc8r-lte`, `orc8r-ha`, `orc8r-policydb`,
`orc8r-subscriberdb`, `orc8r-subscriberdb-cache`, `orc8r-smsd`, `orc8r-nprobe`),
bringing the total from 28 to 35. You can now create an LTE network in the NMS.