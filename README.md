# 🏗️ SecureVault — Arquitectura de Microservicios FOSS

> **Proyecto:** SecureVault | **Licencia:** GNU AGPLv3 | **Versión:** 1.0.0  
> **Mantenido por:** Comunidad FOSS | **Estado:** Activo

---

## 📋 Tabla de Contenidos

1. [Visión General](#-visión-general)
2. [Diagrama de Arquitectura](#-diagrama-de-arquitectura)
3. [Stack Tecnológico](#-stack-tecnológico-completo)
4. [Flujo CI/CD DevSecOps](#-flujo-cicd-devsecops)
5. [Estructura del Repositorio](#-estructura-del-repositorio)
6. [Configuración y Despliegue](#-configuración-y-despliegue)
7. [Seguridad](#-seguridad)
8. [Monitoreo](#-monitoreo)
9. [Contribuir](#-contribuir)
10. [Licencias](#-licencias)

---

## 🌐 Visión General

**SecureVault** es una plataforma distribuida de análisis e inteligencia OSINT construida íntegramente sobre software libre y de código abierto (FOSS). Procesa contenido a través de un pipeline asíncrono de microservicios que incluye publicación, recopilación OSINT y análisis de sentimiento, con seguridad por diseño en cada capa.

### Principios de Diseño

- **FOSS First**: Todas las dependencias tienen licencias aprobadas por OSI
- **Security by Design**: JWT, cifrado en reposo, gestión de secretos con Vault
- **Observabilidad**: Métricas, logs y trazas centralizadas
- **GitOps**: Toda la infraestructura declarativa y versionada
- **Zero-Trust**: Autenticación en cada capa de comunicación

---

## 🗺️ Diagrama de Arquitectura

### Vista de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET / USUARIOS                                 │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │ HTTPS :443
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         CAPA DE PRESENTACIÓN                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │              Frontend SPA — React 18 + Vite                             │    │
│  │         Servido por Nginx 1.25 (reverse proxy + static files)           │    │
│  │         Puerto: 80/443 │ TLS terminado aquí                             │    │
│  └─────────────────────────────────┬───────────────────────────────────────┘   │
└────────────────────────────────────┼───────────────────────────────────────────┘
                                     │ HTTP/REST :8000
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CAPA DE API / GATEWAY                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │              API Gateway — FastAPI 0.111 + Uvicorn                      │    │
│  │   • Autenticación JWT (PyJWT)    • Rate Limiting (slowapi)              │    │
│  │   • Validación de esquemas       • OpenAPI / Swagger UI                 │    │
│  │   • CORS, CSP headers            • Audit logging                        │    │
│  └───────┬─────────────────────────────────┬───────────────────────────────┘   │
└──────────┼─────────────────────────────────┼───────────────────────────────────┘
           │ AMQP :5672                       │ TCP :5432
           ▼                                  ▼
┌─────────────────────────┐       ┌───────────────────────────────────────────────┐
│   MESSAGE BROKER         │       │              CAPA DE DATOS                     │
│  ┌───────────────────┐  │       │  ┌─────────────────────────────────────────┐  │
│  │   RabbitMQ 3.13   │  │       │  │  PostgreSQL 16 (cifrado pgcrypto)        │  │
│  │  • exchange: main │  │       │  │  • Datos en reposo cifrados (AES-256)   │  │
│  │  • queues:        │  │       │  │  • TLS en tránsito                      │  │
│  │    - publish.q    │  │       │  │  • Row-level Security (RLS)             │  │
│  │    - osint.q      │  │       │  │  • Backups cifrados (pgBackRest)        │  │
│  │    - sentiment.q  │  │       │  └─────────────────────────────────────────┘  │
│  │  • Management UI  │  │       └───────────────────────────────────────────────┘
│  │    :15672         │  │
│  └────────┬──────────┘  │       ┌───────────────────────────────────────────────┐
│           │ AMQP        │       │           GESTIÓN DE SECRETOS                  │
└───────────┼─────────────┘       │  ┌─────────────────────────────────────────┐  │
            │                     │  │       HashiCorp Vault 1.16              │  │
            ▼                     │  │  • KV secrets engine                    │  │
┌─────────────────────────────────┴──│  • PKI (CA interna / TLS certs)        │  │
│         CAPA DE WORKERS             │  • Database secrets (rotación auto)    │  │
│  ┌──────────────────────────────┐  │  • Transit encryption                   │  │
│  │   Worker: Publicación        │  │  • Audit log                            │  │
│  │   • Procesamiento de posts   │  └─────────────────────────────────────────┘  │
│  │   • Celery + Redis            │  └───────────────────────────────────────────┘
│  └──────────────────────────────┘
│  ┌──────────────────────────────┐
│  │   Worker: OSINT              │       ┌───────────────────────────────────┐
│  │   • Shodan, TheHarvester     │       │         OBSERVABILIDAD             │
│  │   • Recon-ng integration     │       │  ┌─────────────────────────────┐  │
│  │   • Scrapy + BeautifulSoup   │       │  │  Prometheus + Grafana        │  │
│  └──────────────────────────────┘       │  │  Loki + Promtail             │  │
│  ┌──────────────────────────────┐       │  │  Jaeger (trazas distribuidas)│  │
│  │   Worker: Sentimiento        │       │  │  AlertManager                │  │
│  │   • NLTK + TextBlob          │       │  └─────────────────────────────┘  │
│  │   • spaCy (NLP)              │       └───────────────────────────────────┘
│  │   • Clasificación multi-lang │
│  └──────────────────────────────┘
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Diagrama de Flujo de Datos (PlantUML)

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FAFAFA
skinparam componentStyle rectangle

actor Usuario
component "Nginx\n(Reverse Proxy)" as nginx
component "React SPA" as spa
component "API Gateway\n(FastAPI + JWT)" as api
component "RabbitMQ\n(Message Broker)" as mq
component "Worker\nPublicación" as wp
component "Worker\nOSINT" as wo
component "Worker\nSentimiento" as ws
database "PostgreSQL\n(Cifrada)" as db
component "HashiCorp Vault\n(Secretos)" as vault
component "Prometheus\n+ Grafana" as obs

Usuario --> nginx : HTTPS :443
nginx --> spa : HTTP :3000 (static)
spa --> api : REST/JSON :8000
api --> mq : AMQP :5672 (publish)
api --> db : TLS :5432
api --> vault : HTTPS :8200 (secrets)

mq --> wp : AMQP (publish.queue)
mq --> wo : AMQP (osint.queue)
mq --> ws : AMQP (sentiment.queue)

wp --> db : TLS :5432
wo --> db : TLS :5432
ws --> db : TLS :5432

wp --> obs : metrics :9090
wo --> obs : metrics :9090
ws --> obs : metrics :9090
api --> obs : metrics :9090
@enduml
```

### Diagrama de Red / Namespaces Kubernetes

```
Namespace: securevault-frontend
├── Deployment: spa (React + Nginx)
│   └── Service: ClusterIP :80
│   └── Ingress: nginx-ingress → :443

Namespace: securevault-api
├── Deployment: api-gateway (FastAPI)
│   └── Service: ClusterIP :8000
│   └── HPA: min=2, max=10, CPU>70%

Namespace: securevault-messaging
├── StatefulSet: rabbitmq (cluster 3 nodos)
│   └── Service: ClusterIP :5672, :15672

Namespace: securevault-workers
├── Deployment: worker-publish  (replicas: 3)
├── Deployment: worker-osint    (replicas: 2)
├── Deployment: worker-sentiment (replicas: 2)

Namespace: securevault-data
├── StatefulSet: postgresql (primary + replica)
│   └── PVC: encrypted-storage (50Gi)

Namespace: securevault-vault
├── StatefulSet: vault (HA 3 nodos)
│   └── PVC: vault-storage (10Gi)

Namespace: monitoring
├── Deployment: prometheus
├── Deployment: grafana
├── DaemonSet: promtail (log collection)
├── Deployment: loki
└── Deployment: jaeger
```

---

## 🛠️ Stack Tecnológico Completo

### Capa de Presentación

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| Framework UI | React | 18.x | MIT |
| Build Tool | Vite | 5.x | MIT |
| Estado global | Zustand | 4.x | MIT |
| HTTP Client | Axios | 1.x | MIT |
| UI Components | shadcn/ui + Radix UI | latest | MIT |
| Estilos | Tailwind CSS | 3.x | MIT |
| Router | React Router v6 | 6.x | MIT |
| Formularios | React Hook Form + Zod | latest | MIT |
| Servidor web | Nginx | 1.25 | BSD-2 |
| Contenedor | Alpine Linux | 3.19 | GPL/MIT mix |

### Capa de API Gateway

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| Framework API | FastAPI | 0.111 | MIT |
| Servidor ASGI | Uvicorn | 0.29 | BSD-3 |
| Autenticación | PyJWT | 2.8 | MIT |
| Validación | Pydantic v2 | 2.7 | MIT |
| Rate Limiting | slowapi | 0.1.9 | MIT |
| ORM | SQLAlchemy 2.0 | 2.0 | MIT |
| Migraciones DB | Alembic | 1.13 | MIT |
| Cliente AMQP | aio-pika | 9.4 | Apache 2.0 |
| Cache | Redis (aioredis) | 7.x | BSD-3 |
| Trazas | OpenTelemetry | 1.x | Apache 2.0 |

### Capa de Message Broker

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| Message Broker | RabbitMQ | 3.13 | MPL 2.0 |
| Plugin gestión | rabbitmq-management | 3.13 | MPL 2.0 |
| Plugin métricas | rabbitmq-prometheus | 3.13 | MPL 2.0 |
| Erlang runtime | Erlang/OTP | 26.x | Apache 2.0 |

### Capa de Workers

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| Task Queue | Celery | 5.4 | BSD-3 |
| Backend Celery | Redis | 7.x | BSD-3 |
| NLP — Sentimiento | NLTK | 3.8 | Apache 2.0 |
| NLP — TextBlob | TextBlob | 0.18 | MIT |
| NLP avanzado | spaCy | 3.7 | MIT |
| Web scraping | Scrapy | 2.11 | BSD-3 |
| HTTP requests | httpx | 0.27 | BSD-3 |
| OSINT framework | TheHarvester | 4.x | GPL v2 |
| OSINT framework | Recon-ng | 5.x | GPL v3 |
| HTML parsing | BeautifulSoup4 | 4.12 | MIT |
| DNS recon | dnspython | 2.6 | ISC |

### Capa de Base de Datos

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| RDBMS | PostgreSQL | 16.x | PostgreSQL License |
| Cifrado columnas | pgcrypto | incluida | PostgreSQL License |
| Backup cifrado | pgBackRest | 2.51 | MIT |
| Connection pooling | PgBouncer | 1.22 | ISC |
| Alta disponibilidad | Patroni | 3.3 | MIT |

### Gestión de Secretos

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| Secrets manager | HashiCorp Vault | 1.16 | BSL 1.1 (*) |
| Alternativa FOSS | OpenBao | 2.x | MPL 2.0 |
| PKI interna | Vault PKI Engine | 1.16 | BSL 1.1 (*) |

> (*) HashiCorp cambió a BSL 1.1 en v1.14+. **OpenBao** es el fork comunitario bajo MPL 2.0 y es 100% compatible. Se recomienda OpenBao para despliegues puramente FOSS.

### Infraestructura y Orquestación

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| Orquestación | Kubernetes | 1.30 | Apache 2.0 |
| Distribución K8s | k3s (edge) / kubeadm | latest | Apache 2.0 |
| Ingress Controller | ingress-nginx | 1.10 | Apache 2.0 |
| Service Mesh | Cilium | 1.15 | Apache 2.0 |
| IaC | Terraform | 1.8 | BSL 1.1 (*) |
| Alt. IaC FOSS | OpenTofu | 1.7 | MPL 2.0 |
| Config Mgmt | Ansible | 2.16 | GPL v3 |
| Helm | Helm | 3.14 | Apache 2.0 |
| Registry | Harbor | 2.10 | Apache 2.0 |
| GitOps | Flux CD | 2.3 | Apache 2.0 |

> (*) Para proyectos FOSS estrictos, usar **OpenTofu** como alternativa a Terraform.

### Monitoreo y Observabilidad

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| Métricas | Prometheus | 2.51 | Apache 2.0 |
| Dashboards | Grafana | 10.4 | AGPL v3 |
| Logs agregación | Loki | 3.0 | AGPL v3 |
| Log shipping | Promtail | 3.0 | Apache 2.0 |
| Trazas | Jaeger | 1.57 | Apache 2.0 |
| Alertas | AlertManager | 0.27 | Apache 2.0 |
| Uptime | Uptime Kuma | 1.23 | MIT |
| K8s dashboards | Kubernetes Dashboard | 3.0 | Apache 2.0 |

### CI/CD y DevSecOps

| Componente | Tecnología | Versión | Licencia |
|---|---|---|---|
| CI/CD Platform | Forgejo Actions / GitLab CI | latest | MIT / MIT |
| Container Build | Buildah / Kaniko | latest | Apache 2.0 |
| SAST | Semgrep OSS | latest | LGPL v2.1 |
| SAST Python | Bandit | latest | Apache 2.0 |
| Secrets detection | Gitleaks | latest | MIT |
| Secrets detection | TruffleHog | latest | AGPL v3 |
| SCA / Deps | Trivy (deps) | latest | Apache 2.0 |
| Image scan | Trivy | latest | Apache 2.0 |
| DAST | OWASP ZAP | latest | Apache 2.0 |
| IaC scan | Checkov | latest | Apache 2.0 |
| IaC scan | tfsec / tofu-scan | latest | MIT |
| Firma imágenes | Cosign (Sigstore) | latest | Apache 2.0 |
| SBOM | Syft | latest | Apache 2.0 |
| Policy | OPA / Kyverno | latest | Apache 2.0 |
| Fuzzing | OSS-Fuzz / atheris | latest | Apache 2.0 |

---

## 🔄 Flujo CI/CD DevSecOps

### Pipeline Completo

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      PIPELINE DEVSECOPS — SECUREVAULT                            │
│                                                                                   │
│  ① PLAN          ② CODE           ③ BUILD          ④ TEST                       │
│  ──────────      ──────────       ──────────        ──────────                   │
│  • Jira/Plane    • Pre-commit     • Buildah/        • Unit tests                 │
│    (tickets)       hooks:           Kaniko:           (pytest/jest)               │
│  • Threat          - Gitleaks       - Multi-stage   • Integration                │
│    Modeling        - Semgrep        - Rootless        tests                       │
│  • Revisión        - black/         - Non-root      • Contract tests              │
│    de seguridad      ruff             user             (Pact)                      │
│  • Diseño        • Conventional   • SBOM gen        • SAST:                      │
│    de API          Commits          (Syft)            - Semgrep                   │
│                  • Branch          • Sign image       - Bandit                   │
│                    protection       (Cosign)         • SCA (Trivy deps)           │
│                                                      • Secrets scan               │
│                                                        (Gitleaks)                 │
│                                                                                   │
│  ⑤ RELEASE       ⑥ DEPLOY         ⑦ OPERATE        ⑧ MONITOR                  │
│  ──────────      ──────────        ──────────        ──────────                   │
│  • Image scan    • GitOps          • K8s Runtime     • Prometheus                 │
│    (Trivy)         (FluxCD)          Security:         métricas                   │
│  • IaC scan      • Helm charts       - Falco          (RED: Rate,                 │
│    (Checkov)     • Canary/            - Seccomp         Errors,                   │
│  • DAST            Blue-Green         - AppArmor        Duration)                 │
│    (OWASP ZAP)   • Smoke tests     • Vault agent     • Grafana                   │
│  • Policy gate   • Rollback          sidecar           dashboards                 │
│    (OPA/Kyverno)   automático       (secret sync)   • Loki logs                  │
│  • SemVer tag    • Ansible         • Network         • Jaeger traces              │
│  • Changelog       playbooks         policies        • AlertManager               │
│    (git-cliff)   • Notify Slack      (Cilium)        • Uptime Kuma               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Ejemplo de Workflow GitHub/Forgejo Actions

```yaml
# .forgejo/workflows/devsecops.yml
# Licencia: AGPLv3

name: DevSecOps Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: harbor.internal.securevault.io
  IMAGE_NAME: securevault/api-gateway

jobs:
  # ─── FASE 1: DETECCIÓN DE SECRETOS ────────────────────────────────────────
  secrets-scan:
    name: 🔐 Secrets Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Historial completo para Gitleaks
      
      - name: Gitleaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: TruffleHog scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

  # ─── FASE 2: SAST ─────────────────────────────────────────────────────────
  sast:
    name: 🛡️ SAST Analysis
    needs: secrets-scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Semgrep SAST
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/python
            p/fastapi
            p/jwt
            p/sql-injection
            p/owasp-top-ten
      
      - name: Bandit Python SAST
        run: |
          pip install bandit
          bandit -r ./api_gateway ./workers -f json -o bandit-report.json
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: bandit-report.json

  # ─── FASE 3: BUILD ────────────────────────────────────────────────────────
  build:
    name: 🏗️ Build & Sign Image
    needs: [sast]
    runs-on: ubuntu-latest
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Build with Buildah
        id: build
        run: |
          buildah bud \
            --layers \
            --tag $REGISTRY/$IMAGE_NAME:${{ github.sha }} \
            --file docker/api-gateway/Dockerfile .
      
      - name: Generate SBOM
        run: |
          syft $REGISTRY/$IMAGE_NAME:${{ github.sha }} \
            -o spdx-json=sbom.spdx.json
      
      - name: Scan image (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1  # Falla si hay CVE críticos
      
      - name: Sign image (Cosign)
        run: |
          cosign sign --yes \
            --key env://COSIGN_PRIVATE_KEY \
            $REGISTRY/$IMAGE_NAME:${{ github.sha }}
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}

  # ─── FASE 4: IaC SCAN ─────────────────────────────────────────────────────
  iac-scan:
    name: 📋 IaC Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Checkov IaC scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./infra
          framework: terraform,kubernetes,helm
          output_format: sarif
          output_file_path: checkov-results.sarif
          soft_fail: false
      
      - name: tfsec scan
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: ./infra/terraform

  # ─── FASE 5: TESTS ────────────────────────────────────────────────────────
  test:
    name: 🧪 Tests & Coverage
    needs: build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
      redis:
        image: redis:7-alpine
    steps:
      - uses: actions/checkout@v4
      
      - name: Run unit tests
        run: |
          pytest api_gateway/tests/ \
            --cov=api_gateway \
            --cov-report=xml \
            --cov-fail-under=80
      
      - name: Run integration tests
        run: |
          pytest tests/integration/ -v --timeout=60

  # ─── FASE 6: DAST ─────────────────────────────────────────────────────────
  dast:
    name: 🕷️ DAST (OWASP ZAP)
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: ZAP API Scan
        uses: zaproxy/action-api-scan@v0.7.0
        with:
          target: 'http://staging.securevault.internal/openapi.json'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'

  # ─── FASE 7: DEPLOY ───────────────────────────────────────────────────────
  deploy:
    name: 🚀 Deploy via GitOps
    needs: [dast, iac-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Update Helm values
        run: |
          # Actualiza el tag de imagen en el repositorio de GitOps
          yq e '.image.tag = "${{ github.sha }}"' \
            -i gitops/helm/api-gateway/values.yaml
      
      - name: Commit & push (FluxCD detecta el cambio)
        run: |
          git config user.email "ci@securevault.io"
          git config user.name "CI Bot"
          git commit -am "chore: update api-gateway image to ${{ github.sha }}"
          git push
```

### Stages del Pipeline — Detalle de Seguridad

```
ETAPA           HERRAMIENTA         ACCIÓN                      FALLA EN
─────────────── ─────────────────── ─────────────────────────── ──────────────
Pre-commit      Gitleaks            Detecta secretos en código   Commit
Pre-commit      Semgrep             Linting de seguridad local   Commit
CI: Secrets     TruffleHog          Escaneo histórico de git     PR
CI: SAST        Semgrep OSS         Vulnerabilidades en código   PR
CI: SAST        Bandit              Python-specific SAST         PR
CI: SCA         Trivy (deps)        CVEs en dependencias         Build
CI: Build       Buildah             Imagen rootless, no-new-priv Build
CI: SBOM        Syft                Genera SBOM SPDX 2.3         Build
CI: Image Scan  Trivy               CVEs CRITICAL/HIGH en imagen Push
CI: Sign        Cosign              Firma con clave privada      Push
CI: IaC         Checkov             Mala configuración Terraform PR
CI: IaC         tfsec               Terraform security rules     PR
CI: Policy      OPA / Kyverno       Validación de manifiestos K8 Deploy
CI: DAST        OWASP ZAP           Escaneo dinámico API REST    Staging
Runtime         Falco               Detección anomalías K8s      Alert
Runtime         Cilium              Network policies             Drop packet
```

---

## 📁 Estructura del Repositorio

```
securevault/
│
├── 📄 README.md                        ← Este archivo
├── 📄 LICENSE                          ← GNU AGPLv3
├── 📄 CONTRIBUTING.md                  ← Guía de contribución
├── 📄 SECURITY.md                      ← Política de seguridad / divulgación
├── 📄 CHANGELOG.md                     ← Generado con git-cliff
├── 📄 .editorconfig                    ← Configuración de editor
├── 📄 .gitignore
├── 📄 .gitleaks.toml                   ← Config detección de secretos
├── 📄 .pre-commit-config.yaml          ← Hooks pre-commit
│
├── 🖥️  frontend/                        ← React SPA + Nginx
│   ├── public/
│   ├── src/
│   │   ├── assets/
│   │   ├── components/
│   │   │   ├── ui/                    ← Componentes shadcn/ui
│   │   │   ├── layout/                ← Header, Sidebar, Footer
│   │   │   └── features/              ← Componentes por funcionalidad
│   │   ├── hooks/                     ← Custom React hooks
│   │   ├── lib/
│   │   │   ├── api.ts                 ← Cliente Axios configurado
│   │   │   └── auth.ts                ← Gestión JWT en cliente
│   │   ├── pages/
│   │   ├── store/                     ← Zustand stores
│   │   ├── types/                     ← TypeScript types
│   │   └── main.tsx
│   ├── docker/
│   │   ├── Dockerfile                 ← Multi-stage: build + nginx
│   │   └── nginx.conf                 ← Config nginx con headers CSP
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── tailwind.config.ts
│
├── 🔌 api_gateway/                     ← FastAPI + JWT Gateway
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                    ← FastAPI app factory
│   │   ├── config.py                  ← Settings (Pydantic BaseSettings)
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── auth.py            ← Login, logout, refresh token
│   │   │   │   ├── publish.py         ← Endpoints publicación
│   │   │   │   ├── osint.py           ← Endpoints OSINT
│   │   │   │   └── sentiment.py       ← Endpoints análisis
│   │   ├── core/
│   │   │   ├── security.py            ← JWT, hashing, validación
│   │   │   ├── deps.py                ← Dependencias FastAPI
│   │   │   └── middleware.py          ← Rate limit, CORS, logging
│   │   ├── models/                    ← SQLAlchemy 2.0 models
│   │   ├── schemas/                   ← Pydantic schemas
│   │   ├── crud/                      ← CRUD operations
│   │   └── broker/
│   │       └── publisher.py           ← aio-pika: publicar a RabbitMQ
│   ├── migrations/                    ← Alembic migrations
│   ├── tests/
│   │   ├── unit/
│   │   └── integration/
│   ├── docker/
│   │   └── Dockerfile                 ← Multi-stage Python rootless
│   ├── pyproject.toml
│   └── requirements/
│       ├── base.txt
│       ├── dev.txt
│       └── prod.txt
│
├── ⚙️  workers/                         ← Celery Workers
│   ├── shared/                        ← Código compartido entre workers
│   │   ├── db.py                      ← Pool de conexiones DB
│   │   ├── vault_client.py            ← Cliente HashiCorp Vault
│   │   └── telemetry.py               ← OpenTelemetry setup
│   │
│   ├── publish/                       ← Worker de Publicación
│   │   ├── worker.py
│   │   ├── tasks/
│   │   │   ├── process_post.py
│   │   │   └── notify.py
│   │   ├── tests/
│   │   └── docker/
│   │       └── Dockerfile
│   │
│   ├── osint/                         ← Worker OSINT
│   │   ├── worker.py
│   │   ├── tasks/
│   │   │   ├── harvester.py           ← TheHarvester integration
│   │   │   ├── recon.py               ← Recon-ng integration
│   │   │   ├── scraper.py             ← Scrapy spiders
│   │   │   └── dns_recon.py           ← dnspython
│   │   ├── tests/
│   │   └── docker/
│   │       └── Dockerfile
│   │
│   ├── sentiment/                     ← Worker Análisis de Sentimiento
│   │   ├── worker.py
│   │   ├── tasks/
│   │   │   ├── nltk_analyzer.py       ← VADER + NLTK
│   │   │   ├── textblob_analyzer.py   ← TextBlob polarity
│   │   │   └── spacy_pipeline.py      ← spaCy NLP pipeline
│   │   ├── models/                    ← Modelos NLP pre-entrenados
│   │   ├── tests/
│   │   └── docker/
│   │       └── Dockerfile
│   │
│   └── docker-compose.workers.yml
│
├── 🏗️  infra/                           ← Infraestructura como Código
│   ├── terraform/                     ← (OpenTofu compatible)
│   │   ├── environments/
│   │   │   ├── staging/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   └── terraform.tfvars.example
│   │   │   └── production/
│   │   │       ├── main.tf
│   │   │       └── variables.tf
│   │   └── modules/
│   │       ├── kubernetes/            ← K8s cluster provisioning
│   │       ├── postgresql/            ← Managed DB config
│   │       ├── networking/            ← VPC, subnets, firewall
│   │       └── vault/                 ← Vault cluster provisioning
│   │
│   ├── ansible/
│   │   ├── inventory/
│   │   │   ├── staging.yml
│   │   │   └── production.yml
│   │   ├── playbooks/
│   │   │   ├── bootstrap.yml          ← Setup inicial de nodos
│   │   │   ├── harden.yml             ← CIS hardening
│   │   │   ├── vault-init.yml         ← Inicializar Vault
│   │   │   └── deploy.yml             ← Despliegue de servicios
│   │   └── roles/
│   │       ├── common/
│   │       ├── docker/
│   │       ├── kubernetes/
│   │       └── vault/
│   │
│   ├── kubernetes/                    ← Manifiestos K8s (Helm)
│   │   ├── base/                      ← Kustomize base
│   │   │   ├── namespaces.yaml
│   │   │   ├── rbac.yaml
│   │   │   └── network-policies.yaml
│   │   └── helm/
│   │       ├── api-gateway/
│   │       │   ├── Chart.yaml
│   │       │   ├── values.yaml
│   │       │   ├── values-staging.yaml
│   │       │   └── templates/
│   │       │       ├── deployment.yaml
│   │       │       ├── service.yaml
│   │       │       ├── ingress.yaml
│   │       │       ├── hpa.yaml
│   │       │       └── servicemonitor.yaml
│   │       ├── frontend/
│   │       ├── workers/
│   │       ├── rabbitmq/
│   │       ├── postgresql/
│   │       └── vault/
│   │
│   └── vault-policies/                ← Vault ACL policies
│       ├── api-gateway-policy.hcl
│       ├── worker-publish-policy.hcl
│       ├── worker-osint-policy.hcl
│       └── worker-sentiment-policy.hcl
│
├── 📊 monitoring/                      ← Observabilidad
│   ├── prometheus/
│   │   ├── prometheus.yml             ← Scrape config
│   │   ├── rules/
│   │   │   ├── api-alerts.yml         ← Alertas API (latency, errors)
│   │   │   ├── worker-alerts.yml      ← Alertas workers
│   │   │   ├── db-alerts.yml          ← Alertas PostgreSQL
│   │   │   └── security-alerts.yml    ← Alertas Falco/seguridad
│   │   └── targets/
│   │       └── services.yml
│   │
│   ├── grafana/
│   │   ├── dashboards/
│   │   │   ├── api-gateway.json       ← RED metrics dashboard
│   │   │   ├── workers.json           ← Worker queue metrics
│   │   │   ├── postgresql.json        ← DB performance
│   │   │   ├── rabbitmq.json          ← Broker metrics
│   │   │   ├── kubernetes.json        ← Cluster overview
│   │   │   └── security.json          ← Falco events
│   │   └── provisioning/
│   │       ├── datasources/
│   │       └── dashboards/
│   │
│   ├── loki/
│   │   ├── loki-config.yaml
│   │   └── promtail-config.yaml
│   │
│   └── jaeger/
│       └── jaeger-config.yaml
│
├── 🔄 .forgejo/                        ← CI/CD (Forgejo Actions)
│   └── workflows/
│       ├── devsecops.yml              ← Pipeline completo (ver arriba)
│       ├── pr-checks.yml             ← Checks en PRs
│       ├── release.yml               ← Release y changelog automático
│       ├── dependency-update.yml     ← Renovate/Dependabot equivalente
│       └── security-audit.yml        ← Auditoría semanal de seguridad
│
└── 📚 docs/                            ← Documentación
    ├── architecture/
    │   ├── decisions/                 ← ADRs (Architecture Decision Records)
    │   │   ├── ADR-001-foss-only.md
    │   │   ├── ADR-002-jwt-auth.md
    │   │   └── ADR-003-openbao-vault.md
    │   ├── diagrams/
    │   │   ├── architecture.puml
    │   │   ├── data-flow.puml
    │   │   └── deployment.puml
    │   └── threat-model.md
    ├── api/
    │   └── openapi.yaml              ← Spec OpenAPI 3.1
    ├── operations/
    │   ├── runbooks/
    │   │   ├── incident-response.md
    │   │   ├── vault-unseal.md
    │   │   └── db-failover.md
    │   └── onboarding.md
    └── security/
        ├── hardening-guide.md
        └── vulnerability-disclosure.md
```

---

## ⚙️ Configuración y Despliegue

### Requisitos Previos

```bash
# Herramientas requeridas (todas FOSS)
kubectl >= 1.30      # Kubernetes CLI
helm >= 3.14         # Kubernetes package manager
opentofu >= 1.7      # IaC (alternativa FOSS a Terraform)
ansible >= 2.16      # Configuration management
vault >= 1.16        # O: openbao >= 2.x
buildah >= 1.35      # Container build (sin Docker daemon)
cosign >= 2.x        # Image signing
```

### Inicio Rápido (Desarrollo Local)

```bash
# 1. Clonar el repositorio
git clone https://github.com/tu-org/securevault.git
cd securevault

# 2. Instalar hooks de pre-commit
pip install pre-commit
pre-commit install

# 3. Copiar variables de entorno
cp api_gateway/.env.example api_gateway/.env
cp workers/.env.example workers/.env

# 4. Levantar stack completo (desarrollo)
docker compose -f docker-compose.yml \
               -f docker-compose.vault.yml \
               -f docker-compose.workers.yml up -d

# 5. Inicializar la base de datos
docker compose exec api_gateway alembic upgrade head

# 6. Verificar estado
docker compose ps
curl http://localhost:8000/health
```

### Variables de Entorno (Ejemplo)

```bash
# api_gateway/.env.example
# NUNCA commitear valores reales — usar HashiCorp Vault en producción

# API
SECRET_KEY=changeme-use-vault-in-production
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# Database (rotadas automáticamente por Vault en prod)
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/securevault

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@localhost:5672/

# Vault
VAULT_ADDR=http://localhost:8200
VAULT_TOKEN=dev-token   # Solo desarrollo; usar AppRole en prod

# Telemetría
OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
OTEL_SERVICE_NAME=api-gateway
```

### Despliegue en Kubernetes

```bash
# 1. Configurar contexto K8s
kubectl config use-context securevault-prod

# 2. Crear namespaces
kubectl apply -f infra/kubernetes/base/namespaces.yaml

# 3. Instalar Vault (OpenBao)
helm repo add openbao https://openbao.github.io/openbao-helm
helm install vault openbao/openbao \
  -n securevault-vault \
  -f infra/kubernetes/helm/vault/values.yaml

# 4. Inicializar y unseal Vault
# (ver docs/operations/runbooks/vault-unseal.md)

# 5. Desplegar PostgreSQL con Helm
helm install postgresql \
  oci://registry-1.docker.io/bitnamicharts/postgresql \
  -n securevault-data \
  -f infra/kubernetes/helm/postgresql/values.yaml

# 6. Desplegar API Gateway
helm install api-gateway \
  ./infra/kubernetes/helm/api-gateway \
  -n securevault-api \
  -f infra/kubernetes/helm/api-gateway/values-production.yaml

# 7. FluxCD gestiona el resto automáticamente (GitOps)
flux bootstrap github \
  --owner=tu-org \
  --repository=securevault-gitops \
  --path=clusters/production
```

---

## 🔒 Seguridad

### Modelo de Amenazas (Resumen)

| Amenaza | Mitigación |
|---|---|
| Robo de credenciales | Vault + rotación automática, sin secrets en código |
| Inyección SQL | SQLAlchemy ORM + Pydantic validación estricta |
| JWT tampering | Firma RS256, expiración corta, refresh rotation |
| Container escape | Rootless, seccomp, AppArmor, no-new-privileges |
| CVEs en dependencias | Trivy + Renovate, actualizaciones automáticas |
| Movimiento lateral | Network policies Cilium, namespaces K8s |
| Secrets en logs | Falco rules + log scrubbing |
| Supply chain attacks | Cosign firma + Sigstore transparency log |
| DDoS | Rate limiting slowapi + Nginx limit_req |
| Datos en reposo | pgcrypto AES-256 + Vault transit encryption |

### Hardening de Contenedores

```dockerfile
# Ejemplo: Dockerfile seguro para API Gateway
# Etapa 1: Build
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements/prod.txt .
RUN pip install --no-cache-dir --user -r prod.txt

# Etapa 2: Runtime (imagen mínima)
FROM python:3.12-slim AS runtime

# No ejecutar como root
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --no-create-home appuser

WORKDIR /app

# Copiar sólo las dependencias compiladas
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appgroup ./app /app

USER appuser

# Security headers
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/home/appuser/.local/bin:$PATH"

EXPOSE 8000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Política de Vault (Ejemplo)

```hcl
# infra/vault-policies/api-gateway-policy.hcl
# Permisos mínimos necesarios para el API Gateway

# Leer secretos de la aplicación
path "secret/data/securevault/api-gateway/*" {
  capabilities = ["read"]
}

# Acceso al motor de base de datos (credenciales temporales)
path "database/creds/api-gateway-role" {
  capabilities = ["read"]
}

# Cifrado/descifrado via Transit engine (nunca la clave raw)
path "transit/encrypt/api-gateway-key" {
  capabilities = ["update"]
}

path "transit/decrypt/api-gateway-key" {
  capabilities = ["update"]
}

# Renovar el propio token
path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

---

## 📊 Monitoreo

### Alertas Configuradas

```yaml
# monitoring/prometheus/rules/api-alerts.yml
groups:
  - name: api_gateway_alerts
    rules:
      # Alta tasa de errores 5xx
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])
          /
          rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Alta tasa de errores en API Gateway (>5%)"

      # Latencia P99 alta
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latencia P99 superior a 2 segundos"

      # Cola de RabbitMQ saturada
      - alert: RabbitMQQueueDepth
        expr: rabbitmq_queue_messages > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cola RabbitMQ con más de 10k mensajes pendientes"
```

### Dashboards Grafana

Los dashboards están pre-configurados en `monitoring/grafana/dashboards/` e incluyen:

- **API Gateway**: Tasa de requests, latencia (P50/P95/P99), errores por endpoint, tasa JWT
- **Workers**: Tareas procesadas/fallidas por worker, tiempo de procesamiento, profundidad de cola
- **PostgreSQL**: Conexiones activas, query latency, locks, replication lag
- **RabbitMQ**: Mensajes publicados/consumidos, profundidad de colas, consumers activos
- **Seguridad**: Eventos Falco, intentos de autenticación fallidos, scans detectados

---

## 🤝 Contribuir

Lee [CONTRIBUTING.md](CONTRIBUTING.md) para el proceso completo. Resumen:

1. **Fork** el repositorio
2. Crea una rama: `git checkout -b feat/mi-feature`
3. Instala hooks: `pre-commit install`
4. Realiza tus cambios con tests
5. Verifica el pipeline localmente:
   ```bash
   # Ejecutar checks de seguridad localmente
   pre-commit run --all-files
   pytest --cov=. --cov-report=term-missing
   bandit -r . -ll
   ```
6. Abre un **Pull Request** con descripción detallada
7. El pipeline CI/CD revisará automáticamente tu código

### Convención de Commits

Este proyecto usa [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: nueva funcionalidad
fix: corrección de bug
docs: cambios en documentación
sec: corrección de seguridad (prioridad alta)
chore: tareas de mantenimiento
ci: cambios en pipeline CI/CD
```

---

## 📄 Licencias

### Licencia del Proyecto

```
SecureVault — Plataforma OSINT y Análisis de Microservicios
Copyright (C) 2024 Contribuidores de SecureVault

Este programa es software libre: puedes redistribuirlo y/o modificarlo
bajo los términos de la Licencia Pública General Affero de GNU publicada
por la Free Software Foundation, versión 3 de la Licencia.

Este programa se distribuye con la esperanza de que sea útil,
pero SIN NINGUNA GARANTÍA; ni siquiera la garantía implícita de
COMERCIABILIDAD o IDONEIDAD PARA UN PROPÓSITO PARTICULAR.

Ver: https://www.gnu.org/licenses/agpl-3.0.html
```

### Resumen de Licencias de Dependencias

| Categoría | Licencias usadas | Compatibilidad AGPLv3 |
|---|---|---|
| Frontend (React, Vite, etc.) | MIT | ✅ Compatible |
| Backend (FastAPI, SQLAlchemy) | MIT, BSD-3 | ✅ Compatible |
| NLP (NLTK, spaCy, TextBlob) | Apache 2.0, MIT | ✅ Compatible |
| OSINT (TheHarvester, Recon-ng) | GPL v2/v3 | ✅ Compatible |
| Infraestructura (K8s, Helm, Flux) | Apache 2.0 | ✅ Compatible |
| Monitoreo (Prometheus, Grafana) | Apache 2.0, AGPL v3 | ✅ Compatible |
| RabbitMQ | MPL 2.0 | ✅ Compatible |
| PostgreSQL | PostgreSQL License | ✅ Compatible |
| OpenBao (Vault) | MPL 2.0 | ✅ Compatible |
| OpenTofu (Terraform) | MPL 2.0 | ✅ Compatible |

> ⚠️ **Nota**: HashiCorp Vault >= 1.14 y Terraform >= 1.6 usan **BSL 1.1**, que **no es aprobada por OSI** como licencia de código abierto. Para un proyecto 100% FOSS, usar **OpenBao** y **OpenTofu** respectivamente.

---

## 📞 Soporte y Contacto

- **Issues**: [GitHub Issues](https://github.com/tu-org/securevault/issues)
- **Seguridad**: Ver [SECURITY.md](SECURITY.md) para divulgación responsable
- **Discusión**: [GitHub Discussions](https://github.com/tu-org/securevault/discussions)
- **Matrix/IRC**: `#securevault:matrix.org`

---

<div align="center">

**Hecho con ❤️ por la comunidad FOSS**  
GNU AGPLv3 — La libertad del software es la libertad del usuario

</div>
