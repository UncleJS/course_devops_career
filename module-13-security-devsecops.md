# Module 13: Security & DevSecOps

> **Course**: DevOps Career Path  
> **Audience**: Beginner → Intermediate  
> **Prerequisites**: Module 05 (Containers), Module 06 (Kubernetes), Module 08 (IaC)

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 13 of 15](https://img.shields.io/badge/module-13%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Advanced-red) ![Trivy 0.58+](https://img.shields.io/badge/Trivy-0.58%2B-1904DA) ![Vault 1.18+](https://img.shields.io/badge/Vault-1.18%2B-FFCF25?logo=vault&logoColor=black) ![Falco 0.39+](https://img.shields.io/badge/Falco-0.39%2B-00ADEF) ![Container · Runtime · Secrets](https://img.shields.io/badge/scope-Container%20%C2%B7%20Runtime%20%C2%B7%20Secrets-grey)

---

## Table of Contents

1. [Overview](#overview)
2. [Learning Objectives](#learning-objectives)
3. [The DevSecOps Mindset](#the-devsecops-mindset)
4. [Identity & Access Management (IAM)](#identity--access-management-iam)
5. [RBAC in Kubernetes](#rbac-in-kubernetes)
6. [Secrets Management with HashiCorp Vault](#secrets-management-with-hashicorp-vault)
7. [SAST — Static Application Security Testing](#sast--static-application-security-testing)
8. [DAST — Dynamic Application Security Testing](#dast--dynamic-application-security-testing)
9. [Software Composition Analysis (SCA)](#software-composition-analysis-sca)
10. [Container Security](#container-security)
11. [Policy as Code — OPA & Gatekeeper](#policy-as-code--opa--gatekeeper)
12. [Network Security](#network-security)
13. [Cloud Security Fundamentals](#cloud-security-fundamentals)
14. [Security in CI/CD Pipelines](#security-in-cicd-pipelines)
15. [Compliance & Audit Frameworks](#compliance--audit-frameworks)
16. [Runtime Security with Falco](#runtime-security-with-falco)
17. [Tools & Commands Reference](#tools--commands-reference)
18. [Hands-On Labs](#hands-on-labs)
19. [Further Reading](#further-reading)

---

## Overview

Security is not a phase at the end of development — it is a continuous discipline integrated at every layer of the DevOps lifecycle. DevSecOps shifts security left, embedding automated security checks into CI/CD pipelines, treating security policy as code, and establishing least-privilege access patterns from day one. This module covers the foundational security pillars every DevOps practitioner must understand.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module, you will be able to:

- Describe the DevSecOps shift-left philosophy and threat model
- Apply least-privilege IAM patterns on AWS, Azure, and GCP
- Configure Kubernetes RBAC for users, service accounts, and namespaces
- Deploy and operate HashiCorp Vault for dynamic secrets management
- Integrate SAST, DAST, and SCA tools into CI/CD pipelines
- Identify and remediate container image vulnerabilities
- Write and enforce OPA/Gatekeeper policies in Kubernetes
- Apply Kubernetes NetworkPolicies for micro-segmentation
- Describe the cloud shared responsibility model
- Design a security scanning pipeline from commit to deploy
- Install and configure Falco for runtime threat detection in Kubernetes

[↑ Back to TOC](#table-of-contents)

---

## The DevSecOps Mindset

### Shift Left

```
Traditional:  Dev → Test → Stage → [Security Audit] → Prod
DevSecOps:   [Sec] Dev → [Sec] CI → [Sec] CD → [Sec] Prod → [Sec Monitor]
```

Security checks run at **every gate**, not just before production.

### OWASP Top 10 — Cloud/DevOps relevance

| # | Category | DevOps Impact |
|---|----------|---------------|
| A01 | Broken Access Control | RBAC misconfig, overprivileged service accounts |
| A02 | Cryptographic Failures | Secrets in code/env vars, weak TLS |
| A03 | Injection | SQL/command injection in apps |
| A05 | Security Misconfiguration | Public S3 buckets, open ports, default creds |
| A06 | Vulnerable Components | Outdated container images, unpatched libraries |
| A07 | Auth & Session Mgmt | Weak API keys, shared credentials |
| A08 | Software & Data Integrity | Supply chain attacks, unsigned images |
| A09 | Logging & Monitoring Failures | No audit trail, silent breaches |

### Threat modeling — STRIDE

| Threat | Description | Mitigation |
|--------|-------------|------------|
| **S**poofing | Impersonating an identity | MFA, certificate auth |
| **T**ampering | Modifying data/code | Signed commits, image signing, checksums |
| **R**epudiation | Denying an action occurred | Immutable audit logs |
| **I**nformation Disclosure | Exposing sensitive data | Encryption at rest/transit, secrets management |
| **D**enial of Service | Overwhelming resources | Rate limiting, autoscaling, WAF |
| **E**levation of Privilege | Gaining unauthorized access | Least privilege, RBAC, Pod Security Standards |

[↑ Back to TOC](#table-of-contents)

---

## Identity & Access Management (IAM)

### Principle of Least Privilege

> Grant only the permissions required to perform a specific task — nothing more.

### AWS IAM

```json
// ✅ Good — scoped to specific bucket and actions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/uploads/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-app-bucket",
      "Condition": {
        "StringLike": { "s3:prefix": ["uploads/*"] }
      }
    }
  ]
}
```

```json
// ❌ Bad — wildcard actions and resources
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

#### IAM Role for EC2 / EKS (avoid static keys)

```bash
# EC2 instance profile — app gets credentials via metadata service
aws iam create-role --role-name my-app-role \
  --assume-role-policy-document file://ec2-trust-policy.json

aws iam attach-role-policy --role-name my-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam create-instance-profile --instance-profile-name my-app-profile
aws iam add-role-to-instance-profile --instance-profile-name my-app-profile \
  --role-name my-app-role
```

### AWS IAM Access Analyzer

```bash
# Find public S3 buckets and overprivileged roles
aws accessanalyzer list-findings --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789:analyzer/default

# Enable in all regions via AWS Config
aws accessanalyzer create-analyzer --analyzer-name "default" --type ACCOUNT
```

### GCP IAM — Workload Identity (Kubernetes)

```bash
# Bind Kubernetes ServiceAccount to GCP Service Account
gcloud iam service-accounts add-iam-policy-binding \
  my-gsa@my-project.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:my-project.svc.id.goog[my-namespace/my-ksa]"

kubectl annotate serviceaccount my-ksa \
  --namespace my-namespace \
  iam.gke.io/gcp-service-account=my-gsa@my-project.iam.gserviceaccount.com
```

[↑ Back to TOC](#table-of-contents)

---

## RBAC in Kubernetes

Kubernetes RBAC controls who can do what to which resources within the cluster.

### Core objects

| Object | Scope | Purpose |
|--------|-------|---------|
| **Role** | Namespace | Grants permissions within a namespace |
| **ClusterRole** | Cluster-wide | Grants permissions across all namespaces |
| **RoleBinding** | Namespace | Binds a Role to a user/group/SA |
| **ClusterRoleBinding** | Cluster-wide | Binds a ClusterRole to a user/group/SA |
| **ServiceAccount** | Namespace | Identity for pods (non-human) |

### Role — namespace-scoped

```yaml
# Allow read access to pods and logs in 'production' namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

```yaml
# Bind the role to a user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: production
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole for monitoring

```yaml
# Grant Prometheus read access to all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-server
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-server
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-server
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount — pod identity

```yaml
# Create a dedicated ServiceAccount for your app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-api
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-api-role  # AWS IRSA
automountServiceAccountToken: false  # ← disable unless needed
```

```yaml
# Reference it in the deployment
spec:
  serviceAccountName: my-api
  automountServiceAccountToken: false
```

### Audit RBAC with kubectl

```bash
# Who can do what?
kubectl auth can-i create pods --as=alice@example.com -n production
kubectl auth can-i delete deployments --as=system:serviceaccount:production:my-api -n production

# List all permissions a ServiceAccount has
kubectl auth can-i --list --as=system:serviceaccount:production:my-api -n production

# See all RoleBindings in a namespace
kubectl get rolebindings,clusterrolebindings -n production -o wide

# Find overprivileged service accounts (secrets access)
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'
```

[↑ Back to TOC](#table-of-contents)

---

## Secrets Management with HashiCorp Vault

HashiCorp Vault is a secrets management system that provides centralized, audited, time-limited access to credentials, API keys, certificates, and database passwords.

### Why not Kubernetes Secrets?

| | Kubernetes Secrets | Vault |
|--|-------------------|-------|
| **Encoding** | Base64 (not encrypted) | AES-256-GCM encrypted at rest |
| **Audit log** | Limited | Full audit trail |
| **Rotation** | Manual | Automatic (dynamic secrets) |
| **Leasing** | No | Time-limited, auto-expire |
| **Dynamic creds** | No | Yes (DB, cloud, PKI) |
| **Multi-cluster** | No | Yes |

### Vault Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      VAULT                              │
│                                                         │
│  Auth Methods          Secret Engines                   │
│  ┌──────────────┐      ┌──────────────────────────────┐ │
│  │ Kubernetes   │      │ KV v2 (key-value)            │ │
│  │ AWS IAM      │      │ Database (dynamic creds)     │ │
│  │ AppRole      │  ←→  │ PKI (TLS certificates)       │ │
│  │ GitHub       │      │ AWS/GCP/Azure (cloud creds)  │ │
│  │ LDAP / OIDC  │      │ SSH (signed certificates)    │ │
│  └──────────────┘      └──────────────────────────────┘ │
│                                                         │
│  Policies              Audit Devices                    │
│  ┌──────────────┐      ┌──────────────────────────────┐ │
│  │ HCL rules    │      │ File / Syslog / Socket       │ │
│  │ per-path ACL │      │ Immutable audit trail        │ │
│  └──────────────┘      └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Install and initialize Vault (dev mode)

```bash
# Install Vault
wget https://releases.hashicorp.com/vault/1.16.0/vault_1.16.0_linux_amd64.zip
unzip vault_1.16.0_linux_amd64.zip && mv vault /usr/local/bin/

# Start in dev mode (in-memory, insecure — for learning only)
vault server -dev &
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'  # printed at startup

# Check status
vault status
```

### KV Secrets Engine (v2)

```bash
# Enable KV v2 (already enabled in dev mode at secret/)
vault secrets enable -path=secret kv-v2

# Write a secret
vault kv put secret/myapp/database \
  username="app_user" \
  password="SuperSecret123!"

# Read a secret
vault kv get secret/myapp/database

# Read as JSON
vault kv get -format=json secret/myapp/database | jq .data.data

# List secrets
vault kv list secret/myapp/

# Delete (soft-delete in KV v2)
vault kv delete secret/myapp/database

# Destroy a specific version
vault kv destroy -versions=1 secret/myapp/database
```

### Dynamic Database Credentials

```bash
# Enable the database secrets engine
vault secrets enable database

# Configure MySQL connection
vault write database/config/my-mysql \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(db-01:3306)/" \
  allowed_roles="app-role" \
  username="vault_admin" \
  password="VaultAdminPass!"

# Create a role (Vault generates creds with this role)
vault write database/roles/app-role \
  db_name=my-mysql \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT, INSERT, UPDATE ON appdb.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"

# Generate dynamic credentials (used by app at startup)
vault read database/creds/app-role
# Returns: username=v-token-app-role-abc123, password=A1B2-C3D4-...
# Credentials auto-expire in 1 hour
```

### Vault Policy

```hcl
# policy: app-policy.hcl
# Allow app to read its secrets
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# Allow app to generate dynamic DB creds
path "database/creds/app-role" {
  capabilities = ["read"]
}

# Deny everything else
path "*" {
  capabilities = ["deny"]
}
```

```bash
vault policy write app-policy app-policy.hcl
```

### Kubernetes Auth Method

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# Configure (from inside the cluster)
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://${KUBERNETES_PORT_443_TCP_ADDR}:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Bind a Kubernetes ServiceAccount to a Vault policy
vault write auth/kubernetes/role/my-api \
  bound_service_account_names=my-api \
  bound_service_account_namespaces=production \
  policies=app-policy \
  ttl=1h
```

### Vault Agent — automatic secret injection into pods

```yaml
# Kubernetes deployment with Vault Agent sidecar (via annotations)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
  namespace: production
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "my-api"
        vault.hashicorp.com/agent-inject-secret-db: "secret/data/myapp/database"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "secret/data/myapp/database" -}}
          DB_USER={{ .Data.data.username }}
          DB_PASS={{ .Data.data.password }}
          {{- end }}
    spec:
      serviceAccountName: my-api
      containers:
        - name: my-api
          image: myapp:latest
          # Secret available at: /vault/secrets/db
          command: ["/bin/sh", "-c", "source /vault/secrets/db && ./app"]
```

[↑ Back to TOC](#table-of-contents)

---

## SAST — Static Application Security Testing

SAST analyzes source code **without executing it** to find security vulnerabilities.

### Tools

| Tool | Language | Type | Free |
|------|----------|------|------|
| **Semgrep** | Multi-language | SAST | ✅ OSS |
| **Bandit** | Python | SAST | ✅ OSS |
| **ESLint security** | JavaScript | SAST | ✅ OSS |
| **SpotBugs / FindSecBugs** | Java | SAST | ✅ OSS |
| **Gosec** | Go | SAST | ✅ OSS |
| **SonarQube** | Multi-language | SAST + metrics | Freemium |
| **Checkmarx** | Multi-language | SAST | Commercial |
| **Snyk Code** | Multi-language | SAST | Freemium |

### Semgrep — multi-language SAST

```bash
# Install
pip install semgrep

# Run with OWASP ruleset
semgrep --config=p/owasp-top-ten ./src

# Run with security audit rules
semgrep --config=p/security-audit ./src

# Run custom rule
cat > no-hardcoded-secrets.yml << 'EOF'
rules:
  - id: hardcoded-password
    patterns:
      - pattern: password = "..."
      - pattern: PASSWORD = "..."
      - pattern: passwd = "..."
    message: "Hardcoded password detected: $X"
    languages: [python, javascript, go]
    severity: ERROR
EOF

semgrep --config=no-hardcoded-secrets.yml ./src

# Output SARIF (for GitHub Code Scanning)
semgrep --config=p/owasp-top-ten --sarif ./src > results.sarif
```

### Bandit — Python security linter

```bash
# Install
pip install bandit

# Scan a directory
bandit -r ./src -l -i

# Scan with specific tests (B201-B608)
bandit -r ./src -t B201,B602,B607

# Generate HTML report
bandit -r ./src -f html -o bandit-report.html

# Exit code 1 if HIGH severity found (for CI gate)
bandit -r ./src -l --exit-zero
```

### Gosec — Go security scanner

```bash
# Install
go install github.com/securego/gosec/v2/cmd/gosec@latest

# Scan
gosec ./...

# Output JSON
gosec -fmt json -out gosec-report.json ./...

# Only report HIGH issues
gosec -severity high ./...
```

### CI/CD integration — GitHub Actions

```yaml
# .github/workflows/sast.yml
name: SAST Security Scan

on: [push, pull_request]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
            p/secrets
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

  bandit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install bandit
      - run: bandit -r src/ -ll -f json -o bandit-results.json || true
      - uses: actions/upload-artifact@v4
        with:
          name: bandit-results
          path: bandit-results.json
```

[↑ Back to TOC](#table-of-contents)

---

## DAST — Dynamic Application Security Testing

DAST tests a **running application** by sending malicious inputs and observing responses.

### Tools

| Tool | Type | Free |
|------|------|------|
| **OWASP ZAP** | Full DAST proxy | ✅ OSS |
| **Nuclei** | Template-based scanner | ✅ OSS |
| **Nikto** | Web server scanner | ✅ OSS |
| **Burp Suite** | Manual + automated | Freemium |
| **Invicti (Netsparker)** | DAST | Commercial |

### OWASP ZAP — automated baseline scan

```bash
# Run a quick baseline scan (passive rules only — no active attacks)
docker run --rm \
  -v $(pwd):/zap/wrk:rw \
  ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t https://staging.example.com \
  -r zap-report.html \
  -J zap-report.json

# Full active scan (sends actual attack payloads)
docker run --rm \
  -v $(pwd):/zap/wrk:rw \
  ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
  -t https://staging.example.com \
  -r zap-full-report.html \
  -I  # Don't fail build on warnings
```

### ZAP in CI/CD (GitHub Actions)

```yaml
- name: OWASP ZAP Baseline Scan
  uses: zaproxy/action-baseline@v0.12.0
  with:
    target: 'https://staging.example.com'
    rules_file_name: '.zap/rules.tsv'
    cmd_options: '-a'     # Ajax spider
    fail_action: false    # Don't fail the build (report only)
```

### Nuclei — template-based vulnerability scanner

```bash
# Install
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest

# Update templates
nuclei -update-templates

# Scan a target
nuclei -u https://example.com -t cves/ -t exposures/

# Only critical/high severity
nuclei -u https://example.com -severity critical,high

# Scan with auth
nuclei -u https://example.com -H "Authorization: Bearer $TOKEN"

# Output JSON
nuclei -u https://example.com -json -o nuclei-results.json
```

[↑ Back to TOC](#table-of-contents)

---

## Software Composition Analysis (SCA)

SCA identifies vulnerabilities in **third-party libraries and dependencies** (the software supply chain).

### Tools

| Tool | Targets | Free |
|------|---------|------|
| **Trivy** | Containers, packages, IaC, code | ✅ OSS |
| **Grype** | Containers and packages | ✅ OSS |
| **Syft** | Generate SBOMs | ✅ OSS |
| **OWASP Dependency-Check** | Maven, npm, pip, etc. | ✅ OSS |
| **Snyk** | Multi-ecosystem | Freemium |
| **Dependabot** | GitHub-native | ✅ Free |

### Trivy — container + filesystem scanning

```bash
# Install
wget https://github.com/aquasecurity/trivy/releases/download/v0.51.0/trivy_0.51.0_Linux-64bit.tar.gz
tar xvf trivy_0.51.0_Linux-64bit.tar.gz
mv trivy /usr/local/bin/

# Scan a container image
trivy image nginx:latest

# Scan only CRITICAL and HIGH vulnerabilities
trivy image --severity CRITICAL,HIGH nginx:latest

# Scan a filesystem (your project)
trivy fs --scanners vuln,secret,misconfig .

# Scan a Dockerfile / IaC
trivy config Dockerfile
trivy config terraform/

# Fail if CRITICAL vuln found (for CI gate)
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Generate SBOM (Software Bill of Materials)
trivy image --format cyclonedx --output sbom.json nginx:latest

# JSON output for integration
trivy image --format json --output results.json myapp:latest
```

### SBOM — Software Bill of Materials

```bash
# Generate SBOM with Syft
syft myapp:latest -o cyclonedx-json > sbom.cyclonedx.json
syft myapp:latest -o spdx-json > sbom.spdx.json

# Scan SBOM for vulnerabilities with Grype
grype sbom:./sbom.cyclonedx.json

# Sign SBOM with Cosign (supply chain attestation)
cosign attest --predicate sbom.cyclonedx.json --type cyclonedx myapp:latest
```

[↑ Back to TOC](#table-of-contents)

---

## Container Security

### Dockerfile security best practices

```dockerfile
# ✅ Use a specific, minimal base image (not :latest)
FROM python:3.12-slim-bookworm

# ✅ Run as non-root user
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/bash --create-home appuser

WORKDIR /app

# ✅ Copy only what's needed, set ownership
COPY --chown=appuser:appgroup requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appgroup . .

# ✅ Drop all capabilities, use read-only filesystem
USER appuser

# ✅ Expose specific port only
EXPOSE 8080

# ✅ Use exec form (no shell injection)
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Kubernetes Pod Security Standards (PSS)

PSS replaces the deprecated PodSecurityPolicy (PSP). Three modes:

| Level | Description |
|-------|-------------|
| **privileged** | Unrestricted — allows everything |
| **baseline** | Prevents known privilege escalations |
| **restricted** | Strongest — follows security best practices |

```yaml
# Enforce restricted PSS on a namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### Secure Pod spec

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
  namespace: production
spec:
  template:
    spec:
      # ✅ Non-root user
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault

      # ✅ Disable automount of ServiceAccount token unless needed
      automountServiceAccountToken: false

      containers:
        - name: my-api
          image: myapp:1.2.3   # ✅ Specific tag, not :latest
          securityContext:
            allowPrivilegeEscalation: false   # ✅
            readOnlyRootFilesystem: true       # ✅
            capabilities:
              drop: ["ALL"]                    # ✅
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: tmp
              mountPath: /tmp               # Only writable dir
      volumes:
        - name: tmp
          emptyDir: {}
```

### Image signing with Cosign

```bash
# Install Cosign
wget https://github.com/sigstore/cosign/releases/download/v2.2.3/cosign-linux-amd64
chmod +x cosign-linux-amd64 && mv cosign-linux-amd64 /usr/local/bin/cosign

# Generate a key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key registry.example.com/myapp:1.2.3

# Verify signature
cosign verify --key cosign.pub registry.example.com/myapp:1.2.3

# Sign with GitHub OIDC (keyless — Sigstore)
cosign sign registry.example.com/myapp:1.2.3  # Prompts OIDC login
```

[↑ Back to TOC](#table-of-contents)

---

## Policy as Code — OPA & Gatekeeper

**Open Policy Agent (OPA)** is a general-purpose policy engine. **Gatekeeper** is OPA for Kubernetes — it enforces policies as Kubernetes admission webhooks.

### OPA basics (Rego language)

```rego
# policy.rego — deny root containers
package kubernetes.admission

deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    not container.securityContext.runAsNonRoot
    msg := sprintf("Container '%s' must set runAsNonRoot=true", [container.name])
}
```

### Install Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.15.0/deploy/gatekeeper.yaml

# Verify
kubectl get pods -n gatekeeper-system
```

### ConstraintTemplate — define the policy schema

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requirerunasnonroot
spec:
  crd:
    spec:
      names:
        kind: RequireRunAsNonRoot
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requirerunasnonroot

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' must set runAsNonRoot=true", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Init container '%v' must set runAsNonRoot=true", [container.name])
        }
```

### Constraint — apply the policy

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireRunAsNonRoot
metadata:
  name: require-run-as-non-root
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production", "staging"]
  enforcementAction: deny  # or: warn, dryrun
```

### Common Gatekeeper policies

```yaml
# 1. Require resource limits on all containers
# 2. Disallow privileged containers
# 3. Require specific labels on namespaces
# 4. Restrict container image registries to approved ones
# 5. Require readOnlyRootFilesystem
# 6. Disallow HostPath volumes
# 7. Require Liveness and Readiness probes
```

```yaml
# Example: Restrict image registry
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: allowedregistries
spec:
  crd:
    spec:
      names:
        kind: AllowedRegistries
      validation:
        openAPIV3Schema:
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowedregistries
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not starts_with_allowed(container.image)
          msg := sprintf("Image '%v' is not from an allowed registry", [container.image])
        }
        starts_with_allowed(image) {
          allowed := input.parameters.registries[_]
          startswith(image, allowed)
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRegistries
metadata:
  name: allowed-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    registries:
      - "registry.example.com/"
      - "gcr.io/my-project/"
```

[↑ Back to TOC](#table-of-contents)

---

## Network Security

### Kubernetes NetworkPolicy

NetworkPolicy provides micro-segmentation inside Kubernetes. By default, all pods can communicate with all other pods. A NetworkPolicy restricts this.

```yaml
# Default deny all ingress and egress in 'production' namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # Applies to all pods
  policyTypes:
    - Ingress
    - Egress
```

```yaml
# Allow API pods to receive traffic only from frontend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    # From frontend pods in same namespace
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

    # From ingress controller (nginx namespace)
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
```

```yaml
# Allow API pods to reach only the database and DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    # To database pods
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # DNS resolution
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### Firewall rules on Linux (firewalld)

```bash
# Allow specific port
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=9100/tcp   # node_exporter (monitoring only)
firewall-cmd --reload

# Allow source IP range
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" port port="22" protocol="tcp" accept'

# Drop everything else
firewall-cmd --set-default-zone=drop

# List rules
firewall-cmd --list-all
```

### TLS best practices

```bash
# Check TLS certificate details
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -text | grep -E "Not (Before|After)|Subject:"

# Check for weak ciphers (using testssl.sh)
./testssl.sh --severity HIGH --quiet https://example.com

# Generate strong DH parameters
openssl dhparam -out /etc/ssl/dhparam.pem 4096

# Nginx TLS hardening
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:10m;
add_header Strict-Transport-Security "max-age=63072000" always;
```

[↑ Back to TOC](#table-of-contents)

---

## Cloud Security Fundamentals

### Shared Responsibility Model

```
PROVIDER RESPONSIBILITY:
  Physical security, hardware, hypervisor, managed service security

CUSTOMER RESPONSIBILITY:
  ┌─────────────────────────────────────────────────────────────────┐
  │ Data                                                            │
  │ Applications                                                    │
  │ OS configuration & patching (IaaS)                              │
  │ IAM / access controls                                           │
  │ Network configuration (security groups, NACLs)                  │
  │ Encryption (at rest + in transit)                               │
  └─────────────────────────────────────────────────────────────────┘
```

### AWS Security essentials

```bash
# Enable CloudTrail (API audit logging) in all regions
aws cloudtrail create-trail \
  --name global-trail \
  --s3-bucket-name my-cloudtrail-bucket \
  --is-multi-region-trail \
  --include-global-service-events

# Enable GuardDuty (threat detection)
aws guardduty create-detector --enable

# Enable Security Hub (aggregate findings)
aws securityhub enable-security-hub

# Check for public S3 buckets
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  xargs -I {} aws s3api get-bucket-acl --bucket {}

# Block all public access for a bucket
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Enable S3 bucket versioning and encryption
aws s3api put-bucket-versioning --bucket my-bucket \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket my-bucket \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms"}}]}'
```

### CIS Benchmarks

CIS (Center for Internet Security) Benchmarks provide security hardening guidelines for:
- Linux (CIS RHEL 9)
- Docker
- Kubernetes
- AWS / Azure / GCP

```bash
# Run CIS Kubernetes benchmark
# (kube-bench by Aqua Security)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -l app=kube-bench

# CIS Docker benchmark
docker run --rm --net host --pid host \
  -v /etc:/etc:ro -v /var/lib/docker:/var/lib/docker:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
```

[↑ Back to TOC](#table-of-contents)

---

## Security in CI/CD Pipelines

### Security pipeline stages

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURITY IN CI/CD                                │
│                                                                     │
│  Commit   → Pre-commit hooks (secret detection, linting)           │
│  Build    → SAST (Semgrep, Bandit)                                  │
│           → SCA (Trivy fs, Snyk)                                    │
│  Image    → Container scan (Trivy image, Grype)                     │
│           → Image signing (Cosign)                                  │
│  Deploy   → IaC scan (Trivy config, Checkov, tfsec)                 │
│           → DAST on staging (ZAP, Nuclei)                           │
│  Runtime  → Runtime security (Falco)                                │
│           → Continuous CVE monitoring (Trivy operator)              │
└─────────────────────────────────────────────────────────────────────┘
```

### Pre-commit hooks — prevent secrets from being committed

```bash
# Install pre-commit
pip install pre-commit

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
      - id: check-merge-conflict
      - id: trailing-whitespace

  - repo: https://github.com/bridgecrewio/checkov
    rev: 3.2.0
    hooks:
      - id: checkov
        args: ['--framework', 'dockerfile', '--framework', 'kubernetes']
```

```bash
pre-commit install     # Install hooks into .git/hooks/
pre-commit run --all-files  # Run manually
```

### Full security pipeline — GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # Full history for Gitleaks
      - name: Gitleaks — detect secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Semgrep SAST
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit p/owasp-top-ten

  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  container-scan:
    runs-on: ubuntu-latest
    needs: [sast, sca]
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          severity: 'CRITICAL'
          exit-code: '1'
      - name: Sign image with Cosign
        if: github.ref == 'refs/heads/main'
        uses: sigstore/cosign-installer@v3
        # ...

  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Checkov — IaC scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform/
          framework: terraform
          soft_fail: false
```

### Falco — Runtime security for Kubernetes

Falco detects anomalous activity in running containers (syscall-level).

```yaml
# Example Falco rule — detect shell in container
- rule: Terminal shell in container
  desc: A shell was spawned inside a container
  condition: >
    evt.type = execve and
    evt.dir = < and
    container and
    shell_procs and
    not proc.pname in (shell_procs)
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    image=%container.image.repository:%container.image.tag
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```

```bash
# Install Falco via Helm
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=ebpf
```

[↑ Back to TOC](#table-of-contents)

---

## Compliance & Audit Frameworks

| Framework | Industry | Key Controls |
|-----------|----------|--------------|
| **SOC 2 Type II** | Tech/SaaS | Access control, encryption, availability, monitoring |
| **PCI-DSS** | Payments | Cardholder data protection, network segmentation, logging |
| **HIPAA** | Healthcare | PHI protection, audit logs, access control |
| **ISO 27001** | All industries | Information security management system (ISMS) |
| **GDPR** | EU data subjects | PII protection, right to erasure, data residency |
| **NIST CSF** | US Government | Identify, Protect, Detect, Respond, Recover |

### DevOps controls for compliance

```
✅ Infrastructure as Code — auditable, version-controlled changes
✅ Immutable infrastructure — no SSH to prod, changes via pipeline only
✅ Signed commits and image signing — non-repudiation
✅ Centralized logging with immutable storage — audit trail
✅ Automated security scanning in CI/CD — documented evidence
✅ RBAC with MFA — access control
✅ Secrets management (Vault) — no plaintext credentials
✅ Automated patching cadence — vulnerability management
✅ Disaster recovery tests — documented and scheduled
```

[↑ Back to TOC](#table-of-contents)

---

## Runtime Security with Falco

SAST, DAST, and image scanning are all **pre-runtime** checks. They cannot catch what happens after a container starts — a process spawning a shell, an unexpected outbound connection, or a file in `/etc` being modified. **Falco** fills this gap by monitoring kernel system calls at runtime and alerting on anomalous behaviour.

### What Falco detects

| Category | Example rule |
|---|---|
| **Privilege escalation** | Container running as root when it shouldn't |
| **Shell spawned in container** | `bash` or `sh` started in a running container |
| **Filesystem tampering** | Write to `/etc`, `/usr`, or `/bin` in a running container |
| **Sensitive file read** | `/etc/shadow`, `/etc/kubernetes/admin.conf` accessed |
| **Outbound network connection** | Unexpected connection to an external IP |
| **Crypto mining signals** | High CPU + network + specific binary patterns |

### Install Falco on Kubernetes (Helm)

```bash
# Add the Falco Helm repository
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# Install Falco as a DaemonSet (runs on every node)
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set tty=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true

# Verify Falco pods are running on all nodes
kubectl get pods -n falco -o wide

# Check Falco is detecting events
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50
```

### Understanding Falco rules

Falco rules are written in YAML and describe syscall patterns to watch for:

```yaml
# /etc/falco/falco_rules.yaml (excerpt — this is built-in)
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint or is running inside a container
  condition: >
    spawned_process
    and container
    and shell_procs
    and proc.tty != 0
    and not container_entrypoint
  output: >
    A shell was spawned in a container with an attached terminal
    (user=%user.name %container.info shell=%proc.name parent=%proc.pname
    cmdline=%proc.cmdline)
  priority: NOTICE
  tags: [container, shell, mitre_execution]
```

### Write a custom Falco rule

```yaml
# custom-rules.yaml — alert on any write to /etc inside a container
- rule: Write to sensitive directory in container
  desc: Detect any file write to /etc inside a running container
  condition: >
    open_write
    and container
    and fd.directory startswith /etc
    and not proc.name in (package_mgmt_binaries)
  output: >
    Sensitive directory write in container
    (user=%user.name command=%proc.cmdline file=%fd.name container=%container.name image=%container.image.repository)
  priority: WARNING
  tags: [container, filesystem]
```

```bash
# Apply a custom rules file
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set-file falco.rulesFile[0]=/path/to/custom-rules.yaml
```

### Route Falco alerts to Slack

Falco has a companion tool called **Falcosidekick** that routes alerts to Slack, PagerDuty, Elasticsearch, and more.

```bash
# Install with Slack output enabled
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/XXX/YYY/ZZZ" \
  --set falcosidekick.config.slack.minimumpriority="warning"
```

### Trigger a test alert

```bash
# Manually trigger a Falco alert by spawning a shell in a container
kubectl exec -it $(kubectl get pod -l app=api -o name | head -1) -- /bin/sh

# In another terminal, watch Falco logs for the detection
kubectl logs -n falco -l app.kubernetes.io/name=falco -f | grep "shell was spawned"
```

### Falco in the CI/CD pipeline (falco-event-generator)

```bash
# Run the Falco event generator to test your rules are working
kubectl apply -f https://raw.githubusercontent.com/falcosecurity/event-generator/main/deployment/event-generator.yaml

# It performs common attack patterns — Falco should fire alerts for all of them
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

```bash
# Vault CLI
vault status
vault kv get secret/myapp/config
vault kv put secret/myapp/config key=value
vault policy list
vault auth list
vault audit list
vault token lookup

# Trivy
trivy image --severity CRITICAL,HIGH nginx:latest
trivy fs --scanners vuln,secret,misconfig .
trivy config terraform/
trivy image --format cyclonedx --output sbom.json myapp:latest

# Semgrep
semgrep --config=p/security-audit ./src
semgrep --config=p/secrets ./src

# OPA / Gatekeeper
kubectl get constrainttemplates
kubectl get constraints
kubectl describe K8sRequiredLabels namespace-required-labels

# kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -l app=kube-bench

# Cosign
cosign sign --key cosign.key registry.example.com/myapp:1.2.3
cosign verify --key cosign.pub registry.example.com/myapp:1.2.3

# Falco
helm install falco falcosecurity/falco -n falco --create-namespace
kubectl get pods -n falco -o wide
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50
kubectl -n falco logs -l app.kubernetes.io/name=falco | grep WARNING
```

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 1 — Install Vault and Work with Secrets (Beginner)

```bash
# Start Vault dev server
vault server -dev &
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

# Store an API key
vault kv put secret/myapp/api api_key="sk_live_abc123" api_url="https://api.example.com"

# Read it back
vault kv get secret/myapp/api
vault kv get -field=api_key secret/myapp/api

# Create a read-only policy
cat > readonly.hcl << 'EOF'
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
EOF
vault policy write myapp-readonly readonly.hcl

# Create a token with that policy
vault token create -policy=myapp-readonly

# Try to write with the new token (should fail)
export VAULT_TOKEN=<new-token>
vault kv put secret/myapp/api api_key="new_key"  # Permission denied
```

---

### Lab 2 — Scan a Container Image for CVEs (Beginner)

```bash
# Pull an older, vulnerable image
docker pull python:3.8

# Scan it
trivy image python:3.8 --severity CRITICAL,HIGH

# Compare with a newer version
trivy image python:3.12-slim

# Scan your own application image
docker build -t myapp:lab .
trivy image myapp:lab

# Generate SBOM
trivy image --format cyclonedx --output sbom.json myapp:lab
cat sbom.json | jq '.components | length'  # Count dependencies
```

---

### Lab 3 — Kubernetes RBAC (Intermediate)

```bash
# Create a test namespace and ServiceAccount
kubectl create namespace rbac-lab
kubectl create serviceaccount dev-user -n rbac-lab

# Create a Role (read-only on pods)
kubectl apply -f - << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-lab
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
EOF

# Bind the role to the ServiceAccount
kubectl create rolebinding dev-user-pod-reader \
  --role=pod-reader \
  --serviceaccount=rbac-lab:dev-user \
  -n rbac-lab

# Test: can dev-user list pods?
kubectl auth can-i list pods \
  --as=system:serviceaccount:rbac-lab:dev-user \
  -n rbac-lab
# → yes

# Test: can dev-user delete pods?
kubectl auth can-i delete pods \
  --as=system:serviceaccount:rbac-lab:dev-user \
  -n rbac-lab
# → no
```

---

### Lab 4 — Gatekeeper Policy (Intermediate)

```bash
# Install Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.15.0/deploy/gatekeeper.yaml

# Apply the RequireRunAsNonRoot ConstraintTemplate from this module
kubectl apply -f constraint-template.yaml
kubectl apply -f constraint.yaml

# Try to deploy a root container (should be denied)
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: root-test
  namespace: production
spec:
  containers:
    - name: nginx
      image: nginx:latest
      # Missing: securityContext.runAsNonRoot: true
EOF
# → Error: container 'nginx' must set runAsNonRoot=true

# Deploy a compliant pod (should succeed)
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-test
  namespace: production
spec:
  containers:
    - name: nginx
      image: nginx:latest
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
EOF
```

---

## Further Reading

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [OPA / Gatekeeper Documentation](https://open-policy-agent.github.io/gatekeeper/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Semgrep Rules Registry](https://semgrep.dev/r)
- [Falco Documentation](https://falco.org/docs/)
- [Falcosidekick — Alert Routing](https://github.com/falcosecurity/falcosidekick)
- [Falco Rules Hub](https://falco.org/docs/reference/rules/)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [Sigstore / Cosign](https://docs.sigstore.dev/cosign/overview/)
- [SLSA Supply Chain Security](https://slsa.dev/)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [Creative Commons BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution.*
