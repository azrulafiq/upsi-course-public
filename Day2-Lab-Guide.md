# Day 2 Lab Guide — The Big Guns: GKE, Terraform & Ansible
## Managing Docker & Kubernetes Clusters with Ansible and Terraform (Day 2 of 2)

**Straightcut IT Solution — Instructor-led Technical Training**

Same WSL terminal as yesterday; today it talks to Google Cloud. Conventions: `userNN` = your ID, `groupNN` = your assigned Lab 2/3 group (on the board). Project `acserver-497017`, zone `asia-southeast1-a`.

| What | Address |
|---|---|
| Your app (after Lab 1) | `http://userNN.gkecluster.eunoai.my` |
| Shared cluster | `training-cluster` — one for the class, a namespace each |

---

# Lab 0 — Log in and look around (20 min)

If you ran last night's three commands, steps 0.1–0.2 are a victory lap.

### 0.1 Authenticate as your service account

```bash
gcloud auth activate-service-account --key-file=~/userNN-key.json
gcloud config set project acserver-497017
gcloud config set compute/zone asia-southeast1-a
gcloud auth list                  # your SA has the * (active)
```

### 0.2 Meet today's cast

```bash
gcloud compute instances list     # the class VMs (your group's target arrives in Lab 2)
gcloud container clusters list    # training-cluster — built by Terraform, as you'll see
```

### 0.3 One-time: authorize Docker for the registry

```bash
gcloud auth configure-docker asia-southeast1-docker.pkg.dev
```

✅ **Done when:** your SA is active and configure-docker reports the registry added.

---

# Lab 1 — Ship it: registry → GKE → real hostname (60 min)

### 1.1 Push yesterday's image to Artifact Registry

The registry address decodes as `REGION-docker.pkg.dev / PROJECT / REPO / IMAGE : TAG`:

```bash
docker tag hello-web:v1 \
  asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v1
docker push \
  asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v1

gcloud artifacts docker images list \
  asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN
```

### 1.2 Connect to the shared cluster & claim your namespace

```bash
gcloud container clusters get-credentials training-cluster --zone asia-southeast1-a
kubectl get nodes                          # Google's machines this time
kubectl create namespace userNN
kubectl config set-context --current --namespace=userNN
```

One cluster for the whole class — your namespace is your room in it. **House rule #1: stay in yours.**

### 1.3 The manifests — yesterday's, plus two additions

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
        image: asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v1
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

### 1.4 Deploy and go public

```bash
kubectl apply -f .
kubectl get pods                # Running after the image pull (~30s)
kubectl get ingress             # your host + the shared class IP
```

Open **`http://userNN.gkecluster.eunoai.my`** — in your laptop's browser, your phone, anywhere. That's a real URL on the public internet, and it's yours for the rest of the course.

### 1.5 Same tricks, bigger stage

```bash
kubectl scale deployment hello-web --replicas=4
kubectl delete pod <one>        # self-heal, cloud edition
kubectl top pods                # your actual usage vs your 50m request
```

**Stretch (5 min, do it if you're ahead):** push your `hello-web:v2` from yesterday and roll it out here — `kubectl set image deployment/hello-web hello-web=...:v2` — the exact Lab 4 motion, now behind a public hostname.

✅ **Done when:** `userNN.gkecluster.eunoai.my` serves your app from 4 self-healing pods.

**If stuck:** `ImagePullBackOff` → wrong `userNN` in the image path. Hostname won't load → typo in the ingress host, or check `kubectl get ingress`. Anything weird → `kubectl describe pod`, read Events.

---

# Lab 2 — Provision infrastructure from code (50 min · assigned groups)

**Where:** the group driver's WSL terminal (swap drivers for Lab 3!).

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

The target has no public IP, so we travel through **IAP** (Identity-Aware Proxy) — Google's front door to private machines. Your service account already has the permission.

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

**`site.yml`** (replace `userNN` — use the driver's image):

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
        image: asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v1
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
| gcloud "permission denied" | `gcloud auth list` — right SA active? |
| Push denied | Wrong `userNN` in the path, or re-run configure-docker |
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
