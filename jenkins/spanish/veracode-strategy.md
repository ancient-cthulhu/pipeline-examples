# Pipeline de Seguridad Veracode para Jenkins

Estrategia de seguridad automatizada que integra multiples productos de Veracode en el SDLC, balanceando velocidad de feedback con profundidad de analisis segun el contexto de desarrollo.

**Tecnologias soportadas**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, y mas.

**Tipo de pipeline**: Declarative Pipeline, disenado para jobs de tipo **Multibranch Pipeline** (o equivalentes: GitHub Organization, Bitbucket Team, GitLab Group). Los directivos `when { branch ... }` y `when { changeRequest() }` requieren un contexto multibranch.

---

## Estrategia de Escaneo

| Contexto | Producto Veracode | Tiempo | Gate | Proposito |
|----------|-------------------|--------|------|-----------|
| Ramas `feature/*` | Pipeline Scan | 3-10 min | No | Feedback rapido para desarrolladores |
| Change Requests (PR/MR) | Pipeline Scan | 3-10 min | Si (Very High / High) | Prevenir vulnerabilidades antes del merge |
| Ramas `release/*` | Sandbox Scan | 30-90 min | No | Validacion aislada pre-produccion |
| Rama `main` | Policy Scan | 25-60 min | Opcional | Certificacion de produccion |

**En todos los contextos**: SCA basado en agente (analisis de dependencias) corre en paralelo.

---

## Estructura del Pipeline

```text
+------------------------------------------------------------------+
|         Trigger: push a rama / change request                    |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
|  ETAPA: Empaquetar Artefactos (siempre se ejecuta)               |
|   - Resolver APP_NAME (VERACODE_APP_NAME o JOB_NAME)             |
|   - Instalar Veracode CLI                                        |
|   - Ejecutar autoempaquetador: veracode package --source .       |
|   - Construir artifact_list.txt                                  |
|   - Validar que existan artefactos                               |
|   - Stash de verascan/, artifact_list.txt, app_name.txt          |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
|  ETAPA: Escaneos Veracode (paralelo)                             |
+------------------------------------------------------------------+
   |              |                |                |             |
   v              v                v                v             v
+-------+  +-----------+  +-----------+  +-----------+  +-----------+
| SCA   |  | Pipeline  |  | Pipeline  |  | Sandbox   |  | Policy    |
|       |  | (feature) |  | (PR Gate) |  | (release) |  | (main)    |
|siempre|  | feature/* |  | PR / MR   |  | release/* |  | main      |
+-------+  +-----------+  +-----------+  +-----------+  +-----------+
```

Cada etapa en paralelo esta condicionada por un `when {}`, de modo que solo una etapa de escaneo se ejecuta por build (ademas del SCA, que siempre corre).

---

## Credenciales Requeridas en Jenkins

Configurar en **Manage Jenkins > Credentials > System > Global credentials**.

| Credential ID | Tipo | Requerida | Descripcion |
|---------------|------|-----------|-------------|
| `veracode-api-id` | Secret text | Si | Veracode API ID |
| `veracode-api-key` | Secret text | Si | Veracode API Key |
| `srcclr-api-token` | Secret text | No | Token de agente SCA (solo si se usa SCA) |

> Si `srcclr-api-token` no esta configurada, la etapa SCA se omite limpiamente y el resto del pipeline continua.

### Variables de Entorno Opcionales

Definir a nivel folder, job o pipeline via **Configure > Environment variables** o el bloque `environment {}`.

| Variable | Descripcion |
|----------|-------------|
| `VERACODE_APP_NAME` | Sobreescribe el nombre del perfil de aplicacion (por defecto `JOB_NAME`) |

### Como Obtener Credenciales

1. Ingresar a la [Plataforma Veracode](https://analysiscenter.veracode.com)
2. Click en tu perfil (esquina superior derecha) > **API Credentials**
3. Generar o copiar tu API ID y Key
4. Para SCA: **Workspace > Agents > Generate Token**

---

## Plugins de Jenkins Requeridos

El pipeline asume que estan instalados los siguientes plugins en el controlador:

| Plugin | Proposito |
|--------|-----------|
| Pipeline (Declarative) | Sintaxis base del pipeline declarativo |
| Pipeline: Multibranch | Deteccion de ramas y change requests |
| Credentials Binding | `withCredentials` y `credentials()` |
| Workspace Cleanup | Step `cleanWs` en `post { always }` |
| Timestamper | Opcion `timestamps()` |
| AnsiColor | Opcion `ansiColor('xterm')` |

Todos los agentes deben tener `java` (JRE 8+), `curl` y `unzip` disponibles en el `PATH`.

---

## Descripcion de Etapas

### 1. Empaquetar Artefactos

**Se ejecuta en**: Todos los triggers (cada rama y cada change request)

**Proposito**: Preparar artefactos para escaneo usando el autoempaquetador del Veracode CLI.

| Sub-paso | Descripcion |
|----------|-------------|
| Resolver App Name | Usa `VERACODE_APP_NAME` si esta definida, si no `JOB_NAME` |
| Instalar Veracode CLI | Descarga desde `tools.veracode.com` |
| Ejecutar Autoempaquetador | `veracode package --source . --output verascan --trust` |
| Listar Artefactos | Encuentra todos los `.war`, `.jar`, `.zip` en `artifact_list.txt` |
| Validar | Falla el build si no se producen artefactos |
| Stash | Guarda `verascan/`, `artifact_list.txt`, `app_name.txt` para etapas posteriores |

---

### 2. SCA basado en Agente

**Se ejecuta en**: Todos los triggers, en paralelo

**Proposito**: Analisis de Composicion de Software para vulnerabilidades en dependencias de terceros.

| Sub-paso | Descripcion |
|----------|-------------|
| Verificar token | Si la credencial `srcclr-api-token` no existe, la etapa se omite |
| Ejecutar SCA | `curl ... | bash -s -- scan --recursive --update-advisor` |

**Por que correr siempre SCA**: Aproximadamente 80% de las vulnerabilidades provienen de dependencias. SCA complementa al SAST analizando codigo de terceros en cada build.

---

### 3. Pipeline Scan (feature)

**Se ejecuta en**: Pushes a ramas `feature/*` (excluyendo change requests)

**Proposito**: Feedback rapido durante desarrollo activo. Sin gate.

| Sub-paso | Descripcion |
|----------|-------------|
| Unstash | Recupera `verascan/` y `artifact_list.txt` |
| Descargar Scanner | Obtiene `pipeline-scan.jar` de `downloads.veracode.com` |
| Escanear cada artefacto | Itera sobre `artifact_list.txt` y escanea cada uno |
| Guardar resultados | `scan_results/${ARTIFACT_NAME}_results.json` |
| Archivar | Guarda resultados como artefactos del build de Jenkins |

**Gate**: Ninguno (`--fail_on_severity ""`)

**Por que sin gate**: Los desarrolladores necesitan feedback rapido y no-bloqueante durante el desarrollo activo. Esto permite iteracion rapida sobre codigo seguro.

---

### 4. Pipeline Scan (PR Gate)

**Se ejecuta en**: Change requests (`changeRequest()` matchea PR/MR de GitHub, Bitbucket, GitLab, etc.)

**Proposito**: Gate de seguridad en pull requests.

| Sub-paso | Descripcion |
|----------|-------------|
| Unstash | Recupera `verascan/` y `artifact_list.txt` |
| Descargar Scanner | Obtiene `pipeline-scan.jar` |
| Escanear con Gate | `--policy_name "Veracode Recommended Very High"` |
| Trackear fallas | Colecta artefactos que fallan; reporta al final |
| Guardar resultados | `scan_results/${ARTIFACT_NAME}_results.json` |
| Codigo de salida | Diferente de cero si algun artefacto falla la policy |

**Gate**: Falla en findings Very High y High

**Por que gate en PRs**: Previene la entrada de vulnerabilidades al codigo principal. Integra seguridad al code review.

---

### 5. Sandbox Scan (release)

**Se ejecuta en**: Pushes a ramas `release/*`

**Proposito**: Analisis SAST completo en un sandbox aislado.

| Sub-paso | Descripcion |
|----------|-------------|
| Unstash | Recupera `verascan/` y `app_name.txt` |
| Descargar API Wrapper | Obtiene la version mas reciente de `vosp-api-wrappers-java` desde Maven Central |
| Subir a Sandbox | `-filepath "verascan"` (upload por directorio, sin zip) |

**Nombre del Sandbox**: `jenkins-release` (configurable)

**Por que sandbox para releases**: Validacion completa sin afectar metricas de produccion. Permite experimentar de forma segura con nuevas features.

---

### 6. Policy Scan (main)

**Se ejecuta en**: Pushes a la rama `main`

**Proposito**: Certificacion de produccion para compliance.

| Sub-paso | Descripcion |
|----------|-------------|
| Unstash | Recupera `verascan/` y `app_name.txt` |
| Descargar API Wrapper | Obtiene la version mas reciente dinamicamente |
| Subir a Plataforma | `-filepath "verascan"` para evaluacion de policy |

**Por que policy scan en main**: Certificacion oficial de seguridad para codigo en produccion. Proporciona trazabilidad para regulaciones como SOC2, PCI-DSS, etc.

---

## Nombres de Perfiles de Aplicacion

El pipeline construye automaticamente el nombre del perfil:

**Formato por defecto**: `JOB_NAME` (multibranch: `org/repo/branch`)

| Job de Jenkins | Veracode App Name (por defecto) |
|----------------|----------------------------------|
| `acme-corp/api-service/main` | `acme-corp/api-service/main` |
| `myorg/frontend/feature%2Fauth` | `myorg/frontend/feature%2Fauth` |

Para la mayoria de los equipos el sufijo de rama por defecto no es deseable. Definir `VERACODE_APP_NAME` con un valor estable (por ejemplo `acme-corp/api-service`) a nivel de folder o job para que todas las ramas escaneen contra el mismo perfil.

---

## Manejo Multi-Artefacto

El pipeline maneja repositorios con multiples artefactos desplegables.

### Pipeline Scans (feature / PR)

Cada artefacto se escanea individualmente:

```text
verascan/
  backend-api.jar    -> escaneado por separado
  frontend.zip       -> escaneado por separado
  common-lib.jar     -> escaneado por separado

scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

### Platform Scans (release / main)

Todos los artefactos se suben juntos via upload por directorio:

```text
-filepath "verascan"    # El API Wrapper maneja el upload multi-archivo nativamente
```

**Nota**: No se crea ningun bundle zip. El Java API Wrapper acepta una ruta de directorio directamente, lo que evita rechazos por zip bomb.

---

## Resultados de Escaneo

### Resultados de Pipeline Scan

Los resultados se guardan por artefacto en `scan_results/` y se archivan como artefactos del build:

```text
scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

**Ver en Jenkins**: Abrir el build > **Build Artifacts** > `scan_results/`

**Estructura JSON**:
```json
{
  "findings": [...],
  "pipeline_scan": {...},
  "scan_status": "SUCCESS"
}
```

### Resultados de Platform Scans (Sandbox / Policy)

Verificar en la Plataforma Veracode:
1. Ingresar a [analysiscenter.veracode.com](https://analysiscenter.veracode.com)
2. Navegar al perfil de aplicacion
3. Ver findings, estado de compliance y tendencias

---

## Personalizacion

### Cambiar la Policy del Gate de PR

En la etapa `Pipeline Scan (PR Gate)`:

```groovy
--policy_name "Tu Policy Personalizada"
```

### Cambiar el Gate de Severidad

```bash
--fail_on_severity "Very High"           # Falla solo en Very High
--fail_on_severity "Very High, High"     # Falla en Very High y High (default)
--fail_on_severity ""                    # Sin gate (solo informativo)
```

### Nombre de Sandbox Personalizado

En la etapa `Sandbox Scan (release)`:

```groovy
-sandboxname "tu-nombre-sandbox"
```

### Agregar un Paso de Build

Si el autoempaquetador no funciona para tu proyecto, agrega una etapa de build antes de `Empaquetar Artefactos`:

```groovy
stage('Build') {
    steps {
        sh '''
            # Maven
            mvn -B clean package -DskipTests

            # Gradle
            ./gradlew build -x test

            # npm
            npm ci && npm run build
        '''
    }
}
```

### Cambiar Ramas de Trigger

Modificar los directivos `when {}`:

```groovy
branch pattern: 'develop|release/*|feature/*', comparator: 'REGEXP'
```

### Fijar el Label del Agente

El pipeline usa `agent { label 'linux' }`. Ajustar segun tus agentes:

```groovy
agent { label 'docker-linux' }
// o
agent any
```

---

## Parametros de Pipeline Scan

| Parametro | Descripcion | Usado en |
|-----------|-------------|----------|
| `-f` | Archivo a escanear | Todos los pipeline scans |
| `-vid` | Veracode API ID | Todos los scans |
| `-vkey` | Veracode API Key | Todos los scans |
| `--fail_on_severity` | Severidades que fallan el build | Feature (vacio), PR (Very High, High) |
| `--policy_name` | Policy contra la cual evaluar | PR scans |
| `--issue_details` | Incluir detalle de findings | Todos los scans |
| `-jo` | Solo salida JSON | Todos los scans |

---

## Parametros del API Wrapper

| Parametro | Descripcion | Usado en |
|-----------|-------------|----------|
| `-action` | `UploadAndScan` | Todos los platform scans |
| `-appname` | Nombre del perfil de aplicacion | Todos los platform scans |
| `-createprofile` | Crear app si no existe | Todos los platform scans |
| `-autoscan` | Iniciar el scan automaticamente | Todos los platform scans |
| `-sandboxname` | Nombre del sandbox | Release scans |
| `-createsandbox` | Crear sandbox si no existe | Release scans |
| `-filepath` | Ruta a los artefactos (archivo o directorio) | Todos los platform scans |
| `-version` | Etiqueta de version del scan | Todos los platform scans |

---

## Alternativa: Plugin Veracode de Jenkins

Este pipeline usa el Veracode CLI, el Pipeline Scanner JAR y el Java API Wrapper directamente. Esto mantiene el pipeline portable y evita una dependencia rigida con el plugin Veracode de Jenkins.

Si preferis usar el plugin oficial de Veracode para Jenkins, las etapas de platform scan pueden reemplazarse con el step `veracode`:

```groovy
veracode(
    applicationName: env.VERACODE_APP_NAME_RESOLVED,
    criticality: 'High',
    sandboxName: 'jenkins-release',
    createSandbox: true,
    scanName: "Release ${BUILD_NUMBER}",
    waitForScan: false,
    timeout: 60,
    createProfile: true,
    teams: '',
    canFailJob: true,
    debug: false,
    uploadIncludesPattern: 'verascan/**/*',
    vid: "${VERACODE_API_ID}",
    vkey: "${VERACODE_API_KEY}"
)
```

Ver la [referencia del plugin Veracode Scan](https://www.jenkins.io/doc/pipeline/steps/veracode-scan/) para la lista completa de parametros.

---

## Troubleshooting

### No se encontraron artefactos

**Sintomas**: La etapa Package falla con "No se encontraron artefactos empaquetados"

**Soluciones**:
1. Asegurarse de que el proyecto compila exitosamente antes del empaquetado
2. Verificar que los artefactos compilados existen en las ubicaciones esperadas
3. Revisar la salida del autoempaquetador del Veracode CLI
4. Agregar un paso de build explicito antes del autoempaquetador

### Pipeline Scan no produce resultados

**Sintomas**: `results.json` no se crea para un artefacto

**Causas**:
- El artefacto no es escaneable (test JARs, resource bundles)
- El artefacto no contiene codigo de aplicacion
- Tipo de modulo no soportado

**Soluciones**:
1. Revisar la salida del Pipeline Scanner buscando warnings
2. Verificar que el artefacto contiene codigo de aplicacion
3. Esto es normal para algunos artefactos; el pipeline continua

### El Policy Gate falla inesperadamente

**Sintomas**: El PR pipeline scan falla sin issues obvios

**Soluciones**:
1. Verificar que el nombre de la policy matchea exactamente (case-sensitive)
2. Confirmar que la policy existe en tu cuenta de Veracode
3. Revisar `results.json` para findings especificos
4. Verificar los umbrales de la policy

### La etapa de Change Request no se dispara

**Sintomas**: Los builds de PR / MR no ejecutan la etapa gateada

**Causa**: `changeRequest()` solo se dispara en un job **Multibranch Pipeline**. Un job Pipeline clasico no provee `CHANGE_ID`.

**Soluciones**:
1. Convertir el job a Multibranch Pipeline
2. Para GitHub: usar el plugin **GitHub Branch Source**
3. Para Bitbucket: usar el plugin **Bitbucket Branch Source**
4. Para GitLab: usar el plugin **GitLab Branch Source**

### Falla el upload de Sandbox / Policy

**Sintomas**: El API Wrapper falla durante el upload

**Soluciones**:
1. Verificar que las credenciales `veracode-api-id` y `veracode-api-key` esten configuradas
2. Verificar que las credenciales tengan permisos de upload
3. Asegurarse de que el nombre del perfil de aplicacion sea valido (sin caracteres especiales)
4. Revisar la salida del API Wrapper

### `unstash` falla en etapas paralelas

**Sintomas**: Una de las etapas de escaneo paralelas falla con "No such saved stash"

**Causa**: La etapa `Empaquetar Artefactos` fallo antes de que corriera el `stash` (en `post { success }`).

**Soluciones**:
1. Inspeccionar los logs de la etapa Package
2. Confirmar que el autoempaquetador produjo al menos un `.war` / `.jar` / `.zip`
3. Agregar un paso de build explicito si fuera necesario

---

## Archivos

| Archivo | Descripcion |
|---------|-------------|
| `Jenkinsfile` | Definicion del pipeline declarativo |
| `veracode-strategy.md` | Esta documentacion |

---

## Quick Start

1. Copiar el `Jenkinsfile` a la raiz de tu repositorio
2. Crear un job **Multibranch Pipeline** en Jenkins apuntando a tu repositorio
3. Agregar credenciales en **Manage Jenkins > Credentials**:
   - `veracode-api-id`
   - `veracode-api-key`
   - `srcclr-api-token` (opcional)
4. (Opcional) Definir `VERACODE_APP_NAME` a nivel folder o job
5. Push a una rama `feature/*` o abrir un PR para disparar el primer scan
6. Revisar resultados en **Build Artifacts > scan_results/** (Pipeline Scan) o en la Plataforma Veracode (Sandbox / Policy)

---

## Buenas Practicas

**Shift-Left**:
- Ejecutar Pipeline Scan en cada commit
- Habilitar gates en PRs para bloquear findings High y Very High
- Educar al equipo sobre findings comunes

**Compliance**:
- Requerir Policy Scan antes de produccion
- Mantener historial de scans
- Documentar excepciones y mitigaciones

**Optimizacion**:
- Usar Pipeline Scan para iteracion rapida
- Reservar Policy Scan para releases oficiales
- Cachear dependencias en el agente para acelerar builds

**SCA**:
- Monitorear nuevos CVEs continuamente
- Actualizar dependencias regularmente
- Revisar licencias antes de adoptar librerias

---

## Recursos

- [Documentacion de Veracode](https://docs.veracode.com)
- [Instalacion del Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [Plugin Veracode de Jenkins](https://plugins.jenkins.io/veracode-scan/)
- [Sintaxis de Jenkins Declarative Pipeline](https://www.jenkins.io/doc/book/pipeline/syntax/)
