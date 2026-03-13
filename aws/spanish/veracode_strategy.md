# Estrategia de Seguridad con Veracode

Estrategia de seguridad automatizada que integra múltiples productos Veracode en el SDLC, balanceando velocidad de feedback con profundidad de análisis según el contexto de desarrollo.

**Tecnologías Soportadas**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, y más.

---

## Estrategia de Escaneo

| Contexto | Producto Veracode | Tiempo | Gate | Propósito |
|----------|-------------------|--------|------|-----------|
| Feature branches | Pipeline Scan | 3-10 min | No | Feedback rápido para desarrolladores |
| Pull Requests | Pipeline Scan | 3-10 min | Sí (Very High/High) | Prevenir vulnerabilidades antes de merge |
| Release branches | Sandbox Scan | 30-90 min | No | Validación pre-producción aislada |
| Branch main | Policy Scan | 25-60 min | Opcional | Certificación para producción |

**Todos los contextos**: SCA Agent-Based (análisis de dependencias)

---

## Productos Veracode Utilizados

### 1. Veracode CLI - Autopackager

- Detecta automáticamente el tipo de aplicación y tecnología
- Compila y empaqueta el código para análisis
- Genera artefactos listos para escaneo (JAR, WAR, ZIP, DLL, etc.)

**Por qué**:
- Elimina configuración manual por tecnología
- Garantiza empaquetado consistente

**Comando**: `veracode package --source . --output verascan --trust`

---

### 2. Pipeline Scan

- Análisis SAST rápido e incremental
- Detecta vulnerabilidades críticas en minutos
- Genera reportes JSON con línea de código exacta

**Por qué en feature branches**:
- Feedback inmediato durante desarrollo activo
- No bloquea el flujo de trabajo
- Permite iteración rápida sobre código seguro

**Por qué en Pull Requests (con gate)**:
- Previene vulnerabilidades antes de merge
- Bloquea código inseguro automáticamente
- Integra seguridad en code review

**Comando básico**:
```bash
java -jar pipeline-scan.jar \
  -f artefacto.jar \
  -vid $VERACODE_API_ID \
  -vkey $VERACODE_API_KEY
```

**Comando con gate**:
```bash
java -jar pipeline-scan.jar \
  -f artefactos.zip \
  --fail_on_severity "Very High, High" \
  --policy "Veracode Recommended Very High"
```

---

### 3. Sandbox Scan

- Análisis SAST completo en ambiente aislado
- Comparación con escaneos anteriores
- Reportes detallados sin afectar la politica perfil de producción

**Por qué en release branches**:
- Validación exhaustiva pre-producción
- Testing de fixes sin impactar métricas oficiales
- Permite experimentación segura con nuevas features

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

- Análisis SAST exhaustivo contra políticas corporativas
- Genera certificación oficial de seguridad
- Historial completo para compliance y auditoría

**Por qué en main branch**:
- Certificación formal para código en producción
- Cumplimiento de políticas de seguridad organizacionales
- Trazabilidad completa para regulaciones (SOC2, PCI-DSS, etc.)

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

- Análisis de composición de software (dependencias third-party)
- Detección de vulnerabilidades conocidas (CVEs)
- Identificación de librerías desactualizadas o con licencias problemáticas

**Por qué en todos los escaneos**:
- El 80% de vulnerabilidades están en dependencias
- Detecta riesgos de supply chain
- Complementa SAST con análisis de third-party code

**Uso**:
```bash
curl -sSL https://download.sourceclear.com/ci.sh | bash -s -- scan --recursive --update-advisor
```

---

## Flujo de Seguridad

```
Commit → Autopackager → SCA + SAST (según branch) → Resultados
```

**Fase 1: Empaquetado** (Veracode CLI)
- Detecta tecnología automáticamente
- Compila y empaqueta código

**Fase 2: Análisis de Composición** (SCA)
- Escanea dependencias third-party
- Reporta CVEs conocidos

**Fase 3: Análisis Estático** (SAST)
- Pipeline Scan: Feedback rápido (dev/PR)
- Sandbox Scan: Validación completa (release)
- Policy Scan: Certificación oficial (producción)

---

## Criterios de Selección de Scan

**Pipeline Scan cuando**:
- Necesitas feedback en < 10 minutos
- Estás en ciclo activo de desarrollo
- Quieres validar fixes rápidamente

**Sandbox Scan cuando**:
- Preparas un release candidate
- Necesitas análisis completo sin afectar métricas
- Quieres comparar con versiones anteriores

**Policy Scan cuando**:
- Código va a producción
- Necesitas certificación oficial
- Requieres compliance para auditorías

**SCA siempre** porque las dependencias son el vector de ataque más común.

---

## Variables de Configuración

El pipeline requiere estas variables de entorno:

| Variable | Propósito | Ejemplo |
|----------|-----------|---------|
| `VERACODE_API_ID` | Autenticación API | (desde Veracode Platform) |
| `VERACODE_API_KEY` | Autenticación API | (desde Veracode Platform) |
| `APP_NAME` | Nombre en Veracode Platform | "MiAplicacion" |
| `SANDBOX_NAME` | Nombre del sandbox | "release-candidate" |
| `SCAN_TYPE` | Forzar tipo específico | "pipeline", "sandbox", "policy" |
| `BRANCH_NAME` | Detección automática | Inyectado por CI/CD |

**Detección automática**: Si no se especifica `SCAN_TYPE`, el pipeline detecta el tipo de scan por el nombre del branch.

---

## Resultados y Reportes

**Pipeline Scan genera**:
- `results.json`: Todos los hallazgos
- `filtered_results.json`: Hallazgos según policy
- Código de salida: 0 (pass) o 1 (fail)

**Policy/Sandbox Scan genera**:
- Reportes en Veracode Platform
- PDF descargable
- Dashboard con métricas y tendencias
- APIs para integración con SIEM/tickets

**SCA genera**:
- Listado de dependencias vulnerables
- Scores de riesgo por librería
- Recomendaciones de actualización
- Análisis de licencias

---

## Mejores Prácticas

**Shift-Left**:
- Ejecutar Pipeline Scan en cada commit
- Habilitar gates en PRs para bloquear vulnerabilidades High/Very High
- Educar al equipo sobre findings comunes

**Compliance**:
- Policy Scan obligatorio antes de producción
- Manténer historial de todos los scans
- Documentar excepciones y mitigaciones

**Optimización**:
- Usar Pipeline Scan para iteración rápida
- Reservar Policy Scan para releases oficiales
- Cachear dependencias para acelerar builds

**SCA**:
- Monitorer CVEs nuevos continuamente
- Actualizar dependencias regularmente
- Revisar licencias antes de adoptar librerías

---

## Soporte

- **Documentación Veracode**: [docs.veracode.com](https://docs.veracode.com)
- **Veracode CLI**: [https://docs.veracode.com/r/Install_the_Veracode_CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- **Pipeline Scan**: [https://docs.veracode.com/r/Pipeline_Scan](https://docs.veracode.com/r/Pipeline_Scan)
- **SCA**: [https://docs.veracode.com/r/Agent_Based_Scans](https://docs.veracode.com/r/Agent_Based_Scans)
