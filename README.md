# Kubernetes Installation with Vagrant and Ansible

This project builds a three-node Kubernetes cluster using **Vagrant, VirtualBox, Ansible, kubeadm, containerd, and Flannel**.

The host Mac is used to run Vagrant and VirtualBox. Vagrant creates the following Ubuntu virtual machines:

| Host           | Role                                            | IP Address      |
| -------------- | ----------------------------------------------- | --------------- |
| `controlplane` | Kubernetes Control Plane and Ansible Controller | `192.168.56.10` |
| `node01`       | Kubernetes Worker                               | `192.168.56.11` |
| `node02`       | Kubernetes Worker                               | `192.168.56.12` |

Ansible is installed **inside the `controlplane` VM**.

The `controlplane` VM then acts as the Ansible controller and uses YAML playbooks to install and configure Kubernetes on all three machines.

The Kubernetes Pod network uses:

```text
10.244.0.0/16
```

Flannel provides the Kubernetes CNI network.

---

# Deployment Architecture

```text
Mac Host
   |
   | Vagrant
   v
VirtualBox
   |
   +----------------------+----------------------+
   |                      |                      |
   v                      v                      v
controlplane             node01                 node02
192.168.56.10            192.168.56.11          192.168.56.12
   |
   | Ansible installed here
   |
   +---------- Ansible SSH ----------+
   |                                 |
   +---------------> node01          |
   |
   +-------------------------------> node02
   |
   +--> Configures controlplane locally
   |
   v
Kubernetes Cluster
   |
   +--> kubeadm
   +--> containerd
   +--> kubelet
   +--> kubectl
   +--> Flannel
```

---

# 1. Install Vagrant on the Mac

Vagrant is used from the Mac to create and manage the virtual machines.

Verify Vagrant:

```bash
vagrant --version
```

---

# 2. Install VirtualBox on the Mac

VirtualBox provides the virtualization layer used by Vagrant.

Verify VirtualBox:

```bash
VBoxManage --version
```

---

# 3. Create the Virtual Machines

The `Vagrantfile` creates three Ubuntu virtual machines:

```text
controlplane
node01
node02
```

Start the environment from the Mac:

```bash
vagrant up
```

Verify:

```bash
vagrant status
```

Expected:

```text
controlplane    running
node01          running
node02          running
```

---

# 4. Connect to the Control Plane

From the Mac:

```bash
vagrant ssh controlplane
```

All Ansible commands are run from this VM.

---

# 5. Install Ansible on the Control Plane

Once logged into `controlplane`, update the package repository:

```bash
sudo apt update
```

Install Ansible:

```bash
sudo apt install -y ansible
```

Verify:

```bash
ansible --version
```

The `controlplane` VM now acts as the Ansible controller.

---

# 6. Verify Network Connectivity

From the `controlplane` VM, verify that the worker nodes are reachable:

```bash
ping -c 3 192.168.56.11
```

```bash
ping -c 3 192.168.56.12
```

Both worker nodes should respond.

---

# 7. Configure SSH Access for Ansible

Ansible needs SSH access from `controlplane` to the worker nodes.

Generate an SSH key on `controlplane` if one does not already exist:

```bash
ssh-keygen -t ed25519
```

Press Enter to accept the default file location.

Copy the key to `node01`:

```bash
ssh-copy-id vagrant@192.168.56.11
```

Copy the key to `node02`:

```bash
ssh-copy-id vagrant@192.168.56.12
```

Test:

```bash
ssh vagrant@192.168.56.11
```

Exit:

```bash
exit
```

Then:

```bash
ssh vagrant@192.168.56.12
```

Exit again:

```bash
exit
```

---

# 8. Create the Ansible Inventory

On `controlplane`, create:

```bash
nano inventory.ini
```

Example:

```ini
[controlplane]
controlplane ansible_connection=local

[workers]
node01 ansible_host=192.168.56.11 ansible_user=vagrant
node02 ansible_host=192.168.56.12 ansible_user=vagrant

[kubernetes:children]
controlplane
workers
```

The control plane is configured locally, while Ansible connects to the workers over SSH.

---

# 9. Test Ansible

Run from `controlplane`:

```bash
ansible all -i inventory.ini -m ping
```

Expected:

```text
controlplane | SUCCESS
node01       | SUCCESS
node02       | SUCCESS
```

Each host should return:

```text
"ping": "pong"
```

---

# 10. Create the Kubernetes Ansible Playbooks

The Kubernetes installation is automated with YAML playbooks.

The project contains:

```text
inventory.ini
site.yml
common.yml
controlplane.yml
workers.yml
```

The workflow is:

```text
site.yml
   |
   +--> common.yml
   |
   +--> controlplane.yml
   |
   +--> workers.yml
```

`common.yml` configures all Kubernetes systems.

`controlplane.yml` initializes Kubernetes and installs Flannel.

`workers.yml` joins the worker nodes to the Kubernetes cluster.

---

# 11. Run the Kubernetes Installation

From the `controlplane` VM:

```bash
ansible-playbook -i inventory.ini site.yml
```

Ansible performs the Kubernetes installation across all three virtual machines.

The playbooks automate:

1. Disabling swap
2. Loading kernel modules
3. Configuring sysctl networking
4. Installing containerd
5. Enabling systemd cgroups
6. Installing kubelet
7. Installing kubeadm
8. Installing kubectl
9. Configuring node IP addresses
10. Initializing the control plane
11. Installing Flannel
12. Generating the worker join command
13. Joining `node01`
14. Joining `node02`

---

# 12. Verify the Kubernetes Cluster

Because you are already logged into the control plane, run:

```bash
kubectl get nodes -o wide
```

Expected:

```text
NAME           STATUS   ROLES           INTERNAL-IP
controlplane   Ready    control-plane   192.168.56.10
node01         Ready    <none>          192.168.56.11
node02         Ready    <none>          192.168.56.12
```

Verify all system Pods:

```bash
kubectl get pods -A -o wide
```

Check the Kubernetes API:

```bash
kubectl cluster-info
```

Check API readiness:

```bash
kubectl get --raw='/readyz'
```

Expected:

```text
ok
```

---

# Summary

The deployment process is:

```text
Mac
 |
 +--> Vagrant
       |
       +--> controlplane
       +--> node01
       +--> node02

controlplane
 |
 +--> Install Ansible
 |
 +--> Run YAML playbooks
       |
       +--> Configure Linux
       +--> Install containerd
       +--> Install Kubernetes
       +--> kubeadm init
       +--> Install Flannel
       +--> Join workers

Result
 |
 +--> Three-node Kubernetes cluster
```

The important point is that **Ansible runs from the Kubernetes control-plane VM, not from the Mac host**.

