# Pipeline de Seguridad Veracode para GitHub Actions

Estrategia de seguridad automatizada que integra multiples productos Veracode en el SDLC, balanceando velocidad de feedback con profundidad de analisis segun el contexto de desarrollo.

**Tecnologias Soportadas**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, y mas.

---

## Estrategia de Escaneo

| Contexto | Producto Veracode | Tiempo | Gate | Proposito |
|----------|-------------------|--------|------|-----------|
| Feature branches | Pipeline Scan | 3-10 min | No | Feedback rapido para desarrolladores |
| Pull Requests | Pipeline Scan | 3-10 min | Si (Very High/High) | Prevenir vulnerabilidades antes de merge |
| Release branches | Sandbox Scan | 30-90 min | No | Validacion pre-produccion aislada |
| Branch main | Policy Scan | 25-60 min | Opcional | Certificacion para produccion |

**Todos los contextos**: SCA Agent-Based (analisis de dependencias)

---

## Estructura del Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    on: push / pull_request                       │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  JOB PACKAGE (siempre se ejecuta)                                │
│  ├─ Checkout de codigo                                           │
│  ├─ Configurar APP_NAME desde github.repository                  │
│  ├─ Instalar Veracode CLI                                        │
│  ├─ Ejecutar autopackager                                        │
│  ├─ Listar y validar artefactos → artifact_list.txt              │
│  └─ Subir verascan/ como artefacto del workflow                  │
│                                                                  │
│  Outputs: artifact_count, app_name                               │
└─────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
┌───────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
│  JOB SCA          │ │  JOBS PIPELINE  │ │  JOBS PLATAFORMA    │
│  (todos branches) │ │  SCAN           │ │                     │
│                   │ │                 │ │                     │
│  Analisis de      │ │  feature-*:     │ │  release/*:         │
│  dependencias     │ │   Sin gate      │ │   Sandbox Scan      │
│  basado en agente │ │                 │ │                     │
│                   │ │  PR:            │ │  main:              │
│                   │ │   Gate politica │ │   Policy Scan       │
└───────────────────┘ └─────────────────┘ └─────────────────────┘
```

---

## Secrets Requeridos

Configurar en: **Settings > Secrets and variables > Actions**

| Secret | Requerido | Descripcion |
|--------|-----------|-------------|
| `VERACODE_API_ID` | Si | ID de API Veracode para autenticacion |
| `VERACODE_API_KEY` | Si | Key de API Veracode para autenticacion |
| `SRCCLR_API_TOKEN` | No | Token del agente SCA (solo si usas SCA) |
| `VERACODE_APP_NAME` | No | Sobreescribir nombre de perfil de app por defecto |

### Como Obtener Credenciales

1. Inicia sesion en [Plataforma Veracode](https://analysiscenter.veracode.com)
2. Click en tu perfil (esquina superior derecha) > **API Credentials**
3. Genera o copia tu API ID y Key
4. Para SCA: **Workspace > Agents > Generate Token**

---

## Descripcion de Jobs

### 1. Job Package

**Se ejecuta en**: Todos los triggers (push y pull_request)

**Proposito**: Preparar artefactos para escaneo usando Veracode CLI autopackager

| Paso | Descripcion |
|------|-------------|
| Checkout | Clonar codigo del repositorio |
| Configurar App Name | Construir nombre de perfil desde `github.repository` o usar secret `VERACODE_APP_NAME` |
| Instalar Veracode CLI | Descargar e instalar CLI desde tools.veracode.com |
| Ejecutar Autopackager | Ejecutar `veracode package --source . --output verascan --trust` |
| Listar Artefactos | Encontrar todos los archivos `.war`, `.jar`, `.zip`, crear `artifact_list.txt` |
| Validar | Fallar si no se encuentran artefactos |
| Subir | Hacer `verascan/` disponible para jobs posteriores |

**Outputs**:
- `artifact_count`: Numero de artefactos encontrados
- `app_name`: Nombre del perfil de aplicacion Veracode

---

### 2. Job SCA

**Se ejecuta en**: Todos los triggers (en paralelo con otros jobs)

**Proposito**: Software Composition Analysis para vulnerabilidades en dependencias third-party

| Paso | Descripcion |
|------|-------------|
| Checkout | Clonar codigo del repositorio |
| Ejecutar SCA | Ejecutar escaneo basado en agente con `--recursive --update-advisor` |

**Por que siempre ejecutar SCA**: 80% de las vulnerabilidades vienen de dependencias. SCA complementa SAST analizando codigo third-party.

---

### 3. Feature Pipeline Scan

**Se ejecuta en**: Push a branches `feature/*`

**Proposito**: Feedback rapido durante desarrollo activo (sin gate)

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` del job package |
| Descargar Scanner | Obtener `pipeline-scan.jar` de downloads.veracode.com |
| Escanear Cada Artefacto | Iterar `artifact_list.txt`, escanear individualmente |
| Guardar Resultados | `scan_results/${ARTIFACT_NAME}_results.json` |
| Subir Resultados | Publicar como artefacto del workflow |

**Gate**: Ninguno (`--fail_on_severity ""`)

**Por que sin gate**: Los desarrolladores necesitan feedback rapido y no bloqueante durante desarrollo activo. Esto permite iteracion rapida sobre codigo seguro.

---

### 4. PR Pipeline Scan

**Se ejecuta en**: Pull requests (excluyendo forks)

**Proposito**: Gate de seguridad para pull requests

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` del job package |
| Descargar Scanner | Obtener `pipeline-scan.jar` |
| Escanear con Gate | Usar `--policy_name "Veracode Recommended Very High"` |
| Rastrear Fallos | Recolectar artefactos fallidos, reportar al final |
| Guardar Resultados | `scan_results/${ARTIFACT_NAME}_results.json` |
| Codigo de Salida | Distinto de cero si algun artefacto falla la politica |

**Gate**: Falla en hallazgos de severidad Very High y High

**Por que gate en PRs**: Previene que vulnerabilidades entren al codigo principal. Integra seguridad en el code review.

---

### 5. Release Sandbox Scan

**Se ejecuta en**: Push a branches `release/*`

**Proposito**: Analisis SAST completo en ambiente sandbox aislado

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` del job package |
| Listar Contenido | Mostrar artefactos siendo subidos |
| Descargar API Wrapper | Obtener ultimo `vosp-api-wrappers-java` de Maven Central |
| Subir a Sandbox | `-filepath "verascan"` (upload de carpeta, no necesita zip) |

**Nombre de Sandbox**: `github-release` (configurable)

**Por que sandbox para releases**: Validacion completa sin afectar metricas de produccion. Permite experimentacion con nuevas features de forma segura.

---

### 6. Main Policy Scan

**Se ejecuta en**: Push a branch `main`

**Proposito**: Certificacion de produccion para compliance

| Paso | Descripcion |
|------|-------------|
| Descargar Artefactos | Obtener `verascan/` del job package |
| Listar Contenido | Mostrar artefactos siendo subidos |
| Descargar API Wrapper | Obtener ultima version dinamicamente |
| Subir a Plataforma | `-filepath "verascan"` para evaluacion de politica |

**Por que policy scan en main**: Certificacion de seguridad oficial para codigo en produccion. Proporciona trazabilidad para regulaciones (SOC2, PCI-DSS, etc.).

---

## Nombres de Perfil de Aplicacion

El workflow construye automaticamente el nombre del perfil de aplicacion Veracode:

**Formato por defecto**: `{organizacion}/{repositorio}`

| Repositorio GitHub | Nombre de App Veracode |
|--------------------|------------------------|
| `acme-corp/api-service` | `acme-corp/api-service` |
| `myorg/frontend` | `myorg/frontend` |
| `company/backend-api` | `company/backend-api` |

**Sobreescribir**: Configura el secret `VERACODE_APP_NAME` para usar un nombre personalizado.

---

## Manejo Multi-Artefacto

El workflow maneja repositorios con multiples artefactos desplegables:

### Pipeline Scans (feature/PR)
Cada artefacto se escanea individualmente:
```
verascan/
  ├── backend-api.jar      → escaneado por separado
  ├── frontend.zip         → escaneado por separado  
  └── common-lib.jar       → escaneado por separado

scan_results/
  ├── backend-api.jar_results.json
  ├── frontend.zip_results.json
  └── common-lib.jar_results.json
```

### Platform Scans (release/main)
Todos los artefactos subidos juntos via carpeta:
```
-filepath "verascan"    # API Wrapper maneja upload multi-archivo nativamente
```

**Nota**: No se crea bundle zip. El Java API Wrapper acepta una ruta de directorio directamente, evitando rechazos de zip bomb.

---

## Resultados de Escaneo

### Resultados de Pipeline Scan

Los resultados se guardan por artefacto en `scan_results/`:

```
scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

**Descargar**: Workflow run > seccion Artifacts > `pipeline-scan-results` o `pipeline-scan-gate-results`

**Estructura JSON**:
```json
{
  "findings": [...],
  "pipeline_scan": {...},
  "scan_status": "SUCCESS"
}
```

### Resultados de Platform Scan (Sandbox/Policy)

Consultar en Plataforma Veracode:
1. Iniciar sesion en [analysiscenter.veracode.com](https://analysiscenter.veracode.com)
2. Navegar a tu perfil de aplicacion
3. Ver hallazgos, estado de compliance y tendencias

---

## Personalizacion

### Cambiar Politica de Gate de PR

En el job `pr-pipeline-scan`:
```yaml
--policy_name "Tu Politica Personalizada"
```

### Cambiar Gate de Severidad

```yaml
--fail_on_severity "Very High"           # Solo falla en Very High
--fail_on_severity "Very High, High"     # Falla en Very High y High (default)
--fail_on_severity ""                    # Sin gate (solo informativo)
```

### Nombre de Sandbox Personalizado

En el job `release-sandbox-scan`:
```yaml
-sandboxname "tu-nombre-de-sandbox"
```

### Agregar Paso de Build

Si el autopackager no funciona para tu proyecto, agrega un paso de build:

```yaml
- name: Build Application
  run: |
    # Maven
    mvn -B clean package -DskipTests
    
    # Gradle
    ./gradlew build -x test
    
    # npm
    npm ci && npm run build
```

### Cambiar Branches de Trigger

Modifica la seccion `on`:
```yaml
on:
  push:
    branches:
      - main
      - develop
      - 'release/**'
      - 'feature/**'
```

---

## Parametros de Pipeline Scan

| Parametro | Descripcion | Usado En |
|-----------|-------------|----------|
| `-f` | Archivo a escanear | Todos los pipeline scans |
| `-vid` | ID de API Veracode | Todos los scans |
| `-vkey` | Key de API Veracode | Todos los scans |
| `--fail_on_severity` | Severidades que fallan el build | Feature (vacio), PR (Very High,High) |
| `--policy_name` | Politica contra la cual evaluar | Scans de PR |
| `--issue_details` | Incluir info detallada de hallazgos | Todos los scans |
| `-jo` | Solo salida JSON | Todos los scans |

---

## Parametros de API Wrapper

| Parametro | Descripcion | Usado En |
|-----------|-------------|----------|
| `-action` | UploadAndScan | Todos los platform scans |
| `-appname` | Nombre del perfil de aplicacion | Todos los platform scans |
| `-createprofile` | Crear app si no existe | Todos los platform scans |
| `-autoscan` | Iniciar escaneo automaticamente | Todos los platform scans |
| `-sandboxname` | Nombre del sandbox | Scans de release |
| `-createsandbox` | Crear sandbox si no existe | Scans de release |
| `-filepath` | Ruta a artefactos (archivo o carpeta) | Todos los platform scans |
| `-version` | Etiqueta de version del escaneo | Todos los platform scans |

---

## Solucion de Problemas

### No se encontraron artefactos

**Sintomas**: Job package falla con "No se encontraron artefactos empaquetados"

**Soluciones**:
1. Asegurar que el proyecto compila exitosamente antes del empaquetado
2. Verificar que artefactos compilados existen en ubicaciones esperadas
3. Revisar salida del autopackager de Veracode CLI por errores
4. Agregar paso de build explicito antes del autopackager

### Pipeline scan no produce resultados

**Sintomas**: `results.json` no se crea para un artefacto

**Causas**:
- Artefacto no escaneable (JARs de test, resource bundles)
- Artefacto no contiene codigo de aplicacion
- Tipo de modulo no soportado

**Soluciones**:
1. Revisar salida del Pipeline Scanner por advertencias
2. Verificar que el artefacto contiene codigo de aplicacion
3. Esto es normal para algunos artefactos; el workflow continua

### Gate de politica falla inesperadamente

**Sintomas**: Pipeline scan de PR falla sin problemas obvios

**Soluciones**:
1. Verificar que nombre de politica coincide exactamente (sensible a mayusculas)
2. Confirmar que la politica existe en tu cuenta Veracode
3. Revisar `results.json` por hallazgos especificos
4. Verificar umbrales de la politica

### PRs de forks fallan o se omiten

**Sintomas**: PRs de forks no se ejecutan o fallan con errores de autenticacion

**Causa**: PRs de forks no acceden a secrets del repositorio por defecto

**Soluciones**:
1. El workflow omite intencionalmente PRs de forks
2. Para open source, considera `pull_request_target` (con precaucion)
3. Requerir que contribuidores ejecuten scans localmente

### Upload de Sandbox/Policy falla

**Sintomas**: API Wrapper falla al subir

**Soluciones**:
1. Verificar que `VERACODE_API_ID` y `VERACODE_API_KEY` estan configurados
2. Verificar que credenciales tienen permisos de upload
3. Asegurar que nombre de perfil de app es valido (sin caracteres especiales)
4. Revisar salida del API Wrapper por errores

---

## Archivos

| Archivo | Idioma | Descripcion |
|---------|--------|-------------|
| `veracode-scans.yml` | Espanol | Workflow de GitHub Actions |
| `veracode-strategy.md` | Ingles | Documentacion en ingles |

---

## Inicio Rapido

1. Copia `veracode-scans.yml` a `.github/workflows/veracode.yml`
2. Agrega secrets en **Settings > Secrets and variables > Actions**:
   - `VERACODE_API_ID`
   - `VERACODE_API_KEY`
   - `SRCCLR_API_TOKEN` (opcional)
3. Haz push a un feature branch para disparar el primer escaneo
4. Revisa resultados en los artefactos del workflow

---

## Mejores Practicas

**Shift-Left**:
- Ejecutar Pipeline Scan en cada commit
- Habilitar gates en PRs para bloquear hallazgos High/Very High
- Educar al equipo sobre hallazgos comunes

**Compliance**:
- Policy Scan obligatorio antes de produccion
- Mantener historial de escaneos
- Documentar excepciones y mitigaciones

**Optimizacion**:
- Usar Pipeline Scan para iteracion rapida
- Reservar Policy Scan para releases oficiales
- Cachear dependencias para acelerar builds

**SCA**:
- Monitorear nuevos CVEs continuamente
- Actualizar dependencias regularmente
- Revisar licencias antes de adoptar librerias

---

## Recursos

- [Documentacion Veracode](https://docs.veracode.com)
- [Instalacion Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [Documentacion GitHub Actions](https://docs.github.com/en/actions)
