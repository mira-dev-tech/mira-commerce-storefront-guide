# 01 — Autenticação e tenancy

A API é **multi-tenant**: um mesmo endpoint serve várias lojas, e a sua é
identificada pelo header `X-Tenant-ID` com o seu `member_id`. Autenticação e
identificação são coisas separadas — este doc explica as duas.

## Quem é quem — os 3 tipos de credencial

| Credencial | Formato | Quem usa | Onde pode viver |
|------------|---------|----------|-----------------|
| **Token de integrador** | `mc_test_…` / `mc_live_…` | O *seu backend/build* — catálogo, preços, pedidos | Server-side ou build. **Nunca no browser** |
| **Token de sessão de checkout** | `cks_tok_…` | O *comprador* — durante um checkout headless | Browser (cookie httpOnly via BFF, de preferência) |
| **Sessão de cliente** | `mc_sess_…` | O *comprador autenticado* — área pós-compra ("meus pedidos") | Browser, obtido via `/auth/customer/*` |

A separação existe de propósito: o token de integrador tem escopos amplos
(catálogo, pedidos); o de sessão de checkout só movimenta **aquela** sessão,
com TTL de 30 minutos. Vazou um `cks_tok_`? O estrago é uma sessão. Vazou um
`mc_live_`? É incidente — por isso ele nunca toca o browser.

## Anatomia de uma chamada

```bash
curl -s "https://api.mira-dev.tech/v1/products?limit=10" \
  -H "Authorization: Bearer mc_test_SEU_TOKEN" \
  -H "X-Tenant-ID: SEU_MEMBER_ID"
```

- `Authorization: Bearer …` — autentica (quem você é).
- `X-Tenant-ID: <member_id>` — identifica a loja (qual tenant). É um UUID
  público; sem ele a API compartilhada não sabe qual loja atender.
- Chamadas de sessão de checkout usam `X-Checkout-Session-Token: cks_tok_…`
  no lugar do Bearer.

## Sandbox vs produção

| | Sandbox | Produção |
|---|---------|----------|
| Token | `mc_test_…` | `mc_live_…` |
| Para quê | Desenvolver e testar tudo, inclusive checkout com método offline | Loja no ar |

Desenvolva **inteiro** com `mc_test_` antes de pedir o token live. Os
endpoints são os mesmos; o que muda são os escopos e as regras antifraude.

## A regra do BFF (a mais importante deste guia)

O browser **nunca** vê um token de integrador. Os dois padrões corretos:

1. **Storefront estático** (recomendado para a vitrine): o token é usado só em
   **build time** para gerar as páginas do catálogo. O HTML final não contém o
   token. → ver [doc 05](05-storefront-estatico-nextjs.md).
2. **BFF** (necessário para checkout headless): o browser fala com o *seu*
   backend (`/api/...`), e ele repassa à Core API com o Bearer server-side.
   O `cks_tok_` do comprador fica num cookie httpOnly gerido pelo BFF.

```text
browser ──(sem tokens de integrador)──▶ seu BFF ──(Bearer mc_…)──▶ Core API
```

Anti-padrão que reprova a integração: `NEXT_PUBLIC_API_TOKEN=mc_live_…`.

## Rate limits

Comprador (sessões de checkout) e integrador (Bearer) têm limites
independentes. Ao receber `429`, respeite o header `Retry-After` com backoff —
insistir imediatamente só estende o bloqueio.
