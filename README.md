# github-workflows

Biblioteca de workflows reutilizables de GitHub Actions para todos mis
proyectos: build por lenguaje, imágenes, despliegue, seguridad y, cuando haga
falta, infraestructura y operación. Es el equivalente en pipelines de
[terraform-modules-iac](https://github.com/miguelortiz13/terraform-modules-iac):
cada proyecto arma su pipeline combinando piezas pequeñas y versionadas.

## Catálogo

| Categoría | Workflow | Qué hace | Permisos que pide |
|---|---|---|---|
| **build** | [`build-java-maven.yml`](.github/workflows/build-java-maven.yml) | Maven con caché, build y pruebas, reportes como artefacto | `contents: read` |
| **build** | [`build-python.yml`](.github/workflows/build-python.yml) | pip con caché, instala los `requirements` indicados y ejecuta lint y pruebas | `contents: read` |
| **build** | [`build-node.yml`](.github/workflows/build-node.yml) | npm con caché; ejecuta los scripts que existan (`lint test build` por defecto) y sube lo compilado | `contents: read` |
| **container** | [`container-build.yml`](.github/workflows/container-build.yml) | Build de la imagen, escaneo **Trivy** (umbral configurable con `fail-severity`) y push a **GHCR** (`<sha>` y `latest`) | `packages: write` |
| **container** | [`container-sign.yml`](.github/workflows/container-sign.yml) | **SBOM** (SPDX JSON), firma keyless con **Cosign**, atestación y verificación | `packages: write`, `id-token: write` |
| **deploy** | [`deploy-azure-container-app.yml`](.github/workflows/deploy-azure-container-app.yml) | Despliega en **Azure Container Apps** con OIDC, actualiza los **jobs** que usan la misma imagen y espera la ruta de salud | `id-token: write` |
| **deploy** | [`deploy-azure-static-web-app.yml`](.github/workflows/deploy-azure-static-web-app.yml) | Compila y publica en **Azure Static Web Apps** y verifica rutas | `id-token: write` |
| **security** | [`security-baseline.yml`](.github/workflows/security-baseline.yml) | **Gitleaks** y, en PRs de repos públicos, **dependency review** | `pull-requests: write` |

La infraestructura con Terraform usa por ahora
[`terraform-azure.yml`](https://github.com/miguelortiz13/terraform-modules-iac/blob/main/.github/workflows/terraform-azure.yml),
del repo de módulos.

### Convención de nombres

GitHub solo admite workflows reutilizables directamente en `.github/workflows/`
(sin subcarpetas), así que la categoría va en el nombre:

```
<categoría>-<herramienta o destino>.yml
```

| Categoría | Para qué | Ejemplos que se agregarían |
|---|---|---|
| `build-` | Compilar y probar por lenguaje | `build-go.yml`, `build-dotnet.yml` |
| `container-` | Imágenes y cadena de suministro | `container-scan-sarif.yml` |
| `deploy-` | Despliegue por proveedor y servicio | `deploy-aws-ecs.yml`, `deploy-k8s-gitops.yml` |
| `iac-` | Infraestructura como código | `iac-terraform-azure.yml`, `iac-terraform-aws.yml` |
| `security-` | Controles de seguridad | `security-codeql.yml`, `security-sast.yml` |
| `release-` | Versionado y publicación | `release-please.yml` |
| `ops-` | Operación programada | `ops-drift-detection.yml`, `ops-uptime.yml` |

Un workflow nuevo entra cuando un proyecto lo necesita. No se crean por adelantado.

## Cómo se arma un pipeline

Cada workflow hace **una cosa** y el proyecto los encadena con `needs`. Ejemplo
de una API Java desplegada en Azure:

```yaml
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  security:
    uses: miguelortiz13/github-workflows/.github/workflows/security-baseline.yml@v0.3.0
    permissions: { contents: read, pull-requests: write }

  build:
    uses: miguelortiz13/github-workflows/.github/workflows/build-java-maven.yml@v0.3.0
    with:
      java-version: "17"

  image:
    needs: build
    uses: miguelortiz13/github-workflows/.github/workflows/container-build.yml@v0.3.0
    permissions: { contents: read, packages: write }
    with:
      image-name: mi-api
      push: ${{ github.event_name == 'push' }}

  sign:
    needs: image
    if: needs.image.outputs.ref != ''
    uses: miguelortiz13/github-workflows/.github/workflows/container-sign.yml@v0.3.0
    permissions: { contents: read, packages: write, id-token: write }
    with:
      image-ref: ${{ needs.image.outputs.ref }}

  deploy:
    needs: [image, sign]
    uses: miguelortiz13/github-workflows/.github/workflows/deploy-azure-container-app.yml@v0.3.0
    permissions: { contents: read, id-token: write }
    with:
      app-name: ${{ vars.API_APP }}
      resource-group: ${{ vars.RESOURCE_GROUP }}
      image: ${{ needs.image.outputs.image }}:${{ needs.image.outputs.tag }}
      health-path: /health
```

Los pasos propios del proyecto (pruebas e2e, smoke tests, migraciones) se
quedan como jobs del proyecto, intercalados con `needs`.

### Decisiones de diseño

- **Un workflow por responsabilidad y permisos mínimos por job.** Solo
  `container-sign` pide `id-token: write`, el permiso más sensible: el build, el
  escaneo y el push no lo necesitan.
- **El escaneo va antes del push**, dentro de `container-build`: una imagen que
  no pasa el umbral no llega al registro y no hace falta pasar la imagen entre jobs.
- **Imágenes inmutables:** se despliega `<sha>` o el digest, nunca `latest`, y
  solo se despliega una imagen firmada (`needs: [image, sign]`).
- **Sin secretos de larga vida:** Azure con OIDC, GHCR con el `GITHUB_TOKEN` y
  firma keyless.

## Requisitos en el proyecto

Los deja listos `platform-governance/scripts/configurar-github.sh`:

- `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` y `AZURE_SUBSCRIPTION_ID` (identidad de
  despliegue del proyecto, OIDC) como **variables de repo** o como **secretos de
  repo** que el proyecto pasa explícitamente. En repos públicos conviene usar
  secretos: las variables aparecen en los logs. Los workflows de despliegue además
  ocultan las URLs.

  ```yaml
  deploy:
    uses: miguelortiz13/github-workflows/.github/workflows/deploy-azure-container-app.yml@v0.3.1
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ```

  > Los **secretos y variables del environment** (p. ej. `dev`) no sirven aquí:
  > el job que llama a un workflow reutilizable no puede declarar environment,
  > así que no los ve y no puede pasarlos.
- Environment `prod` con aprobación y su credencial federada.
- Para `security-baseline.yml`: **Dependency graph** activo en el repo. Se activa
  junto con las alertas de Dependabot: `gh api -X PUT repos/<dueño>/<repo>/vulnerability-alerts`.
  Sin él, el job `dependency-review` falla con *Dependency review is not supported*.
  Si no puedes activarlo (no eres admin del repo), llama al workflow con
  `dependency-review: false`: Gitleaks sigue corriendo.

## Verificar una imagen firmada

```bash
cosign verify ghcr.io/<dueño>/<imagen>@sha256:<digest> \
  --certificate-identity-regexp '^https://github.com/miguelortiz13/github-workflows/\.github/workflows/container-sign\.yml@' \
  --certificate-github-workflow-repository <dueño>/<repo> \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Límites de GitHub Actions a tener en cuenta

- Hasta **4 niveles** de workflows anidados y **20** workflows reutilizables
  distintos por archivo.
- Los inputs solo son `string`, `number` o `boolean`; las listas van como texto
  (una por línea o separadas por espacios).
- Los secretos se pasan explícitamente (`secrets:`) o con `secrets: inherit`.
- `vars` y los environments son los del repo que llama, pero el job que llama no
  ve los secretos ni las variables de un environment: lo que haya que pasar como
  input o secreto debe estar a nivel de repo.

## Calidad

- `actionlint` (con shellcheck) en cada PR.
- Pruebas reales de `container-build` y `build-node` contra los proyectos de
  `tests/`, sin publicar nada.
- Versiones fijadas de las acciones de terceros; Dependabot las actualiza.

## Versionado

[Conventional Commits](https://www.conventionalcommits.org/) y release-please:
`feat:` sube la versión menor, `fix:` el parche y `feat!:` la mayor (cambio de
inputs, outputs o nombres). Fija siempre `@vX.Y.Z` en los proyectos.

## Licencia

[MIT](LICENSE)
