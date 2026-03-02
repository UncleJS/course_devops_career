# Module 10: CI/CD Pipelines

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: CI/CD Fundamentals](#beginner-cicd-fundamentals)
- [Beginner: Deployment Strategies](#beginner-deployment-strategies)
- [GitHub Actions](#github-actions)
- [GitLab CI/CD](#gitlab-cicd)
- [Jenkins](#jenkins)
- [Intermediate: Pipeline Design Patterns](#intermediate-pipeline-design-patterns)
- [Intermediate: Secrets Management in Pipelines](#intermediate-secrets-management-in-pipelines)
- [Intermediate: Artifact Management](#intermediate-artifact-management)
- [Intermediate: CI/CD for Containers & Kubernetes](#intermediate-cicd-for-containers--kubernetes)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

CI/CD (Continuous Integration / Continuous Delivery/Deployment) is the engine of modern DevOps. It automates the journey from a developer pushing code to that code running in production — testing, building, scanning, and deploying at every step.

This module covers three industry-standard CI/CD tools: **GitHub Actions** (the most widely adopted modern tool), **GitLab CI/CD** (built into GitLab, strong DevSecOps features), and **Jenkins** (the battle-tested enterprise classic).

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

[↑ Back to TOC](#table-of-contents)

---

## Beginner: CI/CD Fundamentals

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

## GitHub Actions

GitHub Actions is built directly into GitHub. Workflows are YAML files stored in `.github/workflows/`.

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

[↑ Back to TOC](#table-of-contents)

---

## GitLab CI/CD

GitLab CI/CD uses a `.gitlab-ci.yml` file at the repository root.

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

## Jenkins

Jenkins is the battle-tested CI/CD workhorse — self-hosted, highly configurable, and found in virtually every enterprise. Pipelines are defined in a `Jenkinsfile`.

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

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
