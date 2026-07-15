# Day 2 Lab Guide — The Big Guns: GKE, Terraform & Ansible
## Managing Docker & Kubernetes Clusters with Ansible and Terraform (Day 2 of 2)

**Straightcut IT Solution — Instructor-led Technical Training**

One shared VM — the **jumphost** — is today's entire toolchain: gcloud, kubectl, docker, terraform, and ansible are already installed. SSH in with the same key as yesterday, username `upsiNN` (that login is separate from your GCP identity — more in Lab 0.2). Collect `userNN-key.json` from the instructor (USB/DM) if you don't already have it; you'll activate it once you're in. Conventions: `upsiNN` = your SSH login, `userNN` = your GCP identity (same number, different name — say it out loud once so it sticks), `groupNN` = your assigned Lab 2/3 group (on the board). Project `acserver-497017`, zone `asia-southeast1-a`.

| What | Address |
|---|---|
| Jumphost | `ssh -i <private key> upsiNN@<jumphost-ip>` (IP on the board) |
| Your app (after Lab 1) | `http://userNN.gkecluster.eunoai.my` |
| Shared cluster | `training-cluster` — one for the class, a namespace each |

---

# Lab 0 — Get on the jumphost, sign in as yourself (20 min)

### 0.1 SSH in

```bash
ssh -i <path-to-your-private-key> upsiNN@<jumphost-ip>
```

Same private key as yesterday, new host — yesterday's VM is gone, the jumphost is everyone's machine today. First login creates your `upsiNN` account automatically, passwordless sudo included, same as Day 1.

### 0.2 Bring your GCP identity onto the box

The jumphost is shared, but nothing about your session is — separate `upsiNN` accounts mean separate home directories, separate `gcloud`/`docker` configs. From **your own laptop's terminal**:

```bash
scp -i <path-to-your-private-key> userNN-key.json upsiNN@<jumphost-ip>:~
```

Back on the jumphost:

```bash
chmod 600 ~/userNN-key.json          # lock it down — you're not the only account on this box
gcloud auth activate-service-account --key-file=~/userNN-key.json
gcloud config set project acserver-497017
gcloud config set compute/zone asia-southeast1-a
gcloud auth configure-docker asia-southeast1-docker.pkg.dev --quiet
gcloud auth list                     # userNN-sa marked active — your session, nobody else's
```

### 0.3 Confirm the toolchain

```bash
gcloud --version | head -1
kubectl version --client
docker --version
terraform version | head -1
ansible --version | head -1
```

### 0.4 Meet today's cast

```bash
gcloud compute instances list     # the class VMs (your group's target arrives in Lab 2)
gcloud container clusters list    # training-cluster — built by Terraform, as you'll see
```

✅ **Done when:** all five tools print a version, `userNN-sa` shows active, and `training-cluster` is listed.

---

# Lab 1 — Ship it: your Day 1 image → GKE → real hostname (60 min)

### 1.1 Get yesterday's image onto the jumphost

You `scp`'d `hello-web.tar` down to your laptop before Day 1's VM was destroyed. Now the reverse hop, from **your own laptop's terminal**:

```bash
scp -i <path-to-your-private-key> hello-web.tar upsiNN@<jumphost-ip>:~
```

Back on the jumphost:

```bash
docker load -i hello-web.tar
docker images | grep hello-web     # hello-web:upsiNN-v2, restored
```

### 1.2 Push it to Artifact Registry

The registry address decodes as `REGION-docker.pkg.dev / PROJECT / REPO / IMAGE : TAG`. `docker load` only restores the image *locally* — GKE can't reach into your Docker daemon the way minikube could yesterday; it only ever pulls from a registry:

```bash
docker tag hello-web:upsiNN-v2 \
  asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v2
docker push \
  asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v2

gcloud artifacts docker images list \
  asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN
```

(The local image tag still says `upsiNN` from yesterday — that's just a local label, fine to leave. The registry *path*, `labs/userNN/...`, is what matters here: it's tied to your `userNN-sa` push permission from Lab 0.2.)

### 1.3 Connect to the shared cluster & claim your namespace

```bash
gcloud container clusters get-credentials training-cluster --zone asia-southeast1-a
kubectl get nodes                          # Google's machines this time
kubectl create namespace userNN
kubectl config set-context --current --namespace=userNN
```

One cluster for the whole class — your namespace is your room in it. **House rule #1: stay in yours.**

### 1.4 The manifests — yesterday's, plus two additions

```bash
mkdir ~/gke && cd ~/gke
```

**`deployment.yaml`** — same shape as yesterday **plus the `resources` block** (shared-cluster etiquette: requests reserve your slice, limits cap your burst). Replace `userNN` in the image path:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
      - name: hello-web
        image: asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v2
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "250m"
            memory: "128Mi"
```

**`service.yaml`** — identical to yesterday's:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-web
spec:
  type: ClusterIP
  selector:
    app: hello-web
  ports:
  - port: 80
    targetPort: 8080
```

**`ingress.yaml`** — the second addition: your public hostname on the class's shared front door (replace `userNN`!):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-web
spec:
  ingressClassName: nginx
  rules:
  - host: userNN.gkecluster.eunoai.my
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-web
            port:
              number: 80
```

### 1.5 Deploy and go public

```bash
kubectl apply -f .
kubectl get pods                # Running after the image pull (~30s)
kubectl get ingress             # your host + the shared class IP
```

Open **`http://userNN.gkecluster.eunoai.my`** — in your laptop's browser, your phone, anywhere. That's a real URL on the public internet, and it's yours for the rest of the course.

### 1.6 Same tricks, bigger stage

```bash
kubectl scale deployment hello-web --replicas=4
kubectl delete pod <one>        # self-heal, cloud edition
kubectl top pods                # your actual usage vs your 50m request
```

✅ **Done when:** `userNN.gkecluster.eunoai.my` serves your app from 4 self-healing pods.

**If stuck:** `ImagePullBackOff` → wrong `userNN` in the image path, or you skipped 1.2's push. Hostname won't load → typo in the ingress host, or check `kubectl get ingress`. Anything weird → `kubectl describe pod`, read Events.

---

# Lab 2 — Provision infrastructure from code (50 min · assigned groups)

**Where:** the jumphost — one group member's SSH session drives, everyone else watches over their shoulder (swap who's driving for Lab 3!). Whoever drives runs this in their own `upsiNN` home directory, authenticated as their own `userNN-sa` from Lab 0.2 — that's whose `.tf` files and state this ends up in.

### 2.1 The files

```bash
mkdir ~/lab-tf && cd ~/lab-tf
```

**`main.tf`** — note what's *missing*: no `access_config` block means **no public IP**. Internal-only is the security default in real environments; reaching it is Lab 3's lesson.

```hcl
terraform {
  required_providers {
    google = { source = "hashicorp/google" }
  }
}

provider "google" {
  project = var.project_id
  region  = "asia-southeast1"
  zone    = "asia-southeast1-a"
}

resource "google_compute_instance" "target" {
  name         = "${var.group}-ansible-target"
  machine_type = "e2-small"
  tags         = ["ansible-target"]   # matches the class IAP firewall rule

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    network = "default"
    # no access_config {}  ->  internal IP only
  }
}
```

**`variables.tf`**:

```hcl
variable "project_id" { type = string }
variable "group"      { type = string }
```

**`outputs.tf`**:

```hcl
output "vm_ip" {
  description = "Internal IP of the new VM"
  value       = google_compute_instance.target.network_interface[0].network_ip
}
```

**`terraform.tfvars`**:

```hcl
project_id = "acserver-497017"
group      = "groupNN"
```

### 2.2 The workflow

```bash
terraform init
terraform plan      # READ IT. Expect: 1 to add, 0 to change, 0 to destroy.
terraform apply     # yes
```

### 2.3 Verify from both sides

```bash
gcloud compute instances list --filter="name=groupNN-ansible-target"
# EXTERNAL_IP column is empty — exactly as declared
terraform output vm_ip
terraform plan      # "No changes." — reality matches code
```

✅ **Done when:** `groupNN-ansible-target` exists with an internal IP only, and a second plan shows no changes.

⚠️ **Do NOT run `terraform destroy`** — Lab 3 configures this VM.

---

# Lab 3 — Capstone: blank VM → running app, hands off (50 min · same groups, new driver)

The target has no public IP, so we travel through **IAP** (Identity-Aware Proxy) — Google's front door to private machines. Your `userNN-sa` already has the permission, granted in `iam.tf` alongside Lab 2's.

### 3.1 Seed SSH once, through the tunnel

```bash
gcloud compute ssh groupNN-ansible-target --tunnel-through-iap --command "echo ssh works"
whoami        # note your username — needed in the inventory
```

### 3.2 The Ansible folder

```bash
mkdir ~/lab-ansible && cd ~/lab-ansible
```

**`inventory.ini`** — the host is the instance **name** (IAP resolves it); one line routes SSH through the tunnel:

```ini
[web]
groupNN-ansible-target ansible_user=YOUR_USERNAME ansible_ssh_private_key_file=~/.ssh/google_compute_engine

[web:vars]
ansible_ssh_common_args=-o ProxyCommand="gcloud compute start-iap-tunnel %h 22 --listen-on-stdin --zone=asia-southeast1-a --verbosity=warning"
```

**`ansible.cfg`**:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
```

### 3.3 Ping, then the playbook

```bash
ansible web -m ping     # green "pong" = Ansible can drive the machine
```

**`site.yml`** (replace `userNN` — use the driver's image from Lab 1):

```yaml
- name: Turn a blank VM into a Docker host running my app
  hosts: web
  become: true

  tasks:
    - name: Install Docker and the Python Docker SDK
      ansible.builtin.apt:
        name: [docker.io, python3-docker]
        state: present
        update_cache: true

    - name: Make sure Docker is running
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Log in to Artifact Registry
      community.docker.docker_login:
        registry_url: asia-southeast1-docker.pkg.dev
        username: oauth2accesstoken
        password: "{{ gcp_token }}"

    - name: Run the hello-web container
      community.docker.docker_container:
        name: hello-web
        image: asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v2
        ports: ["80:8080"]
        restart_policy: always
        state: started
```

Read it first; predict each task's report. Then:

```bash
ansible-playbook site.yml --extra-vars "gcp_token=$(gcloud auth print-access-token)"
```

Expect `changed=4 failed=0`. (The internal-only VM reaches apt and the registry through the class Cloud NAT — controlled outbound for private machines, the real-world pattern.)

### 3.4 Browse a machine the internet can't see

In a **second** terminal:

```bash
gcloud compute start-iap-tunnel groupNN-ansible-target 80 \
  --local-host-port=localhost:8081 --zone=asia-southeast1-a
```

Leave it running; browse **`http://localhost:8081`** — your app, on a VM with no public IP, that didn't exist an hour ago, that nobody logged into to configure. Ctrl-C when done.

### 3.5 Prove idempotency

```bash
ansible-playbook site.yml --extra-vars "gcp_token=$(gcloud auth print-access-token)"
```

`changed=0` (login may re-assert as 1). The playbook describes state; the state was already true.

✅ **Done when:** the app answers through your tunnel and the second run changes (almost) nothing.

**Count the tools:** Docker packaged it (Day 1) · Kubernetes ran it at scale (Lab 1) · Terraform built the machine (Lab 2) · Ansible wired it together (now). Two days, one system.

---

# Module 4 — Mission board (15:30)

Missions live in the **Challenge Labs handout** — goals and proof conditions only, no commands, AI encouraged, leaderboard on the whiteboard. Ask the instructor for your first mission. Everything stays up after 4:30.

# Quick troubleshooting reference

| Symptom | First move |
|---|---|
| gcloud "permission denied" | `gcloud auth list` — is your own `userNN-sa` active? (Lab 0.2) |
| Push denied | Wrong `userNN` in the path, or Lab 0.2's key never got activated |
| Pod stuck | `kubectl describe pod` → Events |
| Hostname won't load | `kubectl get ingress` — host listed? `userNN` typo? |
| Terraform plans a surprise destroy | **Stop.** Ask the instructor |
| Ansible UNREACHABLE | Re-run 3.1; check username + ProxyCommand line |
| Registry token error in playbook | Tokens last ~1 h; re-run the command (it mints a fresh one) |

# Further reading

- cloud.google.com/kubernetes-engine/docs/quickstarts · cloud.google.com/iap/docs/using-tcp-forwarding
- developer.hashicorp.com/terraform/tutorials/gcp-get-started
- docs.ansible.com — "Getting started with Ansible"
- Next course: CI/CD — Jenkins or GitHub Actions automate everything you did by hand these two days.

*Lab environment provisioned and borne by Straightcut IT Solution.*
