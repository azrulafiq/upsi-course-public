# Managing Docker & Kubernetes Clusters with Ansible and Terraform

A 2-day instructor-led course. This repo has everything you need — read this first so you know which file to open and when.

## Before the course

- **[Pre-Course-Checklist.md](Pre-Course-Checklist.md)** — install WSL, Docker Desktop, and the CLI tools (gcloud, kubectl, terraform, ansible, minikube) *before* Day 1. Do this at home; Day 1 starts with a 2-minute check, not an hour of installs.

## Each day

Each day has one slide deck (concepts) and one lab guide (hands-on steps). Follow along with the deck during lecture, then work through the matching lab guide at your laptop.

| Day | Slides | Lab Guide |
|---|---|---|
| Day 1 | [Day1-Docker-Minikube.pptx](Day1-Docker-Minikube.pptx) | [Day1-Lab-Guide.md](Day1-Lab-Guide.md) |
| Day 2 | [Day2-GKE-Terraform-Ansible.pptx](Day2-GKE-Terraform-Ansible.pptx) | [Day2-Lab-Guide.md](Day2-Lab-Guide.md) |

- **Day 1 — Docker & Kubernetes on your laptop.** Everything runs locally via minikube. No cloud accounts needed.
- **Day 2 — The big guns: GKE, Terraform & Ansible.** Uses a shared Google Cloud project and cluster; you'll authenticate with a service-account key handed out at the end of Day 1.

## How to use the `.md` lab guides

They're plain Markdown — readable on GitHub, in a text editor, or any Markdown viewer. Follow them top to bottom:

- Code blocks are commands to copy-paste into your **WSL Ubuntu terminal**.
- Replace placeholders like `userNN` / `groupNN` with your assigned ID (on the board).
- Each lab ends with a "✅ Done when..." checkpoint — use it to confirm you're on track before moving on.

Stuck? Flag it during the session so you can pair with a neighbour while it's fixed.



docker run -d --name db --network wpnet -v dbdata:/var/lib/mysql \
  -e MARIADB_ROOT_PASSWORD=rootpass -e MARIADB_DATABASE=wordpress \
  -e MARIADB_USER=wp -e MARIADB_PASSWORD=wppass mariadb:latest
