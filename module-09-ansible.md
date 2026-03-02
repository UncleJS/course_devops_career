# Module 09: Ansible

> Part of the [DevOps Career Course](./README.md) by UncleJS

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 09 of 15](https://img.shields.io/badge/module-09%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Intermediate-orange) ![Ansible 2.17+](https://img.shields.io/badge/Ansible-2.17%2B-EE0000?logo=ansible&logoColor=white) ![AWX stable](https://img.shields.io/badge/AWX-stable-EE0000?logo=ansible&logoColor=white) ![YAML · Agentless](https://img.shields.io/badge/features-YAML%20%C2%B7%20Agentless-lightgrey)

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: What is Ansible?](#beginner-what-is-ansible)
- [Beginner: Installation & Setup](#beginner-installation--setup)
- [Beginner: Inventory](#beginner-inventory)
- [Beginner: Ad-Hoc Commands](#beginner-ad-hoc-commands)
- [Beginner: Playbooks](#beginner-playbooks)
- [Intermediate: Variables & Facts](#intermediate-variables--facts)
- [Intermediate: Jinja2 Templates](#intermediate-jinja2-templates)
- [Intermediate: Roles](#intermediate-roles)
- [Intermediate: Handlers](#intermediate-handlers)
- [Intermediate: Ansible Vault — Encrypting Secrets](#intermediate-ansible-vault--encrypting-secrets)
- [Intermediate: Error Handling & Idempotency](#intermediate-error-handling--idempotency)
- [Intermediate: Dynamic Inventory](#intermediate-dynamic-inventory)
- [Advanced: AWX & Ansible Automation Platform](#advanced-awx--ansible-automation-platform)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Ansible is the most widely-used configuration management tool in DevOps. Where Terraform provisions infrastructure (creates servers, networks, databases), Ansible **configures** that infrastructure — installs software, manages services, deploys applications, and enforces system state.

Ansible is agentless — it connects to hosts over SSH and runs tasks. There's nothing to install on managed hosts beyond Python.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain Ansible's architecture and use cases
- Write inventory files to define managed hosts
- Run ad-hoc commands against groups of hosts
- Write playbooks that install software and configure services
- Use variables, facts, and Jinja2 templates for dynamic configs
- Organize reusable automation with roles
- Use handlers for conditional service restarts
- Encrypt sensitive data with Ansible Vault
- Write idempotent playbooks that are safe to re-run
- Use dynamic inventory for cloud environments
- Install and navigate AWX/Ansible Automation Platform for team-scale automation

[↑ Back to TOC](#table-of-contents)

---

## Beginner: What is Ansible?

### How Ansible Works

```
Control Node (your machine)
        │
        │ SSH
        ▼
Managed Hosts (servers you want to configure)
  ├── web01  (runs tasks via Python)
  ├── web02
  └── db01
```

1. Ansible reads your **playbook** (what to do)
2. Reads your **inventory** (which hosts)
3. Connects via **SSH**
4. Copies and executes **modules** (small Python scripts) on each host
5. Reports results back
6. Leaves no agent running on the host

### Ansible vs Other Tools

| Feature | Ansible | Chef/Puppet | Terraform |
|---|---|---|---|
| Language | YAML | Ruby DSL | HCL |
| Agent required | No (SSH) | Yes | No |
| Primary use | Config management, app deploy | Config management | Infrastructure provisioning |
| Learning curve | Low | High | Medium |
| Idempotent | Yes | Yes | Yes |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Installation & Setup

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y ansible

# RHEL/Rocky/Fedora
sudo dnf install -y ansible

# Via pip (always latest version)
pip3 install ansible

# Verify
ansible --version

# Test connection to localhost
ansible localhost -m ping
```

### ansible.cfg — Configuration File

```ini
# ansible.cfg (project directory or ~/.ansible.cfg)
[defaults]
inventory       = ./inventory
remote_user     = ubuntu
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False     # Disable for dev (enable in production)
stdout_callback = yaml        # Prettier output
forks           = 10          # Parallel tasks

[privilege_escalation]
become          = true
become_method   = sudo
become_user     = root
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Inventory

The inventory defines which hosts Ansible manages.

### Static Inventory (INI format)

```ini
# inventory/hosts

# Ungrouped host
bastion.example.com

# Web server group
[web]
web01.example.com
web02.example.com
web03.example.com ansible_port=2222

# Database group
[db]
db01.example.com ansible_user=postgres
db02.example.com

# A group of groups
[production:children]
web
db

# Variables for a group
[web:vars]
nginx_port=80
app_env=production

# Variables for a specific host
web01.example.com ansible_host=10.0.1.10 http_port=8080
```

### Static Inventory (YAML format)

```yaml
# inventory/hosts.yml
all:
  children:
    web:
      hosts:
        web01.example.com:
          ansible_host: 10.0.1.10
        web02.example.com:
          ansible_host: 10.0.1.11
      vars:
        nginx_port: 80
        app_env: production
    db:
      hosts:
        db01.example.com:
          ansible_host: 10.0.2.10
          ansible_user: postgres
```

```bash
# Test inventory
ansible-inventory --list
ansible-inventory --graph
ansible all --list-hosts
ansible web --list-hosts
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Ad-Hoc Commands

Ad-hoc commands run a single module against hosts without a playbook.

```bash
# Syntax: ansible <pattern> -m <module> -a "<arguments>"

# Ping all hosts
ansible all -m ping

# Run a shell command on web servers
ansible web -m shell -a "uptime"
ansible web -m command -a "df -h"    # command module (safer, no shell features)

# Check disk space on all hosts
ansible all -m shell -a "df -h / | tail -1"

# Install nginx on web group
ansible web -m apt -a "name=nginx state=present" --become

# Ensure a service is running
ansible web -m service -a "name=nginx state=started enabled=yes" --become

# Copy a file
ansible web -m copy -a "src=./index.html dest=/var/www/html/index.html" --become

# Create a directory
ansible all -m file -a "path=/opt/myapp state=directory mode=755" --become

# Gather facts about hosts
ansible web -m setup
ansible web -m setup -a "filter=ansible_os_family"

# Reboot all hosts
ansible all -m reboot --become
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Playbooks

A playbook is a YAML file containing one or more **plays** — each play targets a group of hosts and runs a list of **tasks**.

```yaml
# playbooks/install-nginx.yml
---
- name: Install and configure Nginx
  hosts: web
  become: true              # Run tasks as root (sudo)
  gather_facts: true        # Collect host information

  tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install nginx
      apt:
        name: nginx
        state: present       # present = install if missing

    - name: Ensure nginx is started and enabled
      service:
        name: nginx
        state: started
        enabled: true

    - name: Copy custom nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx   # Trigger handler if changed

    - name: Create web root directory
      file:
        path: /var/www/myapp
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Deploy index.html
      copy:
        content: "<h1>Hello from {{ inventory_hostname }}</h1>"
        dest: /var/www/myapp/index.html
        mode: '0644'

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

```bash
# Run a playbook
ansible-playbook playbooks/install-nginx.yml

# Dry run (check mode — no changes made)
ansible-playbook playbooks/install-nginx.yml --check

# Show diff of what would change
ansible-playbook playbooks/install-nginx.yml --check --diff

# Limit to specific hosts
ansible-playbook playbooks/install-nginx.yml --limit web01

# Run with verbose output
ansible-playbook playbooks/install-nginx.yml -v
ansible-playbook playbooks/install-nginx.yml -vvv    # Extra verbose
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Variables & Facts

### Variable Precedence (lowest to highest)

1. Role defaults
2. Inventory vars
3. Playbook group_vars
4. Playbook host_vars
5. Play vars
6. Task vars (`vars:` in a task)
7. Extra vars (`-e` on command line) ← highest priority

### Defining Variables

```yaml
# group_vars/web.yml — variables for 'web' group
nginx_port: 80
app_name: myapp
max_connections: 1024

# host_vars/web01.example.com.yml — variables for specific host
nginx_port: 8080    # Override for this host only

# In playbook
- name: Deploy app
  hosts: web
  vars:
    deploy_version: "1.5.2"
    config_dir: "/etc/{{ app_name }}"
```

### Using Variables

```yaml
- name: Create app config directory
  file:
    path: "{{ config_dir }}"
    state: directory

- name: Configure nginx port
  lineinfile:
    path: /etc/nginx/nginx.conf
    regexp: 'listen'
    line: "    listen {{ nginx_port }};"
```

### Facts — Gathered Host Information

```yaml
# Access facts about the managed host
- name: Print OS information
  debug:
    msg: "Running {{ ansible_distribution }} {{ ansible_distribution_version }}"

- name: Configure based on OS family
  apt:
    name: nginx
  when: ansible_os_family == "Debian"

- name: Configure based on OS family (RHEL)
  dnf:
    name: nginx
  when: ansible_os_family == "RedHat"

# Useful facts
# ansible_hostname       — short hostname
# ansible_fqdn           — fully qualified hostname
# ansible_os_family      — "Debian" or "RedHat"
# ansible_distribution   — "Ubuntu", "CentOS", etc.
# ansible_memtotal_mb    — total RAM in MB
# ansible_processor_vcpus — number of CPU cores
# ansible_default_ipv4.address — primary IP
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Jinja2 Templates

Templates let you generate configuration files dynamically from variables.

### nginx.conf.j2

```jinja2
# /templates/nginx.conf.j2
worker_processes {{ ansible_processor_vcpus }};

events {
    worker_connections {{ max_connections | default(1024) }};
}

http {
    server {
        listen {{ nginx_port }};
        server_name {{ ansible_fqdn }};

        location / {
            root /var/www/{{ app_name }};
            index index.html;
        }

        {% if enable_ssl | default(false) %}
        listen 443 ssl;
        ssl_certificate /etc/ssl/{{ app_name }}.crt;
        ssl_certificate_key /etc/ssl/{{ app_name }}.key;
        {% endif %}
    }

    upstream backend {
        {% for server in backend_servers %}
        server {{ server }}:{{ app_port }};
        {% endfor %}
    }
}
```

### Using the Template Module

```yaml
- name: Deploy nginx configuration
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: nginx -t -c %s    # Validate before deploying
  notify: Reload nginx
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Roles

Roles are the standard way to organize and reuse Ansible automation.

### Role Directory Structure

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml         # Main task list
    ├── handlers/
    │   └── main.yml         # Handlers
    ├── templates/
    │   └── nginx.conf.j2    # Jinja2 templates
    ├── files/
    │   └── index.html       # Static files
    ├── vars/
    │   └── main.yml         # Role variables (high priority)
    ├── defaults/
    │   └── main.yml         # Default variables (low priority, overridable)
    ├── meta/
    │   └── main.yml         # Role metadata and dependencies
    └── README.md
```

```bash
# Generate role skeleton
ansible-galaxy role init roles/nginx
```

### Role defaults/main.yml

```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_user: www-data
max_connections: 1024
enable_ssl: false
```

### Using Roles in a Playbook

```yaml
# playbooks/site.yml
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - nginx          # Shorthand
    - role: postgresql
      vars:
        pg_version: 16
    - role: app-deploy
      when: deploy_app | default(false)
```

### Installing Community Roles

```bash
# From Ansible Galaxy
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install -r requirements.yml

# requirements.yml
# roles:
#   - name: geerlingguy.nginx
#   - name: geerlingguy.postgresql
#     version: 5.0.0
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Handlers

Handlers run only when notified, and only once — even if notified multiple times. Perfect for service restarts.

```yaml
# tasks/main.yml
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Copy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - Validate nginx config
    - Reload nginx

- name: Copy SSL certificate
  copy:
    src: files/cert.pem
    dest: /etc/ssl/cert.pem
  notify: Reload nginx     # Same handler — only runs ONCE at end

# handlers/main.yml
- name: Validate nginx config
  command: nginx -t
  changed_when: false

- name: Reload nginx
  service:
    name: nginx
    state: reloaded

- name: Restart nginx
  service:
    name: nginx
    state: restarted
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Ansible Vault — Encrypting Secrets

Vault encrypts sensitive data in your playbooks and variable files.

```bash
# Create a new encrypted file
ansible-vault create group_vars/all/secrets.yml

# Edit an encrypted file
ansible-vault edit group_vars/all/secrets.yml

# Encrypt an existing file
ansible-vault encrypt group_vars/all/secrets.yml

# Decrypt (permanently — careful!)
ansible-vault decrypt group_vars/all/secrets.yml

# View encrypted file without decrypting to disk
ansible-vault view group_vars/all/secrets.yml

# Encrypt a single string value
ansible-vault encrypt_string 'mysecretpassword' --name 'db_password'

# Run playbook with vault password
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_password
```

### Encrypted Variable File

```yaml
# group_vars/all/secrets.yml (encrypted with ansible-vault)
# After decryption it contains:
db_password: "SuperSecret123!"
api_key: "sk-abc123xyz"
ssl_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvgIBAD...
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Error Handling & Idempotency

```yaml
# Idempotency — same result whether run once or 100 times
- name: Create user
  user:
    name: appuser
    state: present           # Will skip if user exists

- name: Ensure directory exists
  file:
    path: /opt/myapp
    state: directory         # Will skip if directory exists

# Ignoring errors
- name: Check if service exists
  command: systemctl status myapp
  register: service_status
  ignore_errors: true

- name: Start service if it exists
  service:
    name: myapp
    state: started
  when: service_status.rc == 0

# Blocks for error handling (try/catch/finally)
- block:
    - name: Attempt deployment
      shell: ./deploy.sh

    - name: Verify deployment
      uri:
        url: http://localhost:8080/healthz
        status_code: 200

  rescue:
    - name: Rollback on failure
      shell: ./rollback.sh

    - name: Send alert
      mail:
        to: ops@example.com
        subject: "Deployment FAILED on {{ inventory_hostname }}"

  always:
    - name: Clean up temp files
      file:
        path: /tmp/deploy
        state: absent

# Register and use task output
- name: Get disk usage
  command: df -h /
  register: disk_info

- name: Show disk info
  debug:
    var: disk_info.stdout_lines

- name: Fail if disk is over 90%
  fail:
    msg: "Disk usage is critical!"
  when: "'9' in disk_info.stdout"
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Dynamic Inventory

For cloud environments where servers come and go, use dynamic inventory that queries the cloud API.

```bash
# Install AWS dynamic inventory plugin
pip3 install boto3 botocore

# aws_ec2 dynamic inventory
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2
filters:
  instance-state-name: running
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
hostnames:
  - private-ip-address

# Use it
ansible-inventory -i inventory/aws_ec2.yml --list
ansible -i inventory/aws_ec2.yml role_web -m ping
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: AWX & Ansible Automation Platform

The command-line is fine for a single engineer, but teams need **role-based access control, audit logs, scheduling, credentials vaulting, and a GUI**. **AWX** is the open-source upstream for **Red Hat Ansible Automation Platform (AAP)**.

### AWX vs Ansible Automation Platform

| Feature | AWX | Ansible Automation Platform |
|---|---|---|
| **License** | Open source (Apache 2.0) | Red Hat subscription |
| **Support** | Community | Red Hat SLA |
| **Execution environments** | Yes | Yes (enhanced) |
| **Best for** | Self-hosted, open-source shops | Enterprise, regulated environments |

### Installing AWX on Kubernetes

```bash
# Install AWX Operator (manages AWX lifecycle as a K8s CR)
kubectl apply -k "https://github.com/ansible/awx-operator/config/default?ref=2.19.1"

# Create the AWX instance
cat <<'EOF' | kubectl apply -f -
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: nodeport
  nodeport_port: 30080
EOF

# Watch the operator deploy AWX (takes 5–10 minutes)
kubectl get pods -n awx -w

# Retrieve the auto-generated admin password
kubectl get secret awx-admin-password -n awx \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Access AWX at `http://<node-ip>:30080` with `admin` / `<retrieved password>`.

### Key AWX Concepts

| Concept | Description |
|---|---|
| **Organization** | Top-level namespace — groups users, inventories, projects |
| **Project** | A Git repository containing playbooks |
| **Inventory** | Hosts/groups — can sync from a Git file or cloud provider |
| **Credentials** | Encrypted SSH keys, vault passwords, cloud tokens |
| **Job Template** | A saved "run this playbook against this inventory" configuration |
| **Workflow Template** | Chain multiple Job Templates with success/failure branching |
| **Schedule** | Cron-style trigger for Job Templates |

### Connecting a Git Project

```
1. Add Credentials → Source Control → SSH or HTTPS token for your Git host
2. Create a Project → point to your playbook repo URL + branch
3. AWX will sync (git clone) the project on demand or on schedule
4. Create an Inventory → Source: "Sourced from a Project" → point to your inventory file in the repo
5. Create a Job Template → Project + Playbook + Inventory + Credentials → Save
6. Launch → AWX runs the playbook, streams output live, stores audit log
```

### AWX REST API & CLI

AWX exposes a full REST API — useful for triggering pipelines from CI/CD:

```bash
# Install the AWX CLI
pip install awxkit

# Configure connection
awx login --conf.host https://awx.example.com \
          --conf.username admin \
          --conf.password "${AWX_PASSWORD}"

# List job templates
awx job_templates list --all

# Launch a job template by ID
awx job_templates launch 42 \
  --extra_vars '{"target_env": "staging"}' \
  --monitor    # Stream output and block until complete
```

**Triggering AWX from GitHub Actions:**

```yaml
- name: Trigger Ansible playbook via AWX
  run: |
    curl -s -X POST \
      -H "Authorization: Bearer ${AWX_TOKEN}" \
      -H "Content-Type: application/json" \
      -d '{"extra_vars": {"image_tag": "${{ github.sha }}"}}' \
      https://awx.example.com/api/v2/job_templates/42/launch/
```

### Execution Environments

AWX 19+ uses **Execution Environments** — container images that bundle Ansible, collections, and Python dependencies. This eliminates "works on my machine" problems:

```bash
# Build a custom EE with ansible-builder
pip install ansible-builder

cat > execution-environment.yml <<'EOF'
version: 3
images:
  base_image:
    name: quay.io/ansible/awx-ee:latest
dependencies:
  galaxy:
    collections:
      - name: amazon.aws
        version: ">=6.0.0"
      - name: community.postgresql
  python:
    - boto3>=1.28
    - psycopg2-binary
EOF

ansible-builder build -t my-company/custom-ee:1.0 --prune-images
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Purpose |
|---|---|
| `ansible all -m ping` | Test connectivity to all hosts |
| `ansible-playbook playbook.yml` | Run a playbook |
| `ansible-playbook --check` | Dry run |
| `ansible-playbook --diff` | Show file diffs |
| `ansible-playbook --limit host` | Run on specific hosts |
| `ansible-galaxy role init` | Create role skeleton |
| `ansible-galaxy install` | Install community roles |
| `ansible-vault create/edit/encrypt` | Manage encrypted files |
| `ansible-inventory --graph` | Visualize inventory |
| `ansible-playbook -e "key=val"` | Pass extra variables |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 9.1 — Setup & First Ping

1. Install Ansible on your machine
2. Create an inventory with `localhost`
3. Run `ansible all -m ping`
4. Run `ansible all -m setup | head -50` to view gathered facts

### Lab 9.2 — Install a Web Stack

Write a playbook that:
1. Installs Nginx on web hosts
2. Ensures the service is started and enabled
3. Deploys a custom `index.html`
4. Runs with `--check` first, then `--diff`, then for real

### Lab 9.3 — Dynamic Configuration with Templates

1. Create a Jinja2 template for an nginx virtual host config
2. Use `ansible_fqdn` and variables for dynamic values
3. Deploy using the `template` module with a `notify` handler

### Lab 9.4 — Build a Role

1. Use `ansible-galaxy role init` to create a `webserver` role
2. Migrate your nginx playbook into the role structure
3. Add defaults for port and document root
4. Call the role from a site.yml playbook

### Lab 9.5 — Ansible Vault

1. Create an encrypted `secrets.yml` file with a fake database password
2. Use the secret in a task via a template
3. Run the playbook using `--ask-vault-pass`
4. Create a vault password file and run without interactive prompt

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/) — Community roles
- [Jeff Geerling's Ansible for DevOps](https://www.ansiblefordevops.com/)
- [Molecule — Ansible Role Testing](https://ansible.readthedocs.io/projects/molecule/)
- [Glossary: Ansible](./glossary.md#a), [Idempotent](./glossary.md#i), [Jinja2](./glossary.md#j), [Playbook](./glossary.md#p)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
