# TP17 – Secret Scanning con Gitleaks

Se incorpora Gitleaks al pipeline CI/CD para detectar posibles credenciales y secretos expuestos en el repositorio.

Se implementan dos jobs independientes:

| Job en Pipeline | Tipo de evaluación | Exit Code | Acción ante hallazgos | Evidencia generada |
|---|---|---|---|---|
| `gitleaks-andon-cord` | Secret scanning en historial Git | 1 | Bloquea el pipeline | Resumen en `$GITHUB_STEP_SUMMARY` |
| `gitleaks-audit-report` | Reporte de inspección JSON | 0 | Informativo: genera evidencia | Artifact JSON descargable |

También se incorpora un hook local `pre-commit` para prevenir commits que contengan secretos.

---

## TP16 — Escaneo de Seguridad de Contenedores, Dependencias e IaC con Trivy (Pipeline 3 Fases)

Integración de **Trivy** en la fábrica de software (CI/CD) aplicando la arquitectura de **3 Fases con Paso de Artefactos** y la resolución de falsos positivos mediante el patrón **Render First (TP10B)**.

### Matriz de Control e Integración (Formato CSV)
```csv
Fase / Job;Dominio Evaluado;Severidades;Exit Code;Acción ante Hallazgos;Evidencia
Fase 1: build-and-package;Compilación Docker;-;0;Exporta app-image.tar como artefacto;Artefacto en GitHub Actions
Fase 2A: trivy-andon-cord;SCA, Contenedor e IaC Renderizado (TP10B);HIGH, CRITICAL;1;Andon Cord: Detiene el pipeline y bloquea el despliegue;Resumen en $GITHUB_STEP_SUMMARY
Fase 2B: trivy-audit-report;Contenedores y Dependencias;LOW, MEDIUM;0;Informativo: Genera reporte de inspección;Artifact descargable + $GITHUB_STEP_SUMMARY
Fase 3: deploy-k8s-helm;Publicación y Despliegue;-;0;Promueve la imagen aprobada a Docker Hub y Helm;Release en Kubernetes
Comandos de Auditoría Local
# 1. Auditoría SCA
trivy fs ./backend

# 2. Auditoría de Imagen
trivy image devops-portfolio:latest

# 3. Renderizado previo e IaC Scanning (TP10B)
helm template mi-app ./devops-portfolio -f devops-portfolio/values-prod.yaml > manifests-rendered-prod.yaml
trivy config manifests-rendered-prod.yaml


### Script de Verificación Local (`scripts/verificar-trivy.sh`)

```bash
mkdir -p scripts
cat > scripts/verificar-trivy.sh << 'EOF'
#!/bin/bash
set -uo pipefail

echo "================================================="
echo "  VERIFICACIÓN TÉCNICA - TP16 TRIVY SECURITY (v4)"
echo "================================================="
echo ""

# 1. Verificar instalación local o disponibilidad de Docker
if command -v trivy &> /dev/null; then
  echo "  [OK] Trivy instalado localmente ($(trivy --version | head -n 1))"
else
  echo "  [WARN] Trivy no detectado localmente en PATH (Se verificará ejecución mediante CI/CD)"
fi

# 2. Verificar configuración en el pipeline
WORKFLOW_FILE=".github/workflows/cicd.yml"
if [ -f "$WORKFLOW_FILE" ]; then
  if grep -q "trivy-andon-cord" "$WORKFLOW_FILE" && grep -q "manifests-rendered-prod.yaml" "$WORKFLOW_FILE"; then
    echo "  [OK] Workflow configurado con 3 fases y renderizado de Helm TP10B"
  else
    echo "  [FAIL] Falta la configuración de 3 fases o TP10B en $WORKFLOW_FILE"
    exit 1
  fi
else
  echo "  [FAIL] No se encontró el archivo $WORKFLOW_FILE"
  exit 1
fi

echo ""
echo "=== Verificación completada con éxito ==="

chmod +x scripts/verificar-trivy.sh
bash scripts/verificar-trivy.sh

# Subir la actualización al repositorio de GitHub
git add .
git commit -m "TP16 v4: Integración de Trivy con 3 fases, precondiciones TP10B/TP12C y nota pedagógica sobre refactorización continua"
git push origin main
