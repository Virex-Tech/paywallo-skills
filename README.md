# Paywallo Skills

Conjunto de **skills (playbooks/guias)** para o Claude Code integrar o SDK [`@virex-tech/paywallo-sdk`](https://www.npmjs.com/package/@virex-tech/paywallo-sdk) em apps React Native + Expo.

Cada arquivo em [`skill/`](./skill) é um guia focado num pedaço da integração. Você (ou o Claude) lê a skill relevante e segue passo a passo — sem precisar caçar doc espalhada.

---

## O que tem aqui

| Skill                                                                   | Quando usar                                                                                              |
| :---------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| [`paywallo-full-skill.md`](./skill/paywallo-full-skill.md)              | **Master guide.** Visão arquitetural + checklist ponta-a-ponta. Use como índice e ponto de partida.      |
| [`paywallo-sdk-setup.md`](./skill/paywallo-sdk-setup.md)                | Setup mínimo funcional: instalação, `PaywalloProvider`, `identify` no login, `reset` no logout, errors.  |
| [`paywallo-paywall-skill.md`](./skill/paywallo-paywall-skill.md)        | Apresentar paywalls e gatear conteúdo premium (`requireSubscriptionWithCampaign`, `presentCampaign`...). |
| [`paywall-ab-testing.md`](./skill/paywall-ab-testing.md)                | Feature flags e A/B testing sem flicker (`sessionFlags`, `getVariantCached`, `getVariant`).              |
| [`paywallo-funnel-tracking.md`](./skill/paywallo-funnel-tracking.md)    | Instrumentação do funil: onboarding, eventos custom, drop-off automático.                                |

> Todos os arquivos pressupõem `paywallo-sdk-setup.md` aplicado primeiro. O `full-skill` é o índice — comece por ele se for a primeira integração.

---

## Como usar no seu projeto

Existem três formas, da mais simples pra mais integrada.

### Opção 1 — Apontar o Claude pra uma skill quando precisar (mais simples)

Copie a pasta `skill/` pra dentro do seu repo (ou clone esse repo como submódulo) e, quando for mexer no Paywallo, fale pro Claude:

```
Lê skill/paywallo-sdk-setup.md e aplica o setup no app.
```

Funciona em qualquer cliente (Claude Code, Cursor, Windsurf, web). Zero configuração.

### Opção 2 — Carregar no `CLAUDE.md` do projeto (sempre em contexto)

Se o app vai mexer no Paywallo com frequência, referencie as skills no `CLAUDE.md` da raiz do seu projeto pra elas serem carregadas automaticamente toda sessão:

```markdown
<!-- CLAUDE.md do seu app -->

## Integração Paywallo

Este app usa `@virex-tech/paywallo-sdk`. Antes de mexer em qualquer coisa relacionada a
paywall, subscription, A/B test ou funnel tracking, leia a skill correspondente em
`skill/`:

- Setup / Provider / identify → `skill/paywallo-sdk-setup.md`
- Apresentar paywall / gate premium → `skill/paywallo-paywall-skill.md`
- A/B testing / feature flags → `skill/paywall-ab-testing.md`
- Funnel tracking / onboarding → `skill/paywallo-funnel-tracking.md`
- Guia completo (índice) → `skill/paywallo-full-skill.md`
```

O Claude Code lê o `CLAUDE.md` automaticamente em toda sessão dentro do diretório.

### Opção 3 — Como **Claude Code Skills** nativas (auto-trigger)

Se você usa Claude Code e quer que as skills sejam **invocadas automaticamente** quando o assunto for Paywallo, converta cada arquivo no formato oficial de skill (`.claude/skills/<nome>/SKILL.md` com frontmatter):

```
seu-app/
└── .claude/
    └── skills/
        ├── paywallo-setup/
        │   └── SKILL.md     ← conteúdo de paywallo-sdk-setup.md + frontmatter
        ├── paywallo-paywall/
        │   └── SKILL.md
        ├── paywallo-ab-testing/
        │   └── SKILL.md
        └── paywallo-funnel/
            └── SKILL.md
```

Adicione no topo de cada `SKILL.md`:

```markdown
---
name: paywallo-setup
description: Use esta skill quando o usuário pedir para instalar, configurar ou plugar o SDK @virex-tech/paywallo-sdk em um app React Native/Expo. Cobre PaywalloProvider, identify, reset, error wiring.
---

# (conteúdo original aqui)
```

O Claude Code lê o `description` e dispara a skill sozinho quando o pedido casar.

> Mais sobre o formato em https://docs.claude.com/en/docs/claude-code/skills

---

## Stack pressuposta nas skills

As skills foram escritas pro `base-app` da Virex Tech:

- React Native + Expo SDK 54
- TypeScript strict
- Expo Router
- Zustand + React Query
- i18next + theme tokens

A maior parte se aplica a qualquer RN/Expo — os trechos específicos do `base-app` (ex: `AppProviders`, `useOnboarding`) servem como template; adapte ao seu app.

---

## Versão alvo do SDK

- `@virex-tech/paywallo-sdk` **v2.1.x**

Se você está em v1.x ou v2.0.x, algumas APIs (`useOnboarding`, `requireSubscriptionWithCampaign`, `sessionFlags`) podem não existir ou ter assinatura diferente. Veja o `CHANGELOG` do pacote antes.

---

## Links úteis

- Pacote: https://www.npmjs.com/package/@virex-tech/paywallo-sdk
- Dashboard Paywallo: configure `placements`, `campaigns`, `variants` e `flags` lá
- Suporte: accounts@virextech.com.br
