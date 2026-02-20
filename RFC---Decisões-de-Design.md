# 0021 — Arquitetura de Autorização Granular (Permify) e Padrão de Governança

**Status:** Padrão submetido  
**Data:** 04 de Novembro de 2025  
**Confidencial** — não pode ser compartilhado para fora da comunidade ClickBus

| # | Colaborador | Papel |
|:---:|:---|:---|
| 1 | @Euler Santana | Autor Principal |
| 2 | @Gabriel Gomes | Revisor Principal |
| 3 | @Rodrigo Matos | Revisor Principal |
| 4 | @Fabio Wakim Trentini (trentas) | Revisor Principal |
| 5 | @Paula Antunes Ferreira | Revisor Principal |

> Para saber se todo o processo está "ok", favor entrar em contato com o responsável: **@Rodrigo Matos**.

---

## Resumo

Esta RFC define a adoção do **Permify** como solução centralizada de Autorização (AuthZ), complementando o **Keycloak** (AuthN). O Permify implementa o modelo **ReBAC** (Relationship-Based Access Control), inspirado no **Google Zanzibar**, onde permissões são derivadas de grafos de relacionamento em tempo real.

**Objetivos:**

- Eliminar lógica de permissão hardcoded nos microsserviços
- Viabilizar segregação multi-tenant (B2B) com hierarquia flexível: **Organization → [Company] → Module → Resource**
- Padronizar onboarding de novos módulos via fluxo auditável
- Enforcement multi-camada: API Gateway + BFF + Microserviço

---

## 1. Motivação

| Problema | Impacto |
|:---|:---|
| **Explosão de Roles** | RBAC tradicional exige uma role por variação de acesso (`admin_empresa_A`, `operador_financeiro_B`), tornando o JWT pesado |
| **Performance em Listas (N+1)** | Filtrar recursos visíveis exige lógica complexa ou dados excessivos do banco |
| **Segurança Fragmentada** | Sem ponto central auditável para decisões de acesso |
| **Autorização duplicada** | Permissões hard-coded em controllers de múltiplos serviços |
| **Sem multi-empresa** | Impossível isolar permissões por company dentro de uma organização |

**Solução:** Modelo ReBAC com o Permify — open-source (Apache 2.0), CNCF Silver Member, suporta RBAC, ABAC e ReBAC em um único modelo.

---

## 2. Decisões de Arquitetura

### 2.1 Separação AuthN × AuthZ

O JWT **não carrega permissões** — carrega apenas identidade (`sub`, `email`). As permissões são resolvidas em tempo real pelo Permify.

| Componente | Pergunta | Responsabilidade |
|:---|:---|:---|
| **Keycloak** | "Quem é você?" | Autenticação, JWT, refresh tokens, JWKS |
| **Permify** | "O que você pode fazer?" | Permissões, resolução de relações, lookup de recursos |

> Detalhes do fluxo completo em [Arquitetura AuthN + AuthZ](Arquitetura-AuthN-+-AuthZ).

### 2.2 Enforcement Multi-Camada

| Camada | Granularidade | Exemplo |
|:---|:---|:---|
| **API Gateway** (Lambda Authorizer) | Rota / módulo | "Pode acessar `/finance/*`?" |
| **BFF** (NestJS Guards) | Módulo / ação | "Pode `edit` no módulo `finance`?" |
| **Microserviço** | Recurso específico | "Pode aprovar a invoice `inv-123`?" |
| **Frontend** | UI only (não substitui backend) | Esconde botões/menus sem permissão |

### 2.3 Fluxo Resumido

1. Usuário autentica via **Keycloak** (PKCE) → recebe JWT com `sub` e `email`
2. **API Gateway** valida JWT (JWKS) e faz check coarse-grained no Permify → DENY = 403
3. **BFF** valida token (`KeycloakJwtAuthGuard`) e consulta Permify via `PermifyGuard` → DENY = 403
4. **Microserviço** faz check fine-grained quando necessário (recurso específico)
5. Permify percorre o grafo de relações → `ALLOW` ou `DENY`

### 2.4 Política de Falha

**Fail-Closed:** se o Permify estiver indisponível, o acesso é **NEGADO** por padrão.

---

## 3. Modelo de Autorização

### 3.1 Hierarquia de Entidades

A hierarquia é flexível — **Company é opcional**:

- **Sem Company:** `Organization → Module → Resource`
- **Com Company (B2B):** `Organization → Company → Module → Resource`

Entidades auxiliares: **Team** (time dentro de org/company) e **Group** (cross-company).

> Detalhes e exemplos em [Hierarquia Org → Company → Module](Hierarquia-Org-→-Company-→-Module).

### 3.2 Roles

| Nível | Roles | Herança |
|:---|:---|:---|
| **Organization / Company** | `admin`, `manager`, `member` | admin herda tudo; member precisa de grant explícito para modules |
| **Module** | `owner`, `editor`, `viewer`, `guest` | owner herda tudo; guest = leitura mínima |
| **Team** | `lead`, `member` | members herdam permissões atribuídas ao team |
| **Group** | `admin`, `member` | members herdam permissões atribuídas ao group |

**Regra fundamental:** `organization.admin` **sempre** herda acesso a tudo abaixo. Isso evita lock-out de administradores.

### 3.3 Mecanismos de Grant

| Mecanismo | Exemplo de Tuple |
|:---|:---|
| **Direto** | `module:insights#editor_user@user:alice` |
| **Via Group** | `module:insights#viewer_group@group:auditores` |
| **Via Team** | `module:insights#editor_team@team:comercial` |
| **Via Hierarquia** | `company:sc1#admin@user:bob` → acesso a todos modules da SC1 |

### 3.4 Convenções Obrigatórias

- **Namespaces:** `{module}_{resource}` (ex: `finance_invoice`, não `invoice`)
- **Herança de Admin:** todo recurso DEVE herdar de `organization.admin`
- **Vinculação:** module/team vinculado a `organization` **OU** `company`, nunca ambos

> Schema base completo e política de criação em [Política de Permissões](Política-de-Permissões).

---

## 4. Multi-Tenancy

**Decisão:** Tenant por Ambiente (3 tenants fixos: `dev`, `staging`, `production`).

O isolamento entre organizações é garantido pela hierarquia do schema, não por tenants separados. Isso simplifica governança (1 PR atualiza todos os clientes) e onboarding (criar org = criar tuples).

O `PERMIFY_TENANT_ID` é configurado no deploy (env var), **nunca** resolvido dinamicamente.

> Detalhes de configuração em [Configuração do Permify](Configuração-do-Permify).

---

## 5. Workflow de Implementação (Golden Path)

Cada novo módulo segue 8 passos:

| Passo | O que | Output |
|:---|:---|:---|
| **1. Definição** | Definir módulo, recursos, papéis e permissões | RFC do módulo aprovada |
| **2. Schema** | Criar `.permify` a partir do template, validar localmente e no Playground | Arquivo de schema |
| **3. Seed** | Criar script de seed com tuples do módulo | Script `seed-{modulo}.sh` |
| **4. Guards** | Aplicar `@UsePermifyCheck` nos controllers do BFF | Controllers protegidos |
| **5. Config** | Configurar env vars (`PERMIFY_ENABLED`, `PERMIFY_BASE_URL`, `PERMIFY_TENANT_ID`) | BFF configurado |
| **6. Validação** | Testar happy path (200) e unhappy path (403) | Testes passando |
| **7. Docs** | Criar `docs/{modulo}-permissions.md` | Documentação |
| **8. Deploy** | PR → code review → CI/CD (dev → staging → production) | Em produção |

### Template de Schema

```permify
// modules/{module}/{module}.permify
entity {module}_{resource} {
  relation organization @organization
  relation company @company
  relation module @module
  
  // Roles específicos do módulo
  relation {role}_user @user
  relation {role}_group @group
  relation {role}_team @team
  
  permission {action} = {role}_user or {role}_group.member or {role}_team.member or module.{level} or organization.admin or company.admin or company.organization.admin
}
```

---

## 6. Padrões e Anti-Padrões

### ✅ Padrões Obrigatórios

1. **Fail-Closed**
   ```typescript
   try {
     const result = await permify.check(params);
     return result.can;
   } catch (error) {
     return false; // Sempre deny em caso de erro
   }
   ```

2. **Lookup para Listas**
   ```typescript
   // ✅ Correto - lookup + filtro
   const allowedIds = await permify.lookupEntity('report', 'view', userId);
   const reports = await db.query('SELECT * FROM reports WHERE id = ANY($1)', [allowedIds]);
   ```

3. **Logs Estruturados**
   ```json
   {
     "event": "authz_check",
     "userId": "alice",
     "resource": "module:insights",
     "action": "view",
     "decision": "ALLOW",
     "traceId": "abc-123"
   }
   ```

### ❌ Anti-Padrões

1. **N+1 Checks**
   ```typescript
   // ❌ NÃO FAZER
   const reports = await db.getAllReports();
   const filtered = [];
   for (const report of reports) {
     if (await check('report', report.id, 'view', userId)) {
       filtered.push(report);
     }
   }
   ```

2. **Permissões no JWT**
   ```json
   // ❌ NÃO FAZER
   {
     "sub": "alice",
     "permissions": ["admin:finance", "viewer:insights"] // Pesado, obsoleto
   }
   ```

3. **Cache Longo Demais**
   ```typescript
   // ❌ NÃO FAZER
   cache.set(key, result, { ttl: 3600000 }); // 1 hora = muito
   ```

---

## 7. Métricas e SLAs

### Performance

| Métrica | Target | Alerta |
|:---|:---|:---|
| p50 latência check | < 5ms | > 10ms |
| p95 latência check | < 15ms | > 30ms |
| p99 latência check | < 50ms | > 100ms |
| Cache hit rate | > 80% | < 70% |

### Disponibilidade

| Métrica | Target | Alerta |
|:---|:---|:---|
| Uptime | 99.9% | < 99.5% |
| Error rate | < 0.1% | > 1% |

### Segurança

| Evento | Ação |
|:---|:---|
| DENY inesperado | Investigar (possível bug) |
| Multiplas falhas | Escalar (possível ataque) |
| Schema sem PR | Bloquear deploy |

---

## 8. Governança

### 8.1 Processo de Mudança

1. **Proposta** - Abrir issue com `RFC: {título}`
2. **Discussão** - Revisão por arquitetos e security
3. **Aprovação** - Merge se não houver objeções
4. **Implementação** - Seguir golden path
5. **Documentação** - Atualizar docs e exemplos

### 8.2 Responsabilidades

| Papel | Responsabilidade |
|:---|:---|
| **Platform Team** | Manter schema base, tooling, CI/CD |
| **Product Teams** | Definir schemas de módulo, implementar guards |
| **DevOps/SRE** | Operação, monitoramento, backup |
| **Security** | Revisar schemas, auditorias, compliance |

### 8.3 Auditoria

- **Mensal**: Revisão de tuples sensíveis
- **Trimestral**: Teste de penetração dos controles
- **Anual**: Revisão completa da arquitetura

---

## 9. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|:---|:---:|:---:|:---|
| **Vazamento em listas** | Médio | Alto | Padrão lookup + filtro obrigatório |
| **Performance** | Baixo | Médio | Cache adequado, monitoramento |
| **Drift de schema** | Médio | Alto | GitOps, CI/CD automatizado |
| **Indisponibilidade** | Baixo | Alto | HA, fail-closed, runbooks |
| **Complexidade** | Médio | Médio | Documentação, templates, treinamento |

---

## 10. Implementação Status

### ✅ Completo

- [x] Schema base definido
- [x] BFF com guards implementado
- [x] CI/CD para schemas
- [x] Documentação inicial
- [x] Módulo piloto (insights)

### 🚧 Em Progresso

- [ ] Lambda Authorizer no API Gateway
- [ ] PEP nos microserviços
- [ ] Dashboard de auditoria
- [ ] Automação de onboarding

### 📋 Planejado

- [ ] UI de gestão (Admin Plane)
- [ ] Integração com SIEM
- [ ] Testes automatizados de segurança
- [ ] Métricas avançadas

---

## 11. Referências

- [PRD Completo](PRD---Permify-AuthZ) - Requisitos detalhados
- [Arquitetura AuthN + AuthZ](Arquitetura-AuthN-+-AuthZ) - Fluxo completo
- [Integração NestJS + Spring Boot](Integração-NestJS-+-Spring-Boot) - Exemplos práticos
- [Política de Permissões](Política-de-Permissões) - Regras e padrões
- [Configuração do Permify](Configuração-do-Permify) - Setup e operação

---

## 12. Apêndice

### A. Exemplo de Schema Completo

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

entity insights_dashboard {
  relation organization @organization
  relation company @company
  relation module @module
  relation owner_user @user
  relation viewer_user @user

  permission owner = owner_user
  permission viewer = viewer_user

  permission view = viewer or owner or module.viewer or module.editor or module.owner or organization.admin or organization.manager or company.admin or company.manager or company.organization.admin or company.organization.manager
  permission edit = owner or module.editor or module.owner or organization.admin or organization.manager or company.admin or company.manager or company.organization.admin or company.organization.manager
  permission delete = owner or module.owner or organization.admin or company.admin or company.organization.admin
}
```

### B. Exemplo de Seed Script

```bash
#!/bin/bash
# seed-insights.sh

TENANT=${PERMIFY_TENANT_ID:-dev}
PERMIFY_URL=${PERMIFY_URL:-http://localhost:3476}

# Organização
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "organization", "id": "clickbus"},
      "relation": "admin",
      "subject": {"type": "user", "id": "carlos"}
    }
  }'

# Módulo
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "module", "id": "insights"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "clickbus"}
    }
  }'

# Permissões no módulo
curl -X POST "$PERMIFY_URL/v1/tenants/$TENANT/tuples/write" \
  -H "Content-Type: application/json" \
  -d '{
    "tuple": {
      "entity": {"type": "module", "id": "insights"},
      "relation": "owner_user",
      "subject": {"type": "user", "id": "carlos"}
    }
  }'

echo "Seed completed for insights module"
```

---

**Platform Team · 2026**