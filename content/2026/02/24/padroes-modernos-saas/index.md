---
title: "Padrões modernos em SaaS multi-tenant"
slug: padroes-modernos-saas
date: 2026-02-24T14:29:42-0300
draft: false
description: "Padrões modernos para SaaS multi-tenant: eventos, APIs e integrações (sem perder isolamento)"
tags:
  - saas
  - arquitetura
  - dotnet
  - react
  - multitenancy
  - keycloak
  - observabilidade
---

## Padrões modernos para SaaS multi-tenant: eventos, APIs e integrações (sem perder isolamento)
Quando seu SaaS vira multi-tenant de verdade, a complexidade não está só no banco: ela explode nas fronteiras do sistema — **eventos, APIs e integrações**. Alguns padrões ajudam a escalar sem comprometer **isolamento**, **segurança** e **observabilidade**.
---
### 1) Eventos sempre “tenant-aware”
Em arquitetura orientada a eventos, todo evento precisa carregar o contexto do tenant (ex.: `tenantId`, `environment`, `correlationId`).
- Evita consumidores “adivinharem” o tenant  
- Habilita métricas/alertas por tenant  
- Simplifica troubleshooting  
**Regra prática:** se um evento pode ser processado sem `tenantId`, você provavelmente está criando risco de vazamento de dados.


**Exemplo (C# 10): envelope mínimo**
```csharp
public record Event<T>(string TenantId, string Type, string CorrelationId, T Data);
public record UserCreated(string UserId);
var evt = new Event<UserCreated>(
    TenantId: tenantId,
    Type: "users.created",
    CorrelationId: corrId,
    Data: new UserCreated("u123")
);
```
---
### 2) Outbox Pattern + idempotência
Integrações falham. Reprocessamentos acontecem. Por isso:
- **Outbox Pattern:** grave “mudança de estado” + “evento a publicar” na mesma transação.  
- **Idempotência:** consumidores e endpoints devem aceitar reentrega sem duplicar efeitos.  
**Dica:** use uma chave idempotente por tenant (`tenantId + idempotencyKey`) para evitar colisão entre clientes.


**Exemplo (C# 10): Outbox em 3 linhas**
```csharp
db.Outbox.Add(new Outbox { TenantId = tenantId, Type = "users.created", Payload = json });
await db.SaveChangesAsync(); // mesmo commit da mudança do domínio
// um worker publica depois e marca PublishedAt
```


**Exemplo (C# 10): consumo idempotente por tenant**
```csharp
if (await db.Processed.AnyAsync(x => x.TenantId == e.TenantId && x.EventId == e.EventId)) return;
await HandleBusinessAsync(e);
db.Processed.Add(new() { TenantId = e.TenantId, EventId = e.EventId });
await db.SaveChangesAsync();
```
---
### 3) Webhooks por tenant com governança
Webhooks viram rapidamente uma plataforma dentro da plataforma. Padrões mínimos:
- Assinatura (HMAC) por tenant  
- Retry com backoff + DLQ (dead-letter queue)  
- Rate limit por tenant  
- Versão de payload (ex.: `eventVersion`) para evoluir sem quebrar clientes 


**Exemplo (C# 10): assinatura HMAC do payload**
```csharp
static string Sign(string secret, string payload)
{
    using var h = new System.Security.Cryptography.HMACSHA256(
        System.Text.Encoding.UTF8.GetBytes(secret));
    return Convert.ToHexString(h.ComputeHash(System.Text.Encoding.UTF8.GetBytes(payload)))
        .ToLowerInvariant();
}
// uso: request.Headers.Add("X-Signature", Sign(secret, payload));
```
---
### 4) API multi-tenant: o “contrato” do isolamento
Defina explicitamente como o tenant é resolvido:
- Via JWT claim (`tenant_id`)  
- Via subdomínio (`tenant.suaapp.com`)  
- Via header (mais simples, mas exige disciplina)  
O importante é ser **consistente**, e garantir que a autorização usa o tenant resolvido (não o que vem no body/query).


**Exemplo (C# 10): resolver tenant no início do request**
```csharp
app.Use(async (ctx, next) =>
{
    var tenantId = ctx.User.FindFirst("tenant_id")?.Value
        ?? ctx.Request.Headers["X-Tenant-Id"].ToString();
    if (string.IsNullOrWhiteSpace(tenantId)) { ctx.Response.StatusCode = 401; return; }
    ctx.Items["tenantId"] = tenantId;
    await next();
});
```
---
### 5) OAuth e credenciais “por cliente”
Em integrações (Google/Microsoft/CRM/Payments), é comum cada tenant ter suas próprias credenciais. Padrões recomendados:
- Guardar tokens com envelope encryption/KMS  
- Suportar rotação e revogação  
- Separar “conector” (tipo de integração) de “instalação” (config do tenant)  

Dica: Aqui o tenant pode estár em alguma propriedade do proprio token.
**Exemplo (C# 10): modelo mínimo para token por tenant**
```csharp
public record TenantOAuthToken(
    string TenantId,
    string Provider,
    string AccessToken,
    DateTimeOffset ExpiresAt
);
```
---
### 6) Observabilidade por tenant (não é opcional)
Sem isso, o suporte vira loteria.
- Logs com `tenantId`, `requestId`, `traceId`  
- Métricas por tenant (latência, erros, filas)  
- Tracing distribuído para seguir uma ação do usuário → API → evento → worker → webhook  


**Exemplo (C# 10): log com TenantId**
```csharp
logger.LogInformation("tenant={TenantId} request={RequestId}", tenantId, ctx.TraceIdentifier);
```
---
## Checklist rápido (para você validar sua arquitetura)
- [ ] Eventos carregam `tenantId` e `correlationId`
- [ ] Publicação confiável (Outbox) e consumo idempotente
- [ ] Webhooks assinados, com retry/backoff e DLQ
- [ ] Rate limit e quotas por tenant
- [ ] Tokens/segredos por tenant com KMS e rotação
- [ ] Logs/métricas/traces com dimensão por tenant
Se você quiser, eu transformo isso numa sequência de posts (1 por tópico) com um **anti-pattern** por post e exemplos de payload/headers.