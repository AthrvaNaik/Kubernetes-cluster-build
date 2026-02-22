# 1. Architecture Overview

Cluster topology used:

| Node        | Role          | IP             |
| ----------- | ------------- | -------------- |
| k8s-master  | Control Plane | 192.168.228.17 |
| k8s-worker1 | Worker Node   | 192.168.228.19 |
| k8s-worker2 | Worker Node   | 192.168.228.22 |

Final cluster state:

```
NAME          STATUS   ROLES           VERSION
k8s-master    Ready    master         v1.34.4
k8s-worker1   Ready    worker         v1.34.4
k8s-worker2   Ready    worker         v1.34.4
```

# 2. Kubernetes Cluster Components Explanation

Control plane components:

|Component|Purpose|
|---|---|
|kube-apiserver|Main Kubernetes API|
|etcd|Cluster database|
|kube-scheduler|Assigns pods to nodes|
|kube-controller-manager|Maintains desired state|

Worker components:

|Component|Purpose|
|---|---|
|kubelet|Node agent|
|kube-proxy|Network routing|
|CRI-O|Container runtime|

---

# 3. Container Runtime: CRI-O Explanation

CRI-O is the container runtime used instead of Docker.


Runtime flow:

```
kubectl → kube-apiserver → kubelet → CRI-O → container
```


# 4. Master Node Setup

SSH into master:

```
ssh root@192.168.228.17
```


# Step 1: Disable swap

```
swapoff -a
```

Why?

Kubernetes requires swap disabled for memory management.

Make permanent:

```
sed -i '/swap/d' /etc/fstab
```

# Step 2: Enable kernel modules

```
modprobe overlay
modprobe br_netfilter
```

Persist:

```
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Why?

Required for container networking.

# Step 3: Enable sysctl parameters

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```

Apply:

```
sysctl --system
```

Why?

Allows pod networking.

# Step 4: Disable SELinux enforcement

Temporary:

```
setenforce 0
```

Permanent:

```
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Why?

SELinux can block container runtime operations.

# Step 5: Install CRI-O

Configure repo:

```
cat <<EOF > /etc/yum.repos.d/crio.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.34/rpm/
enabled=1
gpgcheck=0
EOF
```

Install:

```
dnf install -y cri-o
```

Start service:

```
systemctl enable --now crio
```

Verify:

```
systemctl status crio
```

Expected:

```
Active: active (running)
```

# Step 6: Install Kubernetes

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

Install:

```
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Enable kubelet:

```
systemctl enable --now kubelet
```

# Step 7: Initialize cluster

```
kubeadm init \
--apiserver-advertise-address=192.168.228.17 \
--pod-network-cidr=192.168.0.0/16 \
--cri-socket=unix:///var/run/crio/crio.sock
```

Explanation:

|Parameter|Purpose|
|---|---|
|apiserver-advertise-address|master IP|
|pod-network-cidr|pod network range|
|cri-socket|container runtime|

# Step 8: Configure kubectl

```
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```
kubectl get nodes
```


# Step 9: Install Calico Network

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Explanation:

Calico provides:

- Pod networking
    
- IP routing
    
- Network policies


Wait until node Ready:

```
kubectl get nodes
```


# 5. Worker Node Setup

SSH into worker:

```
ssh root@192.168.228.19
```

Repeat same steps:

Disable swap  
Enable kernel modules  
Enable sysctl  
Disable SELinux

Install CRI-O:

```
dnf install -y cri-o
systemctl enable --now crio
```

Install Kubernetes:

```
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```


# Join worker node

Run join command from master output:

```
kubeadm join 192.168.228.17:6443 \
--token TOKEN \
--discovery-token-ca-cert-hash HASH \
--cri-socket=unix:///var/run/crio/crio.sock
```

The above command can be obtained by using following command in *Master* Node
```
kubeadm token create --print-join-command
```



# Verify from master

```
kubectl get nodes
```


# 6. Label Worker Nodes

You executed:

```
kubectl label node k8s-worker1 node-role.kubernetes.io/worker=worker-1
kubectl label node k8s-worker2 node-role.kubernetes.io/worker=worker-2
```

Result:

```
NAME          STATUS   ROLES
k8s-master    Ready    control-plane
k8s-worker1   Ready    worker
k8s-worker2   Ready    worker
```

Explanation:

Labels allow workload scheduling control.



# 7. Verification Steps

Check nodes:

```
kubectl get nodes
```

Check pods:

```
kubectl get pods -A
```

Check CRI-O runtime:

```
crictl ps -a
```


# 8. Troubleshooting Guide


Problem: CRI-O service fails

Error:

```
conmon executable file not found
```

Fix:

```
dnf install -y conmon crun containers-common
systemctl restart crio
```



Problem: Node NotReady

Check:

```
kubectl get pods -n kube-system
```

Fix:

```
kubectl apply -f calico.yaml
```


Problem: ImagePullBackOff

Fix:

```
mkdir -p /etc/containers
cat <<EOF > /etc/containers/policy.json
{
"default":[{"type":"insecureAcceptAnything"}]
}
EOF
```

Restart:

```
systemctl restart crio kubelet
```



Problem: kubelet fails

Check:

```
journalctl -xeu kubelet
```

Common fix:

```
systemctl restart kubelet
```



Problem: worker join fails

Fix:

```
kubeadm reset -f
rm -rf /etc/kubernetes
systemctl restart kubelet crio
```

Join again.

# 9. Reset Guide

To completely clean node:

```
kubeadm reset -f
dnf remove -y kubeadm kubelet kubectl cri-o
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/containers
systemctl daemon-reload
reboot
```

---

# 10. Final Cluster State

```
kubectl get nodes
```

Output:

```
k8s-master Ready control-plane
k8s-worker1 Ready worker
k8s-worker2 Ready worker
```

