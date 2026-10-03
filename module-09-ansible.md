# Module 09: Ansible

> Part of the [DevOps Career Course](./README.md) by UncleJS

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 09 of 15](https://img.shields.io/badge/module-09%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Intermediate-orange) ![Ansible 2.17+](https://img.shields.io/badge/Ansible-2.17%2B-EE0000?logo=ansible&logoColor=white) ![AWX stable](https://img.shields.io/badge/AWX-stable-EE0000?logo=ansible&logoColor=white) ![YAML · Agentless](https://img.shields.io/badge/features-YAML%20%C2%B7%20Agentless-lightgrey)

**Prerequisites:** Modules 01–04. You can use SSH, edit YAML, and run commands in a terminal. Ansible is installed on the control node (`sudo apt install -y pipx && pipx install 'ansible-core>=2.17,<2.19'`, then `pipx ensurepath`).

**Time:** About 8 hours, including the labs.

**Lab:** Ubuntu 24.04 unless a step says otherwise.

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: What is Ansible?](#beginner-what-is-ansible)
- [Beginner: Inventory](#beginner-inventory)
- [Beginner: Ad-Hoc Commands](#beginner-ad-hoc-commands)
- [Beginner: Playbooks](#beginner-playbooks)
- [Intermediate: Jinja2 Templates](#intermediate-jinja2-templates)
- [Intermediate: Roles](#intermediate-roles)
- [Intermediate: Handlers](#intermediate-handlers)
- [Intermediate: Dynamic Inventory](#intermediate-dynamic-inventory)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Ansible is the most widely-used configuration management tool in DevOps. Where Terraform provisions infrastructure (creates servers, networks, databases), Ansible **configures** that infrastructure — installs software, manages services, deploys applications, and enforces system state.

Ansible is agentless on the managed hosts — it connects to hosts over SSH and runs tasks. There's nothing to install on managed hosts beyond Python. Control-side tooling such as AWX Execution Environments does not change that host-side agentless model.

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
| Agent required | No on managed hosts (SSH/WinRM) | Yes | No |
| Primary use | Config management, app deploy | Config management | Infrastructure provisioning |
| Learning curve | Low | High | Medium |
| Idempotent | Yes | Yes | Yes |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Installation & Setup

Good installation and setup are less about getting the CLI on your laptop and more about creating a predictable execution environment. Automation is only trustworthy when it behaves the same way every time it runs. That means deciding where inventory lives, which SSH key is used, what user connects by default, how privilege escalation works, and how output is formatted for debugging. Those choices seem small at first, but they become very important once multiple engineers or CI jobs are running the same playbooks.

This is also where beginners often learn their first Ansible lesson: connection problems are usually more common than playbook problems. Before building elaborate roles, make sure the control node can authenticate cleanly, reach the target hosts, and escalate privileges safely. If those basics are shaky, every later section feels harder than it should.

```bash
# Ubuntu/Debian — Ansible 2.17 or 2.18
sudo apt update
sudo apt install -y pipx && pipx install 'ansible-core>=2.17,<2.19'
pipx ensurepath

# RHEL/Rocky/Fedora
sudo dnf install -y ansible

# Via pip (always latest version)
pip3 install ansible

# Verify
ansible --version

# Implicit localhost uses the local connection.
# A host line that only says `localhost` uses SSH unless you set ansible_connection=local.
ansible localhost -m ping -c local
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
web01.example.com ansible_host=10.0.1.10 http_port=8080
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

# Variables for a group — assignments only, no host lines
[web:vars]
nginx_port=80
app_env=production
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

# Ubuntu: install nginx on the web group
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

The playbook in this section uses only `apt`. It is an Ubuntu example. A Rocky or RHEL host needs `dnf` for the same package install, as the facts section shows later.

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

Variable precedence is why a value you set in one place is not the value Ansible uses. Extra vars (`-e` or `--extra-vars`) win over everything else. Role defaults in `defaults/main.yml` are the weakest and exist to be overridden. Role vars in `vars/main.yml` outrank play vars, so a `vars:` block on the play does not override them. `set_fact` and registered results outrank task vars. Extra vars always win. Put overridable defaults in `defaults/main.yml`, environment-specific values in `group_vars` or `host_vars`, and reserve extra vars for a one-off or CI override.

**Ansible facts** are variables automatically gathered from the managed host at the start of a play (via the `setup` module). They include the OS family and distribution, available memory, number of CPUs, network interfaces, IP addresses, and hostname. Facts make playbooks genuinely dynamic: you can write a single playbook that installs packages correctly on both Ubuntu (using `apt`) and RHEL (using `dnf`) by branching on `ansible_os_family`. Facts also let templates render host-specific values like `ansible_fqdn` or `ansible_default_ipv4.address` without requiring those values to be manually maintained in inventory.

This is also the point where automation can become confusing if naming and precedence are sloppy. Many Ansible mistakes come from not knowing which value wins, where it came from, or whether a variable was intended as a default, an override, or a secret. Treat variables as an interface and facts as runtime context, and the rest of the section becomes much easier to reason about.

### Variable precedence

- Role defaults (`defaults/main.yml`) are the weakest.
- Play vars outrank inventory variables, `group_vars`, and `host_vars`.
- Role vars (`vars/main.yml`) outrank play vars.
- `set_fact` and registered results outrank task vars.
- Extra vars (`-e`) win over everything. Extra vars always win.

### Defining Variables

```yaml
# group_vars/web.yml — variables for the web group
nginx_port: 80
app_name: myapp
max_connections: 1024
```

```yaml
# host_vars/web01.example.com.yml — overrides the group value for this host
nginx_port: 8080
```

```yaml
# In the playbook
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

`import_role` is **static**. Ansible expands it at parse time, so the role's tasks exist before the play runs. `--tags` and `--skip-tags` can target tasks inside an import more reliably for that reason. Because the expansion is static, you cannot loop `import_role`. `include_role` is **dynamic**: Ansible loads the role when the task runs, and that is the one you loop. Prefer `include_role` when looping, and loop with `loop`.

```yaml
- name: Apply the baseline once
  import_role:
    name: baseline

- name: Apply one role per app
  include_role:
    name: "{{ item }}"
  loop:
    - nginx
    - app
```

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
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install -r requirements.yml
```

```yaml
# requirements.yml
---
roles:
  - name: geerlingguy.nginx
  - name: geerlingguy.postgresql
    # This role's release line is 3.x. Pin a 3.x tag from Galaxy when you need a lock.
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
  changed_when: false

- name: Show disk info
  debug:
    var: disk_info.stdout_lines

- name: Parse the Use% integer on the / line
  set_fact:
    root_use_pct: "{{ ((disk_info.stdout_lines | select('search', '\\s/$') | first).split()[-2] | regex_replace('%', '')) | int }}"

- name: Fail if disk is over 90%
  fail:
    msg: "Disk usage is critical!"
  when: root_use_pct | int > 90
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Dynamic Inventory

For cloud environments where servers come and go, use dynamic inventory that queries the cloud API.

Dynamic inventory becomes necessary when your infrastructure stops being static enough for hand-maintained host files. Autoscaling groups, ephemeral instances, blue-green environments, and multi-region deployments all create churn that static inventory struggles to represent accurately. In those environments, the safest source of truth is often the cloud control plane itself.

This shift is important because it changes how you think about host targeting. Instead of managing named machines manually, you start targeting groups derived from tags, regions, roles, or other metadata. That approach is usually more resilient, but only if your cloud tagging discipline is strong. Poor tags produce poor inventory just as quickly as poor static files do.

`boto3` is the AWS SDK the plugin calls. Installing `boto3` alone does not install the plugin. The plugin ships in the `amazon.aws` collection.

```bash
ansible-galaxy collection install amazon.aws
pip3 install boto3 botocore
```

```yaml
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
```

```bash
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

AWX 19+ uses **Execution Environments** — container images that bundle Ansible, collections, and Python dependencies. These run on the control side, not on the managed hosts, so Ansible remains agentless while eliminating "works on my machine" problems:

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

An inventory host named `localhost` uses SSH unless `ansible_connection=local` is set. Filter facts in Ansible so a closed pipe is not the thing that fails the command.

```bash
sudo apt update
sudo apt install -y pipx && pipx install 'ansible-core>=2.17,<2.19'
pipx ensurepath
export PATH="$HOME/.local/bin:$PATH"
mkdir -p ~/labs/module-09-ping
cd ~/labs/module-09-ping
printf 'localhost ansible_connection=local\n' > inventory
ansible -i inventory all -m ping
ansible -i inventory all -m setup -a 'filter=ansible_distribution*'
```

**Expected:** `ping` returns `pong` with `SUCCESS`. The setup task prints Ubuntu distribution facts. Ansible does not open an SSH session to localhost.

**Cleanup:** `rm -rf ~/labs/module-09-ping`

### Lab 9.2 — Install a Web Stack

This lab uses `apt` on Ubuntu. The playbook sets `become: true`, so each run passes `--ask-become-pass`. Passwordless sudo is required if you omit that flag.

```bash
mkdir -p ~/labs/module-09-web/files
cd ~/labs/module-09-web
printf 'localhost ansible_connection=local\n' > inventory
printf '<h1>Hello from Ansible</h1>\n' > files/index.html
cat > install-nginx.yml <<'EOF'
---
- name: Install nginx on Ubuntu
  hosts: all
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: true
    - name: Ensure nginx is started and enabled
      service:
        name: nginx
        state: started
        enabled: true
    - name: Deploy index.html
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
EOF
ansible-playbook -i inventory install-nginx.yml --ask-become-pass
curl -fsS http://127.0.0.1/
ansible-playbook -i inventory install-nginx.yml --ask-become-pass
ansible-playbook -i inventory install-nginx.yml --check --ask-become-pass
```

**Expected:** The first run installs nginx, so `curl` prints `Hello from Ansible`. The second run reports `ok` for the install, service, and file tasks. `--check` then reports that a further run would not change those tasks.

**Cleanup:** `sudo apt remove -y nginx && rm -rf ~/labs/module-09-web`

### Lab 9.3 — Dynamic Configuration with Templates

```bash
mkdir -p ~/labs/module-09-tpl/templates
cd ~/labs/module-09-tpl
printf 'localhost ansible_connection=local\n' > inventory
cat > templates/vhost.conf.j2 <<'EOF'
server {
    listen {{ nginx_port }};
    server_name {{ ansible_fqdn }};
    root /var/www/{{ app_name }};
}
EOF
cat > vhost.yml <<'EOF'
---
- name: Render a vhost
  hosts: all
  vars:
    nginx_port: 8080
    app_name: myapp
  tasks:
    - name: Deploy vhost
      template:
        src: templates/vhost.conf.j2
        dest: /tmp/myapp-vhost.conf
      notify: Record render
  handlers:
    - name: Record render
      copy:
        dest: /tmp/myapp-vhost.rendered
        content: "{{ ansible_fqdn }}\n"
EOF
ansible-playbook -i inventory vhost.yml
grep listen /tmp/myapp-vhost.conf
cat /tmp/myapp-vhost.rendered
```

**Expected:** `/tmp/myapp-vhost.conf` contains `listen 8080` and a `server_name` equal to this machine's FQDN. `/tmp/myapp-vhost.rendered` contains that same FQDN because the handler ran.

**Cleanup:** `rm -rf ~/labs/module-09-tpl /tmp/myapp-vhost.conf /tmp/myapp-vhost.rendered`

### Lab 9.4 — Build a Role

```bash
mkdir -p ~/labs/module-09-role
cd ~/labs/module-09-role
printf 'localhost ansible_connection=local\n' > inventory
ansible-galaxy role init roles/webserver
cat > roles/webserver/defaults/main.yml <<'EOF'
web_port: 80
web_root: /var/www/html
EOF
cat > roles/webserver/tasks/main.yml <<'EOF'
- name: Install nginx on Ubuntu
  apt:
    name: nginx
    state: present
    update_cache: true
- name: Ensure nginx is started
  service:
    name: nginx
    state: started
    enabled: true
EOF
cat > site.yml <<'EOF'
---
- name: Configure the web host
  hosts: all
  become: true
  roles:
    - webserver
EOF
ansible-playbook -i inventory site.yml
ansible-playbook -i inventory site.yml
```

**Expected:** The first run reports `changed` for the package and service tasks. The second run reports `ok` for both. `systemctl is-active nginx` prints `active`.

**Cleanup:** `sudo apt remove -y nginx && rm -rf ~/labs/module-09-role`

### Lab 9.5 — Ansible Vault

```bash
mkdir -p ~/labs/module-09-vault/templates
cd ~/labs/module-09-vault
printf 'localhost ansible_connection=local\n' > inventory
printf 'correct horse battery staple\n' > .vault-pass
chmod 600 .vault-pass
cat > secrets.yml <<'EOF'
db_password: fake-db-password
EOF
ansible-vault encrypt --vault-password-file .vault-pass secrets.yml
cat > templates/db.env.j2 <<'EOF'
DB_PASSWORD={{ db_password }}
EOF
cat > use-secret.yml <<'EOF'
---
- name: Render a secret into a local file
  hosts: all
  vars_files:
    - secrets.yml
  tasks:
    - name: Write db.env
      template:
        src: templates/db.env.j2
        dest: /tmp/db.env
        mode: "0600"
EOF
ansible-playbook -i inventory use-secret.yml --vault-password-file .vault-pass
cat /tmp/db.env
```

To prompt instead of reading the file, run `ansible-playbook -i inventory use-secret.yml --ask-vault-pass` and type the passphrase stored in `.vault-pass`.

**Expected:** `secrets.yml` starts with `$ANSIBLE_VAULT`. `/tmp/db.env` contains `DB_PASSWORD=fake-db-password`. The password-file run does not prompt.

**Cleanup:** `rm -rf ~/labs/module-09-vault /tmp/db.env`

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/) — Community roles
- [Jeff Geerling's Ansible for DevOps](https://www.ansiblefordevops.com/)
- [Molecule — Ansible Role Testing](https://ansible.readthedocs.io/projects/molecule/)
- [Glossary: Ansible](./glossary.md#a), [Idempotent](./glossary.md#i), [Jinja2](./glossary.md#j), [Playbook](./glossary.md#p)

[↑ Back to TOC](#table-of-contents)
