# Integração Permify — NestJS e Spring Boot

**Versão**: 1.0  
**Data**: Fevereiro 2026  
**Owner**: Platform Team  
**Status**: Draft

---

## 1. SDKs e Dependências

### NestJS (Node.js / TypeScript)

```bash
npm install @permify/permify-node
```

> SDK oficial: [github.com/Permify/permify-node](https://github.com/Permify/permify-node)  
> Protocolo: **gRPC** (porta `3478`) ou **REST** (porta `3476`)

### Spring Boot (Java)

```xml
<!-- pom.xml -->
<dependency>
  <groupId>is.permify</groupId>
  <artifactId>permify-java</artifactId>
  <version>0.1.0</version>
</dependency>
```

> SDK oficial: [github.com/Permify/permify-java](https://github.com/Permify/permify-java)

**Variáveis de ambiente obrigatórias (ambos):**

| Variável | Descrição | Exemplo |
|:---|:---|:---|
| `PERMIFY_URL` | Endpoint do Permify | `http://permify:3476` |
| `PERMIFY_TENANT_ID` | Tenant fixo por ambiente | `dev`, `staging`, `production` |

> **Regra:** `PERMIFY_TENANT_ID` é configurado no deploy — nunca resolvido dinamicamente a partir do request.

---

## 2. Configuração do Client

### NestJS — PermifyModule

```typescript
// permify/permify.module.ts
import { Module, Global } from '@nestjs/common';
import * as permify from '@permify/permify-node';

export const PERMIFY_CLIENT = 'PERMIFY_CLIENT';

@Global()
@Module({
  providers: [
    {
      provide: PERMIFY_CLIENT,
      useFactory: () =>
        permify.grpc.newClient({
          endpoint: process.env.PERMIFY_URL ?? 'localhost:3478',
        }),
    },
  ],
  exports: [PERMIFY_CLIENT],
})
export class PermifyModule {}
```

```typescript
// app.module.ts
import { PermifyModule } from './permify/permify.module';

@Module({ imports: [PermifyModule] })
export class AppModule {}
```

### Spring Boot — PermifyClient Bean

```java
// config/PermifyConfig.java
@Configuration
public class PermifyConfig {

    @Value("${permify.url:http://localhost:3476}")
    private String permifyUrl;

    @Bean
    public PermifyClient permifyClient() {
        return PermifyClient.builder()
            .endpoint(permifyUrl)
            .build();
    }
}
```

```yaml
# application.yml
permify:
  url: ${PERMIFY_URL:http://localhost:3476}
  tenant-id: ${PERMIFY_TENANT_ID:dev}
```

---

## 3. Arquitetura: Três Fluxos Distintos

O Permify é consultado em **três camadas independentes**, cada uma com responsabilidade e granularidade diferente. O **frontend nunca consulta o Permify diretamente** — ele consome o endpoint `/api/v4/permissions/me` do BFF.

```
┌─────────────────────────────────────────────────────────────────┐
│                         USUÁRIO (Browser)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │ JWT + x-org-id
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              FLUXO 1 — API Gateway (Lambda Authorizer)          │
│                                                                 │
│  • Valida JWT via JWKS (Keycloak)                               │
│  • Mapeia rota → entidade + permissão                           │
│  • check(module, "view", user) — coarse-grained                 │
│  • Cache TTL: 300s por token+resource                           │
│  • DENY → 403 (request nunca chega ao BFF)                      │
│  • ALLOW → passa userId + orgId no context                      │
└────────────────────────────┬────────────────────────────────────┘
                             │ userId + orgId no context
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              FLUXO 2 — BFF / NestJS (Guards)                    │
│                                                                 │
│  • Recebe userId/orgId do Gateway (não decodifica JWT)          │
│  • check(module, "edit", user) — medium-grained                 │
│  • lookup-entity para endpoints de lista                        │
│  • Cache TTL: 2–30s em memória                                  │
│  • DENY → 403                                                   │
│  • Expõe GET /api/v4/permissions/me para o frontend             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              FLUXO 3 — Microserviço (resource-level)            │
│                                                                 │
│  • check(b2b_contract:inv-123, "approve", user) — fine-grained  │
│  • Recebe userId/orgId do context do Gateway                    │
│  • Usado quando o BFF não tem contexto suficiente               │
│  • Workers/jobs assíncronos também checam aqui                  │
│  • DENY → 403 / rejeita processamento                           │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              FRONTEND (Next.js / React)                         │
│                                                                 │
│  • NÃO consulta Permify diretamente                             │
│  • Chama GET /api/v4/permissions/me (BFF)                       │
│  • Renderiza UI condicionalmente (botões, menus, rotas)         │
│  • Proteção real está nas camadas 1, 2 e 3                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Fluxo 1 — API Gateway (Lambda Authorizer — Node.js)

Lambda Authorizer em Node.js/TypeScript que faz o check coarse-grained antes de qualquer request chegar ao BFF.

```typescript
// lambda/authorizer.ts
import * as permify from '@permify/permify-node';

const ROUTE_MAP = [
  { pattern: /^GET \/api\/insights/,        entity: 'module', id: 'insights', permission: 'view'   },
  { pattern: /^POST \/api\/insights/,       entity: 'module', id: 'insights', permission: 'edit'   },
  { pattern: /^DELETE \/api\/modules/,      entity: 'module', id: 'insights', permission: 'delete' },
  { pattern: /^POST \/api\/b2b\/contracts/, entity: 'module', id: 'b2b',      permission: 'edit'   },
];

const client = permify.grpc.newClient({ endpoint: process.env.PERMIFY_URL! });

export const handler = async (event: any) => {
  const tenantId = process.env.PERMIFY_TENANT_ID!;
  const userId   = event.requestContext?.authorizer?.userId;
  const orgId    = event.headers?.['x-org-id'];
  const method   = event.requestContext?.httpMethod;
  const path     = event.requestContext?.path;

  if (!userId || !orgId) return deny(event.methodArn);

  const rule = ROUTE_MAP.find(r => r.pattern.test(`${method} ${path}`));
  if (!rule) return allow(userId, event.methodArn, { userId, orgId });

  try {
    const result = await client.permission.check({
      tenantId,
      metadata:   { snapToken: '', schemaVersion: '', depth: 20 },
      entity:     { type: rule.entity, id: `${rule.id}-${orgId}` },
      permission: rule.permission,
      subject:    { type: 'user', id: userId },
    });

    return result.can 
      ? allow(userId, event.methodArn, { userId, orgId })
      : deny(event.methodArn);
  } catch (err) {
    console.error('Permify check failed:', err);
    return deny(event.methodArn); // Fail-closed
  }
};

function allow(principalId: string, resourceArn: string, context: any) {
  return {
    principalId,
    policyDocument: {
      Version: '2012-10-17',
      Statement: [{ Effect: 'Allow', Action: 'execute-api:Invoke', Resource: resourceArn }],
    },
    context,
    usageIdentifierKey: undefined,
  };
}

function deny(resourceArn: string) {
  return {
    principalId: 'anonymous',
    policyDocument: {
      Version: '2012-10-17',
      Statement: [{ Effect: 'Deny', Action: 'execute-api:Invoke', Resource: resourceArn }],
    },
    usageIdentifierKey: undefined,
  };
}
```

---

## 5. Fluxo 2 — BFF com NestJS

### 5.1 Guard de permissão

```typescript
// permify/permify.guard.ts
import { CanActivate, ExecutionContext, ForbiddenException,
         Injectable, Inject, SetMetadata } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Client } from '@permify/permify-node';
import { PERMIFY_CLIENT } from './permify.module';

export const REQUIRE_PERMISSION = 'REQUIRE_PERMISSION';
export const RequirePermission = (permission: string) => 
  SetMetadata(REQUIRE_PERMISSION, permission);

@Injectable()
export class PermifyGuard implements CanActivate {
  constructor(
    @Inject(PERMIFY_CLIENT) private readonly client: Client,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const permission = this.reflector.get<string>(
      REQUIRE_PERMISSION,
      context.getHandler(),
    );
    
    if (!permission) return true; // Sem decorator = público

    const request = context.switchToHttp().getRequest();
    const { userId, orgId } = request; // Vem do Lambda Authorizer
    
    const { moduleId } = request.params;
    const tenantId = process.env.PERMIFY_TENANT_ID!;

    try {
      const result = await this.client.permission.check({
        tenantId,
        metadata: { snapToken: '', schemaVersion: '', depth: 20 },
        entity: { type: 'module', id: `${moduleId}-${orgId}` },
        permission,
        subject: { type: 'user', id: userId },
      });

      if (!result.can) {
        throw new ForbiddenException('Access denied');
      }

      // Log obrigatório
      console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        event: 'authz_check',
        userId,
        orgId,
        moduleId,
        permission,
        decision: 'ALLOW',
        source: 'bff_guard'
      }));

      return true;
    } catch (err) {
      if (err instanceof ForbiddenException) throw err;
      
      console.error('Permify check failed:', err);
      throw new ForbiddenException('Authorization service unavailable');
    }
  }
}
```

### 5.2 Controller com Guard

```typescript
// insights/insights.controller.ts
import { Controller, Get, Post, Param, Query } from '@nestjs/common';
import { RequirePermission, PermifyGuard } from '../permify/permify.guard';
import { InsightsService } from './insights.service';

@Controller('api/insights')
@UseGuards(PermifyGuard)
export class InsightsController {
  constructor(private readonly insightsService: InsightsService) {}

  @Get(':moduleId/reports')
  @RequirePermission('view')
  async getReports(@Param('moduleId') moduleId: string) {
    return this.insightsService.getReports(moduleId);
  }

  @Post(':moduleId/reports')
  @RequirePermission('edit')
  async createReport(@Param('moduleId') moduleId: string, body: any) {
    return this.insightsService.createReport(moduleId, body);
  }
}
```

### 5.3 Service com lookup-entity (listas)

```typescript
// insights/insights.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { Client } from '@permify/permify-node';
import { PERMIFY_CLIENT } from '../permify/permify.module';

@Injectable()
export class InsightsService {
  constructor(@Inject(PERMIFY_CLIENT) private readonly client: Client) {}

  async getReports(moduleId: string) {
    const tenantId = process.env.PERMIFY_TENANT_ID!;
    const { userId, orgId } = this.getCurrentUser(); // Vem do request

    // 1. Check coarse: pode acessar o módulo?
    const canAccess = await this.client.permission.check({
      tenantId,
      metadata: { snapToken: '', schemaVersion: '', depth: 20 },
      entity: { type: 'module', id: `${moduleId}-${orgId}` },
      permission: 'view',
      subject: { type: 'user', id: userId },
    });

    if (!canAccess.can) {
      throw new ForbiddenException('Access denied');
    }

    // 2. Lookup: quais reports pode ver?
    const allowed = await this.client.permission.lookupEntity({
      tenantId,
      metadata: { snapToken: '', schemaVersion: '', depth: 20 },
      entityType: 'insights_report',
      permission: 'view',
      subject: { type: 'user', id: userId },
    });

    // 3. Filtra no banco
    return this.database.query(`
      SELECT * FROM insights_reports 
      WHERE module_id = $1 
        AND id = ANY($2)
    `, [moduleId, allowed.entities.map(e => e.id)]);
  }

  private getCurrentUser() {
    // Implementar extração do request context
    return { userId: 'user-123', orgId: 'clickbus' };
  }
}
```

### 5.4 Endpoint de permissões (para o frontend)

```typescript
// permissions/permissions.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { PermifyGuard } from '../permify/permify.guard';
import { PermissionsService } from './permissions.service';

@Controller('api/v4/permissions')
@UseGuards(PermifyGuard)
export class PermissionsController {
  constructor(private readonly permissionsService: PermissionsService) {}

  @Get('me')
  async getMyPermissions() {
    return this.permissionsService.getUserPermissions();
  }
}

// permissions/permissions.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { Client } from '@permify/permify-node';
import { PERMIFY_CLIENT } from '../permify/permify.module';

@Injectable()
export class PermissionsService {
  constructor(@Inject(PERMIFY_CLIENT) private readonly client: Client) {}

  async getUserPermissions() {
    const tenantId = process.env.PERMIFY_TENANT_ID!;
    const { userId, orgId } = this.getCurrentUser();

    // Lookup de todos os módulos que pode ver
    const modules = await this.client.permission.lookupEntity({
      tenantId,
      metadata: { snapToken: '', schemaVersion: '', depth: 20 },
      entityType: 'module',
      permission: 'view',
      subject: { type: 'user', id: userId },
    });

    // Para cada módulo, busca ações permitidas
    const permissions = [];
    for (const module of modules.entities) {
      const actions = ['view', 'edit', 'manage'];
      for (const action of actions) {
        const can = await this.client.permission.check({
          tenantId,
          metadata: { snapToken: '', schemaVersion: '', depth: 20 },
          entity: { type: 'module', id: module.id },
          permission: action,
          subject: { type: 'user', id: userId },
        });

        if (can.can) {
          permissions.push({
            module: module.id,
            action,
            resource: 'module'
          });
        }
      }
    }

    return {
      userId,
      orgId,
      permissions,
      timestamp: new Date().toISOString()
    };
  }

  private getCurrentUser() {
    return { userId: 'user-123', orgId: 'clickbus' };
  }
}
```

---

## 6. Fluxo 3 — Microserviço com Spring Boot

### 6.1 PermifyService

```java
// service/PermifyService.java
@Service
@Slf4j
public class PermifyService {

    @Autowired
    private PermifyClient permifyClient;

    @Value("${permify.tenant-id}")
    private String tenantId;

    public boolean check(String entityType, String entityId, String permission, String userId) {
        try {
            CheckRequest request = CheckRequest.newBuilder()
                .setTenantId(tenantId)
                .setMetadata(Metadata.newBuilder()
                    .setSnapToken("")
                    .setSchemaVersion("")
                    .setDepth(20)
                    .build())
                .setEntity(Entity.newBuilder()
                    .setType(entityType)
                    .setId(entityId)
                    .build())
                .setPermission(permission)
                .setSubject(Subject.newBuilder()
                    .setType("user")
                    .setId(userId)
                    .build())
                .build();

            CheckResponse response = permifyClient.check(request);
            
            log.info("AuthZ check: user={}, entity={}, permission={}, result={}", 
                userId, entityType + ":" + entityId, permission, response.getCan());
            
            return response.getCan();
        } catch (Exception e) {
            log.error("Permify check failed", e);
            return false; // Fail-closed
        }
    }

    public List<String> lookupResources(String entityType, String permission, String userId) {
        try {
            LookupEntityRequest request = LookupEntityRequest.newBuilder()
                .setTenantId(tenantId)
                .setMetadata(Metadata.newBuilder()
                    .setSnapToken("")
                    .setSchemaVersion("")
                    .setDepth(20)
                    .build())
                .setEntityType(entityType)
                .setPermission(permission)
                .setSubject(Subject.newBuilder()
                    .setType("user")
                    .setId(userId)
                    .build())
                .build();

            LookupEntityResponse response = permifyClient.lookupEntity(request);
            return response.getEntityList().stream()
                .map(Entity::getId)
                .collect(Collectors.toList());
        } catch (Exception e) {
            log.error("Permify lookup failed", e);
            return Collections.emptyList();
        }
    }
}
```

### 6.2 Anotação @RequirePermission

```java
// annotation/RequirePermission.java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();
    String entityType() default "module";
    String idExpression() default "#request.moduleId";
}
```

### 6.3 Aspect para AOP

```java
// aspect/PermifyAspect.java
@Aspect
@Component
@Slf4j
public class PermifyAspect {

    @Autowired
    private PermifyService permifyService;

    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint, RequirePermission requirePermission) throws Throwable {
        
        // Extrai userId do contexto (vem do API Gateway)
        String userId = SecurityContextHolder.getContext().getAuthentication().getName();
        
        // Extrai entityId da expressão SpEL
        StandardEvaluationContext context = new StandardEvaluationContext();
        context.setVariable("request", getRequestFromJoinPoint(joinPoint));
        
        ExpressionParser parser = new SpelExpressionParser();
        String entityId = parser.parseExpression(requirePermission.idExpression()).getValue(context, String.class);
        
        boolean allowed = permifyService.check(
            requirePermission.entityType(),
            entityId,
            requirePermission.value(),
            userId
        );
        
        if (!allowed) {
            log.warn("Access denied: user={}, entity={}, permission={}", 
                userId, requirePermission.entityType() + ":" + entityId, requirePermission.value());
            throw new AccessDeniedException("Access denied");
        }
        
        return joinPoint.proceed();
    }

    private HttpServletRequest getRequestFromJoinPoint(ProceedingJoinPoint joinPoint) {
        for (Object arg : joinPoint.getArgs()) {
            if (arg instanceof HttpServletRequest) {
                return (HttpServletRequest) arg;
            }
        }
        throw new IllegalArgumentException("HttpServletRequest not found in method arguments");
    }
}
```

### 6.4 Controller com Anotação

```java
// controller/ContractController.java
@RestController
@RequestMapping("/api/b2b")
@Slf4j
public class ContractController {

    @Autowired
    private ContractService contractService;

    @GetMapping("/contracts/{contractId}")
    @RequirePermission(value = "view", entityType = "b2b_contract", idExpression = "#contractId")
    public ResponseEntity<Contract> getContract(@PathVariable String contractId) {
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

### 6.5 Worker Assíncrono (SQS Listener)

```java
// worker/InvoiceProcessor.java
@Component
@Slf4j
public class InvoiceProcessor {

    @Autowired
    private PermifyService permifyService;

    @Autowired
    private InvoiceService invoiceService;

    @SqsListener(queueName = "${sqs.invoice-queue}")
    public void processInvoice(InvoiceMessage message) {
        log.info("Processing invoice: {}", message.getInvoiceId());

        // Check de permissão fine-grained
        boolean canProcess = permifyService.check(
            "invoice",
            message.getInvoiceId(),
            "process",
            message.getUserId()
        );

        if (!canProcess) {
            log.warn("User {} not allowed to process invoice {}", 
                message.getUserId(), message.getInvoiceId());
            return;
        }

        try {
            invoiceService.process(message.getInvoiceId(), message.getUserId());
            log.info("Invoice processed successfully: {}", message.getInvoiceId());
        } catch (Exception e) {
            log.error("Failed to process invoice: " + message.getInvoiceId(), e);
            throw e;
        }
    }
}
```

---

## 7. Frontend — Consumindo o BFF

### 7.1 Hook de permissões (React/Next.js)

```typescript
// hooks/usePermissions.ts
import { useQuery } from '@tanstack/react-query';
import { api } from '../services/api';

interface Permission {
  module: string;
  action: string;
  resource: string;
}

interface PermissionsResponse {
  userId: string;
  orgId: string;
  permissions: Permission[];
  timestamp: string;
}

export function usePermissions() {
  return useQuery<PermissionsResponse>({
    queryKey: ['permissions'],
    queryFn: async () => {
      const { data } = await api.get('/api/v4/permissions/me');
      return data;
    },
    staleTime: 5 * 60 * 1000, // 5 minutos
  });
}

export function useCan(module: string, action: string) {
  const { data } = usePermissions();
  
  return {
    can: data?.permissions.some(p => 
      p.module === module && p.action === action
    ) ?? false
  };
}
```

### 7.2 Componente condicional

```tsx
// components/ProtectedButton.tsx
import { useCan } from '../hooks/usePermissions';

interface ProtectedButtonProps {
  module: string;
  action: string;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function ProtectedButton({ module, action, children, fallback }: ProtectedButtonProps) {
  const { can } = useCan(module, action);

  if (!can) {
    return <>{fallback || null}</>;
  }

  return <>{children}</>;
}

// Uso:
<ProtectedButton module="insights" action="edit" fallback={<span>Sem permissão</span>}>
  <button onClick={handleEdit}>Editar Relatório</button>
</ProtectedButton>
```

### 7.3 Guard de rota (Next.js)

```tsx
// components/RouteGuard.tsx
import { useCan } from '../hooks/usePermissions';
import { useRouter } from 'next/router';
import { useEffect } from 'react';

interface RouteGuardProps {
  module: string;
  action: string;
  children: React.ReactNode;
}

export function RouteGuard({ module, action, children }: RouteGuardProps) {
  const { can } = useCan(module, action);
  const router = useRouter();

  useEffect(() => {
    if (can === false) {
      router.push('/unauthorized');
    }
  }, [can, router]);

  if (can === false) {
    return <div>Verificando permissões...</div>;
  }

  return <>{children}</>;
}

// pages/insights/[moduleId]/reports.tsx
export default function ReportsPage() {
  return (
    <RouteGuard module="insights" action="view">
      <ReportsList />
    </RouteGuard>
  );
}
```

---

## 8. Admin Plane — Escrita de Tuples

### 8.1 NestJS — TuplesService

```typescript
// admin/tuples.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { Client } from '@permify/permify-node';
import { PERMIFY_CLIENT } from '../permify/permify.module';

@Injectable()
export class TuplesService {
  constructor(@Inject(PERMIFY_CLIENT) private readonly client: Client) {}

  async assignRoleToUser(
    entityType: string,
    entityId: string,
    relation: string,
    userId: string
  ) {
    const tenantId = process.env.PERMIFY_TENANT_ID!;

    await this.client.tuple.write({
      tenantId,
      metadata: { snapToken: '', schemaVersion: '' },
      tuple: {
        entity: { type: entityType, id: entityId },
        relation,
        subject: { type: 'user', id: userId },
      },
    });

    console.log(`Role assigned: ${entityType}:${entityId}#${relation}@user:${userId}`);
  }

  async removeRoleFromUser(
    entityType: string,
    entityId: string,
    relation: string,
    userId: string
  ) {
    const tenantId = process.env.PERMIFY_TENANT_ID!;

    await this.client.tuple.delete({
      tenantId,
      metadata: { snapToken: '', schemaVersion: '' },
      tuple: {
        entity: { type: entityType, id: entityId },
        relation,
        subject: { type: 'user', id: userId },
      },
    });

    console.log(`Role removed: ${entityType}:${entityId}#${relation}@user:${userId}`);
  }
}
```

### 8.2 Spring Boot — TuplesService

```java
// service/TuplesService.java
@Service
@Slf4j
public class TuplesService {

    @Autowired
    private PermifyClient permifyClient;

    @Value("${permify.tenant-id}")
    private String tenantId;

    public void assignRoleToUser(String entityType, String entityId, String relation, String userId) {
        try {
            TupleWriteRequest request = TupleWriteRequest.newBuilder()
                .setTenantId(tenantId)
                .setMetadata(Metadata.newBuilder()
                    .setSnapToken("")
                    .setSchemaVersion("")
                    .build())
                .setTuple(Tuple.newBuilder()
                    .setEntity(Entity.newBuilder()
                        .setType(entityType)
                        .setId(entityId)
                        .build())
                    .setRelation(relation)
                    .setSubject(Subject.newBuilder()
                        .setType("user")
                        .setId(userId)
                        .build())
                    .build())
                .build();

            permifyClient.tuple().write(request);
            log.info("Role assigned: {}:{}#{}@user:{}", entityType, entityId, relation, userId);
        } catch (Exception e) {
            log.error("Failed to assign role", e);
            throw new RuntimeException("Failed to assign role", e);
        }
    }

    public void removeRoleFromUser(String entityType, String entityId, String relation, String userId) {
        try {
            TupleDeleteRequest request = TupleDeleteRequest.newBuilder()
                .setTenantId(tenantId)
                .setMetadata(Metadata.newBuilder()
                    .setSnapToken("")
                    .setSchemaVersion("")
                    .build())
                .setTuple(Tuple.newBuilder()
                    .setEntity(Entity.newBuilder()
                        .setType(entityType)
                        .setId(entityId)
                        .build())
                    .setRelation(relation)
                    .setSubject(Subject.newBuilder()
                        .setType("user")
                        .setId(userId)
                        .build())
                    .build())
                .build();

            permifyClient.tuple().delete(request);
            log.info("Role removed: {}:{}#{}@user:{}", entityType, entityId, relation, userId);
        } catch (Exception e) {
            log.error("Failed to remove role", e);
            throw new RuntimeException("Failed to remove role", e);
        }
    }
}
```

---

## 9. Padrões Obrigatórios

### 9.1 Fail-Closed

```typescript
// Sempre deny em caso de erro
try {
  const result = await client.permission.check(params);
  return result.can;
} catch (error) {
  console.error('Permify unavailable:', error);
  return false; // Fail-closed
}
```

### 9.2 Logs Obrigatórios

```typescript
// Estrutura de log padrão
console.log(JSON.stringify({
  timestamp: new Date().toISOString(),
  event: 'authz_check',
  userId: 'user-123',
  orgId: 'clickbus',
  resourceType: 'module',
  resourceId: 'insights-clickbus',
  action: 'view',
  decision: 'ALLOW', // ou 'DENY'
  latencyMs: 12,
  source: 'bff_guard', // ou 'gateway_lambda', 'microservice'
  traceId: 'abc-123'
}));
```

### 9.3 Evitar N+1 com lookup-entity

```typescript
// ❌ NÃO FAZER - N+1 checks
const reports = await database.getReports();
const filtered = [];
for (const report of reports) {
  const can = await check('report', report.id, 'view', userId);
  if (can) filtered.push(report);
}

// ✅ FAZER - lookup-entity + filtro em DB
const allowedIds = await lookupEntity('report', 'view', userId);
const reports = await database.query(`
  SELECT * FROM reports 
  WHERE id = ANY($1)
`, [allowedIds]);
```

---

## 10. Referências

- [Permify Documentation](https://permify.co/docs)
- [SDK Node.js](https://github.com/Permify/permify-node)
- [SDK Java](https://github.com/Permify/permify-java)
- [PRD Completo](PRD---Permify-AuthZ)
- [Arquitetura AuthN + AuthZ](Arquitetura-AuthN-+-AuthZ)

---

**Platform Team · 2026**