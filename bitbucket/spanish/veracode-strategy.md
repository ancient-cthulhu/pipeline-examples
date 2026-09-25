# Pipeline de Seguridad Veracode para Bitbucket Pipelines

Archivo del pipeline: [`veracode-scans.yml`](./veracode-scans.yml). Renombralo a `bitbucket-pipelines.yml` en la raiz de tu repositorio.

**Tecnologias soportadas**: todo lo que soporta el [autopackager de Veracode CLI](https://docs.veracode.com/r/About_auto_packaging). La imagen por defecto es `maven:3.9-eclipse-temurin-17`; cambiala si tu proyecto necesita otro toolchain.

---

## Estrategia de Escaneo

| Disparador | Pipeline custom | Producto Veracode | Gate |
|------------|-----------------|-------------------|------|
| Push a `main` | `veracode-policy` | Policy Scan | Politica de la plataforma |
| Push a `feature/**` | `veracode-feature` | Pipeline Scan | Falla con cualquier hallazgo |
| PR con destino `main` | `veracode-pr` | Pipeline Scan | Politica `Veracode Recommended Very High` |
| Todos los anteriores | (mismo pipeline) | SCA basado en agente | No bloqueante |

Codigos de salida de Pipeline Scan ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` sin hallazgos, `1-200` cantidad de hallazgos que cumplen el criterio, `253-255` timeout o error. Cualquier codigo distinto de cero falla el step.

---

## Estructura del Pipeline

El archivo usa las [condiciones de inicio](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/) de Bitbucket (`triggers:`). Los selectores clasicos `pull-requests:` solo coinciden con la rama **origen** del PR; `pullrequest-push` con `BITBUCKET_PR_DESTINATION_BRANCH` filtra por destino de forma nativa, asi los PRs a otras ramas no inician nada.

```text
triggers:
  repository-push   BITBUCKET_BRANCH == "main"               -> veracode-policy
  repository-push   glob(BITBUCKET_BRANCH, "feature/**")     -> veracode-feature
  pullrequest-push  BITBUCKET_PR_DESTINATION_BRANCH == "main" -> veracode-pr

cada pipeline:  package  ->  parallel( sca, <step de escaneo> )
```

Los triggers solo pueden iniciar pipelines definidos en `pipelines.custom`. Esos pipelines tambien aparecen en **Run pipeline** para ejecuciones manuales.

---

## Variables del Repositorio

**Repository settings > Pipelines > Repository variables**

| Variable | Secured | Requerida | Descripcion |
|----------|---------|-----------|-------------|
| `VERACODE_API_ID` | Si | Si | Veracode API ID |
| `VERACODE_API_KEY` | Si | Si | Veracode API Key |
| `SRCCLR_API_TOKEN` | Si | Para SCA | Token de SCA basado en agente |
| `VERACODE_APP_NAME` | No | No | Nombre del perfil de aplicacion. Por defecto `$BITBUCKET_REPO_FULL_NAME` (`workspace/repo`) |

Credenciales de API: [Generar credenciales de API](https://docs.veracode.com/r/t_create_api_creds). Token de SCA: [Crear un agente SCA](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Detalle de Steps

### Empaquetar Artefactos

Instala Veracode CLI, ejecuta `veracode package --source . --output verascan --trust`, escribe cada `.war`, `.jar`, `.zip` en `artifact_list.txt` y falla si no hay ninguno. `verascan/**` y `artifact_list.txt` pasan a los siguientes steps como artefactos.

### SCA Basado en Agente

Ejecuta `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"`. El valor de `--appname` es el mismo perfil de aplicacion al que sube el policy scan, asi los hallazgos del agente quedan en el mismo perfil. Los errores se ignoran con `|| echo` para que SCA nunca bloquee. Quita ese sufijo para aplicar la politica de SCA.

### Pipeline Scan (feature / gate PR)

Ambos steps comparten un script mediante un anchor YAML (`&pipeline_scan_loop`). Cada step define `GATE_POLICY` primero:

| Step | `GATE_POLICY` | Resultado |
|------|---------------|-----------|
| `Pipeline Scan (feature)` | vacio | Sin criterio de fallo, cualquier hallazgo falla el step |
| `Pipeline Scan (gate PR)` | `Veracode Recommended Very High` | Agrega `--policy_name`, solo fallan los hallazgos que violan la politica |

Cada artefacto se escanea por separado, el ciclo continua despues de un fallo y el step falla al final si algun artefacto fallo. Los resultados se guardan en `scan_results/<artefacto>_results.json`.

### Policy Scan

Descarga el ultimo [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) y ejecuta `UploadAndScan` sobre toda la carpeta `verascan/`.

| Parametro | Valor |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` o `workspace/repo` |
| `-createprofile` / `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `<rama>-<build number>-<timestamp>` (unico al reintentar el step) |

---

## Notas de Comportamiento

- **Rama por defecto**: Bitbucket no tiene una variable predefinida para la rama por defecto. Si la tuya no es `main`, cambia ambas condiciones `main` en `triggers:`.
- **Ejecuciones duplicadas**: un push a `feature/x` con un PR abierto a `main` inicia `veracode-feature` y `veracode-pr`. Es esperado. Usa el resultado de `veracode-pr` como check de merge.
- **PRs desde forks**: las variables secured no estan disponibles para forks, asi que la autenticacion de los escaneos fallara ahi.
- **Las lineas del script comparten shell**: las variables definidas en un item de `script:` (por ejemplo `GATE_POLICY`) estan disponibles en los items siguientes del mismo step.

---

## Personalizacion

**Cambiar la politica del gate de PR**: edita `GATE_POLICY` en el step `Pipeline Scan (gate PR)`. Para una politica personalizada, descargala con `--request_policy` y usa `--policy_file` ([parametros](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relajar el gate de feature**: agrega `--fail_on_severity "Very High, High"` en la rama `else` del ciclo de escaneo.

**Escanear todas las ramas que no son la por defecto**: cambia la condicion de feature a:

```yaml
- condition: glob(BITBUCKET_BRANCH, "**") && BITBUCKET_BRANCH != "main"
```

**Usar selectores clasicos**: mueve los tres pipelines custom a `branches:` / `pull-requests:` y elimina `triggers:`. En ese caso necesitas validar `BITBUCKET_PR_DESTINATION_BRANCH` dentro del script, como indica la [KB de Atlassian](https://support.atlassian.com/bitbucket-cloud/kb/trigger-pipelines-on-pull-request-to-destination-branch/).

---

## Solucion de Problemas

| Sintoma | Revisar |
|---------|---------|
| No inicia ningun pipeline | El nombre de la rama no coincide con una condicion, o las condiciones dicen `main` y tu rama por defecto es otra. |
| `No se encontraron artefactos empaquetados` | La imagen no tiene tu toolchain de build. Cambia `image:` o agrega un paso de build antes del autopackager. |
| Pipeline Scan sale con `255` | Credenciales invalidas, red bloqueada hacia `api.veracode.com`, o artefacto no soportado. |
| Pipeline Scan sale con `253` o `254` | Timeout del escaneo. Agrega `--timeout <minutos>` (maximo 60). |
| `UploadAndScan` rechazado | Puede haber un escaneo previo en curso para el perfil. Esperalo o cancelalo en la Plataforma. |

---

## Recursos

- [Condiciones de inicio de Bitbucket](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/)
- [Parametros de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Codigos de estado de Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Script CI de SCA](https://docs.veracode.com/r/c_sc_ci_script)
