# Pipeline de Seguridad Veracode para AWS CodeBuild

Buildspec: [`buildspec.yml`](./buildspec.yml). Colocalo en la raiz de tu repositorio (nombre de buildspec por defecto de CodeBuild).

**Tecnologias soportadas**: todo lo que soporta el [autopackager de Veracode CLI](https://docs.veracode.com/r/About_auto_packaging). Usa una imagen `aws/codebuild/standard` (Ubuntu) o `amazonlinux` que soporte `java: corretto17`, y agrega los toolchains que necesite tu proyecto.

---

## Estrategia de Escaneo

| Disparador | Modo | Producto Veracode | Gate |
|------------|------|-------------------|------|
| Push a `feature/*` | `feature` | Pipeline Scan | Falla con cualquier hallazgo |
| Pull request a la rama por defecto | `pr` | Pipeline Scan | Politica `Veracode Recommended Very High` |
| Push a la rama por defecto | `policy` | Policy Scan | Politica de la plataforma |
| Cualquier otro caso | `skip` | ninguno | El build pasa sin escanear |
| Cualquier modo con escaneo | | SCA basado en agente | No bloqueante |

Codigos de salida de Pipeline Scan ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` sin hallazgos, `1-200` cantidad de hallazgos que cumplen el criterio, `253-255` timeout o error. Cualquier codigo distinto de cero falla el build.

---

## Como se Resuelve el Modo de Escaneo

El primer comando de `build` define `SCAN_MODE` a partir de las [variables de webhook de CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html):

| Origen | Logica |
|--------|--------|
| `SCAN_MODE` ya definido | Se usa tal cual (override manual) |
| `CODEBUILD_WEBHOOK_TRIGGER=pr/<n>` | `pr` si `CODEBUILD_WEBHOOK_BASE_REF` es `refs/heads/$DEFAULT_BRANCH`, si no `skip` |
| `CODEBUILD_WEBHOOK_TRIGGER=branch/<nombre>` | `policy` para la rama por defecto, `feature` para `feature/*`, si no `skip` |
| Sin variables de webhook (CodePipeline, inicio manual) | Misma logica de ramas usando `BRANCH_NAME` |

La resolucion del modo corre en la misma fase que los escaneos porque el buildspec `0.2` comparte un shell entre comandos.

---

## Configuracion

### 1. Credenciales en Secrets Manager

Crea un secreto llamado `veracode/credentials` con pares clave/valor:

```json
{
  "VERACODE_API_ID": "...",
  "VERACODE_API_KEY": "...",
  "SRCCLR_API_TOKEN": "..."
}
```

Otorga al rol de servicio de CodeBuild `secretsmanager:GetSecretValue` sobre el secreto. El buildspec lo lee con `env.secrets-manager` usando el formato `secret-id:json-key` ([referencia de buildspec](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)). Descomenta la linea `SRCCLR_API_TOKEN` cuando la clave exista; una clave faltante hace fallar el build al inicio.

Credenciales de API: [Generar credenciales de API](https://docs.veracode.com/r/t_create_api_creds). Token de SCA: [Crear un agente SCA](https://docs.veracode.com/r/t_sc_cli_agent).

### 2. Variables de entorno del proyecto (opcionales)

| Variable | Por defecto | Descripcion |
|----------|-------------|-------------|
| `VERACODE_APP_NAME` | Nombre del proyecto CodeBuild | Nombre del perfil de aplicacion |
| `DEFAULT_BRANCH` | `main` | Rama por defecto |
| `BRANCH_NAME` | ninguno | Rama cuando CodePipeline inicia el build. Pasa `#{SourceVariables.BranchName}` desde la accion de origen. |
| `SCAN_MODE` | resuelto | Fuerza `policy`, `feature`, `pr` o `skip` |

### 3. Filter groups del webhook (origenes GitHub, GitHub Enterprise Server, Bitbucket)

Configura **Primary source webhook events** con dos filter groups:

| Grupo | `EVENT` | Filtro adicional |
|-------|---------|------------------|
| 1 | `PUSH` | `HEAD_REF` = `^refs/heads/(main\|feature/.*)$` |
| 2 | `PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED, PULL_REQUEST_REOPENED` | `BASE_REF` = `^refs/heads/main$` |

El buildspec igual valida el disparador, asi que un filtro mas amplio solo cuesta minutos de build, nunca ejecuta el escaneo equivocado.

**CodePipeline**: las variables de webhook no estan disponibles cuando CodePipeline inicia CodeBuild. Define `BRANCH_NAME` en la accion de CodeBuild. CodePipeline no tiene un evento nativo de PR, asi que usa `SCAN_MODE=pr` en un pipeline dedicado si necesitas gate de PR ahi.

---

## Pasos del Build

1. **Resolver modo** y validar credenciales (solo se requieren cuando se ejecutara un escaneo).
2. **Package**: instala Veracode CLI, ejecuta `veracode package --source . --output verascan --trust`, escribe `artifact_list.txt`, falla si esta vacio.
3. **SCA**: `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"`, errores ignorados con `|| echo`. El valor de `--appname` es el mismo perfil de aplicacion al que sube el policy scan, asi los hallazgos del agente quedan en el mismo perfil. `APP_NAME` se resuelve una vez en el paso de resolucion de modo y lo reutiliza el policy scan.
4. **Pipeline Scan** (`feature`/`pr`): cada artefacto se escanea por separado, los fallos se acumulan y el build falla al final. `pr` agrega `--policy_name "$PR_GATE_POLICY"`. Los resultados van a `scan_results/`.
5. **Policy Scan** (`policy`): ultimo [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) `UploadAndScan` sobre `verascan/`, version `<rama por defecto>-<CODEBUILD_BUILD_NUMBER>`.

Para conservar los resultados de Pipeline Scan, agrega una seccion `artifacts` para `scan_results/**/*` y configura artefactos del proyecto (S3).

---

## Personalizacion

**Cambiar la politica del gate de PR**: edita `PR_GATE_POLICY` en `env.variables`.

**Relajar el gate de feature**: define `GATE_ARGS=(--fail_on_severity "Very High, High")` en la rama `else`.

**Escanear todas las ramas que no son la por defecto**: cambia `feature/*)` por `*)` en el `case` de ramas y amplia el filtro `HEAD_REF`.

**Parameter Store en lugar de Secrets Manager**: reemplaza el bloque por entradas `env.parameter-store` apuntando a parametros SecureString.

---

## Solucion de Problemas

| Sintoma | Revisar |
|---------|---------|
| El build falla al inicio con un error de Secrets Manager | Nombre del secreto o clave JSON distinta, o falta el permiso `GetSecretValue`. |
| Siempre `Modo de escaneo: skip` desde CodePipeline | `BRANCH_NAME` no se pasa a la accion de CodeBuild. |
| Build de PR muestra `skip` | El destino del PR no es `DEFAULT_BRANCH`. |
| `No se encontraron artefactos empaquetados` | La imagen de build no tiene tu toolchain. |
| Pipeline Scan sale con `255` | Credenciales invalidas, red bloqueada hacia `api.veracode.com`, o artefacto no soportado. |
| Pipeline Scan sale con `253` o `254` | Timeout del escaneo. Agrega `--timeout <minutos>` (maximo 60). |
| `UploadAndScan` rechazado | Puede haber un escaneo previo en curso para el perfil. |

---

## Recursos

- [Referencia de buildspec de CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
- [Variables de entorno de CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html)
- [Parametros de Pipeline Scan](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Script CI de SCA](https://docs.veracode.com/r/c_sc_ci_script)
