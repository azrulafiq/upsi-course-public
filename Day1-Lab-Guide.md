# Day 1 Lab Guide — Docker & Kubernetes on Your Laptop
## Managing Docker & Kubernetes Clusters with Ansible and Terraform (Day 1 of 2)

**Straightcut IT Solution — Instructor-led Technical Training**

Everything today runs on **your own cloud VM** — Docker, kubectl, and minikube are already installed; you connect over SSH with a key the instructor gives you. Nothing you can break that `minikube delete` (or a full VM rebuild) won't fix. Replace `upsiNN` with your assigned ID wherever you see it.

---

# Lab 0 — Toolkit check (15 min)

```bash
docker version --format 'Docker {{.Server.Version}}'   # Docker daemon must be running
minikube version | head -1
kubectl version --client
```

Then the traditional first container:

```bash
docker run hello-world
```

**Read the output** — it explains what just happened in four steps. Bonus: that container already exited; find its corpse with `docker ps -a`, then remove it.

✅ **Done when:** three version numbers + you can explain hello-world's four steps in your own words.

VM misbehaving? Flag it **now** and pair with a neighbour while we fix it.

---

# Lab 1 — Build & run your first app (40 min)

### 1.1 A container you didn't build

```bash
docker run -d -p 8080:80 --name web nginx
docker ps
curl localhost:8080
docker logs web
docker exec -it web bash
  cat /etc/os-release      # a different OS than your laptop!
  exit
docker stop web && docker rm web
```

### 1.2 Now one you did

```bash
mkdir ~/hello-web && cd ~/hello-web
```

**`app.py`** (set `upsiNN` to your ID):

```python
from flask import Flask
import socket, os

app = Flask(__name__)

@app.route("/")
def hello():
    return (
        f"<h1>Hello from {os.environ.get('OWNER', 'upsiNN')}!</h1>"
        f"<p>Served by container: <b>{socket.gethostname()}</b></p>"
        f"<p>Version: v1</p>"
    )

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

**`requirements.txt`**:

```
flask==3.0.3
```

**`Dockerfile`** (set `upsiNN`):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
ENV OWNER=upsiNN
CMD ["python", "app.py"]
```

### 1.3 Build, run, multiply

```bash
docker build -t hello-web .
docker images
docker run -d -p 8080:8080 --name hello1 hello-web
docker run -d -p 8090:8080 --name hello2 hello-web
curl localhost:8080
curl localhost:8090        # different container hostname in the banner!
docker rm -f hello1 hello2
```

✅ **Done when:** both ports answered with different container hostnames — two containers, one image, zero conflicts.

---

# Lab 2 — Bend a container to your will (50 min)

### 2.1 Runtime configuration — no rebuild

```bash
docker run -d -p 8080:8080 -e OWNER=production --name prod hello-web
curl localhost:8080        # banner says "production"
docker images              # confirm: no new image was created
docker rm -f prod
```

`ENV` in the Dockerfile set a *default*; `-e` overrides it per run. One image, many environments.

### 2.2 Data that survives death

```bash
docker run -d -v notes:/data --name writer alpine sh -c 'echo survived > /data/proof; sleep 999'
docker rm -f writer
docker run --rm -v notes:/data alpine cat /data/proof     # -> survived
docker volume ls
```

The container died; the named volume didn't.

### 2.3 The dev loop — bind mount

```bash
docker run -d -p 8080:8080 -v ~/hello-web:/app --name dev hello-web
curl localhost:8080
# now edit app.py — change the version line to v1.1 — then:
docker restart dev
curl localhost:8080        # v1.1, without a rebuild: the code came from YOUR folder
docker rm -f dev
```

### 2.4 Two containers, one network

Containers can't see each other until you give them a shared network — then they find each other **by name**:

```bash
docker network create appnet
docker run -d --name web2 --network appnet nginx
docker run --rm --network appnet alpine sh -c "wget -qO- http://web2 | head -3"
docker rm -f web2 && docker network rm appnet
```

That `http://web2` resolved with no IP address and no `-p` flag — internal traffic never needs published ports. Remember this: the client challenge's two-container stack stands on exactly this trick.

### 2.5 Break-fix: the cursed Dockerfile

```bash
mkdir ~/cursed && cd ~/cursed && cp ~/hello-web/app.py ~/hello-web/requirements.txt .
```

Create this **`Dockerfile`** exactly as written — it contains **two bugs**, one that bites at build time and one at run time:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirement.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python", "app"]
```

Build it, run it, and fix both — using the 4-step ritual (`ps -a` → `logs` → `exec` → `inspect`) to *diagnose* before you fix. Say out loud which step exposed each bug.

✅ **Done when:** override, volume and bind-mount all demonstrated, and the cursed image builds *and* serves — with both bugs named.

<details><summary>Only if truly stuck (ask for hints first!)</summary>
Bug 1: <code>COPY requirement.txt</code> — the file is called requirements.txt (build fails, error names the file). Bug 2: <code>CMD ["python", "app"]</code> — there is no file called <code>app</code>; the container exits instantly and <code>docker logs</code> shows Python's complaint.
</details>

---

# Lab 3 — Your own cluster (50 min)

### 3.1 Start it

```bash
minikube start --driver=docker     # 1–2 min; it's a full cluster inside Docker
kubectl get nodes                  # NAME: minikube   STATUS: Ready
```

### 3.2 Make it yours

Today's cluster is personal, but tomorrow's Day 2 cluster is **shared** — every student's `hello-web:v1` would land on the exact same cluster under the exact same name and silently overwrite each other's deployment. Bake your username into the tag now, before that's a problem:

```bash
docker tag hello-web hello-web:upsiNN-v1     # swap upsiNN for your real ID
docker images | grep hello-web               # both tags listed, same IMAGE ID
```

`docker tag` doesn't copy anything — it just adds a second label pointing at the same image layers. From here on, every command uses that personalized tag, not the bare `hello-web`.

### 3.3 Hand it your image

Kubernetes treats the `:latest` tag specially (it always tries to pull) — good thing you already have a real one:

```bash
minikube image load hello-web:upsiNN-v1   # ~30s: copies the image into the cluster
```

### 3.4 Run it

```bash
kubectl create deployment hello-web --image=hello-web:upsiNN-v1
kubectl get pods                   # 1 pod, Running
kubectl port-forward deployment/hello-web 8080:8080
```

In a **second terminal**: `curl localhost:8080` — your app, served by Kubernetes. Ctrl-C the port-forward when done looking.

### 3.5 Scale, then vandalize

```bash
kubectl scale deployment hello-web --replicas=4
kubectl get pods
kubectl delete pod <pick-one>
kubectl get pods                   # the replacement is already starting
```

### 3.6 The twin ritual

Yesterday's Docker ritual has an exact Kubernetes twin — practise it on a pod:

```bash
kubectl get pods                   # ~ docker ps
kubectl describe pod <name>        # ~ docker inspect (read the Events!)
kubectl logs <name>                # ~ docker logs
kubectl exec -it <name> -- sh      # ~ docker exec
```

✅ **Done when:** the app answers via port-forward from 4 pods and a deleted pod self-heals within seconds.

---

# Lab 4 — Manifests, v2, and the undo button (40 min)

### 4.1 From commands to files

`kubectl create deployment` was training wheels. Real Kubernetes is declared in YAML:

```bash
kubectl delete deployment hello-web     # clear the training-wheels version
mkdir ~/lab4 && cd ~/lab4
```

**`deployment.yaml`**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
spec:
  replicas: 4
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
        image: hello-web:upsiNN-v1     # your tag from Lab 3.2, not the bare one
        ports:
        - containerPort: 8080
```

**`service.yaml`**:

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

```bash
kubectl apply -f .
kubectl port-forward service/hello-web 8080:80
# second terminal: curl localhost:8080  -> v1 (or v1.1 if you kept Lab 2's edit)
```

### 4.2 Ship v2 with zero downtime

```bash
cd ~/hello-web
# edit app.py: change the version line to v2
docker build -t hello-web:upsiNN-v2 .
minikube image load hello-web:upsiNN-v2
```

Open a terminal with `kubectl get pods -w` running, then:

```bash
kubectl set image deployment/hello-web hello-web=hello-web:upsiNN-v2
```

Watch the waves: new pods start, old pods drain, traffic never stops. Refresh your curl: **v2**.

### 4.3 The undo button

```bash
kubectl rollout history deployment/hello-web
kubectl rollout undo deployment/hello-web      # waves in reverse -> v1
kubectl rollout undo deployment/hello-web      # ...and forward again -> v2
```

✅ **Done when:** you shipped v2 with zero downtime and reversed it with one command — and can say what the two ReplicaSets were doing during the waves.

---

# Before you leave — get your image off this VM (10 min)

Tomorrow this VM is gone — `terraform destroy` doesn't ask twice. Everything you built today (`hello-web:upsiNN-v1`, `upsiNN-v2`) exists in exactly one place: this Docker daemon, on this one disk. **This is the exact problem image registries exist to solve.** You'll feel the problem by hand first; tomorrow you'll meet the fix.

### 1. Package the image, not just the container

```bash
cd ~
docker save -o hello-web.tar hello-web:upsiNN-v2
ls -lh hello-web.tar        # a few hundred MB — every layer, baked into one file
```

`docker save` captures the whole *image*: layers, tags, and the `CMD`/`ENV` metadata that make it runnable. There's a similarly-named pair — `docker export` / `docker import` — that works on a *container's* filesystem instead: it flattens everything into one layer and throws away `CMD`, `ENV`, `EXPOSE` in the process. Use `save`, not `export`, or the image that comes back tomorrow won't know how to start itself.

### 2. Pull it down to your own laptop

Run this from **your own laptop's terminal**, not the VM — this direction only works pulling *from* the VM to somewhere that survives after it:

```bash
scp -i <path-to-your-private-key> upsiNN@<vm-external-ip>:~/hello-web.tar .
```

That file landing on your own disk is the only proof this image survives past today.

### 3. Confirm it's real

```bash
tar -tvf hello-web.tar | head -5     # peek inside — no Docker daemon needed for this part
```

You should see `manifest.json` and a handful of layer directories: your whole image, sitting in one file.

**Watch for:** `docker export`/`import` instead of `save`/`load` (loses `CMD` — the image boots into nothing); dropping the tag on `docker save hello-web` — that resolves to `hello-web:latest`, not the personalized `upsiNN-v2` you actually built (run `docker images` first and copy the exact tag); running `scp` *from* the VM instead of your laptop (wrong direction — nothing to push yet).

✅ **Done when:** `hello-web.tar` sits on your own laptop, and you can say out loud why `docker save`, not `docker export`, is the one that keeps the image runnable.

*Tomorrow: `docker load` this same tar on the Day 2 jumphost, then `docker tag` + `docker push` it to Artifact Registry — GKE can't reach into your Docker daemon the way minikube could, it only ever pulls from a registry. Load alone isn't enough; that push step is Lab 1.*

---

# Before you leave

Bring the same private key tomorrow — the Day 2 jumphost is a shared machine, already set up with gcloud/kubectl/docker/terraform/ansible, so there's nothing to install tonight. Two things to leave with:

- **`hello-web.tar`** on your own laptop (from above).
- **`userNN-key.json`** from the instructor (USB/DM) — this is your personal GCP identity for tomorrow, separate from tonight's `upsiNN` SSH login. You'll activate it once on the jumphost (Lab 0.2); nothing to do with it tonight except not lose it. **Guard it; never commit it to Git.**

---

# Quick troubleshooting reference

| Symptom | First move |
|---|---|
| Docker errors | Is the Docker daemon running? `sudo systemctl status docker` |
| Build fails | Read the error — it names the line and usually the file |
| Container exits instantly | `docker ps -a` for the exit code, then `docker logs` |
| minikube start fails | `minikube delete && minikube start --driver=docker` |
| Pod `ErrImagePull` on minikube | Did you `minikube image load` that exact tag? `:latest` is a trap |
| port-forward "connection refused" | Wrong port pair, or the pods aren't Ready yet (`kubectl get pods`) |

# Further reading

- docs.docker.com/get-started · kubernetes.io/docs/tutorials/kubernetes-basics
- minikube.sigs.k8s.io/docs — your cluster's manual
- 12factor.net — why config lives in the environment

*minikube is yours forever — keep experimenting tonight.*
