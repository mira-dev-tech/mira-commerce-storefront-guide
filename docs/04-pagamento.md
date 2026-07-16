# 04 — Pagamento

Regra número zero: **seu código nunca vê número de cartão**. Todas as opções
abaixo mantêm o dado sensível dentro da plataforma ou do adquirente — é isso
que deixa sua loja no escopo PCI mais leve que existe (SAQ A) e te livra de
certificação própria.

## Opção A — página do gateway (`checkout_url`)

A mais simples: o `place-order` ([doc 03](03-checkout.md)) devolve
`checkout_url` quando o método de pagamento exige página hospedada. Você só
redireciona:

```ts
const placed = await placeOrder(sessionId);
if (placed.checkout_url) window.location.href = placed.checkout_url;
```

O comprador paga na página do gateway e volta. O status do pedido é liquidado
por webhook do adquirente — consulte o pedido para confirmar.

## Opção B — pagamento embutido (iframe na sua página)

Para quando você é dono do checkout mas quer a etapa de pagamento dentro da
sua própria página, sem redirect. A plataforma fornece uma view *chrome-less*
(sem header/footer) que você embute num `<iframe>`; o widget de cartão do
adquirente (com desafio 3DS **in-page**) renderiza lá dentro.

### O fluxo

```text
1. POST /checkout/sessions/{id}/payment-intent    → reserva o pedido + config do widget
2. widget do adquirente tokeniza o cartão no iframe (o dado nunca sai de lá)
3. POST /checkout/sessions/{id}/confirm-payment   → cobra o token
     → {"status":"paid"}            pronto
     → {"status":"requires_3ds"}    desafio 3DS renderiza no próprio iframe
4. GET  /checkout/sessions/{id}/payment-status    → poll até paid/failed
```

Todos autenticados com o `X-Checkout-Session-Token` da sessão — o pagamento
embutido opera **a mesma sessão** que seu checkout construiu.

### Integração no seu front

```html
<iframe
  src="https://SEU_DOMINIO_DE_CHECKOUT/compra/pagamento/embed"
  allow="payment"
  style="border:0;width:100%"
></iframe>
```

O iframe conversa com sua página por `postMessage`:

```ts
window.addEventListener("message", (event) => {
  // valide sempre a origem antes de reagir
  if (event.origin !== EXPECTED_ORIGIN) return;

  const msg = event.data;
  if (msg?.type === "mira:payment") {
    // status: ready | processing | 3ds_challenge | success | failed | pending
    if (msg.status === "success") showConfirmation(msg.order_id);
    if (msg.status === "failed") showGenericPaymentError();
  }
  if (msg?.type === "mira:resize") {
    iframe.style.height = `${msg.height}px`; // o embed pede o tamanho certo
  }
});
```

Dois cuidados que a plataforma exige de você:

1. **Valide `event.origin`** em todo `message` — nunca reaja a mensagem de
   origem desconhecida.
2. O domínio do seu storefront precisa estar **autorizado a enquadrar o
   embed** (CSP `frame-ancestors`) — isso é configurado pela equipe Mirá no
   onboarding do seu domínio. Iframe carregando em branco costuma ser isso.

### Respostas do `payment-intent` que você deve tratar

| Resposta | Significado | Ação |
|----------|-------------|------|
| `200` com `session` | Widget in-page disponível | Renderizar o widget |
| `200` com `session: null` | Adquirente da loja é só-redirect | Ofereça a Opção A (ou offline, se ativo) |
| `409 already_paid` | Pedido da sessão já pago | Ir direto à confirmação |

## Opção C — método offline ("pagar na entrega")

Para **testar o funil inteiro sem cartão** — e como opção real para lojas que
liquidam na entrega. Quando a loja tem o método `cash` ativo, o
`payment-intent` devolve `offline_enabled: true`; ao escolher, chame
`confirm-payment` com `{"method": "cash"}` (sem token). O pedido nasce com
pagamento pendente, a liquidar na entrega. É o jeito recomendado de validar
sua integração de ponta a ponta no sandbox.

## Confirmação: nunca confie só no front

Sucesso no `postMessage`/redirect é sinal de UX, não verdade contábil — a
liquidação real chega por webhook do adquirente à plataforma, de forma
assíncrona. Antes de mostrar "pedido confirmado" com efeitos colaterais
(e-mail, estoque próprio, ERP seu), confirme o status pelo
`payment-status`/consulta do pedido.
