# 07 — SDKs TypeScript

Tudo que este guia mostra em `curl` existe também como **SDK TypeScript
tipado**, gerado da mesma especificação OpenAPI que define a API — ou seja, o
SDK nunca diverge do contrato: cada release da API regenera o cliente.

| Pacote | O que é |
|--------|---------|
| `@mira/commerce-client-sdk` | Cliente base: uma função tipada por operação da API (`listProducts`, `resolvePrice`, `createCheckoutSession`…) + `createCommerceClient` para configurar auth/tenant uma vez |
| `@mira/commerce-checkout-sdk` | Helpers finos do funil de checkout por cima do cliente base (`createCheckoutSession` / `patchCheckoutSession` / `placeCheckoutOrder`) |

> **Disponibilidade:** os pacotes ainda **não estão no npm público** — são
> fornecidos no onboarding (a publicação npm está no roadmap). Por isso este
> guia é REST-first: o `curl` é o contrato de referência, e o SDK é a mesma
> coisa com tipos. Tudo que você aprendeu nos docs 01–06 se aplica igual.

## Configurar o cliente (uma vez)

```ts
import { createCommerceClient } from "@mira/commerce-client-sdk";

const client = createCommerceClient({
  baseUrl: `${process.env.COMMERCE_API_URL}/v1`, // https://api.mira-dev.tech/v1
  token: process.env.COMMERCE_STOREFRONT_BEARER, // Bearer — SÓ server-side/build
  memberId: process.env.NEXT_PUBLIC_MEMBER_ID,   // vira o header X-Tenant-ID
});
```

O cliente resolve por você as duas coisas que mais geram erro na integração
manual: o `Authorization: Bearer` e o `X-Tenant-ID` entram automaticamente em
toda chamada. As regras de segurança são as mesmas do
[doc 01](01-autenticacao.md) — um cliente com `token` só pode existir
server-side ou no build.

## Vitrine — catálogo tipado

```ts
import { listProducts, resolvePrice } from "@mira/commerce-client-sdk";

const { data: products } = await listProducts({
  client,
  query: { limit: 50 },
});

const { data: price } = await resolvePrice({
  client,
  query: { sku: "SKU-001", channel: "web" },
});
```

Cada operação da API tem uma função com o mesmo `operationId` do OpenAPI —
o nome que você vê nos docs é o nome que você importa. Autocomplete do editor
substitui a leitura do spec.

## Checkout — os helpers do funil

```ts
import { createCommerceClient } from "@mira/commerce-client-sdk";
import {
  createCheckoutSession,
  patchCheckoutSession,
  placeCheckoutOrder,
} from "@mira/commerce-checkout-sdk";

// no seu BFF — o comprador nunca vê estes tokens
const session = await createCheckoutSession(client, {
  memberId: MEMBER_ID,
  warehouseId: WAREHOUSE_ID,
  channel: "web",
});
// guarde session.sessionToken em cookie httpOnly

await patchCheckoutSession(client, session.id, sessionToken, {
  cart: [{ sku: "SKU-001", qty: 2, selected: true }],
  customer_email: "maria@example.com",
});

const placed = await placeCheckoutOrder(client, {
  sessionId: session.id,
  sessionToken,
  idempotencyKey: crypto.randomUUID(),
});
if (placed.checkout_url) redirect(placed.checkout_url);
```

Repare que os helpers **obrigam** o que o [doc 03](03-checkout.md) recomenda:
`placeCheckoutOrder` não aceita ser chamado sem `idempotencyKey` — a regra de
segurança virou assinatura de função.

## Onde o SDK roda (e onde não roda)

| Lugar | Pode? | Configuração |
|-------|-------|--------------|
| Build do storefront estático (`generateStaticParams`) | ✅ | `token` do bearer de build |
| Seu BFF (Route Handler, Worker, serviço próprio) | ✅ | `token` do integrador; sessão do comprador em cookie |
| Browser | ⚠️ Só **sem** `token` | Chamadas públicas de sessão (`X-Checkout-Session-Token`) via seu BFF; nunca instancie com Bearer no client bundle |

## SDK ou REST puro?

Os dois são o mesmo contrato. Guia prático:

- **Stack TypeScript/Next.js** → use o SDK; tipos pegam quebra de contrato no
  build, e os helpers de checkout codificam as boas práticas.
- **Outra linguagem / integração server-to-server** → REST puro pelos docs
  01–06; a especificação OpenAPI está disponível no onboarding para gerar
  cliente na sua linguagem.
