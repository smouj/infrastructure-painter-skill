---
title: Infrastructure Painter
version: 1.2.4
description: Genera automáticamente diagramas visuales de infraestructura en la nube a partir de definiciones de IaC
author: Infrastructure Painter Team
maintainer: devops@example.com
license: MIT
tags:
  - infrastructure
  - diagrams
  - cloud
  - visualization
  - architect
  - terraform
  - cloudformation
  - aws
  - azure
  - gcp
  - kubernetes
os:
  - linux
  - macos
  - windows
requires:
  - terraform >= 1.0
  - graphviz >= 2.40
  - python >= 3.8
  - jq
  - aws-cli (optional)
  - azure-cli (optional)
  - gcloud (optional)
---

# Infrastructure Painter

Genera automáticamente diagramas visuales de infraestructura en la nube a partir de definiciones de Infrastructure as Code (IaC). Soporta Terraform, AWS CloudFormation, plantillas de Azure Resource Manager y manifiestos de Kubernetes. Produce diagramas de arquitectura profesionales en múltiples formatos (PNG, SVG, PDF, Mermaid, PlantUML).

## Purpose

Infrastructure Painter resuelve estos problemas específicos:

- **Cloud Architecture Visualization**: Transforma módulos complejos de Terraform en diagramas de arquitectura limpios que muestran VPCs, subredes, grupos de seguridad, instancias EC2, clústeres RDS, balanceadores de carga y sus relaciones
- **Documentation Generation**: Genera automáticamente diagramas de infraestructura actualizados como parte de pipelines CI/CD, garantizando que la documentación siempre coincida con la infraestructura desplegada
- **Impact Analysis**: Antes de ejecutar `terraform apply`, genera un diagrama de vista previa de los cambios planificados para entender el radio de explosión
- **Multi-cloud Mapping**: Visualiza implementaciones híbridas en AWS, Azure y GCP en diagramas unificados
- **Kubernetes Topology**: Mapea clústeres de K8s mostrando namespaces, deployments, services, ingress controllers y network policies
- **Security Review**: Destaca límites de seguridad, roles IAM y flujo de datos para auditorías de cumplimiento
- **Onboarding**: Nuevos ingenieros pueden visualizar toda la pila de infraestructura en lugar de leer cientos de líneas de HCL

Casos de uso reales:
- Generar diagramas de arquitectura para reuniones de revisión de arquitectura trimestrales
- Crear diagramas de topología de red que muestren VPC peering y conexiones de Transit Gateway
- Visualizar patrones de comunicación de microservicios en un service mesh
- Documentar configuraciones de recuperación de desastres mostrando replicación multi-región
- Mapear límites de cumplimiento PCI mostrando segmentación del Cardholder Data Environment (CDE)

## Alcance

Infrastructure Painter proporciona estos comandos concretos:

### Main Commands

1. **`infrastructure-painter generate`** - Generación de diagramas central
   - `--source, -s <path>`: Ruta al directorio Terraform, plantilla CloudFormation o manifiesto K8s
   - `--output, -o <file>`: Ruta del archivo de salida (auto-detecta formato por extensión)
   - `--format, -f <format>`: Forzar formato de salida (png, svg, pdf, mermaid, plantuml, dot)
   - `--layout, -l <type>`: Algoritmo de layout (hierarchical, circular, orthogonal, fruchterman)
   - `--theme, -t <theme>`: Tema de color (aws, azure, gcp, minimal, dark, corporate)
   - `--exclude, -e <pattern>`: Excluir recursos que coincidan con patrón (ej. "aws_iam.*", "kubernetes_*")
   - `--include, -i <pattern>`: Solo incluir recursos que coincidan con patrón
   - `--group-by, -g <field>`: Agrupar recursos por tag, tipo o environment (ej. "tag:Environment", "type")
   - `--show-labels, --no-show-labels`: Alternar etiquetas de recursos (default: true)
   - `--show-connections, --no-show-connections`: Alternar líneas de conexión (default: true)
   - `--max-depth, -d <n>`: Profundidad máxima de jerarquía a renderizar (default: ilimitado)
   - `--state-file, -state <path>`: Usar archivo de estado Terraform específico para datos de recursos en vivo
   - `--plan-file, -plan <path>`: Usar JSON de plan Terraform para mostrar cambios planificados
   - `--plan-mode <mode>`: Cómo mostrar cambios (highlight-new, highlight-changed, side-by-side)

2. **`infrastructure-painter inspect`** - Analizar IaC sin generar diagrama completo
   - `--source, -s <path>`: Igual que generate
   - `--list-resources`: Listar todos los recursos encontrados
   - `--list-connections`: Listar todas las conexiones/relaciones
   - `--json`: Salida de análisis como JSON
   - `--summary`: Imprimir conteo de recursos por tipo
   - `--validate`: Verificar problemas comunes (recursos huérfanos, dependencias circulares)

3. **`infrastructure-painter validate`** - Validar definiciones de IaC para compatibilidad de diagrama
   - `--source, -s <path>`: Ruta a escanear
   - `--fix`: Auto-corregir problemas comunes (tags faltantes, nombres inconsistentes)
   - `--strict`: Fallar en cualquier advertencia
   - `--schema <schema>`: Validar contra versión de schema específica

4. **`infrastructure-painter diff`** - Comparar dos diagramas/infraestructuras
   - `--baseline <path>`: IaC o diagrama de línea base
   - `--target <path>`: IaC o diagrama objetivo
   - `--output <format>`: Formato de salida (text, html, json)
   - `--highlight-changes`: Enfatizar recursos añadidos/eliminados/cambiados
   - `--ignore <field>`: Ignorar cambios en campos específicos (tags, timestamps)

5. **`infrastructure-painter watch`** - Monitorear continuamente y actualizar diagramas
   - `--source <dir>`: Observar directorio para cambios
   - `--output <dir>`: Directorio de salida para diagramas generados
   - `--interval <seconds>`: Intervalo de polling (default: 30)
   - `--debounce <ms>`: Retraso de debounce después de cambios (default: 5000)
   - `--clear-cache`: Limpiar cache de estado en cada regeneración

## Proceso de Trabajo

Infrastructure Painter sigue este proceso detallado:

### Diagram Generation Pipeline

1. **Source Detection & Parsing**
   - Auto-detectar tipo de fuente por extensión de archivo:
     - `.tf`, `.tf.json` → Terraform
     - `.yaml`, `.yml`, `.json` con Cabecera CloudFormation → CloudFormation
     - `azuredeploy.json` → Plantilla ARM
     - `*.yaml`, `*.yml` con versión API Kubernetes → Manifiestos K8s
     - `*.k8s.yaml` → K8s (explícito)
   - Parsear IaC en representación intermedia de recursos
   - Para Terraform: Usar parser HCL2 para extraer recursos, variables, outputs y estructura de módulos
   - Para CloudFormation: Parsear template, resolver funciones intrínsecas (Ref, Fn::GetAtt, Fn::Join)
   - Para K8s: Parsear manifiestos, inferir relaciones desde selectores y referencias

2. **Resource Enrichment**
   - Si se proporciona `--state-file`: Cargar estado Terraform para obtener atributos reales de recursos (IPs, ARNs, estado)
   - Si se proporciona `--plan-file`: Parsear JSON de plan para identificar operaciones create/update/delete
   - Resolver referencias entre recursos (ej. subnet_id → recurso subnet real)
   - Inferir conexiones implícitas no definidas en IaC (ej. instancia EC2 → VPC basado en subnet)
   - Aplicar agrupación basada en tags si se especifica `--group-by tag:*`
   - Normalizar tipos de recursos a categorías estandarizadas (compute, network, storage, security)

3. **Graph Construction**
   - Crear representación Graphviz DOT o sintaxis Mermaid/PlantUML
   - Nodos representan recursos con:
     - Icono basado en provider/service (AWS S3, Azure Storage, GCP Compute)
     - Etiqueta mostrando nombre de recurso y atributos clave
     - Codificación de color por estado (planned=azul, created=verde, destroyed=rojo)
     - Forma basada en tipo (box=compute, cylinder=database, cloud=load balancer)
   - Aristas representan relaciones:
     - Línea sólida: dependencia concreta (depends_on, conexión explícita)
     - Línea punteada: relación inferida (conectividad de red, permisos IAM)
     - Dirección de flecha: flujo de datos o flujo de control

4. **Layout Rendering**
   - Aplicar algoritmo de layout:
     - `hierarchical`: Capas de arriba hacia abajo (mejor para arquitecturas en capas)
     - `circular`: Disposición circular (mejor para mostrar pares iguales)
     - `orthogonal`: Enrutamiento de ángulo recto (mejor para diagramas de red)
     - `fruchteman`: Dirigido por fuerza (mejor para redes complejas)
   - Manejar agrupación: Crear subgrafos cluster para grupos
   - Minimizar cruces de aristas usando algoritmos heurísticos

5. **Output Formatting**
   - Convertir grafo a formato seleccionado:
     - PNG/SVG/PDF: Via comando Graphviz `dot`
     - Mermaid: Generar sintaxis mermaid.js para incrustación web
     - PlantUML: Generar sintaxis PlantUML para integración
     - DOT: Fuente Graphviz cruda para edición manual
   - Aplicar colores de tema:
     - Tema AWS: Esquema de color naranja/blanco/azul estilo AWS Simple Icons
     - Tema Azure: Degradado azul con estilo de forma Azure
     - Tema GCP: Multicolor con colores de marca GCP
     - Tema Corporate: Escala de grises con mínimo branding
     - Tema Dark: Alto contraste para presentaciones
   - Añadir leyenda/anotaciones si se establece flag `--legend`
   - Generar pie de página de metadatos con timestamp de generación y checksum de fuente

6. **Caching & Optimization**
   - Cachear estado de IaC parseado en `~/.cache/infrastructure-painter/`
   - Clave de cache: ruta fuente + tiempo de modificación + hash de opciones
   - Omitir regeneración si cache válido y `--force` no establecido
   - Para modo `watch`: Usar regeneración incremental (solo subgrafos afectados)

### Comandos de Ejemplo

```bash
# Generar diagrama PNG desde proyecto Terraform
infrastructure-painter generate -s ./infra/aws -o architecture.png -f png -l hierarchical -t aws

# Generar diagrama Mermaid para README de GitHub
infrastructure-painter generate -s ./k8s/manifests -o diagram.mmd -f mermaid --show-connections

# Analizar qué recursos se crearán antes de terraform apply
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
infrastructure-painter generate -s . --plan-file plan.json --plan-mode highlight-new -o planned-changes.svg

# Observar directorio y auto-actualizar diagrama cuando cambian archivos
infrastructure-painter watch -s ./environments/prod -o ./docs/diagrams --interval 60

# Comparar infraestructura entre ramas
git checkout feature-branch
infrastructure-painter generate -s . -o /tmp/feature.dot
git checkout main
infrastructure-painter generate -s . -o /tmp/main.dot
infrastructure-painter diff --baseline /tmp/main.dot --target /tmp/feature.dot --output html > diff.html

# Listar todos los recursos encontrados en un módulo Terraform
infrastructure-painter inspect -s ./modules/vpc --list-resources --json

# Validar Terraform para compatibilidad de diagrama (verifica tags requeridos)
infrastructure-painter validate -s ./infra --strict
```

### Variables de Entorno

```bash
# Autenticación (opcional para parsing, requerida para enrichment de estado)
export AWS_PROFILE=production
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_REGION=us-east-1

export AZURE_SUBSCRIPTION_ID=xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
export AZURE_CLIENT_ID=xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
export AZURE_CLIENT_SECRET=secret
export AZURE_TENANT_ID=xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

export GOOGLE_APPLICATION_CREDENTIALS=/path/to/gcp-key.json

# Configuración
export INFRASTRUCTURE_PAINTER_CACHE_DIR=/custom/cache/path
export INFRASTRUCTURE_PAINTER_THEME=aws
export INFRASTRUCTURE_PAINTER_DEFAULT_LAYOUT=hierarchical
export INFRASTRUCTURE_PAINTER_GRAPHVIZ_DOT=/usr/local/bin/dot  # Ruta a Graphviz

# Logging
export INFRASTRUCTURE_PAINTER_LOG_LEVEL=INFO  # DEBUG, INFO, WARN, ERROR
export INFRASTRUCTURE_PAINTER_LOG_FILE=/var/log/infra-painter.log
```

## Reglas de Oro

Infrastructure Painter DEBE adherirse a estas reglas:

1. **Never expose secrets**: Las etiquetas de recursos NUNCA deben mostrar campos sensibles (contraseñas, claves privadas, connection strings). Filtrar cualquier atributo llamado `*password*`, `*secret*`, `*key*`, `*token*` antes de etiquetar. Sanitizar en `graph_construction.py:312`.

2. **Maintain consistency**: El mismo tipo de recurso debe usar icono y color idénticos en todos los diagramas. Definir todo el styling en directorio `themes/`; nunca hardcodear colores.

3. **Inference limits**: Solo inferir conexiones que sean:
   - Definidas explícitamente en IaC (referencias, depends_on)
   - Topología de red estándar (subnet → VPC, instance → subnet)
   - Documentadas en `inference_rules.yaml`
   - Marcadas con flag `--infer-implicit` (deshabilitado por defecto)

4. **Handle large graphs**: Si el conteo de recursos > 500, habilitar automáticamente clustering y `--max-depth 3`. Emitir advertencia: "Diagram contains 500+ resources; limiting depth to 3. Use --max-depth to override."

5. **Respect exclusions**: Los patrones `--exclude` se aplican ANTES del enrichment de estado. Ningún tipo de recurso excluido debe aparecer en el diagrama, incluso si es referenciado por recursos incluidos.

6. **Plan mode accuracy**: Al usar `--plan-file`, solo marcar recursos como "planned" si `change.actions` contiene "create". Recursos con "update" son "modified" (color diferente). Recursos "delete" mostrados con tachado.

7. **Exit codes**:
   - `0`: Success
   - `1`: Invalid arguments or unsupported format
   - `2`: Source parsing failed (syntax error in IaC)
   - `3`: Missing dependencies (terraform, graphviz not found)
   - `4`: State/plan file validation failed
   - `5`: Graphviz rendering error (likely memory limit)
   - `10-127`: Internal errors with specific codes documented in `errors.py`

8. **Backward compatibility**: Parsear versiones de estado Terraform antiguas (v4, v5) pero requerir >= v6. Documentar versiones mínimas soportadas en `README.md`.

9. **No destructive operations**: Infrastructure Painter es read-only. Nunca modificar archivos IaC o estado. Todos los archivos de cache almacenados en directorios escribibles por usuario.

10. **Performance budget**: Generar diagrama con 100 recursos en < 5 segundos en runner CI estándar (2 vCPU, 4GB RAM). Usar lazy loading y parsing paralelo donde sea posible.

## Ejemplos

### Ejemplo 1: Arquitectura AWS de Tres Niveles

**Terraform de Entrada** (`infra/main.tf`):
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "production-vpc" }
}

resource "aws_subnet" "web" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  tags = { Name = "web-subnet" }
}

resource "aws_subnet" "app" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  tags = { Name = "app-subnet" }
}

resource "aws_subnet" "db" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.3.0/24"
  tags = { Name = "db-subnet" }
}

resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.web.id
  tags = { Name = "web-1" }
}

resource "aws_lb" "app_lb" {
  load_balancer_type = "application"
  subnets            = [aws_subnet.web.id, aws_subnet.app.id]
}

resource "aws_db_instance" "postgres" {
  engine               = "postgres"
  db_instance_class    = "db.t3.micro"
  vpc_security_group_ids = [aws_security_group.db.id]
  subnet_id            = aws_subnet.db.id
}
```

**Comando**:
```bash
infrastructure-painter generate \
  -s ./infra \
  -o ./docs/aws-architecture.svg \
  -f svg \
  -l hierarchical \
  -t aws \
  --group-by tag:Name \
  --show-labels
```

**Diagrama de Salida**:
- VPC mostrado como contenedor de nube grande conteniendo tres subredes
- ALB conectado a subredes web y app con flechas mostrando flujo de tráfico
- Instancia EC2 en subred web con etiqueta t3.micro
- Instancia RDS en subred db con etiqueta postgres
- Grupos de seguridad mostrados como iconos de escudo con reglas inbound/outbound
- Codificación de color: compute=amarillo, network=cian, storage=verde

**Mermaid Generado** (si `-f mermaid`):
```mermaid
graph TD
    subgraph "production-vpc"
        WEB_SUBNET[web-subnet]
        APP_SUBNET[app-subnet]
        DB_SUBNET[db-subnet]
    end
    ALB[app_lb<br/>ALB]
    WEB1[web-1<br/>t3.micro]
    PG[postgres<br/>db.t3.micro]
    
    ALB --> WEB_SUBNET
    ALB --> APP_SUBNET
    WEB_SUBNET --> WEB1
    DB_SUBNET --> PG
```

### Ejemplo 2: Microservicios Kubernetes

**Entrada** (`k8s/production.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: api
  ports:
  - port: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

**Comando**:
```bash
infrastructure-painter generate \
  -s ./k8s/production.yaml \
  -o ./docs/k8s-topology.png \
  -f png \
  -l hierarchical \
  --group-by namespace \
  --include 'Deployment|Service|Ingress'
```

**Diagrama de Salida**:
- Nodo de clúster mostrado como contenedor exterior
- Namespace "production" como subgrafo de clúster
- Jerarquía Deployment → ReplicaSet → Pod mostrada
- Service conectado a todos los pods coincidiendo con selector
- Ingress conectado a service
- Colores: ingress=púrpura, service=azul, deployment=verde, pod=naranja

### Ejemplo 3: Multi-cloud con Vista Previa de Plan

**Escenario**: Planificando migrar de t2.micro a t3.large y añadir caché Redis

**Comando**:
```bash
# Generar diagrama de línea base
infrastructure-painter generate \
  -s ./infra \
  --state-file terraform.tfstate \
  -o baseline.svg

# Aplicar cambios y generar plan
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Generar diagrama mostrando cambios
infrastructure-painter generate \
  -s ./infra \
  --plan-file plan.json \
  --plan-mode highlight-new \
  -o with-changes.svg
```

**Diferencias de Salida**:
- Etiqueta de instancia EC2 cambia de `t2.micro` (gris) a `t3.large` (resaltado azul)
- Nuevo recurso Redis aparece con borde punteado (planificado)
- Flecha de EC2 a Redis muestra nueva conexión
- Pie de página muestra: "Generated from plan: 1 change (1 add, 0 modify, 0 destroy)"

### Ejemplo 4: Validación e Inspección

**Comando**:
```bash
# Verificar tags faltantes y nombres inconsistentes
infrastructure-painter validate -s ./infra --strict

# Salida:
# WARN: Resource aws_instance.web_server missing tag 'Environment'
# WARN: Resource aws_db_instance.postgres missing tag 'Owner'
# ERROR: Resource aws_lb.app_lb has unstandardized name (should be 'app-lb')
# Validation failed with 1 ERROR, 2 WARNINGS
# Exit code 1
```

**Inspeccionar recursos**:
```bash
infrastructure-painter inspect -s ./infra --list-resources --json

# Salida:
# {
#   "resources": [
#     {"type": "aws_vpc", "name": "main", "id": "vpc-12345"},
#     {"type": "aws_subnet", "name": "web", "cidr": "10.0.1.0/24"},
#     {"type": "aws_instance", "name": "web_server", "type": "t3.micro"}
#   ],
#   "connections": [
#     {"from": "aws_subnet.web", "to": "aws_instance.web_server", "type": "contains"}
#   ]
# }
```

## Comandos de Rollback

Infrastructure Painter soporta estas operaciones de rollback/recovery:

1. **Limpiar cache corrupto**
   ```bash
   infrastructure-painter cache --clear
   # o manualmente: rm -rf ~/.cache/infrastructure-painter/
   ```

2. **Regenerar diagrama desde estado anterior**
   ```bash
   # Si el diagrama actual está roto, regenerar desde último estado bueno conocido
   infrastructure-painter generate -s ./infra --state-file terraform.tfstate.backup -o architecture.svg
   ```

3. **Revertir a versión específica** (si diagramas versionados almacenados)
   ```bash
   git checkout v1.2.3 -- docs/architecture.svg
   ```

4. **Deshabilitar cache temporalmente** (si se sospecha corrupción de cache)
   ```bash
   infrastructure-painter generate -s ./infra --no-cache -o architecture.svg
   ```

5. **Fallback a salida DOT** (si falla el rendering)
   ```bash
   infrastructure-painter generate -s ./infra --format dot -o architecture.dot
   # Luego renderizar manualmente con: dot -Tsvg architecture.dot > architecture.svg
   ```

6. **Recuperarse de OOM de Graphviz**
   ```bash
   # Reducir complejidad
   infrastructure-painter generate -s ./infra --max-depth 2 --exclude 'aws_iam*' -o simple.svg
   
   # Aumentar límite de memoria de Graphviz
   export INFRASTRUCTURE_PAINTER_GRAPHVIZ_OPTS="- memory=4096"
   ```

7. **Restaurar configuración por defecto**
   ```bash
   # Eliminar config personalizado
   rm ~/.config/infrastructure-painter/config.yaml
   infrastructure-painter --defaults generate -s ./infra -o architecture.svg
   ```

## Solución de Problemas

### Issue: "Error: Graphviz not found"
**Solución**: Instalar Graphviz
```bash
# Ubuntu/Debian
apt-get install graphviz

# macOS
brew install graphviz

# Verificar: dot -V
```

### Issue: "Parsing failed: Unexpected token"
**Solución**: Verificar sintaxis Terraform con `terraform validate`. Asegurar formato HCL2 (sin sintaxis legacy). Actualizar parser: `pip install --upgrade infrastructure-painter`.

### Issue: Diagram missing resources
**Solución**:
- Usar `--verbose` para ver recursos parseados
- Verificar patrones `--exclude` no estén filtrando recursos necesarios
- Si se usa `--state-file`, asegurar que esté actualizado: `terraform state pull > terraform.tfstate`
- Algunos backends de estado remoto requieren flag `--state` explícito

### Issue: Connections not showing
**Solución**:
- Infrastructure Painter solo muestra referencias explícitas. Añadir `depends_on` explícito o referencia en IaC.
- Usar flag `--infer-implicit` para habilitar inferencia de red/grupo (puede producir falsos positivos)
- K8s: Asegurar services tienen `selector` correcto coincidiendo con etiquetas de pod

### Issue: Graphviz OOM or timeout
**Solución**:
- Usar `--max-depth` para limitar profundidad
- Excluir tipos de recursos: `--exclude "aws_iam*" --exclude "aws_cloudwatch*"`
- Cambiar layout: `-l orthogonal` (más eficiente que fruchterman)
- Aumentar memoria: `export GRAPHVIZ_MEMORY=4096`

### Issue: Plan mode not highlighting changes
**Solución**:
- Asegurar archivo de plan es JSON, no binario: `terraform show -json tfplan > plan.json`
- Verificar JSON de plan tiene array `resource_changes` con `change.actions`
- Acciones soportadas: "create", "update", "delete", "no-op"
- Usar `--plan-mode highlight-new` para mejor visibilidad

### Issue: Colors/icons look wrong
**Solución**:
- Verificar tema: `-t aws` para recursos AWS; `-t azure` para Azure
- Asegurar tipo de recurso está mapeado en `~/.config/infrastructure-painter/icons.yaml`
- Limpiar cache de iconos: `infrastructure-painter cache --clear-icons`

### Issue: Watch mode not triggering
**Solución**:
- Verificar permisos de archivo: usuario debe tener acceso de lectura a archivos fuente
- Usar rutas absolutas: `-s $(pwd)/infra -o $(pwd)/docs`
- Intervalo por defecto es 30s; usar `--interval 5` para feedback más rápido
- Asegurar archivos tienen actualizaciones de tiempo de modificación (algunos editores no actualizan mtime)

### Issue: Performance slow on large codebases
**Solución**:
- Habilitar cache (por defecto) o usar flag `--reuse-cache`
- Usar `--exclude` para omitir módulos irrelevantes
- Dividir por environment: ejecutar separadamente por directorio `env/`
- Paralelizar: `infrastructure-painter generate -s env1 -o out1 &; infrastructure-painter generate -s env2 -o out2 &`
- Aumentar workers: `export INFRASTRUCTURE_PAINTER_WORKERS=4`
```