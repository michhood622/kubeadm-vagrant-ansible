# Kubernetes Installation with Ansible and Vagrant

This project builds a three-node Kubernetes cluster using **Vagrant, VirtualBox, Ansible, kubeadm, containerd, and Flannel**.

The goal is to automate the Kubernetes installation using **Ansible YAML playbooks** rather than manually installing and configuring Kubernetes on each virtual machine.

## Cluster Architecture

| Host           | Role                     | IP Address      |
| -------------- | ------------------------ | --------------- |
| `controlplane` | Kubernetes Control Plane | `192.168.56.10` |
| `node01`       | Kubernetes Worker        | `192.168.56.11` |
| `node02`       | Kubernetes Worker        | `192.168.56.12` |

The Kubernetes Pod network is:

```text
10.244.0.0/16
```

Flannel is used as the Kubernetes Container Network Interface (CNI).

The completed environment looks like:

```text
                         Mac Host
                            |
                      Ansible/Vagrant
                            |
                  VirtualBox Network
                      192.168.56.0/24
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
      +--------------+ +----------+ +----------+
      | controlplane | |  node01  | |  node02  |
      |192.168.56.10 | | .56.11   | | .56.12   |
      +--------------+ +----------+ +----------+
             |              |              |
             +--------------+--------------+
                            |
                       Flannel CNI
                            |
                       10.244.0.0/16
                            |
                  Kubernetes Pod Network
```

---

# 1. Install Ansible

Ansible is installed on the Mac and is used to configure all three Kubernetes virtual machines.

Install Ansible with Homebrew:

```bash
brew install ansible
```

Verify the installation:

```bash
ansible --version
```

---

# 2. Install Vagrant

Vagrant is used to create and manage the virtual machines.

Install Vagrant:

```bash
brew install --cask vagrant
```

Verify:

```bash
vagrant --version
```

---

# 3. Install VirtualBox

VirtualBox provides the virtualization platform used by Vagrant.

Install VirtualBox:

```bash
brew install --cask virtualbox
```

Verify:

```bash
VBoxManage --version
```

---

# 4. Project Structure

The project uses separate YAML playbooks for the common Kubernetes configuration, control-plane configuration, and worker-node configuration.

```text
kubernetes-ansible/
├── Vagrantfile
├── inventory.ini
├── site.yml
├── common.yml
├── controlplane.yml
├── workers.yml
└── README.md
```

The main Ansible files are:

* `inventory.ini` — defines the Kubernetes nodes.
* `common.yml` — installs and configures Kubernetes prerequisites on all nodes.
* `controlplane.yml` — initializes the Kubernetes control plane and installs Flannel.
* `workers.yml` — joins the worker nodes to the cluster.
* `site.yml` — runs the playbooks in the correct order.

---

# 5. Create the Virtual Machines

The `Vagrantfile` defines three Ubuntu virtual machines.

Create:

```bash
nano Vagrantfile
```

Add:

```ruby
Vagrant.configure("2") do |config|

  config.vm.box = "bento/ubuntu-22.04"

  config.vm.define "controlplane" do |cp|
    cp.vm.hostname = "controlplane"
    cp.vm.network "private_network", ip: "192.168.56.10"

    cp.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus = 2
    end
  end

  config.vm.define "node01" do |node|
    node.vm.hostname = "node01"
    node.vm.network "private_network", ip: "192.168.56.11"

    node.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 2
    end
  end

  config.vm.define "node02" do |node|
    node.vm.hostname = "node02"
    node.vm.network "private_network", ip: "192.168.56.12"

    node.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 2
    end
  end

end
```

Start the virtual machines:

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

# 6. Create the Ansible Inventory

Create:

```bash
nano inventory.ini
```

The inventory identifies the control plane and worker nodes:

```ini
[controlplane]
controlplane ansible_host=192.168.56.10 ansible_user=vagrant

[workers]
node01 ansible_host=192.168.56.11 ansible_user=vagrant
node02 ansible_host=192.168.56.12 ansible_user=vagrant

[kubernetes:children]
controlplane
workers
```

---

# 7. Test Ansible Connectivity

Before installing Kubernetes, verify that Ansible can communicate with all three VMs.

```bash
ansible all -i inventory.ini -m ping
```

A successful test should return:

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

# 8. Create the Common Kubernetes Playbook

Create:

```bash
nano common.yml
```

This playbook configures all three Kubernetes nodes.

```yaml
---
- name: Configure Kubernetes nodes
  hosts: kubernetes
  become: true

  tasks:

    - name: Disable swap
      command: swapoff -a

    - name: Disable swap permanently
      replace:
        path: /etc/fstab
        regexp: '^([^#].*\sswap\s.*)$'
        replace: '# \1'

    - name: Load overlay kernel module
      modprobe:
        name: overlay
        state: present

    - name: Load br_netfilter kernel module
      modprobe:
        name: br_netfilter
        state: present

    - name: Configure required kernel modules
      copy:
        dest: /etc/modules-load.d/k8s.conf
        content: |
          overlay
          br_netfilter

    - name: Configure Kubernetes networking
      copy:
        dest: /etc/sysctl.d/k8s.conf
        content: |
          net.bridge.bridge-nf-call-iptables = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward = 1

    - name: Apply sysctl configuration
      command: sysctl --system

    - name: Install required packages
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gpg
          - containerd
        state: present
        update_cache: yes
```

---

# 9. Configure containerd

Kubernetes requires a container runtime. This project uses **containerd**.

Add the following tasks to `common.yml`:

```yaml
    - name: Create containerd configuration directory
      file:
        path: /etc/containerd
        state: directory

    - name: Generate containerd configuration
      shell: containerd config default > /etc/containerd/config.toml

    - name: Enable systemd cgroups
      replace:
        path: /etc/containerd/config.toml
        regexp: 'SystemdCgroup = false'
        replace: 'SystemdCgroup = true'

    - name: Restart containerd
      systemd:
        name: containerd
        state: restarted
        enabled: yes
```

The important setting is:

```text
SystemdCgroup = true
```

This ensures that containerd and Kubernetes use compatible cgroup management.

---

# 10. Install Kubernetes Packages

The playbook installs the three primary Kubernetes packages:

* `kubelet`
* `kubeadm`
* `kubectl`

Add the Kubernetes repository and packages to `common.yml`.

```yaml
    - name: Create Kubernetes keyring directory
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Download Kubernetes repository key
      shell: |
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key |
        gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      args:
        creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    - name: Add Kubernetes repository
      copy:
        dest: /etc/apt/sources.list.d/kubernetes.list
        content: |
          deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /

    - name: Install Kubernetes packages
      apt:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present
        update_cache: yes

    - name: Hold Kubernetes packages
      dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop:
        - kubelet
        - kubeadm
        - kubectl
```

Holding the packages prevents unexpected Kubernetes upgrades.

---

# 11. Configure Kubernetes Node IP Addresses

This is an important configuration for Kubernetes running inside Vagrant.

Vagrant creates a NAT interface that commonly uses:

```text
10.0.2.15
```

Kubernetes should **not** use this interface for cluster communication.

The cluster must use:

```text
controlplane = 192.168.56.10
node01       = 192.168.56.11
node02       = 192.168.56.12
```

Add the following tasks to `common.yml`:

```yaml
    - name: Configure controlplane kubelet node IP
      copy:
        dest: /etc/default/kubelet
        content: |
          KUBELET_EXTRA_ARGS=--node-ip=192.168.56.10
      when: inventory_hostname == "controlplane"

    - name: Configure node01 kubelet node IP
      copy:
        dest: /etc/default/kubelet
        content: |
          KUBELET_EXTRA_ARGS=--node-ip=192.168.56.11
      when: inventory_hostname == "node01"

    - name: Configure node02 kubelet node IP
      copy:
        dest: /etc/default/kubelet
        content: |
          KUBELET_EXTRA_ARGS=--node-ip=192.168.56.12
      when: inventory_hostname == "node02"

    - name: Restart kubelet
      systemd:
        name: kubelet
        state: restarted
        daemon_reload: yes
```

This ensures that Kubernetes registers each node using its `192.168.56.x` address.

---

# 12. Create the Control Plane Playbook

Create:

```bash
nano controlplane.yml
```

The control-plane playbook initializes Kubernetes using `kubeadm`.

```yaml
---
- name: Configure Kubernetes control plane
  hosts: controlplane
  become: true

  tasks:

    - name: Initialize Kubernetes control plane
      command: >
        kubeadm init
        --apiserver-advertise-address=192.168.56.10
        --pod-network-cidr=10.244.0.0/16
      args:
        creates: /etc/kubernetes/admin.conf
```

The Kubernetes API server uses:

```text
192.168.56.10
```

The Kubernetes Pod network uses:

```text
10.244.0.0/16
```

---

# 13. Configure kubectl

After `kubeadm init`, Kubernetes creates:

```text
/etc/kubernetes/admin.conf
```

Add the following tasks to `controlplane.yml`:

```yaml
    - name: Create kube directory for vagrant user
      file:
        path: /home/vagrant/.kube
        state: directory
        owner: vagrant
        group: vagrant
        mode: '0755'

    - name: Copy Kubernetes admin configuration
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/vagrant/.kube/config
        remote_src: yes
        owner: vagrant
        group: vagrant
        mode: '0600'
```

The `vagrant` user can now use `kubectl` without sudo.

---

# 14. Install Flannel CNI

Kubernetes requires a CNI plugin so Pods can communicate across nodes.

This project uses **Flannel**.

Add to `controlplane.yml`:

```yaml
    - name: Install Flannel CNI
      become_user: vagrant
      command: >
        kubectl apply -f
        https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Flannel uses:

```text
10.244.0.0/16
```

which matches the network specified during `kubeadm init`.

---

# 15. Generate the Kubernetes Join Command

Worker nodes need a `kubeadm join` command to join the cluster.

Add to `controlplane.yml`:

```yaml
    - name: Generate Kubernetes worker join command
      command: kubeadm token create --print-join-command
      register: join_command

    - name: Display join command
      debug:
        var: join_command.stdout
```

The command generated by kubeadm will look similar to:

```text
kubeadm join 192.168.56.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Ansible can then pass this command to the worker nodes.

---

# 16. Create the Worker Playbook

Create:

```bash
nano workers.yml
```

The worker playbook joins `node01` and `node02` to the Kubernetes cluster.

The join operation ultimately executes:

```text
kubeadm join 192.168.56.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

The playbook should only execute the join operation if the worker has not already joined the cluster.

For example:

```yaml
---
- name: Join Kubernetes worker nodes
  hosts: workers
  become: true

  tasks:

    - name: Join worker to Kubernetes cluster
      command: "{{ hostvars['controlplane']['join_command']['stdout'] }}"
      args:
        creates: /etc/kubernetes/kubelet.conf
```

The `creates` condition helps make the task idempotent by preventing an already-joined node from attempting to join again.

---

# 17. Create the Main Ansible Playbook

Create:

```bash
nano site.yml
```

Add:

```yaml
---
- import_playbook: common.yml
- import_playbook: controlplane.yml
- import_playbook: workers.yml
```

The `site.yml` playbook controls the order of the Kubernetes installation.

```text
site.yml
   |
   +--> common.yml
   |
   +--> controlplane.yml
   |
   +--> workers.yml
```

---

# 18. Run the Kubernetes Installation

From the Mac, run:

```bash
ansible-playbook -i inventory.ini site.yml
```

Ansible will now automate the Kubernetes installation.

The process performs:

1. Linux configuration
2. Swap configuration
3. Kernel module configuration
4. Network/sysctl configuration
5. containerd installation
6. containerd configuration
7. Kubernetes repository configuration
8. kubelet installation
9. kubeadm installation
10. kubectl installation
11. Node IP configuration
12. Kubernetes control-plane initialization
13. kubectl configuration
14. Flannel installation
15. Worker join command generation
16. Worker-node cluster join

A successful playbook run should finish with something similar to:

```text
PLAY RECAP

controlplane : ok=XX changed=XX unreachable=0 failed=0
node01       : ok=XX changed=XX unreachable=0 failed=0
node02       : ok=XX changed=XX unreachable=0 failed=0
```

The important values are:

```text
unreachable=0
failed=0
```

---

# 19. Connect to the Control Plane

Connect to the Kubernetes control plane:

```bash
vagrant ssh controlplane
```

Verify the cluster:

```bash
kubectl get nodes
```

Expected:

```text
NAME           STATUS   ROLES
controlplane   Ready    control-plane
node01         Ready    <none>
node02         Ready    <none>
```

---

# 20. Verify Node IP Addresses

Run:

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

If a node shows `10.0.2.15`, check:

```bash
cat /etc/default/kubelet
```

The correct `--node-ip` should be configured for that machine.

---

# 21. Verify Kubernetes System Pods

Run:

```bash
kubectl get pods -A -o wide
```

Important components should show `Running`, including:

```text
coredns
etcd
kube-apiserver
kube-controller-manager
kube-proxy
kube-scheduler
kube-flannel
```

---

# 22. Verify Flannel

Check the Flannel Pods:

```bash
kubectl get pods -n kube-flannel -o wide
```

There should be one Flannel Pod running on each Kubernetes node.

Verify the Flannel public IP addresses:

```bash
kubectl get nodes \
-o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.metadata.annotations.flannel\.alpha\.coreos\.com/public-ip}{"\n"}{end}'
```

Expected:

```text
controlplane => 192.168.56.10
node01 => 192.168.56.11
node02 => 192.168.56.12
```

This confirms that Flannel is using the correct VirtualBox private network.

---

# 23. Verify the Kubernetes API Server

Run:

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

# 24. Deploy a Test NGINX Application

Create a namespace:

```bash
kubectl create namespace nginx-demo
```

Deploy two NGINX replicas:

```bash
kubectl create deployment nginx \
  --image=nginx:stable \
  --replicas=2 \
  -n nginx-demo
```

Verify:

```bash
kubectl get pods -n nginx-demo -o wide
```

Example:

```text
NAME                     READY   STATUS    IP           NODE
nginx-xxxxxxxxxx-aaaaa   1/1     Running   10.244.1.x   node01
nginx-xxxxxxxxxx-bbbbb   1/1     Running   10.244.2.x   node02
```

This also verifies that the scheduler can place workloads on the worker nodes.

---

# 25. Create an NGINX Service

Expose NGINX internally:

```bash
kubectl expose deployment nginx \
  --name=nginx-service \
  --port=80 \
  --target-port=80 \
  -n nginx-demo
```

Verify:

```bash
kubectl get svc -n nginx-demo
```

---

# 26. Test NGINX

Create a temporary test Pod:

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  -n nginx-demo \
  -- curl -s http://nginx-service
```

Check the result:

```bash
kubectl logs curl-test -n nginx-demo
```

The output should contain the NGINX welcome page HTML.

Remove the test Pod:

```bash
kubectl delete pod curl-test -n nginx-demo
```

---

# 27. Verify Pod Networking

Run:

```bash
kubectl get pods -n nginx-demo -o wide
```

The Pod IPs should use the Flannel network.

For example:

```text
node01 pod -> 10.244.1.x
node02 pod -> 10.244.2.x
```

This demonstrates that Flannel is providing Pod networking across the Kubernetes nodes.

---

# 28. Useful Kubernetes Troubleshooting Commands

## Check Nodes

```bash
kubectl get nodes -o wide
```

## Check All Pods

```bash
kubectl get pods -A -o wide
```

## Describe a Pod

```bash
kubectl describe pod <pod-name> -n <namespace>
```

## View Pod Logs

```bash
kubectl logs <pod-name> -n <namespace>
```

## View Previous Container Logs

Useful for `CrashLoopBackOff`:

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

## Check Cluster Events

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

## Check Services

```bash
kubectl get svc -A
```

## Check Endpoints

```bash
kubectl get endpoints -A
```

## Check kubelet

```bash
sudo systemctl status kubelet
```

## Check kubelet Logs

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

## Check containerd

```bash
sudo systemctl status containerd
```

## Check containerd Logs

```bash
sudo journalctl -u containerd -n 100 --no-pager
```

## Check Network Interfaces

```bash
ip -br address
```

Look for interfaces such as:

```text
flannel.1
cni0
```

---

# 29. Common Issue: Node Uses the Wrong IP Address

If:

```bash
kubectl get nodes -o wide
```

shows:

```text
10.0.2.15
```

instead of a `192.168.56.x` address, check:

```bash
cat /etc/default/kubelet
```

The control plane should contain:

```text
KUBELET_EXTRA_ARGS=--node-ip=192.168.56.10
```

`node01` should contain:

```text
KUBELET_EXTRA_ARGS=--node-ip=192.168.56.11
```

`node02` should contain:

```text
KUBELET_EXTRA_ARGS=--node-ip=192.168.56.12
```

After correcting the configuration:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---

# 30. Common Issue: kubectl Connects to localhost:8080

If `kubectl` returns an error similar to:

```text
couldn't get current server API group list:
Get "http://localhost:8080/api?timeout=32s"
```

verify that the kubeconfig exists:

```bash
ls -l ~/.kube/config
```

The control-plane configuration originates from:

```text
/etc/kubernetes/admin.conf
```

The Ansible control-plane playbook copies this file to:

```text
/home/vagrant/.kube/config
```

---

# 31. Common Issue: Nodes Remain NotReady

Check:

```bash
kubectl get nodes
```

Then check Flannel:

```bash
kubectl get pods -n kube-flannel -o wide
```

Check recent events:

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

Check networking:

```bash
ip -br address
```

Check kubelet:

```bash
sudo systemctl status kubelet
```

Check kubelet logs:

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

---

# 32. Common Issue: Worker Cannot Join the Cluster

Verify that the worker can reach the Kubernetes API server:

```bash
nc -vz 192.168.56.10 6443
```

If a new join command is needed, run on the control plane:

```bash
sudo kubeadm token create --print-join-command
```

The join operation must be executed with root privileges.

---

# 33. Destroy and Rebuild the Cluster

One of the primary benefits of this project is repeatability.

Destroy the existing virtual machines:

```bash
vagrant destroy -f
```

Recreate them:

```bash
vagrant up
```

Verify Ansible connectivity:

```bash
ansible all -i inventory.ini -m ping
```

Run the Kubernetes installation again:

```bash
ansible-playbook -i inventory.ini site.yml
```

Connect to the control plane:

```bash
vagrant ssh controlplane
```

Verify:

```bash
kubectl get nodes -o wide
```

The complete Kubernetes environment should be recreated using the Ansible YAML automation.

---

# 34. Complete Deployment Workflow

Once the YAML files and Vagrant configuration exist, the normal deployment process is:

### Start the virtual machines

```bash
vagrant up
```

### Verify the machines

```bash
vagrant status
```

### Test Ansible connectivity

```bash
ansible all -i inventory.ini -m ping
```

### Run the Kubernetes installation

```bash
ansible-playbook -i inventory.ini site.yml
```

### Connect to the control plane

```bash
vagrant ssh controlplane
```

### Verify Kubernetes

```bash
kubectl get nodes -o wide
```

### Verify Kubernetes Pods

```bash
kubectl get pods -A -o wide
```

### Verify the API server

```bash
kubectl get --raw='/readyz'
```

The expected result is:

```text
controlplane   Ready   192.168.56.10
node01         Ready   192.168.56.11
node02         Ready   192.168.56.12
```

---

# How the Automation Works

The complete deployment flow is:

```text
Mac
 |
 +--> Ansible
 |
 +--> Vagrant / VirtualBox
 |        |
 |        +--> controlplane
 |        +--> node01
 |        +--> node02
 |
 +--> site.yml
        |
        +--> common.yml
        |      |
        |      +--> Disable swap
        |      +--> Configure kernel modules
        |      +--> Configure Linux networking
        |      +--> Install containerd
        |      +--> Configure systemd cgroups
        |      +--> Install Kubernetes packages
        |      +--> Configure kubelet node IPs
        |
        +--> controlplane.yml
        |      |
        |      +--> kubeadm init
        |      +--> Configure kubectl
        |      +--> Install Flannel
        |      +--> Generate worker join command
        |
        +--> workers.yml
               |
               +--> Join node01
               +--> Join node02
```

---

# Technologies Demonstrated

This project demonstrates hands-on experience with:

* Kubernetes
* kubeadm
* kubelet
* kubectl
* Ansible
* Ansible YAML playbooks
* Linux administration
* containerd
* Flannel CNI
* Kubernetes networking
* VirtualBox
* Vagrant
* Infrastructure automation
* YAML
* Kubernetes troubleshooting
* Automated cluster provisioning

---

# Summary

This project demonstrates how **Ansible YAML playbooks can automate the installation and configuration of a multi-node Kubernetes cluster**.

Instead of manually installing Kubernetes on each server, the cluster configuration is defined as code.

The environment consists of:

```text
1 Kubernetes Control Plane
2 Kubernetes Worker Nodes
containerd Container Runtime
Flannel CNI
Ansible Automation
Vagrant / VirtualBox Infrastructure
```

The primary deployment command is:

```bash
ansible-playbook -i inventory.ini site.yml
```

After the playbook completes, the result is a functional three-node Kubernetes cluster with the control plane and workers communicating over the `192.168.56.0/24` private network and Pods communicating through the `10.244.0.0/16` Flannel network.

The cluster can then be validated with:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl cluster-info
```

This provides a repeatable Kubernetes lab environment that can be destroyed, recreated, and configured using Ansible automation.

