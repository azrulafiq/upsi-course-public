# Command Cheat Sheet — Docker, Kubernetes, Terraform & Ansible

Quick reference for the commands used across Day 1 and Day 2. Grouped by tool, roughly in the order you'll reach for them. `userNN` / `groupNN` = your assigned ID (on the board).

---

## Docker — containers

| Command | What it does |
|---|---|
| `docker version --format 'Docker {{.Server.Version}}'` | Confirm the Docker daemon is up and check its version |
| `docker run hello-world` | Sanity-check container: pulls, runs, exits, prints an explanation |
| `docker run -d -p 8080:80 --name web nginx` | Run detached (`-d`), map host port 8080 → container port 80, name it `web` |
| `docker ps` | List **running** containers |
| `docker ps -a` | List **all** containers, including stopped/exited ones (find the "corpses") |
| `docker logs web` | Show a container's stdout/stderr — first stop when something crashed |
| `docker logs -f web` | Follow the logs live (like `tail -f`) |
| `docker exec -it web bash` | Open an interactive shell inside a running container (use `sh` if `bash` isn't installed) |
| `docker inspect web` | Full JSON detail on a container — config, mounts, network, exit code |
| `docker stop web` | Gracefully stop a running container |
| `docker rm web` | Remove a stopped container |
| `docker rm -f web` | Force stop + remove in one shot |
| `docker restart web` | Restart a container (picks up bind-mounted file changes, not code baked into the image) |

## Docker — building images

| Command | What it does |
|---|---|
| `docker build -t hello-web .` | Build an image from the `Dockerfile` in the current dir, tag it `hello-web` |
| `docker images` | List images stored locally |
| `docker tag hello-web hello-web:v1` | Give an image a real version tag (avoid `:latest` — Kubernetes always tries to re-pull it) |
| `docker rmi hello-web:v1` | Delete a local image |

**4-step diagnose ritual** (use in this order before you "fix" anything):
`docker ps -a` → `docker logs <name>` → `docker exec -it <name> sh` → `docker inspect <name>`

## Docker — volumes (data that survives)

| Command | What it does |
|---|---|
| `docker volume create dbdata` | Create a named volume explicitly (or let `-v name:/path` create it on first use) |
| `docker volume ls` | List all volumes |
| `docker volume inspect dbdata` | Show where a volume lives and what's using it |
| `docker volume rm dbdata` | Delete a volume (only if unused) |
| `docker run -d -v notes:/data --name writer alpine ...` | Mount **named volume** `notes` at `/data` — data outlives the container |
| `docker run -d -v ~/hello-web:/app --name dev hello-web` | **Bind mount** your local folder into the container — edit locally, see it live without rebuilding |

Named volume = Docker manages the storage location (use for real data, e.g. databases).
Bind mount = you choose the host path (use for live dev, editing code without rebuilds).

## Docker — networking

| Command | What it does |
|---|---|
| `docker network create appnet` | Create a user-defined bridge network |
| `docker network ls` | List networks |
| `docker network rm appnet` | Delete a network |
| `docker run -d --name web2 --network appnet nginx` | Attach a container to a network at start time |
| `docker run --rm --network appnet alpine sh -c "wget -qO- http://web2"` | Containers on the same network reach each other **by name**, no IP, no published port needed |

### Worked example — WordPress + MariaDB on one network

Two containers, one shared network, talking to each other by container name (`db`):

```bash
docker network create wpnet

docker run -d --name db --network wpnet -v dbdata:/var/lib/mysql \
  -e MARIADB_ROOT_PASSWORD=rootpass -e MARIADB_DATABASE=wordpress \
  -e MARIADB_USER=wp -e MARIADB_PASSWORD=wppass mariadb:latest

docker run -d --name wp --network wpnet -p 8088:80 \
  -e WORDPRESS_DB_HOST=db -e WORDPRESS_DB_NAME=wordpress \
  -e WORDPRESS_DB_USER=wp -e WORDPRESS_DB_PASSWORD=wppass wordpress:latest
```

Then browse `http://localhost:8088`. `db` resolves purely because both containers share `wpnet` — no `-p` was published for MariaDB, and none needed to be. `dbdata` is a named volume, so the database survives even if the `db` container is removed and recreated.

---

## Kubernetes (kubectl) — the Docker-to-K8s "twin ritual"

| Docker | Kubernetes equivalent | What it does |
|---|---|---|
| `docker ps` | `kubectl get pods` | List running things |
| `docker inspect` | `kubectl describe pod <name>` | Full detail — **read the Events section** for the "why" |
| `docker logs` | `kubectl logs <name>` | Container output |
| `docker exec -it` | `kubectl exec -it <name> -- sh` | Shell into a running pod |

## Kubernetes — cluster & nodes

| Command | What it does |
|---|---|
| `minikube start --driver=docker` | Spin up a local single-node cluster inside Docker |
| `minikube delete` | Nuke the local cluster (fixes almost anything, costs 1–2 min to rebuild) |
| `minikube image load hello-web:v1` | Copy a local Docker image straight into the cluster (needed since minikube can't see your local Docker image cache otherwise) |
| `kubectl get nodes` | List cluster nodes and their status |
| `kubectl version --client` | Check the kubectl client version |

## Kubernetes — running things

| Command | What it does |
|---|---|
| `kubectl create deployment hello-web --image=hello-web:v1` | Quick imperative deployment (training wheels — real work uses YAML manifests) |
| `kubectl get pods` | List pods and their status |
| `kubectl get pods -w` | Watch pods update live (great during a rollout) |
| `kubectl port-forward deployment/hello-web 8080:8080` | Tunnel a local port into a pod/deployment for testing |
| `kubectl port-forward service/hello-web 8080:80` | Same, but via a Service |
| `kubectl scale deployment hello-web --replicas=4` | Change the number of running replicas |
| `kubectl delete pod <name>` | Kill a pod on purpose — the Deployment replaces it automatically (self-healing) |
| `kubectl delete deployment hello-web` | Remove a deployment entirely |
| `kubectl top pods` | Live CPU/memory usage per pod vs. requested resources |

## Kubernetes — manifests (declarative YAML)

| Command | What it does |
|---|---|
| `kubectl apply -f .` | Create/update every resource described in the YAML files in this folder |
| `kubectl apply -f deployment.yaml` | Apply a single manifest file |
| `kubectl get deployment` / `kubectl get svc` / `kubectl get ingress` | List each resource type |
| `kubectl describe pod <name>` | Deep detail + Events — the first thing to check when something's stuck |

Minimum trio for a web app: **Deployment** (runs your pods) → **Service** (stable internal address) → **Ingress** (public hostname, cloud only).

## Kubernetes — rollouts (zero-downtime deploys)

| Command | What it does |
|---|---|
| `kubectl set image deployment/hello-web hello-web=hello-web:v2` | Update the image on a running Deployment — triggers a rolling update |
| `kubectl rollout status deployment/hello-web` | Watch a rollout until it completes |
| `kubectl rollout history deployment/hello-web` | List past revisions of a Deployment |
| `kubectl rollout undo deployment/hello-web` | Roll back to the previous revision |
| `kubectl rollout undo deployment/hello-web --to-revision=1` | Roll back to a specific revision |

## Kubernetes — namespaces & context (shared clusters, Day 2)

| Command | What it does |
|---|---|
| `kubectl create namespace userNN` | Create your own namespace on a shared cluster — "your room in the building" |
| `kubectl config set-context --current --namespace=userNN` | Make that namespace the default target for future kubectl commands |
| `kubectl config get-contexts` | List available contexts (clusters/users/namespaces) |
| `kubectl get pods -n userNN` | Target a namespace explicitly without switching context |

---

## Google Cloud (gcloud)

| Command | What it does |
|---|---|
| `gcloud auth activate-service-account --key-file=~/userNN-key.json` | Log in as your service account using the handed-out key |
| `gcloud auth list` | Show which account is currently active (marked with `*`) |
| `gcloud config set project acserver-497017` | Set the default project for future commands |
| `gcloud config set compute/zone asia-southeast1-a` | Set the default compute zone |
| `gcloud compute instances list` | List VMs in the project |
| `gcloud container clusters list` | List GKE clusters |
| `gcloud container clusters get-credentials training-cluster --zone asia-southeast1-a` | Pull cluster credentials into your local kubeconfig so `kubectl` can talk to GKE |
| `gcloud auth configure-docker asia-southeast1-docker.pkg.dev` | One-time: let Docker push/pull to that Artifact Registry using your gcloud login |
| `gcloud artifacts docker images list <path>` | List images pushed to an Artifact Registry repo |
| `gcloud auth print-access-token` | Print a short-lived OAuth token (used by Ansible/Docker logins; expires ~1 h) |

### Docker ↔ Artifact Registry

```bash
docker tag hello-web:v1 asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v1
docker push asia-southeast1-docker.pkg.dev/acserver-497017/labs/userNN/hello-web:v1
```
Registry path shape: `REGION-docker.pkg.dev / PROJECT / REPO / IMAGE : TAG`.

### IAP tunnel (reach a VM with no public IP)

| Command | What it does |
|---|---|
| `gcloud compute ssh <vm> --tunnel-through-iap --command "..."` | SSH to a private VM through Identity-Aware Proxy |
| `gcloud compute start-iap-tunnel <vm> <remote-port> --local-host-port=localhost:<local-port> --zone=<zone>` | Open a local port that tunnels to a port on the private VM (leave running in its own terminal) |

---

## Terraform

| Command | What it does |
|---|---|
| `terraform init` | Download providers, set up the working directory (run once per project) |
| `terraform plan` | Show what would change — **always read this before applying** |
| `terraform apply` | Apply the plan (prompts `yes`) |
| `terraform output vm_ip` | Print a declared output value |
| `terraform destroy` | Tear down everything Terraform created — ⚠️ never run this against a shared/in-use resource without checking first |

Healthy workflow: `init` once → `plan` → read it → `apply` → `plan` again (should say **"No changes."** — reality now matches code).

---

## Ansible

| Command | What it does |
|---|---|
| `ansible web -m ping` | Confirm Ansible can reach every host in the `web` group (green "pong") |
| `ansible-playbook site.yml` | Run a playbook against the inventory |
| `ansible-playbook site.yml --extra-vars "gcp_token=$(gcloud auth print-access-token)"` | Run a playbook, injecting a fresh token as a variable |

Idempotency check: run the same playbook twice. First run: `changed=N`. Second run: `changed=0` (state was already correct — that's the whole point of Ansible).

---

## Quick troubleshooting

| Symptom | First move |
|---|---|
| Docker errors in WSL | Is Docker Desktop running, with WSL integration on? |
| Build fails | Read the error — it names the line and usually the file |
| Container exits instantly | `docker ps -a` for the exit code, then `docker logs` |
| minikube start fails | `minikube delete && minikube start --driver=docker` |
| Pod `ErrImagePull` / `ImagePullBackOff` on minikube | Did you `minikube image load` that exact tag? `:latest` is a trap |
| `ImagePullBackOff` on GKE | Wrong `userNN` in the image path, or the image was never pushed |
| port-forward "connection refused" | Wrong port pair, or the pods aren't Ready yet (`kubectl get pods`) |
| gcloud "permission denied" | `gcloud auth list` — is the right service account active? |
| Push to registry denied | Wrong `userNN` in the path, or re-run `gcloud auth configure-docker` |
| Pod stuck / weird | `kubectl describe pod <name>` → read Events |
| Hostname won't load (Ingress) | `kubectl get ingress` — is the host listed? Check for a `userNN` typo |
| Terraform plans a surprise destroy | **Stop.** Ask the instructor before applying |
| Ansible `UNREACHABLE` | Re-check username + the `ProxyCommand` line in `inventory.ini` |
| Registry token error in playbook | Tokens last ~1 h; re-run the command — it mints a fresh one |
