# Skill: Paywallo SDK — A/B Testing & Feature Flags

> **SDK alvo:** `@virex-tech/paywallo-sdk` ^2.10.0

> **Pré-requisito:** [`paywallo-sdk-setup.md`](./paywallo-sdk-setup.md) já aplicado.

Esta skill cobre como usar feature flags e A/B testing do Paywallo no seu app **sem flicker**, sem requests redundantes e sem gerenciar state em paralelo.

> ⚠️ **A partir da 2.10, o Paywallo não apresenta paywall e não faz A/B de paywall.** O paywall próprio e a engine de campanhas foram removidos do SDK (`presentPaywall`, `presentCampaign`, `getPaywall`, `getCampaign`, `preloadCampaign`/`preloadPaywall(s)`, `requireSubscriptionWithCampaign`, `gateContentWithCampaign` e afins viram stubs `@deprecated` inertes — não chame nenhum deles).
>
> **A/B de PAYWALL é feito no Superwall** (campanhas/audiences no dashboard do Superwall), usando os atributos `pw_*` que o Paywallo sincroniza e `syncSuperwallAttributes({ timeoutMs: 1500 })` chamado **imediatamente antes** de cada `registerPlacement` — o Superwall avalia audiences on-device no `register()`, e atributo que chega depois não reclassifica ninguém.
>
> As flags do Paywallo descritas nesta skill (`sessionFlags`/`getSessionFlag`, `getVariantCached`, `getVariant`, `evaluateFlags`, `getConditionalFlag`) servem para **A/B de UI, onboarding e features** — não para escolher qual paywall/placement apresentar.

---

## 1. As três APIs e quando usar cada uma

| API                                   | Quando                                                                 | Latência     | Default       |
| :------------------------------------- | :----------------------------------------------------------------------- | :----------- | :------------ |
| **`sessionFlags` + `getSessionFlag`** | Variantes que decidem renderização inicial (Welcome, onboarding, home)  | **Síncrono** | `null`        |
| **`getVariantCached(key, default)`**  | Flags consultadas em runtime, com cache local                          | ~ms (cache)  | Você define   |
| **`getVariant(key)`**                 | Flag fresca, sem cache (raro — analytics, experimentos novos)          | Rede         | Server decide |

> **Regra de ouro:** se a flag decide **a primeira renderização** de uma tela, ela tem que estar em `sessionFlags`. Caso contrário, você verá flicker (UI default → UI variante).

---

## 2. Padrão A — `sessionFlags` (anti-flicker)

Liste as flags críticas no config do `<PaywalloProvider>` (declare como variável tipada — `sessionFlags` está em `PaywalloInitConfig`, não no subset `PaywalloConfig` que é o type público do Provider):

```tsx
// src/components/core/AppProviders/index.tsx
import { PaywalloProvider, type PaywalloInitConfig } from "@virex-tech/paywallo-sdk";

const paywalloConfig: PaywalloInitConfig = {
  appKey: PAYWALLO_APP_KEY,
  debug: __DEV__,
  sessionFlags: ["welcome_variant", "onboarding_v2", "home_feature_x"],
};

<PaywalloProvider config={paywalloConfig}>
```

O SDK pré-resolve essas flags **durante o init** (antes do app montar telas reais). A leitura depois é **síncrona** (não retorna Promise):

```tsx
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

const variant = PaywalloClient.getSessionFlag("welcome_variant");
//      ^? string | null   (sem await!)
```

### Hook reutilizável

```tsx
// src/lib/paywallo/hooks/useSessionFlag.ts
import { useMemo } from "react";

import { PaywalloClient } from "@virex-tech/paywallo-sdk";

interface IUseSessionFlagReturn {
  variant: string;
  is: (value: string) => boolean;
}

export const useSessionFlag = (
  key: string,
  defaultValue: string = "control",
): IUseSessionFlagReturn => {
  const variant = useMemo<string>(
    () => PaywalloClient.getSessionFlag(key) ?? defaultValue,
    [key, defaultValue],
  );

  return {
    variant,
    is: (value: string): boolean => variant === value,
  };
};
```

### Uso na feature (A/B de UI, não de paywall)

```tsx
// src/features/welcome/hooks/useWelcomeVariant.ts
import { useSessionFlag } from "@/lib/paywallo/hooks/useSessionFlag";

interface IUseWelcomeVariantReturn {
  showVideo: boolean;
  ctaLabel: string;
}

export const useWelcomeVariant = (): IUseWelcomeVariantReturn => {
  const { variant, is } = useSessionFlag("welcome_variant", "control");

  return {
    showVideo: is("with_video"),
    ctaLabel: variant === "urgent_cta" ? "Comece agora" : "Continuar",
  };
};
```

### Usando uma flag para escolher qual placement do Superwall registrar

Se você quer variar **qual paywall aparece** por audience, isso é responsabilidade do Superwall (audiences no dashboard dele). Mas nada impede usar uma flag do Paywallo para escolher **qual placement** disparar — o placement em si (e o paywall dentro dele) continua sendo do Superwall:

```tsx
import { registerPlacement } from "expo-superwall";

import { syncSuperwallAttributes } from "@virex-tech/paywallo-sdk";

import { useSessionFlag } from "@/lib/paywallo/hooks/useSessionFlag";

const { is } = useSessionFlag("checkout_flow_variant", "default");

await syncSuperwallAttributes({ timeoutMs: 1500 });
await registerPlacement({
  placement: is("aggressive") ? "checkout_aggressive" : "checkout_default",
});
```

**O que NÃO fazer:**

- ❌ Persistir `sessionFlags` em Zustand+AsyncStorage com lógica custom — o SDK já cacheia. Você adicionaria um bug de "valor stale após mudança no dashboard".
- ❌ Chamar `getSessionFlag` em `useEffect` → `setState`. É síncrono, pode ler direto no render.
- ❌ Esquecer o `defaultValue`. `null` é um valor válido se a flag ainda não resolveu (offline no first boot).
- ❌ Usar uma flag do Paywallo para chamar `presentPaywall`/`presentCampaign`/`getPaywall`/`getCampaign` — esses métodos são stubs inertes desde a 2.10.0 (sempre resolvem "não apresentado", sem chamada de rede).

---

## 3. Padrão B — `getVariantCached` (flags em runtime)

Para flags consultadas **depois** que o app já está rodando — features condicionais, gates de UI, switches de comportamento que não estão na primeira tela:

```tsx
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

const result = await PaywalloClient.getVariantCached("home_feature_x", "off");
//      ^? FlagVariant { variant: string | null, payload?: Record<string, unknown> }

if (result.variant === "on") {
  // mostra feature
}
```

### Cache em cascata (do mais rápido ao mais lento)

| Camada         | TTL                    | Quando é usado                                         |
| :-------------- | :---------------------- | :-------------------------------------------------------- |
| Memória         | **5 minutos**           | Mesma sessão, chamadas repetidas                           |
| Storage local   | **7 dias**              | Reabertura do app, sem rede                                 |
| `sessionFlags`  | enquanto sessão ativa   | Pré-resolvido no boot (síncrono via `getSessionFlag`)      |
| Servidor        | sempre fresco           | Cache miss / `getVariant` (sem cached)                     |

> Se o device estiver totalmente offline e nunca tiver visto a flag, retorna `variant: null`. Sempre passe um `defaultValue`.

### Hook com Suspense-friendly state

```tsx
// src/lib/paywallo/hooks/useVariant.ts
import { useEffect, useState } from "react";

import { PaywalloClient } from "@virex-tech/paywallo-sdk";

interface IUseVariantReturn {
  variant: string;
  isLoading: boolean;
}

export const useVariant = (
  key: string,
  defaultValue: string = "control",
): IUseVariantReturn => {
  const [variant, setVariant] = useState<string>(defaultValue);
  const [isLoading, setIsLoading] = useState<boolean>(true);

  useEffect(() => {
    let cancelled = false;

    void PaywalloClient.getVariantCached(key, defaultValue).then((result) => {
      if (cancelled) return;
      setVariant(result.variant ?? defaultValue);
      setIsLoading(false);
    });

    return () => {
      cancelled = true;
    };
  }, [key, defaultValue]);

  return { variant, isLoading };
};
```

`getVariantCached` retorna do cache local imediatamente se já resolveu antes; em cache miss, dispara `getVariant` em segundo plano (não bloqueia — a primeira chamada já volta com `defaultValue` se não houver nada em cache). Se a flag é crítica para a primeira renderização, **inclua-a em `sessionFlags`** em vez de depender de `getVariantCached`.

---

## 4. Padrão C — `getVariant` (sem cache, raro)

Quase nunca é o que você quer. Use **apenas** para:

- Experimentos curtos onde o user pode mudar de variante mid-sessão (raro)
- Dashboards admin que mostram a flag em tempo real

```tsx
const result = await PaywalloClient.getVariant("admin_debug_flag");
//                                ^^ sempre vai ao server (com coalescing interno — ver nota)
```

Para tudo mais, prefira `getVariantCached` ou `sessionFlags`.

> **Nota de implementação (2.10.0):** se seu app chama `getVariant()` várias vezes seguidas no boot (uma flag de cada vez), o SDK agrupa automaticamente as chamadas concorrentes numa janela de ~50ms e resolve todas com **uma única chamada em lote** ao servidor — você não precisa fazer nada para aproveitar isso, é transparente à assinatura pública.

---

## 5. Padrão D — `evaluateFlags` (lote explícito)

Quando você já sabe, de antemão, um conjunto de flags que quer resolver de uma vez (por exemplo, fora do boot do Provider, ou para popular um cache próprio), use `evaluateFlags` diretamente:

```ts
const results = await PaywalloClient.evaluateFlags(["flag_a", "flag_b", "flag_c"]);
// { flag_a: "on", flag_b: null, flag_c: "control" }
```

Retorna `Record<string, string | null>` — sem `payload` (para isso, use `getVariant`/`getVariantCached` por chave). É a mesma chamada que `sessionFlags` usa internamente no boot; prefira declarar as flags em `sessionFlags` quando elas afetam a primeira tela, e reserve uma chamada manual a `evaluateFlags` para os casos fora desse fluxo.

---

## 6. Remote config — payload customizado por variante

Cada variante pode ter um `payload` JSON configurado no dashboard. Útil para mudar valores (preços, textos, números) sem republicar o app:

```tsx
const { variant, payload } = await PaywalloClient.getVariant("pricing_config");
// payload = { trialDays: 7, primaryPrice: 9.99, ctaText: "Começar grátis" }

const trialDays = payload?.trialDays ?? 7;
const ctaText = payload?.ctaText ?? "Assinar";
```

**Casos de uso:**

- Preços / descontos sazonais (ajusta no dashboard, app reflete na próxima fetch)
- Textos de CTA por experimento
- Configs numéricas (TTL, limites, thresholds)
- Conteúdo de telas estáticas (FAQ, terms)

> Sempre cheque o tipo do valor — `payload` é `Record<string, unknown>`, não há validação de schema. Use Zod ou cast com fallback se for crítico. `payload` nunca é `null` no tipo público — quando a variante não tem payload, o campo vem **ausente** (`undefined`), nunca `null`.

---

## 7. Conditional flags (regras server-side)

`getConditionalFlag` resolve `true`/`false` baseado em contexto (platform, country, app version) — útil para rollouts graduais:

```tsx
const isEnabled = await PaywalloClient.getConditionalFlag("ai_chat_rollout", {
  platform: Platform.OS,
  appVersion: Constants.expoConfig?.version,
  country: locale.regionCode,
});

if (isEnabled) {
  // mostra feature
}
```

A regra (ex: "habilitar para 10% dos users em iOS BR com app ≥ 2.3.0") é configurada no dashboard. O SDK só envia o contexto.

---

### Persistência da identidade

O bucketing de variante é determinístico por `(distinctId, flagKey)` no servidor. O `distinctId` é persistido localmente em **Keychain** (iOS) e **EncryptedSharedPreferences** (Android), então:

- Reinstalar o app **mantém** a mesma variante (Keychain sobrevive uninstall em iOS, EncryptedSharedPreferences é limpo em Android)
- Logout + login com mesmo email → mesma variante (após `identify`, distinctId é vinculado ao email)
- Trocar de device → **nova variante** (distinctId é por device até identify)

---

## 8. A/B test — significância estatística

No dashboard, cada flag do Paywallo com múltiplas variantes vira um A/B test automático (para UI/onboarding/features). Métricas de conversão por variante são calculadas em tempo real. No código, A/B test é só uma flag — você consome via `getVariant(key)`/`getVariantCached(key, default)` igual a qualquer outra.

**A/B de paywall (preço, criativo, holdout) é uma responsabilidade separada, do Superwall** — configurado como campanhas/audiences no dashboard dele, não como flag do Paywallo.

---

## 9. Tracking que correlaciona variant ↔ outcome

O dashboard do Paywallo correlaciona automaticamente as variantes de `sessionFlags` com todos os eventos da sessão. **Não precisa** passar `variant` manualmente em cada `track`.

Para correlacionar variante de **paywall** com resultado de compra, isso acontece no dashboard do **Superwall** — os atributos `pw_*` sincronizados via `syncSuperwallAttributes` alimentam as audiences dele, e o próprio Superwall reporta conversão por experimento/variante.

Se você integra com Sentry / PostHog / Meta CAPI e quer visibilidade cruzada, passe a variante lá também:

```tsx
const { variant } = useSessionFlag("welcome_variant");

useEffect(() => {
  Sentry?.setTag("welcome_variant", variant);
  posthog?.register({ welcome_variant: variant });
}, [variant]);
```

---

## 10. Naming de flags e variantes

### Flags

`snake_case`, escopo claro, sem sufixo de versão:

```
✅ welcome_variant
✅ home_feature_x
✅ onboarding_flow

❌ welcomeVariant         (camelCase)
❌ welcome_v2             (sufixo de versão — use a variante para isso)
❌ flag1, flag2           (sem semântica)
```

### Variantes

`snake_case`, descritivas. Reserve `control` para o baseline:

```
✅ control, with_video, urgent_cta
✅ price_low, price_mid, price_high

❌ a, b, c                (sem semântica)
❌ test1, test2           (idem)
```

---

## 11. Encerrando experimentos

Quando um A/B test conclui:

1. **Implemente a variante vencedora como default** no código — remova condicionais
2. **Remova a flag de `sessionFlags`** do Provider
3. **Apague o hook** específico do experimento (ex: `useWelcomeVariant`)
4. **Não apague a flag no dashboard** imediatamente — mantenha em modo "completed" para histórico
5. Commit com mensagem explicando: `experiment: roll out 'with_video' as default for welcome_variant`

> Não deixe condicionais zumbi (`if (variant === "control")` quando todos veem `with_video`). Quem ler o código depois vai assumir que ainda é experimento.

---

## 12. Checklist de A/B test

- [ ] Flag declarada no dashboard com variantes nomeadas
- [ ] Se afeta **primeira renderização** de uma tela: incluída em `sessionFlags`
- [ ] Hook próprio em `src/lib/paywallo/hooks/` ou na própria feature
- [ ] Default value definido (`control` ou equivalente)
- [ ] Testado offline (sem rede no primeiro boot) → cai no default sem crashar
- [ ] A/B de **paywall** (preço, criativo) está no dashboard do Superwall, não como flag do Paywallo
- [ ] `syncSuperwallAttributes({ timeoutMs: 1500 })` chamado antes de todo `registerPlacement` que dependa de audience por atribuição
- [ ] Variantes adicionais como `tag` no Sentry/PostHog se usar
- [ ] Documentado em comentário ou ADR: o que está sendo testado, hipótese, métrica de sucesso

---

## 13. Anti-patterns

| ❌ Não faça                                                          | ✅ Faça                                                                 |
| :---------------------------------------------------------------------- | :--------------------------------------------------------------------- |
| Zustand store custom replicando flags do Paywallo                       | `sessionFlags` no config + `useSessionFlag` síncrono                    |
| `await PaywalloClient.getVariant(...)` em primeira tela                 | Adicione a key em `sessionFlags`                                        |
| Usar flag do Paywallo para escolher `presentCampaign`/`presentPaywall`/`getPaywall`/`getCampaign` | Esses métodos são stubs inertes desde 2.10.0 — A/B de paywall é 100% Superwall |
| Escolher preço/criativo de paywall via flag do Paywallo                 | Configurar como campanha/audience no dashboard do Superwall             |
| Variantes em `camelCase` ou com espaços                                 | `snake_case` consistente                                               |
| Esquecer `defaultValue` — assumir que SDK sempre devolveu valor          | Sempre passe default                                                   |
| Manter `if (variant === "control") ...` após experimento concluído      | Remover condicional, codar a variante vencedora como base               |
| Chamar `registerPlacement` sem `syncSuperwallAttributes` antes, esperando audience por atribuição correta | `await syncSuperwallAttributes({ timeoutMs: 1500 })` imediatamente antes |

---

## 14. Próximos passos

- Visão consolidada de todos os padrões: [`paywallo-full-skill.md`](./paywallo-full-skill.md)
