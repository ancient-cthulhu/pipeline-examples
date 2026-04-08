# Estrategia de Seguridad con Veracode para Azure Pipelines

Estrategia de seguridad automatizada que integra multiples productos Veracode en el SDLC mediante Azure Pipelines, balanceando velocidad de feedback con profundidad de analisis segun el contexto de desarrollo.

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

## Arquitectura del Pipeline

El pipeline es un unico archivo `azure-pipelines.yml` con stages condicionales. La deteccion de branch se maneja nativamente via `Build.SourceBranch` y `Build.Reason`, sin necesidad de overrides manuales.

```
Stage 1: Package        (siempre)  - Veracode CLI autopackager
Stage 2: SCA            (siempre)  - Agent-Based SCA
Stage 3a: Pipeline Scan (feature)  - SAST rapido, sin gate
Stage 3b: Pipeline Scan (PR)       - SAST rapido, gate en Very High/High
Stage 4: Sandbox Scan   (release)  - SAST completo en sandbox
Stage 5: Policy Scan    (main)     - SAST completo, certificacion produccion
```

Solo un stage de SAST se ejecuta por build dependiendo del contexto del branch. SCA siempre se ejecuta en paralelo.

---

## Productos Veracode Utilizados

### 1. Veracode CLI - Autopackager

Detecta automaticamente el tipo de aplicacion y tecnologia, compila y empaqueta el codigo para analisis, genera artefactos listos para escaneo (JAR, WAR, ZIP, DLL, etc.).

**Por que**: Elimina configuracion manual por tecnologia y garantiza empaquetado consistente.

**Comando**: `veracode package --source . --output verascan --trust`

---

### 2. Pipeline Scan

Analisis SAST rapido e incremental. Detecta vulnerabilidades criticas en minutos y genera reportes JSON con linea de codigo exacta.

**Por que en feature branches**: Feedback inmediato durante desarrollo activo sin bloquear el flujo de trabajo.

**Por que en Pull Requests (con gate)**: Previene vulnerabilidades antes de merge. Bloquea codigo inseguro automaticamente e integra seguridad en code review.

**Comando basico**:
```bash
java -jar pipeline-scan.jar \
  -f artefacto.jar \
  -vid $VERACODE_API_ID \
  -vkey $VERACODE_API_KEY
```

**Con gate**:
```bash
java -jar pipeline-scan.jar \
  -f artefactos.zip \
  --fail_on_severity "Very High, High" \
  --policy "Veracode Recommended Very High"
```

---

### 3. Sandbox Scan

Analisis SAST completo en ambiente aislado. Comparacion con escaneos anteriores y reportes detallados sin afectar la politica/perfil de produccion.

**Por que en release branches**: Validacion exhaustiva pre-produccion, testing de fixes sin impactar metricas oficiales.

**Uso**:
```bash
java -jar vosp-api-wrapper-java.jar \
  -action UploadAndScan \
  -sandboxname "release-candidate" \
  -createsandbox true \
  -filepath artefacto.zip
```

---

### 4. Policy Scan

Analisis SAST exhaustivo contra politicas corporativas. Genera certificacion oficial de seguridad con historial completo para compliance y auditoria.

**Por que en main branch**: Certificacion formal para codigo en produccion, cumplimiento de politicas organizacionales, trazabilidad completa para regulaciones (SOC2, PCI-DSS, etc.).

**Uso**:
```bash
java -jar vosp-api-wrapper-java.jar \
  -action UploadAndScan \
  -appname "ProductionApp" \
  -autoscan true \
  -filepath artefacto.jar \
  -version "v1.2.3"
```

---

### 5. SCA Agent-Based

Analisis de composicion de software para dependencias third-party. Detecta CVEs conocidos, librerias desactualizadas y licencias problematicas.

**Por que en todos los escaneos**: ~80% de vulnerabilidades estan en dependencias. Detecta riesgos de supply chain y complementa SAST con analisis de third-party code.

**Uso**:
```bash
curl -sSL https://download.sourceclear.com/ci.sh | bash -s -- scan --recursive --update-advisor
```

---

## Flujo de Seguridad

```
Commit -> Autopackager -> SCA + SAST (segun branch) -> Resultados
```

**Fase 1: Empaquetado** (Veracode CLI) - Detecta tecnologia automaticamente, compila y empaqueta codigo.

**Fase 2: Analisis de Composicion** (SCA) - Escanea dependencias third-party, reporta CVEs conocidos.

**Fase 3: Analisis Estatico** (SAST) - Pipeline Scan para dev/PR, Sandbox Scan para release, Policy Scan para produccion.

---

## Configuracion en Azure Pipelines

### Configuracion de Variable Group

Crear un variable group llamado `veracode-credentials` en Azure DevOps:

1. Ir a **Pipelines > Library > + Variable group**
2. Nombre: `veracode-credentials`
3. Agregar variables:

| Variable | Tipo | Origen |
|----------|------|--------|
| `VERACODE_API_ID` | Secreto | Veracode Platform |
| `VERACODE_API_KEY` | Secreto | Veracode Platform |
| `SRCCLR_API_TOKEN` | Secreto | Veracode Platform (opcional) |

**Recomendado**: Vincular a Azure Key Vault en lugar de almacenar secretos directamente.

### Variables del Pipeline

| Variable | Proposito | Ejemplo |
|----------|-----------|---------|
| `APP_NAME` | Nombre en Veracode Platform | "MiAplicacion" |
| `SANDBOX_NAME` | Identificador del sandbox | "release-candidate" |

### Deteccion de Branch

El pipeline usa variables nativas de Azure Pipelines para seleccion automatica de escaneo:

| Variable | Uso |
|----------|-----|
| `Build.SourceBranch` | Ref completa del branch (ej: `refs/heads/feature/xyz`) |
| `Build.Reason` | Razon del trigger (`PullRequest`, `IndividualCI`, etc.) |
| `Build.BuildNumber` | Usado como identificador de version en Veracode |

### Manejo de Variables Secretas

Las variables secretas de variable groups NO se expanden como macro dentro de bloques `script:`. El pipeline usa mappings `env:` para inyectar secretos como variables de entorno, y luego los referencia como `$VERACODE_API_ID` (variable de shell) en el cuerpo del script. Variables no secretas como `$(APP_NAME)` y `$(PACKAGED_FILE)` se expanden normalmente como macro.

### Disparadores de Pull Request (Azure Repos Git)

Cuando se utiliza **Azure Repos Git**, las builds de Pull Request **no se activan únicamente con el bloque `pr:` en el YAML**.

Para garantizar que se ejecute correctamente el **Pipeline Scan para PR (con gate)**, es necesario configurar una **Política de Rama (Branch Policy) con Build Validation** en la rama destino (por ejemplo, `main`):

1. Ir a **Repos → Branches**
2. Seleccionar la rama destino (`main`)
3. Abrir **Branch policies**
4. En **Build validation**, agregar este pipeline
5. (Opcional) Marcarlo como **Required** para bloquear la finalización del PR si fallan los controles de seguridad

Una vez que la política de rama dispara el pipeline, la lógica del YAML utiliza  
`Build.Reason = PullRequest` para ejecutar automáticamente la etapa de **Pipeline Scan con gate para PR**.

---

## Resultados y Reportes

**Pipeline Scan genera**:
- `results.json`: Todos los hallazgos
- `filtered_results.json`: Hallazgos segun policy
- Codigo de salida: 0 (pass) o 1 (fail)
- Publicado como Azure Pipeline Artifact

**Policy/Sandbox Scan genera**:
- Reportes en Veracode Platform
- PDF descargable
- Dashboard con metricas y tendencias
- APIs para integracion con SIEM/tickets

**SCA genera**:
- Listado de dependencias vulnerables
- Scores de riesgo por libreria
- Recomendaciones de actualizacion
- Analisis de licencias

---

## Mejores Practicas

**Shift-Left**: Ejecutar Pipeline Scan en cada commit. Habilitar gates en PRs para bloquear vulnerabilidades High/Very High. Educar al equipo sobre findings comunes.

**Compliance**: Policy Scan obligatorio antes de produccion. Mantener historial de todos los scans. Documentar excepciones y mitigaciones.

**Optimizacion**: Usar Pipeline Scan para iteracion rapida. Reservar Policy Scan para releases oficiales. Cachear dependencias para acelerar builds.

**SCA**: Monitorear CVEs nuevos continuamente. Actualizar dependencias regularmente. Revisar licencias antes de adoptar librerias.

**Especifico de Azure**: Usar variable groups vinculados a Key Vault para rotacion de credenciales. Agregar branch policies para requerir que Pipeline Scan pase antes de completar PR. Usar environments con approval gates para el stage de Policy Scan en produccion.

---

## Soporte

- **Documentacion Veracode**: [docs.veracode.com](https://docs.veracode.com)
- **Veracode CLI**: [https://docs.veracode.com/r/Install_the_Veracode_CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- **Pipeline Scan**: [https://docs.veracode.com/r/Pipeline_Scan](https://docs.veracode.com/r/Pipeline_Scan)
- **SCA**: [https://docs.veracode.com/r/Agent_Based_Scans](https://docs.veracode.com/r/Agent_Based_Scans)
