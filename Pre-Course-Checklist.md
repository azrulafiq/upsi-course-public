# Pre-Course Setup Checklist — send to participants ≥1 week before Day 1
## Managing Docker & Kubernetes Clusters with Ansible and Terraform (2-day course)

You will run all labs **from your own Windows laptop** using WSL (Windows Subsystem for Linux). Please complete ALL steps below **before** the course — Day 1 starts with a 2-minute verification, not an hour of installs. If anything fails and you can't resolve it, email the instructor *before* the course; a locked-down corporate laptop is solvable, but not at 8:30 AM.

**Requirements:** Windows 10 (21H2+) or Windows 11, admin rights, ~10 GB free disk, virtualization enabled in BIOS (usually already on).

---

## Step 1 — WSL 2 + Ubuntu

Open **PowerShell as Administrator**:

```powershell
wsl --install -d Ubuntu
```

Reboot when prompted. Ubuntu opens and asks you to create a Linux username + password — remember them. If you installed WSL long ago: `wsl --update` and confirm `wsl -l -v` shows VERSION 2.

## Step 2 — Docker Desktop

Or access here for more options: https://docs.docker.com/engine/install/ubuntu/

1. Download from docker.com/products/docker-desktop and install (choose the WSL 2 backend).
2. Start Docker Desktop → **Settings → Resources → WSL Integration** → enable for **Ubuntu** → Apply & Restart.
3. Verify — in an **Ubuntu (WSL) terminal**:

```bash
docker run --rm hello-world
```

## Step 3 — Tools inside WSL (Ubuntu terminal, copy-paste the whole block)

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


sudo apt-get update && sudo apt-get install -y git curl gnupg apt-transport-https ca-certificates lsb-release software-properties-common

# --- Google Cloud CLI + kubectl + GKE auth plugin ---
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list
sudo apt-get update && sudo apt-get install -y google-cloud-cli kubectl google-cloud-cli-gke-gcloud-auth-plugin

# --- Terraform ---
wget -qO- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform

# --- Ansible (+ the Docker collection used on Day 2) ---
sudo apt-get install -y ansible
ansible-galaxy collection install community.docker

# --- minikube (your own Kubernetes cluster for Day 1) ---
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

## Step 4 — Verify (all six must print a version)

```bash
docker version --format 'Docker {{.Server.Version}}'
minikube version | head -1
gcloud --version | head -1
kubectl version --client
terraform version | head -1
ansible --version | head -1
```

## Step 5 — Strongly recommended: pre-warm minikube (saves 10 min in class)

The first `minikube start` downloads ~1 GB. Do it at home on good Wi-Fi:

```bash
minikube start --driver=docker
kubectl get nodes        # NAME: minikube  STATUS: Ready
minikube stop            # leave it stopped; Day 1 starts it again
```

Screenshot the output — that's your "ready" proof.

## What you do NOT need to install

- **Kubernetes itself** — minikube (above) gives you a personal cluster on Day 1; Day 2 uses a Google-managed cluster in the classroom cloud.
- **A Google account or GCP billing** — you'll receive a course service-account key at the end of Day 1. Day 1 itself needs no cloud at all.

## Common issues

| Problem | Fix |
|---|---|
| `wsl --install` errors about virtualization | Enable VT-x/AMD-V ("SVM") in BIOS/UEFI |
| Docker commands in WSL: "cannot connect to daemon" | Docker Desktop not running, or WSL Integration not enabled for Ubuntu |
| Corporate policy blocks WSL or Docker Desktop | Contact the instructor — Google Cloud Shell (browser-only) is the fallback |
| Behind a corporate proxy, apt/curl fail | Try the steps from home / hotspot, or ask IT for proxy settings |

See you at 8:30. — Straightcut IT Solution
