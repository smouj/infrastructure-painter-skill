---
title: Infrastructure Painter
version: 1.2.4
description: Automatically generates visual diagrams of cloud infrastructure from IaC definitions
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

Automatically generates visual diagrams of cloud infrastructure from Infrastructure as Code (IaC) definitions. Supports Terraform, AWS CloudFormation, Azure Resource Manager templates, and Kubernetes manifests. Produces professional architecture diagrams in multiple formats (PNG, SVG, PDF, Mermaid, PlantUML).

## Purpose

Infrastructure Painter solves these specific problems:

- **Cloud Architecture Visualization**: Transform complex Terraform modules into clean architecture diagrams showing VPCs, subnets, security groups, EC2 instances, RDS clusters, load balancers, and their relationships
- **Documentation Generation**: Auto-generate up-to-date infrastructure diagrams as part of CI/CD pipelines, ensuring documentation always matches deployed infrastructure
- **Impact Analysis**: Before running `terraform apply`, generate a preview diagram of planned changes to understand the blast radius
- **Multi-cloud Mapping**: Visualize hybrid deployments across AWS, Azure, and GCP in unified diagrams
- **Kubernetes Topology**: Map out K8s clusters showing namespaces, deployments, services, ingress controllers, and network policies
- **Security Review**: Highlight security boundaries, IAM roles, and data flow for compliance audits
- **Onboarding**: New engineers can visualize the entire infrastructure stack instead of reading hundreds of lines of HCL

Real use cases:
- Generate architecture diagrams for quarterly architecture review meetings
- Create network topology diagrams showing VPC peering and Transit Gateway connections
- Visualize microservices communication patterns in a service mesh
- Document disaster recovery setups showing multi-region replication
- Map out PCI compliance boundaries showing Cardholder Data Environment (CDE) segmentation

## Scope

Infrastructure Painter provides these concrete commands:

### Main Commands

1. **`infrastructure-painter generate`** - Core diagram generation
   - `--source, -s <path>`: Path to Terraform directory, CloudFormation template, or K8s manifest
   - `--output, -o <file>`: Output file path (auto-detects format from extension)
   - `--format, -f <format>`: Force output format (png, svg, pdf, mermaid, plantuml, dot)
   - `--layout, -l <type>`: Layout algorithm (hierarchical, circular, orthogonal, fruchterman)
   - `--theme, -t <theme>`: Color theme (aws, azure, gcp, minimal, dark, corporate)
   - `--exclude, -e <pattern>`: Exclude resources matching pattern (e.g., "aws_iam.*", "kubernetes_*")
   - `--include, -i <pattern>`: Only include resources matching pattern
   - `--group-by, -g <field>`: Group resources by tag, type, or environment (e.g., "tag:Environment", "type")
   - `--show-labels, --no-show-labels`: Toggle resource labels (default: true)
   - `--show-connections, --no-show-connections`: Toggle connection lines (default: true)
   - `--max-depth, -d <n>`: Maximum hierarchy depth to render (default: unlimited)
   - `--state-file, -state <path>`: Use specific Terraform state file for live resource data
   - `--plan-file, -plan <path>`: Use Terraform plan JSON to show planned changes
   - `--plan-mode <mode>`: How to show changes (highlight-new, highlight-changed, side-by-side)

2. **`infrastructure-painter inspect`** - Analyze IaC without generating full diagram
   - `--source, -s <path>`: Same as generate
   - `--list-resources`: List all resources found
   - `--list-connections`: List all connections/relationships
   - `--json`: Output analysis as JSON
   - `--summary`: Print resource count by type
   - `--validate`: Check for common issues (orphaned resources, circular dependencies)

3. **`infrastructure-painter validate`** - Validate IaC definitions for diagram compatibility
   - `--source, -s <path>`: Path to scan
   - `--fix`: Auto-fix common issues (missing tags, inconsistent naming)
   - `--strict`: Fail on any warnings
   - `--schema <schema>`: Validate against specific schema version

4. **`infrastructure-painter diff`** - Compare two diagrams/infrastructures
   - `--baseline <path>`: Baseline IaC or diagram
   - `--target <path>`: Target IaC or diagram
   - `--output <format>`: Output format (text, html, json)
   - `--highlight-changes`: Emphasize added/removed/changed resources
   - `--ignore <field>`: Ignore changes in specific fields (tags, timestamps)

5. **`infrastructure-painter watch`** - Continuously monitor and update diagrams
   - `--source <dir>`: Watch directory for changes
   - `--output <dir>`: Output directory for generated diagrams
   - `--interval <seconds>`: Polling interval (default: 30)
   - `--debounce <ms>`: Debounce delay after changes (default: 5000)
   - `--clear-cache`: Clear state cache on each regeneration

## Work Process

Infrastructure Painter follows this detailed process:

### Diagram Generation Pipeline

1. **Source Detection & Parsing**
   - Auto-detect source type by file extension:
     - `.tf`, `.tf.json` → Terraform
     - `.yaml`, `.yml`, `.json` with CloudFormation header → CloudFormation
     - `azuredeploy.json` → ARM template
     - `*.yaml`, `*.yml` with Kubernetes API version → K8s manifests
     - `*.k8s.yaml` → K8s (explicit)
   - Parse IaC into intermediate resource representation
   - For Terraform: Use HCL2 parser to extract resources, variables, outputs, and module structure
   - For CloudFormation: Parse template, resolve intrinsic functions (Ref, Fn::GetAtt, Fn::Join)
   - For K8s: Parse manifests, infer relationships from selectors and references

2. **Resource Enrichment**
   - If `--state-file` provided: Load Terraform state to get actual resource attributes (IPs, ARNs, status)
   - If `--plan-file` provided: Parse plan JSON to identify create/update/delete operations
   - Resolve cross-resource references (e.g., subnet_id → actual subnet resource)
   - Infer implicit connections not defined in IaC (e.g., EC2 instance → VPC based on subnet)
   - Apply tag-based grouping if `--group-by tag:*` specified
   - Normalize resource types to standardized categories (compute, network, storage, security)

3. **Graph Construction**
   - Create Graphviz DOT representation or Mermaid/PlantUML syntax
   - Nodes represent resources with:
     - Icon based on provider/service (AWS S3, Azure Storage, GCP Compute)
     - Label showing resource name and key attributes
     - Color coding by status (planned=blue, created=green, destroyed=red)
     - Shape based on type (box=compute, cylinder=database, cloud=load balancer)
   - Edges represent relationships:
     - Solid line: concrete dependency (depends_on, explicit connection)
     - Dashed line: inferred relationship (network connectivity, IAM permissions)
     - Arrow direction: data flow or control flow

4. **Layout Rendering**
   - Apply layout algorithm:
     - `hierarchical`: Top-down layering (best for layered architectures)
     - `circular`: Circular arrangement (best for showing equal peers)
     - `orthogonal`: Right-angle routing (best for network diagrams)
     - `fruchteman`: Force-directed (best for complex webs)
   - Handle grouping: Create cluster subgraphs for groups
   - Minimize edge crossings using heuristic algorithms

5. **Output Formatting**
   - Convert graph to selected format:
     - PNG/SVG/PDF: Via Graphviz `dot` command
     - Mermaid: Generate mermaid.js syntax for web embedding
     - PlantUML: Generate PlantUML syntax for integration
     - DOT: Raw Graphviz source for manual editing
   - Apply theme colors:
     - AWS theme: Orange/white/blue color scheme with AWS Simple Icons style
     - Azure theme: Blue gradient with Azure shape styling
     - GCP theme: Multicolor with GCP brand colors
     - Corporate theme: Grayscale with minimal branding
     - Dark theme: High-contrast for presentations
   - Add legend/annotations if `--legend` flag set
   - Generate metadata footer with generation timestamp and source checksum

6. **Caching & Optimization**
   - Cache parsed IaC state in `~/.cache/infrastructure-painter/`
   - Cache key: source path + modification time + options hash
   - Skip regeneration if cache valid and `--force` not set
   - For `watch` mode: Use incremental regeneration (only affected subgraphs)

### Command Examples

```bash
# Generate PNG diagram from Terraform project
infrastructure-painter generate -s ./infra/aws -o architecture.png -f png -l hierarchical -t aws

# Generate Mermaid diagram for GitHub README
infrastructure-painter generate -s ./k8s/manifests -o diagram.mmd -f mermaid --show-connections

# Analyze what resources will be created before terraform apply
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
infrastructure-painter generate -s . --plan-file plan.json --plan-mode highlight-new -o planned-changes.svg

# Watch directory and auto-update diagram when files change
infrastructure-painter watch -s ./environments/prod -o ./docs/diagrams --interval 60

# Compare infrastructure between branches
git checkout feature-branch
infrastructure-painter generate -s . -o /tmp/feature.dot
git checkout main
infrastructure-painter generate -s . -o /tmp/main.dot
infrastructure-painter diff --baseline /tmp/main.dot --target /tmp/feature.dot --output html > diff.html

# List all resources found in a Terraform module
infrastructure-painter inspect -s ./modules/vpc --list-resources --json

# Validate Terraform for diagram compatibility (checks for required tags)
infrastructure-painter validate -s ./infra --strict
```

### Environment Variables

```bash
# Authentication (optional for parsing, required for state enrichment)
export AWS_PROFILE=production
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_REGION=us-east-1

export AZURE_SUBSCRIPTION_ID=xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
export AZURE_CLIENT_ID=xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
export AZURE_CLIENT_SECRET=secret
export AZURE_TENANT_ID=xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

export GOOGLE_APPLICATION_CREDENTIALS=/path/to/gcp-key.json

# Configuration
export INFRASTRUCTURE_PAINTER_CACHE_DIR=/custom/cache/path
export INFRASTRUCTURE_PAINTER_THEME=aws
export INFRASTRUCTURE_PAINTER_DEFAULT_LAYOUT=hierarchical
export INFRASTRUCTURE_PAINTER_GRAPHVIZ_DOT=/usr/local/bin/dot  # Path to Graphviz

# Logging
export INFRASTRUCTURE_PAINTER_LOG_LEVEL=INFO  # DEBUG, INFO, WARN, ERROR
export INFRASTRUCTURE_PAINTER_LOG_FILE=/var/log/infra-painter.log
```

## Golden Rules

Infrastructure Painter MUST adhere to these rules:

1. **Never expose secrets**: Resource labels must NEVER display sensitive fields (passwords, private keys, connection strings). Filter out any attribute named `*password*`, `*secret*`, `*key*`, `*token*` before labeling. Sanitize at `graph_construction.py:312`.

2. **Maintain consistency**: Same resource type must use identical icon and color across all diagrams. Define all styling in `themes/` directory; never hardcode colors.

3. **Inference limits**: Only infer connections that are:
   - Explicitly defined in IaC (references, depends_on)
   - Standard network topology (subnet → VPC, instance → subnet)
   - Documented in `inference_rules.yaml`
   - Marked with `--infer-implicit` flag (disabled by default)

4. **Handle large graphs**: If resource count > 500, automatically enable clustering and `--max-depth 3`. Emit warning: "Diagram contains 500+ resources; limiting depth to 3. Use --max-depth to override."

5. **Respect exclusions**: `--exclude` patterns apply BEFORE state enrichment. No excluded resource type should appear in diagram, even if referenced by included resources.

6. **Plan mode accuracy**: When using `--plan-file`, only mark resources as "planned" if `change.actions` contains "create". Resources with "update" are "modified" (different color). "delete" resources shown with strikethrough.

7. **Exit codes**:
   - `0`: Success
   - `1`: Invalid arguments or unsupported format
   - `2`: Source parsing failed (syntax error in IaC)
   - `3`: Missing dependencies (terraform, graphviz not found)
   - `4`: State/plan file validation failed
   - `5`: Graphviz rendering error (likely memory limit)
   - `10-127`: Internal errors with specific codes documented in `errors.py`

8. **Backward compatibility**: Parse older Terraform state versions (v4, v5) but require >= v6. Document minimum supported versions in `README.md`.

9. **No destructive operations**: Infrastructure Painter is read-only. Never modify IaC files or state. All cache files stored in user-writable directories.

10. **Performance budget**: Generate diagram with 100 resources in < 5 seconds on standard CI runner (2 vCPU, 4GB RAM). Use lazy loading and parallel parsing where possible.

## Examples

### Example 1: AWS Three-Tier Architecture

**Input Terraform** (`infra/main.tf`):
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

**Command**:
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

**Output Diagram**:
- VPC shown as large cloud container containing three subnets
- ALB connected to web and app subnets with arrows showing traffic flow
- EC2 instance in web subnet with t3.micro label
- RDS instance in db subnet with postgres label
- Security groups shown as shield icons with inbound/outbound rules
- Color coding: compute=yellow, network=cyan, storage=green

**Generated Mermaid** (if `-f mermaid`):
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

### Example 2: Kubernetes Microservices

**Input** (`k8s/production.yaml`):
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

**Command**:
```bash
infrastructure-painter generate \
  -s ./k8s/production.yaml \
  -o ./docs/k8s-topology.png \
  -f png \
  -l hierarchical \
  --group-by namespace \
  --include 'Deployment|Service|Ingress'
```

**Output Diagram**:
- Cluster node shown as outer container
- Namespace "production" as cluster subgraph
- Deployment → ReplicaSet → Pod hierarchy shown
- Service connected to all pods with selector matching
- Ingress connected to service
- Colors: ingress=purple, service=blue, deployment=green, pod=orange

### Example 3: Multi-Cloud with Plan Preview

**Scenario**: Planning to migrate from t2.micro to t3.large and add Redis cache

**Command**:
```bash
# Generate baseline diagram
infrastructure-painter generate \
  -s ./infra \
  --state-file terraform.tfstate \
  -o baseline.svg

# Apply changes and generate plan
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Generate diagram showing changes
infrastructure-painter generate \
  -s ./infra \
  --plan-file plan.json \
  --plan-mode highlight-new \
  -o with-changes.svg
```

**Output Differences**:
- EC2 instance label changes from `t2.micro` (gray) to `t3.large` (blue highlight)
- New Redis resource appears with dashed border (planned)
- Arrow from EC2 to Redis shows new connection
- Footer shows: "Generated from plan: 1 change (1 add, 0 modify, 0 destroy)"

### Example 4: Validation and Inspection

**Command**:
```bash
# Check for missing tags and inconsistent naming
infrastructure-painter validate -s ./infra --strict

# Output:
# WARN: Resource aws_instance.web_server missing tag 'Environment'
# WARN: Resource aws_db_instance.postgres missing tag 'Owner'
# ERROR: Resource aws_lb.app_lb has unstandardized name (should be 'app-lb')
# Validation failed with 1 ERROR, 2 WARNINGS
# Exit code 1
```

**Inspect resources**:
```bash
infrastructure-painter inspect -s ./infra --list-resources --json

# Output:
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

## Rollback Commands

Infrastructure Painter supports these rollback/recovery operations:

1. **Clear corrupted cache**
   ```bash
   infrastructure-painter cache --clear
   # or manually: rm -rf ~/.cache/infrastructure-painter/
   ```

2. **Regenerate diagram from previous state**
   ```bash
   # If current diagram is broken, regenerate from last known good state file
   infrastructure-painter generate -s ./infra --state-file terraform.tfstate.backup -o architecture.svg
   ```

3. **Revert to specific version** (if versioned diagrams stored)
   ```bash
   git checkout v1.2.3 -- docs/architecture.svg
   ```

4. **Disable caching temporarily** (if cache corruption suspected)
   ```bash
   infrastructure-painter generate -s ./infra --no-cache -o architecture.svg
   ```

5. **Fallback to DOT output** (if rendering fails)
   ```bash
   infrastructure-painter generate -s ./infra --format dot -o architecture.dot
   # Then manually render with: dot -Tsvg architecture.dot > architecture.svg
   ```

6. **Recover from Graphviz OOM**
   ```bash
   # Reduce complexity
   infrastructure-painter generate -s ./infra --max-depth 2 --exclude 'aws_iam*' -o simple.svg
   
   # Increase Graphviz memory limit
   export INFRASTRUCTURE_PAINTER_GRAPHVIZ_OPTS="- memory=4096"
   ```

7. **Restore default configuration**
   ```bash
   # Remove custom config
   rm ~/.config/infrastructure-painter/config.yaml
   infrastructure-painter --defaults generate -s ./infra -o architecture.svg
   ```

## Troubleshooting

### Issue: "Error: Graphviz not found"
**Solution**: Install Graphviz
```bash
# Ubuntu/Debian
apt-get install graphviz

# macOS
brew install graphviz

# Verify: dot -V
```

### Issue: "Parsing failed: Unexpected token"
**Solution**: Check Terraform syntax with `terraform validate`. Ensure HCL2 format (no legacy syntax). Upgrade parser: `pip install --upgrade infrastructure-painter`.

### Issue: Diagram missing resources
**Solution**:
- Use `--verbose` to see parsed resources
- Check `--exclude` patterns aren't filtering needed resources
- If using `--state-file`, ensure it's up-to-date: `terraform state pull > terraform.tfstate`
- Some remote state backends require explicit `--state` flag

### Issue: Connections not showing
**Solution**:
- Infrastructure Painter only shows explicit references. Add explicit `depends_on` or reference in IaC.
- Use `--infer-implicit` flag to enable network/group inference (may produce false positives)
- K8s: Ensure services have correct `selector` matching pod labels

### Issue: Graphviz OOM or timeout
**Solution**:
- Use `--max-depth` to limit depth
- Exclude resource types: `--exclude "aws_iam*" --exclude "aws_cloudwatch*"`
- Switch layout: `-l orthogonal` (more efficient than fruchterman)
- Increase memory: `export GRAPHVIZ_MEMORY=4096`

### Issue: Plan mode not highlighting changes
**Solution**:
- Ensure plan file is JSON, not binary: `terraform show -json tfplan > plan.json`
- Check plan JSON has `resource_changes` array with `change.actions`
- Supported actions: "create", "update", "delete", "no-op"
- Use `--plan-mode highlight-new` for best visibility

### Issue: Colors/icons look wrong
**Solution**:
- Verify theme: `-t aws` for AWS resources; `-t azure` for Azure
- Ensure resource type is mapped in `~/.config/infrastructure-painter/icons.yaml`
- Clear icon cache: `infrastructure-painter cache --clear-icons`

### Issue: Watch mode not triggering
**Solution**:
- Check file permissions: user must have read access to source files
- Use absolute paths: `-s $(pwd)/infra -o $(pwd)/docs`
- Default interval is 30s; use `--interval 5` for faster feedback
- Ensure files have modification time updates (some editors don't update mtime)

### Issue: Performance slow on large codebases
**Solution**:
- Enable cache (default) or use `--reuse-cache` flag
- Use `--exclude` to skip irrelevant modules
- Split by environment: run separately per `env/` directory
- Parallelize: `infrastructure-painter generate -s env1 -o out1 &; infrastructure-painter generate -s env2 -o out2 &`
- Increase workers: `export INFRASTRUCTURE_PAINTER_WORKERS=4`
```