# 05 — Storefront estático com Next.js (o padrão de produção)

O padrão que roda em produção nas lojas da plataforma: um Next.js com
**`output: "export"`** — a vitrine inteira vira HTML/JS estático no build,
hospedável em qualquer CDN (Cloudflare Pages, S3, Netlify…). Zero servidor
para operar, performance máxima, e o token de integrador morre no build.

```text
build (CI) ──Bearer──▶ Core API ──▶ páginas estáticas ──▶ CDN ──▶ comprador
                                                             │
                                     checkout: hospedado, embed ou seu BFF
```

## As três variáveis de ambiente

```bash
# públicas — podem ir para o browser
NEXT_PUBLIC_COMMERCE_API_URL=https://api.mira-dev.tech
NEXT_PUBLIC_MEMBER_ID=SEU_MEMBER_ID          # tenant; identifica, não autentica

# secreta — SÓ no ambiente de build (CI). NUNCA prefixe com NEXT_PUBLIC_
COMMERCE_STOREFRONT_BEARER=mc_live_…
```

O erro que reprova o go-live: `NEXT_PUBLIC_` no bearer. O prefixo embute o
valor no bundle do browser — qualquer visitante lê seu token no DevTools.

## Gerando o catálogo no build

Toda rota dinâmica (`[slug]`) num static export precisa de
`generateStaticParams` — é ela que consulta a API **no build** e materializa
uma página por produto:

```tsx
// app/produto/[slug]/page.tsx
export async function generateStaticParams() {
  const res = await fetch(
    `${process.env.NEXT_PUBLIC_COMMERCE_API_URL}/v1/products?limit=500`,
    {
      headers: {
        Authorization: `Bearer ${process.env.COMMERCE_STOREFRONT_BEARER}`,
        "X-Tenant-ID": process.env.NEXT_PUBLIC_MEMBER_ID!,
      },
    },
  );
  const { data } = await res.json();
  return data.map((p: { sku: string }) => ({ slug: p.sku }));
}

export default async function ProductPage({ params }: { params: { slug: string } }) {
  // mesmo fetch com Bearer — roda no build, não no browser
  const product = await getProductBySku(params.slug);
  return <ProductView product={product} />;
}
```

## As restrições do static export (aprenda antes de sofrer)

| Não funciona | Porquê | Use no lugar |
|--------------|--------|--------------|
| `cookies()` / `headers()` em Server Components | Não há request no build | Estado no client, ou BFF |
| `searchParams` sem params estáticos | Página não existe até o build | `generateStaticParams` / filtros client-side |
| Rota de API (`app/api/*`) no export puro | Não há servidor | BFF hospedado à parte (ver abaixo) |
| Dado "ao vivo" no HTML | O HTML nasce no build | Fetch client-side de dados públicos, ou rebuild |

## Dados vivos numa página estática

O HTML é do build, mas o browser pode buscar dados frescos **que não exigem o
bearer** — por exemplo, preço/estoque via um endpoint público do seu BFF, ou o
fluxo de checkout por sessão (o `cks_tok_` é do comprador). O padrão:

- **Vitrine (produto, descrição, mídia)**: baked no build.
- **Preço/estoque em tempo real** (opcional): fetch client-side ao seu BFF.
- **Checkout**: nunca estático — hospedado ([README](../README.md), caminho 1),
  embed ([doc 04](04-pagamento.md)) ou seu funil com BFF ([doc 03](03-checkout.md)).

## Quando você precisa de um BFF (e onde ele roda)

Se escolheu checkout headless ou quer preço em tempo real, algum código seu
precisa rodar em servidor (para guardar o bearer e o cookie httpOnly do
comprador). Opções que convivem bem com a vitrine estática:

- **Cloudflare Pages Functions / Workers** no mesmo domínio (`/api/*`).
- Um serviço pequeno separado (`bff.sualoja.com.br`) — qualquer runtime.

O BFF expõe só o que o funil precisa (criar sessão, patch, place-order,
payment-intent/confirm/status) e nada mais — ele é um proxy com opinião, não
um espelho da API.

## Rebuild: mantendo a vitrine atualizada

Duas estratégias, combináveis:

1. **Agendado**: build a cada N horas no seu CI.
2. **Por evento**: a plataforma pode notificar seu CI (webhook) quando o
   catálogo muda → dispare o build.

## Checklist de go-live

1. [ ] `mc_test_` → funil completo testado no sandbox (inclusive método offline)
2. [ ] Bearer só no CI; nenhuma env secreta com `NEXT_PUBLIC_`
3. [ ] `generateStaticParams` cobre todas as rotas dinâmicas
4. [ ] Tratamento de 402/403/409/429 com mensagens de UX ([doc 03](03-checkout.md))
5. [ ] `Idempotency-Key` em place-order/confirm
6. [ ] Domínio autorizado para o embed de pagamento (se usar o caminho 3)
7. [ ] Estratégia de rebuild definida (cron e/ou webhook)
8. [ ] Confirmação de pedido lê o status real, não o `postMessage`
