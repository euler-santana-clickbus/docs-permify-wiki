# Arquitetura — Fluxo de AuthN e AuthZ

Este documento apresenta a arquitetura completa do fluxo de autenticação (Keycloak) e autorização (Permify).

## Visão Geral

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENTE (Browser)                            │
│                      maestro-frontend (Next.js)                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               │ 1. Login (PKCE)
                               ▼
                    ┌──────────────────────┐
                    │      Keycloak        │
                    │   (Authentication)   │
                    └──────────┬───────────┘
                               │
                               │ 2. Access Token (JWT)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENTE (Browser)                            │
│                      maestro-frontend (Next.js)                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               │ 3. API Request
                               │    Authorization: Bearer {token}
                               │    x-org-id: org-1
                               ▼
                    ┌──────────────────────┐
                    │         BFF          │
                    │  (clickbus-platform) │
                    └──────────┬───────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
                │ 4a. AuthN    │              │ 4b. AuthZ
                ▼              │              ▼
    ┌──────────────────┐      │      ┌──────────────────┐
    │    Keycloak      │      │      │     Permify      │
    │  (JWT via JWKS)  │      │      │  (Authorization) │
    └──────────┬───────┘      │      └────────┬─────────┘
               │              │               │
               │ 5a. Valid?   │               │ 5b. Allowed?
               │   ✅/❌      │               │   ✅/❌
               ▼              │               ▼
    ┌──────────────────┐      │      ┌──────────────────┐
    │  request.user    │      │      │  403 Forbidden   │
    │  { sub, ... }    │      │      │  or Continue     │
    └──────────────────┘      │      └──────────────────┘
                               │
                               │ 6. Response
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENTE (Browser)                            │
│                      maestro-frontend (Next.js)                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Fluxo Detalhado

### 1. Login (PKCE Flow)

**Cliente → Keycloak**

```
1. maestro-frontend gera code_verifier e code_challenge
2. Redireciona para Keycloak:
   GET /realms/demo/protocol/openid-connect/auth
     ?client_id=insights-frontend
     &redirect_uri=http://localhost:8088/callback
     &response_type=code
     &scope=openid profile email
     &code_challenge={hash}
     &code_challenge_method=S256
     &state={random}

3. Usuário faz login (alice/alice)

4. Keycloak redireciona de volta:
   GET /callback?code={auth_code}&state={state}

5. maestro-frontend troca code por tokens:
   POST /realms/demo/protocol/openid-connect/token
     code={auth_code}
     code_verifier={verifier}
     grant_type=authorization_code
     client_id=insights-frontend
     redirect_uri=http://localhost:8088/callback

6. Keycloak retorna:
   {
     "access_token": "eyJhbG...",
     "refresh_token": "eyJhbG...",
     "expires_in": 300
   }

7. maestro-frontend armazena em cookies:
   - accessToken (httpOnly)
   - refreshToken (httpOnly)
   - orgId (x-org-id header)
```

### 2. API Request com AuthN e AuthZ

**Cliente → BFF → Keycloak + Permify**

```
1. maestro-frontend faz request:
   GET /api/v4/maestro/trends/summary
   Headers:
     Authorization: Bearer eyJhbG...
     x-org-id: org-1

2. BFF recebe request

3. KeycloakJwtAuthGuard (AuthN):
   a. Extrai token do header Authorization
   b. Valida JWT via JWKS (chave pública do Keycloak)
   c. Verifica issuer, audience, expiration
   d. Popula request.user = { sub, email, ... }
   e. Se inválido → 401 Unauthorized

4. PermifyGuard (AuthZ):
   a. Lê metadados do decorator @UsePermifyCheck
   b. Resolve entityId:
      - De header: x-org-id → "org-1"
      - Ou de param: /:id → "123"
   c. Extrai subjectId de request.user.sub
   d. Chama Permify:
      POST /v1/tenants/t1/permissions/check
      {
        "entity": {"type": "organization", "id": "org-1"},
        "permission": "access",
        "subject": {"type": "user", "id": "{sub}"}
      }
   e. Permify verifica tuples e schema
   f. Se DENY → 403 Forbidden
   g. Se ALLOW → continua

5. Controller executa lógica de negócio

6. BFF retorna response (200 OK ou 403 Forbidden)
```

### 3. Permify Check Interno

**BFF → Permify → Postgres**

```
1. PermifyGuard chama PermifyService.check()

2. PermifyService verifica cache local (TTL 2s)
   - Key: t1|organization:org-1|access|user:{sub}
   - Se hit → retorna cached result

3. Se miss, faz HTTP request:
   POST http://localhost:3476/v1/tenants/t1/permissions/check
   {
     "metadata": {"depth": 20},
     "entity": {"type": "organization", "id": "org-1"},
     "permission": "access",
     "subject": {"type": "user", "id": "{sub}"}
   }

4. Permify processa:
   a. Lê schema do tenant t1
   b. Resolve permission "access":
      permission access = admin or member
   c. Busca tuples no Postgres:
      SELECT * FROM relation_tuples
      WHERE entity_type = 'organization'
        AND entity_id = 'org-1'
        AND relation IN ('admin', 'member')
        AND subject_type = 'user'
        AND subject_id = '{sub}'
   d. Se encontrou tuple → ALLOW
   e. Se não encontrou → DENY

5. Permify retorna:
   {"can": "CHECK_RESULT_ALLOWED"}
   ou
   {"can": "CHECK_RESULT_DENIED"}

6. PermifyService armazena no cache e retorna boolean
```

## Componentes

### Frontend (maestro-frontend)

**Responsabilidades:**
- Login via Keycloak PKCE
- Armazenar tokens em cookies (httpOnly)
- Adicionar headers em requests:
  - `Authorization: Bearer {token}`
  - `x-org-id: {orgId}`
- Refresh token automático

**Tecnologias:**
- Next.js 13+ (App Router)
- next-auth para PKCE
- Axios para API calls

### BFF (clickbus-platform)

**Responsabilidades:**
- Validar JWT (AuthN)
- Verificar permissões (AuthZ)
- Expor API REST para frontend
- Proxy para microserviços

**Guards:**
1. **KeycloakJwtAuthGuard** - Autenticação
   - Valida JWT via JWKS
   - Popula request.user
   - Retorna 401 se inválido

2. **PermifyGuard** - Autorização
   - Consulta Permify
   - Cache local (TTL 2s)
   - Retorna 403 se negado

**Tecnologias:**
- NestJS (Node.js/TypeScript)
- @permify/permify-node SDK
- Redis para cache (opcional)

### Keycloak

**Responsabilidades:**
- Autenticação de usuários
- Emissão de JWTs
- Gerenciamento de realms e clients

**Endpoints:**
- Auth: `/realms/{realm}/protocol/openid-connect/auth`
- Token: `/realms/{realm}/protocol/openid-connect/token`
- JWKS: `/realms/{realm}/protocol/openid-connect/certs`

### Permify

**Responsabilidades:**
- Autorização fine-grained
- Schema de permissões
- Armazenamento de tuples
- Engine de decisão

**Endpoints:**
- Check: `POST /v1/tenants/{tenant}/permissions/check`
- Lookup: `POST /v1/tenants/{tenant}/permissions/lookup-entity`
- Write: `POST /v1/tenants/{tenant}/tuples/write`
- Delete: `POST /v1/tenants/{tenant}/tuples/delete`

## Schema de Permissões

### Entidades Base

```permify
entity user {}

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

entity module {
  relation organization @organization
  relation company @company
  relation owner_user @user
  relation editor_user @user
  relation viewer_user @user
  relation guest_user @user

  permission owner = owner_user
  permission editor = editor_user
  permission viewer = viewer_user
  permission guest = guest_user

  permission view = guest or viewer or editor or owner or organization.admin or organization.manager or company.admin or company.manager or company.organization.admin or company.organization.manager
  permission edit = editor or owner or organization.admin or organization.manager or company.admin or company.manager or company.organization.admin or company.organization.manager
  permission manage = owner or organization.admin or company.admin or company.organization.admin
  permission delete = organization.admin or company.admin or company.organization.admin
}
```

### Exemplos de Tuples

```
# Usuários na organização
organization:clickbus#admin@user:carlos
organization:clickbus#manager@user:maria
organization:clickbus#member@user:alice

# Empresa dentro da organização
company:santa-cruz#organization@organization:clickbus
company:santa-cruz#admin@user:bob

# Módulo vinculado à organização
module:insights#organization@organization:clickbus
module:insights#owner_user@user:carlos
module:insights#editor_user@user:maria
module:insights#viewer_user@user:alice

# Módulo vinculado à empresa
module:b2b#company@company:santa-cruz
module:b2b#owner_user@user:bob
```

## Exemplos de Fluxo

### Exemplo 1: Acesso ao Dashboard

```
1. Alice acessa /dashboard
2. Frontend: GET /api/v4/dashboard
   Headers: Authorization: Bearer {token}, x-org-id: clickbus
3. BFF: KeycloakJwtAuthGuard valida JWT → request.user.sub = "alice"
4. BFF: PermifyGuard check
   Entity: module:insights-clickbus
   Permission: view
   Subject: user:alice
5. Permify: encontra tuple module:insights#viewer_user@user:alice → ALLOW
6. Controller: retorna dados do dashboard
7. Frontend: renderiza dashboard
```

### Exemplo 2: Edição Negada

```
1. Alice tenta editar relatório: POST /api/v4/reports/123
2. BFF: KeycloakJwtAuthGuard OK
3. BFF: PermifyGuard check
   Entity: module:insights-clickbus
   Permission: edit
   Subject: user:alice
4. Permify: NÃO encontra tuple de editor → DENY
5. BFF: retorna 403 Forbidden
6. Frontend: mostra "Acesso negado"
```

### Exemplo 3: Admin Global

```
1. Carlos (organization.admin) acessa qualquer módulo
2. Permify: resolve permission "view"
   view = guest or viewer or editor or owner or organization.admin or ...
3. Carlos é organization.admin → ALLOW por herança
4. Acesso concedido sem tuple direto no módulo
```

## Performance e Cache

### Estratégia de Cache

1. **BFF Cache Local** (TTL 2s)
   - Redis ou memória
   - Chave: `{tenant}|{entity}:{id}|{permission}|{subject}`
   - Reduz 90% das chamadas ao Permify

2. **Permify Cache Interno**
   - Cache de schema e tuples
   - Invalidação automática em escritas

3. **API Gateway Cache** (TTL 300s)
   - Cache por token + resource
   - Reduz invocações da Lambda

### Métricas Alvo

| Métrica | Target |
|:---|:---|
| p50 latência check | < 5ms |
| p95 latência check | < 15ms |
| p99 latência check | < 50ms |
| Cache hit rate | > 80% |

## Segurança

### Defesa em Profundidade

1. **Frontend** - Esconde elementos (UX apenas)
2. **API Gateway** - Lambda Authorizer (coarse)
3. **BFF** - Guards (medium)
4. **Microserviços** - Checks fine-grained

### Headers Obrigatórios

| Header | Valor | Exemplo |
|:---|:---|:---|
| `Authorization` | Bearer JWT do Keycloak | `Bearer eyJhbG...` |
| `x-org-id` | Contexto da organização | `clickbus` |
| `x-company-id` | Contexto da empresa (opcional) | `santa-cruz` |

### Fail-Closed

- Se Permify indisponível → DENY
- Se JWT inválido → 401
- Se sem permissão → 403

## Monitoramento

### Logs Obrigatórios

```json
{
  "timestamp": "2024-02-20T10:15:30Z",
  "event": "authz_check",
  "userId": "alice",
  "orgId": "clickbus",
  "resourceType": "module",
  "resourceId": "insights-clickbus",
  "action": "view",
  "decision": "ALLOW",
  "latencyMs": 12,
  "source": "bff_guard",
  "traceId": "abc-123"
}
```

### Métricas

- Taxa de ALLOW/DENY por recurso
- Latência dos checks
- Cache hit rate
- Erros de comunicação

## Deploy

### Ambientes

| Ambiente | Tenant ID | Keycloak Realm | Permify URL |
|:---|:---|:---|:---|
| Dev | `dev` | `demo` | `http://permify-dev:3476` |
| Staging | `staging` | `demo-stg` | `http://permify-stg:3476` |
| Production | `production` | `clickbus` | `http://permify-prod:3476` |

### Configuração

```yaml
# docker-compose.yml
services:
  permify:
    image: ghcr.io/permify/permify:latest
    environment:
      - PERMIFY_DATABASE_URI=postgres://user:pass@postgres:5432/permify
      - PERMIFY_SERVE_GRPC_PORT=3478
      - PERMIFY_SERVE_HTTP_PORT=3476
    ports:
      - "3476:3476"
      - "3478:3478"

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=permify
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

## Referências

- [PRD Completo](PRD---Permify-AuthZ)
- [Integração NestJS + Spring Boot](Integração-NestJS-+-Spring-Boot)
- [SDK Node.js](https://github.com/Permify/permify-node)
- [SDK Java](https://github.com/Permify/permify-java)
- [Permify Documentation](https://permify.co/docs)

---

**Platform Team · 2026**