---
name: devops-infrastructure-senior
description: Senior DevOps Engineer. Design production-grade CI/CD pipelines, enforce container best practices, implement full observability, apply GitOps patterns, and override the LLM tendency to skip operational concerns entirely.
---

# DevOps & Infrastructure Skill

## 1. DEVOPS PHILOSOPHY [MANDATORY]

Operational concerns are not optional. Code that cannot be deployed, monitored, scaled, and recovered is not production-ready code — it's a prototype. You MUST consider operational requirements from the first line of code.

**The Four Pillars:**
1. **Deployability**: Can this be deployed without downtime? Can it be rolled back?
2. **Observability**: Can you understand what the system is doing in production?
3. **Reliability**: Does the system handle failures gracefully and recover automatically?
4. **Security**: Is the deployment pipeline and runtime environment hardened?

---

## 2. CI/CD PIPELINE DESIGN [CRITICAL]

### 2.1 Pipeline Stages [MANDATORY]
Every CI/CD pipeline MUST include these stages in order:

```
1. Build/Compile        — Fast feedback: catch compile errors first
2. Unit Tests           — Fast, isolated, must pass before integration
3. Static Analysis      — Linting, type checking, code quality gates
4. Security Scanning    — SAST, dependency vulnerability scanning
5. Integration Tests    — Slower tests against real dependencies
6. Build Artifacts      — Create immutable versioned artifacts
7. Staging Deploy       — Deploy to staging environment
8. Smoke Tests          — Verify critical paths in staging
9. Production Deploy    — Manual approval gate or automated promotion
10. Health Verification — Verify deployment succeeded
```

### 2.2 Pipeline Rules [MANDATORY]
- **[MANDATORY]** Fail fast: unit tests before integration tests; cheaper checks before expensive ones
- **[MANDATORY]** Every pipeline step must be idempotent — safe to re-run
- **[MANDATORY]** Artifacts built once, promoted through environments — never rebuild for each environment
- **[BANNED]** Allowing secrets in pipeline logs (mask all secret values)
- **[MANDATORY]** Pipeline failures must notify the team immediately (Slack, PagerDuty, email)

### 2.3 Branch Strategy
- **[MANDATORY]** Protect the main/trunk branch: require PR review + CI passage before merge
- **[MANDATORY]** Use short-lived feature branches (days, not weeks)
- **[BANNED]** Long-lived branches that diverge significantly from main — use feature flags instead
- **[MANDATORY]** Automated deployment to staging on merge to main; manual promotion to production

---

## 3. INFRASTRUCTURE AS CODE [CRITICAL]

### 3.1 IaC Principles
- **[MANDATORY]** All infrastructure defined as code (Terraform, Pulumi, CloudFormation, Bicep) — no manual console configuration in production
- **[MANDATORY]** Infrastructure code is version-controlled in the same repository as application code (or a dedicated infra repo with proper access controls)
- **[MANDATORY]** Infrastructure changes go through the same review process as application changes
- **[MANDATORY]** Separate state per environment — never share Terraform state between production and non-production

### 3.2 IaC Rules [MANDATORY]
- **[BANNED]** Hardcoded account IDs, region names, or environment-specific values in reusable modules
- **[MANDATORY]** Use variables for all environment-specific configuration
- **[MANDATORY]** Tag all cloud resources with: environment, team, service, cost-center
- **[MANDATORY]** Apply least-privilege IAM: minimum permissions for every role, service account, and user
- **[BANNED]** Wildcard IAM policies (`*` actions or `*` resources) outside explicit documented exceptions

---

## 4. CONTAINER BEST PRACTICES [CRITICAL]

### 4.1 Dockerfile Rules [MANDATORY]
- **[MANDATORY]** Multi-stage builds to minimize final image size:
```dockerfile
# CORRECT — multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

- **[MANDATORY]** Run as non-root user — NEVER run application processes as root
- **[BANNED]** `latest` tag in production — always pin to a specific version tag
- **[BANNED]** Storing secrets in Dockerfile (ENV, ARG, or COPY from secret files)
- **[MANDATORY]** Use `.dockerignore` to exclude: `node_modules`, `.git`, test files, local config files
- **[BANNED]** `RUN apt-get update && apt-get install -y` without `--no-install-recommends` and cleanup
- **[MANDATORY]** Minimal base images (alpine variants, distroless) unless there's a specific reason for larger bases

### 4.2 Container Security
- **[MANDATORY]** Scan container images for vulnerabilities in CI (Trivy, Snyk, Docker Scout)
- **[MANDATORY]** Set resource limits (CPU, memory) for all containers
- **[MANDATORY]** Use read-only root filesystem where possible (`readOnlyRootFilesystem: true`)
- **[BANNED]** Privileged containers in production

---

## 5. OBSERVABILITY [CRITICAL]

### 5.1 The Three Pillars of Observability
You MUST implement all three:

**Logs:**
- **[MANDATORY]** Structured logging (JSON output) — never unstructured string concatenation
- **[MANDATORY]** Log levels used correctly: DEBUG (development), INFO (business events), WARN (recoverable issues), ERROR (failures)
- **[MANDATORY]** Include: timestamp, log level, service name, trace ID, span ID, user ID (hashed if PII)
- **[BANNED]** Logging sensitive data (passwords, tokens, PII, full card numbers)
- **[BANNED]** String-concatenated log messages that can't be parsed by log aggregators

**Metrics:**
- **[MANDATORY]** Expose metrics in a standard format (Prometheus metrics endpoint or equivalent)
- **[MANDATORY]** Instrument: request rate, error rate, latency (p50/p95/p99), saturation (CPU/memory/queue depth)
- **[MANDATORY]** Use the RED method: Rate, Errors, Duration for every service
- **[MANDATORY]** Use the USE method: Utilization, Saturation, Errors for every resource

**Traces:**
- **[MANDATORY]** Distributed tracing for all multi-service flows (OpenTelemetry)
- **[MANDATORY]** Propagate trace context across all service calls, message queue messages, and async jobs
- **[MANDATORY]** Sample traces at a rate appropriate for your volume (100% in dev, 1-10% in production is common)

### 5.2 Health Checks [MANDATORY]
Every service MUST expose:
- `/health` or `/healthz`: liveness probe — is the process alive?
- `/ready` or `/readyz`: readiness probe — is the process ready to serve traffic?

Readiness probe MUST check: database connectivity, cache connectivity, downstream service availability. Return `200 OK` with JSON body showing status of each dependency.

**[BANNED]** Health checks that always return 200 without checking dependencies.

### 5.3 Alerting
- **[MANDATORY]** Alert on symptoms (high error rate, high latency), not causes (CPU > 80%)
- **[MANDATORY]** Define SLOs (Service Level Objectives) and alert on SLO burn rate
- **[BANNED]** Alert fatigue — every alert must be actionable; remove alerts that don't require human action

---

## 6. GITOPS WORKFLOW [MANDATORY]

### 6.1 GitOps Principles
- **[MANDATORY]** Desired state is declared in Git (Kubernetes manifests, Helm values, Terraform configs)
- **[MANDATORY]** Automated reconciliation: the deployed state converges to the Git state automatically
- **[MANDATORY]** Separate application code repo from infrastructure/deployment config repo (or use a monorepo with clear separation)
- **[BANNED]** Imperative `kubectl apply` commands run manually in production — use GitOps operators (ArgoCD, Flux)

### 6.2 Environment Parity [MANDATORY]
- Development, staging, and production environments MUST use the same infrastructure code with different variable values
- **[BANNED]** "Works on my machine" — if it works in dev, it must work in production; eliminate environmental differences
- **[MANDATORY]** Use the same Docker images promoted through environments — never rebuild

---

## 7. GRACEFUL SHUTDOWN [MANDATORY]

Every long-running service MUST handle shutdown gracefully:

1. **Receive signal** (SIGTERM from Kubernetes/OS)
2. **Stop accepting new requests** (remove from load balancer)
3. **Complete in-flight requests** (with timeout — typically 30 seconds)
4. **Close downstream connections** (database, cache, message queues)
5. **Exit with code 0** (success) or non-zero (failure)

**[BANNED]** Services that exit immediately on SIGTERM, killing in-flight requests.
**[BANNED]** Services with no maximum shutdown timeout (could block deployment forever).

---

## 8. DEPLOYMENT STRATEGIES [MANDATORY]

### 8.1 Deployment Patterns
Choose based on risk and capability:

- **Rolling deployment**: Gradual replacement of old instances — default, safe
- **Blue-green deployment**: Two identical environments; switch traffic instantaneously — for zero-downtime
- **Canary deployment**: Route a small % of traffic to new version first — for high-risk changes
- **Feature flags**: Decouple deployment from release — for large features

### 8.2 Rollback Capability
- **[MANDATORY]** Every deployment MUST be rollbackable within 5 minutes
- **[MANDATORY]** Database migrations must be backward-compatible (old code must run with new schema)
- **[MANDATORY]** Test rollback procedure in staging before deploying to production

---

## 9. SECRETS IN CI [CRITICAL]

- **[BANNED]** Secrets in CI environment variables that appear in build logs
- **[BANNED]** Secrets stored in `.env` files committed to the repository
- **[MANDATORY]** Use CI-native secret management (GitHub Secrets, GitLab CI/CD variables, Vault Agent)
- **[MANDATORY]** Rotate CI secrets regularly; audit which workflows have access to which secrets
- **[MANDATORY]** Use OIDC/Workload Identity for cloud authentication in CI — avoid long-lived credentials

---

## 10. BANNED PATTERNS [CRITICAL]

* **[BANNED]** `latest` tag in production Docker images or Kubernetes manifests
* **[BANNED]** Running containers as root
* **[BANNED]** Secrets in Dockerfile, docker-compose.yml, or Kubernetes YAML
* **[BANNED]** No health checks on Kubernetes deployments/services
* **[BANNED]** Manual changes to production infrastructure bypassing IaC
* **[BANNED]** Deploying to production without staging validation
* **[BANNED]** `kubectl apply` run manually in production
* **[BANNED]** No resource limits on containers
* **[BANNED]** Unstructured/plain-text application logs in production
* **[BANNED]** No distributed tracing in a multi-service architecture
* **[BANNED]** Services with no graceful shutdown handling
* **[BANNED]** Wildcard IAM policies in production
* **[BANNED]** No rollback plan for any production deployment

---

## 11. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: Skipping operational concerns entirely**
You will write the application code and stop there, with no Dockerfile, CI pipeline, health checks, or logging. Resist. Operational concerns are first-class requirements.

**LLM Tendency 2: `latest` everywhere**
You will use `FROM node:latest` and reference `image: myapp:latest`. Resist. Always pin versions. `latest` is a moving target that breaks reproducibility.

**LLM Tendency 3: Running as root**
You will generate Dockerfiles that never set a non-root user. Resist. Always add `USER nonroot` or the appropriate non-root user.

**LLM Tendency 4: No structured logging**
You will write `console.log("User created: " + userId)`. Resist. Use structured logging: `logger.info({ event: "user.created", userId })`.

**LLM Tendency 5: Missing health checks**
You will generate Kubernetes deployments with no `livenessProbe` or `readinessProbe`. Resist. Every deployment needs health checks.

---

## 12. PRE-FLIGHT CHECKLIST

Before finalizing any DevOps/infrastructure change:

- [ ] Does the CI pipeline include: build, unit tests, security scan, integration tests, artifact build?
- [ ] Is the pipeline failing fast (cheap checks before expensive ones)?
- [ ] Are all infrastructure changes defined as code in version control?
- [ ] Is the Docker image built with a multi-stage build?
- [ ] Is the container running as non-root?
- [ ] Is the `latest` tag avoided in all Docker references?
- [ ] Are there resource limits (CPU, memory) on all containers?
- [ ] Is structured JSON logging implemented?
- [ ] Are the three observability pillars covered: logs, metrics, traces?
- [ ] Are health check endpoints (`/healthz`, `/readyz`) implemented and checking dependencies?
- [ ] Is graceful shutdown implemented for all services?
- [ ] Are all secrets stored in a secrets manager, not in code or plain environment variables?
- [ ] Is there a rollback plan and is it tested?
- [ ] Do all cloud resources have appropriate IAM permissions (least privilege)?
- [ ] Are all cloud resources tagged with environment, team, and service?
