# Skill: Paywallo SDK — Tracking de Funil & Eventos

> **SDK alvo:** `@virex-tech/paywallo-sdk` ^2.10.0

> **Pré-requisito:** [`paywallo-sdk-setup.md`](./paywallo-sdk-setup.md) já aplicado.

Esta skill cobre como instrumentar o funil do seu app (welcome → onboarding → auth → home → paywall) para que o dashboard do Paywallo calcule taxa de conversão e identifique drop-off automaticamente.

> ⚠️ **A partir da 2.10, o Paywallo não apresenta paywall.** O paywall próprio e a engine de campanhas foram removidos do SDK — a apresentação é 100% do Superwall, com um bridge automático (`SuperwallAutoBridge`) que traduz os callbacks do Superwall (`onPaywallPresent`/`onPaywallDismiss`/`transactionComplete`) em eventos Paywallo. Você não instrumenta paywall manualmente em nenhum cenário coberto por esta skill.
>
> ⚠️ **`useOnboarding().drop()` não existe mais** (removido na 2.6.0). O abandono do funil é inferido no **servidor** por inatividade — não há chamada de "desistência" no SDK.

---

## 1. Anatomia do funil que o Paywallo mede

| Camada               | Como o SDK rastreia                                                      | O que você instrumenta               |
| :------------------- | :------------------------------------------------------------------------ | :----------------------------------- |
| Install / first open | Automático no boot (evento `$app_installed`)                             | Nada                                 |
| Sessão               | Automático (`$session_start` / `lifecycle{type:"session_end"}`)          | Nada                                 |
| Identify             | `PaywalloClient.identify({ email })`                                     | Após login                           |
| **Onboarding steps** | `useOnboarding().step(stepName, order, options?)`                        | Em cada tela do onboarding           |
| **Onboarding done**  | `useOnboarding().complete(options?)`                                     | Na última tela / antes do paywall    |
| Paywall view/compra  | Automático via bridge do Superwall (evento `paywall`/`transaction`)      | Nada                                  |
| Purchase             | Automático após validação StoreKit / Play (evento `transaction`)        | Nada                                 |
| Eventos custom       | `PaywalloClient.track(name, { properties })`                             | Eventos de produto (CTA, share, ...) |

> Drop-off é detectado pelo **server**: se um user dispara `step("X", n)` mas nunca `step("Y", n+1)` ou `complete()`, ele é contado como drop em "X". Você não precisa (e não pode mais) rastrear isso manualmente — não existe `drop()`.

---

## 2. Onboarding: o padrão correto

Se o app tem uma estrutura tipo `src/features/onboarding/` com fluxo multi-step, plugue o tracking via hook:

```tsx
// src/features/onboarding/hooks/useOnboardingTracking.ts
import { useCallback } from "react";

import { useOnboarding } from "@virex-tech/paywallo-sdk";

interface IUseOnboardingTrackingReturn {
  trackStep: (stepName: string, order: number, options?: { variantKey?: string; timeOnPrevS?: number }) => Promise<void>;
  trackComplete: (options?: { variantKey?: string }) => Promise<void>;
}

export const useOnboardingTracking = (): IUseOnboardingTrackingReturn => {
  const { step, complete } = useOnboarding();

  const trackStep = useCallback(
    async (
      stepName: string,
      order: number,
      options?: { variantKey?: string; timeOnPrevS?: number },
    ): Promise<void> => {
      await step(stepName, order, options);
    },
    [step],
  );

  const trackComplete = useCallback(
    async (options?: { variantKey?: string }): Promise<void> => {
      await complete(options);
    },
    [complete],
  );

  return { trackStep, trackComplete };
};
```

### Assinatura confirmada (`useOnboarding`)

```ts
interface UseOnboardingResult {
  step: (stepName: string, order: number, options?: { variantKey?: string; timeOnPrevS?: number }) => Promise<void>;
  complete: (options?: { variantKey?: string }) => Promise<void>;
}
```

- **`order` é obrigatório** — número finito e **não-negativo** (recomendado 1-based). Sem ele, ou com valor inválido (negativo, `NaN`, `Infinity`), `step()` **lança `OnboardingError`** (`ONBOARDING_INVALID_ORDER`) — quebra em runtime, não é um warning silencioso.
- O servidor faz o **floor** do `order` — não arredonde você mesmo.
- Use **decimais** para variantes A/B do mesmo passo: parte inteira = posição no funil, decimal = variante (ex: `step("paywall_variant_a", 2.1)` e `step("paywall_variant_b", 2.2)`). Para a maioria dos casos, prefira `variantKey` em `options` — é mais legível do que codificar a variante no decimal.
- `options.variantKey` identifica a variante de um teste A/B com uma string legível (`"control"`, `"variant_b"`); `options.timeOnPrevS` registra o tempo gasto na tela anterior, em segundos.
- `complete(options?)` aceita `{ variantKey? }` para fechar o funil pela variante correta.

### Forma imperativa (sem hook)

`PaywalloClient` também expõe um atalho para o mesmo pipeline, útil fora de componentes React (ex: em serviços, listeners de deep link):

```ts
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

await PaywalloClient.onboardingStep(order, stepName);
```

> ⚠️ **Ordem de argumentos invertida em relação ao hook:** `onboardingStep(order, stepName)` recebe `order` **primeiro**, enquanto `useOnboarding().step(stepName, order, options?)` recebe `stepName` primeiro. É uma pegadinha fácil de errar — confira a assinatura antes de usar. Além disso, `onboardingStep` não aceita `options` (`variantKey`/`timeOnPrevS`); para isso use o hook.

### Convenção de naming

`stepName` é uma **string estável e única por tela**, em `snake_case`:

```
welcome
goal_selection
gender
age
weight
target_weight
loading_personalization
result_summary
```

**Regras:**

- Nomes consistentes — não mude depois que estiver em produção (quebra histórico do funil)
- Sem espaços, sem maiúsculas
- Sem `step_1`, `step_2` — use o **nome semântico** da tela

### Plugando no fluxo

No hook que orquestra o passo atual da feature `onboarding`:

```tsx
// src/features/onboarding/hooks/useOnboardingFlow.ts
import { useEffect } from "react";

import { useOnboardingTracking } from "./useOnboardingTracking";

export const useOnboardingFlow = () => {
  const { trackStep, trackComplete } = useOnboardingTracking();
  const currentStepIndex = useOnboardingStore((s) => s.currentStepIndex);
  const currentStep = useOnboardingStore((s) => s.currentStep);
  const isLastStep = useOnboardingStore((s) => s.isLastStep);

  useEffect(() => {
    if (!currentStep?.name) return;
    // order 1-based: index 0 → order 1
    void trackStep(currentStep.name, currentStepIndex + 1);
  }, [currentStep?.name, currentStepIndex, trackStep]);

  const handleFinish = async (): Promise<void> => {
    await trackComplete();
    router.replace(ROUTES.PAYWALL);
  };

  return { handleFinish };
};
```

**Por que `useEffect` aqui é OK** (apesar do CLAUDE.md restringir useEffect a hooks): o efeito de tracking é uma função do estado da máquina de onboarding, não de uma renderização. Mantenha apenas no hook, nunca no componente.

### Abandono explícito (botão "skip")

Não existe mais `drop()`. Para **abandono passivo** (user fechou o app, não terminou), **não chame nada** — o server detecta automaticamente pela ausência do próximo `step` ou de `complete()`.

Se o produto precisa de um sinal explícito de "usuário clicou em pular" (diferente de abandono por inatividade), use um evento **custom** — nunca reaproveite a taxonomia de onboarding para isso:

```tsx
const handleSkipOnboarding = async (): Promise<void> => {
  await PaywalloClient.track("onboarding_skip_clicked", {
    properties: { step_name: currentStep.name },
  });
  router.replace(ROUTES.HOME);
};
```

Isso é opcional — o funil de drop-off no dashboard já funciona sem ele.

---

## 3. Eventos custom (`track`)

Para eventos de produto que **não** são onboarding nem paywall:

```tsx
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

// Engajamento
await PaywalloClient.track("recipe_saved", {
  properties: { recipe_id: "rec_123", category: "vegan" },
});

// Conversão de funil custom
await PaywalloClient.track("share_invitation_sent", {
  properties: { channel: "whatsapp" },
});

// Evento crítico (flush imediato — não enfileira)
await PaywalloClient.track("checkout_started", {
  properties: { plan: "annual" },
  priority: "critical",
});
```

### Assinatura de `track`

```ts
track(eventName: string, options?: {
  properties?: UserProperties;       // Record<string, string|number|boolean|null | Record<string, primitivo>>
  timestamp?: number;                // ms — default: agora
  priority?: "critical" | "normal";  // default: normal
}): Promise<void>
```

**Quando usar `priority: "critical"`:**

- Eventos onde a perda é inaceitável: `transaction`, `identify`, `lifecycle{type:"install"}`, `subscription_*`/`refund`, `checkout_started`
- O SDK já marca automaticamente os eventos de IAP / lifecycle de install como críticos. **Você só precisa marcar eventos de produto que sejam financeiros.**
- `normal` é o default e bufferiza numa janela rolante de **até 25 eventos ou 30 segundos** (o que disparar primeiro) — ideal para tracking de UI, onboarding e eventos de paywall (que também são `normal`).

### Durabilidade (não existe mais "fila offline" do jeito antigo)

A fila offline durável antiga (`getOfflineQueueSize`/`clearOfflineQueue`/`processOfflineQueue`) foi **removida** após um incidente de perda de eventos — esses três métodos continuam exportados por compatibilidade, mas são **no-ops `@deprecated`**: `getOfflineQueueSize()` sempre retorna `0`, os outros dois não fazem nada. Eles saem de vez na 3.0.0 — não construa lógica em cima deles.

O que existe hoje, sem nada para você configurar:

- **Eventos `critical`**: retry durável limitado via `PendingRetry` (2 tentativas, backoff 1min/5min), sobrevive a restart do app.
- **Eventos `normal`** (a partir da 2.10.1): também ganham retry durável via `PendingNormalQueue` — fila persistida com teto de 500 eventos e idade máxima de 24h (o mais antigo é descartado primeiro quando estoura). Reenviado no próximo flush, na volta a foreground e quando a rede volta. 4xx continua descartando na hora (payload inválido — reenviar não muda nada).
- Todo evento perdido de vez (4xx, estouro de fila, expiração por idade) é contado internamente e reportado num evento `$sdk_events_dropped` no próximo flush bem-sucedido — você não precisa fazer nada para isso acontecer, só saber que existe se for auditar perda de eventos no dashboard.

### Convenção de naming de eventos

| Tipo                        | Convenção             | Exemplo                    |
| :-------------------------- | :--------------------- | :-------------------------- |
| Ação do user                | `noun_verb` (passado)  | `recipe_saved`, `goal_set`  |
| Funil custom (não onb.)     | `flow_step_action`     | `meal_plan_step_completed`  |
| Evento crítico (financeiro) | `noun_verb`            | `checkout_started`          |

**Não use** prefixos `$` em eventos custom — esses são reservados para taxonomia interna do SDK. Misturar quebra dashboards e, dependendo do nome, pode ser rejeitado por validação de nome reservado.

---

## 4. Identify enriquecido (atributos de segmentação)

Use `identify` (não `track`) para **atributos do user** que não mudam a cada sessão:

```tsx
await PaywalloClient.identify({
  email: user.email,
  properties: {
    plan: user.plan, // free | premium | trial
    signup_at: user.createdAt, // ISO string
    cohort: user.cohort, // ex: "2026-q2"
    locale: user.locale, // pt-BR
    has_completed_onboarding: true,
  },
});
```

Atributos de identify aparecem no dashboard como filtros segmentáveis ("usuários do plano free que viram paywall X").

**Não envie em identify:**

- `lastActiveAt` — muda a cada sessão (use `track` com session events)
- PII além de email (telefone, endereço) — a menos que use os campos dedicados (`phone`, `firstName`, `lastName`, `dateOfBirth`, `zipCode`, `city`, `state` fazem parte de `IdentifyOptions`)
- Tokens, IDs de sessão

### Limites de propriedades

| Limite                           | Valor                              |
| :------------------------------- | :---------------------------------- |
| Chaves por usuário                | **50** (`boundedProperties`)        |
| Profundidade (se valor é objeto)  | **3 níveis** (recomendado: flat)    |

> O merge de `identify` é **não-destrutivo**: chamar `identify` de novo adiciona/atualiza campos sem apagar os existentes. Eventos disparados antes do `identify` ficam associados ao mesmo usuário. A partir da 2.10.0, `identify()` também **deduplica**: se o payload (traits + PII + atribuição) não mudou nas últimas 24h, a chamada não reenvia rede — o app pode chamar `identify` de novo sem se preocupar com custo.

### Atualizando properties sem mudar email

```tsx
await PaywalloClient.identify({
  properties: { plan: "premium" },
});
```

Passar `identify` sem `email` apenas atualiza properties — útil quando o user faz upgrade de plano e você quer refletir no dashboard sem chamar identify completo.

---

## 5. Que eventos **não** rastrear manualmente

O SDK 2.x usa uma taxonomia de **famílias canônicas** (`lifecycle`, `identify`, `paywall`, `transaction`, `onboarding`, `notification`) — cada uma com um campo `type` que distingue o subtipo. Rastrear manualmente qualquer coisa nessas famílias causa duplicação:

| Família / evento                              | `type` / detalhe                                                                 | Emitido automaticamente em                                              |
| :--------------------------------------------- | :---------------------------------------------------------------------------------- | :------------------------------------------------------------------------ |
| `$app_installed`                               | —                                                                                     | Primeira abertura do app (inclui device model, OS, locale, atribuição)   |
| `$session_start`                               | —                                                                                     | Nova sessão começa                                                       |
| `lifecycle`                                    | `cold_start` / `foreground` / `background` / `session_end`                          | Ciclo de vida do app (foreground, background, fim de sessão)            |
| `paywall`                                      | `viewed` / `closed` / `purchased`                                                    | Bridge automático do Superwall (`onPaywallPresent`/`onPaywallDismiss`)   |
| `transaction`                                  | `completed` / `trial_started` / `renewed` / `refunded` / `canceled` / `expired` / `failed` | Compra validada (via Superwall `transactionComplete` ou IAP direto)      |
| `onboarding`                                   | `step` / `complete`                                                                   | `useOnboarding().step(...)` / `useOnboarding().complete(...)`            |
| `notification`                                 | `delivered` / `displayed` / `clicked` / `dismissed`                                  | Push notification lifecycle                                              |
| `identify`                                     | —                                                                                     | `PaywalloClient.identify(...)`                                            |

> **Não existe mais `$onboarding_dropped`** — a taxonomia de onboarding só tem `step` e `complete`; abandono é inferido, não emitido.

---

## 6. Debugging do funil

### Ver eventos em tempo real

`debug: __DEV__` no Provider faz o SDK logar cada evento enviado. O onboarding tem seu próprio prefixo:

```
[Paywallo:Onboarding] event emitted: onboarding.step { step_name: "goal_selection", order: 2 }
[Paywallo:Onboarding] event emitted: onboarding.complete {}
```

### Forçar flush imediato

Útil em testes E2E:

```tsx
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

await PaywalloClient.track("e2e_marker", { properties: { run: runId } });
// SDK normalmente bufferiza — em E2E você quer flush imediato:
await PaywalloClient.endSession(); // força flush + finaliza sessão
await PaywalloClient.startSession(); // recomeça
```

---

## 7. Checklist de instrumentação

- [ ] Cada tela do onboarding chama `step(stepName, order)` no mount via hook — `order` **sempre** presente e válido (finito, ≥ 0)
- [ ] `complete()` é chamado **uma vez** ao fim do onboarding (antes do paywall)
- [ ] Nenhuma chamada a `drop()` no codebase — não existe desde a 2.6.0
- [ ] Nomes de step são `snake_case`, semânticos, estáveis
- [ ] `identify` é chamado após login com email + atributos de segmentação
- [ ] Eventos críticos (financeiros / de checkout custom) usam `priority: "critical"`
- [ ] Nenhum `track("paywall", ...)`, `track("transaction", ...)` ou `track("onboarding", ...)` manual no codebase — são famílias reservadas ao SDK
- [ ] Em dev, console mostra `[Paywallo:Onboarding] event emitted: ...` para cada step

---

## 8. Anti-patterns

| ❌ Não faça                                                                  | ✅ Faça                                                                |
| :---------------------------------------------------------------------------- | :----------------------------------------------------------------------- |
| `useOnboarding().step("goal_selection")` sem `order`                          | `useOnboarding().step("goal_selection", 3)` — `order` é obrigatório      |
| `useOnboarding().drop(stepName)` (não existe desde a 2.6.0)                  | Nenhuma chamada — abandono é inferido pelo servidor                      |
| `PaywalloClient.onboardingStep(stepName, order)`                              | `PaywalloClient.onboardingStep(order, stepName)` — ordem invertida!      |
| Wrapper `paywalloService.trackOnboardingStep(...)` (não existe)              | Hook `useOnboarding` direto                                              |
| Tracking de step em `useEffect` no **componente** (proibido pelo CLAUDE.md)  | Em hook customizado da feature                                           |
| Renomear `stepName` mid-funcionamento                                        | Decidir nome **antes** de subir para produção                            |
| `track("login_success")` após `identify`                                     | `identify` já é o evento — não duplique                                  |
| `properties` com objetos aninhados em múltiplos níveis                      | Achatar (ex: `goal_type` em vez de `goal: { type: ... }`)                |
| Confiar em `getOfflineQueueSize()`/`processOfflineQueue()` (no-op desde 2.7.0) | Confiar no retry durável interno (`PendingRetry`/`PendingNormalQueue`)   |

---

## 9. Próximos passos

- A/B test de variantes do onboarding ou paywall: [`paywall-ab-testing.md`](./paywall-ab-testing.md)
- Visão consolidada: [`paywallo-full-skill.md`](./paywallo-full-skill.md)
