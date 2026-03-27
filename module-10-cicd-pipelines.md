# Module 10: CI/CD Pipelines

> Part of the [DevOps Career Course](./README.md) by UncleJS

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 10 of 15](https://img.shields.io/badge/module-10%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Intermediate%20%E2%86%92%20Advanced-red) ![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-stable-2088FF?logo=githubactions&logoColor=white) ![GitLab CI](https://img.shields.io/badge/GitLab%20CI-stable-FC6D26?logo=gitlab&logoColor=white) ![Jenkins 2.479+](https://img.shields.io/badge/Jenkins-2.479%2B%20LTS-D24939?logo=jenkins&logoColor=white) ![Linux · Docker agents](https://img.shields.io/badge/agents-Linux%20%C2%B7%20Docker-grey?logo=docker&logoColor=white)

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: CI/CD Fundamentals](#beginner-cicd-fundamentals)
- [Beginner: Deployment Strategies](#beginner-deployment-strategies)
- [Beginner: Platform Comparison — GitHub Actions vs GitLab CI vs Jenkins](#beginner-platform-comparison--github-actions-vs-gitlab-ci-vs-jenkins)
- [Intermediate: GitHub Actions](#intermediate-github-actions)
- [Intermediate: GitLab CI/CD](#intermediate-gitlab-cicd)
- [Intermediate: Jenkins](#intermediate-jenkins)
- [Intermediate: Pipeline Design Patterns](#intermediate-pipeline-design-patterns)
- [Intermediate: Secrets Management in Pipelines](#intermediate-secrets-management-in-pipelines)
- [Intermediate: Artifact Management](#intermediate-artifact-management)
- [Intermediate: CI/CD for Containers & Kubernetes](#intermediate-cicd-for-containers--kubernetes)
- [Advanced: Reusable Workflows & DRY Pipelines](#advanced-reusable-workflows--dry-pipelines)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

CI/CD (Continuous Integration / Continuous Delivery/Deployment) is the engine of modern DevOps. It automates the journey from a developer pushing code to that code running in production — testing, building, scanning, and deploying at every step.

This module covers three industry-standard CI/CD tools: **GitHub Actions** (the most widely adopted modern tool), **GitLab CI/CD** (built into GitLab, strong DevSecOps features), and **Jenkins** (the battle-tested enterprise classic).

```mermaid
flowchart LR
    PUSH["Code push<br/>(git push)"]
    TRIG["Trigger<br/>(webhook)"]
    BUILD["Build<br/>(compile / install deps)"]
    TEST["Test<br/>(unit + integration)"]
    SCAN["Security scan<br/>(SAST + dependency audit)"]
    PKG["Build artifact<br/>(container image)"]
    STAGING["Deploy to staging<br/>(automatic)"]
    INTTEST["Integration tests<br/>(smoke + E2E)"]
    PROD["Deploy to production<br/>(manual gate or auto)"]

    PUSH --> TRIG --> BUILD --> TEST --> SCAN --> PKG --> STAGING --> INTTEST --> PROD
```

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain CI/CD concepts and the value they provide
- Choose the right deployment strategy for a given scenario
- Write GitHub Actions workflows that build, test, and deploy code
- Write GitLab CI/CD pipelines with stages and jobs
- Configure Jenkins pipelines using declarative Jenkinsfile syntax
- Store and use secrets securely in all three platforms
- Build and push container images through a pipeline
- Deploy to Kubernetes from a CI/CD pipeline
- Design reusable, DRY pipelines using GitHub Actions reusable workflows and GitLab CI includes
- Compare GitHub Actions, GitLab CI/CD, and Jenkins across key dimensions to choose the right tool

[↑ Back to TOC](#table-of-contents)

---

## Beginner: CI/CD Fundamentals

CI/CD is the operational heart of DevOps — the system that closes the loop between writing code and running code in front of users. Before CI/CD, teams integrated infrequently, which meant integration problems accumulated and became painful to untangle. Before CD, deployment was a high-ceremony event requiring careful coordination, usually resulting in infrequent releases that bundled many changes and made root-cause analysis difficult when something went wrong.

The three terms represent increasing levels of automation. **Continuous Integration** is a practice: merge code frequently (at least daily), and have every merge trigger an automated build and test run. The benefit is not the tooling — it is the discipline of never letting branches diverge long enough for integration to become hard. **Continuous Delivery** is a capability: the main branch should always be in a state that could be deployed to production. That requires disciplined testing, environment parity, and deployment automation, but the final production deployment still requires a human decision. **Continuous Deployment** is a policy: every passing change is automatically deployed to production with no human gate. This is the most aggressive form and requires the highest level of automated quality coverage and observability.

CI/CD pipelines fail, and designing for recovery is as important as designing for success. A pipeline that breaks and is not immediately fixed is a broken development workflow. The most common failure modes are flaky tests (intermittent failures that erode trust in the pipeline signal), slow pipelines (encouraging developers to skip running them), credential expiry (a midnight production deployment blocked by an expired token), and configuration drift between environments (staging passes but production breaks). Designing for recovery means: make tests deterministic, parallelize aggressively to keep pipelines under ten minutes, rotate credentials before they expire, and keep environment configuration as code.

```mermaid
flowchart LR
    COMMIT["Commit to<br/>feature branch"]
    CI["CI: build + test + scan"]
    ART["artifact<br/>(tagged image)"]
    STAGING["CD Delivery:<br/>deploy to staging"]
    GATE["approval gate<br/>(human review)"]
    PROD["CD Deployment:<br/>automatic deploy to prod"]

    COMMIT --> CI --> ART --> STAGING --> GATE --> PROD
```

CI/CD is really about reducing the amount of uncertainty between writing code and running code in production. Without a pipeline, teams rely on memory, handoffs, and manual repetition: someone builds locally, someone else runs tests later, and deployment becomes a special event that feels risky every time. A good pipeline turns that fragile sequence into a repeatable system that checks quality continuously and makes releases boring in the best possible way.

As you read this module, try to connect each stage to a concrete failure it prevents. CI catches integration problems before they pile up. CD shortens the distance between a passing change and a working deployment. Together, they create feedback loops that are fast enough for developers and reliable enough for operations. The tools differ, but that operating model is the constant.

### What is CI?

**Continuous Integration**: Every code push triggers an automated pipeline that builds and tests the code. Problems are caught immediately — not weeks later when merging becomes painful.

```
Developer pushes code
       ↓
CI Pipeline triggers automatically
       ↓
Build → Unit Tests → Integration Tests → Code Quality → Security Scan
       ↓
Pass: merge allowed ✓
Fail: PR blocked ✗
```

### What is CD?

**Continuous Delivery**: Every passing build is automatically deployed to staging. Deployment to production requires a manual approval step.

**Continuous Deployment**: Every passing build is automatically deployed all the way to production — no human in the loop.

### Pipeline Stages

```
Code Push → Build → Test → Scan → Package → Deploy to Staging → Approve → Deploy to Production
```

| Stage | What Happens |
|---|---|
| **Build** | Compile code, install dependencies, create artifact |
| **Unit Test** | Fast, isolated tests (milliseconds each) |
| **Integration Test** | Test interactions between components |
| **Code Quality** | Lint, style checks, complexity analysis |
| **Security Scan** | SAST, dependency vulnerability scan |
| **Package** | Build container image, create deployment artifact |
| **Deploy Staging** | Automatic deployment to staging environment |
| **Smoke Test** | Basic health check against deployed app |
| **Deploy Production** | Manual or automatic deployment to production |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Deployment Strategies

Once a pipeline can build and test software, the next challenge is changing production safely. That is where deployment strategy matters. The question is not just "how do we push a new version?" but "how do we limit blast radius, observe behavior, and recover quickly if the release is bad?" Different strategies make different tradeoffs between infrastructure cost, rollout speed, complexity, and rollback safety.

**Blue-green** deployment has a zero-downtime cutover and instant rollback — you simply flip traffic back to the old environment. The cost is that you need double the production infrastructure running simultaneously during the deployment window, and both environments must have access to shared state (databases, caches) in a compatible way. **Rolling updates** are cheaper because you replace instances gradually without duplicating the entire fleet. The tradeoff is that you briefly run two versions simultaneously, which requires backward-compatible APIs and database schemas during the transition period. **Canary deployments** are the most conservative strategy: you route a small percentage of real production traffic (5–10%) to the new version, observe metrics and error rates, and only promote the new version to full traffic if it looks healthy. Canaries catch version-specific issues that staging environments cannot reproduce, but they require traffic-splitting infrastructure (usually a load balancer or service mesh) and solid observability to know when to promote or roll back.

The right strategy depends on context. A simple internal API might tolerate rolling updates with a brief mixed-version window. A payment processing service should probably use canaries with automatic rollback triggers on error rate increases. Blue-green is worth the cost when you need zero downtime and you have the budget for duplicate environments — common in larger organizations where the infrastructure cost is smaller than the business cost of a failed deployment.

```mermaid
flowchart LR
    subgraph "Blue-Green"
        BG_OLD["Blue<br/>v1.0 (live)"]
        BG_NEW["Green<br/>v1.1 (deployed)"]
        BG_SW["switch traffic<br/>(instant rollback available)"]
        BG_OLD -->|"deploy new version"| BG_NEW
        BG_NEW --> BG_SW
    end
    subgraph "Rolling"
        R1["v1.0 x10"]
        R2["v1.0 x8<br/>v1.1 x2"]
        R3["v1.1 x10"]
        R1 -->|"replace gradually"| R2 -->|"complete"| R3
    end
    subgraph "Canary"
        C_MAIN["v1.0<br/>95% traffic"]
        C_CAN["v1.1 canary<br/>5% traffic"]
        C_MON["monitor metrics"]
        C_PROM["promote to 100%<br/>or rollback"]
        C_MAIN & C_CAN --> C_MON --> C_PROM
    end
```

This is why deployment strategy should never be chosen in isolation from your platform. A small internal service might be perfectly fine with rolling updates, while a customer-facing payment service might justify canaries or blue-green cutovers. The right choice depends on risk tolerance, traffic patterns, observability maturity, and how expensive downtime is for the business.

### Blue-Green Deployment

```
Traffic → Blue (v1.0)     ←── Current production
           Green (v1.1)   ←── Deploy new version here, test
                          ←── Switch traffic: Traffic → Green
                          ←── Blue is standby for instant rollback
```

- ✅ Zero downtime, instant rollback
- ❌ Requires 2x infrastructure

### Rolling Update

```
10 pods running v1.0
Replace 1 pod at a time with v1.1:
[v1.0 × 9, v1.1 × 1] → [v1.0 × 8, v1.1 × 2] → ... → [v1.1 × 10]
```

- ✅ No extra infrastructure, gradual rollout
- ❌ Briefly runs two versions simultaneously, slower rollback

### Canary Deployment

```
100% traffic → v1.0
       ↓
5% traffic → v1.1  (canary)  ← Monitor metrics
95% traffic → v1.0
       ↓
If stable: 20% → v1.1, 80% → v1.0
       ↓
If stable: 100% → v1.1
```

- ✅ Real-world testing with minimal blast radius
- ❌ Complex traffic routing required

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Platform Comparison — GitHub Actions vs GitLab CI vs Jenkins

Before diving into each tool, here's a decision-oriented comparison to help you choose the right platform.

Most pipeline tools can be made to perform the same core jobs: build, test, package, and deploy. The real differences show up in workflow friction, governance model, ecosystem fit, and operational overhead. That is why tool choice should be driven less by feature checklists and more by context: where your source code lives, how much infrastructure you are willing to maintain, and whether you need deep enterprise customization or fast developer onboarding.

It is also worth remembering that switching pipeline platforms is usually more expensive than it looks. You are not just rewriting YAML or Groovy; you are rebuilding credentials flows, runner strategy, approval gates, logging, artifact storage, and team habits. Choose a platform that matches the shape of your organization so the pipeline reinforces delivery instead of fighting it.

| Dimension | GitHub Actions | GitLab CI/CD | Jenkins |
|---|---|---|---|
| **Hosting** | Cloud (GitHub-managed) | Cloud or self-hosted | Self-hosted only |
| **Config file** | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `Jenkinsfile` |
| **Setup time** | Minutes (no install) | Minutes (no install) | Hours (server + plugins) |
| **Runners** | GitHub-hosted or self-hosted | GitLab-hosted or self-hosted | Your own servers |
| **Secrets management** | Repository/org secrets | CI/CD Variables (masked) | Credentials Store |
| **Built-in registry** | GitHub Container Registry | GitLab Container Registry | No (plugin required) |
| **Built-in SAST** | Via Marketplace actions | Native (GitLab Ultimate) | Via plugins |
| **Pricing** | Free tier + pay-per-minute | Free tier + pay-per-minute | Free (infra costs only) |
| **Best for** | GitHub-hosted projects | End-to-end GitLab DevSecOps | Enterprise / complex custom needs |

### When to choose which

```
Is your code on GitHub?
  └── YES → GitHub Actions (zero friction, massive Marketplace)

Is your code on GitLab?
  └── YES → GitLab CI/CD (native integration, free SAST/DAST)

Do you need full on-premises control, custom agents, or complex enterprise integrations?
  └── YES → Jenkins (most flexible, most work to maintain)

Multi-cloud or platform-agnostic pipelines?
  └── Consider a dedicated tool: Tekton, Argo Workflows, or Buildkite
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: GitHub Actions

GitHub Actions is built directly into GitHub. Workflows are YAML files stored in `.github/workflows/`.

GitHub Actions is popular because it removes most of the startup friction. If your code already lives on GitHub, the CI/CD system is effectively waiting next to it. That tight integration is valuable for smaller teams and fast-moving projects because pull requests, secrets, environments, reusable actions, and status checks all live close to the source of truth. The result is less glue code and fewer moving parts to maintain at the start.

The tradeoff is that convenience can encourage pipelines to grow organically without much design. A workflow that starts as ten lines of YAML can become an unreviewed platform in its own right. As you read the examples below, pay attention not just to syntax but to job boundaries, permissions, artifact flow, and deployment gates. Those are the details that separate a useful workflow from a fragile one.

### Core Concepts

| Term | Description |
|---|---|
| **Workflow** | Automated process defined in a YAML file |
| **Event** | Trigger that starts a workflow (`push`, `pull_request`, `schedule`, etc.) |
| **Job** | A set of steps that run on the same runner |
| **Step** | A single task — either a shell command or an Action |
| **Action** | A reusable unit of code from the GitHub Marketplace |
| **Runner** | The virtual machine that executes jobs |

### Basic Workflow

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()                          # Run even if tests fail
        with:
          name: test-results
          path: coverage/
```

### Build & Push Docker Image

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Deploy to Kubernetes

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  workflow_run:
    workflows: ["Build and Push Docker Image"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment: production          # Requires manual approval

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBECONFIG }}

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            app=ghcr.io/${{ github.repository }}:sha-${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=5m

      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/myapp
```

### Matrix Builds

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: ['18', '20', '21']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

### GitHub Actions: Event Model

Understanding the event model is essential to writing workflows that do the right thing at the right time. The `on:` key accepts a wide range of triggers: `push`, `pull_request`, `schedule`, `workflow_dispatch`, `workflow_call`, `release`, and more. These triggers are not equivalent. A `push` trigger fires on every direct commit to a branch — useful for building but dangerous for production deployments. A `workflow_dispatch` trigger fires only when manually triggered via the GitHub UI or API, making it the right choice for operational actions that should never run automatically. A `pull_request` trigger fires when a PR is opened, updated, or synchronized, and its workflows run in the context of the PR's head commit — meaning they do not have access to repository secrets from the base branch by default, which is a security boundary worth understanding.

Event filters narrow triggers further. You can restrict a `push` trigger to specific branches with `branches:` or to specific file paths with `paths:`. A workflow that rebuilds documentation only when docs change, or that runs production deployment only on `main`, is using event filters to remove noise and prevent unintended side effects. Without filters, pipelines fire on every commit to every branch, wasting runner minutes and creating alert fatigue.

Matrix builds are one of the most efficient features of GitHub Actions. When you specify a `strategy.matrix`, GitHub Actions fans out the job into N parallel runs, one per matrix value combination. A 3-node × 2-OS matrix produces 6 concurrent jobs without any extra workflow logic. This is the cleanest way to validate cross-platform or cross-version compatibility without serializing test runs. Matrix jobs share the same workflow YAML but each run in complete isolation on its own runner, so failures in one combination do not block the others unless you set `fail-fast: true`.

```mermaid
flowchart LR
    PUSH["push event<br/>(branch filter: main)"]
    PR["pull_request event<br/>(types: opened, synchronize)"]
    SCHED["schedule event<br/>(cron: 0 2 * * *)"]
    DISPATCH["workflow_dispatch<br/>(manual trigger)"]
    WCALL["workflow_call<br/>(called by another workflow)"]

    PUSH --> JOB_CI["CI job<br/>(build + test)"]
    PR --> JOB_CI
    SCHED --> JOB_NIGHTLY["nightly job<br/>(full regression suite)"]
    DISPATCH --> JOB_DEPLOY["deploy job<br/>(production gate)"]
    WCALL --> JOB_REUSABLE["reusable job<br/>(shared across repos)"]

    JOB_CI --> MATRIX["matrix strategy<br/>(Node 18 / 20 / 21<br/>× ubuntu / macos)"]
    MATRIX --> RESULTS["parallel results<br/>(6 concurrent runs)"]
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: GitLab CI/CD

GitLab CI/CD uses a `.gitlab-ci.yml` file at the repository root.

GitLab CI/CD is strongest when a team wants a single platform for source control, pipelines, package registry, environments, and parts of the security workflow. That integrated model can simplify operations because fewer external systems need to be stitched together. It also encourages teams to think of delivery as an end-to-end value stream rather than as isolated scripts triggered by commits.

The downside is that the pipeline file can become dense quickly as more concerns accumulate: build logic, caching, artifacts, security scans, deploy jobs, environment rules, and reusable templates all converge in one place. For that reason, strong GitLab pipelines usually invest early in stage design, templating, and naming discipline so the configuration remains understandable months later.

The GitLab CI/CD model is stage-first. You declare an ordered list of stages — `build`, `test`, `scan`, `package`, `deploy` — and then assign jobs to each stage. All jobs within a stage run in parallel by default. Jobs in later stages only start after all jobs in the preceding stage have succeeded. This is an important difference from GitHub Actions, where job dependencies are expressed job-by-job with `needs:`. In GitLab, stage order is the default dependency model, and `needs:` is an opt-in override that activates DAG (Directed Acyclic Graph) mode, allowing a job to start as soon as its direct dependencies finish rather than waiting for the entire previous stage.

The integrated platform advantage is most visible in security workflows. GitLab can run SAST, dependency scanning, container scanning, and secret detection as first-class pipeline jobs without requiring external integrations. Results show up natively in merge request pipelines, environments, and compliance dashboards. For teams with regulatory requirements — auditing, separation of duties, security approvals — that built-in traceability can significantly reduce the effort of demonstrating compliance compared to assembling the same capabilities from separate tools.

### Core Concepts

| Term | Description |
|---|---|
| **Pipeline** | The entire CI/CD process for a commit |
| **Stage** | A phase of the pipeline (build, test, deploy) |
| **Job** | A task that runs in a stage |
| **Runner** | The machine that executes jobs |
| **Artifact** | Files passed between jobs |
| **Cache** | Files cached between pipelines (e.g., node_modules) |

### Full Pipeline Example

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - package
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  KUBECONFIG_PATH: /tmp/kubeconfig

# Global cache for all jobs
cache:
  key: $CI_COMMIT_REF_SLUG
  paths:
    - node_modules/

# ─── Build Stage ───────────────────────────────────────────
build:
  stage: build
  image: node:20-alpine
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

# ─── Test Stage ────────────────────────────────────────────
unit-tests:
  stage: test
  image: node:20-alpine
  script:
    - npm ci
    - npm test -- --coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    when: always

lint:
  stage: test
  image: node:20-alpine
  script:
    - npm ci
    - npm run lint

# ─── Security Scan Stage ───────────────────────────────────
sast:
  stage: scan
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto --error --json > gl-sast-report.json
  artifacts:
    reports:
      sast: gl-sast-report.json

dependency-scan:
  stage: scan
  image: node:20-alpine
  script:
    - npm audit --audit-level=high

# ─── Package Stage ─────────────────────────────────────────
build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
    - docker tag $DOCKER_IMAGE $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

# ─── Deploy Stage ──────────────────────────────────────────
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - echo $KUBECONFIG_CONTENT | base64 -d > $KUBECONFIG_PATH
    - export KUBECONFIG=$KUBECONFIG_PATH
    - kubectl set image deployment/myapp app=$DOCKER_IMAGE -n staging
    - kubectl rollout status deployment/myapp -n staging
  only:
    - main

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://app.example.com
  when: manual                      # Require human approval
  script:
    - echo $KUBECONFIG_CONTENT | base64 -d > $KUBECONFIG_PATH
    - export KUBECONFIG=$KUBECONFIG_PATH
    - kubectl set image deployment/myapp app=$DOCKER_IMAGE -n production
    - kubectl rollout status deployment/myapp -n production
  only:
    - main
```

### Reusable Templates

```yaml
# Define a reusable template
.deploy-template: &deploy-template
  image: bitnami/kubectl:latest
  script:
    - echo $KUBECONFIG | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    - kubectl set image deployment/myapp app=$DOCKER_IMAGE -n $DEPLOY_NAMESPACE
    - kubectl rollout status deployment/myapp -n $DEPLOY_NAMESPACE

deploy-staging:
  <<: *deploy-template
  variables:
    DEPLOY_NAMESPACE: staging
  environment:
    name: staging

deploy-production:
  <<: *deploy-template
  variables:
    DEPLOY_NAMESPACE: production
  when: manual
  environment:
    name: production
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Jenkins

Jenkins is the battle-tested CI/CD workhorse — self-hosted, highly configurable, and found in virtually every enterprise. Pipelines are defined in a `Jenkinsfile`.

Jenkins remains common because many enterprises need a level of control that hosted platforms do not always provide. They may have private networks, custom agents, legacy build tools, approval requirements, or plugin-based integrations accumulated over many years. Jenkins can usually accommodate those needs, which is both its biggest strength and its biggest operational burden.

That burden matters. Running Jenkins means you are operating the CI/CD platform itself: controller availability, plugin compatibility, agent security, credential handling, upgrade cadence, and backup strategy all become your problem. Teams choose Jenkins successfully when that flexibility is worth the cost and when they have the maturity to maintain it well.

### Installation

```bash
# Ubuntu
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update && sudo apt install -y jenkins openjdk-17-jdk
sudo systemctl enable --now jenkins
# Access: http://localhost:8080
```

### Declarative Jenkinsfile

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any

    // Run on a specific agent label
    // agent { label 'docker-agent' }

    environment {
        DOCKER_IMAGE = "myapp:${env.BUILD_NUMBER}"
        REGISTRY     = "registry.example.com"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        pollSCM('H/5 * * * *')    // Poll Git every 5 minutes
        // cron('0 2 * * *')      // Or run on a schedule
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm test'
                    }
                    post {
                        always {
                            junit 'reports/junit.xml'
                        }
                    }
                }
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        def image = docker.build("${REGISTRY}/${DOCKER_IMAGE}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh "kubectl set image deployment/myapp app=${REGISTRY}/${DOCKER_IMAGE}"
                    sh "kubectl rollout status deployment/myapp --timeout=5m"
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    sh "kubectl set image deployment/myapp app=${REGISTRY}/${DOCKER_IMAGE} -n production"
                    sh "kubectl rollout status deployment/myapp -n production --timeout=10m"
                }
            }
        }
    }

    post {
        success {
            slackSend color: 'good', message: "✅ Build ${env.BUILD_NUMBER} succeeded: ${env.BUILD_URL}"
        }
        failure {
            slackSend color: 'danger', message: "❌ Build ${env.BUILD_NUMBER} FAILED: ${env.BUILD_URL}"
            emailext(
                subject: "FAILED: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                body: "Build failed. See: ${env.BUILD_URL}",
                to: 'devops@example.com'
            )
        }
        always {
            cleanWs()    // Clean workspace after build
        }
    }
}
```

The `post` block and `input` directive in the example above illustrate two Jenkins-specific strengths. `post` conditions (`success`, `failure`, `always`) allow notification and cleanup logic to be attached declaratively to any stage or the full pipeline, keeping side-effect logic out of the main `stages` block. The `input` directive pauses a pipeline and waits for a human to click an approval button before proceeding — a simple but powerful production gate that many teams value for operational safety.

Jenkins pipelines can also be visualized in the Blue Ocean UI as a stage graph, making it easy to see which stages passed, which failed, and how long each took. The stage view is one of the clearest representations of a CI/CD pipeline available in any tool, which is part of why Jenkins remains popular in organizations where pipeline transparency to non-engineers matters.

```mermaid
flowchart LR
    CHECKOUT["Checkout<br/>(checkout scm)"]
    BUILD["Build<br/>(npm ci + npm run build)"]
    UNIT["Unit Tests<br/>(parallel)"]
    LINT["Lint<br/>(parallel)"]
    DOCKER["Build Docker Image<br/>(docker.withRegistry)"]
    STAGING["Deploy to Staging<br/>(withKubeConfig)"]
    GATE["Input Gate<br/>(human approval)"]
    PROD["Deploy to Production<br/>(withKubeConfig)"]
    POST["Post<br/>(Slack notify + cleanWs)"]

    CHECKOUT --> BUILD --> UNIT & LINT --> DOCKER --> STAGING --> GATE --> PROD --> POST
```

### Jenkins Shared Libraries

```groovy
// vars/deployToKubernetes.groovy (in a shared library repo)
def call(Map config) {
    withKubeConfig([credentialsId: config.credentialsId]) {
        sh "kubectl set image deployment/${config.deployment} app=${config.image} -n ${config.namespace}"
        sh "kubectl rollout status deployment/${config.deployment} -n ${config.namespace} --timeout=5m"
    }
}

// Using the shared library in Jenkinsfile:
@Library('my-shared-library') _
deployToKubernetes(
    credentialsId: 'kubeconfig-staging',
    deployment: 'myapp',
    image: "${DOCKER_IMAGE}",
    namespace: 'staging'
)
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Pipeline Design Patterns

Once you understand individual tools, the next level is pipeline architecture. Design patterns matter because the same pipeline stages can support either fast, low-friction delivery or slow, approval-heavy chaos depending on how they are arranged. Good pipeline design aligns with your branching model, keeps feedback loops short, and reserves slower checks for the points where they create the most value.

This is also where platform engineering thinking starts to emerge. Instead of asking only "how do we run tests," you start asking "what is the shortest trustworthy path from commit to customer impact?" The answer shapes everything from branch policy to artifact promotion to whether production changes happen by merge, by tag, or by GitOps reconciliation.

### The Trunk-Based Pipeline

```
push to main
    │
    ▼
build + test (< 5 min)
    │
    ▼
build image + push to registry
    │
    ▼
auto-deploy to staging
    │
    ▼
smoke tests
    │
    ▼
manual gate → deploy to production
```

### Branching Strategy Alignment

| Branching Model | CI/CD Behavior |
|---|---|
| **GitHub Flow** | PR → run tests; merge to main → deploy staging; manual → prod |
| **Git Flow** | feature branch → test; develop → deploy dev; release → deploy staging; main → prod |
| **Trunk-Based** | Every commit to main → full pipeline → auto-deploy |

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Secrets Management in Pipelines

Never hardcode credentials in pipeline files. Always use the platform's secrets store.

Pipelines are high-value targets because they sit at the intersection of source code, package registries, deployment credentials, and production automation. A leaked secret in CI/CD is often more damaging than a leaked local developer credential because the pipeline usually has broad, automated access to build and deploy systems. That is why secret handling must be treated as a first-class design concern rather than a checkbox.

Good pipeline security is about minimizing exposure, not just hiding values. Use short-lived credentials where possible, scope secrets to environments, restrict who can trigger sensitive jobs, and make sure logs do not accidentally print secret material. The platform examples below show the storage mechanisms, but the operational principle is least privilege plus careful auditability.

### GitHub Actions

```yaml
# Access repository secrets
steps:
  - name: Deploy
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      API_KEY: ${{ secrets.PRODUCTION_API_KEY }}
    run: ./deploy.sh

# Secrets set in: Settings → Secrets and variables → Actions
```

### GitLab CI/CD

```yaml
# Access CI/CD variables
deploy:
  script:
    - echo $DATABASE_URL | kubectl create secret generic db --from-literal=url=$DATABASE_URL
# Variables set in: Settings → CI/CD → Variables (masked + protected)
```

### Jenkins

```groovy
// Using credentials binding
withCredentials([
    string(credentialsId: 'api-key', variable: 'API_KEY'),
    usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')
]) {
    sh './deploy.sh'
}
// Credentials stored in: Manage Jenkins → Manage Credentials
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Artifact Management

Artifacts are the handoff point between pipeline stages, environments, and sometimes teams. Without a clear artifact strategy, later jobs rebuild what earlier jobs already produced, deployments become inconsistent, and rollback becomes harder because nobody can say exactly what was released. Treating artifacts as named, versioned outputs is what makes the pipeline reproducible instead of merely automated.

This is also one of the bridges between CI and CD. Builds create artifacts, but deployments should consume those exact artifacts rather than recomputing them under slightly different conditions. That distinction is subtle and important: promotion is safer when you are moving a known artifact forward, not re-running a build with production on the line.

```yaml
# GitHub Actions — upload build artifacts
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 7

# Download in a later job
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: dist/

# GitLab — pass artifacts between jobs
build:
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

test:
  needs: [build]      # Receive artifacts from build job
  script:
    - ls dist/         # Files are available here
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: CI/CD for Containers & Kubernetes

Container and Kubernetes delivery adds another layer to pipeline design because you are no longer shipping just application code. You are shipping an image, its metadata, its vulnerability posture, and a deployment action that changes a running cluster. That means the pipeline has to do more than compile and test. It has to produce an immutable artifact, verify it, publish it, and update the orchestration layer in a way that is observable and reversible.

This is where many teams discover the difference between deployment automation and delivery discipline. A pipeline that can run `kubectl set image` is not automatically safe. Safety comes from image tagging strategy, rollout verification, health checks, rollback paths, environment promotion rules, and a clear separation between build concerns and cluster concerns.

### Full Container CI/CD Pattern

```
1. Developer pushes code
2. Pipeline triggers
3. Build stage: npm install + npm test
4. Security: trivy scan, npm audit
5. Build Docker image
6. Scan image with Trivy
7. Push to registry with tag: sha-<commit>
8. Update Kubernetes deployment image
9. kubectl rollout status (wait for rollout)
10. Health check: curl /healthz
11. If failed: kubectl rollout undo
```

### GitOps Pattern (with ArgoCD)

```
1. Pipeline builds and pushes image: myapp:sha-abc123
2. Pipeline updates the Kubernetes manifest repo:
   - Opens a PR updating image tag in manifests/production/deployment.yaml
   - Or directly commits to a deployments repo
3. ArgoCD detects the manifest change in Git
4. ArgoCD syncs the cluster to match the new manifest
5. Deployment happens — zero manual kubectl commands
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Reusable Workflows & DRY Pipelines

As your organization grows, you'll find yourself copy-pasting pipeline YAML across dozens of repositories. Reusable workflows (GitHub Actions) and CI includes (GitLab) solve this.

Reuse becomes important the moment you have more than a handful of repositories doing similar things. Copy-paste feels fast initially, but it creates a silent maintenance tax: security fixes must be repeated everywhere, build logic drifts over time, and different teams end up with pipelines that look similar but behave differently in subtle ways. Shared workflow building blocks are a way to standardize delivery without forcing every application into the same monolithic pipeline.

The real challenge is balancing consistency with flexibility. Centralized workflows should encode the things that truly ought to be common, such as image build policy, test conventions, or signing steps. They should not erase every application-specific need. The healthiest pattern is usually a thin shared platform layer with clearly documented extension points.

In GitHub Actions, there are two distinct reuse mechanisms and they solve different problems. A **reusable workflow** (`workflow_call`) is a complete workflow file with its own jobs and runners. The caller triggers it as an entire job-level unit and passes in inputs and secrets. This is the right tool when you want to standardize a multi-step delivery process — build, scan, push — across many repos. A **composite action**, stored in `.github/actions/`, bundles a set of steps into a single named step. It runs on the caller's runner inside the caller's job rather than spinning up a new one. Composite actions are the right tool when you want to extract repetitive step sequences (like setup + install + cache) without the overhead of a separate job boundary.

The practical rule is: use composite actions to avoid repeating step-level boilerplate within a job; use reusable workflows to avoid repeating job-level (or multi-job) pipeline logic across repositories. The two can also nest: a reusable workflow can use composite actions internally, and the caller only sees the clean reusable workflow interface. This layering is how platform teams build meaningful abstraction without requiring application teams to understand the implementation details of the shared delivery infrastructure.

### GitHub Actions: Reusable Workflows

A reusable workflow lives in `.github/workflows/` and is called with `workflow_call`.

```yaml
# .github/workflows/_reusable-docker-build.yml  (in a central repo or the same repo)
# Prefix with _ by convention to distinguish reusable from triggered workflows
name: Reusable — Build & Push Docker Image

on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      registry:
        required: false
        type: string
        default: ghcr.io
    secrets:
      REGISTRY_TOKEN:
        required: true

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ inputs.registry }}
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ inputs.registry }}/${{ inputs.image-name }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Calling the reusable workflow from any repo:**

```yaml
# .github/workflows/ci.yml  (in your application repo)
name: CI

on:
  push:
    branches: [main]

jobs:
  build:
    uses: my-org/devops-workflows/.github/workflows/_reusable-docker-build.yml@main
    with:
      image-name: ${{ github.repository }}
    secrets:
      REGISTRY_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### GitHub Actions: Composite Actions

For reusing individual *steps* (not full jobs), use a composite action:

```yaml
# .github/actions/setup-node-cache/action.yml
name: Setup Node with cache
description: Install Node and restore npm cache

inputs:
  node-version:
    default: '20'

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm

    - shell: bash
      run: npm ci
```

```yaml
# Usage in any workflow:
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-node-cache    # local composite action
    with:
      node-version: '20'
  - run: npm test
```

### GitLab CI: YAML Includes

GitLab's `include:` keyword is the equivalent — pull shared templates from a central project:

```yaml
# .gitlab-ci.yml  (application repo)
include:
  - project: 'my-org/cicd-templates'
    ref: main
    file:
      - '/templates/docker-build.yml'
      - '/templates/security-scan.yml'

# Override variables without duplicating the job definition
variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

stages:
  - build
  - scan
  - deploy
```

```yaml
# cicd-templates/templates/docker-build.yml  (central templates repo)
.docker-build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main

docker-build:
  extends: .docker-build
```

### Jenkins: Shared Libraries

Jenkins calls its equivalent feature **Shared Libraries** — Groovy functions stored in a separate Git repo:

```
(shared-library repo)
├── vars/
│   ├── buildDockerImage.groovy
│   ├── deployToKubernetes.groovy
│   └── runSecurityScan.groovy
└── src/
    └── org/company/PipelineUtils.groovy
```

```groovy
// vars/buildDockerImage.groovy
def call(String imageName, String tag = env.BUILD_NUMBER) {
    docker.withRegistry('https://registry.example.com', 'registry-creds') {
        def img = docker.build("${imageName}:${tag}")
        img.push()
        img.push('latest')
    }
}
```

```groovy
// Jenkinsfile  (any project)
@Library('company-shared-library@main') _

pipeline {
    agent any
    stages {
        stage('Build Image') {
            steps {
                buildDockerImage('myapp', env.GIT_COMMIT.take(7))
            }
        }
        stage('Deploy') {
            steps {
                deployToKubernetes(
                    credentialsId: 'kubeconfig-staging',
                    deployment: 'myapp',
                    namespace: 'staging'
                )
            }
        }
    }
}
```

### DRY Pipeline Principles

| Principle | Applies to |
|---|---|
| **Single source of truth** for build/deploy logic | Shared library / reusable workflow / CI template |
| **Inputs over hardcoding** — parameterize image names, namespaces, environments | All platforms |
| **Version-pin your templates** (`@v2`, `ref: v1.3`) to avoid surprise breakage | All platforms |
| **Test your pipeline templates** — treat them as production code | All platforms |
| **Keep secrets out of templates** — always pass via caller's secrets | All platforms |

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Tool | Config File | Trigger | Key Feature |
|---|---|---|---|
| GitHub Actions | `.github/workflows/*.yml` | Push, PR, schedule, webhook | Native GitHub integration, vast Marketplace |
| GitLab CI/CD | `.gitlab-ci.yml` | Push, MR, schedule, API | Built-in registry, security scanning, environments |
| Jenkins | `Jenkinsfile` | SCM poll, webhook, schedule | Self-hosted, most configurable, shared libraries |

| Concept | GitHub Actions | GitLab CI | Jenkins |
|---|---|---|---|
| Pipeline unit | Workflow | Pipeline | Pipeline |
| Execution unit | Job | Job | Stage |
| Secret storage | Secrets | CI/CD Variables | Credentials |
| Container builds | docker/build-push-action | docker:dind service | Docker plugin |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 10.1 — GitHub Actions: Basic CI

1. Create a GitHub repository with a simple Node.js or Python app
2. Create `.github/workflows/ci.yml`
3. Add a job that: checks out code, installs dependencies, runs tests
4. Push code and watch the workflow run in the GitHub Actions tab
5. Introduce a failing test and observe the pipeline fail

### Lab 10.2 — Build & Push Container Image (GitHub Actions)

1. Extend your workflow to build a Docker image
2. Configure GHCR (GitHub Container Registry) as the registry
3. Push the image on every merge to `main`
4. Use image tags based on git SHA
5. Verify the image appears in your GitHub package registry

### Lab 10.3 — GitLab Pipeline with Stages

1. Create a GitLab project
2. Write a `.gitlab-ci.yml` with stages: build, test, deploy-staging
3. Use GitLab's built-in container registry
4. Set a CI/CD variable for a fake API key
5. Verify the variable is masked in logs

### Lab 10.4 — Jenkins Pipeline

1. Install Jenkins locally or via Docker: `docker run -p 8080:8080 jenkins/jenkins:lts`
2. Install recommended plugins
3. Create a Pipeline job pointing to a Jenkinsfile in your repo
4. Write a Jenkinsfile with build, test, and a manual deploy stage
5. Trigger a build and observe the stage visualization

### Lab 10.5 — Deployment Strategy: Blue-Green

1. Create two Kubernetes deployments: `myapp-blue` and `myapp-green`
2. Create a Service pointing to `myapp-blue` via label selector
3. Deploy a new version to `myapp-green`
4. Switch the Service selector to `myapp-green`
5. Observe zero-downtime traffic switch

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Jenkins User Documentation](https://www.jenkins.io/doc/)
- [Continuous Delivery (book)](https://continuousdelivery.com/) — Jez Humble & Dave Farley
- [Glossary: CI](./glossary.md#c), [CD](./glossary.md#c), [Artifact](./glossary.md#a), [Pipeline](./glossary.md#p), [Webhook](./glossary.md#w)
- **Certification**: GitLab Certified CI/CD Associate

[↑ Back to TOC](#table-of-contents)

---

## Trunk-Based Development

Trunk-based development (TBD) is a branching model where all developers commit to a single main branch (the "trunk") at least once a day, with short-lived feature branches that last no more than 2-3 days.

### Why TBD Enables Real CI/CD

Long-lived feature branches are the enemy of continuous integration. If a feature branch diverges from main for two weeks, merging it becomes a painful integration event. The longer the branch lives, the higher the merge conflict probability and the harder it is to detect integration bugs early.

TBD forces integration continuously. Every developer is always working on code that is nearly identical to what is in production. This makes true CI — where every commit is built, tested, and potentially deployable — achievable.

### Feature Flags: Deploying Without Releasing

TBD requires a way to merge incomplete features to main without affecting users. Feature flags solve this:

```javascript
// Application code using a feature flag
const { isEnabled } = require('./feature-flags');

app.get('/checkout', async (req, res) => {
  if (await isEnabled('new_checkout_flow', { userId: req.user.id })) {
    return newCheckoutHandler(req, res);
  }
  return legacyCheckoutHandler(req, res);
});
```

Feature flag tools:
- **LaunchDarkly:** SaaS platform, SDKs for every language, targeting rules, kill switches
- **Unleash:** Open-source, self-hosted or cloud, activation strategies
- **OpenFeature:** CNCF standard abstraction layer — write once, swap providers

```yaml
# OpenFeature with Unleash backend
const client = OpenFeature.getClient();

// Evaluate a flag with context
const enabled = await client.getBooleanValue(
  'new-checkout-flow',
  false,  // default value
  {
    targetingKey: user.id,
    attributes: {
      email: user.email,
      plan: user.subscription.plan,
      country: user.country,
    }
  }
);
```

### TBD Branching Rules

- **One-person commits directly to main** for small changes
- **Short-lived feature branches** (< 2 days) for changes that need PR review
- **No long-lived release branches** — use tags to mark releases
- **Hotfixes** go to main first, then cherry-pick to any active release tag if needed
- **`--no-ff` merges** preserve commit history clarity

[↑ Back to TOC](#table-of-contents)

---

## Supply Chain Security in CI/CD

The software supply chain — the chain from code to running software — has become a major attack surface. The SolarWinds attack (2020) and XZ Utils backdoor (2024) demonstrated that sophisticated attackers target the build process itself.

### SLSA: Supply-chain Levels for Software Artifacts

SLSA (pronounced "salsa") is a framework for securing the software supply chain. It defines four levels of build integrity:

| SLSA Level | Requirements |
|-----------|-------------|
| SLSA 1 | Build process is scripted; provenance exists |
| SLSA 2 | Hosted build platform; signed provenance |
| SLSA 3 | Hardened build platform; non-falsifiable provenance |
| SLSA 4 | Two-person review; hermetic builds |

Most organisations target SLSA 2–3. Achieving SLSA 2 requires that your CI/CD system (GitHub Actions, GitLab CI) generates a signed provenance document for every build.

### Signing Container Images with `cosign`

`cosign` (from Sigstore) signs container images with cryptographic signatures. Kubernetes admission controllers can then verify signatures before allowing images to run.

```bash
# Generate a key pair (or use keyless signing with GitHub Actions OIDC)
cosign generate-key-pair

# Sign an image after pushing to registry
cosign sign \
  --key cosign.key \
  myregistry.io/myapp:1.4.2-abc1234

# Verify a signature before deploying
cosign verify \
  --key cosign.pub \
  myregistry.io/myapp:1.4.2-abc1234

# Keyless signing in GitHub Actions (uses OIDC identity)
# No key management needed — signature is anchored to the GH Actions OIDC token
cosign sign \
  --oidc-issuer https://token.actions.githubusercontent.com \
  myregistry.io/myapp@sha256:abc123
```

### Generating SBOMs

A Software Bill of Materials (SBOM) lists every package in your container image. It is the foundation of vulnerability management.

```bash
# Generate SBOM with Syft
syft myregistry.io/myapp:1.4.2 -o spdx-json > sbom.spdx.json

# Scan SBOM for vulnerabilities with Grype
grype sbom:./sbom.spdx.json --fail-on high

# Attach SBOM to image in registry
cosign attach sbom --sbom sbom.spdx.json myregistry.io/myapp:1.4.2

# Verify SBOM is attached
cosign download sbom myregistry.io/myapp:1.4.2
```

### GitHub Actions: Hardened Workflow

```yaml
name: Secure Build

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write    # required for keyless cosign signing

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0     # needed for tag signing

      - uses: docker/setup-buildx-action@v3

      - name: Build image
        uses: docker/build-push-action@v5
        id: build
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          sbom: true         # BuildKit generates SBOM automatically
          provenance: true   # BuildKit generates SLSA provenance

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign image (keyless)
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
          format: sarif
          output: trivy-results.sarif
          exit-code: 1
          severity: CRITICAL,HIGH

      - name: Upload Trivy results to Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

[↑ Back to TOC](#table-of-contents)

---

## DORA Metrics: Measuring Pipeline Performance

Google's DevOps Research and Assessment (DORA) identified four key metrics that correlate with high software delivery performance:

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| Deployment frequency | Multiple times/day | Once/week to once/month | Once/month to once/6 months | Less than once/6 months |
| Lead time for changes | < 1 hour | 1 day – 1 week | 1 week – 1 month | > 6 months |
| Change failure rate | 0-5% | 5-10% | 10-15% | > 15% |
| Time to restore service | < 1 hour | < 1 day | 1 day – 1 week | > 6 months |

### Collecting DORA Metrics from Your Pipeline

```python
# Script to calculate lead time from CI events
# (GitHub API example)
import requests
from datetime import datetime

token = os.environ['GITHUB_TOKEN']
repo = 'mycompany/myapp'
headers = {'Authorization': f'Bearer {token}'}

# Get all merged PRs in the last 30 days
prs = requests.get(
    f'https://api.github.com/repos/{repo}/pulls?state=closed&per_page=100',
    headers=headers
).json()

lead_times = []
for pr in prs:
    if not pr['merged_at']:
        continue
    # Lead time = time from first commit to deployment
    commits = requests.get(
        f"https://api.github.com/repos/{repo}/pulls/{pr['number']}/commits",
        headers=headers
    ).json()
    first_commit_time = datetime.fromisoformat(commits[0]['commit']['committer']['date'].replace('Z', '+00:00'))
    merge_time = datetime.fromisoformat(pr['merged_at'].replace('Z', '+00:00'))
    lead_time_hours = (merge_time - first_commit_time).total_seconds() / 3600
    lead_times.append(lead_time_hours)

avg_lead_time = sum(lead_times) / len(lead_times)
print(f"Average lead time for changes: {avg_lead_time:.1f} hours")
```

### Improving DORA Metrics

**Deployment frequency:** The main bottleneck is usually manual approval steps or slow test suites. Automate approvals for low-risk changes, parallelize tests, and adopt trunk-based development.

**Lead time:** Reduce PR review time (smaller PRs), reduce CI build time (caching, parallelism), and eliminate manual gates.

**Change failure rate:** Increase test coverage, add pre-deployment smoke tests, and use progressive delivery (canary, blue-green) to catch problems before full rollout.

**Time to restore:** Improve observability (faster detection), create runbooks for common failures, and practice incident response.

[↑ Back to TOC](#table-of-contents)

---

## Pipeline Performance Optimisation

Slow pipelines are a developer productivity killer. If a pipeline takes 45 minutes, developers stop working in small increments and batch up large PRs — destroying the feedback loop that CI/CD is designed to create.

### Caching Dependencies

```yaml
# GitHub Actions — cache node_modules
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Docker layer caching with BuildKit
- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
    context: .
    push: true
    tags: ${{ env.IMAGE_TAG }}
```

### Parallelising Jobs

```yaml
# GitHub Actions — parallel test matrix
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
      fail-fast: false
    steps:
      - name: Run test shard
        run: |
          npx jest \
            --shard=${{ matrix.shard }}/4 \
            --ci \
            --coverage=false
```

Sharding splits your test suite across parallel runners. A 20-minute sequential test suite becomes a 5-minute parallel run with 4 shards.

### Conditional Job Execution

```yaml
jobs:
  detect-changes:
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      frontend: ${{ steps.filter.outputs.frontend }}
      infra: ${{ steps.filter.outputs.infra }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            backend:
              - 'src/api/**'
              - 'package.json'
            frontend:
              - 'src/ui/**'
            infra:
              - 'terraform/**'

  test-backend:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    # ... backend tests

  test-frontend:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    # ... frontend tests
```

Conditional execution skips test jobs when the relevant code has not changed. A change to a README will not trigger a 15-minute backend test run.

[↑ Back to TOC](#table-of-contents)

---

## Database Migrations in CI/CD

Database migrations are one of the trickiest parts of continuous delivery. A bad migration can take down production faster than a bad application deploy, and rollback is often harder.

### Migration Safety Rules

1. **Never drop a column or table in the same deploy as the code that stops using it.** First deploy: code that does not use the column (backward-compatible). Second deploy: migration that drops the column. This two-phase approach prevents the new code from reading a column that the migration just dropped.

2. **New columns must be nullable or have a default.** Adding a `NOT NULL` column without a default will fail on large tables (or lock the table for minutes).

3. **Add indexes concurrently.** `CREATE INDEX CONCURRENTLY` (PostgreSQL) / `ALTER TABLE ... ADD INDEX` (MySQL) avoids locking the table.

4. **Test migrations on a production-sized data copy.** A migration that runs in 50ms on dev can take 45 minutes on 50M rows of production data.

### Migrations in GitHub Actions

```yaml
jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: production
    needs: [build, test]
    steps:
      - uses: actions/checkout@v4

      - name: Run database migrations
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: |
          # Ensure migration is idempotent (can be re-run safely)
          npx prisma migrate deploy

      - name: Verify migration completed
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: |
          npx prisma migrate status
          # Exit non-zero if there are pending migrations
```

### Zero-Downtime Migration Strategy

For large tables, use an expand-contract pattern:

1. **Expand:** Add new column with nullable. Deploy app that writes to both old and new column.
2. **Backfill:** Background job populates new column for existing rows.
3. **Contract:** Make new column NOT NULL (after verifying backfill complete). Deploy app that reads only from new column. Drop old column.

This pattern allows migrations to run without locking tables or blocking reads.

[↑ Back to TOC](#table-of-contents)

---

## Common Mistakes & Pitfalls

- **Treating CI and CD as the same thing.** CI = building and testing every commit. CD = automatically deploying passing builds. Many teams have CI but not CD — they still require manual deployment steps, negating most of the benefit.
- **Storing secrets in CI environment variables as plain text.** Use a secret manager (GitHub Secrets, Vault, AWS Secrets Manager). Audit who can view secrets in your CI system.
- **Monolithic pipelines with no parallelism.** A 40-minute single-job pipeline becomes an 8-minute parallel pipeline with the same work — split jobs by concern and run them in parallel.
- **Skipping the `--check` / dry-run step before deploy.** Always verify the artifact before deploying. A build that passes tests but produces a broken Docker image wastes a deploy.
- **No artifact versioning.** Tag Docker images with the git SHA, not `latest`. `latest` makes rollbacks and debugging impossible.
- **Baking secrets into images.** Never `COPY .env /app/.env` in a Dockerfile. Use runtime environment variables or secret injection at startup.
- **No pipeline caching.** `npm install` without a cache adds 2-5 minutes to every run. Cache `node_modules`, Maven `.m2`, Go module cache, and Docker layers.
- **Long-lived feature branches.** Branches that exist for 2+ weeks accumulate merge debt and defeat the purpose of CI.
- **Not testing the deployment rollback.** Deploy a broken version and verify that the rollback procedure works before you need it in an incident.
- **Missing post-deploy smoke tests.** Tests that pass in CI don't guarantee the production deployment works. Add a post-deploy health check that fails the pipeline if the service doesn't respond.
- **No change failure rate tracking.** Without measuring how often deploys cause incidents, you cannot improve. Track it even manually at first.
- **Committing generated files.** Generated code, compiled artifacts, and `package-lock.json` changes from different Node.js versions pollute PRs and cause unnecessary CI runs.
- **Running database migrations before health checks.** If the new app crashes on startup, you may have run a migration that the old code cannot handle. Verify the new version starts healthy before running migrations.
- **No manual approval for production.** For high-risk changes, a manual approval gate in the pipeline ensures a human reviews the deploy plan before it runs. This is different from CD — you automate everything up to production, then require a deliberate confirmation.

[↑ Back to TOC](#table-of-contents)

---

## Interview Prep

**Q: What is the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment?**
A: CI is the practice of merging all developer changes to a shared branch frequently and verifying each merge with an automated build and test suite. Continuous Delivery extends CI by ensuring the main branch is always in a deployable state — every passing build could be deployed, but deployment is a deliberate human decision. Continuous Deployment goes one step further: every passing build is automatically deployed to production without human intervention.

**Q: What makes a good CI pipeline?**
A: Fast (under 10 minutes ideally), reliable (no flaky tests), and informative (clear failure messages, links to logs). It should run on every commit, be reproducible (same inputs → same outputs), and produce a deployable artifact as its primary output. Parallelism, caching, and conditional execution are the main tools for keeping it fast.

**Q: How do you handle secrets in CI/CD pipelines?**
A: Store secrets in the CI platform's secret store (GitHub Secrets, GitLab CI Variables marked secret) or inject from a secret manager (Vault, AWS Secrets Manager) at runtime. Never hardcode secrets in pipeline YAML files or commit them to the repository. Rotate secrets regularly. Audit who has access to view/edit CI secrets.

**Q: What is a deployment strategy and when would you use each?**
A: Rolling: gradually replace old instances with new ones — minimal overhead, brief period with mixed versions. Blue-green: maintain two identical environments, switch traffic atomically — instant rollback, doubles infrastructure cost. Canary: route a small percentage of traffic to the new version, increase gradually — lowest risk, requires traffic splitting infrastructure. Recreate: stop all old, start all new — simplest, causes downtime, only for non-critical services.

**Q: What are DORA metrics and why do they matter?**
A: DORA metrics are four measures of software delivery performance: deployment frequency, lead time for changes, change failure rate, and time to restore service. They matter because they correlate with both business outcomes (revenue, profitability) and developer experience. Elite performers deploy multiple times per day with sub-1-hour lead time and near-zero failure rate. Tracking them helps identify bottlenecks — a high change failure rate points to testing gaps, a high lead time points to slow reviews or pipelines.

**Q: How would you reduce a 45-minute CI pipeline to under 10 minutes?**
A: Profile first — add timing to each step to find the bottleneck. Common wins: parallelise test jobs (sharding), add dependency caching (npm/pip/go modules), add Docker layer caching, skip jobs for unchanged code paths (paths-filter), upgrade runner hardware, and eliminate unnecessary steps (linting that blocks tests instead of running in parallel).

**Q: What is trunk-based development and how does it differ from Gitflow?**
A: In trunk-based development, all developers commit to a single main branch, with short-lived feature branches lasting at most 2-3 days. Gitflow uses long-lived branches (develop, release, hotfix, feature) with infrequent integration. TBD enables true CI because code is integrated continuously. Gitflow creates integration events (merges to develop or main) that are painful at scale.

**Q: How do you handle a failed deployment in production?**
A: Immediate: if health checks fail, trigger automatic rollback (deployment circuit breaker in ECS, or run `kubectl rollout undo`). If not automatic, manually roll back to the previous artifact version. Then: investigate the failure in the non-production pipeline, add a regression test that would have caught it, and re-deploy after fixing. Update the runbook with what happened.

**Q: What is the purpose of a deployment approval gate?**
A: An approval gate requires a human to review and approve a deployment before it proceeds. Use it for high-risk changes to production (breaking schema changes, major feature launches, infrastructure changes). The key design decision is: the gate should require reviewing the diff or plan output, not just clicking approve — otherwise it adds latency without adding safety.

**Q: What is a change failure rate and how do you measure it?**
A: Change failure rate is the percentage of deployments that cause a degradation requiring immediate action (incident, rollback, hotfix). Measure it by tracking: number of deploys in a period, number that resulted in a P1/P2 incident or rollback. Elite performers maintain < 5%. High failure rates indicate insufficient testing, lack of observability, or overly complex deployments.

**Q: How does supply chain security apply to CI/CD?**
A: The build pipeline is an attractive attack target — a compromised build step can inject malicious code into every artifact. Mitigations: pin all build dependencies (actions versions, Docker base images, npm packages) with hashes, sign artifacts with cosign, generate and publish SBOMs, use minimal base images, restrict pipeline permissions (least-privilege token scopes), and audit third-party GitHub Actions before use.

**Q: What is a self-hosted CI runner and when would you use one?**
A: A self-hosted runner is a machine you manage that executes CI jobs. Use them when: you need access to internal network resources (private registries, databases), you need specific hardware (GPU, high-memory), you have high CI volume and cloud runner costs are significant, or your security policy prohibits sending code to external CI services. Downsides: you are responsible for security patching and scaling.

[↑ Back to TOC](#table-of-contents)

---

## A Day in the Life: Platform Engineer (CI/CD Focus)

It is 8:30 AM and you join the daily standup. The backend team lead mentions their pipeline has been failing intermittently on the Docker push step — about 1 in 5 runs fails with a timeout. Your first task.

You pull up the pipeline logs for the last 20 runs. The pattern is clear: the failure always happens at the same step (GHCR push) and always after approximately 90 seconds — exactly matching the default timeout for GitHub Actions steps. The image being pushed is 2.3 GB. After caching common layers, it is still pushing 800 MB every build. You recommend switching to BuildKit's cache-to/cache-from with the GitHub Actions cache backend, which means only changed layers are pushed on subsequent builds. You implement it in a 10-line change to the workflow file and verify with three test runs — the push now completes in under 20 seconds. The team lead is very happy.

By 10 AM you are working on a different project: adding SBOM generation and image signing to the organisation's standard container build workflow. The security team has been asking for this for three months and the tooling has matured enough to make it straightforward. You integrate `syft` for SBOM generation and `cosign` with keyless signing (OIDC-based) into the shared reusable workflow. You write a companion policy document explaining how to verify signatures. The PR is ready by noon.

After lunch, you investigate a report from the SRE team: the production deployment pipeline for the payments service takes 42 minutes. You run a timing analysis. The culprit: a sequential matrix of 8 test jobs that each take 5 minutes, running one after another rather than in parallel. Whoever set up the matrix forgot to set `strategy.fail-fast: false` and didn't realise the jobs were sequential, not parallel. A two-line fix: add `max-parallel: 8` to the matrix strategy. The pipeline drops from 42 minutes to 9 minutes.

At 3 PM you run the weekly CI health review. Metrics: average pipeline duration (9 minutes, down from 14 last month), flaky test rate (0.3%, down from 1.1%), deployment frequency (47 deploys across all services this week), and change failure rate (2.1% — one rollback on a frontend service). You prepare a one-page summary for the engineering leadership meeting.

End of day: three pipeline improvements shipped, SBOM signing in review, weekly health metrics prepared. Infrastructure-as-code and platform work rarely produces visible features — but today's 20-second image push and 9-minute pipeline will accelerate every developer on those teams for years.

[↑ Back to TOC](#table-of-contents)

---

## Progressive Delivery with Argo Rollouts

Feature flags let you decouple deploy from release, but progressive delivery goes further: it automates the promotion or rollback decision based on real metrics. Argo Rollouts adds a `Rollout` custom resource to Kubernetes that replaces a standard `Deployment` and adds canary, blue-green, and analysis capabilities.

### Why Progressive Delivery Matters

A standard Kubernetes rolling update replaces pods one by one. If something is wrong, Kubernetes will pause but it has no idea whether your error rate went from 0.1 % to 5 %. It only knows whether pods become `Ready`. Progressive delivery wires your observability stack into the promotion gate, so a spike in HTTP 500s or a latency p99 above threshold automatically triggers a rollback before most users are affected.

At a SaaS company processing payments, even a 2 % increase in checkout errors during a 10-minute canary window can mean thousands of failed transactions. Without automated analysis, the on-call engineer must manually watch dashboards during every deploy. With Argo Rollouts and a Prometheus `AnalysisTemplate`, the promotion decision is made by the platform — not a tired human at 2 AM.

### Installing Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install the kubectl plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### Canary Rollout with Automated Analysis

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: registry.company.com/payment-service:v2.3.1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
  strategy:
    canary:
      canaryService: payment-service-canary
      stableService: payment-service-stable
      trafficRouting:
        nginx:
          stableIngress: payment-service-ingress
      steps:
      - setWeight: 5
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: payment-service-canary
      - setWeight: 20
      - pause: {duration: 10m}
      - analysis:
          templates:
          - templateName: success-rate
          - templateName: latency-p99
          args:
          - name: service-name
            value: payment-service-canary
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
```

```yaml
# analysis-template-success-rate.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 1m
    count: 5
    successCondition: result[0] >= 0.95
    failureLimit: 2
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(http_requests_total{
            service="{{args.service-name}}",
            status!~"5.."
          }[2m])) /
          sum(rate(http_requests_total{
            service="{{args.service-name}}"
          }[2m]))
```

```yaml
# analysis-template-latency.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-p99
  namespace: production
spec:
  args:
  - name: service-name
  metrics:
  - name: p99-latency
    interval: 1m
    count: 5
    successCondition: result[0] <= 0.5
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{
              service="{{args.service-name}}"
            }[2m])) by (le)
          )
```

### Watching a Rollout

```bash
# Live watch rollout status
kubectl argo rollouts get rollout payment-service -n production --watch

# Example output:
# Name:            payment-service
# Namespace:       production
# Status:          ॥ Paused
# Message:         CanaryPauseStep
# Strategy:        Canary
#   Step:          2/8
#   SetWeight:     5
#   ActualWeight:  5
# Images:          registry.company.com/payment-service:v2.3.0 (stable)
#                  registry.company.com/payment-service:v2.3.1 (canary)
# Replicas:
#   Desired:       10
#   Current:       10
#   Updated:       1
#   Ready:         10
#   Available:     10

# Manually promote (skip a pause step)
kubectl argo rollouts promote payment-service -n production

# Manually abort
kubectl argo rollouts abort payment-service -n production
```

### Blue-Green Deployments

Blue-green is simpler: two identical environments, traffic cuts over at once. You test the green environment via a preview service before switching.

```yaml
strategy:
  blueGreen:
    activeService: payment-service-active
    previewService: payment-service-preview
    autoPromotionEnabled: false    # require manual promotion
    prePromotionAnalysis:
      templates:
      - templateName: success-rate
      args:
      - name: service-name
        value: payment-service-preview
    postPromotionAnalysis:
      templates:
      - templateName: success-rate
      args:
      - name: service-name
        value: payment-service-active
    scaleDownDelaySeconds: 600     # keep blue around 10 min for fast rollback
```

The preview service gets real traffic (or your QA team hammers it), analysis runs, then you call `kubectl argo rollouts promote payment-service` to cut over.

[↑ Back to TOC](#table-of-contents)

---

## Self-Hosted CI Runners

SaaS CI/CD runners (GitHub-hosted, GitLab SaaS runners) are convenient but come with real limitations: no access to internal networks, limited customisation, cold-start latency, and per-minute cost at scale. When you are running 500 pipeline minutes per day, self-hosted runners often cut cost by 60–80 % while also enabling private registry access, build caching on local SSDs, and custom toolchains.

### GitHub Actions Self-Hosted Runners

#### Single Runner (Quick Start)

```bash
# On your runner host (Linux x64)
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.315.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.315.0/actions-runner-linux-x64-2.315.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.315.0.tar.gz

# Register (get the token from Settings > Actions > Runners > New self-hosted runner)
./config.sh --url https://github.com/your-org/your-repo \
            --token AXXXXXXXXXXXXXXXX \
            --name prod-runner-01 \
            --labels self-hosted,linux,x64,prod \
            --work _work \
            --unattended

# Install and start as a systemd service
sudo ./svc.sh install
sudo ./svc.sh start
```

#### Runner Autoscaling with ARC (Actions Runner Controller)

For production workloads you want autoscaling — runners spin up on demand and terminate when idle.

```yaml
# arc-runner-set.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: github-runner
  namespace: arc-runners
spec:
  template:
    spec:
      repository: your-org/your-repo
      labels:
      - self-hosted
      - linux
      - arc
      image: summerwind/actions-runner:latest
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: 2
          memory: 4Gi
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: github-runner-autoscaler
  namespace: arc-runners
spec:
  scaleTargetRef:
    name: github-runner
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - your-org/your-repo
```

#### Security Hardening for Self-Hosted Runners

Self-hosted runners introduce attack surface. A malicious pull request can run arbitrary code on your runner unless you harden it:

```yaml
# .github/workflows/ci.yml — safe self-hosted configuration
jobs:
  build:
    # Only use self-hosted for trusted branches
    runs-on: ${{ github.event_name == 'pull_request' && 'ubuntu-latest' || 'self-hosted' }}
    
    # Never use self-hosted for PRs from forks
    if: >
      github.event_name != 'pull_request' ||
      github.event.pull_request.head.repo.full_name == github.repository
```

Additional hardening steps:
- Run each job in an ephemeral container (ARC ephemeral runners, or `--ephemeral` flag)
- Never store long-lived credentials on the runner filesystem — use OIDC or short-lived tokens
- Mount the Docker socket only if absolutely necessary; prefer Kaniko or Buildah in rootless mode
- Isolate runners in a dedicated network segment with egress filtering
- Audit runner logs: every job, every command, every network call
- Rotate registration tokens regularly and use org-level runners with repository scoping

### GitLab CI Runners

```toml
# /etc/gitlab-runner/config.toml
concurrent = 8
check_interval = 0

[[runners]]
  name = "prod-runner-01"
  url = "https://gitlab.company.com"
  token = "RUNNER_TOKEN"
  executor = "kubernetes"
  
  [runners.kubernetes]
    host = ""                        # use in-cluster config
    namespace = "gitlab-runners"
    image = "ubuntu:22.04"
    privileged = false
    cpu_request = "500m"
    memory_request = "512Mi"
    cpu_limit = "2"
    memory_limit = "2Gi"
    
    [[runners.kubernetes.volumes.empty_dir]]
      name = "repo"
      mount_path = "/builds"
      medium = "Memory"              # fast in-memory workspace
```

```bash
# Register a runner
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.company.com" \
  --registration-token "PROJECT_TOKEN" \
  --executor "kubernetes" \
  --description "k8s-runner" \
  --tag-list "k8s,production" \
  --run-untagged=false \
  --locked=true
```

### Runner Maintenance Patterns

```bash
# Check runner health (GitHub CLI)
gh api /repos/YOUR_ORG/YOUR_REPO/actions/runners \
  --jq '.runners[] | {id, name, status, busy}'

# Drain a runner before maintenance
# 1. Mark offline in GitHub Settings > Runners
# 2. Wait for in-progress jobs to finish (check busy=false)
# 3. Perform maintenance
# 4. Re-register if image was updated

# Monitor queue depth (if using ARC)
kubectl get horizontalrunnerautoscaler -n arc-runners
```

[↑ Back to TOC](#table-of-contents)

---

## Environment Promotion Workflows

Most teams operate three to five environments: development, staging, pre-production (or UAT), and production. The discipline of environment promotion defines what triggers a deploy to each environment and who must approve it.

### Promotion Model

```
feature branch  →  dev  →  staging  →  preprod  →  production
                     ↑          ↑            ↑            ↑
                 auto on      auto on    manual gate   manual gate
                  merge        merge     + smoke test   + approval
```

A common mistake is treating staging and production as identical except for scale. In reality you want:
- **dev**: fast feedback, deploys on every push to any branch, no approval
- **staging**: deploys on merge to `main`, automatic smoke tests, automatic promotion to preprod if tests pass
- **preprod**: mirrors production data (anonymised), performance tests run here, manual approval required
- **production**: deploy window enforced (e.g., Mon–Thu 09:00–17:00), requires two approvals for high-risk services

### GitHub Actions Environment Promotion

```yaml
# .github/workflows/promote.yml
name: Environment Promotion

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - uses: actions/checkout@v4
    - name: Deploy to staging
      run: ./scripts/deploy.sh staging ${{ github.sha }}
    - name: Smoke test staging
      run: ./scripts/smoke-test.sh https://staging.company.com
      
  deploy-preprod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: preprod
      url: https://preprod.company.com
    steps:
    - uses: actions/checkout@v4
    - name: Deploy to preprod
      run: ./scripts/deploy.sh preprod ${{ github.sha }}
    - name: Run performance tests
      run: k6 run --vus 50 --duration 5m scripts/perf-test.js
      
  deploy-production:
    needs: deploy-preprod
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.company.com
    # GitHub will pause here for required reviewers
    steps:
    - uses: actions/checkout@v4
    - name: Deploy to production
      run: ./scripts/deploy.sh production ${{ github.sha }}
    - name: Verify production
      run: ./scripts/smoke-test.sh https://app.company.com
```

In GitHub, each environment can have **required reviewers**, **wait timers** (e.g., 10-minute bake time after staging), and **branch protection rules**. Set these under Settings > Environments.

### Deployment Windows and Freeze Periods

```yaml
# Enforce deployment windows using a job condition
deploy-production:
  needs: deploy-preprod
  runs-on: ubuntu-latest
  environment: production
  steps:
  - name: Check deployment window
    run: |
      DAY=$(date -u +%u)   # 1=Mon 7=Sun
      HOUR=$(date -u +%H)
      if [ "$DAY" -gt 4 ] || [ "$HOUR" -lt 9 ] || [ "$HOUR" -ge 17 ]; then
        echo "Outside deployment window (Mon-Thu 09:00-17:00 UTC)"
        echo "Deployments outside window require emergency approval."
        echo "Create an incident in PagerDuty and get sign-off before proceeding."
        exit 1
      fi
```

Many companies add a **change freeze** mechanism: a flag in a config file or a feature flag system that blocks automated deploys during critical periods (Black Friday, end of quarter, major launches). The CI job reads this flag and fails fast with a clear message linking to the change control process.

### GitOps Promotion with Flux

In a GitOps model, Flux or ArgoCD watches a Git repository for desired state. Promotion means opening a pull request to update the image tag in the target environment's directory.

```bash
# scripts/promote.sh — called from CI after staging smoke tests pass
#!/bin/bash
set -euo pipefail

ENV=$1          # e.g., preprod
IMAGE_TAG=$2    # e.g., sha-abc1234

GITOPS_REPO="git@github.com:company/gitops-config.git"
BRANCH="promote/${ENV}-${IMAGE_TAG}"

git clone "$GITOPS_REPO" /tmp/gitops
cd /tmp/gitops

git checkout -b "$BRANCH"

# Update kustomize image tag
cd "environments/${ENV}/payment-service"
kustomize edit set image "registry.company.com/payment-service:${IMAGE_TAG}"

git add .
git commit -m "chore: promote payment-service ${IMAGE_TAG} to ${ENV}"
git push origin "$BRANCH"

# Open a PR
gh pr create \
  --title "Promote payment-service ${IMAGE_TAG} to ${ENV}" \
  --body "Automated promotion from staging smoke tests.\n\nImage: \`${IMAGE_TAG}\`" \
  --base main \
  --head "$BRANCH" \
  --repo company/gitops-config
```

[↑ Back to TOC](#table-of-contents)

---

## Release Engineering Patterns

Release engineering is distinct from deployment. Deployment is the mechanical act of putting new code on servers. Release engineering is the process of deciding what ships, how it's versioned, and how users and downstream consumers learn about it.

### Semantic Versioning

SemVer (`MAJOR.MINOR.PATCH`) is the de-facto standard for libraries, APIs, and CLIs:
- **PATCH** (1.0.x): backwards-compatible bug fixes
- **MINOR** (1.x.0): new backwards-compatible functionality
- **MAJOR** (x.0.0): breaking changes

For services (not libraries), many teams drop strict SemVer and use **date-based versioning** (`2026.03.27.1`) or **commit-sha tags** (`sha-abc1234`). The advantage is that every deploy has a unique, traceable version. The disadvantage is that the version number tells you nothing about breaking changes.

### Conventional Commits

Conventional Commits is a specification for commit message structure that enables automated changelog generation and version bumping.

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

Types:
- `feat`: a new feature (bumps MINOR)
- `fix`: a bug fix (bumps PATCH)
- `feat!` or `BREAKING CHANGE:` footer: breaking change (bumps MAJOR)
- `chore`, `docs`, `style`, `refactor`, `test`, `ci`: do not bump version

Example:
```
feat(auth): add OAuth2 PKCE flow

Implements RFC 7636 PKCE for public clients.
Resolves #421.
```

### Automated Releases with semantic-release

`semantic-release` reads conventional commits and fully automates version bumping, changelog generation, and GitHub release creation.

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    ["@semantic-release/npm", {
      "npmPublish": false
    }],
    ["@semantic-release/github", {
      "assets": [
        {"path": "dist/*.tar.gz", "label": "Distribution"}
      ]
    }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}
```

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0      # semantic-release needs full history
        persist-credentials: false
    - uses: actions/setup-node@v4
      with:
        node-version: 20
    - run: npm ci
    - run: npx semantic-release
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Release Artifacts and Attestations

When you create a release, attach all relevant artifacts and their provenance attestations.

```yaml
- name: Create release artifacts
  run: |
    # Build binary
    go build -ldflags "-X main.version=${RELEASE_VERSION}" \
      -o dist/myapp-linux-amd64 ./cmd/myapp
    
    # Create tarball
    tar czf dist/myapp-${RELEASE_VERSION}-linux-amd64.tar.gz \
      -C dist myapp-linux-amd64
    
    # Generate checksum
    sha256sum dist/*.tar.gz > dist/checksums.txt
    
    # Sign with cosign (keyless, using OIDC)
    cosign sign-blob \
      --bundle dist/myapp-${RELEASE_VERSION}-linux-amd64.tar.gz.bundle \
      dist/myapp-${RELEASE_VERSION}-linux-amd64.tar.gz

- name: Upload to GitHub Release
  uses: softprops/action-gh-release@v2
  with:
    files: |
      dist/*.tar.gz
      dist/checksums.txt
      dist/*.bundle
    generate_release_notes: true
```

[↑ Back to TOC](#table-of-contents)

---

## Testing Strategy in CI/CD

Where you run tests in the pipeline matters as much as what tests you run. Running all tests in every job wastes time and obscures failures. A well-structured pipeline runs the cheapest, fastest tests first and gates later stages on earlier success.

### The Testing Pyramid in CI

```
            /\
           /  \
          / e2e\          ← few, slow, expensive — run against staging only
         /------\
        /  integ  \       ← moderate count, run in CI on every PR
       /------------\
      /  unit tests  \    ← many, fast, run on every push
     /________________\
```

### Unit Tests

Unit tests are the fastest feedback loop. They should run in under 2 minutes for most codebases. If they are slower, parallelize by package or use test sharding.

```yaml
unit-test:
  runs-on: ubuntu-latest
  steps:
  - uses: actions/checkout@v4
  - name: Run unit tests
    run: |
      go test ./... \
        -race \
        -count=1 \
        -timeout 120s \
        -coverprofile coverage.out
  - name: Upload coverage
    uses: codecov/codecov-action@v4
    with:
      files: coverage.out
      fail_ci_if_error: true
```

### Integration Tests

Integration tests verify that components work together: service + database, service + message queue, service + external API (using a mock or test double).

```yaml
integration-test:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:16-alpine
      env:
        POSTGRES_USER: testuser
        POSTGRES_PASSWORD: testpass
        POSTGRES_DB: testdb
      options: >-
        --health-cmd pg_isready
        --health-interval 5s
        --health-timeout 5s
        --health-retries 10
    redis:
      image: redis:7-alpine
      options: >-
        --health-cmd "redis-cli ping"
        --health-interval 5s
  steps:
  - uses: actions/checkout@v4
  - name: Run integration tests
    env:
      DATABASE_URL: postgres://testuser:testpass@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379
    run: go test ./integration/... -timeout 300s
```

### Contract Testing

Contract tests verify that a service's API matches what its consumers expect. Pact is the most widely used framework.

```typescript
// consumer.pact.spec.ts
import { PactV3, MatchersV3 } from "@pact-foundation/pact";

const provider = new PactV3({
  consumer: "PaymentUI",
  provider: "PaymentService",
});

describe("Payment API contract", () => {
  it("returns a payment by ID", async () => {
    await provider
      .given("payment 123 exists")
      .uponReceiving("a request for payment 123")
      .withRequest({
        method: "GET",
        path: "/payments/123",
        headers: { Accept: "application/json" },
      })
      .willRespondWith({
        status: 200,
        body: {
          id: MatchersV3.string("123"),
          amount: MatchersV3.number(9999),
          currency: MatchersV3.string("GBP"),
          status: MatchersV3.string("completed"),
        },
      });

    await provider.executeTest(async (mockServer) => {
      const result = await getPayment(mockServer.url, "123");
      expect(result.currency).toBe("GBP");
    });
  });
});
```

```yaml
# CI step to publish pact and verify
- name: Publish pact contracts
  run: npx pact-broker publish ./pacts \
    --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
    --broker-token ${{ secrets.PACT_BROKER_TOKEN }} \
    --consumer-app-version ${{ github.sha }} \
    --branch ${{ github.ref_name }}

- name: Can I deploy? (check pact compatibility)
  run: npx pact-broker can-i-deploy \
    --pacticipant PaymentUI \
    --version ${{ github.sha }} \
    --to-environment staging \
    --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
    --broker-token ${{ secrets.PACT_BROKER_TOKEN }}
```

### End-to-End Tests

E2E tests are expensive and brittle. Run them only against staging, not on every PR. Use Playwright or Cypress.

```yaml
e2e-test:
  runs-on: ubuntu-latest
  needs: deploy-staging
  if: github.ref == 'refs/heads/main'
  steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: 20
  - run: npm ci
  - name: Install Playwright browsers
    run: npx playwright install --with-deps chromium
  - name: Run E2E tests
    env:
      BASE_URL: https://staging.company.com
      TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
      TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
    run: npx playwright test --project=chromium
  - uses: actions/upload-artifact@v4
    if: failure()
    with:
      name: playwright-report
      path: playwright-report/
      retention-days: 14
```

### Flaky Test Management

Flaky tests erode trust in CI faster than almost anything else. When a test is flaky, developers start ignoring failures — and then real failures get ignored too.

```yaml
# Detect flakiness: run tests multiple times
- name: Run tests with retry detection
  run: |
    for i in 1 2 3; do
      echo "=== Run $i ==="
      go test ./... -count=1 2>&1 | tee run-${i}.txt
    done
    # If results differ across runs, flag as flaky
    if ! diff run-1.txt run-2.txt > /dev/null 2>&1 || \
       ! diff run-2.txt run-3.txt > /dev/null 2>&1; then
      echo "FLAKY TESTS DETECTED"
      exit 1
    fi
```

When you find a flaky test:
1. Open a dedicated issue tagged `flaky-test` with the failure log
2. Quarantine it immediately (move to a `quarantine` suite that runs separately and does not gate the pipeline)
3. Fix within one sprint — treat it as a P1 bug
4. Never delete a flaky test without understanding why it was flaky

[↑ Back to TOC](#table-of-contents)

---

## CI/CD for Monorepos

A monorepo contains multiple services or packages in a single repository. The naive approach — run all tests and deploys on every commit — becomes untenable once you have 20+ services. You need **affected-only builds**: only build and test what changed.

### Detecting Changes

```bash
# Get list of changed paths since main
CHANGED=$(git diff --name-only origin/main...HEAD)
echo "$CHANGED"

# Check if a specific service changed
if echo "$CHANGED" | grep -q "^services/payment-service/"; then
  echo "payment-service changed"
fi
```

### GitHub Actions with paths-filter

```yaml
# .github/workflows/ci.yml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      payment: ${{ steps.filter.outputs.payment }}
      user: ${{ steps.filter.outputs.user }}
      frontend: ${{ steps.filter.outputs.frontend }}
    steps:
    - uses: actions/checkout@v4
    - uses: dorny/paths-filter@v3
      id: filter
      with:
        filters: |
          payment:
            - 'services/payment-service/**'
            - 'shared/db-client/**'
          user:
            - 'services/user-service/**'
          frontend:
            - 'apps/web/**'
            - 'packages/ui/**'

  test-payment:
    needs: changes
    if: ${{ needs.changes.outputs.payment == 'true' }}
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: cd services/payment-service && npm test

  test-user:
    needs: changes
    if: ${{ needs.changes.outputs.user == 'true' }}
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: cd services/user-service && npm test
```

### Turborepo for JavaScript Monorepos

Turborepo builds a dependency graph of your packages and runs tasks in the correct order, caching aggressively.

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "deploy": {
      "dependsOn": ["build", "test"],
      "outputs": []
    }
  }
}
```

```yaml
# GitHub Actions with Turborepo remote cache
- name: Build and test affected packages
  run: npx turbo run build test --filter=...[origin/main]
  env:
    TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
    TURBO_TEAM: ${{ secrets.TURBO_TEAM }}
    TURBO_REMOTE_ONLY: true
```

`--filter=...[origin/main]` tells Turborepo to only run tasks for packages that changed relative to `main`, plus their dependents.

### Nx for Polyglot Monorepos

Nx works across JavaScript, TypeScript, Go, Python, and more.

```bash
# Install Nx
npm add -D nx

# Run affected tests
npx nx affected --target=test --base=origin/main

# Run affected builds
npx nx affected --target=build --base=origin/main

# Visualise dependency graph
npx nx graph
```

```yaml
# .github/workflows/ci.yml
- name: Run affected tests
  run: npx nx affected --target=test --base=origin/main --parallel=3
  
- name: Run affected builds
  run: npx nx affected --target=build --base=origin/main --parallel=3
```

### Versioning in a Monorepo

Each package in a monorepo may need its own version. Tools like `changesets` (for JavaScript) or `release-please` (language-agnostic) handle this:

```bash
# Changesets workflow
# 1. Developer adds a changeset alongside their PR
npx changeset add

# 2. CI bot opens a "Version Packages" PR when changesets accumulate
# 3. Merging the Version Packages PR bumps versions and publishes

# GitHub Actions for changesets
- name: Create Release PR or Publish
  uses: changesets/action@v1
  with:
    publish: npm run release
    version: npm run version
    commit: "chore: version packages"
    title: "chore: version packages"
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

[↑ Back to TOC](#table-of-contents)

---

## Reusable Workflows and Composite Actions

As your CI/CD matures, you will find yourself copying the same 40-line job definition across 15 repositories. Reusable workflows and composite actions let you DRY this up.

### Reusable Workflows (`workflow_call`)

A reusable workflow is a full workflow file that can be called from other workflows.

```yaml
# .github/workflows/reusable-deploy.yml  (in a shared .github repository)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
      service-name:
        required: true
        type: string
    secrets:
      KUBECONFIG:
        required: true
      REGISTRY_TOKEN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up kubectl
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config
        chmod 600 ~/.kube/config
    
    - name: Deploy
      run: |
        kubectl set image deployment/${{ inputs.service-name }} \
          ${{ inputs.service-name }}=registry.company.com/${{ inputs.service-name }}:${{ inputs.image-tag }} \
          -n ${{ inputs.environment }}
        kubectl rollout status deployment/${{ inputs.service-name }} \
          -n ${{ inputs.environment }} \
          --timeout=300s
    
    - name: Smoke test
      run: |
        sleep 15
        ./scripts/smoke-test.sh \
          https://${{ inputs.environment }}.company.com \
          ${{ inputs.service-name }}
```

```yaml
# .github/workflows/ci.yml  (in each service's repository)
jobs:
  deploy-staging:
    needs: build
    uses: company/.github/.github/workflows/reusable-deploy.yml@main
    with:
      environment: staging
      image-tag: ${{ needs.build.outputs.image-tag }}
      service-name: payment-service
    secrets:
      KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}
      REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}
      
  deploy-production:
    needs: deploy-staging
    uses: company/.github/.github/workflows/reusable-deploy.yml@main
    with:
      environment: production
      image-tag: ${{ needs.build.outputs.image-tag }}
      service-name: payment-service
    secrets:
      KUBECONFIG: ${{ secrets.PROD_KUBECONFIG }}
      REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}
```

### Composite Actions

Composite actions wrap multiple steps into a single reusable action, without needing a full workflow.

```yaml
# .github/actions/setup-go-cache/action.yml
name: Setup Go with Cache
description: Install Go and restore module/build cache

inputs:
  go-version:
    description: Go version to use
    required: false
    default: '1.22'

runs:
  using: composite
  steps:
  - name: Setup Go
    uses: actions/setup-go@v5
    with:
      go-version: ${{ inputs.go-version }}
      
  - name: Cache Go modules
    uses: actions/cache@v4
    with:
      path: |
        ~/go/pkg/mod
        ~/.cache/go-build
      key: ${{ runner.os }}-go-${{ inputs.go-version }}-${{ hashFiles('**/go.sum') }}
      restore-keys: |
        ${{ runner.os }}-go-${{ inputs.go-version }}-
        ${{ runner.os }}-go-
```

```yaml
# Usage in any workflow
- uses: ./.github/actions/setup-go-cache
  with:
    go-version: '1.22'
```

### Versioning Shared Workflows

When you publish reusable workflows, pin them to a release tag, not `main`:

```yaml
# Pinned to a release — safe for production
uses: company/.github/.github/workflows/deploy.yml@v3.1.0

# Pinned to main — will break if deploy.yml changes
uses: company/.github/.github/workflows/deploy.yml@main
```

Use Dependabot to keep action versions up to date:

```yaml
# .github/dependabot.yml
version: 2
updates:
- package-ecosystem: github-actions
  directory: /
  schedule:
    interval: weekly
  groups:
    actions:
      patterns:
      - "*"
```

[↑ Back to TOC](#table-of-contents)

---

## Pipeline Observability and Alerting

A CI/CD pipeline is a production system. It should be monitored like one. When your pipeline silently breaks, developers lose hours before anyone notices.

### What to Measure

| Metric | Why It Matters | Alert Threshold |
|--------|----------------|-----------------|
| Pipeline duration (p50, p95) | Slow pipelines reduce developer productivity | Alert if p95 > 2× baseline |
| Pipeline success rate | A dropping rate signals systemic problems | Alert if 7-day rolling < 85 % |
| Queue wait time | Self-hosted runner capacity issues | Alert if p95 > 5 min |
| Deployment frequency | Core DORA metric | Alert if drops > 50 % week-over-week |
| Change failure rate | Deploy quality | Alert if 7-day rolling > 5 % |
| Flaky test rate | CI trust erosion | Alert if > 1 % of runs have a flaky test |

### Exporting GitHub Actions Metrics to Prometheus

GitHub does not natively expose Prometheus metrics, but you can export them via a scraper job:

```python
# scripts/export-pipeline-metrics.py
import os
import requests
from prometheus_client import Gauge, push_to_gateway, CollectorRegistry

GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]
REPO = os.environ["GITHUB_REPO"]
PUSHGATEWAY = os.environ["PUSHGATEWAY_URL"]

headers = {"Authorization": f"Bearer {GITHUB_TOKEN}"}

registry = CollectorRegistry()
pipeline_duration = Gauge(
    "github_actions_workflow_duration_seconds",
    "Duration of the last workflow run",
    ["workflow", "branch", "status"],
    registry=registry,
)
pipeline_success = Gauge(
    "github_actions_workflow_success",
    "1 if last workflow run succeeded, 0 otherwise",
    ["workflow", "branch"],
    registry=registry,
)

r = requests.get(
    f"https://api.github.com/repos/{REPO}/actions/runs?per_page=100",
    headers=headers,
)
runs = r.json()["workflow_runs"]

for run in runs:
    name = run["name"]
    branch = run["head_branch"]
    status = run["conclusion"] or "in_progress"
    
    if run["created_at"] and run["updated_at"]:
        duration = (
            datetime.fromisoformat(run["updated_at"].rstrip("Z")) -
            datetime.fromisoformat(run["created_at"].rstrip("Z"))
        ).total_seconds()
        pipeline_duration.labels(workflow=name, branch=branch, status=status).set(duration)
    
    success = 1 if run["conclusion"] == "success" else 0
    pipeline_success.labels(workflow=name, branch=branch).set(success)

push_to_gateway(PUSHGATEWAY, job="github-actions-metrics", registry=registry)
```

### Alertmanager Rules for CI/CD

```yaml
# prometheus/alerts/cicd.yml
groups:
- name: cicd
  rules:
  - alert: PipelineSuccessRateLow
    expr: |
      (
        sum(rate(github_actions_workflow_success{workflow="CI"}[7d])) /
        count(rate(github_actions_workflow_success{workflow="CI"}[7d]))
      ) < 0.85
    for: 30m
    labels:
      severity: warning
      team: platform
    annotations:
      summary: "CI pipeline success rate dropped below 85%"
      description: "7-day rolling success rate is {{ humanizePercentage $value }}. Investigate recent failures."
      runbook: "https://runbooks.company.com/cicd/low-success-rate"

  - alert: PipelineDurationHigh
    expr: |
      histogram_quantile(0.95,
        sum(rate(github_actions_workflow_duration_seconds_bucket{
          workflow="CI",
          branch="main"
        }[1h])) by (le)
      ) > 1800
    for: 15m
    labels:
      severity: warning
      team: platform
    annotations:
      summary: "CI pipeline p95 duration exceeds 30 minutes"
      description: "p95 pipeline duration is {{ humanizeDuration $value }}."
```

### Slack Notifications for Pipeline Events

```yaml
# .github/workflows/ci.yml — add to the end of every workflow
  notify-on-failure:
    needs: [test, build, deploy]
    if: failure() && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - name: Notify Slack
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": ":red_circle: Pipeline failed on `main`",
            "attachments": [{
              "color": "danger",
              "fields": [
                {"title": "Repository", "value": "${{ github.repository }}", "short": true},
                {"title": "Commit", "value": "${{ github.sha }}", "short": true},
                {"title": "Triggered by", "value": "${{ github.actor }}", "short": true},
                {"title": "Run URL", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
              ]
            }]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

[↑ Back to TOC](#table-of-contents)

---

## Jenkins Advanced Patterns

Jenkins remains dominant in many enterprises that cannot migrate to SaaS CI/CD due to compliance, air-gap requirements, or the sheer weight of existing Jenkinsfiles. Knowing how to work with Jenkins at an advanced level — shared libraries, Declarative pipeline best practices, and operational hygiene — is a competitive advantage in those environments.

### Declarative Pipeline Structure

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: build
                    image: eclipse-temurin:21-jdk-alpine
                    command: [cat]
                    tty: true
                    resources:
                      requests:
                        cpu: 500m
                        memory: 1Gi
                  - name: docker
                    image: docker:24-dind
                    securityContext:
                      privileged: true
            '''
        }
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds(abortPrevious: true)
        buildDiscarder(logRotator(numToKeepStr: '50'))
        timestamps()
    }
    
    parameters {
        booleanParam(name: 'SKIP_TESTS', defaultValue: false,
                     description: 'Skip test stage (emergency deploys only)')
    }
    
    environment {
        APP_NAME     = 'payment-service'
        IMAGE_REPO   = 'registry.company.com/payment-service'
        SONAR_TOKEN  = credentials('sonarqube-token')
    }
    
    stages {
        stage('Test') {
            when {
                not { expression { params.SKIP_TESTS } }
            }
            steps {
                container('build') {
                    sh './gradlew test jacocoTestReport'
                    junit 'build/test-results/**/*.xml'
                    jacoco execPattern: 'build/jacoco/*.exec'
                }
            }
        }
        
        stage('Static Analysis') {
            steps {
                container('build') {
                    withSonarQubeEnv('company-sonar') {
                        sh './gradlew sonar'
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Image') {
            steps {
                container('docker') {
                    sh """
                        docker build \
                          --label git.commit=${GIT_COMMIT} \
                          --label build.number=${BUILD_NUMBER} \
                          -t ${IMAGE_REPO}:${GIT_COMMIT} \
                          .
                        docker push ${IMAGE_REPO}:${GIT_COMMIT}
                    """
                }
            }
        }
        
        stage('Deploy Staging') {
            when {
                branch 'main'
            }
            steps {
                container('build') {
                    withCredentials([file(credentialsId: 'staging-kubeconfig',
                                         variable: 'KUBECONFIG')]) {
                        sh """
                            kubectl set image deployment/${APP_NAME} \
                              ${APP_NAME}=${IMAGE_REPO}:${GIT_COMMIT} \
                              -n staging
                            kubectl rollout status deployment/${APP_NAME} \
                              -n staging --timeout=300s
                        """
                    }
                }
            }
        }
        
        stage('Deploy Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Yes, deploy"
                submitter "platform-team,release-managers"
                parameters {
                    string(name: 'DEPLOY_REASON',
                           description: 'Reason for production deploy')
                }
            }
            steps {
                container('build') {
                    withCredentials([file(credentialsId: 'prod-kubeconfig',
                                         variable: 'KUBECONFIG')]) {
                        sh """
                            kubectl set image deployment/${APP_NAME} \
                              ${APP_NAME}=${IMAGE_REPO}:${GIT_COMMIT} \
                              -n production
                            kubectl rollout status deployment/${APP_NAME} \
                              -n production --timeout=300s
                        """
                    }
                }
            }
        }
    }
    
    post {
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "FAILED: ${APP_NAME} build #${BUILD_NUMBER} on ${BRANCH_NAME}\n${BUILD_URL}"
            )
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "SUCCESS: ${APP_NAME} build #${BUILD_NUMBER} deployed\n${BUILD_URL}"
            )
        }
        always {
            cleanWs()
        }
    }
}
```

### Jenkins Shared Libraries

Shared libraries let you extract common pipeline logic into a versioned library that all teams can consume.

```
# Directory structure of shared library repository
vars/
  deployService.groovy       # Global variable (callable like a step)
  runTests.groovy
  buildAndPushImage.groovy
src/
  com/company/ci/
    DockerUtils.groovy       # Groovy class
    KubeUtils.groovy
resources/
  com/company/ci/
    pod-template.yaml        # Kubernetes pod template
```

```groovy
// vars/deployService.groovy
def call(Map config) {
    def serviceName = config.serviceName ?: error("serviceName is required")
    def imageTag    = config.imageTag    ?: error("imageTag is required")
    def environment = config.environment ?: error("environment is required")
    def timeout     = config.timeout     ?: 300
    
    withCredentials([file(credentialsId: "${environment}-kubeconfig",
                         variable: 'KUBECONFIG')]) {
        sh """
            kubectl set image deployment/${serviceName} \
              ${serviceName}=registry.company.com/${serviceName}:${imageTag} \
              -n ${environment}
            kubectl rollout status deployment/${serviceName} \
              -n ${environment} --timeout=${timeout}s
        """
    }
    
    // Run smoke test
    def serviceUrl = config.url ?: "https://${environment}.company.com"
    sh "./scripts/smoke-test.sh ${serviceUrl}"
}
```

```groovy
// Usage in any Jenkinsfile after loading the library
@Library('company-shared-library@v2.1.0') _

pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                deployService(
                    serviceName: 'payment-service',
                    imageTag: env.GIT_COMMIT,
                    environment: 'staging'
                )
            }
        }
    }
}
```

Register the shared library in Jenkins: Manage Jenkins > System > Global Trusted Pipeline Libraries. Pin to a tag, not `main`, for stability.

### Jenkins Operational Hygiene

Running Jenkins in production requires active operational care:

```groovy
// Groovy script for Jenkins console — clean up old builds
def jobs = Jenkins.instance.getAllItems(Job.class)
jobs.each { job ->
    def builds = job.getBuilds()
    builds.each { build ->
        // Delete builds older than 90 days
        def age = (System.currentTimeMillis() - build.getTimeInMillis()) / (1000 * 60 * 60 * 24)
        if (age > 90) {
            build.delete()
        }
    }
}
```

Key maintenance tasks:
- **Plugin updates**: run weekly; always test in a staging Jenkins first
- **Disk cleanup**: executor workspaces accumulate fast — use `cleanWs()` in post steps and schedule periodic workspace cleanup
- **Credential rotation**: audit credentials quarterly, remove unused ones, rotate all tokens annually
- **Controller HA**: use Jenkins with a cloud agent backend (Kubernetes plugin) so the controller is stateless; back up JENKINS_HOME with Velero or periodic S3 sync
- **Security hardening**: disable CLI over remoting, enable CSRF protection, restrict script approval, use matrix-based security with role-strategy plugin

[↑ Back to TOC](#table-of-contents)

---

## Feature Flag Lifecycle Management

Feature flags accumulate technical debt faster than almost any other engineering practice. A team that ships flags without a lifecycle process ends up with hundreds of stale flags, configuration drift between environments, and code that is impossible to reason about.

### Flag Creation Standards

Before creating a flag, answer these questions:
1. What is the exact condition this flag controls?
2. What is the default value in production?
3. When will this flag be permanent (never) or removed (target date)?
4. Who owns this flag (team and person)?
5. How will you measure the impact of enabling it?

Document the answers in your feature flag system (LaunchDarkly, Unleash, OpenFeature). Set a **target removal date** for every temporary flag.

### Flag Categories

| Category | Example | Lifetime | Default |
|----------|---------|---------|---------|
| Release flag | `new-checkout-flow` | Weeks — remove after full rollout | `false` |
| Experiment flag | `ab-test-checkout-button-colour` | Days to weeks | random assignment |
| Ops flag | `disable-fraud-check` | Permanent — kill switch | `true` |
| Permission flag | `beta-users-dashboard` | Months | user-group based |

### OpenFeature SDK Integration

```typescript
// src/flags.ts
import { OpenFeature } from "@openfeature/server-sdk";
import { UnleashProvider } from "@openfeature/unleash-provider";

// Configure once at startup
await OpenFeature.setProviderAndWait(
  new UnleashProvider({
    url: process.env.UNLEASH_URL!,
    clientKey: process.env.UNLEASH_CLIENT_KEY!,
    appName: "payment-service",
  })
);

const featureClient = OpenFeature.getClient("payment-service");

// Usage in application code
export async function processPayment(
  request: PaymentRequest,
  userId: string
): Promise<PaymentResult> {
  const useNewFraudEngine = await featureClient.getBooleanValue(
    "new-fraud-engine",
    false,  // default
    { targetingKey: userId }
  );

  if (useNewFraudEngine) {
    return newFraudEngine.process(request);
  }
  return legacyFraudEngine.process(request);
}
```

### Flag Cleanup Automation

Use a scheduled CI job to flag stale feature flags for removal:

```bash
#!/bin/bash
# scripts/audit-feature-flags.sh
# Runs weekly via CI to identify flags past their target removal date

TODAY=$(date +%Y-%m-%d)

# Query Unleash API for all flags
FLAGS=$(curl -s -H "Authorization: ${UNLEASH_ADMIN_TOKEN}" \
  "${UNLEASH_URL}/api/admin/features" | \
  jq -r '.features[] | select(.description | contains("remove-by:")) | 
    {name: .name, description: .description}')

echo "$FLAGS" | jq -r '
  .name + " | " + 
  (.description | capture("remove-by:(?<date>[0-9]{4}-[0-9]{2}-[0-9]{2})").date)
' | while IFS='|' read -r name remove_date; do
  name=$(echo "$name" | xargs)
  remove_date=$(echo "$remove_date" | xargs)
  if [[ "$remove_date" < "$TODAY" ]]; then
    echo "STALE FLAG: $name (was due for removal on $remove_date)"
  fi
done
```

### Rolling Back via Feature Flag vs. Deployment Rollback

One of the most underappreciated benefits of feature flags is instant rollback without a deployment:

| Scenario | Feature Flag Rollback | Deployment Rollback |
|----------|----------------------|---------------------|
| Time to take effect | < 30 seconds | 3-15 minutes |
| Blast radius control | Granular (by user, segment) | All users |
| Requires CI/CD | No | Yes |
| Works for DB schema changes | No | Requires migration |
| Suitable for A/B tests | Yes | No |

When you have an incident caused by a new feature, turning off the flag is almost always faster than rolling back the deployment — especially if the release bundled multiple changes.

[↑ Back to TOC](#table-of-contents)

---

## Secrets Management in CI/CD

Secrets handling is where CI/CD security most often fails. Common patterns: hardcoded tokens in Jenkinsfiles, secrets stored as unencrypted GitHub Actions variables, `.env` files committed to repositories, and CI jobs that print secrets to logs.

### The Principle of Least Privilege for CI Secrets

Each pipeline should have the minimum secrets required for its specific job:
- A test job needs no cloud credentials — only database connection strings for ephemeral test databases
- A build job needs only registry push credentials, scoped to a specific repository
- A staging deploy job needs only staging cluster credentials, not production
- A production deploy job needs production credentials but should validate it is running from `main` and has passed all previous stages

Never give a single CI credential access to multiple environments.

### HashiCorp Vault Integration

```yaml
# GitHub Actions with Vault OIDC
name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Authenticate to Vault
      uses: hashicorp/vault-action@v3
      id: vault
      with:
        url: https://vault.company.com
        method: jwt
        role: github-cicd-production
        jwtGithubAudience: https://vault.company.com
        secrets: |
          secret/data/production/database DB_PASSWORD | DB_PASSWORD ;
          secret/data/production/registry REGISTRY_TOKEN | REGISTRY_TOKEN ;
          secret/data/production/kubeconfig KUBECONFIG_B64 | KUBECONFIG_B64
    
    - name: Deploy
      run: |
        echo "$KUBECONFIG_B64" | base64 -d > /tmp/kubeconfig
        export KUBECONFIG=/tmp/kubeconfig
        ./scripts/deploy.sh production
      env:
        DB_PASSWORD: ${{ steps.vault.outputs.DB_PASSWORD }}
        REGISTRY_TOKEN: ${{ steps.vault.outputs.REGISTRY_TOKEN }}
```

The Vault JWT auth method verifies the GitHub OIDC token — no long-lived secrets stored in GitHub at all.

### Vault Policy for CI/CD Role

```hcl
# vault/policies/github-cicd-production.hcl
path "secret/data/production/*" {
  capabilities = ["read"]
}

# Explicitly deny admin paths
path "auth/*" {
  capabilities = ["deny"]
}

path "sys/*" {
  capabilities = ["deny"]
}

# Allow token renewal only
path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

### Detecting Secrets in Code

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scan

on: [push, pull_request]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0   # scan full history
    - uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

```toml
# .gitleaks.toml — custom rules
[allowlist]
  description = "Global allowlist"
  regexes = [
    '''EXAMPLE_TOKEN''',      # documentation examples
    '''test-api-key-.*'''      # test fixtures
  ]

[[rules]]
  description = "Company internal API keys"
  regex = '''company-api-[0-9a-f]{32}'''
  tags = ["company", "api-key"]
  [rules.allowlist]
    paths = [
      '''tests/fixtures/.*'''
    ]
```

If a secret is found in history, act immediately:
1. Revoke the credential — do this before anything else
2. Use `git filter-repo` to purge the secret from history
3. Force-push to all branches (coordinate with the team)
4. Audit access logs for the revoked credential to check for exploitation
5. Add the pattern to your `gitleaks.toml` so it is caught in future

[↑ Back to TOC](#table-of-contents)

---

## Artifact Management and Container Registries

Every CI pipeline produces artifacts: container images, compiled binaries, npm packages, Helm charts, or test reports. Managing these artifacts well — where they are stored, how long they are kept, and how they are secured — is part of a mature CI/CD practice.

### Container Registry Options

| Registry | Hosting | Strengths | Weaknesses |
|----------|---------|-----------|-----------|
| GitHub Container Registry (ghcr.io) | SaaS | Free for public repos, tight GitHub integration | No geo-replication |
| AWS ECR | SaaS | Native IAM auth, lifecycle policies | AWS-only |
| Google Artifact Registry | SaaS | Multi-format (OCI, Maven, npm, Python) | GCP-only |
| Harbor | Self-hosted | Vulnerability scanning, replication, air-gap | Operational overhead |
| Quay | Self-hosted / SaaS | Strong RBAC, robot accounts | More complex setup |

### Image Lifecycle Policies

Without lifecycle policies, registries accumulate thousands of stale images. A fintech startup discovered they were paying $800/month in ECR storage — almost entirely from untagged images created by feature branches that had long since been merged or abandoned.

```json
// AWS ECR lifecycle policy
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 3 production images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 3
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Remove untagged images older than 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 3,
      "description": "Keep last 10 images on any branch",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    }
  ]
}
```

### Image Vulnerability Scanning in CI

Integrate Trivy into the build pipeline to catch CVEs before images are deployed:

```yaml
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: registry.company.com/payment-service:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1           # fail the build on CRITICAL/HIGH CVEs
    ignore-unfixed: true   # do not fail on unfixed vulnerabilities

- name: Upload SARIF to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: trivy-results.sarif
```

When Trivy finds a vulnerability:
1. Check if it is in the base image — if so, update the base image tag
2. Check if it is in a direct dependency — update the dependency
3. Check if it is in a transitive dependency — update the parent or use `--ignore-unfixed` with a tracked issue
4. If no fix exists, document the acceptance of risk in a `trivy-ignore.yaml` file with an expiry date

```yaml
# .trivyignore.yaml
vulnerabilities:
  - id: CVE-2024-12345
    paths:
      - "usr/local/lib/python3.12/site-packages/some-package"
    statement: "No fix available. Risk accepted. Expires 2026-06-01. See JIRA-1234."
    expires: "2026-06-01"
```

### Helm Chart Repository

If you deploy with Helm, store charts in a chart repository. GitHub Pages works well for small teams:

```yaml
# .github/workflows/chart-release.yml
name: Release Charts

on:
  push:
    branches: [main]
    paths: ['charts/**']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    - name: Configure Git
      run: |
        git config user.name "$GITHUB_ACTOR"
        git config user.email "$GITHUB_ACTOR@users.noreply.github.com"
    - uses: helm/chart-releaser-action@v1.6.0
      env:
        CR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This automatically packages charts, creates GitHub releases with the `.tgz` artifacts, and updates the `index.yaml` on the `gh-pages` branch.

[↑ Back to TOC](#table-of-contents)

---

## Multi-Cloud and Hybrid CI/CD Patterns

As organisations grow, they often end up with workloads spread across AWS, GCP, Azure, and on-premises data centres. CI/CD pipelines must deploy consistently across all of these without duplicating configuration.

### OIDC Federation Across Clouds

All three major clouds support OIDC federation with GitHub Actions, eliminating stored cloud credentials entirely:

```yaml
# AWS
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-cicd
    aws-region: eu-west-1

# GCP
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
    service_account: github-cicd@project.iam.gserviceaccount.com

# Azure
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

With OIDC, the GitHub Actions token is exchanged for a short-lived cloud credential at runtime. There are no long-lived tokens to rotate or accidentally expose.

### Environment-Specific Configuration Management

```yaml
# Manage different config per cloud/region using matrix
deploy:
  strategy:
    matrix:
      include:
        - cloud: aws
          region: eu-west-1
          cluster: prod-eu
          role: arn:aws:iam::111:role/github-cicd
        - cloud: aws
          region: us-east-1
          cluster: prod-us
          role: arn:aws:iam::222:role/github-cicd
        - cloud: gcp
          region: europe-west1
          cluster: prod-eu-gcp
          workload_identity: projects/123/...
  
  runs-on: ubuntu-latest
  steps:
  - name: Deploy to ${{ matrix.cloud }} / ${{ matrix.region }}
    run: |
      ./scripts/deploy-multicloud.sh \
        --cloud ${{ matrix.cloud }} \
        --region ${{ matrix.region }} \
        --cluster ${{ matrix.cluster }} \
        --image-tag ${{ needs.build.outputs.image-tag }}
```

[↑ Back to TOC](#table-of-contents)

---

## Compliance and Audit in CI/CD

In regulated industries — fintech, healthcare, defence — CI/CD pipelines must satisfy audit requirements. Every deployment must have an immutable record of who approved it, what was deployed, when, and from which source.

### Deployment Audit Log

```yaml
# Append to every deployment job
- name: Record deployment audit log
  run: |
    AUDIT_ENTRY=$(jq -n \
      --arg service "$SERVICE_NAME" \
      --arg version "$IMAGE_TAG" \
      --arg env "$ENVIRONMENT" \
      --arg actor "$GITHUB_ACTOR" \
      --arg sha "$GITHUB_SHA" \
      --arg run_id "$GITHUB_RUN_ID" \
      --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      '{
        timestamp: $ts,
        service: $service,
        version: $version,
        environment: $env,
        deployed_by: $actor,
        commit_sha: $sha,
        pipeline_run: $run_id,
        pipeline_url: ("https://github.com/${{ github.repository }}/actions/runs/" + $run_id)
      }')
    
    # Write to immutable audit log (S3 with Object Lock, or append-only DB)
    aws s3 cp - \
      "s3://company-audit-logs/deployments/$(date +%Y/%m/%d)/${SERVICE_NAME}-$(date +%s).json" \
      <<< "$AUDIT_ENTRY"
```

### Change Advisory Board (CAB) Integration

Some regulated environments require CAB approval before production deployments. ServiceNow and Jira Service Management are common systems:

```bash
#!/bin/bash
# scripts/create-change-request.sh
# Creates a ServiceNow change request and waits for approval

CHANGE_ID=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -u "${SN_USER}:${SN_PASSWORD}" \
  "${SN_URL}/api/now/table/change_request" \
  -d "{
    \"short_description\": \"Deploy ${SERVICE_NAME} ${IMAGE_TAG} to production\",
    \"description\": \"Automated deployment via pipeline run ${GITHUB_RUN_ID}\",
    \"type\": \"standard\",
    \"risk\": \"low\",
    \"impact\": \"low\",
    \"assigned_to\": \"platform-team\"
  }" | jq -r '.result.sys_id')

echo "Created change request: $CHANGE_ID"
echo "change_id=$CHANGE_ID" >> "$GITHUB_OUTPUT"

# Poll for approval (max 4 hours)
for i in $(seq 1 240); do
  STATE=$(curl -s \
    -H "Accept: application/json" \
    -u "${SN_USER}:${SN_PASSWORD}" \
    "${SN_URL}/api/now/table/change_request/${CHANGE_ID}" | \
    jq -r '.result.state')
  
  if [ "$STATE" = "implement" ]; then
    echo "Change request approved"
    exit 0
  elif [ "$STATE" = "cancelled" ]; then
    echo "Change request rejected"
    exit 1
  fi
  
  sleep 60
done

echo "Timed out waiting for change approval"
exit 1
```

### SOC 2 Evidence Collection

SOC 2 Type II audits require evidence that every production change was approved, tested, and deployed by authorised personnel. A well-structured CI/CD pipeline produces this evidence automatically:

| Control | CI/CD Evidence |
|---------|---------------|
| CC6.1 — Logical access controls | OIDC-based deploy roles with IAM policies (no shared credentials) |
| CC6.2 — User authentication | Required reviewers on production environment in GitHub |
| CC7.1 — Change management | All changes via PR; no direct commits to `main` |
| CC7.2 — Testing | Required CI checks before merge; coverage thresholds enforced |
| CC8.1 — Change authorisation | GitHub environment protection rules; deployment audit logs |
| A1.1 — Availability monitoring | Post-deployment smoke tests; alerting on deployment failure |

Export this evidence regularly and store it in a read-only S3 bucket or GCS bucket with audit retention policies.

[↑ Back to TOC](#table-of-contents)

---

## Real-World War Story: The Deployment That Took Down Black Friday

At a mid-size e-commerce company — about 2 million monthly active users — the engineering team had a strong CI/CD pipeline: automated tests, image scanning, staging deploys, smoke tests, the works. But they had one gap: the production deployment job ran `kubectl set image` and then checked rollout status, but there was no post-deployment verification against real traffic.

On the Thursday before Black Friday, a deployment went out at 14:30. The smoke test passed. The rollout status reported `successfully rolled out`. Fifteen minutes later, PagerDuty fired: checkout error rate spiked from 0.2 % to 18 %. Revenue was bleeding at roughly £4,000 per minute.

The incident postmortem revealed what happened. The new deployment included a dependency update that changed how the payment service serialised amounts — it now used a float where the previous version used an integer in a shared Kafka message. The smoke test hit the checkout endpoint directly and got a 200. But the background job consuming payment events was throwing deserialization exceptions on every message. Because the consumer ran asynchronously, the smoke test could not see the failure.

The rollback took 11 minutes — not because the `kubectl rollout undo` was slow, but because the on-call engineer spent the first 7 minutes diagnosing which deployment caused it (they had deployed three services in the same hour).

**Changes made after the incident:**

1. Deployments to production are now serialised — only one service deploys at a time, with a 5-minute gap
2. Post-deployment smoke tests now include synthetic transactions that trigger background processing and verify the outcome asynchronously (poll for up to 2 minutes)
3. Every deployment updates a Slack message with the exact service name, version, and a direct link to the rollback command — engineers no longer lose time figuring out what changed
4. DORA metrics (deployment frequency, change failure rate, MTTR) are now reviewed weekly by engineering leadership
5. Argo Rollouts was introduced for the three highest-traffic services, with automatic analysis on HTTP error rate

The total incident cost: approximately £48,000 in lost revenue plus 12 hours of engineering time. The CI/CD improvements took 3 weeks to implement. They have prevented four incidents since, two of which would have been of similar severity.

The lesson is not that CI/CD is insufficient — it is that smoke tests must cover your full transaction path, including asynchronous processing. A pipeline that passes is only as trustworthy as what you are testing.

---

## Tooling Reference Card

A quick reference for the tools covered in this module, with install and usage shortcuts you will reach for regularly.

### GitHub Actions

```bash
# Install GitHub CLI
# Ubuntu/Debian
sudo apt install gh

# Authenticate
gh auth login

# List recent workflow runs
gh run list --limit 10

# Watch a running workflow
gh run watch

# Re-run a failed workflow
gh run rerun <run-id>

# View workflow logs
gh run view <run-id> --log

# List secrets (names only)
gh secret list

# Set a secret
gh secret set MY_SECRET --body "value"

# List environments
gh api /repos/OWNER/REPO/environments | jq '.environments[].name'
```

### ArgoCD

```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login
argocd login argocd.company.com --sso

# List apps
argocd app list

# Sync an app
argocd app sync payment-service

# Check app status
argocd app get payment-service

# Roll back to previous version
argocd app rollback payment-service

# Diff current vs desired state
argocd app diff payment-service
```

### Argo Rollouts

```bash
# Watch rollout
kubectl argo rollouts get rollout payment-service -n production --watch

# Promote (advance past a pause)
kubectl argo rollouts promote payment-service -n production

# Abort (triggers rollback)
kubectl argo rollouts abort payment-service -n production

# Undo (rollback to previous stable)
kubectl argo rollouts undo payment-service -n production

# List all rollouts
kubectl argo rollouts list rollouts -n production
```

### Tekton

```bash
# Install Tekton Pipelines
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Install Tekton CLI
curl -LO https://github.com/tektoncd/cli/releases/latest/download/tkn_linux_amd64.tar.gz
tar xzf tkn_linux_amd64.tar.gz tkn
sudo mv tkn /usr/local/bin/

# List pipelines
tkn pipeline list

# Start a pipeline run
tkn pipeline start build-and-deploy \
  --param image=registry.company.com/payment-service:v1.2.3 \
  --workspace name=source,claimName=pipeline-pvc

# Watch logs
tkn pipelinerun logs --last --follow
```

### Semantic Release

```bash
# Install
npm install -D semantic-release \
  @semantic-release/changelog \
  @semantic-release/git \
  @semantic-release/github

# Dry run (preview what version would be released)
npx semantic-release --dry-run

# Verify configuration
npx semantic-release --verify-conditions
```

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

### Books

- **Continuous Delivery** by Jez Humble and David Farley — the foundational text on the discipline. Every principle in this module traces back here.
- **Accelerate** by Nicole Forsgren, Jez Humble, and Gene Kim — the science behind DORA metrics. Essential reading for making the business case for CI/CD investment.
- **The DevOps Handbook** by Gene Kim et al. — practical implementation patterns across organisations of every size.
- **Release It!** by Michael Nygard — stability patterns that your CI/CD pipeline should verify: circuit breakers, bulkheads, timeouts, and retries.
- **Software Engineering at Google** (O'Reilly) — chapters on testing and code review directly inform how to structure test suites in CI.

### Online Resources

- [GitHub Actions documentation](https://docs.github.com/en/actions) — comprehensive reference for workflows, runners, contexts, and expressions.
- [Argo Rollouts documentation](https://argo-rollouts.readthedocs.io/) — canary and blue-green strategies with analysis templates.
- [OpenFeature specification](https://openfeature.dev/) — vendor-neutral feature flag SDK standard.
- [DORA State of DevOps Report](https://dora.dev/) — annual benchmarking data; use it to compare your metrics against industry peers.
- [slsa.dev](https://slsa.dev/) — Supply-chain Levels for Software Artifacts framework.
- [conventional commits](https://www.conventionalcommits.org/) — commit message specification that enables automated releases.
- [Pact documentation](https://docs.pact.io/) — consumer-driven contract testing.
- [Tekton documentation](https://tekton.dev/docs/) — Kubernetes-native CI/CD primitives.

### Tools to Explore

- **Dagger** — portable CI/CD pipelines written as code (Go, Python, TypeScript), run anywhere: local, CI, cloud.
- **Earthly** — reproducible builds combining Makefile and Dockerfile syntax; strong caching.
- **Buildkite** — hybrid CI/CD that runs agents on your infrastructure with a SaaS control plane.
- **Woodpecker CI** — lightweight self-hosted CI inspired by the original Drone; YAML pipeline definitions, excellent for small teams.
- **Flux** — GitOps toolkit for Kubernetes; pairs with GitHub Actions for full deploy pipelines.
- **cosign** — keyless container image signing; use it in every production image build.
- **Syft + Grype** — SBOM generation and vulnerability scanning; complements Trivy in a defence-in-depth approach.

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution. See [LICENSE.md](./LICENSE.md).*
