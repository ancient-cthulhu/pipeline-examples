# Pipeline de Seguridad Veracode para GitLab CI/CD

Archivo de pipeline: [`.gitlab-ci.yml`](./.gitlab-ci.yml). Copialo a la raiz de tu proyecto.

**Tecnologias soportadas**: todo lo que soporta el [autopackager de Veracode CLI](https://docs.veracode.com/r/About_auto_packaging) (Java, .NET, JavaScript/TypeScript, Python, Go, PHP, Ruby, Scala, Kotlin y mas).

---

## Estrategia de Escaneo

| Disparador | Producto Veracode | Gate | Proposito |
|------------|-------------------|------|-----------|
| Push a `feature/**` | Pipeline Scan | Falla con cualquier hallazgo | Feedback al desarrollador en cada push |
| Merge request a la rama por defecto | Pipeline Scan | Politica `Veracode Recommended Very High` | Demostrar que el MR es seguro para merge |
| Push a la rama por defecto | Policy Scan | Politica de la plataforma | Registro de cumplimiento del perfil de aplicacion |
| Todos los anteriores | SCA basado en agente | No bloqueante (`allow_failure: true`) | Analisis de dependencias de terceros |

Codigos de salida de Pipeline Scan ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` sin hallazgos, `1-200` cantidad de hallazgos que cumplen el criterio, `253-255` timeout o error. Cualquier codigo distinto de cero falla el job.

---

## Estructura del Pipeline

```text
workflow:rules  (MR a rama por defecto | push rama por defecto | push feature/**)
                    |
   stage: package   +-- package (CLI autopackager, APP_NAME via dotenv)
                    +-- sca     (needs: [], allow_failure)
                    |
   stage: scan      +-- pipeline-scan  push feature/**: cualquier hallazgo falla
                    |                  MR: gate de politica
                    +-- policy-scan    push rama por defecto: UploadAndScan
```

| Job | Se ejecuta cuando |
|-----|-------------------|
| `package` | Todo pipeline permitido por `workflow:rules` |
| `sca` | Todo pipeline permitido por `workflow:rules` |
| `pipeline-scan` | Evento MR del mismo proyecto, o push a una rama que no es la por defecto |
| `policy-scan` | Push a `$CI_DEFAULT_BRANCH` |

`workflow:rules` en orden:

1. Pipeline de MR cuyo destino es `$CI_DEFAULT_BRANCH`: se ejecuta.
2. Cualquier otro pipeline de MR: nunca.
3. Push a `$CI_DEFAULT_BRANCH`: se ejecuta. Se evalua antes de la regla 4 para que el policy scan corra aunque la rama por defecto sea origen de un MR abierto.
4. Push a una rama con MR abierto (`$CI_OPEN_MERGE_REQUESTS`): nunca, el pipeline del MR lo cubre. Nota: tambien aplica si el MR abierto apunta a una rama que no es la por defecto, en ese caso la rama feature no tiene pipeline.
5. Push a `feature/*` (regex `^feature\/`, incluye nombres anidados): se ejecuta.

---

## Variables CI/CD Requeridas

**Settings > CI/CD > Variables**

| Variable | Requerida | Flags | Descripcion |
|----------|-----------|-------|-------------|
| `VERACODE_API_ID` | Si | Masked | Veracode API ID |
| `VERACODE_API_KEY` | Si | Masked | Veracode API Key |
| `SRCCLR_API_TOKEN` | Para SCA | Masked | Token de SCA basado en agente. SCA se omite si no esta definido |
| `VERACODE_APP_NAME` | No | | Nombre del perfil de aplicacion. Por defecto `$CI_PROJECT_PATH` |

**No marques estas variables como Protected** salvo que `feature/*` sea un patron de rama protegida. Las variables Protected solo se exponen a pipelines en ramas y tags protegidos, asi que los pipelines de feature y MR correrian con credenciales vacias.

Credenciales de API: [Generar credenciales de API](https://docs.veracode.com/r/t_create_api_creds). Token de SCA: [Crear un agente SCA](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Detalle de Jobs

### package

Imagen `ubuntu:22.04`. Instala Veracode CLI, ejecuta `veracode package --source . --output verascan --trust`, escribe cada `.war`, `.jar`, `.zip` en `artifact_list.txt` y falla si no hay ninguno. Exporta `APP_NAME` mediante un reporte `dotenv` para los jobs siguientes.

### sca

Imagen `eclipse-temurin:17-jdk`, `needs: []` para iniciar de inmediato. Ejecuta `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"`, reutilizando el anchor compartido `&set_app_name`. El valor de `--appname` es el mismo perfil de aplicacion al que sube el policy scan, asi los hallazgos del agente quedan en el mismo perfil. `allow_failure: true` muestra los fallos de SCA como advertencias sin bloquear. Quitalo para aplicar la politica de SCA.

### pipeline-scan

Escanea cada artefacto por separado, continua despues de un fallo y al final falla el job si algun artefacto fallo.

| Origen del pipeline | Argumentos agregados | Resultado |
|---------------------|----------------------|-----------|
| `push` a `feature/**` | ninguno | Cada hallazgo cuenta, cualquier hallazgo falla el job |
| `merge_request_event` | `--policy_name "$PR_GATE_POLICY"` | Solo los hallazgos que violan la politica fallan el job |

Los resultados se guardan como artefactos del job (`scan_results/<artefacto>_results.json`, `when: always`, 1 mes).

### policy-scan

Descarga el ultimo [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) desde Maven Central y ejecuta `UploadAndScan` sobre toda la carpeta `verascan/`.

| Parametro | Valor |
|-----------|-------|
| `-appname` | `$APP_NAME` desde `package` |
| `-createprofile` | `true` |
| `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `<rama>-<pipeline_iid>-<job_id>` (unico por reintento) |

El job termina cuando la carga es aceptada. Los resultados aparecen en la Plataforma Veracode cuando el escaneo finaliza.

---

## Notas de Comportamiento

- **Rama por defecto**: todas las reglas usan `$CI_DEFAULT_BRANCH`. No requiere cambios si tu rama por defecto es `master` o `develop`.
- **MRs a otras ramas**: la regla 2 los bloquea. Si una rama feature tiene un MR abierto hacia otra rama, la regla 3 tambien bloquea sus pipelines de push y esa rama no se escanea. Quita la regla 3 si eso afecta tu flujo (push y MR correran ambos).
- **MRs desde forks**: `pipeline-scan` requiere `$CI_MERGE_REQUEST_SOURCE_PROJECT_ID == $CI_MERGE_REQUEST_PROJECT_ID`. Los pipelines de MR desde forks corren en el fork sin tus variables.
- **Aprobacion de MRs**: para bloquear merges con gate fallido, habilita **Settings > Merge requests > Pipelines must succeed**.

---

## Personalizacion

**Cambiar la politica del gate de MR**: edita `PR_GATE_POLICY` en `variables:`. Para politicas personalizadas, descargala con `--request_policy` y usa `--policy_file` ([parametros](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relajar el gate de feature**: agrega `--fail_on_severity "Very High, High"` a la llamada del scanner en la rama `else`.

**Security Dashboard de GitLab**: agrega `--gl_vulnerability_generation true` a la llamada del scanner y publica `veracode_gitlab_vulnerabilities.json` en `artifacts:reports:sast` ([ejemplo de Veracode](https://docs.veracode.com/r/Pipeline_Scan_Example_for_Using_GitLab_and_Gradle_with_Automatic_Vulnerability_Generation_Using_a_Built_in_Policy)). Renombra el archivo por artefacto dentro del loop.

**Escanear todas las ramas que no son la por defecto**: reemplaza la ultima entrada de `workflow:rules` por `- if: $CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH`.

**Agregar un paso de build**: compila en `package` antes del autopackager, o reemplazalo y escribe las rutas de tus artefactos en `artifact_list.txt`.

---

## Solucion de Problemas

| Sintoma | Revisar |
|---------|---------|
| Credenciales vacias en pipelines de feature/MR | Variables marcadas como Protected. Desprotegelas. |
| No hay pipeline en el push | La rama no coincide con `workflow:rules`, o hay un MR abierto (corre el pipeline del MR). |
| Pipelines duplicados de rama y MR | Bug conocido de GitLab ([#555601](https://gitlab.com/gitlab-org/gitlab/-/issues/555601)): `$CI_OPEN_MERGE_REQUESTS` solo se evalua en `workflow:rules` si un job lo referencia. Manten la linea `echo` en `package`. |
| `No se encontraron artefactos empaquetados` | Ejecuta `veracode package --source . --output verascan --trust` localmente. Agrega un paso de build si hace falta. |
| Pipeline Scan sale con `255` | Credenciales invalidas, red bloqueada hacia `api.veracode.com`, o artefacto no soportado. |
| Pipeline Scan sale con `253` o `254` | Timeout. Agrega `--timeout <minutos>` (maximo 60). |
| `APP_NAME` vacio en `policy-scan` | `package` no escribio el reporte dotenv, o `needs:` se cambio a `artifacts: false`. |
| `UploadAndScan` rechazado | Puede haber un escaneo previo en curso para el perfil. Verifica permisos de carga del usuario de API. |

---

## Recursos

- [Parametros de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Codigos de estado de Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Script CI de SCA](https://docs.veracode.com/r/c_sc_ci_script)
- [GitLab: workflow rules](https://docs.gitlab.com/ci/yaml/workflow/)
- [GitLab: variables predefinidas](https://docs.gitlab.com/ci/variables/predefined_variables/)
