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

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution. See [LICENSE.md](./LICENSE.md).*
