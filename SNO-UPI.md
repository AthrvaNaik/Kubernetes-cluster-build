# OpenShift SNO (Single Node OpenShift) Setup [Connected]
## 1. Overview

### Environment

#### Harbor VM (Installer Machine)

* OS: RHEL 
* Role: Installer, HTTP server, and `oc` client
* IP: `192.168.228.31`

#### SNO Node (Target Node)

* OS: RHCOS (installed via ISO)
* Role: Single Node OpenShift (Control Plane + Worker)
* IP: `192.168.228.38`

---
#### Resources used:
* [Red Hat official documentation for SNO Installation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_on_a_single_node/install-sno-installing-sno)
* [Get the pull-secret here](https://console.redhat.com/openshift/install/pull-secret)

## 2. Architecture Overview

In Single Node OpenShift (SNO):

* Control Plane and Worker run on the same node

### Core Components

* `kube-apiserver`
* `etcd`
* `controller-manager`
* `scheduler`

### Bootstrap Process

1. Temporary bootstrap control plane is created
2. Real control plane starts
3. Bootstrap resources are destroyed

---

## 3. Step 0: Install Required Binaries (Harbor VM)

```bash
cd /root
```

### Download OpenShift Installer

```bash
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-install-linux.tar.gz
tar -xvf openshift-install-linux.tar.gz
mv openshift-install /usr/local/bin/
chmod +x /usr/local/bin/openshift-install
```

### Download OC Client

```bash
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
tar -xvf openshift-client-linux.tar.gz
mv oc kubectl /usr/local/bin/
chmod +x /usr/local/bin/oc
```

### Notes

* `openshift-install`: Used to create the cluster
* `oc`: CLI tool to interact with the cluster

---

## 4. Step 1: Create Install Configuration

```bash
mkdir ~/cluster
cd ~/cluster
vi install-config.yaml
```

### Example Configuration

```yaml
apiVersion: v1
baseDomain: lab.local
metadata:
  name: sno

compute:
- name: worker
  replicas: 0

controlPlane:
  name: master
  replicas: 1

networking:
  machineNetwork:
  - cidr: 192.168.228.0/24
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16

platform:
  none: {}

pullSecret: '<your pull secret>'
sshKey: '<your ssh public key>'
```

### Notes

* `platform: none` indicates bare-metal or local setup
* `replicas: 1` enables SNO

---

## 6. Step 2: Generate Ignition Config

```bash
openshift-install create single-node-ignition-config --dir=~/cluster
```

---

## 7. Step 3: Host Ignition File

Using Podman:

```bash
podman run -d -p 8090:80 -v $(pwd):/usr/share/nginx/html:Z docker.io/library/nginx
```

### Test

```bash
curl http://192.168.228.31:8090/bootstrap-in-place-for-live-iso.ign
```

### Notes

* Ignition file contains bootstrap instructions
* It is executed when the node boots

---

## 8. Step 4: Prepare RHCOS ISO

```bash
export ISO_URL=$(openshift-install coreos print-stream-json | grep location | grep iso | cut -d\" -f4)
curl -L $ISO_URL -o rhcos-live.iso
```

### Embed Ignition

```bash
coreos-installer iso ignition embed -fi bootstrap-in-place-for-live-iso.ign rhcos-live.iso
```

---

## 9. Step 5: Boot SNO Node

* Attach ISO to the VM
* Boot the VM

---

## 10. Step 6: SSH Access

```bash
ssh -i ~/.ssh/id_rsa core@192.168.228.38
```

### Notes

* Default user: `core`
* SSH key is injected from `install-config.yaml`

---

## 11. Step 7: Monitor Bootstrap Process

```bash
journalctl -b -f -u bootkube.service
```

### Expected Output

```
bootkube.service complete
Tearing down temporary bootstrap control plane...
```

### Notes

* Temporary control plane is removed
* Real control plane becomes active

---

## 12. Common Errors and Fixes

### API Connection Refused

```
connection to api.sno.lab.local:6443 refused
```

**Reason:**

* API server is still starting

---

### DNS Issues

```
nslookup failed
```

**Fix:**

* Update `/etc/hosts`
* Configure `dnsmasq`

---

### Hostname Issue

```
node-valid-hostname.service failed
```

**Root Cause:**

* Hostname mismatch

---

## 13. Validation Commands

### API Check

```bash
curl -k https://api.sno.lab.local:6443/healthz
```

Expected:

```
ok
```

---

### Node Check

```bash
export KUBECONFIG=~/cluster/auth/kubeconfig
oc get nodes
```

Expected:

```
sno   Ready   master,worker
```

---

### Watch Cluster

```bash
watch -n 5 oc get nodes
```

---

## 14. Troubleshooting Guide

### Step-by-Step Debug Flow

1. DNS Check

```bash
nslookup api.sno.lab.local
```

2. Connectivity

```bash
ping api.sno.lab.local
```

3. Port Check

```bash
nc -zv api.sno.lab.local 6443
```

4. API Check

```bash
curl -k https://api.sno.lab.local:6443/healthz
```

5. Kubelet Status

```bash
systemctl status kubelet
```

6. Containers

```bash
crictl ps
```

7. Port Listening

```bash
ss -tulnp | grep 6443
```

---

