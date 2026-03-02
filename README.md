# DevOps Career Course

> A comprehensive, progressive DevOps curriculum — from Linux fundamentals to advanced GitOps and production-grade deployments.

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![15 Modules](https://img.shields.io/badge/modules-15-grey) ![7 Phases](https://img.shields.io/badge/phases-7-grey) ![Levels](https://img.shields.io/badge/levels-Beginner%20%E2%86%92%20Advanced-brightgreen) ![Linux](https://img.shields.io/badge/Linux-Bash%20%C2%B7%20systemd-FCC624?logo=linux&logoColor=black) ![Containers](https://img.shields.io/badge/Containers-Docker%20%C2%B7%20Podman%20%C2%B7%20K8s-2496ED?logo=docker&logoColor=white) ![Cloud](https://img.shields.io/badge/Cloud-AWS%20%C2%B7%20Azure%20%C2%B7%20GCP-FF9900?logo=amazonaws&logoColor=white) ![IaC](https://img.shields.io/badge/IaC-Terraform%20%C2%B7%20Ansible-7B42BC?logo=terraform&logoColor=white) ![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions%20%C2%B7%20GitLab%20%C2%B7%20Jenkins-2088FF?logo=githubactions&logoColor=white)

---

## Table of Contents

- [About This Course](#about-this-course)
- [Who Is This For](#who-is-this-for)
- [Learning Path](#learning-path)
- [Course Modules](#course-modules)
- [Tools Covered](#tools-covered)
- [Certifications Alignment](#certifications-alignment)
- [How to Use This Course](#how-to-use-this-course)
- [Glossary](#glossary)
- [License](#license)

---

## About This Course

This course is a structured, hands-on curriculum for learning DevOps from the ground up. It covers every major tool and concept you need to land a job as a DevOps Engineer, Site Reliability Engineer (SRE), Platform Engineer, or Cloud Infrastructure Engineer.

Each module includes:
- **Overview** — what the topic is and why it matters in real-world DevOps
- **Learning Objectives** — what you will be able to do by the end
- **Beginner Section** — core concepts explained clearly
- **Intermediate Section** — production patterns and deeper coverage
- **Tools & Commands Reference** — cheatsheet-style quick reference
- **Hands-On Labs** — practical exercises you can run on your own machine
- **Further Reading** — official docs, certifications, and next steps

[↑ Back to TOC](#table-of-contents)

---

## Who Is This For

| Level | Description |
|---|---|
| **Beginner** | No prior DevOps experience — you will learn everything from scratch |
| **Intermediate** | Developers or sysadmins transitioning into DevOps roles |
| **Career Switcher** | Anyone looking to enter the DevOps job market |

Prerequisites: Basic comfort using a computer. No programming experience required to start.

[↑ Back to TOC](#table-of-contents)

---

## Learning Path

The modules are ordered progressively. Each module builds on the previous one.

```
PHASE 1 — FOUNDATION (Modules 1–4)
  Module 01 → Linux Fundamentals
  Module 02 → Scripting & Automation
  Module 03 → Networking Basics
  Module 04 → Git & Version Control

PHASE 2 — CONTAINERIZATION (Modules 5–6)
  Module 05 → Containers: Docker & Podman
  Module 06 → Kubernetes

PHASE 3 — CLOUD & INFRASTRUCTURE AS CODE (Modules 7–9)
  Module 07 → Cloud Fundamentals (AWS, Azure, GCP)
  Module 08 → IaC: Terraform & OpenTofu
  Module 09 → Ansible

PHASE 4 — CI/CD (Module 10)
  Module 10 → CI/CD Pipelines: GitHub Actions, GitLab CI, Jenkins

PHASE 5 — OBSERVABILITY (Modules 11–12)
  Module 11 → Monitoring: Prometheus, Grafana & Zabbix
  Module 12 → Logging: ELK Stack & Loki

PHASE 6 — SECURITY & RELIABILITY (Modules 13–14)
  Module 13 → Security & DevSecOps
  Module 14 → High Availability & Disaster Recovery

PHASE 7 — ADVANCED (Module 15)
  Module 15 → Advanced Topics, GitOps & Capstone Project
```

[↑ Back to TOC](#table-of-contents)

---

## Course Modules

| # | Module | Topics |
|---|---|---|
| 01 | [Linux Fundamentals](./module-01-linux-fundamentals.md) | Navigation, permissions, processes, SSH, networking commands, piping |
| 02 | [Scripting & Automation](./module-02-scripting-automation.md) | Bash scripting, Python for DevOps, cron jobs, awk, sed |
| 03 | [Networking Basics](./module-03-networking.md) | TCP/IP, DNS, HTTP/HTTPS, ports, firewalls, load balancing |
| 04 | [Git & Version Control](./module-04-git-version-control.md) | Git basics, branching, merging, workflows, hooks, advanced patterns |
| 05 | [Containers: Docker & Podman](./module-05-containers.md) | Images, containers, Dockerfile, Compose, rootless, registries, CLI comparison |
| 06 | [Kubernetes](./module-06-kubernetes.md) | Architecture, pods, deployments, services, Helm, scaling, health checks |
| 07 | [Cloud Fundamentals](./module-07-cloud-fundamentals.md) | AWS, Azure, GCP — compute, storage, networking, IAM, CLI tools |
| 08 | [IaC: Terraform & OpenTofu](./module-08-iac-terraform-opentofu.md) | HCL, providers, state, modules, Terraform vs OpenTofu, migration |
| 09 | [Ansible](./module-09-ansible.md) | Playbooks, roles, inventory, YAML, Jinja2, idempotency |
| 10 | [CI/CD Pipelines](./module-10-cicd-pipelines.md) | GitHub Actions, GitLab CI, Jenkins — pipeline design, artifacts, strategies |
| 11 | [Monitoring & Observability](./module-11-monitoring-observability.md) | Prometheus, Grafana, Zabbix — metrics, dashboards, alerting, agents |
| 12 | [Logging](./module-12-logging.md) | ELK Stack, Loki, structured logging, log retention |
| 13 | [Security & DevSecOps](./module-13-security-devsecops.md) | RBAC, Vault, SAST/DAST, Policy as Code, container security |
| 14 | [High Availability & DR](./module-14-high-availability-dr.md) | HA concepts, RTO/RPO, backups, failover, multi-region |
| 15 | [Advanced Topics & Capstone](./module-15-advanced-topics.md) | GitOps, ArgoCD, service mesh, Operators, capstone project |

[↑ Back to TOC](#table-of-contents)

---

## Tools Covered

| Category | Tools |
|---|---|
| **OS / Shell** | Linux, Bash, Zsh |
| **Scripting** | Bash, Python |
| **Version Control** | Git, GitHub, GitLab |
| **Containers** | Docker, Podman, Docker Compose, Podman Compose |
| **Orchestration** | Kubernetes, Helm, kubectl |
| **Cloud** | AWS, Microsoft Azure, Google Cloud Platform |
| **IaC** | Terraform, OpenTofu |
| **Config Management** | Ansible |
| **CI/CD** | GitHub Actions, GitLab CI/CD, Jenkins |
| **Monitoring** | Prometheus, Grafana, Zabbix |
| **Logging** | Elasticsearch, Logstash, Kibana (ELK), Loki, Promtail |
| **Security** | HashiCorp Vault, Trivy, OPA/Gatekeeper, Snyk |
| **GitOps** | ArgoCD, Flux |
| **Service Mesh** | Istio, Linkerd |
| **Networking** | Nginx, HAProxy |

[↑ Back to TOC](#table-of-contents)

---

## Certifications Alignment

This course prepares you for the following industry certifications:

| Certification | Relevant Modules |
|---|---|
| **Linux Foundation — LFCS** (Linux Foundation Certified Sysadmin) | 01, 02 |
| **HashiCorp Certified: Terraform Associate** | 08 |
| **CKA** (Certified Kubernetes Administrator) | 06 |
| **CKAD** (Certified Kubernetes Application Developer) | 05, 06 |
| **AWS Certified Cloud Practitioner** | 07 |
| **AWS Certified DevOps Engineer** | 07, 08, 09, 10, 11 |
| **GitLab Certified CI/CD Associate** | 10 |
| **Certified DevSecOps Professional (CDP)** | 13 |

[↑ Back to TOC](#table-of-contents)

---

## How to Use This Course

1. **Work through modules in order** — each module builds on the previous
2. **Don't skip the labs** — hands-on practice is how the concepts stick
3. **Set up a lab environment** — a Linux VM or WSL2 on Windows is sufficient for most modules
4. **Use the glossary** — if you encounter an unfamiliar term, check [glossary.md](./glossary.md)
5. **Revisit modules** — it's normal to return to earlier modules as later ones reference them

### Recommended Lab Environment

| Option | Details |
|---|---|
| **Local VM** | Ubuntu 24.04 LTS or Rocky Linux 9 in VirtualBox/VMware |
| **WSL2** | Windows Subsystem for Linux 2 with Ubuntu |
| **Cloud VM** | Free tier EC2 (AWS), VM (Azure), or Compute Engine (GCP) |
| **Local container** | `docker run -it ubuntu:24.04 bash` or `podman run -it ubuntu:24.04 bash` |

[↑ Back to TOC](#table-of-contents)

---

## Glossary

A full A–Z glossary of DevOps terms is available in [glossary.md](./glossary.md).

[↑ Back to TOC](#table-of-contents)

---

## License

**© 2026 UncleJS**

This work is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-nc-sa/4.0/).

[![CC BY-NC-SA 4.0](https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

**You are free to:**
- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material

**Under the following terms:**
- **Attribution** — You must give appropriate credit to **UncleJS**, provide a link to the license, and indicate if changes were made
- **NonCommercial** — You may not use the material for commercial purposes
- **ShareAlike** — If you remix, transform, or build upon the material, you must distribute your contributions under the same license

[↑ Back to TOC](#table-of-contents)
