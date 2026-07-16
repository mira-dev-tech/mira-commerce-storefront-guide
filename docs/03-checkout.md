# 03 — Checkout headless (checkout sessions)

O caminho 2 do guia: você constrói o funil inteiro e a plataforma valida,
cria o pedido e cobra. O mecanismo central é a **sessão de checkout** — um
estado server-side com TTL de 30 minutos que acumula carrinho, identidade e
entrega até o fechamento.

Por que sessão em vez de montar o pedido direto? Porque o funil real é
incremental (o comprador vai e volta), multi-dispositivo, e precisa de
validação progressiva. A sessão dá isso de graça: cada `PATCH` é um
salvamento, e o `place-order` valida tudo de uma vez.

## O fluxo em 4 passos

```text
1. POST  /checkout/sessions                       → cria sessão + session_token
2. PATCH /checkout/sessions/{id}   (n vezes)      → carrinho, identidade, entrega
3.       identidade do comprador   (send-code/verify)
4. POST  /checkout/sessions/{id}/place-order      → valida tudo, cria e submete o pedido
```

### 1. Criar a sessão

```bash
curl -s -X POST "$API/checkout/sessions" \
  -H "X-Tenant-ID: $MEMBER" -H "Content-Type: application/json" \
  -d '{
    "member_id": "'$MEMBER'",
    "warehouse_id": "UUID_DO_DEPOSITO",
    "channel": "web",
    "state": { "cart": [], "identity_verified": false }
  }'
```

Resposta `201` traz `id`, `expires_at` e **`session_token` (`cks_tok_…`) —
mostrado uma única vez**. Guarde-o num cookie httpOnly via seu BFF; ele é a
credencial do comprador para os próximos passos.

### 2. Montar o carrinho (PATCH incremental)

Antes de adicionar um item, resolva preço/estoque
([doc 02](02-catalogo.md)). Depois:

```bash
curl -s -X PATCH "$API/checkout/sessions/$SESSION_ID" \
  -H "X-Checkout-Session-Token: $CKS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "state": {
      "cart": [
        { "sku": "SKU-001", "qty": 2, "selected": true,
          "name": "Produto Exemplo", "unit_price": 19990 }
      ],
      "customer_name": "Maria Silva",
      "customer_email": "maria@example.com",
      "payment_method_id": "pix",
      "delivery_mode": "shipping"
    }
  }'
```

O `PATCH` é **merge parcial** — mande só o que mudou. O estado aceita campos
seus além dos reconhecidos (ele é o "rascunho" do funil), mas quem manda no
valor final é sempre o recálculo do core.

### 3. Verificar a identidade do comprador

O `place-order` exige `identity_verified: true`. O fluxo padrão é OTP:

```text
POST /checkout/sessions/{id}/identity/lookup      → identifica por e-mail/telefone
POST /checkout/sessions/{id}/identity/send-code   → envia o código
POST /checkout/sessions/{id}/identity/verify      → confirma; o core marca a sessão
```

Lojas com verificação reforçada ativa (documento + selfie) têm passos extras
de upload — a config vem de
`GET /checkout/sessions/{id}/identity/verification-config`, então construa o
funil lendo essa config em vez de assumir.

### 4. Fechar o pedido

```bash
curl -s -X POST "$API/checkout/sessions/$SESSION_ID/place-order" \
  -H "X-Checkout-Session-Token: $CKS_TOKEN" \
  -H "Idempotency-Key: $(uuidgen)"
```

Resposta `201`:

```json
{
  "order": { "id": "…", "status": "…" },
  "payment_status": "pending",
  "checkout_url": "https://…"   // quando o método exige página do gateway
}
```

- Veio `checkout_url`? Redirecione o comprador (pagamento hospedado).
- Quer pagamento sem sair da sua página? → [doc 04](04-pagamento.md), iframe.
- **`Idempotency-Key` é obrigatória na sua cabeça**: retry da mesma chave
  nunca duplica pedido. Gere um UUID por tentativa de fechamento.

## O que o place-order valida (e o erro que você recebe)

| Validação | Se falhar |
|-----------|-----------|
| Pelo menos 1 item `selected: true` com `qty > 0` | `409` |
| `identity_verified: true` | `409` |
| Sessão ainda não usada (`placed_order_id` vazio) | `409` — sessão é single-use |
| Depósito de origem definido | `409` |
| Regras de negócio da loja (hooks/antifraude) | `403` com `message` para exibir |
| Pagamento recusado | `402` |

## Tabela de tratamento de erros no front

| HTTP | Significado | O que o comprador vê |
|------|-------------|----------------------|
| `401` | Sessão expirou / token inválido | "Sua sessão expirou" → recomeçar checkout |
| `402` | Pagamento recusado | Mensagem **genérica** — nunca o motivo técnico |
| `403` | Regra da loja bloqueou | O `message` retornado pelo core |
| `409` | Passo faltando ou sessão já usada | Voltar ao passo indicado / nova sessão |
| `429` | Muitas tentativas | "Aguarde um instante" + backoff (`Retry-After`) |

## Alternativa: fluxo granular (sem sessão)

Para integrações server-to-server (ERP, bot, automação) onde não há funil:

```text
POST /orders                            → cria o pedido completo de uma vez
POST /orders/{id}/commands/submit       → submete
POST /orders/{id}/payments/initiate     → inicia o pagamento
```

Mesmas regras de `Idempotency-Key` e tratamento de erro. Para storefront com
comprador humano, prefira sessões — o funil incremental é o caso de uso delas.
