# Pipeline de Seguridad Veracode para Azure Pipelines

Archivo del pipeline: [`azure-pipelines.yml`](./azure-pipelines.yml). Colocalo en la raiz de tu repositorio y crea un pipeline a partir de el.

**Tecnologias soportadas**: todo lo que soporta el [autopackager de Veracode CLI](https://docs.veracode.com/r/About_auto_packaging). El pool es `ubuntu-latest` (Java preinstalado).

---

## Estrategia de Escaneo

| Disparador | Stage | Producto Veracode | Gate |
|------------|-------|-------------------|------|
| Push a `feature/*` | `PipelineScan` | Pipeline Scan | Falla con cualquier hallazgo |
| Pull request a la rama por defecto | `PipelineScan` | Pipeline Scan | Politica `Veracode Recommended Very High` |
| Push a la rama por defecto | `PolicyScan` | Policy Scan | Politica de la plataforma |
| Todos los anteriores | `SCA` | SCA basado en agente | No bloqueante |

Codigos de salida de Pipeline Scan ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` sin hallazgos, `1-200` cantidad de hallazgos que cumplen el criterio, `253-255` timeout o error. Cualquier codigo distinto de cero falla el stage.

---

## Estructura del Pipeline

```text
trigger: main, feature/*        pr: main (solo GitHub/Bitbucket)
                 |
     +-----------+------------+
     |                        |
  Package                    SCA (dependsOn: [], en paralelo)
     |
     +-----------------------------+
     |                             |
  PipelineScan                 PolicyScan
  push feature/*: cualquier    push a DEFAULT_BRANCH:
  hallazgo falla               UploadAndScan
  PR a DEFAULT_BRANCH: politica
```

| Stage | Condicion |
|-------|-----------|
| `Package`, `SCA` | No es un PR desde fork (`System.PullRequest.IsFork`) |
| `PipelineScan` | Build que no es PR en `refs/heads/feature/*`, o PR cuyo `System.PullRequest.TargetBranch` es `DEFAULT_BRANCH` |
| `PolicyScan` | Build que no es PR en `refs/heads/<DEFAULT_BRANCH>` |

La validacion del destino del PR acepta `main` y `refs/heads/main`, asi funciona sin importar el formato del proveedor del repositorio.

---

## Configuracion

### 1. Grupo de variables

**Pipelines > Library > + Variable group**, nombre `veracode-credentials`, luego autorizalo para el pipeline.

| Variable | Secreta | Requerida | Descripcion |
|----------|---------|-----------|-------------|
| `VERACODE_API_ID` | Si | Si | Veracode API ID |
| `VERACODE_API_KEY` | Si | Si | Veracode API Key |
| `SRCCLR_API_TOKEN` | Si | Para SCA | Token de SCA basado en agente |
| `VERACODE_APP_NAME` | **No** | No | Nombre del perfil de aplicacion. Por defecto `{org}/{project}/{repo}` |

Las variables secretas no se exponen automaticamente a los scripts, por eso el pipeline las mapea con `env:`. `VERACODE_APP_NAME` debe ser no secreta porque se lee desde el entorno.

Credenciales de API: [Generar credenciales de API](https://docs.veracode.com/r/t_create_api_creds). Token de SCA: [Crear un agente SCA](https://docs.veracode.com/r/t_sc_cli_agent).

### 2. Disparador de pull request

| Repositorio | Como se disparan las ejecuciones de PR |
|-------------|----------------------------------------|
| Azure Repos Git | `pr:` en YAML se ignora. Agrega una politica de rama **Build Validation** en la rama por defecto (**Project settings > Repositories > Branches > main > Branch policies**) y marcala como Required. |
| GitHub, Bitbucket Cloud | Aplica el bloque `pr:` del YAML. |

Ver [Troubleshoot pipeline triggers](https://learn.microsoft.com/en-us/azure/devops/pipelines/troubleshooting/troubleshoot-triggers).

### 3. Rama por defecto

Si tu rama por defecto no es `main`, cambia `DEFAULT_BRANCH` y ambas listas `branches.include`.

---

## Detalle de Stages

### Package

1. Instala Veracode CLI en `$(Agent.TempDirectory)` para que el binario no se publique.
2. Ejecuta `veracode package --source $(Build.SourcesDirectory) --output verascan --trust` dentro de `$(Build.ArtifactStagingDirectory)/veracode`.
3. Escribe cada `.war`, `.jar`, `.zip` en `artifact_list.txt` junto a (no dentro de) `verascan/`, y falla si no hay ninguno.
4. Publica la carpeta como el artefacto de pipeline `veracode`.

Mantener `artifact_list.txt` fuera de `verascan/` evita que se suba a la Plataforma Veracode durante el policy scan.

### SCA

Ejecuta `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"` en paralelo con Package. El valor de `--appname` es el mismo perfil de aplicacion al que sube el policy scan, asi los hallazgos del agente quedan en el mismo perfil. El stage lo recalcula desde `SYSTEM_COLLECTIONURI`, `SYSTEM_TEAMPROJECT` y `BUILD_REPOSITORY_NAME` para seguir siendo `dependsOn: []`.

`--appname` hace que el agente llame a la Plataforma Veracode, lo que requiere credenciales HMAC ademas de `SRCCLR_API_TOKEN`. El agente las lee unicamente desde `VERACODE_API_KEY_ID` y `VERACODE_API_KEY_SECRET`, por eso el paso mapea tu API ID y key existentes a esos dos nombres. Sin ellas el escaneo falla con `HMAC authentication failed for license API` ([credenciales HMAC](https://docs.veracode.com/r/HMAC_credentials)). Ademas el perfil debe tener al menos un static scan completado antes de que el agente pueda enlazarlo ([comandos del agente SCA](https://docs.veracode.com/r/SCA_agent_commands)). Los errores se ignoran con `|| echo`. Quita ese sufijo para aplicar la politica de SCA.

### PipelineScan

- Valida que las credenciales tengan valores reales y no macros `$(VAR)` sin expandir.
- Escanea cada artefacto por separado, continua despues de un fallo y falla el stage al final si algun artefacto fallo.
- `Build.Reason == PullRequest` agrega `--policy_name "$(PR_GATE_POLICY)"`; los push a feature no agregan argumentos de gate.
- Publica `scan_results/` como `veracode-pipeline-scan-results-<intento>` incluso si falla.

### PolicyScan

Descarga el ultimo [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) y ejecuta `UploadAndScan` sobre `verascan/`.

| Parametro | Valor |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` o `{org}/{project}/{repo}` |
| `-createprofile` / `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `$(Build.BuildNumber)-$(System.JobAttempt)` (unico al reintentar) |

---

## Notas de Comportamiento

- **Ejecuciones duplicadas (repos GitHub/Bitbucket)**: un push a `feature/x` con un PR abierto puede iniciar una ejecucion CI y una de PR. Usa la ejecucion del PR como check requerido.
- **PRs desde forks**: se omiten, porque por defecto los secretos no se exponen a builds de forks.
- **Ejecuciones manuales**: una ejecucion manual en `feature/*` se comporta como un push (Pipeline Scan, cualquier hallazgo falla). En la rama por defecto ejecuta el policy scan.

---

## Personalizacion

**Cambiar la politica del gate de PR**: edita `PR_GATE_POLICY`. Para una politica personalizada, descargala con `--request_policy` y usa `--policy_file` ([parametros](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relajar el gate de feature**: define `GATE_ARGS=(--fail_on_severity "Very High, High")` en la rama `else`.

**Escanear todas las ramas que no son la por defecto**: define `trigger.branches.include` como `'*'` y cambia la condicion de feature a:

```yaml
ne(variables['Build.SourceBranch'], format('refs/heads/{0}', variables['DEFAULT_BRANCH']))
```

---

## Solucion de Problemas

| Sintoma | Revisar |
|---------|---------|
| `VERACODE_API_ID no esta definida` | Grupo de variables no vinculado o no autorizado, o nombre de variable distinto. |
| Los builds de PR nunca corren (Azure Repos) | Falta la politica de rama Build Validation. `pr:` no aplica a Azure Repos. |
| `PipelineScan` omitido en PR | El destino del PR no es `DEFAULT_BRANCH`. |
| Se ignora el nombre de app personalizado | `VERACODE_APP_NAME` esta marcada como secreta, por eso no esta en el entorno del script. Hazla no secreta. |
| Pipeline Scan sale con `255` | Credenciales invalidas, red bloqueada hacia `api.veracode.com`, o artefacto no soportado. |
| Pipeline Scan sale con `253` o `254` | Timeout del escaneo. Agrega `--timeout <minutos>` (maximo 60). |
| `UploadAndScan` rechazado | Puede haber un escaneo previo en curso para el perfil. |

---

## Recursos

- [Disparadores de Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/triggers)
- [Build de repositorios Azure Repos Git](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git)
- [Parametros de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Script CI de SCA](https://docs.veracode.com/r/c_sc_ci_script)
