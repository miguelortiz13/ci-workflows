# ci-workflows

Workflows reutilizables de GitHub Actions para todos mis proyectos: el
equivalente en CI/CD de [terraform-modules-iac](https://github.com/miguelortiz13/terraform-modules-iac).
Un proyecto los llama con unas pocas líneas y recibe los mismos controles de
calidad, seguridad y despliegue que el resto.

## Catálogo

| Workflow | Qué hace |
|---|---|
| [`container-image.yml`](.github/workflows/container-image.yml) | Build de la imagen, escaneo con **Trivy** y, al publicar: push a **GHCR** (`<sha>` y `latest`), **SBOM** (SPDX), firma keyless con **Cosign** y verificación. Devuelve `image`, `tag` y `ref` (digest). |
| [`deploy-container-app.yml`](.github/workflows/deploy-container-app.yml) | Despliega la imagen en **Azure Container Apps** con OIDC (sin secretos) y espera a que la ruta de salud responda. |
| [`deploy-static-web-app.yml`](.github/workflows/deploy-static-web-app.yml) | Compila un frontend Node y lo publica en **Azure Static Web Apps**; pide el token de despliegue en el momento y verifica rutas. |
| [`security-baseline.yml`](.github/workflows/security-baseline.yml) | **Gitleaks** sobre el historial y, en PRs de repos públicos, **dependency review** de dependencias vulnerables. |

La infraestructura (plan y apply de Terraform) usa
[`terraform-azure.yml`](https://github.com/miguelortiz13/terraform-modules-iac/blob/main/.github/workflows/terraform-azure.yml),
en el repo de módulos.

## Uso

Fija siempre una versión (`@vX.Y.Z`). Ejemplo completo de una API:

```yaml
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  security:
    uses: miguelortiz13/ci-workflows/.github/workflows/security-baseline.yml@v0.1.0
    permissions: { contents: read, pull-requests: write }

  image:
    uses: miguelortiz13/ci-workflows/.github/workflows/container-image.yml@v0.1.0
    permissions: { contents: read, packages: write, id-token: write }
    with:
      image-name: mi-api
      push: ${{ github.event_name == 'push' }}

  deploy:
    needs: image
    if: github.event_name == 'push'
    uses: miguelortiz13/ci-workflows/.github/workflows/deploy-container-app.yml@v0.1.0
    permissions: { contents: read, id-token: write }
    with:
      app-name: ${{ vars.API_APP }}
      resource-group: ${{ vars.RESOURCE_GROUP }}
      image: ${{ needs.image.outputs.image }}:${{ needs.image.outputs.tag }}
      health-path: /health
```

## Requisitos en el proyecto

Los deja listos `platform-governance/scripts/configurar-github.sh`:

- Variables de repo `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` y `AZURE_SUBSCRIPTION_ID`
  (identidad de despliegue del proyecto, OIDC).
- Environment `prod` con aprobación y su credencial federada.
- Para `security-baseline.yml`: **Dependency graph** activo en el repo. Se activa
  junto con las alertas de Dependabot: `gh api -X PUT repos/<dueño>/<repo>/vulnerability-alerts`.
  Sin él, el job `dependency-review` falla con *Dependency review is not supported*.

## Estándares

- **Sin secretos de larga vida:** Azure con OIDC, GHCR con el `GITHUB_TOKEN` y
  firma keyless.
- **Imágenes inmutables:** se despliega el tag `<sha>` o el digest, nunca `latest`.
- **Permisos mínimos:** cada job declara solo lo que usa y el que llama los concede explícitamente.
- **Versiones fijadas** de las acciones de terceros; Dependabot las actualiza.
- **Calidad:** `actionlint` (con shellcheck) y una prueba real de
  `container-image.yml` en cada PR.

## Versionado

[Conventional Commits](https://www.conventionalcommits.org/) y release-please:
`feat:` sube la versión menor, `fix:` el parche y `feat!:` la mayor (cambio de
inputs u outputs).

## Licencia

[MIT](LICENSE)
