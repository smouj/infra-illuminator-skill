---
name: infra-illuminator
description: Discovers hidden dependencies and relationships across distributed infrastructure
version: "1.0.0"
author: Kilo Engineering Team
tags: [dependencies, mapping, visualization, discovery, infrastructure, topology]
maintainers: ["ops@kilo.ai"]
homepage: "https://docs.kilo.ai/skills/infra-illuminator"
repository: "https://github.com/Kilo-Org/kilocode-skills"
license: "MIT"
dependencies:
  - name: kubectl
    version: ">=1.28.0"
    purpose: "Kubernetes cluster discovery"
  - name: terraform
    version: ">=1.5.0"
    purpose: "IaC dependency graph generation"
  - name: aws-cli
    version: ">=2.13.0"
    purpose: "AWS resource relationship mapping"
  - name: prometheus
    version: ">=2.45.0"
    purpose: "Service metrics correlation"
  - name: graphviz
    version: ">=2.50.0"
    purpose: "Dependency graph visualization"
  - name: jq
    version: ">=1.6"
    purpose: "JSON processing and filtering"
  - name: docker
    version: ">=24.0.0"
    purpose: "Container network inspection"
platforms: ["linux", "darwin"]
minimum_system_memory: "2GB"
timeout_seconds: 300
---

# Infra Illuminator

Discovers hidden dependencies and relationships across distributed systems, generating visual dependency maps and identifying single points of failure.

## Purpose

**Primary Use Cases:**

- **Service Dependency Mapping**: Uncover implicit couplings between microservices by analyzing traffic logs, configuration files, and network policies. Example: Identify that the `payment-service` indirectly depends on `legacy-auth` through `notification-worker` after tracing 3 hops of message queue connections.

- **Cloud Resource Inventory**: Parse Terraform state files and AWS Config to map relationships between VPCs, subnets, security groups, IAM roles, and data stores. Example: Find that 12 Lambda functions share a single IAM execution role with overly permissive S3 access.

- **Configuration Drift Detection**: Compare live infrastructure against IaC definitions to identify unmanaged resources or relationships. Example: Discover 3 EC2 instances with security group rules not defined in Terraform.

- **Single Point of Failure Analysis**: Calculate criticality scores for each node in the dependency graph based on downstream impact. Example: Flag that `shared-redis-cluster` serves 8 critical services with zero redundancy.

- **Impact Analysis**: Before deployments, trace all downstream systems that could be affected by a change. Example: Determine that updating the `user-api` will impact 5 frontend services, 2 mobile apps, and 15 background workers.

## Scope

**Commands Implemented:**

```bash
# Kubernetes cluster discovery
kubectl get all --all-namespaces -o json | jq -r '.items[] | "\(.kind)/\(.metadata.namespace)/\(.metadata.name)"'
kubectl get networkpolicies --all-namespaces -o json | infra-illuminator parse-network-policies
kubectl get endpoints,services,ingresses --all-namespaces -o wide

# Terraform dependency analysis
terraform show -json | jq '.values.root_module.child_modules[].resources[] | select(.type=="aws_instance")' 
terraform graph | dot -Tpng > dependencies.png
terraform state pull | jq '.resources[] | {type: .type, name: .name, instances: .instances}'

# AWS resource mapping
aws resource-explorer2 list-indexes --output json
aws resource-explorer2 search --query-string "resourceType:AWS::EC2::Instance" --max-results 1000
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId, VpcId, SecurityGroups]' --output json
aws elbv2 describe-load-balancers --query 'LoadBalancers[].[LoadBalancerArn, VpcId, Subnets, SecurityGroups]'
aws apigateway get-rest-apis --query 'items[].[id, name, endpointConfiguration.types]'

# Prometheus service topology
curl -s "http://prometheus:9090/api/v1/query?query=up{job=~\"api-.*\"}" | jq '.data.result[] | {instance, job, value}'
curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total[5m])" | jq '.data.result[] | {service: labels.__name__, instance: labels.instance}'
promtool query instant "sum(rate(http_requests_total[5m])) by (service, instance)"

# Docker/container network inspection
docker network ls --format "table {{.Name}}\t{{.ID}}"
docker network inspect <network-id> --format '{{json .Containers}}' | jq -r '.[] | "\(.Name): \(.IPv4Address)"'
docker ps --format "table {{.Names}}\t{{.Networks}}"

# Service mesh (Istio) dependency mapping
istioctl proxy-config clusters <pod-name> --context <cluster-context>
istioctl proxy-config listeners <pod-name> --context <cluster-context>
kubectl get virtualservices,gateways --all-namespaces -o json | jq -r '.items[] | "\(.kind) \(.metadata.namespace)/\(.metadata.name)"'

# Configuration file scanning
find /etc /opt /var -name "*.yml" -o -name "*.yaml" -o -name "*.json" | xargs grep -l "database\|redis\|rabbitmq\|kafka" 2>/dev/null
grep -r "http://\|https://" /etc/nginx/sites-enabled/ /etc/haproxy/ 2>/dev/null | grep -v "#"
```

## Detailed Work Process

### Phase 1: Discovery Preparation (15-30 minutes)
1. **Environment Validation**
   - Verify kubectl context: `kubectl config current-context`
   - Validate AWS credentials: `aws sts get-caller-identity`
   - Test Prometheus connectivity: `curl -s http://prometheus:9090/-/healthy`
   - Check Terraform workspace: `terraform workspace show`

2. **Scope Definition**
   - Accept input: namespaces, clusters, regions, time windows
   - Build discovery manifest:
   ```json
   {
     "targets": ["prod", "staging"],
     "clusters": ["us-east-1-eks", "eu-west-1-eks"],
     "aws_regions": ["us-east-1", "eu-west-1"],
     "time_range": "24h",
     "output_format": "graphviz"
   }
   ```

### Phase 2: Infrastructure Ingestion (45-90 minutes)
1. **Kubernetes Asset Collection**
   ```bash
   # Collect all namespaces
   NAMESPACES=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}')
   
   # Get all workloads
   kubectl get deployments,statefulsets,daemonsets,poddisruptionbudgets,cronjobs,jobs --all-namespaces -o json > k8s-workloads.json
   
   # Get all networking resources
   kubectl get services,endpoints,ingresses,networkpolicies,serviceentries --all-namespaces -o json > k8s-networking.json
   
   # Get configmaps and secrets (redacted)
   kubectl get configmaps,secrets --all-namespaces -o json > k8s-config.json
   ```

2. **AWS Resource Discovery** (per region)
   ```bash
   # EC2 instances and their networking
   aws ec2 describe-instances --region us-east-1 --query 'Reservations[].Instances[].[InstanceId, VpcId, SubnetId, SecurityGroups, IamInstanceProfile]' > ec2-inventory.json
   
   # Load balancers and target groups
   aws elbv2 describe-load-balancers --region us-east-1 --query 'LoadBalancers[].[LoadBalancerArn, VpcId, SubnetIds, SecurityGroups, Scheme]' > elb-inventory.json
   
   # RDS/Aurora clusters
   aws rds describe-db-clusters --region us-east-1 --query 'DBClusters[].[DBClusterIdentifier, VpcSecurityGroups, DBSubnetGroup]' > rds-inventory.json
   
   # Lambda functions
   aws lambda list-functions --region us-east-1 --query 'Functions[].[FunctionName, Runtime, VpcConfig, Role]' > lambda-inventory.json
   
   # API Gateway
   aws apigateway get-rest-apis --region us-east-1 --query 'items[].[id, name, endpointConfiguration.types]' > apigateway-inventory.json
   
   # Resource Explorer search for everything
   aws resource-explorer2 search --query-string "region:us-east-1" --max-results 1000 --output json > aws-resources-us-east-1.json
   ```

3. **Terraform State Analysis**
   ```bash
   # Parse terraform state for resource relationships
   terraform state pull | jq '.resources[] | {address: .address, type: .type, name: .name, instances: .instances}' > tf-state.json
   
   # Generate terraform graph
   terraform graph --type=plan | tee tf-graph.dot
   
   # Extract dependencies from .tf files
   find . -name "*.tf" -exec grep -H "depends_on\|data\." {} \; > tf-dependencies.txt
   ```

4. **Service Mesh Topology** (if Istio)
   ```bash
   # For each service, get proxy config
   for pod in $(kubectl get pods -n istio-system -l app=istio-pilot -o jsonpath='{.items[*].metadata.name}'); do
     istioctl proxy-config clusters $pod -n istio-system --context $CLUSTER_CONTEXT > clusters-$pod.txt
     istioctl proxy-config routes $pod -n istio-system --context $CLUSTER_CONTEXT > routes-$pod.txt
   done
   ```

### Phase 3: Relationship Extraction (60-120 minutes)
1. **Parse Kubernetes Relationships**
   ```python
   # Build dependency graph from K8s resources
   - Extract service account → pod associations
   - Map ConfigMap/Secret mounts to pods
   - Identify service → endpoint links
   - Parse network policies for allowed communication
   - Extract ingress → service routes
   ```

2. **AWS Resource Dependencies**
   ```python
   # Correlate resources
   - Security groups → ENIs → EC2 instances
   - VPC → subnets → resources
   - IAM roles → Lambda/EC2
   - RDS subnet groups → VPC/subnets
   - Load balancer → target groups → instances/lambdas
   - CloudFront distributions → origins (S3/ALB)
   ```

3. **Terraform Dependency Graph**
   ```dot
   # Convert terraform graph to structured format
   digraph G {
     node [shape=box];
     "module.vpc.aws_vpc.main" -> "module.vpc.aws_subnet.public_a";
     "module.eks.aws_eks_cluster.this" -> "module.vpc.aws_vpc.main";
     "module.nodes.aws_launch_template.this" -> "module.eks.aws_eks_cluster.this";
   }
   ```

4. **Cross-System Correlation**
   - Match K8s services to ELB DNS names
   - Correlate Terraform resource IDs to live AWS resources
   - Map container names to CloudWatch log groups
   - Link IAM roles to assumed identities in audit logs

### Phase 4: Graph Construction (30-45 minutes)
```python
# Build unified graph database structure
{
  "nodes": [
    {
      "id": "arn:aws:ec2:us-east-1:123:instance/i-abc123",
      "type": "aws_ec2_instance",
      "properties": {"vpc_id": "vpc-123", "subnet_id": "subnet-456"},
      "system": "aws"
    },
    {
      "id": "k8s://production/api-gateway",
      "type": "kubernetes_service",
      "properties": {"namespace": "production", "port": 8080},
      "system": "k8s"
    }
  ],
  "edges": [
    {
      "source": "k8s://production/api-gateway",
      "target": "arn:aws:elasticloadbalancing:.../targetgroup/tg-123",
      "relationship": "routes_to",
      "protocol": "http",
      "port": 80
    }
  ]
}
```

### Phase 5: Visualization Generation (15-30 minutes)
1. **Generate Graphviz DOT file**
   ```bash
   # Create topology diagram
   infra-illuminator generate-visualization \
     --input unified-graph.json \
     --format dot \
     --group-by system \
     --rank-direction LR \
     --output topology.dot
   
   # Convert to PNG
   dot -Tpng topology.dot -o topology.png
   
   # Generate interactive HTML with D3.js
   infra-illuminator generate-visualization \
     --input unified-graph.json \
     --format html \
     --template d3-force \
     --output topology.html
   ```

2. **Criticality Analysis**
   ```bash
   # Calculate betweenness centrality
   infra-illuminator analyze-criticality \
     --graph unified-graph.json \
     --metric betweenness \
     --top 20 \
     --output critical-nodes.json
   
   # Find critical paths
   infra-illuminator find-critical-paths \
     --from "k8s://prod/payment-service" \
     --to "aws/rds/primary-db" \
     --graph unified-graph.json \
     --output critical-paths.txt
   ```

### Phase 6: Report Generation (15-20 minutes)
```bash
infra-illuminator generate-report \
  --graph unified-graph.json \
  --template html \
  --title "Infrastructure Dependency Map - Production" \
  --include criticality \
  --include impact-analysis \
  --output report-$(date +%Y%m%d).html
```

Report includes:
- Executive summary with critical nodes count
- Interactive dependency graph (zoomable)
- Table of single points of failure
- Service dependency matrix (grid view)
- Change impact predictions
- Drift detection findings

## Golden Rules

1. **Never mutate production infrastructure** - Infra Illuminator is read-only. All commands use `--dry-run` flags or query-only APIs. Never include `--force`, `--delete`, or write operations.

2. **Always redact secrets** - When scanning configuration files and secrets, automatically replace values with `***REDACTED***`. Never log, display, or store raw secret values. Use `kubectl get secret <name> -o jsonpath='{.data}' | base64 -d` only with explicit `--no-redact` flag (disabled by default).

3. **Validate every discovery** - Cross-reference findings from different sources. If a K8s service claims to connect to `db.internal` but AWS shows no route, flag as "unverified". Require at least 2 data sources for critical path identification.

4. **Respect cluster boundaries** - Never assume cross-cluster communication exists without explicit evidence from service mesh config or DNS records. Tag inferred connections as `confidence: low`.

5. **Rate limit API calls** - Implement exponential backoff for AWS/Terraform Cloud APIs. Maximum 10 requests/second per account. Cache all responses for 5 minutes to avoid rate limits on large inventories.

6. **Handle partial failures gracefully** - If only 3 out of 5 clusters respond, generate a report with `completeness: 60%` and clear warnings. Never fail entirely due to one unreachable cluster.

7. **Encrypt all output files** - Dependency graphs contain sensitive topology. If `--sensitive` flag not set, automatically encrypt output with `gpg --symmetric --cipher-algo AES256`. Key derived from `INFRA_ILLUMINATOR_GPG_KEY` env var.

8. **Never assume default ports** - When mapping service connections, extract actual port numbers from k8s Service spec or AWS Target Group, not from protocol defaults. A "HTTP service" might run on port 8080, not 80.

9. **Disable by default in prod clusters** - The skill must check `kubectl config current-context` and refuse to run if context contains `-prod` or `-production` unless explicitly passed `--ignore-warnings --force`. Production scans must be scheduled during maintenance windows.

10. **Maintain audit trail** - Log every discovery command to `$INFRA_ILLUMINATOR_LOG_FILE` (default: `/var/log/infra-illuminator/audit.log`) with timestamp, user, and command. Never disable logging. Use `--quiet` only for stdout, never for audit log.

## Examples

### Example 1: Map K8s service dependencies

**Prompt:**
```
infra-illuminator map-dependencies \
  --namespace payment \
  --include-ingress \
  --format graphviz \
  --output payment-topology.dot
```

**Internal Commands Run:**
```bash
# 1. Get all resources in payment namespace
kubectl get all,cm,secret,sa,pvc,networkpolicy,ingress -n payment -o json > payment-runtime.json

# 2. Get Terraform state for payment namespace resources
terraform state list | grep "payment" | xargs -I{} terraform state show {} --json > payment-terraform.json

# 3. Extract service endpoints
kubectl get endpoints -n payment -o json | jq -r '.items[] | "\(.metadata.name): \(.subsets[].addresses[].ip):\(.subsets[].ports[].port)"'

# 4. Get ingress rules
kubectl get ingress -n payment -o json | jq -r '.items[] | {host: .spec.rules[].host, paths: .spec.rules[].http.paths}'

# 5. Generate DOT
cat << 'EOF' > payment-topology.dot
digraph payment_dependencies {
  rankdir=LR;
  node [shape=box, style=filled, fillcolor=lightblue];
  
  "payment-api" -> "payment-db" [label="TCP/5432"];
  "payment-api" -> "redis-session" [label="TCP/6379"];
  "payment-api" -> "queue-payments" [label="AMQP/5672"];
  "queue-payments" -> "ledger-service" [label="AMQP/5672"];
  "payment-api" -> "stripe-gateway" [label="HTTPS/443"];
  "ingress/nginx" -> "payment-api" [label="HTTP/80"];
}
EOF
```

**Output:**
```
✓ Discovered 12 K8s resources in namespace 'payment'
✓ Found 5 service dependencies
✓ Generated Graphviz file: payment-topology.dot
📊 Nodes: 6, Edges: 5
⚠️  WARNING: payment-api exposes port 8080 directly (no ingress) - potential security issue
```

---

### Example 2: Find all paths to critical database

**Prompt:**
```
infra-illuminator find-paths \
  --target "arn:aws:rds:us-east-1:123:db:prod-primary-db" \
  --include-transitive \
  --max-depth 5 \
  --format ascii \
  --output paths-to-db.txt
```

**Internal Commands Run:**
```bash
# 1. Get RDS instance details
aws rds describe-db-instances --db-instance-identifier prod-primary-db --query 'DBInstances[0]' > target-db.json

# 2. Find all security groups referencing this DB's SG
aws ec2 describe-security-groups --filters Name=ip-permission.group-id,Values=$(jq -r '.VpcSecurityGroups[0].VpcSecurityGroupId' target-db.json) --query 'SecurityGroups[].GroupId' > sgs.json

# 3. Find EC2 instances with those SGs
SG_IDS=$(jq -r '.[]' sgs.json | paste -sd, -)
aws ec2 describe-instances --filters Name=instance.group-id,Values=$SG_IDS --query 'Reservations[].Instances[].[InstanceId, Tags[?Key==`Name`].Value|[0]]' > ec2-instances.json

# 4. Find Lambda functions in VPC
aws lambda list-functions --query "Functions[?VpcConfig.SecurityGroupIds[]==\`$SG_ID\`].FunctionName" > lambdas.txt

# 5. Find EKS services with matching environment variables
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.spec.containers[].env[]?.valueFrom?.secretKeyRef?.name | contains("prod-primary-db")) | "\(.metadata.namespace)/\(.metadata.name)"' > k8s-pods.txt

# 6. Build graph and find paths
infra-illuminator build-graph --sources ec2-instances.json,lambdas.txt,k8s-pods.txt --target target-db.json --max-hops 5 | dot -Tascii > paths-to-db.txt
```

**Output:**
```
Paths to RDS instance prod-primary-db (max depth: 5):

1. EKS service: payment-api (prod/payment-api-7d8f5) 
   → pod uses secret: db-prod-credentials
   → SECRET contains: host=prod-primary-db.c6q8syq3.us-east-1.rds.amazonaws.com
   Depth: 1 | Risk: HIGH (direct connection, no proxy)

2. EC2 instance: i-0a1b2c3d4e5f6g7h (monitoring-worker)
   → Security Group: sg-monitoring (allows outbound to DB SG on 5432)
   → Process: pgbouncer running, connection pool configured
   Depth: 2 | Risk: MEDIUM (connection pool provides some isolation)

3. Lambda: arn:aws:lambda:us-east-1:123:function:report-generator
   → VPC: vpc-12ab34cd
   → Security Group: sg-lambda (egress to DB SG)
   → ENIs dynamically created in subnets with route to DB
   Depth: 3 | Risk: HIGH (Lambda cold starts cause connection spikes)

Found 23 total paths. Top 3 shown. Full list: paths-to-db.txt
```

---

### Example 3: Detect infrastructure drift

**Prompt:**
```
infra-illuminator detect-drift \
  --terraform-dir ./terraform/production \
  --k8s-context prod-cluster \
  --aws-regions us-east-1,eu-west-1 \
  --report html \
  --output drift-report-$(date +%Y%m%d).html
```

**Internal Commands Run:**
```bash
# 1. Get Terraform planned changes (if any)
cd ./terraform/production && terraform plan -out=tfplan
terraform show -json tfplan | jq '.planned_values.root_module.resources[] | {address, type, values}' > tf-planned.json
terraform state pull | jq '.resources[] | {address, type, values}' > tf-current.json

# 2. Get live K8s resources not in Terraform
kubectl get all,cm,secret,pvc --all-namespaces -o json > k8s-live.json
# Compare with Terraform data sources that reference K8s
grep -r "kubernetes_" ./terraform/production | cut -d: -f1 | sort -u > tf-k8s-refs.txt

# 3. Get AWS resources not tracked in Terraform
aws resource-explorer2 search --query-string "region:us-east-1 AND -tag:ManagedBy:Terraform" --max-results 1000 > aws-untagged.json

# 4. Compare security groups
terraform state pull | jq -r '.resources[] | select(.type=="aws_security_group") | .attributes.id' > tf-sgs.txt
aws ec2 describe-security-groups --query 'SecurityGroups[?tag_keys[?contains(@, `ManagedBy`) && `Terraform` != .Tags[?Key==`ManagedBy`].Value]].GroupId' > aws-untracked-sgs.json

# 5. Generate drift report
infra-illuminator build-drift-report \
  --terraform tf-current.json \
  --planned tf-planned.json \
  --k8s k8s-live.json \
  --aws-untagged aws-untagged.json \
  --template html > drift-report.html
```

**Output:**
```
✓ Terraform state collected (247 resources)
✓ K8s live resources collected (183 resources)
✓ AWS untagged resources discovered (12 resources)

Drift Analysis:
────────────────────────────────────
Category                    Count  Severity
────────────────────────────────────
K8s resources not in TF      7     HIGH
K8s resources in TF missing 3     MEDIUM
AWS resources untagged      12     HIGH
Security group rule drift  24     MEDIUM
Load balancer config drift  5     MEDIUM
────────────────────────────────────

🚨 CRITICAL: 19 high-severity drift items found
📊 Matching resources: 89% (TF ↔ Live)
🔍 Full report: drift-report-20260304.html
```

---

### Example 4: Generate dependency graph for cost allocation

**Prompt:**
```
infra-illuminator generate-cost-graph \
  --aws-regions us-east-1 \
  --monthly-cost-threshold 100 \
  --group-by team \
  --output cost-dependencies.png
```

**Internal Commands Run:**
```bash
# 1. Get Cost Explorer data
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '1 month ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[].Groups[] | {service: .Keys[0], cost: .Metrics.BlendedCost.Amount}' \
  --output json > aws-costs.json

# 2. Map services to resources
for service in $(jq -r '.[].service' aws-costs.json | sort -u); do
  case $service in
    "Amazon Elastic Compute Cloud")
      aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId, Tags[?Key==`Team`].Value|[0]]' --output json > ec2-team-mapping.json
      ;;
    "Amazon Simple Storage Service")
      aws s3api list-buckets --query 'Buckets[].[Name, Tags]' --output json > s3-team-mapping.json
      ;;
    "Amazon Relational Database Service")
      aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier, Tags]' --output json > rds-team-mapping.json
      ;;
  esac
done

# 3. Build cost allocation graph
infra-illuminator build-cost-graph \
  --costs aws-costs.json \
  --mappings ec2-team-mapping.json,s3-team-mapping.json,rds-team-mapping.json \
  --threshold 100 \
  --format png \
  --output cost-dependencies.png
```

**Output:**
```
✓ Cost data collected for 28 AWS services
✓ Resource → team mapping: 1,247 resources
✓ Total monthly cost: $45,672.34

Cost Centers (top 5):
1. platform-team     $18,234.12 (EC2, RDS, S3)
2. analytics-team    $12,456.78 (Glue, Redshift, S3)
3. frontend-team     $8,123.45 (CloudFront, S3, Lambda)
4. mobile-team       $4,567.89 (API Gateway, Lambda, SNS)
5. devops-team       $2,290.10 (ECS, EKS, ECR)

📊 Generated: cost-dependencies.png
```

---

### Example 5: Real-time service health topology

**Prompt:**
```
infra-illuminator health-topology \
  --prometheus-url http://prometheus-prod:9090 \
  --k8s-context prod-us-east-1 \
  --time-window 30m \
  --degraded-threshold 0.95 \
  --output health-topology.html
```

**Internal Commands Run:**
```bash
# 1. Query Prometheus for service health
curl -s "http://prometheus-prod:9090/api/v1/query?query=rate(http_requests_total[5m])" > request-rates.json
curl -s "http://prometheus-prod:9090/api/v1/query?query=rate(http_requests_errors_total[5m])" > error-rates.json
curl -s "http://prometheus-prod:9090/api/v1/query?query=histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))" > p95-latency.json

# 2. Get current K8s pod status
kubectl get pods --all-namespaces -o json > k8s-pods.json

# 3. Correlate services
# Match Prometheus `instance` label to K8s pod IP
for pod in $(jq -r '.items[] | "\(.status.podIP) \(.metadata.namespace)/\(.metadata.name)"' k8s-pods.json); do
  ip=$(echo $pod | cut -d' ' -f1)
  name=$(echo $pod | cut -d' ' -f2)
  echo "$ip -> $name"
done > pod-ip-mapping.txt

# 4. Build health graph
infra-illuminator build-health-graph \
  --request-rates request-rates.json \
  --error-rates error-rates.json \
  --latency p95-latency.json \
  --pod-mapping pod-ip-mapping.txt \
  --threshold 0.95 \
  --format html \
  --output health-topology.html
```

**Output:**
```
✓ Prometheus queries executed (3 time series)
✓ K8s pod mapping: 1,024 pods
✓ Health status computed:

Service Health Summary:
─────────────────────────────────────────────
Service                     Status    P95 Latency  Error Rate  Nodes
─────────────────────────────────────────────
payment-api                 DEGRADED  320ms        2.1%        5/6
user-api                    HEALTHY   45ms         0.0%        8/8
order-api                   DEGRADED  890ms        5.3%        3/4 ⚠️
inventory-service           HEALTHY   12ms         0.0%        3/3
notification-worker         HEALTHY   2ms          0.0%        12/12
─────────────────────────────────────────────

🚨 2 services degraded (threshold: 0.95 success rate)
📊 Generated interactive health map: health-topology.html
```

---

### Example 6: Emergency incident response - impact analysis

**Prompt:**
```
infra-illuminator incident-impact \
  --affected-resource "k8s://prod/payment-api" \
  --time-window 1h \
  --prometheus-url http://prometheus:9090 \
  --output incident-impact-$(date +%s).json
```

**Internal Commands Run:**
```bash
# 1. Identify all downstream services from traffic logs
curl -s "http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{destination_service=\"payment-api\"}[1h])) by (source_service)" > downstream.json

# 2. Check for synchronous dependencies
kubectl get endpoints -n prod -o json | jq -r '.items[] | select(.subsets[].addresses[].targetRef.name=="payment-api") | .metadata.name' > direct-downstream.txt

# 3. Find async dependencies (queues, topics)
kubectl get queues,topics --all-namespaces -o json | jq -r '.items[] | select(.spec.consumers[]?.name=="payment-api") | "\(.kind)/\(.metadata.namespace)/\(.metadata.name)"' > async-dependencies.txt

# 4. Database dependencies
kubectl get pods -n prod -o json | jq -r '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -i db | sort -u > db-secrets.txt

# 5. Generate impact report
infra-illuminator generate-impact-report \
  --resource "k8s://prod/payment-api" \
  --downstream downstream.json \
  --direct direct-downstream.txt \
  --async async-dependencies.txt \
  --databases db-secrets.txt \
  --template json \
  --output incident-impact-$(date +%s).json
```

**Output:**
```
🚨 INCIDENT RESPONSE: Impact analysis for payment-api
────────────────────────────────────────────────────────
Time window: 1h | Generated: $(date)

DOWNSTREAM SERVICES (synchronous):
• order-api (45,231 req/min) - CRITICAL
• checkout-service (12,845 req/min) - CRITICAL
• user-wallet (3,210 req/min) - HIGH

ASYNC DEPENDENCIES:
• queue:payment-confirmations (30K msg/min) → notification-worker
• topic:payment-events → analytics-pipeline, fraud-detector

DATABASES:
• Secret: prod-payment-db-creds → RDS: payment-primary
• Secret: prod-payment-read-replica → RDS: payment-read-replica-1

BUSINESS IMPACT:
✗ Users cannot complete checkout
✗ Payment confirmations delayed
✗ Wallet balance not updating
✓ Analytics still processing (using cached data)

RECOMMENDED ACTIONS:
1. Enable circuit breaker in order-api
2. Route read queries to payment-read-replica
3. Push notification: "Payment service temporarily degraded"

📦 Detailed report: incident-impact-1646324567.json
```

---

### Example 7: Visualize network policy mesh

**Prompt:**
```
infra-illuminator visualize-network-policies \
  --namespace istio-system \
  --include-ingress-egress \
  --format html-d3 \
  --output network-mesh.html
```

**Internal Commands Run:**
```bash
# 1. Get all network policies
kubectl get networkpolicy --all-namespaces -o json > np-all.json

# 2. Get pod labels for matching
kubectl get pods --all-namespaces -o json | jq -r '.items[] | {namespace: .metadata.namespace, name: .metadata.name, labels: .metadata.labels}' > pod-labels.json

# 3. Build graph
infra-illuminator build-network-graph \
  --policies np-all.json \
  --pods pod-labels.json \
  --namespace istio-system \
  --format d3-force \
  --output network-mesh.html
```

**Output (HTML file with interactive D3.js visualization showing:**
- **Pods as circles** colored by namespace
- **Ingress rules as green arrows** (allow traffic)
- **Egress rules as blue arrows**
- **Denied connections as red dashed lines**
- **Hover to see policy name and selector**
- **Drag nodes to rearrange**
- **Search/filter by namespace**
- **Export to PNG**

**Summary printed to console:**
```
✓ Analyzed 47 network policies across 12 namespaces
✓ Identified 234 allowed connections, 89 denials
⚠️  3 pods with no policies (allow all by default)
🔒 98% of traffic governed by explicit policies
📊 Generated: network-mesh.html (interactive D3.js)
```

---

## Troubleshooting

### Issue: "Error: context deadline exceeded" when querying AWS
**Symptoms:** `aws ec2 describe-instances` hangs or times out after 2 minutes.

**Diagnosis:**
```bash
# Check AWS CLI connectivity
time aws ec2 describe-regions --max-items 1

# Check network latency to AWS API endpoint
curl -o /dev/null -s -w "DNS: %{time_namelookup}s Connect: %{time_connect}s Total: %{time_total}s\n" https://ec2.us-east-1.amazonaws.com

# Check AWS credentials expiration
aws sts get-caller-identity --query 'Account' 2>&1
```

**Resolution:**
1. Set explicit region and reduce parallelism: `aws configure set region us-east-1 && aws configure set cli_fallback_region us-east-1`
2. Increase timeout: `export AWS_METADATA_SERVICE_TIMEOUT=10 && export AWS_MAX_ATTEMPTS=10`
3. Use AWS SSO session if MFA expired: `aws sso login --profile production`

---

### Issue: "kubectl: error: the server doesn't have a resource type" for custom resources
**Symptoms:** `kubectl get virtualservices` fails with "the server doesn't have a resource type"

**Diagnosis:**
```bash
# Check if CRD is installed
kubectl get crd virtualservices.networking.istio.io

# Check API versions available
kubectl api-resources | grep virtual
kubectl api-versions | grep istio

# Verify kubectl context
kubectl config current-context
kubectl get ns istio-system
```

**Resolution:**
1. Install missing CRD: `kubectl apply -f https://github.com/istio/istio/releases/download/1.19.0/istio-crds.yaml`
2. Use correct context: `kubectl config use-context prod-eks`
3. Use explicit API version: `kubectl get virtualservices.networking.istio.io --all-namespaces`

---

### Issue: "Terraform state file is locked" error
**Symptoms:** `terraform state pull` fails with "Error: Error locking state"

**Diagnosis:**
```bash
# Check for terraform state lock
terraform state list 2>&1 | grep "state is locked"

# Find stale lock file
find . -name ".terraform.tfstate.lock.info" -ls

# Check for other terraform processes
ps aux | grep terraform
```

**Resolution:**
1. **Never force unlock** without verifying no other process is running.
2. If safe: `terraform force-unlock <LOCK_ID>`
3. Prevent future locks: Set `-input=false` in CI, use remote state backend with proper locking (S3+DynamoDB).

---

### Issue: Graphviz rendering produces empty/blank image
**Symptoms:** `dot -Tpng topology.dot -o topology.png` creates 0-byte file or blank image

**Diagnosis:**
```bash
# Check DOT syntax
dot -Tdot topology.dot -o /dev/null 2>&1 | head -20

# Validate graph structure
grep -c "->" topology.dot  # Should have edges
grep "digraph" topology.dot  # Should have digraph declaration

# Check dot version
dot -V
```

**Resolution:**
1. Ensure graph has nodes: `grep "\[label=" topology.dot | wc -l`
2. Add rank direction: `rankdir=LR;` for left-to-right layout
3. For large graphs (>10k nodes), use `neato` instead of `dot`: `neato -Tpng -Goverlap=scalexy topology.dot -o topology.png`
4. Increase memory limit: `export GVBINDIR=/usr/lib/graphviz && dot -Tpng -Gmaxiter=1000 topology.dot -o topology.png`

---

### Issue: Prometheus query returns "parse error" or empty data
**Symptoms:** `curl "http://prometheus:9090/api/v1/query?query=rate(http_requests_total[5m])"` returns `{"status":"error","errorType":"bad_data","error":"parse error..."}`

**Diagnosis:**
```bash
# Verify Prometheus is healthy
curl -s http://prometheus:9090/-/healthy

# Check exact metric name
curl -s "http://prometheus:9090/api/v1/label/__name__/values" | jq '.data[]' | grep http_requests

# Test query in Prometheus UI first
# http://prometheus:9090/graph?g0.expr=rate(http_requests_total%5B5m%5D)&g0.tab=0
```

**Resolution:**
1. URL-encode query brackets: `rate(http_requests_total[5m])` → `rate(http_requests_total%5B5m%5D)`
2. Use correct metric name (case sensitive): `http_requests_total` not `http_requests`
3. Ensure time range is long enough: `[5m]` requires at least 5 minutes of data
4. Use promtool for validation: `promtool query instant "rate(http_requests_total[5m])"`

---

### Issue: "Found X unmapped resources" warning
**Symptoms:** Warnings about resources that couldn't be correlated between systems.

**Diagnosis:**
```bash
# Check resource identifiers
jq '.unmapped[] | {aws_id: .aws_resource_id, k8s_name: .k8s_name}' unmapped-resources.json | head -20

# Search by common patterns
grep -i "database" unmapped-resources.json
grep "rds-" unmapped-resources.json

# Verify Terraform data sources
terraform state list | grep rds
```

**Resolution:**
1. Accept some unmapped resources are legitimate (e.g., manually created test instances)
2. Add custom mapping rules in `~/.infra-illuminator/mappings.yaml`:
   ```yaml
   custom_mappings:
     - pattern: "rds-.*"
       type: "aws_rds_instance"
       map_to: "k8s://*/database-*"
   ```
3. Use `--ignore-patterns "test-.*,dev-.*"` to exclude dev/test resources

---

## Rollback

### Rollback Procedures

**Scenario 1: Accidental write operation (should never happen)**
```bash
# If infra-illuminator mistakenly modified any resource:

# 1. Stop immediately
pkill -f infra-illuminator

# 2. Review audit log for executed commands
grep "$(date +%Y-%m-%d)" /var/log/infra-illuminator/audit.log | grep -v "read-only"

# 3. Identify changed resources
# For AWS:
aws configservice get-compliance-details-by-config-rule --config-rule-name "infra-illuminator-drift-detection" --compliance-type NON_COMPLIANT

# For K8s:
kubectl get events --sort-by='.lastTimestamp' | grep -i "modified\|created"

# 4. Restore from terraform if under management
terraform apply -target=<resource_address>  # Re-apply desired state

# 5. Manual rollback for unmanaged resources
#   - EC2: Revert security group rules from backup JSON
#   - K8s: `kubectl rollout undo deployment/<name>`
#   - RDS: Use snapshot restore

# 6. Verify restoration
infra-illuminator validate-restoration --since 1h
```

**Scenario 2: Generated graph file corrupted**
```bash
# Restore from backup (auto-created with .bak suffix)
ls -la topology.dot*
# -rw-r--r-- 1 user user 12345 Mar  4 10:30 topology.dot
# -rw-r--r-- 1 user user 12340 Mar  4 10:29 topology.dot.bak

# Restore
cp topology.dot.bak topology.dot

# Regenerate
infra-illuminator generate-visualization --input unified-graph.json --format dot --output topology.dot
```

**Scenario 3: Unintended discovery of sensitive data**
```bash
# If secret values leaked to output files:

# 1. Securely delete compromised files
shred -v -n 3 -z output-with-secrets.json
rm -f output-with-secrets.json

# 2. Rotate exposed secrets immediately
# - AWS: aws secretsmanager rotate-secret --secret-id exposed-secret
# - K8s: kubectl create secret generic new-secret --dry-run=client -o yaml | kubectl apply -f - && rollout restart deployment/<that-uses-it>

# 3. Review git history for accidental commits
git log --oneline --all -- output-with-secrets.json

# 4. If committed, use BFG or filter-branch to purge
```

**Scenario 4: Performance degradation on production cluster**
```bash
# If infra-illuminator caused load issues:

# 1. Stop all scans
pkill -INT -f infra-illuminator

# 2. Throttle future runs
echo "max_concurrent_scans: 2
rate_limit_rps: 5
batch_size: 50" > ~/.infra-illuminator/limits.yaml

# 3. Schedule scans during off-peak
# Edit cron: 0 2 * * * infra-illuminator full-scan  # 2 AM instead of every hour

# 4. Add resource quotas
kubectl create limitrange infra-illuminator-limit --dry-run=client -o yaml --limit-range='{"limits": [{"type":"Container","default":{"memory":"512Mi","cpu":"500m"}}]}' -n infra-illuminator | kubectl apply -f -
```

**Scenario 5: Graph database corruption**
```bash
# If unified-graph.json becomes invalid:

# 1. Rebuild from raw data (takes 2-4 hours)
rm -f unified-graph.json
infra-illuminator ingest \
  --k8s-clusters prod-eks,staging-eks \
  --aws-regions us-east-1,eu-west-1 \
  --terraform-states ./terraform/production/terraform.tfstate

# 2. Validate rebuilt graph
jq empty unified-graph.json && echo "✓ Valid JSON" || echo "✗ Corrupted"

# 3. Take snapshot before destructive operations
cp unified-graph.json "backups/unified-graph-$(date +%Y%m%d-%H%M%S).json"
```

### Complete System Rollback

**To revert all infra-illuminator configuration changes:**
```bash
# 1. Stop service
systemctl stop infra-illuminator.timer
systemctl stop infra-illuminator.service

# 2. Remove configuration files
rm -rf ~/.infra-illuminator/
rm -f /etc/infra-illuminator/config.yaml

# 3. Uninstall dependencies (careful!)
# Only if installed by infra-illuminator:
pip3 uninstall -y networkx pydot python-igraph
brew uninstall graphviz  # macOS

# 4. Restore from backup (if available)
if [ -d /var/backups/infra-illuminator-$(date +%Y%m%d) ]; then
  cp -r /var/backups/infra-illuminator-$(date +%Y%m%d)/* ~/.infra-illuminator/
fi

# 5. Reinstall clean version from package manager
# Debian/Ubuntu:
apt-get install --reinstall infra-illuminator
# macOS:
brew reinstall infra-illuminator

# 6. Verify
infra-illuminator --version
infra-illuminator health-check
```

**Important:** Infra Illuminator is read-only by design. True rollback only needed for:
- Corrupted output files (restore from backup)
- Misconfigured limits (fix config and restart)
- Accidental write operations (use source control/Terraform)
- Performance issues (throttle/schedule)
- Security breach (rotate secrets, purge logs)

Never run rollback commands without first verifying the issue and backing up current state.

## Verification

After running any infra-illuminator command:

1. **Check exit code**: `echo $?` should be `0`
2. **Verify output file exists and is valid**:
   ```bash
   ls -lh topology.dot topology.png
   file topology.png  # Should say: PNG image data, 1920 x 1080
   jq . unified-graph.json 2>/dev/null || echo "Invalid JSON"
   ```
3. **Validate node count**:
   ```bash
   grep -c "->" topology.dot  # Should have edges
   jq '.nodes | length' unified-graph.json  # Should be > 0
   ```
4. **Run integrity check**:
   ```bash
   infra-illuminator verify-graph --input unified-graph.json
   # Should output: ✓ Graph verified: 1,247 nodes, 3,456 edges, 0 orphaned nodes
   ```

5. **Cross-reference critical nodes**:
   ```bash
   infra-illuminator list-critical-nodes --graph unified-graph.json --threshold 0.8
   # Should list known critical services: api-gateway, payment-db, auth-service
   ```

6. **Check audit log**:
   ```bash
   tail -50 /var/log/infra-illuminator/audit.log | grep "$(date +%Y-%m-%d)"
   # Should show commands with timestamps, no errors
   ```

7. **Test visualization renders correctly**:
   ```bash
   # For HTML output
   grep -q "<html" topology.html && echo "✓ HTML valid"
   
   # For PNG
   identify topology.png 2>/dev/null || echo "✗ PNG corrupted"
   
   # For DOT
   dot -Tdot topology.dot -o /dev/null 2>&1 | grep -q "graph" && echo "✓ DOT valid"
   ```
```