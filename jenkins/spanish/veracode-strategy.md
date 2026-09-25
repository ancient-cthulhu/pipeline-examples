# Pipeline de Seguridad Veracode para Jenkins

Archivos del pipeline:

| Archivo | Agente | Shell |
|---------|--------|-------|
| [`Jenkinsfile-linux`](./Jenkinsfile-linux) | label `linux` | POSIX `sh` (funciona con dash) |
| [`Jenkinsfile-win`](./Jenkinsfile-win) | label `windows` | Windows PowerShell 5.1+ |

Renombra el que uses a `Jenkinsfile` en la raiz de tu repositorio y crea un job **Multibranch Pipeline**. Ambos archivos estan probados con [verademo](https://github.com/veracode/verademo) (Maven, `app/pom.xml`).

---

## Estrategia de Escaneo

| Disparador | Stage | Producto Veracode | Gate |
|------------|-------|-------------------|------|
| Push a `feature/*` | `Pipeline Scan` | Pipeline Scan | Falla con cualquier hallazgo |
| Change request (PR) a la rama por defecto | `Pipeline Scan` | Pipeline Scan | Politica `Veracode Recommended Very High` |
| Push a la rama por defecto | `Policy Scan` | Policy Scan | Politica de la plataforma |
| Todos los anteriores | `SCA Basado en Agente` | SCA basado en agente | No bloqueante |

Codigos de salida de Pipeline Scan ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` sin hallazgos, `1-200` cantidad de hallazgos que cumplen el criterio, `253-255` timeout o error. Cualquier codigo distinto de cero falla el stage.

---

## Estructura del Pipeline

```text
Build (Maven) -> Empaquetar Artefactos (stash "verascan")
                         |
         +---------------+----------------+
         |               |                |
  SCA Basado en     Pipeline Scan     Policy Scan
  Agente (siempre)  push feature/*    push a DEFAULT_BRANCH
                    o CR a la rama
                    por defecto
```

| Stage | Condicion `when` |
|-------|------------------|
| `Pipeline Scan` | `!CHANGE_ID && BRANCH_NAME.startsWith('feature/')`, o `CHANGE_ID && CHANGE_TARGET == DEFAULT_BRANCH` |
| `Policy Scan` | `!CHANGE_ID && BRANCH_NAME == DEFAULT_BRANCH` |

`BRANCH_NAME`, `CHANGE_ID` y `CHANGE_TARGET` los definen los jobs Multibranch Pipeline. En un job Pipeline simple estan vacias, asi que solo corre SCA.

Los stages de escaneo hacen `unstash` en su propio subdirectorio (`pipeline-scan/`, `policy-scan/`) para que los stages en paralelo en el mismo agente no se sobrescriban.

---

## Configuracion

### Credenciales

**Manage Jenkins > Credentials**, tipo **Secret text**:

| ID | Requerida | Descripcion |
|----|-----------|-------------|
| `veracode-api-id` | Si | Veracode API ID |
| `veracode-api-key` | Si | Veracode API Key |
| `srcclr-api-token` | No | Token de SCA basado en agente. Si falta, SCA se omite |

Credenciales de API: [Generar credenciales de API](https://docs.veracode.com/r/t_create_api_creds). Token de SCA: [Crear un agente SCA](https://docs.veracode.com/r/t_sc_cli_agent).

### Variables de entorno opcionales (job o carpeta)

| Variable | Por defecto | Descripcion |
|----------|-------------|-------------|
| `VERACODE_APP_NAME` | Ruta del job sin el segmento de rama | Nombre del perfil de aplicacion |
| `MAVEN_POM_PATH` | `app/pom.xml` | Pom de Maven. Usa `pom.xml` para proyectos en la raiz |
| `VERACODE_SOURCE_DIR` | `app` | Directorio para el autopackager. Usa `.` para proyectos en la raiz |

`DEFAULT_BRANCH` y `PR_GATE_POLICY` se definen en el bloque `environment` del Jenkinsfile.

### Plugins

Pipeline (Declarative), Pipeline: Multibranch, Credentials Binding, Workspace Cleanup (`cleanWs`), Timestamper (`timestamps()`), mas el plugin de branch source de tu SCM (GitHub, Bitbucket, GitLab).

### Herramientas del agente

| Agente | Requerido en el PATH |
|--------|----------------------|
| Linux | `java` 8+, `mvn`, `curl`, `unzip`, GNU `grep` (para `-P`) |
| Windows | `java` 8+, `mvn`, Windows PowerShell 5.1+ |

---

## Detalle de Stages

### Build (Maven)

`mvn -B -f $MAVEN_POM_PATH clean package -DskipTests`. Elimina este stage si el autopackager solo es suficiente para tu proyecto.

### Empaquetar Artefactos

Resuelve `APP_NAME`, instala Veracode CLI, ejecuta `veracode package --trust`, escribe rutas relativas de artefactos en `artifact_list.txt`, falla si no hay ninguno y hace stash de `verascan/**` y la lista.

En multibranch `JOB_NAME` es `<carpeta>/<repo>/<rama>`. Se quita el ultimo segmento para que todas las ramas usen el mismo perfil de aplicacion.

### SCA Basado en Agente

| Agente | Comando |
|--------|---------|
| Linux | `curl -sSL https://sca-downloads.veracode.com/ci.sh \| sh -s -- scan --recursive --update-advisor --appname "$APP_NAME"` |
| Windows | Descarga `https://sca-downloads.veracode.com/ci.ps1` y lo ejecuta con `-ArgumentList scan, --recursive, --update-advisor, --appname, $env:APP_NAME` ([docs](https://docs.veracode.com/r/t_sc_agent_proxy)) |

No bloqueante en ambos: Linux ignora errores con `|| echo`, Windows usa `powershell(returnStatus: true)`.

`APP_NAME` es el valor resuelto en el stage de empaquetado, asi los hallazgos del agente quedan en el mismo perfil de aplicacion al que sube el policy scan.

### Pipeline Scan

Cada artefacto se escanea por separado, el ciclo continua despues de un fallo y el stage falla al final si algun artefacto fallo. Los change requests agregan `--policy_name "$PR_GATE_POLICY"`; los push a feature no agregan argumentos de gate. `scan_results/` se archiva incluso si falla.

### Policy Scan

Descarga el ultimo [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) y ejecuta `UploadAndScan` sobre `verascan/` con version `<rama>-<BUILD_NUMBER>`.

---

## Notas de Comportamiento

- **Builds duplicados**: con un branch source que descubre ramas y PRs, un push a `feature/x` con un PR abierto construye `feature/x` (gate de cualquier hallazgo) y `PR-n` (gate de politica). Usa el build del PR como status check requerido.
- **`disableConcurrentBuilds()`** aplica por job de rama, asi un policy scan en `main` nunca se superpone con otro build de `main`.

---

## Personalizacion

**Cambiar la politica del gate de PR**: edita `PR_GATE_POLICY`. Para una politica personalizada, descargala con `--request_policy` y usa `--policy_file` ([parametros](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relajar el gate de feature**: agrega `--fail_on_severity "Very High, High"` a la rama sin politica del comando de escaneo.

**Escanear todas las ramas que no son la por defecto**: cambia la expresion de feature a `!env.CHANGE_ID && env.BRANCH_NAME != env.DEFAULT_BRANCH`.

---

## Solucion de Problemas

| Sintoma | Revisar |
|---------|---------|
| Los stages de escaneo siempre se omiten | El job no es Multibranch, por eso `BRANCH_NAME` esta vacia. |
| `No se encontraron artefactos empaquetados` | `VERACODE_SOURCE_DIR` o `MAVEN_POM_PATH` no coinciden con tu estructura. |
| `grep: invalid option -- 'P'` (Linux) | El agente usa grep de BusyBox o BSD. Instala GNU grep. |
| Pipeline Scan sale con `255` | Credenciales invalidas, red bloqueada hacia `api.veracode.com`, o artefacto no soportado. |
| Pipeline Scan sale con `253` o `254` | Timeout del escaneo. Agrega `--timeout <minutos>` (maximo 60). |
| `UploadAndScan` rechazado | Puede haber un escaneo previo en curso para el perfil. |

---

## Recursos

- [Sintaxis de Jenkins Pipeline: when](https://www.jenkins.io/doc/book/pipeline/syntax/#when)
- [Parametros de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Script CI de SCA](https://docs.veracode.com/r/c_sc_ci_script)
