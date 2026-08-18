# Kubernetes Cluster with Vagrant, VirtualBox, kubeadm, Ansible, Flannel, RBAC, and NGINX

This project builds a three-node Kubernetes cluster using:

- Vagrant
- VirtualBox
- Ubuntu 24.04
- kubeadm
- containerd
- Kubernetes v1.34.x
- Flannel CNI
- Ansible
- Kubernetes RBAC
- NGINX
- Kubernetes Service
- Kubernetes DNS/CoreDNS

The goal is to provide a repeatable Kubernetes environment that can be completely destroyed and rebuilt using configuration stored in Git.

---

## Architecture

The cluster consists of three virtual machines:

```text
                         macOS Host
                             |
                         VirtualBox
                             |
              +--------------+--------------+
              |                             |
        NAT Network                  Host-Only Network
        10.0.2.0/24                  192.168.56.0/24
              |                             |
              +-----------------------------+
                             |
                +--------------------------+
                | Kubernetes Cluster       |
                +--------------------------+
                             |
          +------------------+------------------+
          |                  |                  |
          v                  v                  v
   controlplane           node01             node02
   192.168.56.10       192.168.56.11       192.168.56.12
   Control Plane          Worker              Worker
