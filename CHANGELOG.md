# Changelog

## [0.4.0](https://github.com/miguelortiz13/github-workflows/compare/v0.3.1...v0.4.0) (2026-10-24)


### Features

* **security-baseline:** dependency review opcional ([#17](https://github.com/miguelortiz13/github-workflows/issues/17)) ([328cf13](https://github.com/miguelortiz13/github-workflows/commit/328cf13c1cafdc6464eeff73489de2cb8676909f))

## [0.3.1](https://github.com/miguelortiz13/github-workflows/compare/v0.3.0...v0.3.1) (2026-10-24)


### Bug Fixes

* **deploy:** documentar que los identificadores de Azure van a nivel de repo ([#15](https://github.com/miguelortiz13/github-workflows/issues/15)) ([a4dea7a](https://github.com/miguelortiz13/github-workflows/commit/a4dea7a57bb67b91c902813e946dcd03bef05051))

## [0.3.0](https://github.com/miguelortiz13/github-workflows/compare/v0.2.0...v0.3.0) (2026-10-24)


### ⚠ BREAKING CHANGES

* container-build reemplaza fail-on-critical por fail-severity.

### Features

* build-python, umbral de Trivy configurable, jobs en Container Apps y credenciales como secretos ([#13](https://github.com/miguelortiz13/github-workflows/issues/13)) ([4c00497](https://github.com/miguelortiz13/github-workflows/commit/4c00497a84a480d8eb6d297027df0e04e88255b5))

## [0.2.0](https://github.com/miguelortiz13/github-workflows/compare/v0.1.2...v0.2.0) (2026-10-24)


### ⚠ BREAKING CHANGES

* container-image.yml, deploy-container-app.yml y deploy-static-web-app.yml cambian de nombre; la identidad de la firma pasa a container-sign.yml.

### Features

* biblioteca de workflows por categoría (build, container, deploy, security) ([#11](https://github.com/miguelortiz13/github-workflows/issues/11)) ([2d709e9](https://github.com/miguelortiz13/github-workflows/commit/2d709e9b76c7b2169fb188f40438f753cb270d8b))

## [0.1.2](https://github.com/miguelortiz13/ci-workflows/compare/v0.1.1...v0.1.2) (2026-10-24)


### Bug Fixes

* **container-image:** verificar el SBOM entre todas las atestaciones del digest ([#9](https://github.com/miguelortiz13/ci-workflows/issues/9)) ([4b98e55](https://github.com/miguelortiz13/ci-workflows/commit/4b98e554497b8e2affe98fe98ab5b119345264d3))
