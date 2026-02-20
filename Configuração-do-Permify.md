# Configuração do Permify

Este documento detalha a configuração do Permify em diferentes ambientes.

## Configuração local (Docker Compose)

### Arquivo de configuração

`permify/config.yaml`

```yaml
server:
  http:
    enabled: true
    port: 3476
  grpc:
    port: 3478

logger:
  level: info

authn:
  enabled: false

database:
  engine: postgres
  uri: postgres://permify:permify@permify-db:5432/permify?sslmode=disable
  auto_migrate: true
  max_connections: 20

distributed:
  enabled: false
```

### Docker Compose

`docker-compose.yml`

```yaml
version: '3.8'

services:
  permify-db:
    image: postgres:15
    environment:
      POSTGRES_DB: permify
      POSTGRES_USER: permify
      POSTGRES_PASSWORD: permify
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U permify"]
      interval: 10s
      timeout: 5s
      retries: 5

  permify:
    image: ghcr.io/permify/permify:latest
    command: ["serve", "--config=/config/config.yaml"]
    volumes:
      - ./permify/config.yaml:/config/config.yaml:ro
    ports:
      - "3476:3476"  # HTTP
      - "3478:3478"  # gRPC
    depends_on:
      permify-db:
        condition: service_healthy
    environment:
      - PERMIFY_LOG_LEVEL=info

volumes:
  postgres_data:
```

## Configuração no BFF (NestJS)

### Variáveis de ambiente

**Local** (`clickbus-platform-bff/config/local.ts`):

```typescript
export default () => ({
  permify: {
    enabled: true,
    baseUrl: 'http://127.0.0.1:3476',
    tenantId: 'dev',
    depth: 20,
    timeoutMs: 1500,
    checkCacheTtlMs: 2000,
  },
});
```

**Produção** (via env vars):

```bash
PERMIFY_ENABLED=true
PERMIFY_BASE_URL=https://permify.internal.clickbus.net
PERMIFY_TENANT_ID=production
PERMIFY_DEPTH=20
PERMIFY_TIMEOUT_MS=1500
PERMIFY_CHECK_CACHE_TTL_MS=2000
```

### Descrição das variáveis

- **`PERMIFY_ENABLED`**: Habilita/desabilita o Permify
  - `true`: Faz checks reais no Permify
  - `false`: Bypass (sempre retorna ALLOW)

- **`PERMIFY_BASE_URL`**: URL base do Permify
  - Local: `http://127.0.0.1:3476`
  - Produção: `https://permify.internal.clickbus.net`

- **`PERMIFY_TENANT_ID`**: ID do tenant
  - Local: `dev`
  - Produção: `production`

- **`PERMIFY_DEPTH`**: Profundidade máxima de resolução de permissões
  - Default: `20`
  - Aumentar se houver muitas relações aninhadas

- **`PERMIFY_TIMEOUT_MS`**: Timeout para chamadas ao Permify
  - Default: `1500` (1.5s)

- **`PERMIFY_CHECK_CACHE_TTL_MS`**: TTL do cache local de checks
  - Default: `2000` (2s)
  - Cache em memória no BFF para reduzir latência

## Tenants

### Conceito

Tenants isolam schemas e tuples. Cada tenant tem:
- Schema próprio
- Tuples próprios
- Versionamento independente

### Estratégias

**1) Tenant por Ambiente (recomendado)**

```
tenant: dev        ← ambiente de desenvolvimento
tenant: staging    ← ambiente de homologação
tenant: production ← ambiente de produção
```

**2) Tenant por Organização (multi-tenant real)**

```
tenant: org-1
tenant: org-2
tenant: org-3
```

### Criar tenant

Tenants são criados automaticamente ao escrever o primeiro schema:

```bash
curl -X POST 'http://localhost:3476/v1/tenants/my-tenant/schemas/write' \
  -H 'Content-Type: application/json' \
  -d '{"schema": "entity organization { ... }"}'
```

## Schema Management

### Estrutura de diretórios

```
permify/
├── schemas/
│   ├── base.permify          # Schema base (organization, company, module)
│   ├── insights/
│   │   └── insights.permify  # Schema do módulo insights
│   └── finance/
│       └── finance.permify   # Schema do módulo finance
├── seeds/
│   ├── base.sh              # Seeds base (orgs, companies)
│   ├── insights.sh          # Seeds do módulo insights
│   └── finance.sh           # Seeds do módulo finance
└── config.yaml
```

### Schema base

`permify/schemas/base.permify`

```permify
entity user {}

entity group {
  relation admin @user
  relation member @user
  permission manage = admin
  permission view = admin or member
}

entity organization {
  relation admin @user
  relation manager @user
  relation member @user
  permission manage = admin
  permission administrate = admin or manager
  permission access = admin or manager or member
}

entity company {
  relation organization @organization
  relation admin @user
  relation manager @user
  relation member @user
  permission manage = admin or organization.admin
  permission administrate = admin or manager or organization.admin or organization.manager
  permission access = admin or manager or member or organization.admin or organization.manager or organization.member
}

entity team {
  relation organization @organization
  relation company @company
  relation lead @user
  relation member @user
  permission manage = lead or company.admin or company.manager or organization.admin or organization.manager
  permission view = lead or member or company.admin or company.manager or organization.admin or organization.manager
}

entity module {
  relation organization @organization
  relation company @company
  relation owner_user @user
  relation editor_user @user
  relation viewer_user @user
  relation guest_user @user
  relation owner_group @group
  relation editor_group @group
  relation viewer_group @group
  relation guest_group @group
  relation owner_team @team
  relation editor_team @team
  relation viewer_team @team
  relation guest_team @team

  permission owner = owner_user or owner_group.member or owner_team.member
  permission editor = editor_user or editor_group.member or editor_team.member
  permission viewer = viewer_user or viewer_group.member or viewer_team.member
  permission guest = guest_user or guest_group.member or guest_team.member

  permission view = guest or viewer or editor or owner or organization.admin or organization.manager or company.admin or company.manager or company.organization.admin or company.organization.manager
  permission edit = editor or owner or organization.admin or organization.manager or company.admin or company.manager or company.organization.admin or company.organization.manager
  permission manage = owner or organization.admin or company.admin or company.organization.admin
  permission delete = organization.admin or company.admin or company.organization.admin
}
```

### Deploy de schema

```bash
#!/bin/bash
# deploy-schema.sh

TENANT=${1:-dev}
SCHEMA_FILE=${2:-base.permify}
PERMIFY_URL=${PERMIFY_URL:-http://localhost:3476}

echo "Deploying schema $SCHEMA_FILE to tenant $TENANT..."

curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/schemas/write" \
  -H "Content-Type: application/json" \
  -d "{\"schema\": $(cat $SCHEMA_FILE | jq -Rs .)}"

echo "Schema deployed successfully!"
```

## Seeds (Dados Iniciais)

### Seed base

`permify/seeds/base.sh`

```bash
#!/bin/bash

TENANT=${PERMIFY_TENANT_ID:-dev}
PERMIFY_URL=${PERMIFY_URL:-http://localhost:3476}

echo "Seeding base data for tenant $TENANT..."

# Organização ClickBus
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "organization", "id": "clickbus"},
      "relation": "admin",
      "subject": {"type": "user", "id": "carlos"}
    }
  }'

# Organização Santa Cruz
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "organization", "id": "santa-cruz"},
      "relation": "admin",
      "subject": {"type": "user", "id": "bob"}
    }
  }'

# Company Santa Cruz 1
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "company", "id": "sc1"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "santa-cruz"}
    }
  }'

curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "company", "id": "sc1"},
      "relation": "admin",
      "subject": {"type": "user", "id": "bob"}
    }
  }'

echo "Base seed completed!"
```

### Seed de módulo

`permify/seeds/insights.sh`

```bash
#!/bin/bash

TENANT=${PERMIFY_TENANT_ID:-dev}
PERMIFY_URL=${PERMIFY_URL:-http://localhost:3476}

echo "Seeding insights module for tenant $TENANT..."

# Módulo Insights (vinculado à ClickBus)
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "module", "id": "insights"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "clickbus"}
    }
  }'

# Owner do módulo
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "module", "id": "insights"},
      "relation": "owner_user",
      "subject": {"type": "user", "id": "carlos"}
    }
  }'

# Editor do módulo
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "module", "id": "insights"},
      "relation": "editor_user",
      "subject": {"type": "user", "id": "alice"}
    }
  }'

echo "Insights seed completed!"
```

## Produção (Kubernetes)

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: permify
  namespace: authz
spec:
  replicas: 3
  selector:
    matchLabels:
      app: permify
  template:
    metadata:
      labels:
        app: permify
    spec:
      containers:
      - name: permify
        image: ghcr.io/permify/permify:v0.2.0
        ports:
        - containerPort: 3476
          name: http
        - containerPort: 3478
          name: grpc
        env:
        - name: PERMIFY_DATABASE_URI
          valueFrom:
            secretKeyRef:
              name: permify-secrets
              key: database-uri
        - name: PERMIFY_LOG_LEVEL
          value: "info"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /v1/health
            port: 3476
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /v1/ready
            port: 3476
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: permify-config
---
apiVersion: v1
kind: Service
metadata:
  name: permify
  namespace: authz
spec:
  selector:
    app: permify
  ports:
  - name: http
    port: 3476
    targetPort: 3476
  - name: grpc
    port: 3478
    targetPort: 3478
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: permify-config
  namespace: authz
data:
  config.yaml: |
    server:
      http:
        enabled: true
        port: 3476
      grpc:
        port: 3478
    
    logger:
      level: info
    
    authn:
      enabled: false
    
    database:
      engine: postgres
      uri: ${PERMIFY_DATABASE_URI}
      auto_migrate: true
      max_connections: 50
    
    distributed:
      enabled: false
```

### Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: permify-secrets
  namespace: authz
type: Opaque
data:
  database-uri: cG9zdGdyZXM6Ly9wZXJtaWZ5OnBhc3N3b3JkQHBvc3RncmVzLmF1dGh6LnN2Yy5jbHVzdGVyLmxvY2FsOjU0MjIvcGVybWlmeT9zcG1vZGU9cmVxdWlyZQ==
```

## Monitoramento

### Métricas

O Permify expõe métricas no endpoint `/v1/metrics`:

```bash
curl http://localhost:3476/v1/metrics
```

Métricas importantes:
- `permify_check_duration_seconds` - Latência dos checks
- `permify_check_total` - Total de checks (com labels decision, tenant)
- `permify_cache_hit_ratio` - Cache hit ratio
- `permify_database_connections_active` - Conexões ativas do banco

### Health Checks

```bash
# Health check
curl http://localhost:3476/v1/health

# Readiness check
curl http://localhost:3476/v1/ready
```

### Logs

Configurar nível de log via variável de ambiente:

```bash
PERMIFY_LOG_LEVEL=debug
```

Níveis disponíveis: `debug`, `info`, `warn`, `error`.

## Backup e Restore

### Backup do PostgreSQL

```bash
# Backup
kubectl exec -n authz deployment/permify-db -- pg_dump -U permify permify > backup.sql

# Restore
kubectl exec -i -n authz deployment/permify-db -- psql -U permify permify < backup.sql
```

### Backup de Tuples

```bash
# Exportar todos os tuples
curl "http://localhost:3476/v1/tenants/production/tuples/read" | jq . > tuples-backup.json

# Importar tuples (precisa recriar)
jq -c '.tuples[]' tuples-backup.json | while read tuple; do
  curl -X POST "http://localhost:3476/v1/tenants/production/tuples/write" \
    -H "Content-Type: application/json" \
    -d "{\"tuple\": $tuple}"
done
```

## Troubleshooting

### Problemas Comuns

1. **Conexão recusada**
   ```
   Error: connect ECONNREFUSED 127.0.0.1:3476
   ```
   - Verificar se o Permify está rodando
   - Checar portas com `docker ps` ou `kubectl get svc`

2. **Timeout**
   ```
   Error: Request timeout after 1500ms
   ```
   - Aumentar `PERMIFY_TIMEOUT_MS`
   - Verificar latência da rede
   - Checar performance do banco

3. **Schema inválido**
   ```
   Error: invalid schema: syntax error
   ```
   - Validar schema no [Playground](https://play.permify.co)
   - Verificar indentação e sintaxe

4. **Tenant não encontrado**
   ```
   Error: tenant not found
   ```
   - Criar tenant com primeiro schema
   - Verificar `PERMIFY_TENANT_ID`

### Debug Commands

```bash
# Verificar schemas
curl http://localhost:3476/v1/tenants/dev/schemas/read

# Verificar tuples específicos
curl -X POST http://localhost:3476/v1/tenants/dev/tuples/read \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "entity": {"type": "organization", "id": "clickbus"}
    }
  }'

# Testar check manual
curl -X POST http://localhost:3476/v1/tenants/dev/permissions/check \
  -H "Content-Type: application/json" \
  -d '{
    "entity": {"type": "organization", "id": "clickbus"},
    "permission": "access",
    "subject": {"type": "user", "id": "carlos"}
  }'

# Verificar health
curl http://localhost:3476/v1/health
```

## Referências

- [Permify Documentation](https://permify.co/docs)
- [Schema Language Reference](https://permify.co/docs/schema-language)
- [Playground](https://play.permify.co)
- [GitHub Repository](https://github.com/Permify/permify)

---

**Platform Team · 2026**