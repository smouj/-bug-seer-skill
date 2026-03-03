name: bug-seer
version: 1.2.0
description: Predictive analytics engine that identifies bug hotspots using code pattern analysis and historical defect data
maintainer: OpenClaw Team
tags:
  - predictive-analytics
  - bug-prediction
  - code-quality
  - machine-learning
  - defect-analysis
  - static-analysis
type: analysis
entrypoint: bug-seer
dependencies:
  - name: scikit-learn
    version: ">=1.3.0"
    purpose: ML models for defect prediction
  - name: pandas
    version: ">=2.0.0"
    purpose: Data manipulation and feature engineering
  - name: numpy
    version: ">=1.24.0"
    purpose: Numerical operations
  - name: tree-sitter
    version: ">=0.20.0"
    purpose: AST parsing for code structure analysis
  - name: radon
    version: ">=5.1.0"
    purpose: Cyclomatic complexity metrics
  - name: lizard
    version: ">=1.17.0"
    purpose: Code complexity and length analysis
  - name: gitpython
    version: ">=3.1.0"
    purpose: Git history mining for defect commits
system_requirements:
  python: ">=3.9"
  memory_min_mb: 512
  disk_space_mb: 100
---

# Bug Seer: Análisis Predictivo de Defectos

## Propósito

Bug Seer analiza codebases para predecir posibles ubicaciones de bugs combinando:
- Datos históricos de defectos del historial de git (commits de corrección de bugs)
- Métricas de complejidad de código (complejidad ciclomática, code churn)
- Detección de code smells (métodos largos, código duplicado, god classes)
- Patrones de actividad de desarrolladores y hotspots de propiedad
- Cambios recientes y antigüedad del código

Casos de uso reales:
- **Evaluación de riesgo pre-lanzamiento**: Identificar archivos de alto riesgo antes de etiquetar un release
- **Priorización de code review**: Marcar archivos que necesitan atención adicional en revisiones
- **Roadmap de refactoring**: Encontrar hotspots de deuda técnica para sprints de mejora
- **Asignación de equipo**: Identificar módulos con alta densidad de defectos que necesitan desarrolladores experimentados
- **Enfoque de onboarding**: Mostrar a nuevos empleados qué áreas son más frágiles

## Alcance

Bug Seer proporciona estos comandos:

### `bug-seer analyze [PATH]`
Analizar codebase para predecir hotspots de bugs
- `--model <name>`: Usar modelo entrenado (default: 'default', 'ensemble', o ruta a .pkl personalizado)
- `--threshold <float>`: Umbral de puntuación de riesgo para advertencias (0.0-1.0, default: 0.7)
- `--format <json|table|sarif>`: Formato de salida (default: table)
- `--git-depth <int>`: Número de commits de git a analizar (default: 1000)
- `--features <file>`: Archivo JSON con pesos de características personalizados
- `--exclude <pattern>`: Patrón glob para excluir rutas (se puede repetir)
- `--output <file>`: Escribir reporte a archivo en lugar de stdout

### `bug-seer train [DATASET]`
Entrenar modelo de predicción con datos históricos
- `--git-repo <path>`: Minar historial de git para defectos si no se proporciona dataset
- `--label-bug-keywords <keywords>`: Palabras clave separadas por coma en mensajes de commit (default: "fix,bug,defect,crash,error")
- `--test-split <float>`: Ratio train/test (0.0-1.0, default: 0.2)
- `--model-type <rf|xgb|logistic>`: Algoritmo (default: rf)
- `--cross-validate`: Realizar validación cruzada k-fold
- `--save <path>`: Guardar modelo entrenado en ruta
- `--feature-importance`: Mostrar puntuaciones de importancia de características

### `bug-seer validate [MODEL]`
Validar rendimiento del modelo
- `--metrics <all|precision|recall|f1|roc>`: Métricas a calcular
- `--baseline <model>`: Comparar contra modelo base
- `--confusion-matrix`: Mostrar matriz de confusión
- `--threshold-analysis`: Escanear a través de umbrales

### `bug-seer explain [FILE]`
Explicar predicción para un archivo específico
- `--model <name>`: Modelo a usar
- `--detail <high|medium|low>`: Profundidad de explicación
- `--output <json|text>`: Formato

### `bug-seer history [PATH]`
Mostrar patrones de bugs históricos para un archivo/directorio
- `--commits <int>`: Número de commits pasados (default: 50)
- `--include-blame`: Incluir git blame para correcciones recientes

### `bug-seer list-models`
Listar modelos entrenados disponibles

### `bug-seer import-defects [FILE]`
Importar dataset de defectos externo (CSV/JSON con: file_path, defect_type, severity, commit_hash)

## Proceso de Trabajo

### 1. Fase de Extracción de Características
Para cada archivo fuente en el codebase:
- Parsear AST usando tree-sitter para análisis específico de lenguaje
- Calcular métricas:
  * Complejidad ciclomática (radon)
  * Líneas de código, ratio de comentarios, cantidad de funciones
  * Número de parámetros por función
  * Porcentaje de duplicación (usando lizard)
  * Code churn (git log --oneline --numstat)
  * Frecuencia de corrección de bugs (git log --grep="fix" para ese archivo)
  * Cantidad de desarrolladores (número de autores distintos)
  * Antigüedad del código (días desde última modificación)
  * Frecuencia de modificaciones recientes (commits en últimos 30 días)
  * Concentración de propiedad (% de cambios por principal contributor)

### 2. Minería de Defectos Históricos
Bug Seer mina repositorio git para identificar commits de defectos pasados:
```bash
git log --all --grep="fix" --pretty=format:"%H" > bug_commits.txt
git log --all --grep="bug" --pretty=format:"%H" >> bug_commits.txt
```
Luego mapea cada commit de defecto a archivos afectados:
```bash
for commit in $(cat bug_commits.txt); do
  git show --name-only --pretty=format: "$commit" | tail -n+2
done > defect_files.txt
```
Cada archivo modificado por un commit de defecto recibe etiqueta positiva.

### 3. Entrenamiento del Modelo
- Cargar dataset histórico (minado o proporcionado)
- Realizar feature engineering (normalizar, PCA para dimensionalidad)
- Dividir en conjuntos train/test
- Entrenar clasificador RandomForest (default) o XGBoost
- Evaluar usando precision@k (archivos top-k rankeados)
- Guardar modelo e importancia de características en `~/.openclaw/bug-seer/models/`

### 4. Fase de Predicción
- Extraer mismas características para codebase actual
- Cargar modelo entrenado
- Puntuar cada archivo (probabilidad de contener defectos)
- Rankear archivos por puntuación
- Aplicar umbral para filtrar
- Generar reporte con puntuaciones de riesgo, factores contribuyentes y recomendaciones accionables

### 5. Generación de Reportes
Salida incluye:
- Puntuación de riesgo (0-1) para cada archivo
- Percentil de rank
- Top 3 características contribuyentes (ej: "alta complejidad", "cambios frecuentes")
- Acciones sugeridas ("refactor", "aumentar review", "añadir tests")
- Cantidad histórica de defectos para ese archivo

## Reglas de Oro

1. **No sobreajustar a patrones históricos**: Modelos entrenados en datos pasados pueden perder nuevos tipos de defectos. Validar contra hallazgos recientes.
2. **Requerir historial suficiente**: Archivos con < 6 meses de historia producen predicciones poco fiables. Marcar como "datos insuficientes".
3. **Umbrales específicos por lenguaje**: Los umbrales de complejidad difieren por lenguaje (Python vs C++). Usar defaults calibrados por lenguaje.
4. **Presupuesto de falsos positivos**: Limitar advertencias al top 20% de archivos para evitar fatiga de alertas.
5. **Siempre citar evidencia**: Cada flag de alto riesgo debe incluir métrica específica (ej: "complejidad ciclomática=42 (>35 umbral)").
6. **Preservar contexto git**: Las predicciones deben ser reproducibles desde el commit SHA exacto. Guardar versión de modelo con hash de commit.
7. **Nunca modificar código**: Bug Seer es análisis solo lectura. No hay correcciones automáticas.
8. **Ponderar datos recientes más alto**: Ponderar defectos de últimos 90 días más pesado en entrenamiento (factor 2x).
9. **Contabilizar tamaño de archivo**: Archivos grandes naturalmente tienen más defectos. Normalizar por líneas de código.
10. **Incluir tanto probabilidad como impacto**: Un archivo con alta complejidad y alto churn obtiene mayor riesgo que uno con solo un factor.

## Ejemplos

### Análisis rápido de proyecto actual
```bash
bug-seer analyze src/
```
Salida (formato tabla):
```
File                              Risk  Rank  Top Factors                              Historical Defects
src/game/combat.py                0.82  1%    Complex (CC=45), Frequent changes      12
src/engine/renderer.cpp           0.78  3%    High churn, Many authors               8
src/utils/helpers.py              0.41  45%   Medium complexity                      3
tests/test_core.py                0.12  89%   Low complexity                         0
```

### Entrenar modelo personalizado para estándares de equipo
```bash
bug-seer train --git-repo . --model-type xgb --save models/team-v2.pkl --feature-importance
```
Salida:
```
Training on 847 files (152 defect samples)
Features: ['cc', 'loc', 'churn', 'dev_count', 'bug_fixes', 'duplication', 'age_days']
Best CV F1: 0.73 (threshold=0.6)
Feature importance:
  1. churn (0.31)
  2. cc (0.24)
  3. bug_fixes (0.18)
  4. dev_count (0.12)
Model saved to models/team-v2.pkl
```

### Integración CI/CD con salida SARIF
```bash
bug-seer analyze . --format sarif --threshold 0.75 --output warnings.sarif
```
Snippet SARIF:
```json
{
  "level": "error",
  "message": {"text": "High bug risk (0.82). Contributing: Cyclomatic complexity 45 (>35), 5 authors, 47 changes in 30 days"},
  "locations": [{"physicalLocation": {"artifactLocation": {"uri": "src/game/combat.py"}}}]
}
```

### Explicar predicción de archivo específico
```bash
bug-seer explain src/engine/renderer.cpp --model ensemble --detail high
```
Salida:
```
File: src/engine/renderer.cpp
Predicted risk: 0.78 (High)

Contributing factors (weighted):
  × Code churn: 47 commits in 30 days (normalized score: 0.42)
  × Cyclomatic complexity: 38 (threshold: 25) (score: 0.31)
  × Developer count: 7 distinct authors (score: 0.18)
  - Lines of code: 1250 (acceptable)
  - Bug history: 8 defects (normalized: 0.15)

Recommendation: Schedule dedicated code review session and consider extracting shader management subsystem.
```

### Importar datos de seguimiento de defectos externo
```bash
bug-seer import-defects jira-export.csv
```
Formato CSV:
```csv
file_path,defect_type,severity,commit_hash
src/auth/login.py,security,critical,abc123
src/api/user.py,logic,major,def456
```

### Analizar con pesos de características personalizados
```bash
cat > custom_weights.json <<EOF
{
  "cyclomatic_complexity": 2.5,
  "code_churn": 1.8,
  "developer_count": 1.2
}
EOF
bug-seer analyze . --features custom_weights.json
```

### Validar modelo antes de uso en producción
```bash
bug-seer validate models/team-v2.pkl --metrics precision,recall,f1 --confusion-matrix
```
Salida:
```
Model: models/team-v2.pkl (trained 2026-01-15)

Confusion Matrix (threshold=0.6):
          Predicted
          Yes   No
Actual Yes  42   8
       No   15  782

Precision: 0.74 (42/57)
Recall:    0.84 (42/50)
F1:        0.79
```

## Dependencias

### Paquetes Python
Instalar via pip:
```bash
pip install scikit-learn pandas numpy tree-sitter radon lizard gitpython
```

Para parsers específicos de lenguaje:
```bash
# Soporte Python
pip install tree-sitter-python

# Soporte C/C++
pip install tree-sitter-c

# JavaScript/TypeScript
pip install tree-sitter-javascript tree-sitter-typescript
```

### Dependencias del Sistema
- `git`: Disponible en PATH para minería de historial
- `python3.9+`: Con pip
- 500MB RAM mínimo para codebases medianas

### Requisitos de Datos
- Repositorio git con al menos 100 commits para entrenamiento confiable
- Mínimo 30 archivos etiquetados con defectos para modelo significativo
- 6 meses de historial recomendado

## Variables de Entorno

- `BUG_SEER_MODEL_DIR`: Sobrescribir ubicación de almacenamiento de modelos por defecto (~/.openclaw/bug-seer/models)
- `BUG_SEER_CACHE`: Habilitar/deshabilitar caché (0=deshabilitado, 1=habilitado, default: 1)
- `BUG_SEER_JOBS`: Número de procesos de extracción de características en paralelo (default: CPU count)
- `BUG_SEER_GIT_TIMEOUT`: Segundos para operaciones git (default: 30)
- `OPENCLAW_DEBUG`: Establecer a 1 para logging verbose

Ejemplo:
```bash
export BUG_SEER_MODEL_DIR=/custom/models
export BUG_SEER_JOBS=4
bug-seer analyze .
```

## Pasos de Verificación

Después de instalar Bug Seer, verificar operación correcta:

1. **Verificar instalación**:
```bash
bug-seer --version
# Debería imprimir: bug-seer 1.2.0
```

2. **Verificar dependencias**:
```bash
bug-seer check-deps
# Salida: All dependencies satisfied (tree-sitter, radon, lizard, git)
```

3. **Probar en codebase pequeño**:
```bash
cd /path/to/small/project
bug-seer analyze . --format json --output test-report.json
# Debería producir JSON sin errores
cat test-report.json | python3 -m json.tool  # Validar JSON
```

4. **Validar entrenamiento de modelo** (requiere historial git):
```bash
bug-seer train --git-repo . --test-split 0.3 --cross-validate
# Debería reportar CV F1 score > 0.6 con modelo default
```

5. **Verificar persistencia de modelo**:
```bash
bug-seer list-models
# Debería mostrar al menos 'default' si el entrenamiento tuvo éxito
```

6. **Prueba de integración CI**:
```bash
bug-seer analyze . --format sarif --threshold 0.8 > sarif-output.txt
# Debería salir con código 0 incluso sin hallazgos (sin hallazgos = SARIF vacío válido)
```

## Solución de Problemas

### "No git repository found"
**Causa**: Ejecutando en directorio sin historial git.
**Fix**: cd a repo git o especificar ruta: `bug-seer analyze /path/to/repo`

### "Insufficient defect samples for training"
**Causa**: Menos de 20 archivos de defecto identificados en historial git.
**Fix**: Aumentar `--git-depth` o ajustar `--label-bug-keywords` para capturar más commits de defecto. Etiquetar defectos manualmente via `import-defects`.

### "Tree-sitter language not supported"
**Causa**: Parser de lenguaje faltante para extensión de archivo.
**Fix**: Instalar parser: `pip install tree-sitter-<language>`. Verificar soporte: `bug-seer languages`

### "Memory exhausted during analysis"
**Causa**: Codebase muy grande sin límites paralelos.
**Fix**: Establecer `BUG_SEER_JOBS=2` o usar `--exclude` para omitir directorios. Considerar analizar en lotes.

### "Model predictions are all 0.0"
**Causa**: Modelo no entrenado o mismatch de características.
**Fix**: Entrenar modelo primero: `bug-seer train --git-repo .`. Asegurar que ruta de modelo existe y coincide con lenguaje de codebase.

### "Cyclomatic complexity parsing failed"
**Causa**: radon no puede parsear cierta sintaxis.
**Fix**: Actualizar radon: `pip install --upgrade radon`. Verificar errores de sintaxis en archivos problemáticos manualmente.

### "Feature importance shows negative values"
**Causa**: PCA o normalización produjo pesos no intuitivos.
**Fix**: Deshabilitar escalado de características: usar `--no-normalize` durante train (no recomendado) o inspeccionar métricas raw.

### "Analysis stuck on one file"
**Causa**: Archivo único masivo o timeout de parser.
**Fix**: Aumentar `--git-timeout` o excluir archivo: `--exclude "**/generated/**"`

## Comandos de Rollback

Bug Seer es análisis-only y no hace modificaciones. Rollback se refiere a revertir cambios de modelo o estado de análisis.

### Eliminar modelo específico
```bash
rm ~/.openclaw/bug-seer/models/<model-name>.pkl
rm ~/.openclaw/bug-seer/models/<model-name>.json  # metadata
```

### Restaurar modelo por defecto
```bash
bug-seer delete-model team-custom
# Siguiente análisis retrocederá a 'default'
```

### Limpiar caché de análisis
```bash
rm -rf ~/.openclaw/bug-seer/cache/*
# Siguiente análisis recomputará todas las características
```

### Revertir a versión anterior de modelo (si se mantuvo)
```bash
# Los modelos tienen snapshots con timestamps
ls ~/.openclaw/bug-seer/models/backups/
bug-seer use-model models/backups/team-v1-20260101.pkl
```

### Deshacer importación de dataset
```bash
# Manual: editar base de datos de defectos si se conocen hashes de commit
# O reentrenar sin los datos importados:
bug-seer train --git-repo . --exclude-defects jira-export.csv
```

### Deshabilitar Bug Seer en CI/CD
Remover el comando `bug-seer analyze` de tu script CI. No hay estado persistente.

## Avanzado: Detalles de Feature Engineering

Bug Seer calcula estas características por archivo:

| Feature | Cálculo | Peso (default) | Racional |
|---------|---------|----------------|----------|
| `cc` | complejidad ciclomática promedio (radon) | 1.0 | Alta complejidad → más difícil de entender/probar |
| `loc` | líneas de código | -0.3 (neg) | Mayor superficie pero también más probada |
| `churn` | commits en últimos 30 días | 1.5 | Churn indica inestabilidad/trabajo en curso |
| `dev_count` | autores distintos (git) | 1.2 | Muchos autores → problemas de coordinación |
| `bug_fixes` | número de commits de defecto que tocan archivo | 2.0 | Bugs pasados predicen bugs futuros |
| `duplication` | % líneas duplicadas (lizard) | 0.8 | Copy-paste transmite defectos |
| `age_days` | días desde creación | -0.4 (neg) | Código más viejo generalmente más estable |
| `params_avg` | parámetros promedio de función | 0.5 | Listas largas de parámetros son propensas a error |
| `comment_ratio` | líneas de comentario / líneas totales | -0.2 (neg) | Código bien documentado tiene menos bugs |
| `test_coverage` | si existe tests/ y se encuentran archivos de test | -1.0 (neg) | Código probado es más confiable |

Los pesos de características pueden ser sobrescritos via archivo JSON `--features`.

## Ejemplos de Integración

### Pre-commit hook (advertir en archivos riesgosos)
```bash
#!/bin/bash
# .git/hooks/pre-commit
CHANGED=$(git diff --name-only --cached)
echo "$CHANGED" | xargs bug-seer explain --detail low --threshold 0.75
# Salir 0 para permitir commit; non-zero bloquearía (no recomendado)
```

### GitHub Actions (comentar en PR)
```yaml
- name: Bug Seer Analysis
  run: |
    bug-seer analyze . --format sarif --threshold 0.8 > bugs.sarif
    # Subir como resultados de escaneo de seguridad
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: bugs.sarif
```

### Extensión VS Code (hipotético)
La extensión ejecuta `bug-seer explain` en archivo actual y muestra badge de riesgo inline.

## Características de Rendimiento

- **Codebase pequeño** (< 100 archivos): ~2-5 segundos
- **Codebase mediano** (100-1000 archivos): ~30-60 segundos
- **Codebase grande** (> 1000 archivos): ~2-5 minutos con extracción de características paralela
- Tiempo de entrenamiento: 10-30 segundos después de extracción de características
- Cache hit: Ejecuciones posteriores en archivos sin cambios casi instantáneas

## Limitaciones

- **Soporte de lenguajes**: Completamente soportados: Python, C/C++, Java, JavaScript/TypeScript, Go. Parcial: Ruby, PHP, Swift (solo AST, sin métricas de complejidad).
- **Detección de monorepo**: Puede contar doble si mismo archivo aparece bajo múltiples roots. Usar `--exclude` para evitar.
- **Archivos binarios**: Saltados automáticamente (sin parser).
- **Código generado**: Debe excluirse via patrones glob (ej: `--exclude "**/dist/**"`).
- **VCS no-git**: No soportado. Requiere historial git.
- **Tiempo real**: No suitable para sugerencias en tiempo real; apto para análisis periódico.

## Mejoras Futuras

- Inclusión de análisis de texto de comentarios de code review
- Predicción de defectos embodied (usando code embeddings de LLMs)
- Transfer learning cross-proyecto
- Predicción series-temporales de tasas de introducción de defectos
- Integración con issue tracker (Jira, GitHub Issues) para etiquetas más ricas
```