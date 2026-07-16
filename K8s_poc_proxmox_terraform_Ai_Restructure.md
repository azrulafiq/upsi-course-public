# Building a Kubernetes Cluster on Proxmox with Terraform
## A Step-by-Step Lab Guide — 3 Control Plane + 5 Worker Nodes

**Time required:** ~3–4 hours first run · **Level:** IT professionals new to Kubernetes/Terraform

---

## 0. How to read this guide

### 0.1 You will work on FOUR different machines

The single most common mistake in this lab is running a command on the wrong
machine. Every command block in this guide is labelled:

| Label | Meaning | How you get there |
|---|---|---|
| 🖥️ **WORKSTATION** | Your own laptop/PC | You're already there |
| 🏠 **PROXMOX** | The Proxmox VE server shell | SSH: `ssh root@<proxmox-ip>` or GUI → node → Shell |
| 🎛️ **CP-01** | The *first* control-plane VM only | `ssh ubuntu@10.10.10.11` |
| 📦 **ALL NODES** | Every one of the 8 VMs (done via Ansible for you) | You won't type these manually |

If a block has no label, it continues on the same machine as the previous block.

### 0.2 Every phase ends with a ✅ CHECKPOINT

Do **not** move to the next phase until the checkpoint passes. If it fails,
jump to §12 Troubleshooting, fix it, re-check, then continue. Skipping a
checkpoint means debugging three problems at once later.

### 0.3 Words you'll meet (30-second glossary)

| Term | Plain meaning |
|---|---|
| **Terraform** | A tool that creates VMs from a text description. You describe 8 VMs in files; it builds them. Delete the files' resources → it deletes the VMs. |
| **Provider** | Terraform's plugin for talking to a specific platform — here, Proxmox. |
| **cloud-init** | A first-boot configuration system inside the VM image. Sets hostname, IP, SSH keys automatically so you never touch a console. |
| **Ansible** | A tool that runs the same setup commands on many machines over SSH. We use it so we don't repeat 20 commands on 8 VMs by hand. |
| **kubeadm** | The official Kubernetes bootstrapper. Turns prepared Linux machines into a cluster. |
| **Control plane (master)** | Nodes that run the cluster's "brain": API server, scheduler, etcd database. We run 3 so the brain survives one failure. |
| **etcd** | The database holding the entire cluster state. Needs 2 of 3 members alive ("quorum") to accept writes. |
| **Worker** | Nodes that run your actual applications (in Pods). |
| **VIP** | Virtual IP — one floating address (10.10.10.100) that always points at a *living* control-plane node. |
| **CNI** | The pod networking plugin. Until one is installed, nodes show `NotReady`. That is normal, not an error. |

### 0.4 The finished picture

```
                YOUR WORKSTATION
                (terraform / ansible / kubectl)
                        │
                        ▼
     ┌────────────────────────────────────────┐
     │        PROXMOX HOST (one server)       │
     │                                        │
     │   VIP 10.10.10.100:6443 ◄─ kube-vip    │
     │      │         │         │             │
     │  ┌───┴───┐ ┌───┴───┐ ┌───┴───┐         │
     │  │ cp-01 │ │ cp-02 │ │ cp-03 │  etcd×3 │
     │  │  .11  │ │  .12  │ │  .13  │         │
     │  └───────┘ └───────┘ └───────┘         │
     │  ┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐
     │  │wk-01 ││wk-02 ││wk-03 ││wk-04 ││wk-05 │
     │  │ .21  ││ .22  ││ .23  ││ .24  ││ .25  │
     │  └──────┘└──────┘└──────┘└──────┘└──────┘
     └────────────────────────────────────────┘
```

**Honest note:** because all 8 VMs sit on ONE physical server, this cluster is
highly available *in software only*. If the Proxmox host reboots, everything
goes down. That's acceptable — the purpose is to learn HA behaviour, not to
achieve it.

---

## 1. Requirements

### 1.1 The Proxmox server

| Resource | Minimum | Why |
|---|---|---|
| CPU | 16 physical cores | We allocate 26 vCPU total; ~1.6× overcommit is fine for a lab |
| RAM | 64 GB | VMs use 52 GB. RAM is **not** safely overcommittable here |
| Disk | 500 GB **SSD or NVMe** | etcd writes to disk constantly. On a spinning HDD the cluster will randomly lose its leader. SSD is not optional. |
| Proxmox VE | 8.2 or newer | Needed for the disk-import feature Terraform uses |

Per-VM sizing (Terraform sets this automatically):

| Role | Count | vCPU | RAM | Disk |
|---|---|---|---|---|
| Control plane | 3 | 2 | 4 GB | 40 GB |
| Worker | 5 | 4 | 8 GB | 60 GB |

### 1.2 Your workstation

Linux, macOS, or Windows with WSL2. Install:

🖥️ **WORKSTATION**
```bash
# Check what you already have
terraform version    # need 1.9+
ansible --version    # need 2.16+  (ansible-core)
kubectl version --client   # need 1.31.x ideally
helm version         # need 3.15+
```

If missing (Ubuntu/WSL2 example):

```bash
# Terraform — official HashiCorp repo
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform

# Ansible
sudo apt install -y pipx && pipx install --include-deps ansible

# kubectl 1.31
curl -LO "https://dl.k8s.io/release/v1.31.4/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/ && rm kubectl

# Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 1.3 Network plan — print this and keep it beside you

| Item | Value | Rule |
|---|---|---|
| VM subnet | `10.10.10.0/24` on bridge `vmbr0` | Adjust to your lab — but then adjust it **everywhere** |
| Gateway / DNS | `10.10.10.1` | Your lab router |
| Control-plane VIP | `10.10.10.100` | Must be an UNUSED IP, outside DHCP range |
| Control planes | `.11 .12 .13` | Static, outside DHCP range |
| Workers | `.21 – .25` | Static, outside DHCP range |
| MetalLB pool | `.200 – .240` | Unused range, outside DHCP — apps get IPs from here |
| Pod network | `10.244.0.0/16` | Internal only. Must NOT overlap the VM subnet |
| Service network | `10.96.0.0/12` | Internal only. Must NOT overlap the VM subnet |

> ⚠️ If your lab LAN is already `10.244.x.x` or `10.96.x.x`, change the pod/service
> CIDRs in §7.3 or pods will not reach the LAN.

### 1.4 SSH key — create one if you don't have it

🖥️ **WORKSTATION**
```bash
ls ~/.ssh/id_ed25519.pub 2>/dev/null || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

The output (one line starting `ssh-ed25519 AAAA…`) is your **public key**. It
gets injected into every VM so you can log in without passwords. The file
*without* `.pub` is your **private key** — never paste it anywhere.

### ✅ CHECKPOINT 1
- [ ] `terraform version` ≥ 1.9, `ansible --version` ≥ 2.16, `kubectl`, `helm` all print versions
- [ ] You can open the Proxmox GUI at `https://<proxmox-ip>:8006`
- [ ] You wrote down your actual subnet/gateway if different from the table
- [ ] `cat ~/.ssh/id_ed25519.pub` prints a key

---

## 2. Prepare the Proxmox host (one-time, ~10 min)

Everything in this phase runs on the Proxmox server itself.

🏠 **PROXMOX** — open a shell as root:
```bash
ssh root@<proxmox-ip>
pveversion        # must show pve-manager/8.2 or newer
```

### 2.1 Create a dedicated API user + token for Terraform

Why: Terraform should not use your root password. A token can be revoked
anytime and is limited to VM-related permissions.

```bash
pveum user add terraform@pve

pveum role add TerraformRole -privs \
  "Datastore.Allocate Datastore.AllocateSpace Datastore.Audit \
   Pool.Allocate Sys.Audit Sys.Console Sys.Modify \
   VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit \
   VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory \
   VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor \
   VM.PowerMgmt SDN.Use"

pveum aclmod / -user terraform@pve -role TerraformRole

pveum user token add terraform@pve tfprovider --privsep=0
```

The last command prints something like:

```
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
├──────────────┼──────────────────────────────────────┤
│ full-tokenid │ terraform@pve!tfprovider             │
│ value        │ 12345678-abcd-ef00-1111-222233334444 │
└──────────────┴──────────────────────────────────────┘
```

> 🛑 **STOP — copy the `value` NOW into a password manager.** Proxmox shows it
> exactly once. Your final token string is the two parts joined with `=`:
> `terraform@pve!tfprovider=12345678-abcd-ef00-1111-222233334444`

### 2.2 Allow the `local` datastore to hold snippets and imports

Why: Terraform uploads a small cloud-init file ("snippet") and the Ubuntu
disk image ("import") to Proxmox storage. By default the `local` datastore
doesn't accept those content types.

```bash
pvesm set local --content backup,iso,vztmpl,snippets,import
pvesm status     # confirm 'local' is listed and active
```

### 2.3 Let Terraform SSH into the host

Why: the Proxmox HTTP API cannot import disk images; the Terraform provider
does that part over SSH. For a lab, root SSH with your key is the simple path.

🖥️ **WORKSTATION**
```bash
ssh-copy-id root@<proxmox-ip>
ssh root@<proxmox-ip> hostname    # must log in WITHOUT a password prompt
```

### ✅ CHECKPOINT 2
- [ ] Token string saved (format `terraform@pve!tfprovider=<uuid>`)
- [ ] `pvesm status` shows `local` and your VM datastore (e.g. `local-lvm`) active
- [ ] `ssh root@<proxmox-ip> hostname` works with no password

---

## 3. Create the project files (~15 min)

🖥️ **WORKSTATION**
```bash
mkdir -p ~/k8s-poc/{terraform,ansible/group_vars,manifests}
cd ~/k8s-poc/terraform
```

Final layout you're building toward:

```
k8s-poc/
├── terraform/
│   ├── versions.tf        # which Terraform + provider versions
│   ├── providers.tf       # how to reach Proxmox
│   ├── variables.tf       # every knob, DECLARED
│   ├── terraform.tfvars   # your VALUES for those knobs (gitignored)
│   ├── image.tf           # downloads Ubuntu cloud image
│   ├── nodes.tf           # the 8 VMs
│   └── outputs.tf         # writes the Ansible inventory
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini      # ← Terraform GENERATES this file
│   ├── group_vars/all.yml
│   └── site.yml           # the whole node-preparation playbook
└── manifests/
    ├── kubeadm-config.yaml
    └── metallb-pool.yaml
```

Create each file below exactly as shown (`nano versions.tf`, paste, save —
in nano: `Ctrl+O`, `Enter`, `Ctrl+X`).

### 3.1 `terraform/versions.tf`

```hcl
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.66"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}
```

> Why `bpg/proxmox`? There are two Proxmox providers. The older
> `Telmate/proxmox` is unmaintained and breaks with modern cloud-init.
> Use `bpg`. Always.

### 3.2 `terraform/providers.tf`

```hcl
provider "proxmox" {
  endpoint  = var.pve_endpoint
  api_token = var.pve_api_token
  insecure  = true   # lab only: accepts Proxmox's self-signed certificate

  ssh {
    agent       = false
    username    = "root"
    private_key = file(var.pve_ssh_key)

    node {
      name    = var.pve_node
      address = var.pve_node_ip
    }
  }
}
```

### 3.3 `terraform/variables.tf`

This file only *declares* the knobs and their defaults. Your actual values go
in `terraform.tfvars` (§3.4).

> **HCL syntax rule:** inside a block, each argument goes on its own line.
> `variable "x" { type = string, default = "y" }` — the comma makes this
> INVALID and Terraform will refuse to parse it.

```hcl
# ---------- Proxmox connection ----------

variable "pve_endpoint" {
  description = "Proxmox API URL, e.g. https://10.10.10.5:8006/"
  type        = string
}

variable "pve_api_token" {
  description = "Format: terraform@pve!tfprovider=<uuid>"
  type        = string
  sensitive   = true
}

variable "pve_node" {
  description = "Node name exactly as shown in the Proxmox GUI sidebar"
  type        = string
  default     = "pve01"
}

variable "pve_node_ip" {
  description = "IP address Terraform SSHes to for image import"
  type        = string
}

variable "pve_ssh_key" {
  description = "Path to your PRIVATE key on this workstation"
  type        = string
  default     = "~/.ssh/id_ed25519"
}

# ---------- Infrastructure ----------

variable "vm_storage" {
  description = "Datastore for VM disks (check GUI: local-lvm, local-zfs, ...)"
  type        = string
  default     = "local-lvm"
}

variable "snippet_storage" {
  description = "Datastore that accepts snippets/import (we enabled 'local')"
  type        = string
  default     = "local"
}

variable "bridge" {
  type    = string
  default = "vmbr0"
}

variable "gateway" {
  type    = string
  default = "10.10.10.1"
}

variable "nameserver" {
  type    = string
  default = "10.10.10.1"
}

variable "ssh_public_key" {
  description = "Content of your ~/.ssh/id_ed25519.pub — injected into every VM"
  type        = string
}

# ---------- The 8 nodes ----------
# Commas ARE allowed inside map values and object({...}) types —
# the earlier no-comma rule applies only between a block's arguments.

variable "control_planes" {
  type = map(object({
    vmid = number
    ip   = string
  }))
  default = {
    "k8s-cp-01" = { vmid = 9011, ip = "10.10.10.11/24" }
    "k8s-cp-02" = { vmid = 9012, ip = "10.10.10.12/24" }
    "k8s-cp-03" = { vmid = 9013, ip = "10.10.10.13/24" }
  }
}

variable "workers" {
  type = map(object({
    vmid = number
    ip   = string
  }))
  default = {
    "k8s-wk-01" = { vmid = 9021, ip = "10.10.10.21/24" }
    "k8s-wk-02" = { vmid = 9022, ip = "10.10.10.22/24" }
    "k8s-wk-03" = { vmid = 9023, ip = "10.10.10.23/24" }
    "k8s-wk-04" = { vmid = 9024, ip = "10.10.10.24/24" }
    "k8s-wk-05" = { vmid = 9025, ip = "10.10.10.25/24" }
  }
}
```

### 3.4 `terraform/terraform.tfvars` — YOUR values

Terraform reads values in this order (later wins): defaults in `variables.tf`
→ `terraform.tfvars` → environment variables `TF_VAR_*` → `-var` flags.

```hcl
# EDIT every value to match YOUR lab.

pve_endpoint = "https://10.10.10.5:8006/"
pve_node     = "pve01"          # sidebar name in the GUI, case-sensitive
pve_node_ip  = "10.10.10.5"

vm_storage      = "local-lvm"
snippet_storage = "local"
bridge          = "vmbr0"
gateway         = "10.10.10.1"
nameserver      = "10.10.10.1"

# Deliberately NOT set here (secrets don't belong in files):
#   pve_api_token   -> set via environment variable in §4
#   ssh_public_key  -> set via environment variable in §4
```

Protect the directory from accidental git commits:

```bash
cat > ~/k8s-poc/.gitignore <<'EOF'
terraform.tfvars
*.auto.tfvars
.terraform/
*.tfstate
*.tfstate.*
EOF
```

### 3.5 `terraform/image.tf`

Downloads the official Ubuntu 24.04 cloud image once, plus a small
"vendor" cloud-init file that runs on every VM's first boot (installs the
guest agent, disables swap — Kubernetes refuses to run with swap on).

```hcl
resource "proxmox_virtual_environment_download_file" "ubuntu_2404" {
  content_type = "import"
  datastore_id = var.snippet_storage
  node_name    = var.pve_node
  url          = "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
  file_name    = "noble-server-cloudimg-amd64.qcow2"
  overwrite    = false
}

resource "proxmox_virtual_environment_file" "vendor_cfg" {
  content_type = "snippets"
  datastore_id = var.snippet_storage
  node_name    = var.pve_node

  source_raw {
    file_name = "k8s-vendor.yaml"
    data      = <<-EOT
      #cloud-config
      package_update: true
      packages:
        - qemu-guest-agent
        - apt-transport-https
        - ca-certificates
        - curl
        - gpg
      runcmd:
        - systemctl enable --now qemu-guest-agent
        - swapoff -a
        - sed -i '/\sswap\s/d' /etc/fstab
    EOT
  }
}
```

### 3.6 `terraform/nodes.tf`

One resource block, stamped out 8 times via `for_each`. Control planes and
workers get different sizes through the `local.all_nodes` map.

```hcl
locals {
  all_nodes = merge(
    { for k, v in var.control_planes : k => merge(v, { role = "cp", cores = 2, memory = 4096, disk = 40 }) },
    { for k, v in var.workers : k => merge(v, { role = "worker", cores = 4, memory = 8192, disk = 60 }) },
  )
}

resource "proxmox_virtual_environment_vm" "node" {
  for_each = local.all_nodes

  name      = each.key
  vm_id     = each.value.vmid
  node_name = var.pve_node
  tags      = ["k8s", "poc", each.value.role]

  agent {
    enabled = true
  }

  cpu {
    cores = each.value.cores
    type  = "host"
  }

  memory {
    dedicated = each.value.memory
    floating  = 0   # disable ballooning; etcd needs stable memory
  }

  disk {
    datastore_id = var.vm_storage
    import_from  = proxmox_virtual_environment_download_file.ubuntu_2404.id
    interface    = "scsi0"
    size         = each.value.disk
    ssd          = true
    discard      = "on"
    iothread     = true
  }

  network_device {
    bridge = var.bridge
    model  = "virtio"
  }

  operating_system {
    type = "l26"
  }

  scsi_hardware = "virtio-scsi-single"

  initialization {
    datastore_id        = var.vm_storage
    vendor_data_file_id = proxmox_virtual_environment_file.vendor_cfg.id

    ip_config {
      ipv4 {
        address = each.value.ip
        gateway = var.gateway
      }
    }

    dns {
      servers = [var.nameserver]
    }

    user_account {
      username = "ubuntu"
      keys     = [trimspace(var.ssh_public_key)]
    }
  }

  lifecycle {
    ignore_changes = [initialization[0].user_account]
  }
}
```

### 3.7 `terraform/outputs.tf`

After building the VMs, Terraform writes the Ansible inventory file for you —
so the IP list can never drift out of sync between the two tools.

```hcl
resource "local_file" "inventory" {
  filename        = "${path.module}/../ansible/inventory.ini"
  file_permission = "0644"

  content = <<-EOT
    [control_plane]
    %{for k, v in var.control_planes~}
    ${k} ansible_host=${split("/", v.ip)[0]}
    %{endfor~}

    [workers]
    %{for k, v in var.workers~}
    ${k} ansible_host=${split("/", v.ip)[0]}
    %{endfor~}

    [k8s:children]
    control_plane
    workers

    [k8s:vars]
    ansible_user=ubuntu
    ansible_python_interpreter=/usr/bin/python3
  EOT
}

output "control_plane_ips" {
  value = { for k, v in var.control_planes : k => split("/", v.ip)[0] }
}

output "worker_ips" {
  value = { for k, v in var.workers : k => split("/", v.ip)[0] }
}
```

### ✅ CHECKPOINT 3
🖥️ **WORKSTATION**, in `~/k8s-poc/terraform`:
```bash
ls
# versions.tf providers.tf variables.tf terraform.tfvars image.tf nodes.tf outputs.tf
```
- [ ] All 7 files exist
- [ ] `terraform.tfvars` contains **your** endpoint, node name, storage names
- [ ] `.gitignore` created

---

## 4. Build the 8 VMs with Terraform (~10 min)

### 4.1 Provide the two secrets as environment variables

🖥️ **WORKSTATION**
```bash
cd ~/k8s-poc/terraform

# The leading space keeps the token out of your shell history (bash default)
 export TF_VAR_pve_api_token='terraform@pve!tfprovider=PASTE-YOUR-UUID-HERE'
export TF_VAR_ssh_public_key="$(cat ~/.ssh/id_ed25519.pub)"
```

> These variables die with the terminal window. If you open a new terminal
> tomorrow, export them again before running Terraform.

### 4.2 Initialise (downloads the provider — once)

```bash
terraform init
```

Expected ending:
```
Terraform has been successfully initialized!
```

### 4.3 Validate and plan

```bash
terraform fmt        # tidies formatting
terraform validate   # checks the code is coherent
```
Expected: `Success! The configuration is valid.`

```bash
terraform plan -out=tfplan
```

Read the last line. It must say:
```
Plan: 10 to add, 0 to change, 0 to destroy.
```
That's 8 VMs + 1 downloaded image + 1 snippet file. (The inventory file is a
`local_file` resource too — you may see **11 to add**; both are fine.)
If it says anything about **destroy**, stop and ask — never apply a
destructive plan you don't understand.

### 4.4 Apply

```bash
terraform apply tfplan
```

This takes 4–8 minutes: image download (~600 MB, once), then 8 VMs created
and booted. Expected ending:

```
Apply complete! Resources: 11 added, 0 changed, 0 destroyed.

Outputs:
control_plane_ips = {
  "k8s-cp-01" = "10.10.10.11"
  ...
```

Watch the Proxmox GUI while it runs — you'll see VMs 9011–9025 appear and
start. This is the "aha" moment of infrastructure-as-code.

### 4.5 Verify every VM answers

```bash
for i in 11 12 13 21 22 23 24 25; do
  ssh -o StrictHostKeyChecking=accept-new ubuntu@10.10.10.$i \
    'echo "$(hostname) OK — RAM: $(free -h | awk "/Mem/{print \$2}")"'
done
```

Expected — eight lines, no password prompts:
```
k8s-cp-01 OK — RAM: 3.8Gi
...
k8s-wk-05 OK — RAM: 7.8Gi
```

> If a VM doesn't answer yet, wait 2 minutes — cloud-init may still be
> installing packages on first boot. Check with:
> `ssh ubuntu@10.10.10.11 cloud-init status` → want `status: done`.

### ✅ CHECKPOINT 4
- [ ] `Apply complete!` with 0 destroyed
- [ ] All 8 VMs green/running in the Proxmox GUI
- [ ] All 8 answer SSH with no password
- [ ] `cat ../ansible/inventory.ini` shows all 8 hosts (Terraform wrote it)

---

## 5. Prepare all nodes with Ansible (~15 min)

**What this phase does, in plain terms:** Kubernetes has prerequisites on
every node — swap off, two kernel modules, three network settings, the
containerd runtime configured a specific way, and the kubeadm/kubelet/kubectl
packages pinned to one version. Doing that by hand on 8 machines = 8 chances
for a typo. Ansible does it identically on all 8.

### 5.1 `ansible/ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
interpreter_python = /usr/bin/python3
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
```

### 5.2 `ansible/group_vars/all.yml`

```yaml
kube_version: "1.31"
kube_pkg_version: "1.31.*"
control_plane_vip: "10.10.10.100"
kube_vip_version: "v0.8.7"
pod_cidr: "10.244.0.0/16"
service_cidr: "10.96.0.0/12"
```

### 5.3 `ansible/site.yml` — the complete playbook

```yaml
---
- name: Prepare every node for Kubernetes
  hosts: k8s
  tasks:

    # ---- OS prerequisites -------------------------------------------
    - name: Disable swap now
      ansible.builtin.command: swapoff -a
      changed_when: false

    - name: Keep swap disabled after reboot
      ansible.builtin.replace:
        path: /etc/fstab
        regexp: '^([^#].*\sswap\s.*)$'
        replace: '# \1'

    - name: Load required kernel modules at boot
      ansible.builtin.copy:
        dest: /etc/modules-load.d/k8s.conf
        content: |
          overlay
          br_netfilter

    - name: Load kernel modules now
      community.general.modprobe:
        name: "{{ item }}"
        state: present
      loop: [overlay, br_netfilter]

    - name: Kernel network settings for Kubernetes
      ansible.builtin.copy:
        dest: /etc/sysctl.d/99-k8s.conf
        content: |
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1
      register: sysctl_file

    - name: Apply sysctl settings
      ansible.builtin.command: sysctl --system
      when: sysctl_file.changed

    - name: Every node knows every node by name
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "{{ hostvars[item].ansible_host }} {{ item }}"
      loop: "{{ groups['k8s'] }}"

    # ---- containerd runtime -----------------------------------------
    - name: Install containerd
      ansible.builtin.apt:
        name: containerd
        state: present
        update_cache: true

    - name: Generate default containerd config
      ansible.builtin.shell: containerd config default > /etc/containerd/config.toml
      args:
        creates: /etc/containerd/config.toml

    - name: Use systemd cgroup driver (CRITICAL — see note below)
      ansible.builtin.replace:
        path: /etc/containerd/config.toml
        regexp: 'SystemdCgroup = false'
        replace: 'SystemdCgroup = true'
      register: cgroup_fix

    - name: Restart containerd if config changed
      ansible.builtin.systemd:
        name: containerd
        state: restarted
        enabled: true
      when: cgroup_fix.changed

    # ---- Kubernetes packages ----------------------------------------
    - name: Keyring directory
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: "0755"

    - name: Download Kubernetes apt key
      ansible.builtin.get_url:
        url: "https://pkgs.k8s.io/core:/stable:/v{{ kube_version }}/deb/Release.key"
        dest: /tmp/k8s.key

    - name: Install key in binary form
      ansible.builtin.shell: gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg /tmp/k8s.key
      args:
        creates: /etc/apt/keyrings/kubernetes.gpg

    - name: Add Kubernetes repository
      ansible.builtin.apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://pkgs.k8s.io/core:/stable:/v{{ kube_version }}/deb/ /"
        filename: kubernetes

    - name: Install kubelet, kubeadm, kubectl
      ansible.builtin.apt:
        name:
          - "kubelet={{ kube_pkg_version }}"
          - "kubeadm={{ kube_pkg_version }}"
          - "kubectl={{ kube_pkg_version }}"
        state: present
        update_cache: true

    - name: Freeze the versions (upgrades must be deliberate)
      ansible.builtin.dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop: [kubelet, kubeadm, kubectl]

- name: Install kube-vip manifest on control-plane nodes
  hosts: control_plane
  tasks:
    - name: Manifests directory
      ansible.builtin.file:
        path: /etc/kubernetes/manifests
        state: directory
        mode: "0755"

    - name: Detect default network interface
      ansible.builtin.shell: ip -4 route show default | awk '{print $5}'
      register: default_iface
      changed_when: false

    - name: Pull kube-vip image
      ansible.builtin.command: >
        ctr image pull ghcr.io/kube-vip/kube-vip:{{ kube_vip_version }}
      changed_when: false

    - name: Generate kube-vip static pod manifest
      ansible.builtin.shell: >
        ctr run --rm --net-host
        ghcr.io/kube-vip/kube-vip:{{ kube_vip_version }} vip
        /kube-vip manifest pod
        --interface {{ default_iface.stdout }}
        --address {{ control_plane_vip }}
        --controlplane --arp --leaderElection
        > /etc/kubernetes/manifests/kube-vip.yaml
      args:
        creates: /etc/kubernetes/manifests/kube-vip.yaml

    - name: First CP only — use super-admin.conf during bootstrap
      ansible.builtin.replace:
        path: /etc/kubernetes/manifests/kube-vip.yaml
        regexp: 'path: /etc/kubernetes/admin\.conf'
        replace: 'path: /etc/kubernetes/super-admin.conf'
      when: inventory_hostname == groups['control_plane'][0]
```

> **The one line that breaks most student clusters:** `SystemdCgroup = true`.
> If it stays `false`, everything installs cleanly, `kubeadm init` succeeds —
> and then pods restart forever with no obvious error. Ubuntu uses systemd to
> manage resources; containerd must agree with it.

### 5.4 Run it

🖥️ **WORKSTATION**
```bash
cd ~/k8s-poc/ansible

# 1) Can Ansible reach all 8?
ansible k8s -m ping
```
Expected: eight blocks of `"ping": "pong"` in green.

```bash
# 2) Run the playbook
ansible-playbook site.yml
```

Takes ~5 minutes. Expected final recap — **failed=0 on every line**:

```
PLAY RECAP ***********************************************************
k8s-cp-01 : ok=21 changed=17 unreachable=0 failed=0 ...
...
k8s-wk-05 : ok=18 changed=15 unreachable=0 failed=0 ...
```

> Red output? Read the FIRST red task name, fix, and simply re-run
> `ansible-playbook site.yml`. The playbook is idempotent — running it twice
> is safe; completed steps are skipped.

### ✅ CHECKPOINT 5
```bash
ansible k8s -m shell -a "kubeadm version -o short && free -h | awk '/Swap/{print \$2}'"
```
- [ ] Every node prints `v1.31.x`
- [ ] Every node prints `0B` swap
- [ ] `PLAY RECAP` had `failed=0` everywhere

---

## 6. Bootstrap the cluster with kubeadm (~20 min)

Now the human-driven part. You will run **one init** on CP-01, then **join**
the other 7 machines. Keep two terminals open: one on CP-01, one free for
the joins.

### 6.1 Create the kubeadm config on CP-01

🎛️ **CP-01** (`ssh ubuntu@10.10.10.11`)
```bash
cat > ~/kubeadm-config.yaml <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.31.4
controlPlaneEndpoint: "10.10.10.100:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
apiServer:
  certSANs:
    - "10.10.10.100"
    - "10.10.10.11"
    - "10.10.10.12"
    - "10.10.10.13"
EOF
```

Why `controlPlaneEndpoint` is the VIP and not CP-01's own IP: every node's
kubeconfig will point at `10.10.10.100`. When CP-01 dies, kube-vip moves that
address to a surviving control plane and nobody notices.

### 6.2 Initialise

```bash
sudo kubeadm init --config ~/kubeadm-config.yaml --upload-certs
```

Takes 1–3 minutes. Success looks like:

```
Your Kubernetes control-plane has initialized successfully!
...
You can now join any number of control-plane nodes by running ...
  kubeadm join 10.10.10.100:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1234... \
    --control-plane --certificate-key 5678...

Then you can join any number of worker nodes by running ...
  kubeadm join 10.10.10.100:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1234...
```

> 🛑 **Copy BOTH join commands into a text file now.** The control-plane one
> (with `--certificate-key`) expires in **2 hours**; the token in 24 hours.
> Lost them? Regenerate any time on CP-01:
> ```bash
> sudo kubeadm token create --print-join-command          # worker command
> sudo kubeadm init phase upload-certs --upload-certs     # fresh cert key
> ```

### 6.3 Give yourself kubectl access on CP-01

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

Expected:
```
NAME        STATUS     ROLES           AGE   VERSION
k8s-cp-01   NotReady   control-plane   1m    v1.31.4
```

> `NotReady` is **correct** at this stage — no pod network exists yet (§7).
> Do not troubleshoot it.

### 6.4 Join CP-02 and CP-03

🖥️ **WORKSTATION** — new terminal:
```bash
ssh ubuntu@10.10.10.12
```
🎛️ **CP-02**
```bash
sudo kubeadm join 10.10.10.100:6443 \
  --token <your-token> \
  --discovery-token-ca-cert-hash sha256:<your-hash> \
  --control-plane --certificate-key <your-cert-key>
exit
```
Repeat identically on **CP-03** (`10.10.10.13`).

Each takes ~1 minute and ends with `This node has joined the cluster ... as a control plane`.

Then, back on 🎛️ **CP-01**, revert the bootstrap tweak from §5.3 (kube-vip can
use the normal admin credentials once the cluster is real):

```bash
sudo sed -i 's#path: /etc/kubernetes/super-admin.conf#path: /etc/kubernetes/admin.conf#' \
  /etc/kubernetes/manifests/kube-vip.yaml
```

### 6.5 Join all 5 workers

🖥️ **WORKSTATION** — the worker join needs no cert key, so loop it:
```bash
JOIN='sudo kubeadm join 10.10.10.100:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-hash>'
for i in 21 22 23 24 25; do
  echo "=== 10.10.10.$i ==="
  ssh ubuntu@10.10.10.$i "$JOIN"
done
```

### 6.6 Copy cluster admin access to your workstation

🖥️ **WORKSTATION**
```bash
mkdir -p ~/.kube
scp ubuntu@10.10.10.11:/home/ubuntu/.kube/config ~/.kube/config-poc
export KUBECONFIG=~/.kube/config-poc
kubectl get nodes
```

### ✅ CHECKPOINT 6
```bash
kubectl get nodes
```
- [ ] **8 nodes listed** — 3 `control-plane`, 5 with no role
- [ ] All show `NotReady` (still correct — CNI comes next)
- [ ] `ping -c2 10.10.10.100` answers (the VIP is alive)

---

## 7. Install the cluster add-ons (~20 min)

All commands: 🖥️ **WORKSTATION** with `export KUBECONFIG=~/.kube/config-poc`.

### 7.1 Cilium — the pod network (this turns nodes Ready)

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm install cilium cilium/cilium --version 1.16.5 \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set k8sServiceHost=10.10.10.100 \
  --set k8sServicePort=6443 \
  --set operator.replicas=2

kubectl -n kube-system rollout status ds/cilium --timeout=5m
kubectl get nodes
```

Expected — the moment of truth:
```
NAME        STATUS   ROLES           AGE   VERSION
k8s-cp-01   Ready    control-plane   25m   v1.31.4
...all 8 Ready...
```

### 7.2 MetalLB — gives apps real IP addresses

Why: in a cloud, `type: LoadBalancer` services get an IP from AWS/GCP. On
Proxmox nobody hands out IPs — MetalLB fills that role from the pool you
reserved (`.200–.240`).

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system --create-namespace
kubectl -n metallb-system rollout status deploy/metallb-controller --timeout=3m

cat > ~/k8s-poc/manifests/metallb-pool.yaml <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: poc-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.10.10.200-10.10.10.240
---
apiVersion: metallb.io/v1beta2
kind: L2Advertisement
metadata:
  name: poc-l2
  namespace: metallb-system
spec:
  ipAddressPools: [poc-pool]
EOF

kubectl apply -f ~/k8s-poc/manifests/metallb-pool.yaml
```

### 7.3 Default storage class

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 7.4 Metrics server (enables `kubectl top`)

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system \
  --set 'args={--kubelet-insecure-tls}'
```

### 7.5 Tidy the role labels

```bash
for i in 1 2 3 4 5; do
  kubectl label node k8s-wk-0$i node-role.kubernetes.io/worker=""
done
```

### ✅ CHECKPOINT 7
```bash
kubectl get nodes          # 8 × Ready
kubectl get pods -A        # everything Running or Completed
kubectl top nodes          # CPU/RAM per node (may need ~1 min to populate)
```

---

## 8. Prove it works — deploy a real application

🖥️ **WORKSTATION**
```bash
kubectl create deployment web --image=nginx --replicas=5
kubectl expose deployment web --port=80 --type=LoadBalancer

kubectl get pods -o wide     # 5 pods spread across the workers
kubectl get svc web
```

Expected:
```
NAME   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
web    LoadBalancer   10.101.x.x     10.10.10.200    80:3xxxx/TCP
```

```bash
curl -s http://10.10.10.200 | grep title
```
Expected: `<title>Welcome to nginx!</title>`

Open `http://10.10.10.200` in a browser — a webpage served by your own
5-replica deployment on your own HA cluster.

---

## 9. The HA drills — the actual point of the lab

### Drill 1 — kill one control plane

🏠 **PROXMOX**
```bash
qm stop 9011      # hard-kill CP-01
```
🖥️ **WORKSTATION** — immediately:
```bash
kubectl get nodes         # still answers! kube-vip moved 10.10.10.100
curl -s http://10.10.10.200 >/dev/null && echo "app still up"
```
Within ~1 minute `kubectl get nodes` shows cp-01 `NotReady`; the app never
blinked. **Lesson:** losing 1 of 3 masters is a non-event.

```bash
# restore
🏠 qm start 9011
```

### Drill 2 — kill TWO control planes (break quorum)

🏠 **PROXMOX**
```bash
qm stop 9011 && qm stop 9012
```
🖥️ **WORKSTATION**
```bash
kubectl get nodes     # hangs/errors — etcd lost quorum (1 of 3 < majority)
curl -s http://10.10.10.200 >/dev/null && echo "app STILL up"
```
**Lesson:** the control plane is down but already-running apps keep serving.
Kubernetes' brain and its muscles fail independently.

```bash
🏠 qm start 9011 && qm start 9012
# wait ~2 min, then kubectl works again
```

### Drill 3 — drain a worker for "maintenance"

🖥️ **WORKSTATION**
```bash
kubectl drain k8s-wk-03 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide      # wk-03's pods rescheduled elsewhere
kubectl uncordon k8s-wk-03
```
**Lesson:** this is how you patch/reboot nodes with zero downtime.

---

## 10. Daily start/stop and teardown

Pause the lab (state survives):

🏠 **PROXMOX**
```bash
for id in 9011 9012 9013 9021 9022 9023 9024 9025; do qm shutdown $id; done
# resume later:
for id in 9011 9012 9013 9021 9022 9023 9024 9025; do qm start $id; done
```

Destroy everything:

🖥️ **WORKSTATION**
```bash
cd ~/k8s-poc/terraform
terraform destroy      # type 'yes' — deletes all 8 VMs permanently
```

---

## 11. Common mistakes in this lab (read before you need it)

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Nodes forever `NotReady` | CNI not installed — or you're troubleshooting §6 too early | Finish §7.1 first |
| 2 | kubelet restarts in a loop; `journalctl -u kubelet` shows cgroup errors | `SystemdCgroup` still `false` | Re-run the Ansible playbook; it fixes and restarts containerd |
| 3 | `kubeadm init` fails: "port 6443 in use / swap enabled" | Swap came back, or a previous half-init | `sudo kubeadm reset -f`, `sudo swapoff -a`, retry |
| 4 | CP-02/03 join fails: "certificate key expired" | >2 h since init | On CP-01: `sudo kubeadm init phase upload-certs --upload-certs` |
| 5 | Join hangs on "Running pre-flight checks" | VIP unreachable — kube-vip pod not running on CP-01 | On CP-01: `sudo crictl ps -a \| grep vip`, check §5.3 last task ran |
| 6 | Terraform: `content type 'snippets' not allowed` | §2.2 skipped | `pvesm set local --content backup,iso,vztmpl,snippets,import` |
| 7 | Terraform hangs at "waiting for agent" | qemu-guest-agent not up yet | Wait; check `cloud-init status` in VM console via GUI |
| 8 | `curl http://10.10.10.200` times out | MetalLB range overlaps DHCP, or another device owns the IP | Pick a truly free range; `arping 10.10.10.200` to check |
| 9 | Everything randomly slow, etcd leader elections in logs | VM disks on HDD | Move to SSD storage. No workaround exists. |
| 10 | Terraform in a NEW terminal: prompts for `pve_api_token` | Env vars are per-terminal | Re-run the two `export` lines from §4.1 |

Diagnostic one-liners to teach:

```bash
journalctl -u kubelet -f            # why won't the node come up
sudo crictl ps -a                    # containers below Kubernetes level
kubectl describe node <name>         # why is a node NotReady
kubectl describe pod <name>          # why is a pod Pending/CrashLoop
kubectl -n kube-system logs -l k8s-app=cilium --tail=50
sudo kubeadm certs check-expiration
```

---

## 12. Known limitations & where to go next

Limitations of this PoC (say them out loud in class):
1. One physical host → the "HA" cannot survive host failure or a power cut.
2. No etcd backups. One command fixes that: `etcdctl snapshot save` on cron.
3. TLS verification disabled toward Proxmox (`insecure = true`).
4. Terraform state is a local file — fine solo, unusable in a team.

Graduation path:
1. 3 Proxmox hosts + anti-affinity → real HA.
2. Remote Terraform state (S3/MinIO + locking).
3. Cluster API (CAPI + Proxmox provider) instead of kubeadm+Ansible.
4. Velero backups, kube-prometheus-stack monitoring, default-deny NetworkPolicies.
