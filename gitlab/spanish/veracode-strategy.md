# Pipeline de Seguridad Veracode para GitLab CI/CD

Estrategia de seguridad automatizada que integra multiples productos Veracode en el SDLC, equilibrando la velocidad de retroalimentacion con la profundidad del analisis segun el contexto de desarrollo.

**Tecnologias soportadas**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, y mas.

---

## Estrategia de Escaneo

| Contexto | Producto Veracode | Tiempo | Gate | Proposito |
|----------|-------------------|--------|------|-----------|
| Ramas feature (`feature/*`) | Pipeline Scan | 3-10 min | No | Retroalimentacion rapida para desarrolladores |
| Merge Requests | Pipeline Scan | 3-10 min | Si (Very High/High) | Prevenir vulnerabilidades antes del merge |
| Ramas release (`release/*`) | Sandbox Scan | 30-90 min | No | Validacion aislada pre-produccion |
| Rama `main` | Policy Scan | 25-60 min | Opcional | Certificacion de produccion |

**Todos los contextos**: SCA Basado en Agente (analisis de dependencias), corre en paralelo.

---

## Estructura del Workflow

```text
┌─────────────────────────────────────────────────────────────────┐
│           on: push (ramas) / merge_request_event                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE: package                                                 │
│  job package (siempre corre para ramas/MRs que coinciden)       │
│  - Checkout del codigo                                          │
│  - Configurar APP_NAME desde CI_PROJECT_PATH o sobrescribir     │
│  - Instalar Veracode CLI                                        │
│  - Ejecutar autopackager                                        │
│  - Listar y validar artefactos -> artifact_list.txt             │
│  - Publicar verascan/ como artefactos del job                   │
│  - Salidas: APP_NAME, ARTIFACT_COUNT (via reporte dotenv)       │
└─────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
┌──────────────────┐ ┌──────────────────┐ ┌─────────────────────┐
│  STAGE: sca      │ │  STAGE: scan     │ │  STAGE: scan        │
│  job sca         │ │  Pipeline Scans  │ │  Platform Scans     │
│  (todos los      │ │                  │ │                     │
│  triggers,       │ │  feature/*:      │ │  release/*:         │
│  paralelo)       │ │    Sin gate      │ │    Sandbox Scan     │
│                  │ │                  │ │                     │
│  Analisis de     │ │  MR:             │ │  main:              │
│  dependencias    │ │    Gate de       │ │    Policy Scan      │
│  basado en       │ │    politica      │ │                     │
│  agente          │ │                  │ │                     │
└──────────────────┘ └──────────────────┘ └─────────────────────┘
```

El bloque `workflow:rules` previene pipelines duplicados: cuando hay un MR abierto para una rama, solo corre el pipeline del MR.

---

## Variables CI/CD Requeridas

Configurar en: **Settings > CI/CD > Variables**

Para todos los secretos, habilitar **Masked** y **Protected** (cuando solo ramas protegidas como `main` y `release/*` deban acceder a ellos).

| Variable | Requerida | Descripcion |
|----------|-----------|-------------|
| `VERACODE_API_ID` | Si | ID de API de Veracode para autenticacion |
| `VERACODE_API_KEY` | Si | Clave de API de Veracode para autenticacion |
| `SRCCLR_API_TOKEN` | No | Token del agente SCA (solo si usas SCA) |
| `VERACODE_APP_NAME` | No | Sobrescribe el nombre por defecto del perfil de aplicacion |

### Como Obtener las Credenciales

1. Inicia sesion en la [Plataforma Veracode](https://analysiscenter.veracode.com)
2. Haz clic en tu perfil (esquina superior derecha) > **API Credentials**
3. Genera o copia tu API ID y Key
4. Para SCA: **Workspace > Agents > Generate Token**

### Notas sobre Proteccion de Variables

- `VERACODE_API_KEY` y `SRCCLR_API_TOKEN` deben estar **masked**.
- Si se marcan como **protected**, solo se inyectan en pipelines de ramas/tags protegidas. Asegurate de que `main` y `release/*` esten configuradas como refs protegidas (Settings > Repository > Protected branches), o desmarca **protected** para que las ramas feature y los MRs tambien puedan usarlas.
- Para infraestructura compartida, prefiere variables a **nivel de grupo** para evitar duplicacion entre proyectos.

---

## Descripcion de los Jobs

### 1. `package` (stage: `package`)

**Corre en**: Todos los triggers que coinciden (push a `main`, `release/*`, `feature/*`, y eventos de merge request)

**Proposito**: Preparar artefactos para escaneo usando el autopackager del Veracode CLI

| Paso | Descripcion |
|------|-------------|
| Configurar Nombre de App | Construir el nombre del perfil desde `CI_PROJECT_PATH` o usar la variable `VERACODE_APP_NAME` |
| Instalar Veracode CLI | Descargar e instalar el CLI desde `tools.veracode.com` |
| Ejecutar Autopackager | Correr `veracode package --source . --output verascan --trust` |
| Listar Artefactos | Encontrar todos los `.war`, `.jar`, `.zip` y crear `artifact_list.txt` |
| Validar | Falla si no se encuentran artefactos |
| Publicar | Hacer disponible `verascan/` como artefacto del job para los stages siguientes |

**Salidas (via `reports:dotenv`)**:
- `APP_NAME`: Nombre del perfil de aplicacion Veracode (consumido por los jobs de sandbox/policy)
- `ARTIFACT_COUNT`: Cantidad de artefactos encontrados

---

### 2. `sca` (stage: `sca`)

**Corre en**: Todos los triggers que coinciden, en paralelo con package (usa `needs: []`)

**Proposito**: Software Composition Analysis para vulnerabilidades en dependencias de terceros

| Paso | Descripcion |
|------|-------------|
| Ejecutar SCA | Correr un escaneo basado en agente con `--recursive --update-advisor` contra `sca-downloads.veracode.com/ci.sh` |

**Por que correr SCA siempre**: Una porcion grande de las vulnerabilidades de aplicacion provienen de dependencias. SCA complementa a SAST analizando codigo de terceros.

**Comportamiento si `SRCCLR_API_TOKEN` no esta configurado**: el job registra una advertencia y termina con codigo 0. El job tambien esta marcado como `allow_failure: true` para que SCA nunca bloquee el build.

---

### 3. `feature-pipeline-scan` (stage: `scan`)

**Corre en**: Pushes a ramas `feature/*` (solo cuando no hay un MR abierto para esa rama)

**Proposito**: Retroalimentacion rapida durante desarrollo activo (sin gate)

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` via `needs:artifacts: true` |
| Descargar Scanner | Obtener `pipeline-scan.jar` desde `downloads.veracode.com` |
| Escanear Cada Artefacto | Iterar sobre `artifact_list.txt` y escanear cada artefacto individualmente |
| Guardar Resultados | `scan_results/${ARTIFACT_NAME}_results.json` |
| Publicar | Artefactos del job retenidos por 1 mes |

**Gate**: Ninguno (`--fail_on_severity ""`)

---

### 4. `mr-pipeline-scan` (stage: `scan`)

**Corre en**: Eventos de merge request donde el proyecto origen es igual al proyecto destino (excluye forks)

**Proposito**: Gate de seguridad para merge requests

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` del job package |
| Descargar Scanner | Obtener `pipeline-scan.jar` |
| Escanear con Gate | Usar `--policy_name "Veracode Recommended Very High"` |
| Rastrear Fallas | Recolectar artefactos fallidos y reportarlos al final |
| Guardar Resultados | `scan_results/${ARTIFACT_NAME}_results.json` |
| Codigo de Salida | Distinto de cero si algun artefacto falla la politica |

**Gate**: Falla con hallazgos de severidad Very High y High.

**Manejo de forks**: La regla `$CI_MERGE_REQUEST_SOURCE_PROJECT_ID == $CI_MERGE_REQUEST_PROJECT_ID` omite este job para MRs originados desde forks. Los forks no tienen acceso a las variables enmascaradas del proyecto padre. Para correr escaneos con gate en MRs de fork, un mantenedor puede disparar el pipeline en el proyecto padre (pestania Pipelines del MR > Run pipeline). Ver [Run pipelines in fork merge requests](https://docs.gitlab.com/ci/pipelines/merge_request_pipelines/#run-pipelines-in-fork-merge-requests).

---

### 5. `release-sandbox-scan` (stage: `scan`)

**Corre en**: Pushes a ramas `release/*`

**Proposito**: Analisis SAST completo en un entorno sandbox aislado

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` del job package |
| Listar Contenido | Mostrar los artefactos a subir |
| Descargar API Wrapper | Obtener la version mas reciente de `vosp-api-wrappers-java` desde Maven Central |
| Subir a Sandbox | `-filepath "verascan"` (subida de directorio, sin necesidad de zip) |

**Nombre de Sandbox**: `gitlab-release` (configurable en el script del job).

**Etiqueta de version**: `Release ${CI_PIPELINE_IID}` (con alcance al proyecto, monotonicamente creciente).

---

### 6. `main-policy-scan` (stage: `scan`)

**Corre en**: Pushes a la rama `main`

**Proposito**: Certificacion de produccion para cumplimiento

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` del job package |
| Listar Contenido | Mostrar los artefactos a subir |
| Descargar API Wrapper | Obtener la version mas reciente dinamicamente |
| Subir a Plataforma | `-filepath "verascan"` para evaluacion de politica |

**Etiqueta de version**: `main ${CI_PIPELINE_IID}`.

---

## Nombres de Perfil de Aplicacion

El pipeline construye automaticamente el nombre del perfil de aplicacion Veracode a partir de variables predefinidas de GitLab.

**Formato por defecto**: `${CI_PROJECT_PATH}` que es `{namespace}/{project}`.

| Proyecto GitLab | Nombre de App Veracode |
|-----------------|------------------------|
| `acme-corp/api-service` | `acme-corp/api-service` |
| `myorg/frontend` | `myorg/frontend` |
| `company/group/backend-api` | `company/group/backend-api` |

**Sobrescribir**: Configurar la variable CI/CD `VERACODE_APP_NAME` para usar un nombre personalizado.

---

## Equivalencias GitHub Actions a GitLab CI

Para referencia al comparar ambos pipelines:

| GitHub Actions | GitLab CI/CD |
|----------------|--------------|
| `secrets.X` | `$X` (variable CI/CD) |
| `${{ github.repository }}` | `$CI_PROJECT_PATH` |
| `${{ github.run_number }}` | `$CI_PIPELINE_IID` |
| `${{ github.ref_name }}` | `$CI_COMMIT_BRANCH` |
| `github.event_name == 'pull_request'` | `$CI_PIPELINE_SOURCE == "merge_request_event"` |
| `pull_request.head.repo.fork == false` | `$CI_MERGE_REQUEST_SOURCE_PROJECT_ID == $CI_MERGE_REQUEST_PROJECT_ID` |
| `actions/upload-artifact` | palabra clave `artifacts:` |
| `actions/download-artifact` | `needs:artifacts: true` |
| `outputs` del job | `artifacts:reports:dotenv` |
| `if:` en un job | `rules:` |

---

## Manejo Multi-Artefacto

El pipeline maneja repositorios con multiples artefactos desplegables.

### Pipeline Scans (feature/MR)

Cada artefacto se escanea individualmente:

```text
verascan/
  ├── backend-api.jar      -> escaneado por separado
  ├── frontend.zip         -> escaneado por separado
  └── common-lib.jar       -> escaneado por separado

scan_results/
  ├── backend-api.jar_results.json
  ├── frontend.zip_results.json
  └── common-lib.jar_results.json
```

### Platform Scans (release/main)

Todos los artefactos se suben juntos via directorio:

```text
-filepath "verascan"    # El API Wrapper maneja subida multi-archivo nativamente
```

No se crea un bundle zip. El Java API Wrapper acepta una ruta de directorio directamente, evitando rechazos por zip bomb.

---

## Resultados de Escaneo

### Resultados de Pipeline Scan

Los resultados se guardan por artefacto en `scan_results/`. Descargar desde la pagina del job del pipeline:

**Pipeline > Job > Job artifacts > Browse / Download**

Nombres de los artefactos:
- `pipeline-scan-results-<sha>` para escaneos de feature
- `pipeline-scan-gate-results-<sha>` para escaneos de MR

Estructura JSON:
```json
{
  "findings": [...],
  "pipeline_scan": {...},
  "scan_status": "SUCCESS"
}
```

### Resultados de Platform Scan (Sandbox/Policy)

Revisar en la Plataforma Veracode:
1. Inicia sesion en [analysiscenter.veracode.com](https://analysiscenter.veracode.com)
2. Navega a tu perfil de aplicacion
3. Visualiza hallazgos, estado de cumplimiento, y tendencias

---

## Personalizacion

### Cambiar Politica del Gate de MR

En el job `mr-pipeline-scan`:

```yaml
--policy_name "Tu Politica Personalizada"
```

### Cambiar Gate de Severidad

```yaml
--fail_on_severity "Very High"           # Falla solo en Very High
--fail_on_severity "Very High, High"     # Falla en Very High y High
--fail_on_severity ""                    # Sin gate (solo informativo)
```

### Nombre de Sandbox Personalizado

En el job `release-sandbox-scan`:

```yaml
-sandboxname "tu-nombre-de-sandbox"
```

### Agregar un Paso de Build

Si el autopackager no funciona para tu proyecto, agrega un build explicito antes del empaquetado. Por ejemplo, modifica el `script` del job `package` para incluir:

```yaml
script:
  # Maven
  - mvn -B clean package -DskipTests
  # o Gradle
  - ./gradlew build -x test
  # o npm
  - npm ci && npm run build
  # luego continua con los pasos existentes del autopackager
```

Tambien puede ser necesario cambiar el `image:` por uno con el toolchain apropiado (por ejemplo `maven:3.9-eclipse-temurin-17`, `node:20`).

### Cambiar Ramas que Disparan el Pipeline

Modifica tanto el `workflow:` de nivel superior como las `rules:` de cada job. Por ejemplo, para incluir tambien `develop`:

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/
    - if: $CI_COMMIT_BRANCH =~ /^feature\/.*/
```

---

## Parametros de Pipeline Scan

| Parametro | Descripcion | Usado en |
|-----------|-------------|----------|
| `-f` | Archivo a escanear | Todos los pipeline scans |
| `-vid` | ID de API de Veracode | Todos los escaneos |
| `-vkey` | Clave de API de Veracode | Todos los escaneos |
| `--fail_on_severity` | Severidades que fallan el build | Feature (vacio), MR (via politica) |
| `--policy_name` | Politica contra la cual evaluar | Escaneos de MR |
| `--issue_details` | Incluir info detallada de hallazgos | Todos los escaneos |
| `-jo` | Solo salida JSON | Todos los escaneos |

---

## Parametros del API Wrapper

| Parametro | Descripcion | Usado en |
|-----------|-------------|----------|
| `-action` | UploadAndScan | Todos los platform scans |
| `-appname` | Nombre del perfil de aplicacion | Todos los platform scans |
| `-createprofile` | Crear app si no existe | Todos los platform scans |
| `-autoscan` | Iniciar el escaneo automaticamente | Todos los platform scans |
| `-sandboxname` | Nombre del sandbox | Escaneos de release |
| `-createsandbox` | Crear sandbox si no existe | Escaneos de release |
| `-filepath` | Ruta a los artefactos (archivo o directorio) | Todos los platform scans |
| `-version` | Etiqueta de version del escaneo | Todos los platform scans |

---

## Solucion de Problemas

### No se Encontraron Artefactos

**Sintomas**: El job `package` falla con "No se encontraron artefactos empaquetados"

**Soluciones**:
1. Asegurate que el proyecto compile correctamente antes de empaquetar
2. Verifica que los artefactos compilados existan en las ubicaciones esperadas
3. Revisa la salida del autopackager del Veracode CLI buscando errores
4. Agrega un paso de build explicito antes del autopackager (ver Personalizacion)

### Pipeline Scan No Produce Resultados

**Sintomas**: No se crea `results.json` para un artefacto

**Causas**:
- El artefacto no es escaneable (JARs de prueba, bundles de recursos)
- El artefacto no contiene codigo de aplicacion
- Tipo de modulo no soportado

**Soluciones**:
1. Revisa la salida del Pipeline Scanner buscando advertencias
2. Verifica que el artefacto contenga codigo de aplicacion
3. Esto es normal para algunos artefactos; el pipeline continua

### El Gate de Politica Falla Inesperadamente

**Sintomas**: El pipeline scan de MR falla sin razon obvia

**Soluciones**:
1. Verifica que el nombre de la politica coincida exactamente (sensible a mayusculas/minusculas)
2. Confirma que la politica existe en tu cuenta Veracode
3. Revisa `results.json` en los artefactos del job buscando hallazgos especificos
4. Verifica los umbrales de la politica

### Pipelines Duplicados en MRs

**Sintomas**: Dos pipelines corren para el mismo push cuando hay un MR abierto

**Causa**: `workflow:rules` no esta previniendo ambos pipelines (rama y MR)

**Solucion**: El `workflow:rules` provisto ya incluye `$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS when: never`. Si personalizas el workflow, mantenga esa regla cerca del inicio.

### Los MRs de Fork Son Omitidos

**Sintomas**: Los MRs desde forks no corren el escaneo con gate

**Causa**: Los forks no obtienen variables enmascaradas; la regla los excluye por diseno

**Soluciones**:
1. El pipeline omite los MRs de fork intencionalmente para proteger los secretos
2. Un mantenedor puede disparar el pipeline en el proyecto padre desde la pestania Pipelines del MR
3. Para open source, considera un job separado sin secretos para forks

### Falla la Subida a Sandbox/Policy

**Sintomas**: El API Wrapper falla durante la subida

**Soluciones**:
1. Verifica que `VERACODE_API_ID` y `VERACODE_API_KEY` esten configurados y no restringidos a ramas protegidas cuando sea necesario
2. Verifica que las credenciales tengan permisos de subida en la Plataforma Veracode
3. Asegurate que el nombre del perfil de aplicacion sea valido (sin caracteres especiales)
4. Revisa la salida del API Wrapper buscando errores

### Variables No Disponibles en Ramas Feature

**Sintomas**: Los jobs de feature/MR fallan con "VERACODE_API_ID not set"

**Causa**: Las variables marcadas como **Protected** solo se inyectan en pipelines de refs protegidas

**Soluciones**:
1. Desmarca **Protected** en las variables de credenciales
2. O protege `feature/*`, `main` y `release/*` (Settings > Repository > Protected branches)

---

## Archivos

| Archivo | Descripcion |
|---------|-------------|
| `.gitlab-ci.yml` | Pipeline GitLab CI/CD |
| `veracode-strategy.md` | Esta documentacion |

---

## Inicio Rapido

1. Copia `.gitlab-ci.yml` a la raiz del repo.
2. Agrega variables CI/CD (Settings > CI/CD > Variables):
   - `VERACODE_API_ID` (masked)
   - `VERACODE_API_KEY` (masked)
   - `SRCCLR_API_TOKEN` (masked, opcional)
3. Configura las ramas protegidas segun sea necesario (`main`, `release/*`).
4. Haz push a una rama `feature/*` para disparar el primer escaneo.
5. Revisa los resultados en los artefactos del job del pipeline y en la Plataforma Veracode.

---

## Mejores Practicas

**Shift-Left**:
- Correr Pipeline Scan en cada commit a ramas feature
- Habilitar gates en los MRs para bloquear hallazgos High/Very High
- Educar al equipo sobre hallazgos comunes

**Cumplimiento**:
- Requerir Policy Scan antes del despliegue a produccion
- Mantener el historial de escaneos en la Plataforma Veracode
- Documentar excepciones y mitigaciones

**Optimizacion**:
- Usar Pipeline Scan para iteracion rapida
- Reservar Policy Scan para releases oficiales
- Cachear dependencias de build (palabra clave `cache:`) para acelerar los builds

**SCA**:
- Monitorear continuamente nuevos CVEs
- Actualizar dependencias regularmente
- Revisar licencias antes de adoptar librerias

---

## Recursos

- [Documentacion Veracode](https://docs.veracode.com)
- [Instalacion del Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan)
- [Ejemplos oficiales de GitLab Pipeline Scan](https://docs.veracode.com/r/Gitlab_Pipeline_Scan_Examples)
- [Componente CI/CD de Veracode Pipeline Scan](https://gitlab.com/veracode/veracode-components/veracode-pipeline-scan)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Escaneos SCA Basados en Agente](https://docs.veracode.com/r/Agent_Based_Scans)
- [Referencia YAML de GitLab CI/CD](https://docs.gitlab.com/ci/yaml/)
- [Variables CI/CD Predefinidas de GitLab](https://docs.gitlab.com/ci/variables/predefined_variables/)
