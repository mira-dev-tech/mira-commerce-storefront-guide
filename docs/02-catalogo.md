# 02 — Catálogo, preço e estoque

A matéria-prima da vitrine. Três verdades para gravar:

1. **Produto ≠ preço.** O catálogo devolve o produto (nome, descrição, mídia,
   SKUs); o preço vem de `/prices/resolve`, que considera canal, depósito e
   promoções — e pode mudar sem o produto mudar.
2. **O preço mostrado é informativo; o preço cobrado é o do core.** No
   checkout, a plataforma recalcula tudo server-side. Vitrine desatualizada
   nunca vira prejuízo — vira no máximo uma surpresa de UX (evite com rebuild).
3. **Estoque é uma foto.** Reserve exibi-lo como orientação ("últimas
   unidades"); a reserva real acontece no pedido.

## Listar produtos

```bash
curl -s "$API/products?limit=20&offset=0" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-ID: $MEMBER"
```

Resposta (campos principais):

```json
{
  "data": [
    {
      "id": "uuid",
      "sku": "SKU-001",
      "name": "Produto Exemplo",
      "description": "…",
      "category": "…",
      "ean": "7891234567890",
      "media": [
        { "type": "image", "url": "…", "alt_text": "…", "position": 0 }
      ]
    }
  ]
}
```

Detalhe de um produto: `GET /products/{product_id}` · variações:
`GET /products/{product_id}/skus` · busca por código de barras:
`GET /catalog/ean/{ean}`.

## Resolver preço e estoque

```bash
curl -s "$API/prices/resolve?sku=SKU-001&channel=web" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-ID: $MEMBER"
```

Parâmetros úteis: `warehouse_id` (depósito de origem — afeta preço por praça e
estoque) e `channel` (`web`, `whatsapp`, …; tabelas de preço podem variar por
canal). A resposta traz o preço unitário efetivo (promoções já aplicadas) e a
disponibilidade.

`404 price not found for sku` significa que o SKU existe no catálogo mas não
tem preço cadastrado para aquele canal/depósito — trate como "indisponível".

## Cotação de frete

Na página de produto ou carrinho ("calcule o frete"):

```bash
curl -s -X POST "$API/shipping/quotes" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-ID: $MEMBER" \
  -H "Content-Type: application/json" \
  -d '{
    "destination_postal_code": "01310-100",
    "items": [ { "sku": "SKU-001", "quantity": 2 } ]
  }'
```

A resposta lista opções (`method_id`, transportadora, preço, ETA em dias).
Guarde o `method_id` escolhido — ele entra no pedido e é **revalidado pelo
core** no fechamento (preço de frete adulterado no front não passa).

## Exemplo TypeScript (server-side ou build)

```ts
const API = process.env.COMMERCE_API_URL!; // https://api.mira-dev.tech/v1
const headers = {
  Authorization: `Bearer ${process.env.COMMERCE_STOREFRONT_BEARER}`, // nunca NEXT_PUBLIC_
  "X-Tenant-ID": process.env.NEXT_PUBLIC_MEMBER_ID!,                 // público
};

export async function listProducts(limit = 20) {
  const res = await fetch(`${API}/products?limit=${limit}`, { headers });
  if (!res.ok) throw new Error(`products: HTTP ${res.status}`);
  return (await res.json()).data;
}
```

## Mantendo a vitrine fresca (storefront estático)

Como o preço mora na API e a vitrine é estática, o padrão é **rebuild
periódico ou por evento** (webhook de atualização de catálogo → dispara o
build no seu CI). Entre builds, o preço exibido pode divergir do cobrado — o
checkout sempre corrige a favor do preço real. Detalhes no
[doc 05](05-storefront-estatico-nextjs.md).
