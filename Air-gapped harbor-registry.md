

# Air-Gapped Harbor Registry Deployment with HTTPS on RHEL 


# 1. Project Overview

## 1.1 Objective

To deploy a secure, air-gapped Harbor Container Registry on a RHEL 10 virtual machine using VMware Workstation, configure HTTPS using a private CA-signed certificate, and successfully import and push container images in an offline environment.

# 2. Environment Details

| Component      | Details                                           |
| -------------- | ------------------------------------------------- |
| Hypervisor     | VMware Workstation                                |
| OS             | RHEL 10                                           |
| Docker Version | 29.x                                              |
| Harbor Version | v2.14.2 (Offline Installer)                       |
| Registry URL   | [https://192.168.49.100](https://192.168.49.100/) |
| Network Type   | Air-Gapped (No Internet)                          |

# 3. Resources & Official Download Links

## 3.1 RHEL ISO

Official Download:  
[https://access.redhat.com/downloads](https://access.redhat.com/downloads)
Requires Red Hat account.

Documentation:  
[https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/)

## 3.2 Docker Engine RPMs

Official Docker repository:  
[https://docs.docker.com/engine/install/rhel/](https://docs.docker.com/engine/install/rhel/)

Download RPM packages:  
[https://download.docker.com/linux/rhel/](https://download.docker.com/linux/rhel/)

Packages used:
- docker-ce
- docker-ce-cli
- containerd.io
- docker-compose-plugin

## 3.3 Harbor Offline Installer
Official Harbor GitHub:  
[https://github.com/goharbor/harbor](https://github.com/goharbor/harbor)

Official releases page:  
[https://github.com/goharbor/harbor/releases](https://github.com/goharbor/harbor/releases)

Downloaded:  
harbor-offline-installer-v2.14.2.tgz

Official Harbor Documentation:  
[https://goharbor.io/docs/](https://goharbor.io/docs/)

## 3.4 OpenSSL Documentation

[https://www.openssl.org/docs/](https://www.openssl.org/docs/)


# 4. Architecture Overview

Internet Machine:
- Pull Docker images
- Download RPMs 
- Transfer files via SCP

Harbor VM (Air-Gapped):
- Docker Engine 
- Harbor Core
- Harbor Registry
- Harbor DB
- Harbor Redis
- Harbor Nginx (HTTPS)

Workflow:

Internet → docker pull → docker save → scp → Harbor VM → docker load → docker push


# 5. VM Creation (VMware)

1. Created new VM
2. Attached RHEL ISO
3. Assigned:
  - 8 GB RAM
  - 4 CPUs
  - 120 GB disk
4. Network: Host only

# 6. Static IP Assignment

Checked interface:

```bash
ip a
```

Configured static IP:

```bash
nmcli connection show

nmcli connection modify ens160 \
ipv4.addresses 192.168.49.100/24 \
ipv4.gateway 192.168.49.1 \
ipv4.method manual

nmcli connection down ens160
nmcli connection up ens160
```

Verified:

```bash
ip a
```

On Windows:

```powershell
ipconfig
ping 192.168.49.100
```

---

# 7. File Transfer from Internet Machine

On Windows PowerShell:

```powershell
cd "C:\Users\DELL\Desktop"

scp *.rpm root@192.168.49.100:/root/

scp harbor-offline-installer-v2.14.2.tgz root@192.168.49.100:/root/
```


# 8. Docker Installation (Offline)

On Harbor VM:

```bash
cd /root
ls -lh

dnf install *.rpm -y

systemctl enable docker
systemctl start docker

docker version
systemctl status docker
```


# 9. Harbor Installation

```bash
tar -xvf harbor-offline-installer-v2.14.2.tgz
cd harbor

vi harbor.yml
```

Configured:

```yaml
hostname: 192.168.49.100

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key
```

Deployed:

```bash
./prepare
docker compose up -d
docker ps
```

<img width="800" height="216" alt="image" src="https://github.com/user-attachments/assets/266e1d6b-2cb2-4a8d-9f65-3c8273c528a6" />



# 10. Private CA & HTTPS Configuration

Created CA:

```bash
mkdir /root/ca
cd /root/ca

openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes \
-key ca.key \
-sha512 \
-days 3650 \
-out ca.crt
```

Generated Harbor certificate:

```bash
mkdir /root/certs
cd /root/certs
```

Created harbor.cnf with:

```
IP.1 = 192.168.49.100
```

Then:

```bash
openssl genrsa -out harbor.key 4096

openssl req -new -key harbor.key -out harbor.csr -config harbor.cnf

openssl x509 -req \
-in harbor.csr \
-CA /root/ca/ca.crt \
-CAkey /root/ca/ca.key \
-CAcreateserial \
-out harbor.crt \
-days 3650 \
-extfile harbor.cnf \
-extensions req_ext
```

Copied:

```bash
cp harbor.crt /data/cert/
cp harbor.key /data/cert/
```

Trusted CA:

```bash
cp /root/ca/ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```

Trusted in Docker:

```bash
mkdir -p /etc/docker/certs.d/192.168.49.100
cp /root/ca/ca.crt /etc/docker/certs.d/192.168.49.100/ca.crt
```

Restarted Docker:

```bash
systemctl stop docker.socket
systemctl stop docker
systemctl daemon-reexec
systemctl start docker
```

Restarted Harbor:

```bash
cd /root/harbor
docker compose down
./prepare
docker compose up -d
```

---

# 11. HTTPS Verification

```bash
curl https://192.168.49.100/v2/
```

Expected:

```
{"errors":[{"code":"UNAUTHORIZED","message":"unauthorized"}]}
```

<img width="959" height="110" alt="image" src="https://github.com/user-attachments/assets/89c1525c-3a7f-413c-b377-b085659225b4" />



# 12. Air-Gapped Image Import

On Internet Machine:

```bash
docker pull nginx
docker save nginx -o nginx.tar
scp nginx.tar root@192.168.49.100:/root/
```

On Harbor VM:

```bash
docker load -i nginx.tar

docker tag nginx:latest 192.168.49.100/library/nginx:latest

docker login 192.168.49.100

docker push 192.168.49.100/library/nginx:latest
```

<img width="1225" height="238" alt="image" src="https://github.com/user-attachments/assets/5a76e5e5-193c-4576-b0e5-b7dac2274314" />


---

# 13. Harbor UI Verification

Opened:

[https://192.168.49.100](https://192.168.49.100/)

Verified:

Projects → library → Repositories → nginx

<img width="1915" height="964" alt="image" src="https://github.com/user-attachments/assets/d8aa6a18-f695-4975-b359-8dad57c31267" />



# 14. Troubleshooting Guide

## 14.1 SSH Timeout

Fixed by:

```bash
systemctl enable sshd
systemctl start sshd
```


## 14.2 TLS Unknown Authority

Cause: Self-signed server certificate

Fix: Implemented private CA and signed Harbor cert.


## 14.3 Docker Restart Stops Harbor

Fix:

```bash
docker compose up -d
```


## 14.4 Certificate SAN Missing

Verified with:

```bash
openssl x509 -in harbor.crt -text
```

Ensured SAN includes IP address.


# 15. Official References

Harbor Documentation: [https://goharbor.io/docs/](https://goharbor.io/docs/)

Harbor GitHub:[https://github.com/goharbor/harbor](https://github.com/goharbor/harbor)

Docker Official Docs:[https://docs.docker.com/](https://docs.docker.com/)

Docker RHEL Installation:[https://docs.docker.com/engine/install/rhel/](https://docs.docker.com/engine/install/rhel/)

Red Hat Documentation:[https://access.redhat.com/documentation/](https://access.redhat.com/documentation/)

OpenSSL Docs:[https://www.openssl.org/docs/](https://www.openssl.org/docs/)



