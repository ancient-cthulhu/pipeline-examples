# Pipeline de Seguridad Veracode para Bitbucket Pipelines

Archivo del pipeline: [`veracode-scans.yml`](./veracode-scans.yml). Renombralo a `bitbucket-pipelines.yml` en la raiz de tu repositorio.

**Tecnologias soportadas**: todo lo que soporta el [autopackager de Veracode CLI](https://docs.veracode.com/r/About_auto_packaging). La imagen por defecto es `maven:3.9-eclipse-temurin-17`; cambiala si tu proyecto necesita otro toolchain.

---

## Estrategia de Escaneo

| Disparador | Pipeline | Producto Veracode | Gate |
|------------|----------|-------------------|------|
| Push a `main` | `branches: main` | Policy Scan | Politica de la plataforma |
| Push a cualquier rama que no es la por defecto | `default` | Pipeline Scan | Reporta cualquier hallazgo, no bloqueante (el step queda Failed, Ignored) |
| PR con destino `main` | `veracode-pr` | Pipeline Scan | Politica `Veracode Recommended Very High` |
| Todos los anteriores | (mismo pipeline) | Escaneo IaC (Veracode CLI) | No bloqueante |
| Todos los anteriores | (mismo pipeline) | SCA basado en agente | No bloqueante |

Codigos de salida de Pipeline Scan ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` sin hallazgos, `1-200` cantidad de hallazgos que cumplen el criterio, `253-255` timeout o error. Cualquier codigo distinto de cero falla el step; en el step de rama ese fallo se ignora (ver abajo).

---

## Estructura del Pipeline

Los push usan los selectores clasicos, que Bitbucket resuelve de lo mas especifico a lo mas general y que nunca coinciden con push de tags. Los pull requests usan una [condicion de inicio](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/) (`triggers:`), porque los selectores clasicos `pull-requests:` solo coinciden con la rama **origen** del PR, mientras que `pullrequest-push` con `BITBUCKET_PR_DESTINATION_BRANCH` filtra por destino.

```text
pipelines:
  branches: main   -> package -> parallel( sca, Policy Scan )
  default          -> package -> parallel( sca, Pipeline Scan )   toda otra rama, sin tags
  custom:  veracode-pr -> package -> parallel( sca, Pipeline Scan, gate de politica )

triggers:
  pullrequest-push  BITBUCKET_PR_DESTINATION_BRANCH == "main" -> veracode-pr
```

Un nombre exacto de rama gana sobre `default`, asi `main` recibe el policy scan y toda otra rama el pipeline scan. `default` [excluye los push de tags](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/). Los triggers solo pueden iniciar pipelines definidos en `pipelines.custom`, por eso el pipeline de PR vive ahi; tambien aparece en **Run pipeline** para ejecuciones manuales.

---

## Variables del Repositorio

**Repository settings > Pipelines > Repository variables**

| Variable | Secured | Requerida | Descripcion |
|----------|---------|-----------|-------------|
| `VERACODE_API_ID` | Si | Si | Veracode API ID |
| `VERACODE_API_KEY` | Si | Si | Veracode API Key |
| `SRCCLR_API_TOKEN` | Si | Para SCA | Token de SCA basado en agente |
| `VERACODE_APP_NAME` | No | No | Nombre del perfil de aplicacion. Por defecto `$BITBUCKET_REPO_FULL_NAME` (`workspace/repo`) |
| `VERACODE_BUSINESS_CRITICALITY` | No | No | `VeryHigh`, `High`, `Medium`, `Low`, `VeryLow`. Solo se usa al crear el perfil. Por defecto `VeryHigh` |

Credenciales de API: [Generar credenciales de API](https://docs.veracode.com/r/t_create_api_creds). Token de SCA: [Crear un agente SCA](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Detalle de Steps

### Empaquetar Artefactos

Instala Veracode CLI en `/opt/veracode-cli` (fuera del repositorio clonado, asi el binario del CLI no forma parte del codigo escaneado), ejecuta `veracode package --source . --output verascan --trust`, escribe cada `.war`, `.jar`, `.ear`, `.zip`, `.apk`, `.ipa` (los [tipos de salida del autopackager](https://docs.veracode.com/r/About_auto_packaging)) en `artifact_list.txt` y falla si no hay ninguno. `verascan/**` y `artifact_list.txt` pasan a los siguientes steps como artefactos.

### SCA Basado en Agente

Ejecuta `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor`. Los errores se ignoran con `|| echo` para que SCA nunca bloquee. Quita ese sufijo para aplicar la politica de SCA.

### Escaneo IaC

Corre dentro del step `Empaquetar Artefactos`, justo despues de instalar el CLI, asi escanea los archivos del repositorio y no una salida de build:

```bash
veracode scan --source . --type directory --format table
```

Es el [escaneo de contenedores, IaC y secretos](https://docs.veracode.com/r/veracode_scan) del Veracode CLI apuntando al directorio de trabajo. Se autentica con `VERACODE_API_KEY_ID` y `VERACODE_API_KEY_SECRET`, derivadas de las variables del repositorio existentes. Los errores se ignoran con `|| echo`, asi nunca bloquea.

### Pipeline Scan (rama / gate PR)

Ambos steps comparten un script mediante un anchor YAML (`&pipeline_scan_loop`). Cada step define `GATE_POLICY` primero:

| Step | `GATE_POLICY` | Resultado |
|------|---------------|-----------|
| `Pipeline Scan (rama)` | vacio | Sin criterio de fallo, cualquier hallazgo falla el step. `on-fail: strategy: ignore` mantiene el build en verde |
| `Pipeline Scan (gate PR)` | `Veracode Recommended Very High` | Agrega `--policy_name`, solo fallan los hallazgos que violan la politica |

Cada artefacto se escanea por separado, el ciclo continua despues de un fallo y el step falla al final si algun artefacto fallo. Los resultados se guardan en `scan_results/<artefacto>_results.json` y se suben como artefacto scoped con `capture-on: always`, asi se conservan cuando el escaneo falla (por defecto solo se guardan los artefactos de steps exitosos).

### Policy Scan

Descarga el ultimo [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) y ejecuta `UploadAndScan` sobre toda la carpeta `verascan/`.

| Parametro | Valor |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` o `workspace/repo` |
| `-createprofile` | `true` |
| `-criticality` | `VERACODE_BUSINESS_CRITICALITY` o `VeryHigh`. [Requerido con `-createprofile`](https://docs.veracode.com/r/r_uploadandscan) |
| `-filepath` | `verascan` |
| `-version` | `<rama>-<build number>-<timestamp>` (unico al reintentar el step) |

---

## Notas de Comportamiento

- **Rama por defecto**: Bitbucket no tiene una variable predefinida para la rama por defecto. Si la tuya no es `main`, renombra el selector `main:` en `branches:` y actualiza la condicion en `triggers:`.
- **Ejecuciones duplicadas**: un push a una rama con un PR abierto a `main` inicia el pipeline `default` y `veracode-pr`, porque los triggers son acumulativos con los selectores clasicos. Ambos reportan un estado de build sobre el mismo commit, y el check de merge de Bitbucket ("Minimum number of successful builds for the last commit with no failed builds") cuenta todos los builds de ese commit. Por eso el Pipeline Scan de rama no es bloqueante: si lo fuera, cualquier hallazgo bloquearia el PR aunque el gate de politica de `veracode-pr` pase. Para aplicar el gate, activa ese check de merge y **Prevent a merge with unresolved merge checks** (Premium) en **Settings > Workflow > Branch restrictions**.
- **Limite de tasa**: las ejecuciones duplicadas escanean cada artefacto dos veces. Una cuenta de Veracode puede iniciar como maximo seis Pipeline Scans por minuto (HTTP 429 por encima).
- **PRs desde forks**: Bitbucket no ejecuta los pipelines del repositorio padre para pull requests desde forks ([BCLOUD-13162](https://jira.atlassian.com/browse/BCLOUD-13162)), asi que esos PRs no reciben escaneo de Veracode.
- **Las lineas del script comparten shell**: las variables definidas en un item de `script:` (por ejemplo `GATE_POLICY`) estan disponibles en los items siguientes del mismo step.

---

## Personalizacion

**Cambiar la politica del gate de PR**: edita `GATE_POLICY` en el step `Pipeline Scan (gate PR)`. Para una politica personalizada, descargala con `--request_policy` y usa `--policy_file` ([parametros](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relajar el gate de rama**: agrega `--fail_on_severity "Very High, High"` en la rama `else` del ciclo de escaneo.

**Hacer bloqueante el escaneo de rama**: quita `on-fail` de `Pipeline Scan (rama)`. Los hallazgos de rama fallaran el build, lo que tambien bloquea cualquier PR abierto desde esa rama.

**Limitar que ramas se escanean**: reemplaza `default:` por selectores explicitos en `branches:`, por ejemplo `'feature/**'`. Las ramas que no coincidan con nada no ejecutan pipeline.

**Escanear tambien los push de tags**: agrega un selector `tags:` con los steps que quieras. `default` nunca corre para tags.

**Evitar el trigger**: mueve el pipeline de PR a `pull-requests:` y elimina `triggers:`. Ese selector solo coincide con la rama origen del PR, asi que necesitas validar `BITBUCKET_PR_DESTINATION_BRANCH` dentro del script, como indica la [KB de Atlassian](https://support.atlassian.com/bitbucket-cloud/kb/trigger-pipelines-on-pull-request-to-destination-branch/).

---

## Solucion de Problemas

| Sintoma | Revisar |
|---------|---------|
| No inicia ningun pipeline | Tu rama por defecto no es `main`, asi que el selector `main:` y la condicion del PR apuntan a la rama equivocada. |
| `No se encontraron artefactos empaquetados` | La imagen no tiene tu toolchain de build. Cambia `image:` o agrega un paso de build antes del autopackager. |
| Pipeline Scan sale con `255` | Credenciales invalidas, red bloqueada hacia `api.veracode.com`, o artefacto no soportado. |
| Pipeline Scan sale con `254` | El escaneo supero el limite por defecto de 60 minutos (el maximo). Divide el artefacto o excluye modulos. |
| Pipeline Scan sale con `253` | El escaneo supero tu valor de `--timeout`. Aumentalo (maximo 60). |
| `UploadAndScan` rechazado | Puede haber un escaneo previo en curso para el perfil. Esperalo o cancelalo en la Plataforma. |

---

## Recursos

- [Condiciones de inicio de Bitbucket](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/)
- [Parametros de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Codigos de estado de Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Script CI de SCA](https://docs.veracode.com/r/c_sc_ci_script)
