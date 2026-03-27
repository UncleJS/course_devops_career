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

```mermaid
flowchart LR
    CN["Control Node<br/>(your machine / CI runner)"]
    PB["playbook<br/>(YAML task list)"]
    INV["inventory<br/>(target hosts)"]
    SSH["SSH connection"]
    MOD["module<br/>(Python on remote host)"]
    H1["Managed Host 1<br/>(web01)"]
    H2["Managed Host 2<br/>(web02)"]
    H3["Managed Host 3<br/>(db01)"]
    RES["result<br/>(ok / changed / failed)"]

    PB --> CN
    INV --> CN
    CN -->|"SSH"| SSH
    SSH --> MOD
    MOD --> H1
    MOD --> H2
    MOD --> H3
    H1 -->|"returns"| RES
    H2 -->|"returns"| RES
    H3 -->|"returns"| RES
    RES --> CN
```

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

Ansible matters because most operational work is repetitive long before it becomes complex. Installing packages, templating configs, restarting services, creating users, rotating secrets, and enforcing the same baseline across dozens of hosts are all tasks that humans can do manually but should not keep doing manually. The value of Ansible is that it lets you describe desired system state in a readable format and apply it consistently across many machines without installing a heavy agent everywhere.

**Idempotency** is the central contract Ansible makes. Running a playbook twice should produce the same result as running it once. If nginx is already installed and already running, running the playbook again should report `ok` — not `changed`, and certainly not an error. This matters deeply for safe re-runs. During an incident you might need to run a playbook against fifty hosts to push an emergency configuration change. If you are uncertain whether the playbook has already run on some of them, idempotency means you can run it everywhere without fear of double-applying something that breaks. Most Ansible modules (apt, yum, service, file, user, template) are natively idempotent. The risky ones are `command`, `shell`, and `raw` — they run the command unconditionally unless you add `creates:`, `removes:`, or `changed_when:` logic to make them conditional.

As you move through this module, keep one distinction in mind: Terraform is usually about provisioning infrastructure, while Ansible is usually about configuring and operating what already exists. Those tools overlap sometimes, but they solve different layers of the automation stack. Ansible becomes especially powerful after infrastructure is created, when you need to turn fresh servers into working application environments in a repeatable way.

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

Good installation and setup are less about getting the CLI on your laptop and more about creating a predictable execution environment. Automation is only trustworthy when it behaves the same way every time it runs. That means deciding where inventory lives, which SSH key is used, what user connects by default, how privilege escalation works, and how output is formatted for debugging. Those choices seem small at first, but they become very important once multiple engineers or CI jobs are running the same playbooks.

This is also where beginners often learn their first Ansible lesson: connection problems are usually more common than playbook problems. Before building elaborate roles, make sure the control node can authenticate cleanly, reach the target hosts, and escalate privileges safely. If those basics are shaky, every later section feels harder than it should.

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

Inventory is more than a list of servers. It is the model Ansible uses to understand your estate: which hosts exist, how they are grouped, what variables apply to them, and how tasks should target them. A clean inventory reflects real operational boundaries such as web tier, database tier, production, staging, or region. A messy inventory turns every playbook into a guessing game.

Think of inventory as the bridge between infrastructure reality and automation intent. If host grouping is thoughtful, playbooks become simple and expressive. If grouping is inconsistent, engineers end up hardcoding exceptions directly into tasks, which is usually the start of brittle automation. The examples below show both syntax and structure because both matter in practice.

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

Ad-hoc commands are useful because they let you test connectivity, inspect state, and perform one-off tasks quickly, but they should not become your long-term automation strategy. They are best for exploration, diagnostics, and emergency fixes when you need a fast answer. If you find yourself running the same ad-hoc command repeatedly, that is a sign the action belongs in a playbook or role.

This is an important transition point for learners. Ad-hoc usage teaches the Ansible execution model in a low-risk way: target hosts, choose a module, pass arguments, inspect output. Once that mental model feels natural, playbooks stop looking like abstract YAML and start reading like structured, repeatable operations.

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

Playbooks are where Ansible turns from a remote command runner into an automation system. Instead of saying "run this command on those servers," you start expressing desired state in a durable, reviewable form. That matters operationally because playbooks can be code-reviewed, tested in lower environments, scheduled, and rerun safely. They become part of the delivery workflow, not just a bag of shell commands.

The hierarchy to internalize is **play → task → module**. A play declares the target hosts and context (`hosts`, `become`, `gather_facts`). Tasks are individual units of work, each calling exactly one module. The `become: yes` pattern uses sudo to escalate to root after connecting as a normal user — this is safer than connecting as root directly, because you can audit which user escalated, and many cloud images disable root SSH login by default. Ansible's execution model is sequential: gather facts first (unless disabled), then run each task in order. Handlers are a special case — they accumulate notifications during task execution and fire exactly once at the end of the play, regardless of how many tasks notified them.

Notice the shape of a good playbook: it declares the target hosts, whether privilege escalation is required, whether facts should be gathered, and then a sequence of tasks that each do one understandable thing. This structure is what makes debugging manageable. When a deployment fails, you want to know exactly which task changed what, and why.

```mermaid
flowchart TD
    PLAY1["Play 1<br/>hosts: web<br/>become: true"]
    GF["gather_facts<br/>(collect host info)"]
    T1["Task 1: apt - update cache"]
    T2["Task 2: apt - install nginx"]
    T3["Task 3: template - nginx.conf"]
    H1["Handler: Restart nginx<br/>(only if notified)"]
    PLAY2["Play 2<br/>hosts: db<br/>become: true"]
    T4["Task 1: apt - install postgresql"]
    T5["Task 2: service - start postgresql"]

    PLAY1 --> GF --> T1 --> T2 --> T3
    T3 -->|"notify"| H1
    H1 -.->|"runs at play end"| PLAY1
    PLAY1 --> PLAY2
    PLAY2 --> T4 --> T5
```

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

Variables and facts are what let one playbook adapt to many environments without becoming unreadable. Variables express the choices your automation should accept, while facts describe the machine Ansible is currently talking to. Together, they let you write automation that is flexible but still deterministic. Without them, you end up duplicating playbooks for every environment or baking environment assumptions directly into tasks.

The **variable precedence chain** is essential to understand because its violations are the cause of most "why is Ansible using the wrong value?" debugging sessions. The chain runs roughly from lowest to highest priority: role `defaults/main.yml` (lowest, designed to be overridden), inventory variables, `group_vars`, `host_vars`, play-level `vars:`, task-level `vars:`, and finally extra-vars passed on the command line (`-e`). `extra_vars` always win — this is useful for one-off overrides in CI pipelines but dangerous if someone starts relying on it for permanent configuration. The practical rule is: put defaults in `defaults/main.yml`, put environment-specific values in `group_vars` or `host_vars`, and reserve `extra_vars` for emergency or CI overrides.

**Ansible facts** are variables automatically gathered from the managed host at the start of a play (via the `setup` module). They include the OS family and distribution, available memory, number of CPUs, network interfaces, IP addresses, and hostname. Facts make playbooks genuinely dynamic: you can write a single playbook that installs packages correctly on both Ubuntu (using `apt`) and RHEL (using `dnf`) by branching on `ansible_os_family`. Facts also let templates render host-specific values like `ansible_fqdn` or `ansible_default_ipv4.address` without requiring those values to be manually maintained in inventory.

This is also the point where automation can become confusing if naming and precedence are sloppy. Many Ansible mistakes come from not knowing which value wins, where it came from, or whether a variable was intended as a default, an override, or a secret. Treat variables as an interface and facts as runtime context, and the rest of the section becomes much easier to reason about.

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

Templates are one of the clearest examples of why Ansible is more than package installation. Real systems need configuration files that differ slightly by environment, hostname, port, feature flag, or upstream dependency. Managing those files by hand does not scale, and copying nearly identical files across repositories becomes a maintenance problem quickly. Jinja2 gives you a safe middle ground: shared structure with controlled variation.

The key operational benefit is not just convenience. It is consistency. A template makes configuration changes auditable and repeatable, while validation hooks help prevent you from distributing a broken config file to every server at once. That is why templating and validation usually appear together in mature automation.

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

Roles are the unit of reuse in Ansible. When you create a role for nginx, for postgresql, or for a hardening baseline, you give that automation a home that other playbooks can invoke by name. The directory structure is the role's interface: `defaults/main.yml` for overridable defaults, `vars/main.yml` for constants, `tasks/main.yml` as the entry point, `handlers/main.yml` for event-driven actions, `templates/` for Jinja2 files, and `files/` for static content. That predictable layout means any Ansible practitioner can navigate an unfamiliar role quickly without reading a README.

The **Galaxy dependency system** (declared in `meta/main.yml`) lets roles declare their own dependencies on other roles. When you run `ansible-galaxy install`, Ansible resolves and downloads the full dependency graph. This is powerful for complex setups but requires version discipline — pinning role versions in `requirements.yml` is as important as pinning package versions in application code. An unpinned community role can introduce breaking changes on any `galaxy install` run.

The `include_role` vs `import_role` distinction is subtle but operationally important. `import_role` is **static** — it is processed before the play runs, which means Ansible knows its tasks exist at parse time. This allows `when:` conditionals and `with_items` loops on the role to work predictably. `include_role` is **dynamic** — the role's tasks are loaded at runtime, which means they work inside loops and other dynamic contexts but cannot be targeted by `--tags` or `--skip-tags` unless the included role itself uses those tags. When in doubt, use `import_role` for top-level role inclusions and `include_role` when you need dynamic, loop-driven role execution.

Roles are where Ansible starts to feel like an engineering system instead of a collection of playbooks. They give you a packaging model for automation: defaults, tasks, handlers, templates, files, and metadata all live in predictable places. That structure matters because automation grows quickly. What begins as a simple web server setup often becomes application deployment, secrets handling, OS tuning, monitoring integration, and lifecycle tasks.

The design goal of a role is similar to the design goal of a good software module: one clear responsibility, sensible defaults, and a clean interface for overrides. If roles become giant bundles of unrelated tasks, they are hard to test and reuse. If their scope stays focused, teams can compose them into larger systems without losing clarity.

```mermaid
flowchart TD
    ROLE["role/<br/>(e.g. nginx)"]
    TASKS["tasks/<br/>main.yml"]
    HANDLERS["handlers/<br/>main.yml"]
    TEMPLATES["templates/<br/>nginx.conf.j2"]
    FILES["files/<br/>index.html"]
    VARS["vars/<br/>main.yml"]
    DEFAULTS["defaults/<br/>main.yml"]
    META["meta/<br/>main.yml"]

    ROLE --> TASKS
    ROLE --> HANDLERS
    ROLE --> TEMPLATES
    ROLE --> FILES
    ROLE --> VARS
    ROLE --> DEFAULTS
    ROLE --> META
```

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

Handlers exist to keep automation both efficient and safe. In configuration management, changing a file is usually not the risky part; the risky part is restarting or reloading a service at the wrong time, too often, or without validation. Handlers solve that by making service reactions event-driven. If nothing changed, no restart happens. If five tasks all change related files, the restart still happens only once.

That behavior reduces unnecessary churn and makes runs easier to trust in production. It also encourages a better mental model: tasks declare state changes, and handlers declare the controlled reactions to those changes. Separating those concerns is one of the reasons mature Ansible code stays readable as it grows.

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

Secrets management is where many otherwise clean automation projects become dangerous. SSH keys, API tokens, database passwords, and TLS material inevitably need to flow through automation, but they should never live as plain text in Git or be copied casually between engineers. Ansible Vault is not a complete enterprise secrets platform, but it is an important baseline control that lets teams keep automation versioned without exposing sensitive values everywhere.

The main habit to develop here is separation of structure and secret content. Your playbooks should show how secrets are used without revealing the values themselves. That makes reviews safer, reduces accidental leakage, and gives teams a path toward integrating external secret managers later if the environment grows more regulated.

```mermaid
flowchart LR
    PT["plaintext secret<br/>(e.g. db_password)"]
    ENC["ansible-vault encrypt"]
    BLOB["encrypted blob<br/>(stored in Git)"]
    RUN["ansible-playbook<br/>(with vault password)"]
    DEC["decrypted at runtime<br/>(in memory only)"]
    TASK["passed to task<br/>(template / module)"]

    PT --> ENC --> BLOB
    BLOB --> RUN
    RUN --> DEC
    DEC --> TASK
```

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

Idempotency is one of Ansible's most important promises: you should be able to rerun automation without making unnecessary changes or leaving the system in a worse state. That matters because real operations are full of retries. Networks flap, packages mirror slowly, services take longer than expected to start, and engineers rerun jobs during incident response. Automation that only works once is not automation you can trust.

The `changed_when` and `failed_when` directives are the tools for encoding domain knowledge into Ansible. A `command` or `shell` task reports `changed` every time it runs because Ansible cannot know if running a shell script actually changed anything. Adding `changed_when: false` tells Ansible to always report `ok` regardless of the shell's output — appropriate for read-only checks. More precisely, `changed_when: "'updated' in result.stdout"` tells Ansible to report `changed` only when the script's output contains the word "updated," making the task honest about when it actually modified state. A task that always reports `changed` is misleading because it triggers handlers unnecessarily and makes audit logs harder to read.

`failed_when` applies the same principle to failure detection. A command that exits with code 1 when a service is not running might not be a real failure — it might just mean you need to start the service. `failed_when: result.rc != 0 and 'not found' not in result.stderr` lets you express exactly when you consider the task failed, rather than accepting whatever the exit code means. These directives are how you build automation that tells the truth about what happened and responds to conditions rather than blindly following the script.

Error handling builds on that trust. Good playbooks assume that some steps may fail and define what should happen next: retry, skip, rescue, notify, or abort. The goal is not to hide failure. The goal is to make failure behavior intentional and observable instead of surprising.

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

Dynamic inventory becomes necessary when your infrastructure stops being static enough for hand-maintained host files. Autoscaling groups, ephemeral instances, blue-green environments, and multi-region deployments all create churn that static inventory struggles to represent accurately. In those environments, the safest source of truth is often the cloud control plane itself.

This shift is important because it changes how you think about host targeting. Instead of managing named machines manually, you start targeting groups derived from tags, regions, roles, or other metadata. That approach is usually more resilient, but only if your cloud tagging discipline is strong. Poor tags produce poor inventory just as quickly as poor static files do.

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

This section matters because operational maturity eventually requires more than local CLI execution. Once multiple teams share playbooks, credentials, approval flows, and maintenance windows, the problem is no longer just "can Ansible run this task?" It becomes "who is allowed to run it, against which inventory, with which secrets, and where is the audit trail?" AWX and AAP answer those governance questions.

They also change how automation fits into the wider platform. Instead of every engineer running playbooks from a laptop, automation becomes a managed service with projects, inventories, job templates, schedules, and API-driven execution. That model is often the bridge between ad hoc operations and standardized platform engineering.

```mermaid
flowchart TD
    subgraph "AWX Control Layer"
        UI["Web UI<br/>(browser-based dashboard)"]
        API["REST API<br/>(workflow trigger endpoint)"]
        TE["Task Engine<br/>(job dispatcher)"]
        CREDS["Credentials Store<br/>(encrypted vault)"]
        INVSYNC["Inventory Sync<br/>(cloud provider APIs)"]
    end
    subgraph "Execution Layer"
        WORKER["Worker<br/>(Execution Environment container)"]
    end
    subgraph "Managed Infrastructure"
        H1["Managed Host A"]
        H2["Managed Host B"]
        H3["Managed Host C"]
    end

    UI --> TE
    API --> TE
    CREDS --> TE
    INVSYNC --> TE
    TE --> WORKER
    WORKER -->|"SSH"| H1
    WORKER -->|"SSH"| H2
    WORKER -->|"SSH"| H3
```

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

## Molecule: Testing Ansible Roles

Molecule is the standard testing framework for Ansible roles. It creates isolated test environments (containers or VMs), runs your role, and verifies the outcome with a testing framework (pytest + testinfra).

### Molecule Directory Structure

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── defaults/
    │   └── main.yml
    └── molecule/
        └── default/
            ├── molecule.yml       # platform, driver, provisioner config
            ├── converge.yml       # playbook that applies the role
            ├── verify.yml         # tests that verify the role worked
            └── prepare.yml        # optional pre-role setup
```

### `molecule.yml` Configuration

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy

driver:
  name: docker

platforms:
  - name: instance-ubuntu
    image: geerlingguy/docker-ubuntu2204-ansible:latest
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host

  - name: instance-rhel9
    image: geerlingguy/docker-rockylinux9-ansible:latest
    pre_build_image: true
    privileged: true
    command: /usr/lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host

provisioner:
  name: ansible
  playbooks:
    prepare: prepare.yml
    converge: converge.yml
    verify: verify.yml

verifier:
  name: ansible
```

Testing against multiple platforms (Ubuntu + RHEL/Rocky Linux) catches platform-specific bugs before they reach production.

### `converge.yml`

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true

  vars:
    nginx_worker_processes: 2
    nginx_listen_port: 80

  roles:
    - role: nginx
```

### `verify.yml` with Testinfra

```yaml
# molecule/default/verify.yml (ansible-based)
---
- name: Verify
  hosts: all
  become: true
  tasks:
    - name: Check nginx service is running
      ansible.builtin.service_facts:

    - name: Assert nginx is active
      ansible.builtin.assert:
        that: "'nginx' in services and services['nginx'].state == 'running'"
        fail_msg: "nginx service is not running"

    - name: Check nginx is listening on port 80
      ansible.builtin.wait_for:
        port: 80
        timeout: 5

    - name: Check nginx config syntax
      ansible.builtin.command: nginx -t
      changed_when: false

    - name: Verify response from nginx
      ansible.builtin.uri:
        url: http://localhost:80/
        status_code: 200
```

### Running Molecule

```bash
# Full test cycle: create → prepare → converge → verify → destroy
molecule test

# Iterative development — keep instances running between runs
molecule converge          # apply role
molecule verify            # run tests
molecule login             # SSH into instance for debugging
molecule destroy           # clean up

# Test against a specific platform
molecule converge -s scenario_name

# Run all scenarios
molecule test --all
```

Integrate molecule into CI:

```yaml
# .github/workflows/molecule.yml
name: Molecule Tests
on: [push, pull_request]

jobs:
  molecule:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install molecule molecule-plugins[docker] ansible pytest testinfra
      - run: molecule test
        working-directory: roles/nginx
```

[↑ Back to TOC](#table-of-contents)

---

## ansible-lint: Code Quality for Playbooks

`ansible-lint` checks playbooks and roles for common mistakes, deprecated syntax, and style violations.

```bash
# Install
pip install ansible-lint

# Lint a playbook
ansible-lint site.yml

# Lint all YAML files in current directory
ansible-lint

# Output in a specific format
ansible-lint --format rich          # coloured terminal output
ansible-lint --format json > lint-results.json  # CI/CD integration

# List all available rules
ansible-lint --list-rules
```

### `.ansible-lint` Configuration

```yaml
# .ansible-lint
warn_list:
  - yaml[line-length]      # warn but don't fail on long lines

skip_list:
  - role-name              # skip if your role names don't match convention

exclude_paths:
  - .cache/
  - molecule/

# Enforce specific profile
profile: production        # basic, moderate, safety, shared, production
```

### Common ansible-lint Rules

| Rule | Explanation |
|------|-------------|
| `yaml[truthy]` | Use `true`/`false` not `yes`/`no` |
| `name[missing]` | Every task must have a `name` |
| `no-free-form` | Avoid free-form module invocations |
| `command-instead-of-module` | Use `ansible.builtin.copy` not `command: cp` |
| `risky-file-permissions` | File tasks should specify `mode` |
| `no-changed-when` | `command`/`shell` tasks need `changed_when` |
| `deprecated-bare-vars` | Quote variables in templates |

### Pre-commit Integration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/ansible/ansible-lint
    rev: v24.2.0
    hooks:
      - id: ansible-lint
        args: [--profile=moderate]
```

[↑ Back to TOC](#table-of-contents)

---

## Ansible Collections

Collections are the modern packaging format for Ansible content — modules, plugins, roles, and playbooks bundled together and distributed via Ansible Galaxy or private Automation Hub.

### Installing Collections

```bash
# Install from Galaxy
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws

# Install from requirements file
cat > requirements.yml << 'EOF'
---
collections:
  - name: community.general
    version: ">=9.0.0"
  - name: amazon.aws
    version: ">=8.0.0"
  - name: community.mysql
    version: ">=3.9.0"
  - name: kubernetes.core
    version: ">=3.1.0"
EOF

ansible-galaxy collection install -r requirements.yml
```

### Using Collection Modules

The fully qualified collection name (FQCN) is the recommended way to reference modules, as it avoids ambiguity:

```yaml
---
- name: Deploy application
  hosts: app_servers
  tasks:
    - name: Install packages with dnf
      ansible.builtin.dnf:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - python3

    - name: Start and enable nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true

    - name: Add user to group
      ansible.builtin.user:
        name: app
        groups: nginx
        append: true

    - name: Template nginx config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
      notify: Reload nginx

    - name: Manage AWS EC2 instance (amazon.aws collection)
      amazon.aws.ec2_instance:
        name: my-instance
        state: running
        region: us-east-1
        instance_type: t3.micro
        image_id: "{{ latest_ami }}"
```

### Building a Private Collection

```
my_company/
└── infra/                    # collection: my_company.infra
    ├── galaxy.yml            # collection metadata
    ├── README.md
    ├── plugins/
    │   ├── modules/
    │   │   └── custom_module.py
    │   └── filter/
    │       └── custom_filter.py
    └── roles/
        ├── nginx/
        └── app_deploy/
```

```yaml
# galaxy.yml
namespace: my_company
name: infra
version: 1.3.0
description: Internal infrastructure automation roles and modules
dependencies:
  ansible.builtin: "*"
  community.general: ">=9.0.0"
```

```bash
# Build and publish to private Automation Hub
ansible-galaxy collection build
ansible-galaxy collection publish my_company-infra-1.3.0.tar.gz \
  --server https://automation.mycompany.io/api/galaxy/
```

[↑ Back to TOC](#table-of-contents)

---

## Performance Optimisation

Large Ansible runs against hundreds of hosts can be slow. Here are the primary techniques to speed them up.

### Fact Caching

By default, Ansible gathers facts from every host at the start of every play. Caching facts avoids redundant SSH connections for subsequent runs:

```ini
# ansible.cfg
[defaults]
gathering = smart
fact_caching = redis
fact_caching_connection = localhost:6379
fact_caching_timeout = 86400   # cache for 24 hours
```

With `gathering = smart`, Ansible only gathers facts if the cache is empty or expired.

### Mitogen: 10x Speedup

Mitogen replaces Ansible's SSH/Python execution model with a persistent process tree, dramatically reducing per-task overhead. It is particularly effective when you have hundreds of tasks or many hosts.

```ini
# ansible.cfg
[defaults]
strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

Benchmarks show Mitogen consistently delivers 3–10x speedup for typical playbooks, with larger gains for playbooks with many short tasks.

### Parallel Execution

```ini
# ansible.cfg
[defaults]
forks = 20      # default is 5 — increase for large inventories
```

Ansible processes `forks` hosts simultaneously. Increase this when your control node has enough CPU and your network can handle the concurrent SSH connections. A common production value is 20–50 for inventories of hundreds of hosts.

### `async` Tasks for Long-Running Operations

```yaml
- name: Run long database migration
  ansible.builtin.command: python manage.py migrate
  async: 600        # allow up to 10 minutes
  poll: 0           # fire-and-forget, check later
  register: migration_job

- name: Wait for migration to complete
  ansible.builtin.async_status:
    jid: "{{ migration_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 10
```

`async` tasks run in the background on the remote host. This frees the Ansible controller to move on to other hosts, then poll for completion. Use it for package installs, database migrations, or any operation that takes more than a few seconds.

### `serial` Batching for Rolling Deployments

```yaml
- name: Deploy application
  hosts: app_servers
  serial: "25%"      # deploy to 25% of hosts at a time
  # serial: 3        # or a fixed count
  # serial: [1, 5, "100%"]  # staged: 1 first, then 5, then all remaining

  tasks:
    - name: Drain from load balancer
      # ...
    - name: Update application
      # ...
    - name: Re-add to load balancer
      # ...
    - name: Smoke test
      # ...
```

`serial` deployment gives you a rolling update with a circuit breaker: if the smoke test fails on the first batch, the play stops before affecting the rest of the fleet.

[↑ Back to TOC](#table-of-contents)

---

## AWX and Ansible Automation Platform

AWX is the open-source upstream for Red Hat's Ansible Automation Platform (AAP). It provides a web UI, REST API, role-based access control, credential management, and job scheduling for Ansible.

### Key AWX Concepts

**Projects:** A link to a Git repository containing your playbooks and roles. AWX syncs the repo on demand or on schedule.

**Inventories:** Can be static (YAML/INI files in the project repo) or dynamic (sourced from AWS EC2, Azure, VMware, etc.). Dynamic inventory sources update automatically.

**Credentials:** AWX encrypts and stores SSH keys, vault passwords, cloud credentials, and API tokens. Jobs receive credentials at runtime — operators never see the raw values.

**Job templates:** A combination of project, playbook, inventory, and credentials. Users with limited RBAC permissions can launch a job template without seeing the underlying credentials or being able to modify the playbook.

**Workflows:** String multiple job templates together with success/failure branching. Use workflows for multi-stage deployments (deploy infra → configure → smoke test → notify).

### AWX REST API Usage

```bash
# Launch a job template via API
curl -X POST \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": {"app_version": "1.4.2"}}' \
  https://awx.mycompany.io/api/v2/job_templates/42/launch/

# Check job status
curl -H "Authorization: Bearer ${AWX_TOKEN}" \
  https://awx.mycompany.io/api/v2/jobs/123/

# Get job output
curl -H "Authorization: Bearer ${AWX_TOKEN}" \
  https://awx.mycompany.io/api/v2/jobs/123/stdout/?format=txt
```

### AWX in a GitOps Workflow

1. Developer merges playbook changes to `main` branch
2. GitHub webhook triggers AWX to sync the project
3. AWX job template runs automatically after sync
4. AWX posts job result to Slack channel

This flow means every infrastructure configuration change goes through Git review before AWX applies it — GitOps for configuration management.

[↑ Back to TOC](#table-of-contents)

---

## Event-Driven Ansible (EDA)

Event-Driven Ansible (EDA) enables Ansible to react to events in real time — metric thresholds, webhook payloads, message queue events — without requiring a human to trigger a job.

```yaml
# rulebooks/auto-remediate.yml
---
- name: Auto-remediate high memory usage
  hosts: all

  sources:
    - ansible.eda.alertmanager:
        host: 0.0.0.0
        port: 5000

  rules:
    - name: Restart service on OOM
      condition: event.payload.alerts[0].labels.alertname == "HighMemoryUsage"
      action:
        run_playbook:
          name: playbooks/restart-app.yml
          extra_vars:
            target_host: "{{ event.payload.alerts[0].labels.instance }}"

    - name: Scale out on high CPU
      condition: >
        event.payload.alerts[0].labels.alertname == "HighCPUUsage" and
        event.payload.alerts[0].status == "firing"
      action:
        run_playbook:
          name: playbooks/scale-asg.yml
```

EDA is particularly powerful for auto-remediation: when Alertmanager fires a known alert, EDA can automatically run the documented remediation playbook. This reduces mean time to recovery (MTTR) for common failure modes without human intervention.

[↑ Back to TOC](#table-of-contents)

---

## Common Mistakes & Pitfalls

- **Not testing idempotency.** Run your playbook twice — the second run should report 0 changed tasks. If it does not, find and fix the non-idempotent tasks.
- **Using `command` or `shell` where a module exists.** `command: systemctl restart nginx` should be `ansible.builtin.systemd: state: restarted`. Modules are idempotent; `command`/`shell` are not.
- **Omitting `changed_when` on `command` and `shell`.** Without it, every run reports changed. Add `changed_when: false` for read-only commands, or `changed_when: result.rc != 0` for commands that only change on non-zero exit.
- **Storing secrets in plain text in variables.** Use Ansible Vault or external secret stores (HashiCorp Vault, AWS Secrets Manager via lookup plugins).
- **Ignoring errors with `ignore_errors: true`.** Use `failed_when` to define precisely what constitutes failure. Blanket error ignoring hides real problems.
- **Not setting file permissions explicitly.** Always specify `mode: "0644"` (quoted to prevent YAML octal misinterpretation) on file and template tasks.
- **Using `with_items` instead of `loop`.** `with_items` is deprecated. Use `loop` or `loop` + `loop_control`.
- **Hardcoding inventory groups in playbooks.** Use variables or `group_vars` to decouple playbooks from inventory structure.
- **Running playbooks as root.** Use `become: true` at task level (or block level) rather than running the entire SSH connection as root.
- **Not using `--check` mode before production runs.** `ansible-playbook --check` performs a dry run, showing what would change without making changes.
- **Not using tags.** Add `tags: [deploy, config, packages]` to tasks so you can run a subset of a large playbook.
- **Galaxy roles with no version pinning.** Pin roles and collections in `requirements.yml` to specific versions. Otherwise a `galaxy install -r requirements.yml` can pull a breaking change at any time.
- **Skipping handler notifications.** If a template changes nginx.conf but the `notify: Reload nginx` is missing, the old config stays loaded in memory.
- **Forgetting `become_user` for application tasks.** When managing files or services under a non-root application user, combine `become: true` with `become_user: appuser`.
- **Large playbooks without logical blocks.** Group related tasks in named blocks with shared `become`, `when`, or `tags` to reduce repetition and improve readability.

[↑ Back to TOC](#table-of-contents)

---

## Interview Prep

**Q: What makes a task idempotent, and why does idempotency matter in Ansible?**
A: A task is idempotent if running it multiple times produces the same result as running it once. Ansible modules like `ansible.builtin.copy` and `ansible.builtin.template` check the current state before making changes — if the file already has the correct content and permissions, they do nothing. Idempotency matters because it means you can safely re-run playbooks for configuration drift remediation, new host bootstrapping, and recovery without unintended side effects.

**Q: What is the difference between a role and a playbook?**
A: A playbook is a top-level YAML file that defines which hosts to target and what tasks to run. A role is a reusable, structured directory that encapsulates related tasks, handlers, variables, templates, and files. Playbooks include roles via the `roles` keyword. Roles promote reuse — the same nginx role can be included in a web server playbook and an API server playbook.

**Q: How do you handle secrets in Ansible?**
A: Ansible Vault encrypts variable files or individual variable values. You run `ansible-vault encrypt_string 'password123' --name db_password` to create an encrypted inline variable, or `ansible-vault encrypt vars/secrets.yml` for a whole file. The vault password is provided via `--vault-password-file` or `--ask-vault-pass` at runtime. For production, integrate with an external secret store (HashiCorp Vault, AWS Secrets Manager) using the corresponding lookup plugin.

**Q: What is an Ansible inventory and what types are available?**
A: An inventory defines the hosts and groups Ansible manages. Static inventories are INI or YAML files you maintain manually. Dynamic inventories query an external source (AWS EC2, GCP, VMware) at runtime and return host lists as JSON. You can combine both: use a dynamic source for cloud hosts and a static file for network devices.

**Q: How does Ansible handle task failure and what are your options?**
A: By default, Ansible stops processing a host when a task fails and continues with other hosts. Options: `ignore_errors: true` (proceed despite failure), `failed_when` (custom failure condition), `any_errors_fatal: true` (stop all hosts on first failure), and `max_fail_percentage` (stop if more than N% of hosts fail).

**Q: What is `delegate_to` and when would you use it?**
A: `delegate_to` runs a task on a different host than the current target. Common use cases: deregistering a host from a load balancer before updating it (`delegate_to: localhost` to call an API), running a database migration from a specific app server, or gathering facts about one host to use in tasks on another.

**Q: How does Ansible's execution strategy work?**
A: The default `linear` strategy runs each task across all hosts in batches of `forks` (default 5), waiting for all hosts to complete a task before moving to the next. The `free` strategy lets each host advance to the next task as soon as it completes the current one, without waiting for other hosts. Mitogen provides a high-performance drop-in replacement.

**Q: What is the difference between `vars`, `vars_files`, `defaults`, and `group_vars`?**
A: `vars` in a task or play has highest precedence — it overrides everything. `vars_files` loads variable files into the play. `defaults/main.yml` in a role has lowest precedence — easily overridden by inventory or extra vars. `group_vars/` files apply variables to all hosts in a group and are loaded automatically from the inventory directory.

**Q: How do you test an Ansible role before applying it to production?**
A: Use Molecule. It creates isolated test environments (containers or VMs), runs the role, and verifies the outcome with testinfra or ansible-based verification tasks. Also use `ansible-lint` for static analysis and `--check` mode for dry runs against real hosts.

**Q: What is Ansible Galaxy and how do you use it?**
A: Ansible Galaxy is the public repository for community roles and collections at galaxy.ansible.com. You install roles with `ansible-galaxy role install geerlingguy.nginx` or collections with `ansible-galaxy collection install community.general`. For production, always pin versions in a `requirements.yml` and commit the file to version control.

**Q: What is a handler and when does it run?**
A: A handler is a task that only runs when notified by another task's change. Handlers run at the end of a play, after all regular tasks complete. Multiple notifications of the same handler result in it running only once. Use handlers for service restarts triggered by config file changes — this avoids unnecessary restarts if the config did not actually change.

**Q: How would you migrate from shell scripts to Ansible for server configuration?**
A: Map each shell command to the equivalent Ansible module (e.g., `yum install` → `ansible.builtin.dnf`, `systemctl` → `ansible.builtin.systemd`, `cp` → `ansible.builtin.copy`). Add idempotency by using state-checking modules. Group related tasks into roles. Test with Molecule. The key mindset shift is describing desired state (package X should be installed) rather than actions (install package X).

[↑ Back to TOC](#table-of-contents)

---

## A Day in the Life: Automation Engineer

It is 9:00 AM and the first thing you check is the AWX dashboard. Three scheduled jobs ran overnight: the system patching playbook (succeeded on all 247 hosts), the certificate rotation playbook (failed on two hosts — needs investigation), and the configuration compliance scan (98.4% pass rate, 4 hosts drifted).

You click into the certificate rotation failures. The error is clear: `TASK [Fetch certificate] ... Permission denied`. Two application servers have a tightened sudo policy that was deployed last week — the deploy user can no longer read from `/etc/ssl/private/`. You check the change management system and confirm: this was intentional (a security hardening change), but nobody updated the certificate rotation role to handle it. You open a ticket and assign it to yourself.

By 10 AM you have the fix: add a `become_user: root` override just for the certificate fetch task, wrapping it in a block with appropriate error handling. You test the change with Molecule — creating a scenario that simulates the restricted sudo policy — and verify both the original and corrected paths work. The linter and molecule test pass in under three minutes.

At 11 AM you PR the role fix. While it is in review, you turn to the compliance drift. Four hosts have a modified `/etc/ssh/sshd_config` — specifically, `PermitRootLogin` changed from `no` to `yes`. CloudTrail and auth logs confirm these were manual changes made by a third-party vendor during a midnight maintenance window. You run the compliance playbook in `--check` mode against those four hosts to confirm what would change, share the diff with the vendor for sign-off, then run it in normal mode. The hosts are back in compliance.

After lunch, a developer asks if Ansible can automatically deploy their new microservice when the Docker image is pushed to ECR. You show them how to chain an EDA rulebook (listening on an SNS topic via webhook) to a job template in AWX that runs the deployment playbook. You wire it up in the dev environment — they push a test image, the EDA rulebook fires within three seconds, AWX launches the job, and the new container is running two minutes later. The developer is delighted.

End of day: the certificate rotation role is fixed and merged, four drifted hosts are remediated, and a new EDA-driven deployment pipeline is live in dev. The morning's failures became afternoon's improvements.

[↑ Back to TOC](#table-of-contents)

---

## Windows Automation with Ansible

Ansible manages Windows hosts using WinRM (Windows Remote Management) or SSH. WinRM is the traditional approach; SSH is increasingly preferred because it uses the same transport as Linux automation.

### Configuring WinRM on Windows

```powershell
# Run on the Windows host to enable WinRM
# Download and run the ConfigureRemotingForAnsible.ps1 script:
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"
(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
powershell.exe -ExecutionPolicy ByPass -File $file
```

Ansible inventory for Windows hosts:

```ini
[windows]
win-server-01 ansible_host=192.168.1.10
win-server-02 ansible_host=192.168.1.11

[windows:vars]
ansible_user=administrator
ansible_password="{{ vault_win_admin_password }}"
ansible_connection=winrm
ansible_winrm_scheme=https
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5986
```

### Windows Modules

Ansible provides `ansible.windows.*` and `community.windows.*` collections with modules mirroring the Linux equivalents:

```yaml
---
- name: Configure Windows web server
  hosts: windows
  tasks:
    - name: Install IIS
      ansible.windows.win_feature:
        name: Web-Server
        state: present
        include_management_tools: true

    - name: Copy web content
      ansible.windows.win_copy:
        src: files/index.html
        dest: C:\inetpub\wwwroot\index.html

    - name: Set registry value
      ansible.windows.win_regedit:
        path: HKLM:\SOFTWARE\MyApp
        name: Version
        data: "1.4.2"
        type: String

    - name: Run PowerShell script
      ansible.windows.win_shell: |
        Import-Module WebAdministration
        Set-WebConfigurationProperty -filter /system.webServer/directoryBrowse `
          -name enabled -value True
      changed_when: false

    - name: Manage Windows service
      ansible.windows.win_service:
        name: W3SVC
        state: started
        start_mode: auto

    - name: Create local user
      ansible.windows.win_user:
        name: appservice
        password: "{{ vault_app_service_password }}"
        password_never_expires: true
        groups:
          - IIS_IUSRS
```

### Windows-Specific Vault Usage

Windows passwords and service credentials must be vaulted. Use a dedicated vault file for Windows credentials:

```bash
# Encrypt the entire windows credentials file
ansible-vault encrypt vars/windows_credentials.yml

# Reference in playbook
- name: Include Windows credentials
  ansible.builtin.include_vars: vars/windows_credentials.yml
  no_log: true
```

[↑ Back to TOC](#table-of-contents)

---

## Network Device Automation

Ansible manages network devices (Cisco IOS, Juniper JunOS, Arista EOS, F5, Palo Alto) using connection plugins designed for CLIs and APIs rather than SSH shells.

### Cisco IOS Example

```yaml
---
- name: Configure Cisco IOS switches
  hosts: ios_switches
  gather_facts: false

  vars:
    ansible_network_os: cisco.ios.ios
    ansible_connection: ansible.netcommon.network_cli
    ansible_user: admin
    ansible_password: "{{ vault_cisco_password }}"
    ansible_become: true
    ansible_become_method: enable
    ansible_become_password: "{{ vault_cisco_enable_password }}"

  tasks:
    - name: Gather IOS facts
      cisco.ios.ios_facts:
        gather_subset: all

    - name: Configure hostname
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ inventory_hostname }}"

    - name: Configure VLANs
      cisco.ios.ios_vlans:
        config:
          - name: Production
            vlan_id: 100
            state: active
          - name: Management
            vlan_id: 999
            state: active
        state: merged

    - name: Configure interface
      cisco.ios.ios_interfaces:
        config:
          - name: GigabitEthernet0/1
            description: "Uplink to core"
            enabled: true
        state: merged

    - name: Save running config to startup
      cisco.ios.ios_config:
        save_when: modified
```

### Network Automation Patterns

**Configuration backup:** Run daily to store device configs in Git:

```yaml
- name: Backup network configs
  hosts: all_network
  gather_facts: false
  tasks:
    - name: Get running config
      ansible.netcommon.cli_command:
        command: show running-config
      register: running_config

    - name: Save config to file
      ansible.builtin.copy:
        content: "{{ running_config.stdout }}"
        dest: "backups/{{ inventory_hostname }}-{{ ansible_date_time.date }}.conf"
      delegate_to: localhost

    - name: Commit backup to git
      ansible.builtin.shell: |
        cd backups
        git add .
        git diff --cached --quiet || git commit -m "Config backup {{ inventory_hostname }} {{ ansible_date_time.date }}"
      delegate_to: localhost
      run_once: true
```

**Compliance checking without making changes:**

```yaml
- name: Audit SSH configuration
  hosts: ios_switches
  gather_facts: false
  tasks:
    - name: Check SSH version 2
      cisco.ios.ios_facts:
        gather_subset: ['min']

    - name: Assert SSH v2 enabled
      ansible.builtin.assert:
        that: "'ssh version 2' in ansible_net_config"
        fail_msg: "{{ inventory_hostname }} does not have SSH v2 enabled"
        success_msg: "{{ inventory_hostname }} SSH v2 OK"
      ignore_errors: true
```

[↑ Back to TOC](#table-of-contents)

---

## Custom Modules and Plugins

When existing Ansible modules do not cover your use case, you can write custom modules in Python (or any language that outputs JSON).

### Custom Module Structure

```python
#!/usr/bin/python
# library/myapp_deploy.py

from ansible.module_utils.basic import AnsibleModule
import requests

DOCUMENTATION = '''
---
module: myapp_deploy
short_description: Deploy application via MyApp API
description:
  - Triggers a deployment in the MyApp internal deployment system
options:
  service:
    description: Name of the service to deploy
    required: true
    type: str
  version:
    description: Version tag to deploy
    required: true
    type: str
  api_url:
    description: MyApp API base URL
    default: http://deploy-api.mycompany.io
    type: str
  api_token:
    description: API authentication token
    required: true
    type: str
    no_log: true
'''

EXAMPLES = '''
- name: Deploy payment service
  myapp_deploy:
    service: payment-api
    version: "1.4.2"
    api_token: "{{ vault_deploy_api_token }}"
'''


def run_module():
    module_args = dict(
        service=dict(type='str', required=True),
        version=dict(type='str', required=True),
        api_url=dict(type='str', default='http://deploy-api.mycompany.io'),
        api_token=dict(type='str', required=True, no_log=True),
    )

    result = dict(changed=False, deployment_id='')
    module = AnsibleModule(argument_spec=module_args, supports_check_mode=True)

    if module.check_mode:
        module.exit_json(**result)

    # Check current deployed version
    headers = {'Authorization': f'Bearer {module.params["api_token"]}'}
    current = requests.get(
        f'{module.params["api_url"]}/services/{module.params["service"]}',
        headers=headers,
        timeout=10
    ).json()

    if current.get('version') == module.params['version']:
        module.exit_json(**result)   # already at desired version, no change

    # Trigger deployment
    resp = requests.post(
        f'{module.params["api_url"]}/deployments',
        json={'service': module.params['service'], 'version': module.params['version']},
        headers=headers,
        timeout=30
    )
    resp.raise_for_status()

    result['changed'] = True
    result['deployment_id'] = resp.json().get('id')
    module.exit_json(**result)


def main():
    run_module()


if __name__ == '__main__':
    main()
```

Place the module in a `library/` directory adjacent to your playbook, or inside a collection's `plugins/modules/` directory. Call it like any built-in module:

```yaml
- name: Deploy payment service
  myapp_deploy:
    service: payment-api
    version: "{{ release_version }}"
    api_token: "{{ vault_deploy_api_token }}"
  register: deploy_result
```

### Custom Filter Plugins

```python
# filter_plugins/myapp_filters.py

def cidr_to_netmask(cidr):
    """Convert CIDR notation to dotted netmask."""
    bits = 0xffffffff ^ (1 << (32 - int(cidr))) - 1
    return '.'.join(str((bits >> (i * 8)) & 0xff) for i in reversed(range(4)))


def version_major(version_string):
    """Extract major version from semantic version string."""
    return int(version_string.split('.')[0])


class FilterModule:
    def filters(self):
        return {
            'cidr_to_netmask': cidr_to_netmask,
            'version_major': version_major,
        }
```

Usage in templates or tasks:

```yaml
- name: Configure network
  ansible.builtin.template:
    src: interface.j2
    dest: /etc/network/interfaces
  vars:
    netmask: "{{ '10.0.0.0/24' | cidr_to_netmask }}"  # "255.255.255.0"
```

[↑ Back to TOC](#table-of-contents)

---

## Ansible in CI/CD Pipelines

Ansible is frequently used in CI/CD to configure environments, run post-deployment configuration, or run smoke tests after deployment.

### GitHub Actions Example

```yaml
# .github/workflows/deploy.yml
name: Deploy Application
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python and Ansible
        uses: actions/setup-python@v5
        with: { python-version: "3.12" }

      - run: pip install ansible boto3

      - name: Install requirements
        run: ansible-galaxy collection install -r requirements.yml

      - name: Write vault password
        run: echo "$ANSIBLE_VAULT_PASSWORD" > .vault_password
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}

      - name: Write SSH key
        run: |
          echo "$SSH_PRIVATE_KEY" > /tmp/deploy_key
          chmod 600 /tmp/deploy_key
        env:
          SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_SSH_KEY }}

      - name: Run deployment playbook
        env:
          ANSIBLE_HOST_KEY_CHECKING: "False"
        run: |
          ansible-playbook \
            -i inventories/production \
            --vault-password-file .vault_password \
            --private-key /tmp/deploy_key \
            --extra-vars "app_version=${{ github.sha }}" \
            playbooks/deploy-app.yml

      - name: Clean up secrets
        if: always()
        run: |
          rm -f .vault_password /tmp/deploy_key
```

### Ansible as a Post-Deployment Configurator

A common pattern is to use Terraform to provision infrastructure and Ansible to configure it:

```bash
# 1. Terraform provisions the EC2 instance
terraform apply -var "app_version=${VERSION}"

# 2. Extract newly created instance IPs
INSTANCE_IPS=$(terraform output -json app_server_ips | jq -r '.[]')

# 3. Generate Ansible inventory dynamically
cat > /tmp/new-hosts.ini << EOF
[new_app_servers]
$(echo "$INSTANCE_IPS" | tr '\n' '\n')

[new_app_servers:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=/tmp/deploy_key
EOF

# 4. Wait for SSH to become available
until ansible -i /tmp/new-hosts.ini new_app_servers -m ping --timeout=5 2>/dev/null; do
  echo "Waiting for SSH..."
  sleep 10
done

# 5. Run configuration playbook
ansible-playbook \
  -i /tmp/new-hosts.ini \
  --extra-vars "app_version=${VERSION}" \
  playbooks/configure-app-server.yml
```

[↑ Back to TOC](#table-of-contents)

---

## Ansible Vault Deep Dive

Ansible Vault encrypts sensitive data using AES256. It supports encrypting whole files, individual variables, or strings inline.

### Vault ID: Multiple Passwords

Vault IDs let you use different passwords for different sensitivity levels — for example, one password for dev secrets and a separate one for production secrets:

```bash
# Encrypt with a vault ID label
ansible-vault encrypt_string 'prod_db_password' \
  --vault-id production@prompt \
  --name db_password

# Encrypt dev variable with a different vault ID
ansible-vault encrypt_string 'dev_db_password' \
  --vault-id dev@prompt \
  --name db_password

# Run playbook providing both vault passwords
ansible-playbook site.yml \
  --vault-id production@prod-password-file \
  --vault-id dev@dev-password-file
```

### Encrypted Variable Files

```bash
# Create encrypted vars file
ansible-vault create vars/production-secrets.yml

# Edit encrypted file
ansible-vault edit vars/production-secrets.yml

# View without decrypting to disk
ansible-vault view vars/production-secrets.yml

# Re-encrypt with a new password
ansible-vault rekey vars/production-secrets.yml

# Decrypt to plain text (do not commit the result!)
ansible-vault decrypt vars/production-secrets.yml
```

### External Vault Integration (HashiCorp Vault)

```yaml
# In a task — lookup secrets from HashiCorp Vault at runtime
- name: Configure database connection
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
  vars:
    db_password: "{{ lookup('community.hashi_vault.hashi_vault',
                             'secret/data/production/database:password
                              url=https://vault.mycompany.io:8200
                              auth_method=approle
                              role_id={{ vault_role_id }}
                              secret_id={{ vault_secret_id }}') }}"
  no_log: true
```

This pattern retrieves secrets from HashiCorp Vault at runtime — secrets are never stored in Ansible Vault or in the repository. The `no_log: true` prevents the secret from appearing in output even if the task fails.

[↑ Back to TOC](#table-of-contents)

---

## `ansible-pull` for Node Self-Configuration

`ansible-pull` inverts the normal push model: each node pulls its configuration from a Git repository and applies it locally. This scales infinitely — the Ansible control node has nothing to do, and each host manages itself.

```bash
# On each managed host, run as a cron job or systemd timer:
ansible-pull \
  --url https://github.com/mycompany/ansible-configs.git \
  --directory /opt/ansible-configs \
  --inventory localhost, \
  --playbook local.yml \
  --vault-password-file /etc/ansible/vault-password \
  --extra-vars "env=$(hostname | cut -d- -f1)"
```

`local.yml` is a playbook that configures the local host based on group membership determined by hostname patterns.

### Systemd Timer for ansible-pull

```ini
# /etc/systemd/system/ansible-pull.service
[Unit]
Description=Ansible Pull Configuration
After=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/ansible-pull \
  --url https://github.com/mycompany/ansible-configs.git \
  --directory /opt/ansible-configs \
  --inventory localhost, \
  --playbook local.yml \
  --vault-password-file /etc/ansible/vault-password
StandardOutput=journal
StandardError=journal
```

```ini
# /etc/systemd/system/ansible-pull.timer
[Unit]
Description=Run ansible-pull every 30 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=30min
RandomizedDelaySec=2min    # spread load across hosts

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now ansible-pull.timer
journalctl -u ansible-pull.service -f
```

The `RandomizedDelaySec=2min` spreads the runs across the fleet to avoid all hosts pulling from GitHub simultaneously.

[↑ Back to TOC](#table-of-contents)

---

## Jinja2 Templating Deep Dive

Ansible uses Jinja2 for variable interpolation, conditional logic, and template files. Understanding Jinja2 unlocks much more powerful playbooks and templates.

### Filters

```yaml
# String manipulation
"{{ 'Hello World' | lower }}"          # "hello world"
"{{ 'hello' | upper }}"                # "HELLO"
"{{ '  hello  ' | trim }}"            # "hello"
"{{ 'my-app-name' | replace('-', '_') }}"  # "my_app_name"

# List/dict operations
"{{ [3, 1, 2] | sort }}"              # [1, 2, 3]
"{{ ['a', 'b', 'a'] | unique }}"      # ['a', 'b']
"{{ [1, 2, 3] | join(', ') }}"        # "1, 2, 3"
"{{ {'a': 1, 'b': 2} | dict2items }}" # [{'key': 'a', 'value': 1}, ...]
"{{ items | items2dict }}"            # reverse of dict2items

# Type conversion and defaults
"{{ some_var | default('fallback') }}"        # use fallback if undefined
"{{ some_var | default(omit) }}"              # omit parameter if undefined
"{{ '3.14' | float }}"                        # 3.14
"{{ 42 | string }}"                           # "42"
"{{ some_list | length }}"                    # count items

# Path manipulation
"{{ '/etc/nginx/nginx.conf' | basename }}"    # "nginx.conf"
"{{ '/etc/nginx/nginx.conf' | dirname }}"     # "/etc/nginx"
```

### Conditional Template Logic

```jinja2
{# nginx.conf.j2 #}
worker_processes {{ nginx_worker_processes | default(ansible_processor_vcpus) }};

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
}

http {
    {% if nginx_ssl_enabled | default(false) %}
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    {% endif %}

    {% for vhost in nginx_vhosts %}
    server {
        listen {{ vhost.port | default(80) }};
        server_name {{ vhost.name }};
        root {{ vhost.root }};

        {% if vhost.php | default(false) %}
        location ~ \.php$ {
            fastcgi_pass unix:/run/php-fpm/www.sock;
            fastcgi_index index.php;
            include fastcgi_params;
        }
        {% endif %}
    }
    {% endfor %}
}
```

### Lookup Plugins

```yaml
# Read a file
private_key: "{{ lookup('ansible.builtin.file', '~/.ssh/id_rsa') }}"

# Environment variable
db_host: "{{ lookup('ansible.builtin.env', 'DATABASE_HOST') }}"

# AWS SSM Parameter Store
api_key: "{{ lookup('amazon.aws.aws_ssm', '/production/api_key', region='us-east-1') }}"

# HashiCorp Vault
db_pass: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/db:password') }}"

# Password — generate and cache
admin_pass: "{{ lookup('ansible.builtin.password', '/tmp/admin_password length=24 chars=ascii_letters,digits') }}"
```

[↑ Back to TOC](#table-of-contents)

---

## Dynamic Inventory Deep Dive

Dynamic inventories query an external source at run time and return a JSON structure that Ansible parses as an inventory. This eliminates manual maintenance of host lists.

### AWS EC2 Dynamic Inventory

The `amazon.aws.aws_ec2` inventory plugin discovers EC2 instances based on tags, regions, and filters:

```yaml
# inventories/production/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - eu-west-1

filters:
  instance-state-name: running
  tag:Environment: production

keyed_groups:
  - key: tags.Role
    prefix: role
    separator: "_"
  - key: placement.region
    prefix: region
    separator: "_"
  - key: instance_type
    prefix: instance_type

hostnames:
  - tag:Name
  - private-ip-address

compose:
  ansible_host: private_ip_address

# Result: groups like role_app_server, role_db_server, region_us_east_1
```

```bash
# Test the dynamic inventory
ansible-inventory -i inventories/production --list --graph

# Output:
# @all:
#   |--@role_app_server:
#   |  |--app-server-01
#   |  |--app-server-02
#   |--@role_db_server:
#   |  |--db-server-01
#   |--@region_us_east_1:
#      |--app-server-01
#      |--db-server-01
```

### Constructed Inventory: Custom Grouping Logic

```yaml
# inventories/production/constructed.yml
plugin: ansible.builtin.constructed

use_extra_vars: true

groups:
  web_servers: "'app' in tags.get('Role', '')"
  database_servers: "'db' in tags.get('Role', '')"
  large_instances: "instance_type in ['m5.2xlarge', 'm5.4xlarge', 'r5.2xlarge']"

compose:
  ansible_python_interpreter: >-
    '/usr/bin/python3.11' if ansible_distribution_major_version == '9'
    else '/usr/bin/python3'
```

### Writing a Custom Inventory Script

For systems not covered by existing plugins, write a script that outputs the inventory JSON format:

```python
#!/usr/bin/env python3
# inventories/cmdb_inventory.py
import argparse
import json
import requests

def get_inventory():
    # Fetch from internal CMDB
    resp = requests.get(
        "https://cmdb.mycompany.io/api/hosts",
        headers={"Authorization": f"Bearer {os.environ['CMDB_TOKEN']}"}
    )
    hosts = resp.json()

    inventory = {"_meta": {"hostvars": {}}}

    for host in hosts:
        group = host["environment"]
        inventory.setdefault(group, {"hosts": [], "vars": {}})
        inventory[group]["hosts"].append(host["hostname"])
        inventory["_meta"]["hostvars"][host["hostname"]] = {
            "ansible_host": host["ip_address"],
            "datacenter": host["datacenter"],
            "os_version": host["os_version"],
        }

    return inventory


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true")
    parser.add_argument("--host")
    args = parser.parse_args()

    if args.list:
        print(json.dumps(get_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_inventory()["_meta"]["hostvars"].get(args.host, {})))
```

```bash
chmod +x inventories/cmdb_inventory.py
ansible-inventory -i inventories/cmdb_inventory.py --list
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced Role Patterns

### Role Dependency Management

```yaml
# roles/wordpress/meta/main.yml
dependencies:
  - role: nginx
    vars:
      nginx_listen_port: 80
  - role: php_fpm
    vars:
      php_version: "8.2"
  - role: mysql
    vars:
      mysql_databases:
        - name: wordpress
          encoding: utf8mb4
```

When `wordpress` is included in a play, Ansible automatically includes `nginx`, `php_fpm`, and `mysql` first. Dependency resolution handles circular checks.

### Role Testing Across Multiple OS Families

Create multiple Molecule scenarios for different operating systems:

```
roles/nginx/molecule/
├── ubuntu22/
│   └── molecule.yml      # tests Ubuntu 22.04
├── rocky9/
│   └── molecule.yml      # tests Rocky Linux 9
└── debian12/
    └── molecule.yml      # tests Debian 12
```

```bash
# Run all scenarios
for scenario in ubuntu22 rocky9 debian12; do
  echo "Testing on: $scenario"
  molecule test -s "$scenario"
done
```

### Role Variable Namespacing

Prefix all role variables with the role name to avoid conflicts:

```yaml
# roles/nginx/defaults/main.yml
nginx_version: "1.24"
nginx_worker_processes: "auto"
nginx_worker_connections: 1024
nginx_log_format: combined
nginx_access_log: /var/log/nginx/access.log
nginx_error_log: /var/log/nginx/error.log
nginx_client_max_body_size: "10m"
nginx_keepalive_timeout: 65
nginx_vhosts: []
nginx_ssl_enabled: false
nginx_ssl_certificate: ""
nginx_ssl_certificate_key: ""
```

This way, `nginx_worker_processes` will never accidentally clash with a similarly-named variable from another role.

[↑ Back to TOC](#table-of-contents)

---

## Ansible and Containers

While Ansible is primarily used for VM and bare-metal configuration, it has integrations for container and Kubernetes workflows.

### Managing Containers with Ansible

```yaml
---
- name: Deploy container
  hosts: docker_hosts
  tasks:
    - name: Pull image
      community.docker.docker_image:
        name: "{{ app_image }}"
        tag: "{{ app_version }}"
        source: pull

    - name: Run container
      community.docker.docker_container:
        name: myapp
        image: "{{ app_image }}:{{ app_version }}"
        state: started
        restart_policy: unless-stopped
        ports:
          - "8080:8080"
        env:
          DATABASE_URL: "{{ db_url }}"
          LOG_LEVEL: info
        volumes:
          - "/var/myapp/data:/app/data"
        healthcheck:
          test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
          interval: 30s
          timeout: 5s
          retries: 3
          start_period: 30s
```

### Kubernetes Manifests via Ansible

```yaml
- name: Apply Kubernetes deployment
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
        namespace: production
      spec:
        replicas: 3
        selector:
          matchLabels: { app: myapp }
        template:
          metadata:
            labels: { app: myapp }
          spec:
            containers:
              - name: myapp
                image: "{{ app_image }}:{{ app_version }}"
                ports: [{ containerPort: 8080 }]
                resources:
                  requests: { cpu: "250m", memory: "256Mi" }
                  limits: { cpu: "500m", memory: "512Mi" }

- name: Wait for rollout
  kubernetes.core.k8s_rollout_status:
    name: myapp
    namespace: production
    kind: Deployment
    wait_timeout: 300
```

[↑ Back to TOC](#table-of-contents)

---

## Debugging Ansible Playbooks

### Verbosity Levels

```bash
ansible-playbook site.yml -v    # show task results
ansible-playbook site.yml -vv   # show input/output
ansible-playbook site.yml -vvv  # show SSH connection details
ansible-playbook site.yml -vvvv # add connection debugging (very verbose)
```

### Debug Module

```yaml
- name: Show variable value
  ansible.builtin.debug:
    var: hostvars[inventory_hostname]

- name: Print formatted message
  ansible.builtin.debug:
    msg: "Deploying {{ app_name }} version {{ app_version }} to {{ inventory_hostname }}"

# Show all facts for the host
- name: Show all facts
  ansible.builtin.debug:
    var: ansible_facts
```

### `--start-at-task` and `--step`

```bash
# Start playbook from a specific task (useful after a partial failure)
ansible-playbook site.yml --start-at-task="Restart nginx"

# Interactive step-through: confirm each task before running
ansible-playbook site.yml --step
```

### `--limit` and `-t` (tags)

```bash
# Run only on a subset of hosts
ansible-playbook site.yml --limit=app-server-01

# Run only tasks with specific tags
ansible-playbook site.yml -t deploy,config

# Run all tasks EXCEPT those with specific tags
ansible-playbook site.yml --skip-tags=slow_tasks
```

### `fail` Module for Conditional Stops

```yaml
- name: Stop if wrong environment
  ansible.builtin.fail:
    msg: "This playbook should not run against production without explicit confirmation"
  when:
    - "'production' in group_names"
    - not confirm_production | default(false) | bool
```

[↑ Back to TOC](#table-of-contents)

---

## Ansible Quick Reference

### Essential Commands

```bash
# Run a playbook
ansible-playbook site.yml -i inventory/

# Run with vault
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Dry run (check mode)
ansible-playbook site.yml --check --diff

# Show diff of changed files
ansible-playbook site.yml --diff

# Limit to specific hosts
ansible-playbook site.yml --limit webservers

# Pass extra variables
ansible-playbook site.yml -e "app_version=1.4.2 deploy_env=staging"
ansible-playbook site.yml -e @vars/extra.yml

# Run ad-hoc command
ansible all -i inventory/ -m ping
ansible webservers -i inventory/ -m command -a "uptime"
ansible all -m ansible.builtin.gather_facts --tree /tmp/facts/

# Inventory inspection
ansible-inventory -i inventory/ --list
ansible-inventory -i inventory/ --graph
ansible-inventory -i inventory/ --host=web-01

# Galaxy
ansible-galaxy role install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role list
ansible-galaxy collection list
```

### Common Module Quick Reference

| Task | Module | Key Args |
|------|--------|----------|
| Install package | `ansible.builtin.dnf` / `apt` | `name`, `state` |
| Copy file | `ansible.builtin.copy` | `src`, `dest`, `mode` |
| Template file | `ansible.builtin.template` | `src`, `dest` |
| Create directory | `ansible.builtin.file` | `path`, `state: directory` |
| Manage service | `ansible.builtin.systemd` | `name`, `state`, `enabled` |
| Run command | `ansible.builtin.command` | `cmd`, `changed_when` |
| Run shell | `ansible.builtin.shell` | `cmd`, `creates` |
| Manage user | `ansible.builtin.user` | `name`, `groups`, `state` |
| Add cron job | `ansible.builtin.cron` | `name`, `job`, `minute`, `hour` |
| Set sysctl | `ansible.posix.sysctl` | `name`, `value`, `state` |
| Mount filesystem | `ansible.posix.mount` | `path`, `src`, `fstype`, `state` |
| Set SELinux | `ansible.posix.selinux` | `state`, `policy` |
| Fetch file from remote | `ansible.builtin.fetch` | `src`, `dest` |
| Assert condition | `ansible.builtin.assert` | `that`, `fail_msg` |
| Pause playbook | `ansible.builtin.pause` | `seconds`, `prompt` |
| Get URL | `ansible.builtin.get_url` | `url`, `dest`, `checksum` |

[↑ Back to TOC](#table-of-contents)

---

## Ansible for Application Deployment Patterns

### Blue-Green Deployment with Ansible

```yaml
---
- name: Blue-Green Deployment
  hosts: localhost
  vars:
    alb_arn: "{{ lookup('env', 'ALB_ARN') }}"
    blue_tg_arn: "{{ lookup('env', 'BLUE_TG_ARN') }}"
    green_tg_arn: "{{ lookup('env', 'GREEN_TG_ARN') }}"
    app_version: "{{ lookup('env', 'APP_VERSION') }}"

  tasks:
    - name: Determine current active environment
      amazon.aws.elb_application_lb_info:
        names: [myapp-prod]
      register: alb_info

    - name: Set active/inactive environments
      ansible.builtin.set_fact:
        active_color: "{{ 'blue' if blue_tg_arn in (alb_info.load_balancers[0].listeners[0].default_actions | map(attribute='target_group_arn') | list) else 'green' }}"
        inactive_color: "{{ 'green' if blue_tg_arn in (alb_info.load_balancers[0].listeners[0].default_actions | map(attribute='target_group_arn') | list) else 'blue' }}"
        inactive_tg_arn: "{{ green_tg_arn if blue_tg_arn in (alb_info.load_balancers[0].listeners[0].default_actions | map(attribute='target_group_arn') | list) else blue_tg_arn }}"

    - name: Deploy to inactive environment
      ansible.builtin.debug:
        msg: "Deploying {{ app_version }} to {{ inactive_color }} environment"

    # Deploy new version to inactive ASG/ECS service here

    - name: Wait for inactive environment health checks
      amazon.aws.elb_target_group_info:
        names: ["myapp-{{ inactive_color }}"]
      register: tg_info
      until: >
        (tg_info.target_groups[0].target_health_descriptions |
         selectattr('target_health.state', 'equalto', 'healthy') |
         list | length) >= 2
      retries: 30
      delay: 15

    - name: Switch traffic to new environment
      community.aws.elb_listener:
        elb_arn: "{{ alb_arn }}"
        protocol: HTTPS
        port: 443
        default_actions:
          - Type: forward
            TargetGroupArn: "{{ inactive_tg_arn }}"
        state: present

    - name: Verify switch successful
      ansible.builtin.uri:
        url: https://myapp.example.com/health
        status_code: 200
      register: health_check
      retries: 5
      delay: 10
```

### Canary Deployment with Weighted Routing

```yaml
- name: Canary deploy - shift 10% traffic
  community.aws.elb_listener:
    elb_arn: "{{ alb_arn }}"
    protocol: HTTPS
    port: 443
    default_actions:
      - Type: forward
        ForwardConfig:
          TargetGroups:
            - TargetGroupArn: "{{ stable_tg_arn }}"
              Weight: 9
            - TargetGroupArn: "{{ canary_tg_arn }}"
              Weight: 1
    state: present

- name: Monitor canary for 10 minutes
  ansible.builtin.pause:
    minutes: 10
    prompt: "Canary is receiving 10% traffic. Check metrics. Press Enter to promote or Ctrl-C to abort."

- name: Promote canary to 100%
  community.aws.elb_listener:
    elb_arn: "{{ alb_arn }}"
    protocol: HTTPS
    port: 443
    default_actions:
      - Type: forward
        TargetGroupArn: "{{ canary_tg_arn }}"
    state: present
```

[↑ Back to TOC](#table-of-contents)

---

## Ansible Configuration Drift Detection

Ansible can detect and report configuration drift without making changes, acting as a continuous compliance tool.

### Compliance Playbook

```yaml
---
- name: Compliance scan — SSH hardening
  hosts: all
  become: true
  vars:
    required_sshd_settings:
      PermitRootLogin: "no"
      PasswordAuthentication: "no"
      MaxAuthTries: "3"
      Protocol: "2"
      X11Forwarding: "no"
      AllowAgentForwarding: "no"

  tasks:
    - name: Read sshd_config
      ansible.builtin.slurp:
        src: /etc/ssh/sshd_config
      register: sshd_config_raw

    - name: Parse sshd_config
      ansible.builtin.set_fact:
        sshd_config: "{{ sshd_config_raw.content | b64decode }}"

    - name: Check each required setting
      ansible.builtin.assert:
        that: "setting.value in sshd_config"
        fail_msg: "{{ inventory_hostname }}: sshd_config missing or wrong value for {{ setting.key }}"
        success_msg: "{{ inventory_hostname }}: {{ setting.key }} OK"
      loop: "{{ required_sshd_settings | dict2items }}"
      loop_control:
        loop_var: setting
      ignore_errors: true
      register: compliance_results

    - name: Collect non-compliant hosts
      ansible.builtin.set_fact:
        is_compliant: "{{ compliance_results.results | selectattr('failed', 'defined') | selectattr('failed') | list | length == 0 }}"

    - name: Report non-compliant hosts to Slack
      ansible.builtin.uri:
        url: "{{ slack_webhook_url }}"
        method: POST
        body_format: json
        body:
          text: "SSH compliance failure on {{ inventory_hostname }}: {{ compliance_results.results | selectattr('failed') | map(attribute='msg') | join('; ') }}"
      when: not is_compliant
      delegate_to: localhost
```

### Scheduling Compliance Scans in AWX

In AWX, create a job template for the compliance playbook and schedule it to run nightly. Set up notifications to post failures to a Slack channel or create JIRA tickets via the AWX notification system. This creates a continuous feedback loop: any configuration drift is detected within 24 hours and auto-reported.

[↑ Back to TOC](#table-of-contents)

---

## Ansible Galaxy Roles: Evaluating and Using Community Roles

The community provides thousands of Galaxy roles. Evaluating them carefully before adding to production is critical — a poorly written role can break idempotency, introduce security vulnerabilities, or become unmaintained.

### Evaluation Checklist

Before adopting a community role:
- **Download count and stars:** Higher numbers suggest community vetting
- **Last updated:** Avoid roles not updated in 2+ years
- **Test coverage:** Does it have Molecule tests?
- **Version pinning:** Does the `README.md` recommend pinning a version?
- **Multiple platform support:** Does it test on your target OS?
- **License:** Is it compatible with your use?

### Pinning and Vendoring

Always pin Galaxy roles to a specific version:

```yaml
# requirements.yml
---
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
  - name: geerlingguy.certbot
    version: "6.1.0"
  - name: geerlingguy.mysql
    version: "4.3.2"

collections:
  - name: community.general
    version: "9.4.0"
  - name: amazon.aws
    version: "8.2.0"
```

For critical production roles, consider vendoring: fork the role into your own namespace, review every line, and maintain your own copy. This prevents upstream breaking changes from affecting your production deployments.

### Jeff Geerling's Best Practices

The most widely used Ansible role author (Jeff Geerling) documents his patterns extensively. Key takeaways:
- Keep roles focused: one role, one concern
- Use `defaults/` extensively — roles should be configurable without modification
- Test with Molecule on multiple platforms
- Use FQCNs for all module references
- Document every variable in `defaults/main.yml` with a comment

[↑ Back to TOC](#table-of-contents)

---

## Real-World Ansible War Stories

### The Forgotten `when` Condition

A fintech startup's operations team had an Ansible playbook that rotated database passwords on all servers tagged `environment: production`. A well-intentioned developer added a new test environment and tagged the servers `environment: production-test`. The next scheduled run hit the production database and three production-test databases — and rotated all four passwords.

Unfortunately, the application tier on the production-test servers had already been updated with the new passwords, but the production application tier had not yet restarted. For fifteen minutes, production database connections failed as old-password sessions expired.

The fix was immediate: add an explicit `--limit` to the scheduled AWX job and require `confirm_production: true` as an extra variable for any playbook touching production databases. The lesson: tags like `production-test` will match a `when: "'production' in environment"` condition. Always use exact string matching for critical conditions.

### The Mitogen Migration That Broke Everything

A media company migrated their Ansible control node from standard execution to Mitogen for the promised 10x speedup. Local testing showed impressive results — a 400-task playbook dropped from 18 minutes to under 4 minutes.

On the first production run after the change, 12 out of 60 hosts reported failures on tasks that had worked fine for months. The error: Mitogen uses a different Python execution environment that does not automatically inherit all system Python path modifications. Several custom modules depended on site-packages installed to the system Python path in a non-standard location.

The resolution: add `mitogen_task_isolation: loose` to the Mitogen configuration and explicitly set `ansible_python_interpreter` to the full path for those host groups. But the lesson is wider: test performance optimisations thoroughly against a representative sample of production hosts before fleet-wide rollout.

[↑ Back to TOC](#table-of-contents)

---

## Ansible and HashiCorp Vault Integration

Managing secrets with Ansible Vault is fine for smaller environments, but at scale you want a dedicated secret management system. HashiCorp Vault provides dynamic credentials, secret rotation, and granular access policies.

### AppRole Authentication

```yaml
# group_vars/all/vault.yml (encrypted with ansible-vault)
vault_url: https://vault.mycompany.io:8200
vault_role_id: "{{ lookup('ansible.builtin.env', 'VAULT_ROLE_ID') }}"
vault_secret_id: "{{ lookup('ansible.builtin.env', 'VAULT_SECRET_ID') }}"
```

```yaml
- name: Authenticate with Vault and retrieve secrets
  hosts: app_servers
  tasks:
    - name: Get Vault token via AppRole
      ansible.builtin.uri:
        url: "{{ vault_url }}/v1/auth/approle/login"
        method: POST
        body_format: json
        body:
          role_id: "{{ vault_role_id }}"
          secret_id: "{{ vault_secret_id }}"
      register: vault_auth
      no_log: true

    - name: Set Vault token fact
      ansible.builtin.set_fact:
        vault_token: "{{ vault_auth.json.auth.client_token }}"
      no_log: true

    - name: Retrieve database credentials
      ansible.builtin.uri:
        url: "{{ vault_url }}/v1/secret/data/production/database"
        headers:
          X-Vault-Token: "{{ vault_token }}"
      register: db_secrets
      no_log: true

    - name: Configure app with database credentials
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
        mode: "0600"
        owner: myapp
        group: myapp
      vars:
        db_host: "{{ db_secrets.json.data.data.host }}"
        db_user: "{{ db_secrets.json.data.data.username }}"
        db_pass: "{{ db_secrets.json.data.data.password }}"
      no_log: true
```

### Dynamic Database Credentials

Vault's database secrets engine generates short-lived credentials on demand:

```yaml
- name: Get dynamic database credentials
  ansible.builtin.uri:
    url: "{{ vault_url }}/v1/database/creds/myapp-role"
    headers:
      X-Vault-Token: "{{ vault_token }}"
  register: dynamic_db_creds
  no_log: true

- name: Configure database connection
  ansible.builtin.template:
    src: database.yml.j2
    dest: /etc/myapp/database.yml
    mode: "0600"
  vars:
    db_username: "{{ dynamic_db_creds.json.data.username }}"
    db_password: "{{ dynamic_db_creds.json.data.password }}"
  no_log: true
```

Dynamic credentials expire after a lease period (e.g., 1 hour). This means a compromised credential is automatically invalid after a short time — a significant security improvement over long-lived static passwords.

[↑ Back to TOC](#table-of-contents)

---

## Large-Scale Ansible Practices

### Organising Playbooks for Large Environments

```
ansible/
├── ansible.cfg
├── requirements.yml
├── inventories/
│   ├── production/
│   │   ├── aws_ec2.yml        # dynamic inventory
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── app_servers.yml
│   │   │   └── db_servers.yml
│   │   └── host_vars/
│   │       └── special-host-01.yml
│   └── staging/
│       ├── aws_ec2.yml
│       └── group_vars/
│           └── all.yml
├── roles/
│   ├── base/                  # applied to all servers
│   ├── nginx/
│   ├── nodejs/
│   └── monitoring/
├── playbooks/
│   ├── site.yml               # imports all other playbooks
│   ├── bootstrap.yml          # new host onboarding
│   ├── deploy-app.yml
│   ├── rotate-certs.yml
│   └── compliance-scan.yml
└── library/                   # custom modules
    └── myapp_module.py
```

```yaml
# playbooks/site.yml
---
- name: Apply base configuration to all hosts
  ansible.builtin.import_playbook: base.yml

- name: Configure web servers
  ansible.builtin.import_playbook: webservers.yml

- name: Configure database servers
  ansible.builtin.import_playbook: dbservers.yml
```

### `ansible.cfg` for Production Teams

```ini
# ansible.cfg
[defaults]
inventory         = inventories/production
roles_path        = roles:~/.ansible/roles
collections_paths = ~/.ansible/collections:/usr/share/ansible/collections
remote_user       = ansible
private_key_file  = ~/.ssh/ansible_key
forks             = 20
timeout           = 30
host_key_checking = True
gathering         = smart
fact_caching      = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
stdout_callback   = yaml          # cleaner output format
callbacks_enabled = profile_tasks, timer   # show timing per task

[privilege_escalation]
become        = False
become_method = sudo
become_user   = root

[ssh_connection]
pipelining    = True     # reduces SSH connections — significant speedup
control_path  = ~/.ssh/ansible-cp-%%h-%%p-%%r
control_persist = 60s    # keep SSH multiplexing open for 60s between tasks

[persistent_connection]
command_timeout = 30
```

`pipelining = True` is one of the most impactful performance settings. It sends multiple tasks over a single SSH connection instead of opening a new one for each task. It requires `requiretty` to be disabled in sudoers on managed hosts.

### Limit Operations with Tags

```yaml
---
- name: Full application stack setup
  hosts: all
  roles:
    - role: base
      tags: [base, always]
    - role: nginx
      tags: [nginx, web]
    - role: nodejs
      tags: [nodejs, app]
    - role: monitoring
      tags: [monitoring]
    - role: certbot
      tags: [certs, ssl]
```

```bash
# Only run monitoring setup
ansible-playbook site.yml -t monitoring

# Run everything except slow certificate tasks
ansible-playbook site.yml --skip-tags=certs

# Tag 'always' runs regardless of --tags
ansible-playbook site.yml -t nginx  # also runs 'base' tagged tasks
```

[↑ Back to TOC](#table-of-contents)

---

## Ansible Configuration Management vs Immutable Infrastructure

There is an ongoing debate in the DevOps community about whether configuration management tools like Ansible are still needed in the era of containers and immutable infrastructure. The honest answer is: it depends on your environment.

### Where Ansible Remains Essential

- **Bare-metal servers and VMs:** When you cannot replace a host with a new container, you must configure it in place.
- **Network devices:** Routers and switches are not containers. CLI-based automation is still the norm.
- **Windows environments:** Ansible is the most widely used cross-platform automation tool.
- **Legacy applications:** Many production applications cannot be containerised without significant refactoring.
- **Bootstrap and provisioning:** Even for container-first environments, someone must configure the Kubernetes nodes.
- **Compliance and remediation:** Detecting and fixing drift on existing long-lived infrastructure.

### Where Immutable Infrastructure Wins

- **Stateless applications:** If your app is a Docker container, build a new image instead of patching the running container.
- **Kubernetes workloads:** Use Helm or Kustomize, not Ansible, to manage Kubernetes manifests.
- **Cloud-native environments:** Packer + Terraform + ASG instance refresh replaces Ansible for many EC2 use cases.

### The Pragmatic Position

Most real organisations run a mix. Kubernetes workloads are managed declaratively via GitOps. The Kubernetes nodes themselves are configured with Ansible. Legacy databases and network devices are managed with Ansible playbooks. New applications are containerised and deployed via CI/CD. Ansible fills the gaps that containers and immutable infrastructure cannot address.

The key is not to use Ansible everywhere because it is familiar, but to choose the right tool for each layer of the stack.

[↑ Back to TOC](#table-of-contents)

---

## Key Takeaways

After working through this module, you should be comfortable with:

- **Core Ansible architecture:** control node, managed nodes, inventories (static and dynamic), modules, and the agentless model
- **Playbook structure:** plays, tasks, handlers, variables, `when` conditions, `loop`, `block`, and `rescue`
- **Roles:** directory layout, defaults vs vars, handlers, templates, meta dependencies, and sharing via Galaxy
- **Idempotency:** why it matters and how to ensure it with `changed_when`, state-based modules, and `--check` mode
- **Secrets:** Ansible Vault (inline encryption, vault IDs, file encryption), and external vault integrations
- **Testing:** Molecule for role testing, `ansible-lint` for static analysis, integration with CI/CD
- **Performance:** `forks`, `pipelining`, Mitogen, fact caching, `async` tasks, and `serial` rolling deployments
- **Dynamic inventory:** AWS EC2 plugin, custom inventory scripts, constructed groups
- **Collections:** installing, using FQCNs, building private collections
- **AWX/AAP:** job templates, credentials, workflows, and RBAC for team operations
- **EDA:** event-driven automation for auto-remediation
- **Windows and network:** WinRM, `ansible.windows.*` modules, network device automation with `cisco.ios`, `community.network`

The meta-skill in Ansible is thinking declaratively and building idempotent, tested, reusable roles. Playbooks that run safely on the same host dozens of times — adding only what is missing, changing only what differs — are the foundation of reliable infrastructure automation.

[↑ Back to TOC](#table-of-contents)

---

## Ansible Security Best Practices

### Least Privilege for the Ansible User

The Ansible user should have the minimum permissions needed to run its tasks. Avoid running everything as root via unlimited sudo.

```ini
# /etc/sudoers.d/ansible (on managed hosts)
# Allow the ansible user to run specific commands without a password
ansible ALL=(root) NOPASSWD: /usr/bin/systemctl start nginx, \
                              /usr/bin/systemctl stop nginx, \
                              /usr/bin/systemctl restart nginx, \
                              /usr/bin/dnf install *

# For tasks that genuinely need full root access, require password
ansible ALL=(root) /usr/sbin/*, /usr/bin/tee
```

### Protecting Sensitive Data in Playbook Output

```yaml
# Prevent sensitive task output from appearing in logs
- name: Configure application secrets
  ansible.builtin.template:
    src: secrets.conf.j2
    dest: /etc/myapp/secrets.conf
    mode: "0600"
  vars:
    db_password: "{{ vault_db_password }}"
    api_key: "{{ vault_api_key }}"
  no_log: true    # suppress task output including variable values

# For registered variables
- name: Get API token
  ansible.builtin.uri:
    url: https://api.example.com/auth
    method: POST
    body: { username: admin, password: "{{ vault_admin_pass }}" }
  register: auth_result
  no_log: true

- name: Use the token
  ansible.builtin.uri:
    url: https://api.example.com/resource
    headers:
      Authorization: "Bearer {{ auth_result.json.token }}"
  no_log: true
```

### SSH Key Management

```yaml
# Deploy SSH authorized keys via Ansible
- name: Manage authorised SSH keys
  ansible.posix.authorized_key:
    user: deployer
    key: "{{ item }}"
    state: present
    exclusive: false
  loop: "{{ authorized_public_keys }}"

# Remove old/revoked keys
- name: Remove revoked SSH keys
  ansible.posix.authorized_key:
    user: deployer
    key: "{{ revoked_key }}"
    state: absent
```

Keep the list of `authorized_public_keys` in an encrypted vault file, rotating it whenever team members leave. Running this playbook on all servers removes access for departed team members within minutes.

[↑ Back to TOC](#table-of-contents)

---

## Ansible with Terraform: The Complete Workflow

Terraform and Ansible complement each other perfectly: Terraform provisions and manages cloud resources (VPCs, security groups, EC2 instances, RDS databases), while Ansible configures the software running on those resources.

### Pattern 1: Terraform Outputs → Ansible Inventory

```bash
# After terraform apply, generate an Ansible inventory from outputs
terraform output -json ec2_instance_ips | \
  jq -r '.[] | "app-server ansible_host=\(.) ansible_user=ec2-user"' \
  > inventories/production/hosts

# Run Ansible against the new inventory
ansible-playbook -i inventories/production/hosts playbooks/configure-app.yml
```

### Pattern 2: Terraform `local-exec` to Trigger Ansible

```hcl
# In Terraform, trigger Ansible after resource creation
resource "aws_instance" "app" {
  # ... instance config ...

  provisioner "local-exec" {
    command = <<-EOT
      sleep 30 && \
      ansible-playbook \
        -i "${self.private_ip}," \
        -u ec2-user \
        --private-key ${var.ssh_key_path} \
        --extra-vars "instance_id=${self.id} app_version=${var.app_version}" \
        playbooks/configure-instance.yml
    EOT
  }
}
```

The `sleep 30` gives the instance time to boot and SSH to become available. A more robust approach uses a retry loop.

### Pattern 3: Ansible Dynamic Inventory from Terraform State

```python
#!/usr/bin/env python3
# inventories/terraform_state.py
# Reads Terraform state to build Ansible inventory

import json
import subprocess
import sys

def get_inventory():
    result = subprocess.run(
        ['terraform', 'output', '-json'],
        capture_output=True, text=True,
        cwd='/path/to/terraform'
    )
    outputs = json.loads(result.stdout)

    inventory = {
        'app_servers': {
            'hosts': outputs.get('app_server_ips', {}).get('value', []),
            'vars': {
                'ansible_user': 'ec2-user',
                'app_port': outputs.get('app_port', {}).get('value', 8080),
            }
        },
        '_meta': { 'hostvars': {} }
    }
    return inventory

if __name__ == '__main__':
    if '--list' in sys.argv:
        print(json.dumps(get_inventory(), indent=2))
    elif '--host' in sys.argv:
        print(json.dumps({}))
```

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Lab: LAMP Stack with Ansible

This lab deploys a full LAMP stack (Linux, Apache, MariaDB, PHP) using Ansible roles on a fresh RHEL/Rocky Linux 9 host.

### Lab Setup

```bash
# Prerequisites: Ansible installed, SSH key available
# Target: a fresh RHEL 9 or Rocky Linux 9 host at 192.168.1.50

# Test connectivity
ansible -i '192.168.1.50,' all -m ping -u ec2-user --private-key ~/.ssh/lab_key
```

### Site Playbook

```yaml
# site.yml
---
- name: Bootstrap base configuration
  hosts: all
  become: true
  roles:
    - role: common
      tags: [common, base]

- name: Configure web server
  hosts: webservers
  become: true
  vars:
    http_port: 80
    php_version: "8.2"
  roles:
    - role: apache
      tags: [apache, web]
    - role: php
      tags: [php, web]

- name: Configure database server
  hosts: dbservers
  become: true
  vars:
    mysql_root_password: "{{ vault_mysql_root_password }}"
    mysql_databases:
      - name: webapp
        encoding: utf8mb4
    mysql_users:
      - name: webapp_user
        password: "{{ vault_mysql_webapp_password }}"
        priv: "webapp.*:ALL"
  roles:
    - role: mariadb
      tags: [mariadb, db]
```

### Common Role Tasks

```yaml
# roles/common/tasks/main.yml
---
- name: Set timezone
  community.general.timezone:
    name: UTC

- name: Install common packages
  ansible.builtin.dnf:
    name:
      - vim
      - curl
      - git
      - htop
      - unzip
      - python3
    state: present

- name: Configure sshd hardening
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    mode: "0600"
    validate: /usr/sbin/sshd -t -f %s
  notify: Restart sshd

- name: Install and configure fail2ban
  block:
    - ansible.builtin.dnf:
        name: fail2ban
        state: present
    - ansible.builtin.systemd:
        name: fail2ban
        state: started
        enabled: true
```

### Verify the Deployment

```bash
# Run the playbook
ansible-playbook \
  -i '192.168.1.50,' \
  -u ec2-user \
  --private-key ~/.ssh/lab_key \
  --vault-password-file ~/.vault_pass \
  --extra-vars "env=lab" \
  site.yml

# Verify Apache is serving PHP
ansible -i '192.168.1.50,' all \
  -m ansible.builtin.uri \
  -a "url=http://192.168.1.50/ status_code=200" \
  -u ec2-user

# Check idempotency — second run should show 0 changes
ansible-playbook \
  -i '192.168.1.50,' \
  -u ec2-user \
  --private-key ~/.ssh/lab_key \
  --vault-password-file ~/.vault_pass \
  site.yml \
  | grep -E "(changed|failed|ok)"
# Expected: changed=0, failed=0
```

[↑ Back to TOC](#table-of-contents)

---

## Ansible and Monitoring: Configuring Observability

One of the most common Ansible use cases is deploying and configuring monitoring agents across a fleet.

### Deploying Prometheus Node Exporter

```yaml
---
- name: Install Prometheus Node Exporter
  hosts: all
  become: true
  vars:
    node_exporter_version: "1.7.0"
    node_exporter_listen_address: "0.0.0.0:9100"

  tasks:
    - name: Create node_exporter user
      ansible.builtin.user:
        name: node_exporter
        shell: /sbin/nologin
        system: true

    - name: Download node_exporter
      ansible.builtin.get_url:
        url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
        dest: /tmp/node_exporter.tar.gz
        checksum: "sha256:https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/sha256sums.txt"

    - name: Extract node_exporter
      ansible.builtin.unarchive:
        src: /tmp/node_exporter.tar.gz
        dest: /usr/local/bin/
        remote_src: true
        extra_opts: ["--strip-components=1", "--wildcards", "*/node_exporter"]

    - name: Create systemd service
      ansible.builtin.template:
        src: node_exporter.service.j2
        dest: /etc/systemd/system/node_exporter.service
        mode: "0644"
      notify:
        - Reload systemd
        - Restart node_exporter

    - name: Enable and start node_exporter
      ansible.builtin.systemd:
        name: node_exporter
        state: started
        enabled: true

    - name: Open firewall port
      ansible.posix.firewalld:
        port: 9100/tcp
        permanent: true
        state: enabled
        immediate: true

  handlers:
    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: true

    - name: Restart node_exporter
      ansible.builtin.systemd:
        name: node_exporter
        state: restarted
```

### Dynamically Updating Prometheus scrape configs

After deploying node exporters, update the Prometheus server's scrape config to include the new hosts. This can be done with Ansible on the Prometheus server:

```yaml
- name: Update Prometheus scrape config
  hosts: prometheus_servers
  tasks:
    - name: Generate file-based SD config from inventory
      ansible.builtin.template:
        src: prometheus_targets.json.j2
        dest: /etc/prometheus/file_sd/nodes.json
        mode: "0644"
      notify: Reload Prometheus

  handlers:
    - name: Reload Prometheus
      ansible.builtin.uri:
        url: http://localhost:9090/-/reload
        method: POST
```

```jinja2
{# prometheus_targets.json.j2 #}
[
  {
    "targets": [
      {% for host in groups['all'] %}
      "{{ hostvars[host]['ansible_host'] }}:9100"{% if not loop.last %},{% endif %}
      {% endfor %}
    ],
    "labels": {
      "env": "{{ env }}"
    }
  }
]
```

[↑ Back to TOC](#table-of-contents)

---

## Certificate Management with Ansible

Certificate lifecycle management — issuing, deploying, and rotating TLS certificates — is well-suited to Ansible automation.

### Deploying Certificates from a Secrets Store

```yaml
- name: Deploy TLS certificate from Vault
  hosts: load_balancers
  become: true
  tasks:
    - name: Retrieve certificate from Vault PKI
      ansible.builtin.uri:
        url: "{{ vault_url }}/v1/pki/issue/web-role"
        method: POST
        headers:
          X-Vault-Token: "{{ vault_token }}"
        body_format: json
        body:
          common_name: "{{ inventory_hostname }}.mycompany.io"
          ttl: "720h"   # 30 days
      register: cert_response
      no_log: true

    - name: Write certificate to disk
      ansible.builtin.copy:
        content: "{{ cert_response.json.data.certificate }}"
        dest: /etc/ssl/certs/{{ inventory_hostname }}.crt
        mode: "0644"
      no_log: true
      notify: Reload nginx

    - name: Write private key to disk
      ansible.builtin.copy:
        content: "{{ cert_response.json.data.private_key }}"
        dest: /etc/ssl/private/{{ inventory_hostname }}.key
        mode: "0600"
        owner: root
        group: root
      no_log: true
      notify: Reload nginx
```

### Let's Encrypt with certbot Role

```yaml
- name: Provision Let's Encrypt certificates
  hosts: web_servers
  become: true
  vars:
    certbot_email: ops@mycompany.io
    certbot_certs:
      - domains:
          - myapp.com
          - www.myapp.com
      - domains:
          - api.myapp.com
    certbot_create_method: standalone
    certbot_auto_renew: true
    certbot_auto_renew_hour: "3"
    certbot_auto_renew_minute: "30"

  roles:
    - geerlingguy.certbot
```

The `geerlingguy.certbot` role handles installation, initial certificate issuance, and setting up auto-renewal via cron or systemd timer.

[↑ Back to TOC](#table-of-contents)

---

## Ansible Certification and Career Path

### Red Hat Ansible Certifications

Red Hat offers two Ansible certifications:

**EX294 — Red Hat Certified Engineer in Red Hat Ansible Automation (RHCE):** Practical hands-on exam that tests your ability to write playbooks, use roles, implement Ansible Vault, and manage systems. Duration: 4 hours. Format: live lab.

**EX374 — Ansible for Enterprise Automation (RHCA elective):** Tests advanced topics including Ansible Tower/AWX, workflows, and integrations.

### What to Study for EX294

- Writing playbooks with multiple plays and hosts
- Using Ansible Vault for secrets management
- Creating and using roles with Galaxy dependencies
- Managing variables: `defaults`, `vars`, `group_vars`, `host_vars`, `extra_vars`
- Implementing error handling: `ignore_errors`, `failed_when`, `block`/`rescue`/`always`
- Using common modules: `dnf`, `copy`, `template`, `file`, `service`, `user`, `cron`, `git`
- Writing and applying Jinja2 templates
- Implementing privilege escalation with `become`
- Managing static and dynamic inventories
- Using `--check` and `--diff` modes

### Community Certifications

Beyond Red Hat, the Ansible community has no vendor-neutral certification. However, demonstrating Ansible skills in interviews typically involves:
- Writing a playbook from scratch in under 30 minutes
- Explaining idempotency with examples
- Walking through how you would test a role
- Explaining your approach to secrets management
- Describing a complex Ansible project you have worked on

[↑ Back to TOC](#table-of-contents)

---

## Ansible Maturity Model

Like Terraform, Ansible adoption follows a maturity curve. Here is a practical guide to evolving your Ansible practice.

### Level 1: Ad-Hoc Commands

- Using `ansible` for one-off commands (`ping`, `command`, `shell`)
- No playbooks, no version control
- Works for single-operator environments

**Move to Level 2 when:** You find yourself running the same ad-hoc commands repeatedly.

### Level 2: Basic Playbooks

- Playbooks in a Git repository
- Static inventory files
- Using core modules (`dnf`, `copy`, `template`, `service`)
- Manual runs from the command line

**Move to Level 3 when:** You have more than 2-3 playbooks and start copy-pasting between them.

### Level 3: Roles and Testing

- Functionality extracted into roles
- Molecule tests for roles
- `ansible-lint` in pre-commit hooks or CI
- `requirements.yml` with pinned Galaxy dependencies
- Ansible Vault for secrets

**Move to Level 4 when:** You have a team making changes and need access control and audit trails.

### Level 4: AWX/AAP and GitOps

- AWX for team-based access, credential management, and job scheduling
- Dynamic inventory from cloud APIs
- Playbooks triggered by CI/CD pipelines or EDA
- Drift detection and compliance scanning on a schedule
- All playbook changes go through PR review before reaching production

### Level 5: Platform Integration

- Ansible Collections published to private Automation Hub
- EDA for real-time auto-remediation
- Ansible integrated with ITSM (ServiceNow) for change management
- Custom modules for internal systems
- Full audit trail from ITSM ticket to AWX job to system change

Most production environments should target Level 3–4. Level 5 is for large enterprises with dedicated automation teams.

[↑ Back to TOC](#table-of-contents)

---

## Common ansible.cfg Settings Reference

```ini
[defaults]
# Inventory
inventory = inventories/production

# Performance
forks = 20
timeout = 30
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

# Output formatting
stdout_callback = yaml        # readable output
callbacks_enabled = profile_tasks, timer

# Retry files — disable to keep working directory clean
retry_files_enabled = False

# Roles
roles_path = roles:~/.ansible/roles

# Collections
collections_paths = ~/.ansible/collections

# SSH settings
remote_user = ansible
private_key_file = ~/.ssh/ansible_ed25519
host_key_checking = True     # always True in production

# Log playbook output
log_path = /var/log/ansible.log

[privilege_escalation]
become = False
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True           # major performance improvement
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s
control_path = ~/.ssh/ansible-%%h-%%p-%%r

[persistent_connection]
command_timeout = 30
connect_timeout = 30
```

Each of these settings has a corresponding environment variable (e.g., `ANSIBLE_FORKS`, `ANSIBLE_REMOTE_USER`) so you can override per-run without modifying the config file.

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution. See [LICENSE.md](./LICENSE.md).*
