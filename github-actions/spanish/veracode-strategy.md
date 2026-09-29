# Pipeline de Seguridad Veracode para GitHub Actions

Archivo de workflow: [`veracode-scans.yml`](./veracode-scans.yml). Copialo a `.github/workflows/` en tu repositorio.

**Tecnologias soportadas**: todo lo que soporta el [autopackager de Veracode CLI](https://docs.veracode.com/r/About_auto_packaging) (Java, .NET, JavaScript/TypeScript, Python, Go, PHP, Ruby, Scala, Kotlin y mas).

---

## Estrategia de Escaneo

| Disparador | Producto Veracode | Gate | Proposito |
|------------|-------------------|------|-----------|
| Push a cualquier rama que no es la por defecto | Pipeline Scan | Falla con cualquier hallazgo | Feedback al desarrollador en cada push |
| Pull request a la rama por defecto | Pipeline Scan | Politica `Veracode Recommended Very High` | Demostrar que el PR es seguro para merge |
| Push a la rama por defecto | Policy Scan | Politica de la plataforma | Registro de cumplimiento del perfil de aplicacion |
| Todos los anteriores | Escaneo IaC (Veracode CLI) | No bloqueante | Errores de configuracion y secretos en los archivos del repo |
| Todos los anteriores | SCA basado en agente | No bloqueante | Analisis de dependencias de terceros |

Codigos de salida de Pipeline Scan ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` sin hallazgos, `1-200` cantidad de hallazgos que cumplen el criterio, `253-255` timeout o error. El workflow trata cualquier codigo distinto de cero como fallo.

---

## Estructura del Workflow

```text
on: push (todas las ramas) | pull_request (a main)
                    |
        +-----------+-----------+
        |                       |
     package                   sca
  (CLI autopackager     (no bloqueante)
   + escaneo IaC)
        |
        +---------------------------+
        |                           |
  pipeline-scan                policy-scan
  push a otra rama: cualquier  push rama por defecto:
  hallazgo falla               UploadAndScan
  PR: gate de politica
```

| Job | Se ejecuta cuando |
|-----|-------------------|
| `package` | Siempre (excepto PRs desde forks) |
| `sca` | Siempre (excepto PRs desde forks) |
| `pipeline-scan` | `pull_request`, o `push` a una rama que no es la por defecto |
| `policy-scan` | `push` a `github.event.repository.default_branch` |

---

## Secretos Requeridos

**Settings > Secrets and variables > Actions**

| Secreto | Requerido | Descripcion |
|---------|-----------|-------------|
| `VERACODE_API_ID` | Si | Veracode API ID |
| `VERACODE_API_KEY` | Si | Veracode API Key |
| `SRCCLR_API_TOKEN` | Para SCA | Token de SCA basado en agente |
| `VERACODE_APP_NAME` | No | Nombre del perfil de aplicacion. Por defecto `github.repository` (`org/repo`) |

Credenciales de API: [Generar credenciales de API](https://docs.veracode.com/r/t_create_api_creds). Token de SCA: [Crear un agente SCA](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Detalle de Jobs

### package

1. Hace checkout del codigo.
2. Instala Veracode CLI (`curl -fsS https://tools.veracode.com/veracode-cli/install | sh`).
3. Ejecuta `veracode package --source . --output verascan --trust`.
4. Ejecuta el escaneo IaC (abajo).
5. Escribe cada `.war`, `.jar`, `.zip` encontrado en `artifact_list.txt` y falla si no hay ninguno.
6. Sube `verascan/` y `artifact_list.txt` como el artefacto de workflow `verascan`.

### Escaneo IaC

Corre dentro del job `package`, justo despues de instalar el CLI, asi escanea los archivos del repositorio y no una salida de build:

```bash
./veracode scan --source . --type directory --format table
```

Es el [escaneo de contenedores, IaC y secretos](https://docs.veracode.com/r/veracode_scan) del Veracode CLI apuntando al directorio de trabajo. Lee los archivos de forma estatica y reporta errores de configuracion y secretos en el codigo. El CLI se autentica con `VERACODE_API_KEY_ID` y `VERACODE_API_KEY_SECRET`, que el step deriva de los secretos existentes. Los errores se ignoran con `|| echo`, asi nunca bloquea el build.

### sca

Ejecuta `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor`. Los errores se ignoran con `|| echo` para que SCA nunca bloquee el build. Quita ese sufijo para aplicar la politica de SCA.

### pipeline-scan

Escanea cada artefacto de `artifact_list.txt` por separado, continua despues de un fallo y al final falla el job si algun artefacto fallo.

| Evento | Argumentos agregados | Resultado |
|--------|----------------------|-----------|
| `push` a una rama que no es la por defecto | ninguno | Cada hallazgo cuenta, cualquier hallazgo falla el job |
| `pull_request` | `--policy_name "Veracode Recommended Very High"` | Solo los hallazgos que violan la politica fallan el job |

Los resultados se suben como `pipeline-scan-results` (`<artefacto>_results.json`), incluso si falla.

### policy-scan

Descarga el ultimo [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) desde Maven Central y ejecuta `UploadAndScan` sobre toda la carpeta `verascan/`, asi todos los modulos quedan en un solo build.

| Parametro | Valor |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` o `org/repo` |
| `-createprofile` | `true` |
| `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `<rama>-<run_number>-<run_attempt>` (unico por reintento) |

El job termina cuando la carga es aceptada. Los resultados aparecen en la Plataforma Veracode cuando el escaneo finaliza.

---

## Notas de Comportamiento

- **Rama por defecto**: `on.push.branches` es `'**'`, asi que corre en todas las ramas. Las condiciones de los jobs comparan contra `github.event.repository.default_branch`, asi que la division entre escaneo de rama y policy scan no necesita cambios. Solo el filtro de `pull_request` fija `main`, porque los filtros de disparadores no aceptan expresiones.
- **Ejecuciones duplicadas**: un push a una rama con un PR abierto dispara `push` y `pull_request`. Es esperado: la ejecucion de push da feedback completo, la del PR es el gate de merge. Marca solo el check del PR como requerido en la proteccion de ramas.
- **Concurrencia**: las ejecuciones nuevas cancelan las anteriores en la misma ref, excepto en la rama por defecto, donde las cargas de politica nunca se cancelan.
- **PRs desde forks**: se omiten, porque GitHub no expone secretos a esos PRs.
- **Runners self-hosted**: `actions/checkout@v6`, `upload-artifact@v7` y `download-artifact@v8` usan Node.js 24 y requieren Actions Runner 2.327.1 o superior. Se requiere Java para el scanner y el API wrapper (preinstalado en `ubuntu-latest`).

---

## Personalizacion

**Cambiar la politica del gate de PR**: edita `PR_GATE_POLICY` en el `env:` global. Las politicas integradas se soportan por nombre. Para una politica personalizada, descargala con `--request_policy` y usa `--policy_file` ([parametros](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relajar el gate de rama**: agrega argumentos en la rama `else` del bloque del gate, por ejemplo:

```bash
GATE_ARGS=(--fail_on_severity "Very High, High")
```

**Limitar que ramas se escanean**: reemplaza `'**'` en `on.push.branches` por una lista explicita, por ejemplo `main` y `'feature/**'`. Los push de tags nunca disparan este workflow, porque los filtros `branches` no coinciden con tags.

**Agregar un paso de build**: si el autopackaging no cubre tu proyecto, compila antes del autopackager o reemplazalo y escribe las rutas de tus artefactos en `artifact_list.txt`:

```yaml
- name: Build
  run: mvn -B clean package -DskipTests
```

---

## Solucion de Problemas

| Sintoma | Revisar |
|---------|---------|
| `No se encontraron artefactos empaquetados` | Ejecuta `veracode package --source . --output verascan --trust` localmente. Confirma que el tipo de proyecto esta soportado o agrega un paso de build. |
| Pipeline Scan sale con `255` | Credenciales invalidas, red bloqueada hacia `api.veracode.com`, o artefacto no soportado. |
| Pipeline Scan sale con `253` o `254` | Timeout del escaneo. Agrega `--timeout <minutos>` (maximo 60) o divide artefactos grandes. |
| El gate de PR falla inesperadamente | Abre `pipeline-scan-results` y revisa `results.json`. Confirma que el nombre de la politica coincide exactamente. |
| `UploadAndScan` rechazado | Puede haber un escaneo previo en curso para el perfil. Esperalo o cancelalo en la Plataforma. Verifica que el usuario de API tenga permisos de carga. |
| Jobs omitidos en la rama por defecto | La rama por defecto en GitHub no coincide con el filtro `branches:`. |

---

## Recursos

- [Parametros de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Codigos de estado de Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA basado en agente](https://docs.veracode.com/r/Agent_Based_Scans)
- [GitHub Actions: eventos que disparan workflows](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows)
