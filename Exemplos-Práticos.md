# Exemplos de Uso — Permify

Este documento apresenta exemplos práticos de schemas, tuples e guards para diferentes módulos.

## Módulo Insights (Maestro)

### Schema

`permify/schemas/modules/insights.permify`

```permify
entity insights_dashboard {
  relation organization @organization
  relation module @module
  relation analyst @user
  relation data_viewer @user
  
  permission view_trends = data_viewer or analyst or module.editor or organization.admin
  permission export_data = analyst or module.owner or organization.admin
  permission configure_filters = module.editor or organization.admin
}

entity insights_report {
  relation organization @organization
  relation module @module
  relation report_manager @user
  relation viewer @user
  
  permission view_reports = viewer or report_manager or module.viewer or organization.admin
  permission create_report = report_manager or module.editor or organization.admin
  permission edit_report = report_manager or module.editor or organization.admin
  permission delete_report = module.owner or organization.admin
}
```

### Tuples de Exemplo

```json
{
  "tuples": [
    {
      "entity": {"type": "module", "id": "insights"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "clickbus"}
    },
    {
      "entity": {"type": "module", "id": "insights"},
      "relation": "owner_user",
      "subject": {"type": "user", "id": "carlos"}
    },
    {
      "entity": {"type": "insights_dashboard", "id": "trends-main"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "clickbus"}
    },
    {
      "entity": {"type": "insights_dashboard", "id": "trends-main"},
      "relation": "analyst",
      "subject": {"type": "user", "id": "alice"}
    },
    {
      "entity": {"type": "insights_dashboard", "id": "trends-main"},
      "relation": "data_viewer",
      "subject": {"type": "user", "id": "bob"}
    }
  ]
}
```

### Guards (NestJS)

```typescript
@Controller({ path: 'api/insights', version: '4' })
@UseGuards(KeycloakJwtAuthGuard, PermifyGuard)
export class InsightsController {
  
  @Get(':moduleId/dashboards')
  @RequirePermission('view')
  async getDashboards(@Param('moduleId') moduleId: string) {
    // Verifica module:insights-clickbus#view
    return this.insightsService.getDashboards(moduleId);
  }

  @Get(':moduleId/dashboards/:dashboardId')
  @RequirePermission('view')
  async getDashboard(
    @Param('moduleId') moduleId: string,
    @Param('dashboardId') dashboardId: string
  ) {
    // Verifica insights_dashboard:dashboardId#view_trends
    return this.insightsService.getDashboard(dashboardId);
  }

  @Post(':moduleId/dashboards/:dashboardId/export')
  @RequirePermission('edit')
  async exportDashboard(
    @Param('moduleId') moduleId: string,
    @Param('dashboardId') dashboardId: string
  ) {
    // Verifica insights_dashboard:dashboardId#export_data
    return this.insightsService.exportDashboard(dashboardId);
  }
}
```

### Resultado das Permissões

| Usuário | view_trends | export_data | configure_filters |
|:---|:---:|:---:|:---:|
| **Carlos** (module.owner) | ✅ | ✅ | ✅ |
| **Alice** (analyst) | ✅ | ✅ | ❌ |
| **Bob** (data_viewer) | ✅ | ❌ | ❌ |

## Módulo B2B (Partners/Contracts)

### Schema

`permify/schemas/modules/b2b.permify`

```permify
entity b2b_partner {
  relation organization @organization
  relation module @module
  relation partner_manager @user
  relation account_owner @user
  
  permission view = partner_manager or account_owner or module.viewer or organization.admin
  permission edit = account_owner or module.editor or organization.admin
  permission manage_users = account_owner or organization.admin
}

entity b2b_contract {
  relation partner @b2b_partner
  relation organization @organization
  relation approver @user
  relation signer @user
  
  permission view = partner.view or organization.admin
  permission edit = signer or organization.admin
  permission approve = approver or organization.admin
  permission sign = signer or organization.admin
}
```

### Tuples de Exemplo

```json
{
  "tuples": [
    {
      "entity": {"type": "module", "id": "b2b"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "santa-cruz"}
    },
    {
      "entity": {"type": "module", "id": "b2b"},
      "relation": "editor_user",
      "subject": {"type": "user", "id": "bob"}
    },
    {
      "entity": {"type": "b2b_partner", "id": "partner-123"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "santa-cruz"}
    },
    {
      "entity": {"type": "b2b_partner", "id": "partner-123"},
      "relation": "account_owner",
      "subject": {"type": "user", "id": "charlie"}
    },
    {
      "entity": {"type": "b2b_contract", "id": "contract-456"},
      "relation": "partner",
      "subject": {"type": "b2b_partner", "id": "partner-123"}
    },
    {
      "entity": {"type": "b2b_contract", "id": "contract-456"},
      "relation": "approver",
      "subject": {"type": "user", "id": "david"}
    }
  ]
}
```

### Guards (Spring Boot)

```java
@RestController
@RequestMapping("/api/b2b")
public class B2BController {

    @GetMapping("/partners/{partnerId}")
    @RequirePermission(value = "view", entityType = "b2b_partner", idExpression = "#partnerId")
    public ResponseEntity<B2BPartner> getPartner(@PathVariable String partnerId) {
        return ResponseEntity.ok(b2bService.findById(partnerId));
    }

    @PostMapping("/partners/{partnerId}/users")
    @RequirePermission(value = "manage_users", entityType = "b2b_partner", idExpression = "#partnerId")
    public ResponseEntity<Void> addUserToPartner(
        @PathVariable String partnerId,
        @RequestBody AddUserRequest request
    ) {
        b2bService.addUser(partnerId, request.getUserId());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/contracts/{contractId}")
    @RequirePermission(value = "view", entityType = "b2b_contract", idExpression = "#contractId")
    public ResponseEntity<B2BContract> getContract(@PathVariable String contractId) {
        return ResponseEntity.ok(contractService.findById(contractId));
    }

    @PostMapping("/contracts/{contractId}/approve")
    @RequirePermission(value = "approve", entityType = "b2b_contract", idExpression = "#contractId")
    public ResponseEntity<Void> approveContract(@PathVariable String contractId) {
        contractService.approve(contractId);
        return ResponseEntity.ok().build();
    }
}
```

### Resultado das Permissões

| Usuário | partner-123:view | contract-456:view | contract-456:approve |
|:---|:---:|:---:|:---:|
| **Bob** (module.editor) | ✅ | ✅ | ✅ |
| **Charlie** (account_owner) | ✅ | ✅ | ❌ |
| **David** (approver) | ❌ | ✅ | ✅ |

## Módulo Finance

### Schema

`permify/schemas/modules/finance.permify`

```permify
entity finance_invoice {
  relation organization @organization
  relation company @company
  relation module @module
  relation accountant @user
  relation viewer @user
  
  permission view = viewer or accountant or module.viewer or organization.admin or company.admin
  permission edit = accountant or module.editor or organization.admin or company.admin
  permission approve = accountant or module.owner or organization.admin or company.admin
  permission pay = accountant or organization.admin or company.admin
}

entity finance_payment {
  relation invoice @finance_invoice
  relation processor @user
  relation viewer @user
  
  permission view = viewer or processor or invoice.view
  permission process = processor or invoice.approve
  permission refund = processor or invoice.approve
}
```

### Tuples com Hierarquia

```json
{
  "tuples": [
    {
      "entity": {"type": "organization", "id": "santa-cruz"},
      "relation": "admin",
      "subject": {"type": "user", "id": "bob"}
    },
    {
      "entity": {"type": "company", "id": "sc1"},
      "relation": "organization",
      "subject": {"type": "organization", "id": "santa-cruz"}
    },
    {
      "entity": {"type": "company", "id": "sc1"},
      "relation": "admin",
      "subject": {"type": "user", "id": "eve"}
    },
    {
      "entity": {"type": "module", "id": "finance"},
      "relation": "company",
      "subject": {"type": "company", "id": "sc1"}
    },
    {
      "entity": {"type": "module", "id": "finance"},
      "relation": "owner_user",
      "subject": {"type": "user", "id": "eve"}
    },
    {
      "entity": {"type": "finance_invoice", "id": "inv-123"},
      "relation": "company",
      "subject": {"type": "company", "id": "sc1"}
    },
    {
      "entity": {"type": "finance_invoice", "id": "inv-123"},
      "relation": "accountant",
      "subject": {"type": "user", "id": "frank"}
    }
  ]
}
```

### Teste de Permissões

```typescript
// Testes manuais via API
async function testPermissions() {
  const client = new PermifyClient({ endpoint: 'http://localhost:3476' });
  const tenantId = 'dev';

  // Bob (organization.admin) pode tudo
  console.log('Bob view invoice:', await client.check({
    tenantId,
    entity: { type: 'finance_invoice', id: 'inv-123' },
    permission: 'view',
    subject: { type: 'user', id: 'bob' }
  })); // ALLOW (herança)

  // Eve (company.admin + module.owner) pode aprovar
  console.log('Eve approve invoice:', await client.check({
    tenantId,
    entity: { type: 'finance_invoice', id: 'inv-123' },
    permission: 'approve',
    subject: { type: 'user', id: 'eve' }
  })); // ALLOW

  // Frank (accountant) pode editar mas não aprovar
  console.log('Frank edit invoice:', await client.check({
    tenantId,
    entity: { type: 'finance_invoice', id: 'inv-123' },
    permission: 'edit',
    subject: { type: 'user', id: 'frank' }
  })); // ALLOW

  console.log('Frank approve invoice:', await client.check({
    tenantId,
    entity: { type: 'finance_invoice', id: 'inv-123' },
    permission: 'approve',
    subject: { type: 'user', id: 'frank' }
  })); // DENY
}
```

## Frontend - React Hooks

### Hook de Permissões

```typescript
// hooks/usePermissions.ts
import { useQuery } from '@tanstack/react-query';
import { api } from '../services/api';

interface Permission {
  resource: string;
  action: string;
  allowed: boolean;
}

export function usePermissions() {
  return useQuery<Permission[]>({
    queryKey: ['permissions'],
    queryFn: async () => {
      const { data } = await api.get('/api/v4/permissions/me');
      return data.permissions;
    },
    staleTime: 5 * 60 * 1000, // 5 minutos
  });
}

export function useCan(resource: string, action: string) {
  const { data } = usePermissions();
  
  return {
    can: data?.some(p => 
      p.resource === resource && p.action === action && p.allowed
    ) ?? false,
    isLoading: !data
  };
}
```

### Componente Protegido

```tsx
// components/ProtectedComponent.tsx
import { useCan } from '../hooks/usePermissions';

interface ProtectedComponentProps {
  resource: string;
  action: string;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function ProtectedComponent({ 
  resource, 
  action, 
  children, 
  fallback = null 
}: ProtectedComponentProps) {
  const { can, isLoading } = useCan(resource, action);

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!can) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
}

// Uso:
function InvoiceActions({ invoiceId }: { invoiceId: string }) {
  return (
    <div>
      <ProtectedComponent 
        resource={`finance_invoice:${invoiceId}`} 
        action="edit"
      >
        <button onClick={() => editInvoice(invoiceId)}>
          Edit Invoice
        </button>
      </ProtectedComponent>

      <ProtectedComponent 
        resource={`finance_invoice:${invoiceId}`} 
        action="approve"
      >
        <button onClick={() => approveInvoice(invoiceId)}>
          Approve
        </button>
      </ProtectedComponent>
    </div>
  );
}
```

## Testes Automatizados

### Teste de Integração (NestJS)

```typescript
// test/permissions.e2e-spec.ts
describe('Permissions (e2e)', () => {
  let app: INestApplication;
  let permifyClient: Client;

  beforeEach(async () => {
    app = await createTestModule();
    permifyClient = app.get(PERMIFY_CLIENT);
  });

  describe('Insights Module', () => {
    it('should allow analyst to view but not configure', async () => {
      // Setup: Alice como analyst
      await permifyClient.tuple.write({
        tenantId: 'test',
        tuple: {
          entity: { type: 'insights_dashboard', id: 'test' },
          relation: 'analyst',
          subject: { type: 'user', id: 'alice' }
        }
      });

      // Test: Alice pode ver
      const viewResponse = await request(app.getHttpServer())
        .get('/api/insights/test/dashboards/test')
        .set('Authorization', 'Bearer alice-token')
        .set('x-org-id', 'test-org');

      expect(viewResponse.status).toBe(200);

      // Test: Alice não pode configurar
      const configResponse = await request(app.getHttpServer())
        .post('/api/insights/test/dashboards/test/configure')
        .set('Authorization', 'Bearer alice-token')
        .set('x-org-id', 'test-org');

      expect(configResponse.status).toBe(403);
    });
  });
});
```

### Teste Unitário (Spring Boot)

```java
@SpringBootTest
@AutoConfigureTestDatabase
public class PermissionServiceTest {

    @Autowired
    private PermissionService permissionService;

    @MockBean
    private PermifyClient permifyClient;

    @Test
    void testContractApproval() {
        // Mock: David pode aprovar contract-456
        when(permifyClient.check(any(CheckRequest.class)))
            .thenReturn(CheckResponse.newBuilder()
                .setCan(CheckResult.CHECK_RESULT_ALLOWED)
                .build());

        // Test
        boolean canApprove = permissionService.check(
            "b2b_contract", 
            "contract-456", 
            "approve", 
            "david"
        );

        assertTrue(canApprove);
        verify(permifyClient).check(argThat(req -> 
            req.getEntity().getId().equals("contract-456") &&
            req.getPermission().equals("approve") &&
            req.getSubject().getId().equals("david")
        ));
    }
}
```

## Performance - Lookup Entity

### Problema N+1

```typescript
// ❌ NÃO FAZER - N+1 queries
async function getVisibleReportsBad(userId: string) {
  const allReports = await database.getAllReports();
  const visible = [];
  
  for (const report of allReports) {
    const canView = await permify.check({
      entity: { type: 'insights_report', id: report.id },
      permission: 'view',
      subject: { type: 'user', id: userId }
    });
    
    if (canView) visible.push(report);
  }
  
  return visible; // 100 reports = 100 chamadas ao Permify
}
```

### Solução com Lookup

```typescript
// ✅ FAZER - Lookup + filtro em DB
async function getVisibleReportsGood(userId: string) {
  // 1. Lookup: quais reports posso ver?
  const lookup = await permify.lookupEntity({
    entityType: 'insights_report',
    permission: 'view',
    subject: { type: 'user', id: userId }
  });

  const allowedIds = lookup.entities.map(e => e.id);

  // 2. Filtro no banco (1 query só)
  const reports = await database.query(`
    SELECT * FROM insights_reports 
    WHERE id = ANY($1) 
    ORDER BY created_at DESC
  `, [allowedIds]);

  return reports; // 1 lookup + 1 query DB
}
```

## Debug e Troubleshooting

### Verificar Tuples

```bash
# Verificar todos os tuples de um usuário
curl -X POST http://localhost:3476/v1/tenants/dev/tuples/read \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "subject": {"type": "user", "id": "alice"}
    }
  }'

# Verificar tuples de uma entidade
curl -X POST http://localhost:3476/v1/tenants/dev/tuples/read \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "entity": {"type": "module", "id": "insights"}
    }
  }'
```

### Testar Check Manual

```bash
# Testar permissão específica
curl -X POST http://localhost:3476/v1/tenants/dev/permissions/check \
  -H "Content-Type: application/json" \
  -d '{
    "entity": {"type": "module", "id": "insights"},
    "permission": "view",
    "subject": {"type": "user", "id": "alice"},
    "metadata": {"depth": 20}
  }'

# Resposta esperado:
# {"can": "CHECK_RESULT_ALLOWED"}
# ou
# {"can": "CHECK_RESULT_DENIED"}
```

### Playground Online

Use o [Permify Playground](https://play.permify.co) para:
- Testar schemas sem instalar
- Simular checks e lookups
- Visualizar o grafo de permissões

---

## Referências

- [Integração NestJS + Spring Boot](Integração-NestJS-+-Spring-Boot) - Exemplos completos
- [Configuração do Permify](Configuração-do-Permify) - Setup e deploy
- [PRD Completo](PRD---Permify-AuthZ) - Requisitos e decisões
- [Arquitetura AuthN + AuthZ](Arquitetura-AuthN-+-AuthZ) - Fluxo completo

---

**Platform Team · 2026**