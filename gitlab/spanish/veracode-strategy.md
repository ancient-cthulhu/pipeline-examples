# Pipeline de Seguridad Veracode para Bitbucket

Estrategia de seguridad automatizada que integra multiples productos de Veracode en el SDLC, balanceando velocidad de feedback con profundidad de analisis segun el contexto de desarrollo.

**Tecnologias soportadas**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, y mas.

> Version en ingles: [README.md](./README.md)

---

## Estrategia de Escaneo

| Contexto | Producto Veracode | Tiempo | Gate | Proposito |
|----------|-------------------|--------|------|-----------|
| Ramas `feature/*` | Pipeline Scan | 3-10 min | No | Feedback rapido para desarrolladores |
| Pull Requests | Pipeline Scan | 3-10 min | Si (Very High / High) | Prevenir vulnerabilidades antes del merge |
| Ramas `release/*` | Sandbox Scan | 30-90 min | No | Validacion aislada pre-produccion |
| Rama `main` | Policy Scan | 25-60 min | Opcional | Certificacion de produccion |

**Todos los contextos**: SCA basado en agente (analisis de dependencias) corre en paralelo.

---

## Estructura del Workflow

```text
+------------------------------------------------------------------+
|     Trigger: push a rama  /  pull_request                        |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
|  PASO PACKAGE (siempre se ejecuta)                               |
|   - Resolver APP_NAME (VERACODE_APP_NAME o BITBUCKET_REPO_FULL)  |
|   - Instalar Veracode CLI                                        |
|   - Ejecutar autoempaquetador                                    |
|   - Construir artifact_list.txt                                  |
|   - Validar que existan artefactos                               |
|   - Publicar: CLI veracode, verascan/, artifact_list.txt,        |
|               app_name.txt como artefactos de Bitbucket          |
+------------------------------------------------------------------+
                                |
                  +-------------+-------------+
                  |                           |
                  v                           v
       +------------------+        +-----------------------+
       |   PASO SCA       |        |   PASO DE ESCANEO     |
       |   (paralelo)     |        |   (segun rama)        |
       |                  |        |                       |
       |   Analisis de    |        |   feature/* -> Pipe   |
       |   dependencias   |        |   PR        -> PipeG  |
       |   por agente     |        |   release/* -> Sand   |
       |                  |        |   main      -> Policy |
       +------------------+        +-----------------------+
```

---

## Variables de Repositorio Requeridas

Configurar en **Repository settings > Pipelines > Repository variables**.

| Variable | Requerida | Asegurada | Descripcion |
|----------|-----------|-----------|-------------|
| `VERACODE_API_ID` | Si | No | Veracode API ID para autenticacion |
| `VERACODE_API_KEY` | Si | Si | Veracode API Key para autenticacion |
| `SRCCLR_API_TOKEN` | No | Si | Token de agente SCA (solo si se usa SCA) |
| `VERACODE_APP_NAME` | No | No | Sobreescribe el nombre del perfil de aplicacion |

### Como Obtener Credenciales

1. Ingresar a la [Plataforma Veracode](https://analysiscenter.veracode.com)
2. Click en tu perfil (esquina superior derecha) > **API Credentials**
3. Generar o copiar tu API ID y Key
4. Para SCA: **Workspace > Agents > Generate Token**

---

## Descripcion de Pasos

### 1. Paso Package

**Se ejecuta en**: Todos los triggers (cada rama y cada pull request)

**Proposito**: Preparar artefactos para escaneo usando el autoempaquetador del Veracode CLI.

| Sub-paso | Descripcion |
|----------|-------------|
| Resolver App Name | Usa `VERACODE_APP_NAME` si esta definida, si no `$BITBUCKET_REPO_FULL_NAME` |
| Instalar Veracode CLI | Descarga e instala el CLI desde `tools.veracode.com` |
| Ejecutar Autoempaquetador | Corre `veracode package --source . --output verascan --trust` |
| Listar Artefactos | Encuentra todos los `.war`, `.jar`, `.zip` en `artifact_list.txt` |
| Validar | Falla si no se producen artefactos |
| Publicar | Pasa `verascan/`, `artifact_list.txt`, `app_name.txt` y el binario CLI como artefactos de Bitbucket |

---

### 2. Paso SCA

**Se ejecuta en**: Todos los triggers (en paralelo con el paso de escaneo)

**Proposito**: Analisis de Composicion de Software para vulnerabilidades en dependencias de terceros.

**Por que SCA siempre**: Una gran parte de las vulnerabilidades vienen de dependencias. SCA complementa SAST analizando codigo de terceros, dependencias transitivas y riesgo de licencias.

---

### 3. Pipeline Scan en Feature

**Se ejecuta en**: Push a ramas `feature/*`

**Proposito**: Feedback rapido y no bloqueante durante desarrollo activo.

| Sub-paso | Descripcion |
|----------|-------------|
| Descargar artefactos | Recibir `verascan/` y `artifact_list.txt` del paso package |
| Descargar Scanner | Obtener `pipeline-scan.jar` desde `downloads.veracode.com` |
| Escanear cada artefacto | Iterar `artifact_list.txt`, escanear cada uno individualmente |
| Persistir resultados | Guardar cada escaneo como `scan_results/${ARTIFACT_NAME}_results.json` |

**Gate**: Ninguno (`--fail_on_severity ""`).

**Por que sin gate**: Los desarrolladores necesitan feedback rapido y no bloqueante durante el desarrollo activo.

---

### 4. Pipeline Scan en PR (con Gate)

**Se ejecuta en**: Pull requests hacia cualquier rama

**Proposito**: Bloquear merges con findings de severidad Very High y High.

| Sub-paso | Descripcion |
|----------|-------------|
| Descargar artefactos | Recibir `verascan/` y `artifact_list.txt` del paso package |
| Descargar Scanner | Obtener `pipeline-scan.jar` |
| Escanear con Gate | Usar `--policy_name "Veracode Recommended Very High"` y `--fail_on_severity "Very High, High"` |
| Rastrear fallas | Recolectar cada artefacto que falle el gate, reportar al final |
| Codigo de salida | Distinto de cero si algun artefacto falla el gate |

**Por que gate en PR**: Previene que vulnerabilidades entren a ramas protegidas e integra seguridad al code review.

---

### 5. Sandbox Scan en Release

**Se ejecuta en**: Push a ramas `release/*`

**Proposito**: Analisis SAST completo en un ambiente sandbox aislado.

| Sub-paso | Descripcion |
|----------|-------------|
| Descargar artefactos | Recibir `verascan/` y `app_name.txt` |
| Descargar API Wrapper | Obtener la ultima version de `vosp-api-wrappers-java` desde Maven Central |
| Subir | `-filepath "verascan"` (subida de directorio, no requiere zip) |

**Nombre del Sandbox**: `bitbucket-release` (configurable en el YAML).

**Por que sandbox para releases**: Validacion completa sin afectar las metricas de la politica de produccion. Permite experimentar de forma segura con nuevas features antes de promover a main.

---

### 6. Policy Scan en Main

**Se ejecuta en**: Push a `main`

**Proposito**: Certificacion de produccion para compliance.

| Sub-paso | Descripcion |
|----------|-------------|
| Descargar artefactos | Recibir `verascan/` y `app_name.txt` |
| Descargar API Wrapper | Obtener la ultima version dinamicamente |
| Subir | `-filepath "verascan"` para evaluacion de politica |

**Por que policy scan en main**: Certificacion oficial de seguridad del codigo en produccion. Provee trazabilidad para SOC2, PCI-DSS, ISO 27001 y regulaciones similares.

---

## Nombres de Perfil de Aplicacion

El pipeline construye automaticamente el nombre del perfil de aplicacion en Veracode.

**Formato por defecto**: `$BITBUCKET_REPO_FULL_NAME` (que es `{workspace}/{repo}`).

| Repositorio Bitbucket | Nombre de App en Veracode |
|------------------------|----------------------------|
| `acme-corp/api-service` | `acme-corp/api-service` |
| `myorg/frontend` | `myorg/frontend` |
| `company/backend-api` | `company/backend-api` |

**Sobreescribir**: Definir la variable de repositorio `VERACODE_APP_NAME` para usar un nombre custom.

---

## Manejo Multi-Artefacto

El pipeline maneja repositorios con multiples artefactos desplegables.

### Pipeline Scans (feature / PR)

Cada artefacto se escanea individualmente:

```text
verascan/
  backend-api.jar      -> escaneado por separado
  frontend.zip         -> escaneado por separado
  common-lib.jar       -> escaneado por separado

scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

### Platform Scans (release / main)

Todos los artefactos se suben juntos por directorio:

```text
-filepath "verascan"    # El API Wrapper maneja subida multi-archivo nativamente
```

**Nota**: No se crea un bundle zip. El Java API Wrapper acepta una ruta de directorio directamente, evitando rechazos por zip bomb.

---

## Resultados de Escaneo

### Resultados de Pipeline Scan

Guardados por artefacto en `scan_results/` y expuestos como artefactos de paso de Bitbucket.

**Descarga**: Pipeline run > paso > pestana **Artifacts**.

**Estructura JSON**:
```json
{
  "findings": [...],
  "pipeline_scan": {...},
  "scan_status": "SUCCESS"
}
```

### Resultados de Platform Scan (Sandbox / Policy)

Revisar en la Plataforma Veracode:
1. Ingresar a [analysiscenter.veracode.com](https://analysiscenter.veracode.com)
2. Navegar al perfil de tu aplicacion
3. Revisar findings, estado de compliance y tendencias

---

## Personalizacion

### Cambiar la Politica del Gate de PR

En el paso `&pipeline-scan-gate`:

```yaml
--policy_name "Tu Politica Custom"
```

### Cambiar el Gate de Severidad

```yaml
--fail_on_severity "Very High"             # Falla solo en Very High
--fail_on_severity "Very High, High"       # Falla en Very High y High (default)
--fail_on_severity ""                      # Sin gate (solo informativo)
```

### Nombre Custom de Sandbox

En el paso `&sandbox-scan`:

```yaml
-sandboxname "tu-nombre-de-sandbox"
```

### Agregar un Paso de Build

Si el autoempaquetador no funciona para tu proyecto, agregar un paso de build antes del paso package:

```yaml
- step:
    name: Build
    script:
      # Maven
      - mvn -B clean package -DskipTests
      # Gradle
      # - ./gradlew build -x test
      # npm
      # - npm ci && npm run build
    artifacts:
      - target/**
      - build/**
      - dist/**
```

### Cambiar Ramas Trigger

Modificar la seccion `pipelines`:

```yaml
pipelines:
  branches:
    main:
      - step: *package
    develop:
      - step: *package
    'release/*':
      - step: *package
    'feature/*':
      - step: *package
```

---

## Parametros del Pipeline Scan

| Parametro | Descripcion | Usado en |
|-----------|-------------|----------|
| `-f` | Archivo a escanear | Todos los pipeline scans |
| `-vid` | Veracode API ID | Todos los escaneos |
| `-vkey` | Veracode API Key | Todos los escaneos |
| `--fail_on_severity` | Severidades que fallan el build | Feature (vacio), PR (`Very High, High`) |
| `--policy_name` | Politica Veracode a evaluar | PR scans |
| `--issue_details` | Incluir info detallada de findings | Todos los escaneos |
| `-jo` | Solo salida JSON | Todos los escaneos |

> Nota: `--policy_name` es el parametro correcto (el alias deprecado `--policy` debe evitarse).

---

## Parametros del API Wrapper

| Parametro | Descripcion | Usado en |
|-----------|-------------|----------|
| `-action` | `UploadAndScan` | Todos los platform scans |
| `-appname` | Nombre del perfil de aplicacion | Todos los platform scans |
| `-createprofile` | Crear app si no existe | Todos los platform scans |
| `-autoscan` | Iniciar escaneo automaticamente | Todos los platform scans |
| `-sandboxname` | Nombre del sandbox | Release scans |
| `-createsandbox` | Crear sandbox si no existe | Release scans |
| `-filepath` | Ruta a artefactos (archivo o directorio) | Todos los platform scans |
| `-version` | Etiqueta de version del escaneo | Todos los platform scans |

---

## Troubleshooting

### No se Encuentran Artefactos

**Sintomas**: El paso package falla con "No packaged artifacts found".

**Soluciones**:
1. Asegurar que el proyecto compila exitosamente antes de empaquetar
2. Verificar que los artefactos compilados existan en las ubicaciones esperadas
3. Revisar la salida del autoempaquetador del Veracode CLI por errores
4. Agregar un paso de build explicito antes del autoempaquetador

### Pipeline Scan No Produce Resultados

**Sintomas**: No se crea `results.json` para un artefacto.

**Causas**:
- El artefacto no es escaneable (test JARs, resource bundles)
- El artefacto no contiene codigo de aplicacion
- Tipo de modulo no soportado

**Soluciones**:
1. Revisar la salida del Pipeline Scanner por warnings
2. Verificar que el artefacto contiene codigo de aplicacion
3. Esto es normal para algunos artefactos; el pipeline continua

### Policy Gate Falla Inesperadamente

**Sintomas**: El pipeline scan de PR falla sin issues obvios.

**Soluciones**:
1. Verificar que el nombre de politica coincide exactamente (sensible a mayusculas)
2. Confirmar que la politica existe en tu cuenta Veracode
3. Revisar `results.json` para findings especificos
4. Verificar los thresholds de la politica

### Falla la Subida a Sandbox o Policy

**Sintomas**: El API Wrapper falla durante la subida.

**Soluciones**:
1. Verificar que `VERACODE_API_ID` y `VERACODE_API_KEY` estan definidas como variables de repositorio
2. Verificar que las credenciales tienen permisos de subida
3. Asegurar que el nombre del perfil de aplicacion es valido (sin caracteres especiales)
4. Revisar la salida del API Wrapper por errores

---

## Archivos

| Archivo | Descripcion |
|---------|-------------|
| `veracode-scans.yml` | Configuracion de Bitbucket Pipelines |
| `veracode-strategy.md` | Documentacion |

---

## Inicio Rapido

1. Copiar `veracode-scans.yml` a la raiz de tu repositorio
2. Agregar variables de repositorio en **Repository settings > Pipelines > Repository variables**:
   - `VERACODE_API_ID`
   - `VERACODE_API_KEY` (marcar como asegurada)
   - `SRCCLR_API_TOKEN` (opcional, marcar como asegurada)
3. Habilitar Pipelines en **Repository settings > Pipelines > Settings**
4. Hacer push a una rama `feature/*` para disparar el primer escaneo
5. Revisar resultados en los artefactos del pipeline run y en la Plataforma Veracode

---

## Mejores Practicas

**Shift-Left**:
- Correr Pipeline Scan en cada commit
- Habilitar gates en PRs para bloquear findings High y Very High
- Educar al equipo sobre findings comunes

**Compliance**:
- Requerir Policy Scan antes de produccion
- Mantener historial de escaneos para auditoria
- Documentar excepciones y mitigaciones

**Optimizacion**:
- Usar Pipeline Scan para iteracion rapida
- Reservar Policy Scan para releases oficiales
- Cachear dependencias de Maven, Gradle, npm y pip para acelerar builds

**SCA**:
- Monitorear nuevas CVEs continuamente
- Actualizar dependencias regularmente
- Revisar licencias antes de adoptar nuevas librerias

---

## Recursos

- [Documentacion de Veracode](https://docs.veracode.com)
- [Instalacion del Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Comandos del Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Ejemplos de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_examples)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [Documentacion de Bitbucket Pipelines](https://support.atlassian.com/bitbucket-cloud/docs/get-started-with-bitbucket-pipelines/)
