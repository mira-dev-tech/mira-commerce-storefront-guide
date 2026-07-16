# 06 — Minha conta (área do comprador)

A área logada do comprador — "meus pedidos", perfil — também é **headless**:
você constrói as telas; a plataforma cuida da identidade, da sessão e dos
dados. O login é **sem senha** (magic-link por e-mail com código de 6
dígitos): menos atrito, nada de senha para vazar, e o e-mail verificado é a
âncora da identidade — o mesmo e-mail usado nos pedidos.

## O fluxo de login (passwordless)

```text
1. POST /auth/customer/magic-link   {email}          → envia o código por e-mail
2. POST /auth/customer/verify       {email, code}    → devolve a sessão (mc_sess_…)
3. chamadas autenticadas com a sessão do comprador
```

### 1. Pedir o código

```bash
curl -s -X POST "$API/auth/customer/magic-link" \
  -H "X-Tenant-ID: $MEMBER" -H "Content-Type: application/json" \
  -d '{"email": "maria@example.com"}'
```

Endpoint público (sem Bearer) — a resposta é neutra mesmo para e-mail
desconhecido, para não revelar quem é cliente. O comprador recebe um código
numérico de 6 dígitos por e-mail.

### 2. Verificar o código → sessão

```bash
curl -s -X POST "$API/auth/customer/verify" \
  -H "X-Tenant-ID: $MEMBER" -H "Content-Type: application/json" \
  -d '{"email": "maria@example.com", "code": "123456"}'
```

A resposta traz o token de sessão do comprador (`mc_sess_…`). Trate-o como o
`cks_tok_` do checkout: **cookie httpOnly via seu BFF** — nunca em
`localStorage` (XSS lê `localStorage`; não lê cookie httpOnly).

### Cadastro explícito (opcional)

O comprador nasce automaticamente no primeiro pedido, mas você pode oferecer
cadastro antes da compra:

```bash
curl -s -X POST "$API/auth/customer/signup" \
  -H "X-Tenant-ID: $MEMBER" -H "Content-Type: application/json" \
  -d '{
    "email": "maria@example.com",
    "document_type": "cpf",
    "document_number": "00000000000",
    "metadata": { "origem": "newsletter" }
  }'
```

`metadata` é um objeto livre — use para os campos específicos da sua loja sem
esperar mudança de API.

## Com a sessão em mãos

Todas com `Authorization: Bearer mc_sess_…` (a sessão do comprador, não o seu
token de integrador):

| Chamada | Devolve |
|---------|---------|
| `GET /auth/customer/me` | Perfil do comprador logado |
| `GET /auth/customer/orders` | Os pedidos **dele** (a plataforma filtra; você não constrói esse filtro) |
| `GET /auth/customer/orders/{order_id}` | Detalhe de um pedido — só se for dele |

```bash
curl -s "$API/auth/customer/orders" \
  -H "Authorization: Bearer mc_sess_TOKEN_DA_SESSAO" \
  -H "X-Tenant-ID: $MEMBER"
```

Repare no isolamento: a listagem de pedidos da conta usa a **sessão do
comprador**, nunca o token de integrador — é a API que garante que um
comprador jamais vê pedido de outro. Não implemente "meus pedidos" filtrando
uma listagem geral no seu BFF: além de desnecessário, é um vazamento à espera
de acontecer.

## UX que recomendamos

- **Login sob demanda**: não exija conta para comprar — o checkout já verifica
  a identidade ([doc 03](03-checkout.md)). Ofereça "acompanhe seu pedido" com
  o mesmo e-mail depois da compra; a conversão agradece.
- **Sessão expirada** (`401`): volte à tela de e-mail com mensagem suave
  ("confirme seu e-mail de novo"), preservando a página de destino.
- **Código errado**: mensagem genérica e opção de reenviar — sem contador
  público de tentativas (o rate-limit da plataforma cuida do abuso).

## Checklist da área do cliente

1. [ ] `mc_sess_` em cookie httpOnly (BFF), nunca `localStorage`
2. [ ] "Meus pedidos" via `/auth/customer/orders` — jamais filtro próprio
3. [ ] Fluxo de reenvio de código + tratamento de `401`/`429`
4. [ ] Logout limpa o cookie no BFF
5. [ ] Testado no sandbox com `mc_test_` + e-mail real de teste
