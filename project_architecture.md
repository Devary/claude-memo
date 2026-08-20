---
name: project-architecture
description: "Full architecture overview of the Devary/Redzone microservices ecosystem — all projects, their roles, and how they interconnect"
metadata: 
  node_type: memory
  type: project
  originSessionId: f72047e5-3b36-4037-9590-0c35dcc016c4
---

# Redzone Ecosystem Architecture

## Overview
A full-stack microservices platform with Angular microfrontends (MFE), Quarkus backends, an API gateway, a central auth service, and a Kubernetes-based infra layer. All Java services inherit from a shared Maven parent POM.

## Projects

### Backend (Java/Quarkus)

**quar-parent** (Maven parent POM)
- Central dependency/plugin management for all Quarkus services
- Quarkus 3.35.2, Java 17, Maven
- Enforces 85% JaCoCo coverage, SonarQube, JFrog Artifactory deployment
- Defines build profiles: local, prod, native, with-database
- Property filtering with `@` delimiter for Vault paths and secrets
- All child services inherit CI/CD pipeline patterns from here

**quar-gateway** (port 5550)
- API gateway / reverse proxy for all backend microservices
- Routes `/{service}/{path}` to registered downstream services
- Auth: delegates token verification to redzone-api (`POST /auth/verify`)
- On success, forwards identity headers: `X-Authenticated-Subject`, `X-Authenticated-User`, `X-Authenticated-Roles`, `X-Authenticated-Groups`
- Public bypass routes: `POST auth/login-test`, `GET auth/username-availability/*`
- Service registry backed by config (ConfigBackedServiceRegistry)
- Vault integration for secrets in prod

**redzone-api** (port 5551)
- Central authentication & authorization decision service
- Integrates with Keycloak (OIDC/JWT) and LDAP directory
- Key endpoints:
  - `GET /auth/me` — current user context
  - `POST /auth/verify` — policy decision (used by gateway)
  - `POST /auth/login-test` — password grant via Keycloak
  - `GET /auth/username-availability/{username}` — LDAP check
  - `POST /auth/register` — LDAP user creation
  - `POST /subscriptions/requests` — subscription lifecycle
- Multi-backend subscription store: LDAP or in-memory (strategy pattern)
- Password validation: 12+ chars, upper/lower/digit/special
- Conservative auth: deny-by-default, admin roles get full access
- Vault-backed secrets in production

**quar-service-template** (port 5555, gRPC 9001)
- Blueprint/template for new Quarkus microservices
- REST + gRPC (Protobuf TestGrpc service)
- Hibernate Reactive + Panache, PostgreSQL
- Vault-backed config injection example
- Jenkins/K8s ready out of the box

### Frontend (Angular 21)

**ui-shell** (dev port 4206, base href `/ui-shell/`)
- MFE host (shell) application
- PrimeNG 21 with glassmorphism purple/white theme
- Auth flow: login directly to redzone-api (bypasses gateway auth), token in localStorage
- Hosts remote MFEs as iframes
- Passes auth token to child MFEs via postMessage
- Route discovery from gateway (`/routes`)
- Guards: auth.guard (protected routes), guest.guard (login page)
- Interceptor attaches Bearer token to gateway requests (skips public endpoints)

**ui-service-template** (dev port 4207, base href `/ui-service-template/`)
- Template/blueprint for new Angular MFE remote services
- Embedded as iframe in ui-shell
- Receives token from shell via postMessage (retries: 0ms, 300ms, 1000ms)
- Stores token in sessionStorage
- Calls gateway at `/gateway/service-template/` prefix
- Minimal single-component UI

### Infrastructure

**infra**
- Kubernetes deployment scripts and templates (no Helm, sed-based substitution)
- `deploy.sh`: universal script for CLI and Rundeck modes
  - Bootstraps Vault Kubernetes auth (policy, role, k8s auth method)
  - Hydrates secrets from Vault (JWT_ISSUER, KEYCLOAK_TOKEN_URL, etc.)
  - Creates/updates Deployment, Service, Ingress, ServiceMonitor
  - Validates rollout status
- k8s templates: deployment, service, Traefik IngressRoute (HTTP + gRPC), Prometheus ServiceMonitor
- Local infra: Docker Compose for Jenkins, Bamboo, Rundeck, RabbitMQ, UC4

## Deployment Flow
```
Jenkins (Jenkinsfile in each service)
  → build artifact (npm/maven)
  → docker build + push to Harbor registry
  → clone infra repo
  → trigger Rundeck with deploy.sh

Rundeck (deploy.sh)
  → Vault auth bootstrap
  → apply k8s manifests
  → kubectl rollout status
```

## Connectivity Map
```
Browser
  → nginx → ui-shell (MFE host)
              ↓ iframe
            ui-service-template (MFE remote)

ui-shell ──→ /gateway/* ──→ quar-gateway
ui-service-template ──→ /gateway/service-template/* ──→ quar-gateway

quar-gateway ──→ POST /auth/verify ──→ redzone-api
quar-gateway ──→ /{service}/{path} ──→ target service (e.g. quar-service-template)

redzone-api ──→ Keycloak (JWT issuance)
redzone-api ──→ LDAP (user/subscription store)

All services ──→ HashiCorp Vault (secrets at runtime)
All services ──→ Kubernetes (deployed via infra/deploy.sh)
All services ──→ Prometheus (ServiceMonitor at /q/metrics)
```

## Key Stack Decisions
- All Java services: Quarkus 3.35.2, Java 17, Maven, inherit quar-parent
- Secrets: HashiCorp Vault everywhere (no hardcoded secrets in prod)
- Auth: Keycloak issues JWTs; redzone-api validates and makes policy decisions; gateway enforces
- MFE pattern: iframe embedding with postMessage token passing
- CI/CD: Jenkins → Rundeck → Kubernetes
- Artifact registry: JFrog Artifactory (JARs), Harbor (Docker images)
- Ingress: Traefik with path stripping; gRPC via h2c
- Monitoring: Prometheus + kube-prometheus-stack with auto-discovery via ServiceMonitor labels

**Why:** This is a production Kubernetes-based platform with clear separation of concerns — auth, routing, UI orchestration, and individual services are all independent deployable units.
**How to apply:** When suggesting changes, consider the inheritance chain (quar-parent → child services), the gateway routing model, and the Vault-first secrets approach.
