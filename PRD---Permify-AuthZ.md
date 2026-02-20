# PRD — Permify: Serviço de Autorização Fine-Grained

**Versão**: 1.0
**Data**: Fevereiro 2026
**Owner**: Platform Team
**Status**: Draft

---

## 1. Resumo Executivo

Implantar o **Permify** como serviço centralizado de autorização (AuthZ) para a plataforma multi-tenant e multi-produto, utilizando o modelo relacional inspirado no **Google Zanzibar**. O Permify será a **fonte de verdade** de permissões, consultado pelo BFF/Gateway para decisões de allow/deny e query filtering, enquanto o **Keycloak** permanece responsável exclusivamente pela autenticação (AuthN).

### Por que Permify?

- **Open-source** (Apache 2.0) — sem custo de licença, sem vendor lock-in
- **Modelo Zanzibar** — comprovado em escala pelo Google (YouTube, Drive, Maps)
- **Fine-grained** — suporta RBAC, ABAC e ReBAC em um único modelo
- **Multi-tenancy nativo** — isolamento por tenant sem complexidade adicional
- **Performance** — checks em < 10ms com cache, lookup-resources para listas
- **CNCF Silver Member** — projeto maduro e com comunidade ativa

---

## 2. Problema / Motivação

### Situação Atual

- Autorização **duplicada e inconsistente** entre múltiplos serviços
- **Vazamento de dados** em endpoints de lista por ausência de query filtering
- Permissões **hard-coded** em controllers, difíceis de auditar e evoluir
- Sem suporte a **multi-empresa** (uma organização com N sub-empresas)
- **Impossível responder** "o que o usuário X pode fazer?" sem varrer todo o código

### Situação Desejada

- **Fonte de verdade única** para todas as permissões (Permify)
- **Enforcement centralizado** no BFF/Gateway (PEP)
- **Query filtering** via lookup-resources para endpoints de lista
- **Hierarquia flexível**: Organization → [Company] → Module → Resource (company opcional)
- **Auditabilidade** completa: quem pode o quê, quando mudou, por quê
- **Self-service** para times de produto definirem seus schemas

---

## 3. Objetivos

### Objetivos Primários

- **OBJ-1**: Centralizar 100% das decisões de autorização no Permify
- **OBJ-2**: Suportar hierarquia flexível Organization → [Company] → Module com herança de permissões (company opcional)
- **OBJ-3**: Implementar query filtering (lookup-resources) em endpoints de lista
- **OBJ-4**: Garantir fail-closed (deny por padrão se Permify indisponível)
- **OBJ-5**: Permitir que times de produto definam schemas de forma autônoma

### Não-Objetivos (fora do escopo)

- Migrar todos os produtos legados de uma vez (será por fases)
- Colocar permissões dentro do JWT (explicitamente evitado)
- Substituir o Keycloak (ele continua como IdP)
- Implementar UI de gestão de permissões (fase futura)

---

## 4. Stakeholders e Personas

| Persona | Responsabilidade | Interação com Permify |
|:---|:---|:---|
| **Platform Team** | Owner do schema base, tooling, CI/CD | Define `organization`, `company`, `module`, `group`, `team` |
| **Product Teams** | Owners dos schemas de módulo | Definem resources e permissions dos seus domínios |
| **DevOps/SRE** | Operação, HA, monitoramento | Deploy, backup, alertas, runbooks |
| **Segurança/Compliance** | Auditoria, políticas | Revisam schemas, auditam tuples |
| **Usuário final (externo)** | Acessa produtos dentro do tenant | Sujeito das permissões |
| **Admin (interno)** | Gerencia orgs, companies, usuários | Escreve tuples via Admin Plane |

---

## 5. Modelo de Autorização

### 5.1 Hierarquia de Entidades

A **Company é opcional**. O usuário pode existir apenas na Organization e ter acesso direto a modules.

**Cenário A — Sem Company (org simples):**

```
Organization (ex: ClickBus)
├── Module (ex: insights)
│   └── Resources (ex: insights_dashboard, insights_reports)
├── Module (ex: b2b)
│   └── Resources (ex: b2b_partner)
├── Team (ex: Time Comercial)
└── Group (ex: Auditores)
```

**Cenário B — Com Company (multi-empresa):**

```
Organization (ex: Santa Cruz)
│
├── Company (ex: Santa Cruz 1)
│   ├── Team (ex: Time Comercial)
│   ├── Module (ex: insights)
│   │   └── Resources (ex: insights_dashboard, insights_reports)
│   └── Module (ex: b2b)
│       └── Resources (ex: b2b_partner, b2b_contract)
│
├── Company (ex: Santa Cruz 2)
│   ├── Team (ex: Time Operações)
│   └── Module (ex: insights)
│
└── Group (ex: Auditores — cross-company)
```

### 5.2 Tipos de Usuário (Roles)

#### Organization / Company

| Role | Conceito | O que pode fazer | O que NÃO pode fazer |
|:---|:---|:---|:---|
| **admin** | Controle total. Dono do nível. | Gerencia tudo: usuários, configs, modules, teams. Herda acesso a tudo abaixo. | — |
| **manager** | Gestor operacional. | Cria/edita modules e teams, atribui roles, gerencia operações do dia-a-dia. | Não gerencia admins. Não exclui org/company/modules. |
| **member** | Participante padrão. | Acessa a org/company, aparece como membro. | Precisa de grants explícitos para acessar modules. |

#### Module

| Role | Conceito | O que pode fazer | O que NÃO pode fazer |
|:---|:---|:---|:---|
| **owner** | Dono do módulo. | Controle total: configs, permissões, exclusão de recursos. | — |
| **editor** | Criador/editor de conteúdo. | Cria e edita dados, relatórios, registros. | Não altera configs ou permissões. |
| **viewer** | Somente leitura. | Visualiza dashboards, relatórios, dados. | Não cria, edita ou exclui nada. |
| **guest** | Convidado externo/temporário. | Acesso mínimo de leitura, escopo limitado. | Não edita, não exporta, acesso restrito. |

#### Group e Team

| Entidade | Role | Conceito |
|:---|:---|:---|
| **Group** | `admin` / `member` | Admin gerencia o grupo; member herda permissões do grupo |
| **Team** | `lead` / `member` | Lead gerencia o time; member herda permissões do time |

### 5.3 Herança de Permissões (Top-Down)

| Nível | Escopo | Herda de | Obrigatório? |
|:---|:---|:---|:---:|
| `organization.admin` | Acesso total a TUDO (manage + delete) | — | Sim |
| `organization.manager` | Edita e visualiza tudo, gerencia modules e teams | `organization.admin` | Sim |
| `organization.member` | Acesso básico à org (sem herança para modules) | — | Sim |
| `company.admin` | Acesso total à SUA company | `organization.admin` | Não |
| `company.manager` | Edita e visualiza na SUA company | `organization.admin`, `organization.manager` | Não |
| `company.member` | Acesso básico à company | — | Não |
| `module.owner` | Controle total do módulo (configs, permissões) | `organization.admin`, `company.admin` | Sim |
| `module.editor` | Cria e edita conteúdo | `organization.manager`, `company.manager` | Sim |
| `module.viewer` | Somente leitura | `module.editor` | Sim |
| `module.guest` | Acesso mínimo, convidado externo | — | Sim |

---

## 6. Requisitos Funcionais

### RF-1 — Enforcement multi-camada (defesa em profundidade)

A autorização deve ser aplicada em **múltiplas camadas**, criando uma defesa em profundidade:

| Camada | Tipo de Check | Granularidade | Responsabilidade |
|:---|:---|:---|:---|
| **API Gateway** | Coarse-grained | Rota / módulo | Bloqueia acessos não autorizados antes de chegar ao BFF |
| **BFF** | Medium-grained | Módulo / ação | Valida permissões de ação (view, edit, manage, delete) |
| **Microserviço** | Fine-grained | Recurso específico | Valida permissões no nível do recurso (ex: "pode aprovar o contrato #123?") |
| **Frontend** | UI only | Visibilidade | Esconde elementos que o usuário não pode acessar (não substitui backend) |

### RF-2 — Query filtering (lookup-resources)

Para endpoints que retornam listas:
1. `check` coarse-grained ("pode acessar o módulo?")
2. `lookup-resources` para obter IDs permitidos
3. Filtro na query: `WHERE id IN (:allowedIds)`

### RF-3 — Hierarquia flexível Organization → [Company] → Module

- **Company é opcional** — o usuário pode existir apenas na Organization
- **Cenário A (sem company)**: modules e teams vinculados direto à organization
- **Cenário B (com company)**: uma organization pode ter N companies com modules isolados
- `organization.admin` herda acesso a tudo (modules e companies)
- `company.admin` herda acesso apenas à sua company (cenário B)
- Isolamento total entre companies (exceto via groups cross-company)

### RF-4 — Estratégia de Multi-Tenancy

**Tenant por Ambiente** como base, com **isolamento lógico por Organization** via hierarquia do schema.

```
Permify Tenants:
├── dev        ← ambiente de desenvolvimento
├── staging    ← ambiente de homologação
└── production ← ambiente de produção
```

Dentro de cada tenant, o isolamento entre organizações é garantido pela **hierarquia do schema** (Organization → Company → Module), não por tenants separados.

### RF-5 — Admin Plane (escrita de tuples)

Eventos que geram escrita/atualização de tuples:
- Criação de organization/company
- Adição/remoção de usuário em company
- Atribuição de role em module
- Criação/alteração de team/group
- Habilitação de módulo para uma company

### RF-6 — Política de falha (fail-closed)

- Se Permify indisponível → **deny** para endpoints protegidos
- Exceções devem ser explicitamente classificadas e monitoradas
- Circuit breaker com fallback para deny

### RF-7 — Endpoint de permissões do usuário

- `GET /api/v4/permissions/me` retorna todas as permissões do usuário autenticado
- Usado pelo frontend para renderização condicional de UI

---

## 7. Requisitos Não Funcionais

### NFR-1 — Performance

| Métrica | Target |
|:---|:---|
| Latência p50 (check) | < 5ms |
| Latência p95 (check) | < 15ms |
| Latência p99 (check) | < 50ms |
| Latência p95 (lookup-resources) | < 50ms |
| Cache hit rate | > 80% |
| Cache TTL | 30–120s (configurável) |

### NFR-2 — Disponibilidade

| Métrica | Target |
|:---|:---|
| Uptime (self-hosted) | 99.9% (HA com 2+ instâncias) |
| RTO (Recovery Time Objective) | < 5 min |
| RPO (Recovery Point Objective) | < 1 min (Postgres replication) |

### NFR-3 — Segurança

- Não retornar dados sem autorização (obrigatório)
- Defesa em profundidade: API Gateway + BFF + Microserviço (múltiplas camadas de check)
- Tenant sempre explícito (path/header)
- Token JWT não carrega autorização fina
- Comunicação via rede interna (VPC)
- Tuples sensíveis auditadas

---

## 8. Arquitetura

### 8.1 Componentes

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
│   Frontend   │────>│ API Gateway  │────>│   Lambda Authorizer  │
│  (Next.js)   │     │    (AWS)     │     │  (PEP coarse-grained)│
└──────┬───────┘     └──────┬───────┘     │  1. Valida JWT       │
       │                    │             │  2. Mapeia rota       │
       │ lookup-entity      │ ✅ Allow    │  3. Check Permify    │
       │ (UI only)          │             └──────────┬───────────┘
       │                    ▼                        │
       │             ┌──────────────┐                │
       │             │     BFF      │     ┌──────────▼───────────┐
       │             │   (NestJS)   │     │      Permify         │
       │             │              │────>│      (AuthZ)         │
       │             │  ┌────────┐  │     │                      │
       │             │  │ Guards │  │     │  ┌────────────────┐  │
       │             │  │ AuthZ  │  │     │  │    Postgres    │  │
       │             │  └────────┘  │     │  └────────────────┘  │
       │             └──────┬───────┘     └──────────▲───────────┘
       │                    │                        │
       │                    ▼                        │
       │             ┌──────────────┐                │
       │             │Microserviços │────────────────┘
       │             │ (ECS/Lambda) │  (PEP fine-grained)
       │             └──────────────┘
       │
       │             ┌──────────────┐     ┌──────────────┐
       └────────────>│   Keycloak   │     │ Admin Plane  │
                     │    (AuthN)   │     │ (Backoffice) │
                     └──────────────┘     │ write tuples │
                                          └──────────────┘
```

### 8.2 Camadas de Enforcement (PEP)

```
Request → [1] API Gateway → [2] BFF → [3] Microserviço
              (coarse)       (medium)    (fine-grained)

[1] API Gateway (Lambda Authorizer):
    - Valida JWT (Keycloak JWKS)
    - Check rota-level: "pode acessar /api/insights?"
    - Cache TTL 300s por token+resource
    - Deny → 403 (request nunca chega ao BFF)

[2] BFF (Guards):
    - Check módulo/ação: "pode editar no módulo insights?"
    - lookup-resources para listas
    - Deny → 403

[3] Microserviço:
    - Check recurso específico: "pode aprovar o contrato #123?"
    - Recebe userId/orgId do context do Gateway
    - Workers/jobs assíncronos também checam
```

---

## 9. SDKs Oficiais do Permify

O Permify oferece SDKs oficiais para as principais linguagens. Todos os SDKs suportam:

- **Protocolos**: gRPC (porta 3478) e REST (porta 3476)
- **Operações**: `check`, `lookup-entity`, `lookup-resources`, `write-tuples`, `delete-tuples`
- **Cache**: Redis ou memória local (configurável)
- **Multi-tenancy**: tenant ID explícito em todas as chamadas

| Linguagem | SDK oficial | Instalação | Exemplo de uso |
|:---|:---|:---|:---|
| **Node.js / TypeScript** | [permify-node](https://github.com/Permify/permify-node) | `npm install @permify/permify-node` | `import * as permify from '@permify/permify-node'` |
| **Java** | [permify-java](https://github.com/Permify/permify-java) | Maven/Gradle | `import is.permify.PermifyClient;` |
| **Go** | [permify-go](https://github.com/Permify/permify-go) | `go get github.com/Permify/permify-go` | `import "github.com/Permify/permify-go"` |
| **Python** | [permify-python](https://github.com/Permify/permify-python) | `pip install permify` | `from permify import Permify` |
| **Ruby** | [permify-ruby](https://github.com/Permify/permify-ruby) | `gem install permify` | `require 'permify'` |
| **PHP** | [permify-php](https://github.com/Permify/permify-php) | Composer | `use Permify\Permify;` |
| **C#** | [permify-csharp](https://github.com/Permify/permify-csharp) | NuGet | `using Permify;` |
| **TypeScript (frontend)** | [permify-typescript](https://github.com/Permify/permify-typescript) | `npm install @permify/permify-typescript` | `import { PermifyClient } from '@permify/permify-typescript'` |

### Exemplo rápido (Node.js)

```typescript
import * as permify from '@permify/permify-node';

const client = permify.grpc.newClient({
  endpoint: 'http://localhost:3476',
  tenantId: 'production'
});

// Check de permissão
const result = await client.permissions.check({
  entity: { type: 'module', id: 'insights' },
  permission: 'view',
  subject: { type: 'user', id: 'user-123' }
});

console.log(result.can); // true ou false
```

### Exemplo rápido (Java)

```java
import is.permify.PermifyClient;
import is.permify.grpc.v1.*;

PermifyClient client = new PermifyClient("http://localhost:3476", "production");

CheckResponse resp = client.check(
  Entity.newBuilder().setType("module").setId("insights").build(),
  "view",
  Subject.newBuilder().setType("user").setId("user-123").build()
);

System.out.println(resp.getCan()); // true ou false
```

---

## 10. Análise de Custo

### Recomendação: Community Edition (Self-Hosted)

| Ambiente | Custo estimado/mês |
|:---|---:|
| Dev/Staging (Docker Compose) | ~US$ 50 |
| Produção HA (EKS + RDS Multi-AZ) | ~US$ 300 |
| Produção Alta Escala (EKS + RDS + Redis) | ~US$ 885 |

**Justificativa**:
- Custo zero de licença (Apache 2.0)
- Controle total sobre dados (compliance LGPD)
- Sem limite de MAUs
- Time DevOps já disponível
- Kubernetes já em uso

---

## 11. Métricas de Sucesso

### Cobertura

| Métrica | Target MVP | Target Final |
|:---|:---:|:---:|
| % endpoints usando Permify check | 100% (módulo piloto) | 100% (todos módulos) |
| % endpoints de lista com lookup-resources | 80% (módulo piloto) | 100% |
| % schemas versionados no Git | 100% | 100% |

### Qualidade

| Métrica | Target |
|:---|:---|
| Incidentes de vazamento por ausência de filtro | Zero |
| Divergências de autorização entre serviços | Zero |
| Tempo médio para onboarding de novo módulo | < 1 dia |

### Performance

| Métrica | Target |
|:---|:---|
| p95 latência de check | < 15ms |
| p95 latência de lookup | < 50ms |
| Cache hit rate | > 80% |

---

**Platform Team · 2026**