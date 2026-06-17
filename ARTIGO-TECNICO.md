# Multitenancy de verdade num SaaS de PDV: as decisões que você não pode errar

> [USO INTERNO — não publique este bloco]
> Canais: dev.to, Medium, Hashnode, GitHub. NÃO publicar no LinkedIn.
> Título (campo "Title" do dev.to): use o H1 acima.
> Tags (até 4): java, saas, architecture, backend
> Antes de publicar: troque o marcador [COLAR-LINK-KIWIFY] no final pelo link
> do checkout do ComercialCloud (criado na Kiwify). Copie o corpo a partir do "---".

---

Quem decide construir um SaaS para o varejo costuma subestimar a mesma coisa: não é a tela do PDV que dá trabalho. É tudo o que sustenta vários comércios rodando no mesmo sistema sem um ver os dados do outro, sem uma cobrança vazar, sem uma nota fiscal travar a venda.

Essas decisões de fundação, se erradas no início, custam uma reescrita lá na frente. Vamos às que mais importam.

## 1. Isolamento de tenant: a decisão que atravessa tudo

Há três caminhos para multitenancy: um banco por cliente, um schema por cliente, ou uma coluna `tenant_id` compartilhada. Para um SaaS de PMEs com muitos clientes pequenos, **coluna compartilhada** é o equilíbrio certo entre custo e simplicidade — mas só funciona se o isolamento for **inescapável**.

O erro clássico é confiar que cada query vai lembrar de filtrar por tenant. Uma esquecida e um comércio vê a venda do outro. A defesa é tornar o tenant parte do **contexto da requisição**, resolvido num filtro HTTP antes de qualquer lógica:

```java
// Filtro resolve o tenant uma vez; o resto do código nunca "esquece"
String tenantId = header("X-Tenant-Id"); // ou claim tenant_id do JWT em produção
TenantContext.set(tenantId);
```

A partir daí, todo acesso a dados parte do contexto — não de um parâmetro que alguém pode omitir.

## 2. Fiscal não pode derrubar a venda

Emissão de NFC-e/NF-e depende da SEFAZ, e a SEFAZ cai. Se a emissão for **síncrona** dentro da venda, o dia em que a SEFAZ ficar lenta, seu PDV trava e o caixa para.

A venda e a emissão fiscal são responsabilidades diferentes. Use o **padrão outbox**: registre a venda e a intenção de emitir na mesma transação; um processo separado emite contra o provedor fiscal com retry.

```sql
BEGIN;
  INSERT INTO venda (...) VALUES (...);
  INSERT INTO outbox (tipo, payload, status) VALUES ('EmitirNFCe', ?, 'PENDENTE');
COMMIT;
```

A venda fecha na hora; a nota é emitida de forma resiliente. SEFAZ instável vira "nota pendente", não "loja parada".

## 3. Cobrar a mensalidade — e cortar o inadimplente — sem trabalho manual

Um SaaS vive da recorrência. Mas controlar quem pagou, quem atrasou e quem deve perder acesso vira um pesadelo manual rápido. A resposta é ligar o estado da assinatura ao próprio acesso da API: o tenant inadimplente recebe **HTTP 402 (Payment Required)** até regularizar.

```java
if (assinatura.inadimplente()) {
    throw new PaymentRequiredException(); // 402: bloqueia o tenant, reativa sozinho ao pagar
}
```

Suspensão e reativação automáticas, sem ninguém ligando ou desligando cliente na mão.

## 4. Onboarding self-service: o cliente entra sozinho

Se cada novo comércio exige você criar tenant, loja e admin manualmente, o SaaS não escala. O cadastro precisa ser **uma chamada** que provisiona tudo — tenant, loja, usuário admin, assinatura (com trial) e até dados de demonstração para o cliente ver valor no primeiro minuto.

```
POST /api/v1/public/onboarding
→ cria tenant + loja + admin + assinatura (trial) + dados demo
```

Onboarding automático é o que separa um sistema que você "instala para cada cliente" de um SaaS que cresce sozinho.

## O custo real é a soma

Nenhuma dessas decisões é exótica. O problema é que são **meses** somando isolamento multitenant, autenticação com RBAC, fiscal resiliente, billing recorrente, onboarding, observabilidade — antes de vender a primeira licença. E qualquer uma mal resolvida no começo vira dívida cara.

---

### Atalho

Eu construí exatamente essa fundação como um boilerplate completo: **PDV + retaguarda multitenant** em Java 21/Quarkus e Next.js, com fiscal (outbox), cobrança recorrente com suspensão automática, onboarding self-service e observabilidade — tudo documentado e rodando via Docker Compose.

É o **ComercialCloud**. Se você vai entrar no mercado de PDV/varejo, ele te poupa meses e te entrega uma base que já nasceu pensando nos erros caros.

👉 **[Conheça o ComercialCloud](https://pay.kiwify.com.br/XOuVLZc)**

*Quer trocar ideia sobre alguma dessas decisões de arquitetura? Comenta aí.*
