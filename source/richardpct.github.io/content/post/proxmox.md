---
title: "Automating Proxmox & Ceph with Ansible"
date: 2026-01-05T21:14:27+01:00
draft: false
---

Setting up a high-availability Proxmox VE (PVE) cluster with Ceph storage is a
rite of passage for many homelab enthusiasts. While the GUI is fantastic,
automating the process ensures reproducibility and saves time on rebuilds.
<br />
In this post, I’ll walk through an Ansible playbook I wrote to configure a
3-node PVE cluster, deploy Ceph (including a specific partition setup for NVMe),
and prepare the system for Terraform/OpenTofu provisioning.
<br />
You can find the full repository [here](https://github.com/richardpct/pve-homelab).

## Prerequisites

Before running the playbook, I assume a few things about the environment:

* **Hardware:** 3 nodes. I am using Beelink EQR7 mini-PCs
(8 Cores/16 Threads, 32 GB RAM, 1 TB NVMe disk).
* **OS:** Proxmox VE is already installed on all nodes.
* **Network:** Nodes have static IPs (e.g., 192.168.1.11 to .13).
* **SSH Access:** You must be able to SSH into the nodes as root without a password.
Ensure you have added your public key to the `/root/.ssh/authorized_keys` file
on each node.
* **Partitioning:** A specific partition on the disk is reserved for Ceph
(more on this later).

## 1. Inventory and Secrets

First, we define our cluster in `inventory/cluster.ini`:

```
pve-01 ansible_host=192.168.1.11
pve-02 ansible_host=192.168.1.12
pve-03 ansible_host=192.168.1.13

[cluster]
pve-01
pve-02
pve-03
```

To handle sensitive data like the PVE root password and the Terraform user
password, I use Ansible Vault.
<br />
First, create a directory for your secrets (outside the repo is safer) and
create the vault password file:

    mkdir -p ~/ansible_secrets
    echo "MY_VAULT_PASSWORD" > ~/ansible_secrets/vault
    chmod 600 ~/ansible_secrets/vault

(Replace `MY_VAULT_PASSWORD` with your actual secure password)
<br />
Next, generate the encrypted strings for your variables.

1. **Generate the encrypted Proxmox VE password** (Use the root password you
set during installation):

```
ansible-vault encrypt_string 'MY_PVE_PASS' --name pvepass --vault-password-file ~/ansible_secrets/vault
```

2. **Generate the encrypted Terraform provisionner password** (This user will
be used later for managing VMs via Terraform/OpenTofu):

```
ansible-vault encrypt_string 'MY_TOFU_PASS' --name terraform_prov_user --vault-password-file ~/ansible_secrets/vault
```

Finally, edit `inventory/group_vars/all.yml` and paste the encrypted strings
for `pvepass` and `terraform_prov_user`. You should also adjust the other
variables to match your environment.

```
pvepass: !vault |
          $ANSIBLE_VAULT;1.1;AES256...
public_network: 192.168.1.0/24
ceph_osd_disk: /dev/nvme0n1p4
ceph_part_start: 537GB
pool_name: mypool
terraform_prov_user: !vault |
          $ANSIBLE_VAULT;1.1;AES256...
```

`ceph_part_start`: This variable is crucial for my setup. It defines exactly
where the Ceph partition begins on the disk, allowing me to share the NVMe drive
between the OS and Ceph.

## 2. System Preparation

The first role of the playbook, `pve`, prepares the OS. Since this is a homelab,
we typically don't have an Enterprise subscription.
<br />
The playbook automatically:
* Configures Bash: Sets up nice-to-haves like colorized ls.
* Fixes Repositories: Removes the pve-enterprise repository and adds the
pve-no-subscription and ceph-squid repositories.
* Installs Packages: Installs essentials like vim, parted, and
prometheus-node-exporter.

## 3. Creating the PVE Cluster

Automation gets tricky when dealing with cluster joins because the command
usually requires interactive password input.

### The First Node

On the first node (`pve-01`), we initialize the cluster named "`mycluster`":
```
- name: create cluster on master node
  ansible.builtin.command: pvecm create mycluster
  args:
    creates: /etc/pve/corosync.conf
  when: inventory_hostname == groups['cluster'][0]
```

### Joining Nodes

For the other nodes, I use a shell task with a heredoc to pass the password
(which was decrypted from Ansible Vault) to `pvecm add`.
```
- name: join cluster on the nodes 2
  ansible.builtin.shell: |
    pvecm add "{{ hostvars['pve-01'].ansible_host }}" << EOF
    {{ pvepass }}
    yes
    EOF
  args:
    executable: /usr/bin/bash
    creates: /etc/pve/corosync.conf
  when: inventory_hostname == groups['cluster'][1]
```

The playbook also includes checks (`until: ... retries: 10`) to ensure the nodes
have successfully joined and are visible in `pvecm nodes` before proceeding.

## 4. Deploying Ceph

This is the most complex part of the setup. The playbook handles everything from
network initialization to OSD creation.

### Network and Packages

We install the ceph and ceph-common packages and initialize the Ceph network on
the `public_network` defined in our variables.
```
- name: configure ceph
  ansible.builtin.command: pveceph init --network "{{ public_network }}"
  args:
    creates: /etc/pve/priv/ceph.client.admin.keyring
  when: inventory_hostname == groups['cluster'][0]
```

### Monitors and Managers

The playbook iterates through the nodes to create Monitors (`pveceph mon create`)
on all three nodes and Managers (`pveceph mgr create`) on nodes 2 and 3.

### Custom Disk Partitioning

Instead of giving Ceph the whole disk, I use the `parted` module to create a
specific partition (partition 4) starting at `537GB`. This allows me to use the
beginning of the NVMe drive for the Proxmox OS and other data, while dedicating
the rest to Ceph.
```
- name: create partition disk for ceph
  community.general.parted:
    device: /dev/nvme0n1
    number: 4
    part_start: "{{ ceph_part_start }}"
    part_end: "100%"
    label: gpt
    state: present
  when: not ansible_check_mode
```
We then tell Proxmox to use this specific partition (`/dev/nvme0n1p4`) as an OSD:
```
- name: create ceph osd
  ansible.builtin.command: pveceph osd create "{{ ceph_osd_disk }}"
  args:
    creates: /run/systemd/system/ceph-osd.target.wants/ceph-osd@*.service
```

### Pools and Filesystems
Finally, the playbook creates a resource pool (`mypool`), sets it to autoscale,
and creates a CephFS filesystem. It even creates a subvolume group named
`kubernetes`, making the storage ready for a K8s CSI driver immediately.


## 5. Preparing for Terraform/OpenTofu (IaC)

To manage the virtual machines inside the cluster using Terraform/OpenTofu
later, we need a dedicated user with specific privileges.
<br />
The `ansible/roles/pve/tasks/telmate.yml` task file handles this:
* **Creates a Role:** A custom role TerraformProv is created with permissions like
VM.Allocate, Datastore.AllocateSpace, and SDN.Use. 
* **Creates a User:** The user terraform-prov@pve is created. 
* **Binds User to Role:** The user is assigned the role globally.

## 6. Usage

Now that the configuration is ready, we can deploy the cluster.

1. Clone the Repository

Clone this repository to your local machine:

    git clone https://github.com/richardpct/pve-homelab.git
    cd pve-homelab

2. Set Up the Environment

It is recommended to use a Python virtual environment to manage dependencies
and avoid conflicts with your system packages.
<br />
Navigate to the `ansible` directory:

    cd ansible

Create and activate the virtual environment:

    python3 -m venv venv
    source venv/bin/activate

3. Install Dependencies

Install the Ansible collections specified in the requirements file:

    pip install -r requirements.txt

4. Run the Playbook:

Execute the main playbook using your inventory file and the vault password file
you created earlier:

    ansible-playbook -i inventory/cluster.ini pve_cluster.yml --vault-password-file ~/ansible_secrets/vault

5. Access the GUI

Once the playbook finishes successfully, your cluster is ready.

* Open your web browser and navigate to the master node: https://192.168.1.11:8006
* Log in using the root user.
* Use the password you set during the initial OS installation of your nodes.

## Conclusion

By automating this process, I can rebuild my entire homelab cluster in minutes
rather than hours. This playbook takes three fresh Proxmox installations on
consumer hardware (like the Beelink EQR7) and turns them into a production-grade
hyperconverged infrastructure.
<br /><br />
The cluster is fully prepped for the next phase: Infrastructure as Code.
With the dedicated TerraformProv user and role in place, I am now ready to start
deploying Virtual Machines with Terraform/OpenTofu to build my Kubernetes
cluster.
