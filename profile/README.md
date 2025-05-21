# Plataforma de Gestión de Torneos TCG

## Índice

1. [Visión General](#visión-general)
2. [Topología de Repositorios](#topología-de-repositorios)
3. [Estructura Interna de Cada Repositorio](#estructura-interna-de-cada-repositorio)
4. [Repositorio `.github` – Workflows y Plantillas Reutilizables](#repositorio-github)
5. [Ramas y Reglas de Protección](#ramas-y-reglas-de-protección)
6. [Pipelines de CI/CD](#pipelines-de-cicd)
7. [Seguridad y Cumplimiento](#seguridad-y-cumplimiento)
8. [Infraestructura como Código](#infraestructura-como-código)
9. [Convenciones y Buenas Prácticas](#convenciones-y-buenas-prácticas)
10. [Checklist de Arranque](#checklist-de-arranque)

---

## Visión General

Esta guía describe cómo estructurar la **organización de GitHub** del proyecto de torneos TCG para maximizar la automatización de compilación, despliegue y seguridad. El enfoque sigue buenas prácticas de **microservicios**: cada servicio vive en su propio repositorio (poly‑repo) y comparte flujos comunes mediante *workflows* reutilizables.

---

## Topología de Repositorios

| Repositorio             | Propósito                   | Descripción breve                                                                     |
| ----------------------- | --------------------------- | ------------------------------------------------------------------------------------- |
| `platform-infra`        | Infraestructura como código | Terraform/Bicep para red, bases de datos, contenedores y runners self‑hosted.         |
| `service-users`         | Autenticación & perfiles    | API .NET 7, almacenamiento de usuarios y disponibilidad general.                      |
| `service-tournaments`   | Torneos & sponsors          | CRUD de torneos, fechas y mesas.                                                      |
| `service-inscriptions`  | Inscripciones & equipos     | Lógica de altas individuales, parejas y equipos; registro de extras.                  |
| `service-brackets`      | Emparejamientos             | Algoritmo semi‑aleatorio basado en disponibilidad y asignación de mesas.              |
| `service-notifications` | (Futuro) Notificaciones     | Emails / push para avisos de partidas.                                                |
| `service-payments`      | (Futuro) Pagos Stripe       | Gestión de sesiones y webhooks de Stripe.                                             |
| `web-frontend`          | SPA Angular                 | UI móvil‑first multilenguaje.                                                         |
| `shared-libs`           | Paquetes comunes            | Librerías utilitarias (.NET/NPM) compartidas.                                         |
| `.github`               | Workflows globales          | *Reusable workflows*, **CODEOWNERS**, plantillas de PR/Issue, políticas de seguridad. |

> **Alternativa Monorepo:** si el equipo es muy pequeño, se puede usar `platform-services` con `/services/<nombre>/` y *paths filters* en los workflows. Sin embargo, el enfoque poly‑repo simplifica permisos y despliegues paralelos.

---

## Estructura Interna de Cada Repositorio

```text
.
├─ .github/
│  ├─ workflows/
│  │  ├─ ci.yml        # build + test + Docker
│  │  └─ release.yml   # semantic‑release
│  ├─ PULL_REQUEST_TEMPLATE.md
│  └─ ISSUE_TEMPLATE/
├─ src/                # código fuente
├─ tests/              # pruebas unitarias/integración
└─ Dockerfile          # imagen para GHCR
```

### `ci.yml` mínimo

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: my-org/.github/.github/workflows/reusable-ci.yml@v1
    with:
      language: dotnet
      path: ./src
```

> Cada servicio solo necesita indicar lenguaje y *path*; el resto (caché, CodeQL, publish a GHCR) vive en el **workflow reutilizable** del repo `.github`.

---

## Repositorio `.github`

Contiene todo lo común:

```text
.github/
├─ workflows/
│  ├─ reusable-ci.yml        # build/test + Docker + artefactos
│  ├─ security.yml           # CodeQL, Gitleaks, dependabot‑auto‑merge
│  └─ add-to-project.yml     # añade PR/Issue a Projects v2
├─ CODEOWNERS
├─ SECURITY.md
├─ PULL_REQUEST_TEMPLATE.md
└─ ISSUE_TEMPLATE/
```

### Ejemplo: `reusable-ci.yml`

```yaml
name: Reusable CI
on:
  workflow_call:
    inputs:
      language:
        required: true
        type: string
      path:
        required: true
        type: string
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: |
            ~/.nuget/packages
          key: ${{ runner.os }}-${{ inputs.language }}-${{ hashFiles('**/*.csproj') }}
      - name: Build & Test
        run: |
          dotnet restore ${{ inputs.path }}
          dotnet test ${{ inputs.path }} --collect:"XPlat Code Coverage"
      - name: Build Docker
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
      - name: Push image
        run: |
          echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## Ramas y Reglas de Protección

* **Modelo trunk‑based** → rama `main` protegida.
* Requisitos antes de merge:

  * 1 review de **CODEOWNER**.
  * Status checks verdes: `ci`, `security`.
  * Cobertura mínima de test (por ejemplo 70 %).
* Solo *squash* merge (historial limpio) y **Merge Queue** opcional.

---

## Pipelines de CI/CD

### 1 – Build & Test (CI)

1. `actions/checkout@v4`.
2. Cache de dependencias.
3. Compilación + `dotnet test` / `npm test`.
4. **CodeQL** + **Gitleaks**.
5. Construcción de imagen Docker y push a **GHCR**.

### 2 – Deploy (CD)

*Job* independiente por entorno (`dev`, `staging`, `prod`) usando **GitHub Environments** para inyectar secretos y pedir aprobación manual en producción.

```yaml
deploy-prod:
  needs: build
  runs-on: ubuntu-latest
  environment:
    name: prod
    url: https://tcg.example.com
  steps:
    - uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    - name: Deploy Container App
      run: az containerapp up -n service-users-prod ...
```

---

## Seguridad y Cumplimiento

| Herramienta         | Integración                     | Objetivo                              |
| ------------------- | ------------------------------- | ------------------------------------- |
| **CodeQL**          | `security.yml` diario + on‑push | Análisis estático multi‑lenguaje.     |
| **Gitleaks**        | Paso en CI + cron               | Detección de *secrets*.               |
| **Dependabot**      | `dependabot.yml`                | PR automáticas ante vulnerabilidades. |
| **Secret Scanning** | Activado a nivel org.           | Bloquear commits con claves.          |

Además:

* Archivo `SECURITY.md` con política de divulgación.
* *Branch protection* + **CODEOWNERS** → refuerzo de revisiones.

---

## Infraestructura como Código

En `platform-infra`:

* **Terraform** (`/modules/aks`, `/modules/postgres`, `/modules/keyvault`, `/modules/gh-runners`).
* Workflow `iac-plan-apply.yml` con pasos `fmt`, `validate`, `plan` y `apply` (gated en prod).

```hcl
# ejemplo de módulo PostgreSQL
a resource "azurerm_postgresql_flexible_server" "db" {
  name                = "tcg-users-db"
  resource_group_name = var.rg
  location            = var.region
  sku_name            = "B_Standard_B1ms"
  storage_mb          = 32768
}
```

---

## Convenciones y Buenas Prácticas

* **Commits:** Conventional Commits (`feat:`, `fix:`, etc.).
* **semantic‑release:** genera tags `vX.Y.Z` y *CHANGELOG.md*.
* **Issue labels:** `type/bug`, `type/feature`, `prio/high`, `service/<name>`.
* **Teams:** `backend`, `frontend`, `devops`, `sec` → mapeados en **CODEOWNERS**.
* **Codespaces / Devcontainer:** opcional para onboarding en 1 click.

---

## Checklist de Arranque

1. Crear la organización **`tcg-platform`** y los equipos internos.
2. Añadir repos según la topología y proteger `main`.
3. Clonar `.github` con workflows y plantillas reutilizables.
4. Habilitar **Dependabot**, **CodeQL** y secret scanning a nivel organización.
5. Configurar secretos cloud (por ejemplo `AZURE_CREDENTIALS`) en *Environments*.
6. Probar pipeline CI en `service-users`.
7. Activar **GitHub Projects v2** y el workflow de auto‑triage `add-to-project.yml`.

> ¡Listo! Con esta configuración dispondrás de compilación, tests, análisis de seguridad, versión semántica, despliegue por entornos y trazabilidad centralizada con el mínimo esfuerzo operativo.
