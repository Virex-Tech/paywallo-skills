# Skill: Paywallo SDK — Master Guide (Full Integration)

> **SDK alvo:** `@virex-tech/paywallo-sdk` ^2.10.0
> **Stack alvo:** React Native + Expo SDK 54, TypeScript strict, Expo Router, Zustand, React Query, i18next

Guia consolidado para integrar o Paywallo no seu app ponta-a-ponta. Cobre setup, identidade, apresentação de paywall, funil, A/B testing e push notifications. Cada seção aponta para a skill detalhada correspondente.

---

## Visão geral arquitetural (modelo 2.10)

A partir da 2.10.0, o Paywallo **deixou de apresentar paywall**: o engine de campanhas e o paywall próprio (WebView/modal) foram removidos do SDK — as APIs correspondentes (`presentPaywall`, `presentCampaign`, `getPaywall`, `getCampaign`, `preload*`, `requireSubscriptionWithCampaign`, etc.) viram stubs `@deprecated` inertes, mantidos só por compatibilidade de tipo/runtime. O Paywallo passa a ser a camada de **dados**; a **apresentação** do paywall é responsabilidade de outro sistema.

```
┌──────────────────────────────────────────────────────────────────┐
│  seu app                                                          │
│                                                                   │
│   ┌─ PaywalloProvider (identidade, sessão, flags, push, SKAN) ──┐ │
│   │                                                            │ │
│   │  • init() automático no mount (sem chamar init manualmente)│ │
│   │  • sessão iniciada (autoStartSession)                      │ │
│   │  • sessionFlags pré-resolvidas                             │ │
│   │  • Listener nativo de transactions (bridge de compra)       │ │
│   │  • NotificationsManager pronto (se peer deps instaladas)   │ │
│   │  • SKAdNetwork conversion value (iOS, opt-out via skan:false)│ │
│   │                                                            │ │
│   │   ┌─ SuperwallProvider (opcional, apresentação) ──────────┐ │ │
│   │   │  ou UI própria via useProducts + usePurchase          │ │ │
│   │   │                                                       │ │ │
│   │   │  ┌─ Auth flow                                         │ │ │
│   │   │  │   • após login: PaywalloClient.identify({ userId })│ │ │
│   │   │  │   • após logout: PaywalloClient.reset()            │ │ │
│   │   │  │                                                     │ │ │
│   │   │  ┌─ Onboarding flow (feature)                         │ │ │
│   │   │  │   • useOnboarding().step(name, order, options?)    │ │ │
│   │   │  │   • useOnboarding().complete(options?)             │ │ │
│   │   │  │                                                     │ │ │
│   │   │  ┌─ Apresentação de paywall (Superwall ou própria)     │ │ │
│   │   │  │   • syncSuperwallAttributes() + registerPlacement   │ │ │
│   │   │  │   • ou useProducts() + usePurchase()                │ │ │
│   │   │  │                                                     │ │ │
│   │   │  └─ A/B tests / flags                                  │ │ │
│   │   │       • getSessionFlag (sync) / getVariantCached       │ │ │
│   │   │                                                       │ │ │
│   │   └───────────────────────────────────────────────────────┘ │ │
│   └──────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼ HTTPS (https://serverjs.paywallo.com.br)
                  ┌────────────────────────────┐
                  │  Paywallo Server (SaaS)    │
                  │  • Analytics / funnels      │
                  │  • Atribuição (UTM/IDFA)    │
                  │  • Feature flags / A-B      │
                  │  • Push notifications       │
                  │  • Assinatura (via webhook  │
                  │    Apple/Google/Superwall)  │
                  └────────────────────────────┘
```

**Divisão de responsabilidades:**

| Camada                                        | Dono                                                                 |
| :---------------------------------------------| :---------------------------------------------------------------------- |
| Analytics, funil, atribuição, flags, push      | **Paywallo SDK**                                                        |
| Fonte de verdade de assinatura                 | **Webhooks** (Apple/Google/Superwall) → Paywallo — o SDK não valida/envia compras |
| Apresentação do paywall                        | **Superwall** (`expo-superwall`, bridge automático) **ou** UI própria via `useProducts` + `usePurchase` |

> **`base-server` (seu backend) é opcional no fluxo.** Ele só entra se você quiser sincronizar status de assinatura no seu próprio banco a partir dos webhooks do Paywallo — veja a seção 8.

---

## 1. Setup mínimo

### 1.1. Instalar

```bash
npm install @virex-tech/paywallo-sdk
npx expo install expo-application expo-device expo-localization
cd ios && pod install && cd ..
```

`expo-application`, `expo-device`, `expo-localization` são peer deps **obrigatórias** (import estático — ausência quebra o build). Requer development build; **rebuild nativo a cada atualização do SDK** (OTA não basta).

### 1.2. .env

```env
# .env e .env.example
EXPO_PUBLIC_PAYWALLO_APP_KEY=
```

> URL padrão da API: `https://serverjs.paywallo.com.br`. Não configure `apiUrl` em produção — é só para apontar a um servidor local em teste.

### 1.3. Plugar Provider

```tsx
// src/components/core/AppProviders/index.tsx
import {
  PaywalloProvider,
  type PaywalloInitConfig,
} from "@virex-tech/paywallo-sdk";

const PAYWALLO_APP_KEY = process.env.EXPO_PUBLIC_PAYWALLO_APP_KEY ?? "";

// Variável tipada — `PaywalloInitConfig` evita o excess-property-check do TS
// quando o config inclui campos além do subset público (`PaywalloConfig`).
const paywalloConfig: PaywalloInitConfig = {
  appKey: PAYWALLO_APP_KEY,
  debug: __DEV__,
  sessionFlags: ["welcome_variant"],
  onError: (error: unknown): void => {
    if (__DEV__) return;
    // TODO: Sentry.captureException(error, { tags: { source: "paywallo-sdk" } })
    void error;
  },
};

export const AppProviders: React.FC<IAppProvidersProps> = ({ children }) => {
  return (
    <PaywalloProvider config={paywalloConfig}>
      <ThemeProvider>
        <I18nProvider>
          <QueryClientProvider>{children}</QueryClientProvider>
        </I18nProvider>
      </ThemeProvider>
    </PaywalloProvider>
  );
};
```

Se o app apresenta paywall via Superwall, o `SuperwallProvider` entra **dentro** do `PaywalloProvider` (modelo híbrido — ver seção 2 e `paywallo-paywall-skill.md`).

Campos válidos de `PaywalloInitConfig`: `appKey`, `apiUrl?` (só teste local), `debug?`, `environment?`, `autoStartSession?`, `sessionConfig?`, `sessionFlags?`, `timeout?`, `subscriptionCacheTTL?`, `errorStrings?`, `onError?`, `notifications?`, `skan?`. `offlineQueueEnabled`/`autoPreloadCampaign` são aceitos pelo tipo mas **ignorados** (stubs `@deprecated`).

> **`requestATT` foi removido (2.8.0).** Se o app precisa de IDFA, chame `requestTrackingPermissionsAsync()` (`expo-tracking-transparency`) **antes** de montar o Provider, com `NSUserTrackingUsageDescription` no `Info.plist`.

### 1.4. Identify após login + reset no logout

```tsx
// src/features/auth/hooks/usePaywalloAuthSync.ts
import { useEffect } from "react";

import { PaywalloClient } from "@virex-tech/paywallo-sdk";

import { useStore } from "@/store";

export const usePaywalloAuthSync = (): void => {
  const user = useStore((s) => s.user);

  useEffect(() => {
    if (!user) return;
    void PaywalloClient.identify({
      userId: user.id, // liga o external_user_id — necessário p/ push e webhooks
      email: user.email,
      properties: { plan: user.plan, signup_at: user.createdAt },
    });
  }, [user]);
};

// No logout handler:
await PaywalloClient.reset();
```

`userId` (raiz) é atalho para `properties.userId` — se ambos forem passados e divergirem, `properties.userId` vence. Sem `userId`, push por usuário e webhooks de assinatura não conseguem ligar o evento ao seu usuário.

### 1.5. Verificar

Em dev, o console deve mostrar `[Paywallo INIT] ...`. Se não aparecer: confira `appKey` (formato `pk_xxxxxxxx`), `pod install` (iOS), Gradle sync (Android), peer deps obrigatórias instaladas.

> Detalhes: [`paywallo-sdk-setup.md`](./paywallo-sdk-setup.md)

---

## 2. Apresentar paywall

**O Paywallo não apresenta paywall desde a 2.10.0.** Duas formas suportadas:

**A) Superwall (recomendado, bridge automático de compra)**

```tsx
import { syncSuperwallAttributes } from "@virex-tech/paywallo-sdk";
import { registerPlacement } from "expo-superwall";

await syncSuperwallAttributes({ timeoutMs: 1500 }); // empurra atributos pw_* antes do register
await registerPlacement({ placement: "campaign_trigger" });
```

O bridge do SDK intercepta as compras feitas via Superwall e as envia ao Paywallo automaticamente com o contexto de atribuição — **não chame `track` de compra manualmente**, isso duplicaria a receita no dashboard. É obrigatório configurar o webhook Superwall → Paywallo (URL `https://paywallo.com.br/api/webhook/superwall/{app-key}`, header `x-paywallo-secret`, eventos `initial_purchase`, `renewal`, `cancellation`, `uncancellation`, `expiration`, `billing_issue`, `non_renewing_purchase`) — sem ele a assinatura não vira fonte de verdade no Paywallo.

**B) UI própria** (`useProducts` + `usePurchase`)

```tsx
import { useProducts, usePurchase } from "@virex-tech/paywallo-sdk";

const { formattedProducts, isLoading } = useProducts(["app_premium_monthly", "app_premium_annual"]);
const { purchase, isPurchasing } = usePurchase();

// renderize formattedProducts (preço localizado da loja) e chame
// purchase(productId) → { status: "success" | "cancelled" | "failed" }
```

> ⚠️ Na 2.10.x `useOfferings()` sempre retorna `[]` (o `offeringService` nunca é inicializado) e `useOfferingPurchase().purchase` engole falhas. Detalhes e contorno em [`paywallo-paywall-skill.md`](./paywallo-paywall-skill.md) §10.

**Status de assinatura** (funciona com qualquer uma das duas formas — a assinatura vem dos webhooks, não da apresentação): use `usePaywallo().hasActiveSubscription()` (hook `usePremiumStatus` em [`paywallo-paywall-skill.md`](./paywallo-paywall-skill.md) §8).

> ⚠️ **Não use `useSubscription()` na 2.10.x** — `isActive` fica sempre `false` (o `SubscriptionManager` nunca recebe o `distinctId`).

**Restaurar compras** (botão obrigatório nas lojas):

```tsx
import { usePurchase } from "@virex-tech/paywallo-sdk";

const { restore, isRestoring } = usePurchase();
const purchases = await restore();
```

> Detalhes de apresentação, gate de conteúdo e do modelo híbrido Superwall + offerings: [`paywallo-paywall-skill.md`](./paywallo-paywall-skill.md)

---

## 3. Tracking de funil

**Onboarding** (feature `src/features/onboarding/`):

```tsx
import { useOnboarding } from "@virex-tech/paywallo-sdk";

const { step, complete } = useOnboarding();

await step("goal_selection", 1); // order é obrigatório — define a posição no funil
await complete(); // antes do paywall
```

`order` aceita decimais para variantes A/B do mesmo step (ex.: `2` e `2.1`). Há também `options?.variantKey`/`timeOnPrevS` em `step` e `options?.variantKey` em `complete`. **`drop()` foi removido** — o abandono de onboarding é inferido no backend por inatividade.

**Eventos custom**:

```tsx
await PaywalloClient.track("recipe_saved", {
  properties: { recipe_id, category },
});

// Eventos financeiros — flush imediato, com retry durável
await PaywalloClient.track("checkout_started", {
  properties: { plan: "annual" },
  priority: "critical",
});
```

**Properties:** primitivas (`string`/`number`/`boolean`/`null`), um nível de aninhamento é aceito. Convenção: `snake_case`. **Nunca rastreie manualmente** `$paywall_*`, `$onboarding_*`, `paywall` ou `transaction` — o SDK e o bridge do Superwall cuidam disso.

**Batching:** eventos `normal` batcham em janela de ~30s; `critical` flusha imediato (batch-of-1) e tem retry durável via `PendingRetry`. Desde a 2.10.1, eventos `normal` perdidos por falha também têm retry (fila `PendingNormalQueue`, teto de 500/24h).

> Detalhes: [`paywallo-funnel-tracking.md`](./paywallo-funnel-tracking.md)

---

## 4. A/B Testing e feature flags

**Anti-flicker** (variante decide a primeira renderização):

```tsx
// 1. Declarar no Provider
sessionFlags: ["welcome_variant"];

// 2. Ler síncrono
const variant = PaywalloClient.getSessionFlag("welcome_variant") ?? "control";
```

**Runtime** (flag consultada depois, com cache):

```tsx
const result = await PaywalloClient.getVariantCached("home_feature_x", "off");
if (result.variant === "on") {
  /* ... */
}
```

> Detalhes: [`paywall-ab-testing.md`](./paywall-ab-testing.md)

---

## 5. Push notifications (opcional)

**Pré-requisito:** só `@react-native-firebase/app` no **Android** (FCM). No **iOS** o SDK usa APNS direto — sem Firebase, sem `messaging`/`notifee`.

```bash
npm install @react-native-firebase/app
```

Adicione `google-services.json` (Android) em `android/app/`. No iOS, é necessário fiar o `AppDelegate` para repassar o token APNs ao SDK (`PaywalloAPNSTokenReceived`, `PaywalloRemoteNotificationReceived`, `PaywalloNotificationOpened` via `NSNotificationCenter`) — sem isso `getToken()` fica sempre `null`. Ver `docs/push-setup.md` do pacote para o código completo do AppDelegate e a alternativa de swizzling opt-in.

**Pedir permissão** (em momento contextual, não no boot):

```tsx
import { notificationsManager } from "@virex-tech/paywallo-sdk";

const status = await notificationsManager.requestPermission({ provisional: false });
if (status === "granted" || status === "provisional") {
  // token registrado automaticamente no server Paywallo
}
```

Há também `requestPermissionWithPrePrompt({ title, body, acceptLabel, rejectLabel })`, que devolve um handle síncrono (`accept()`/`reject()`) para você montar seu próprio bottom sheet antes do prompt nativo do SO.

**Vincular o token ao usuário — obrigatório para envio por userId:** o token é registrado como anônimo; sem `identify({ userId })`, o envio por `recipients` retorna `sent: 0, skippedReason: "no_active_tokens"` (falha silenciosa, sem erro). Chame `identify` com `userId` em todo launch logado — o vínculo é retroativo.

**Logout**:

```tsx
await PaywalloClient.reset(); // regenera o distinctId anônimo
```

> `notificationsManager.optOut()` virou stub `@deprecated` inerte na 2.10.0 (a rota de exclusão de token no servidor nunca existiu) — para parar de receber push, o usuário precisa desativar a permissão no OS.

> Detalhes completos: `docs/push-setup.md` do pacote `@virex-tech/paywallo-sdk`.

---

## 6. Convenções recomendadas

| Regra                                       | Como aplica ao Paywallo                                                      |
| :------------------------------------------ | :--------------------------------------------------------------------------- |
| Sem `any`                                   | SDK é totalmente tipado — use os types exportados pelo pacote                |
| Sem `console.log`                           | Use o `onError` callback do Provider                                         |
| Sem `useState/useEffect` em **componentes** | Encapsular lógica em hooks (`useAuthSync`, `useSessionFlag`)                 |
| Sem cores/spacing hardcoded                 | UI própria de paywall: usar `theme.colors.*`, `theme.spacing.*`              |
| `testID` obrigatório em interativos         | Botões "Restore", "Subscribe Now", "Maybe later" precisam de `testID`        |
| Features não importam de outras features    | Hooks Paywallo **compartilhados** vão em `src/lib/paywallo/`                 |
| 1 export por arquivo                        | Um hook / componente por arquivo                                             |
| Imports ordenados                           | `react` → expo/3rd-party → `@virex-tech/paywallo-sdk` → `@/...` → relativos  |

### Estrutura sugerida

```
src/
├── lib/
│   └── paywallo/
│       ├── hooks/
│       │   ├── useSessionFlag.ts
│       │   └── index.ts                   # barrel
│       └── index.ts
│
├── features/
│   ├── auth/
│   │   └── hooks/
│   │       └── usePaywalloAuthSync.ts     # identify on login
│   │
│   ├── onboarding/
│   │   └── hooks/
│   │       └── useOnboardingTracking.ts   # step/complete
│   │
│   └── paywall/                           # Superwall e/ou UI própria via offerings
│       ├── index.tsx
│       ├── hooks/
│       │   └── usePaywallScreen.ts
│       └── components/
│
└── components/core/AppProviders/
    └── index.tsx                          # PaywalloProvider (+ SuperwallProvider dentro)
```

---

## 7. Decisão: precisa de servidor (`base-server`)?

**Não, na maioria dos casos.** O Paywallo já é a fonte de verdade de assinatura (via webhooks Apple/Google/Superwall), expõe estado via `usePaywallo().hasActiveSubscription()` / `getSubscription()`.

**Sim, se você precisa:**

- Bloquear endpoints da sua própria API por status de assinatura
- Mostrar status de subscription em web admin / dashboard interno
- Sincronizar `isPremium` no User do seu banco para queries SQL

Nesse caso, configure no seu backend um domínio `subscription` que recebe webhooks do Paywallo:

```
POST /webhooks/paywallo/{secret-uuid}
Body: { event: "subscription_renewed" | "subscription_cancelled" | ..., data: {...} }
```

E atualize o `User.isPremium` correspondente. **Esta é uma extensão fora do escopo do template** — o backend padrão não tem esse domínio por default. Se for implementar: crie o domínio (factory + service + repository + routes + types), adicione model + migration no banco, e configure a URL do webhook no dashboard Paywallo.

---

## 8. Checklist final de integração

### Setup

- [ ] `@virex-tech/paywallo-sdk` instalado + peer deps obrigatórias (`expo-application`, `expo-device`, `expo-localization`) + `pod install`
- [ ] `EXPO_PUBLIC_PAYWALLO_APP_KEY` em `.env` e `.env.example`
- [ ] `<PaywalloProvider>` montado em `AppProviders/index.tsx`
- [ ] `onError` plugado (Sentry / logger)
- [ ] `debug: __DEV__` configurado
- [ ] Rebuild nativo feito após instalar/atualizar o SDK

### Identidade

- [ ] `identify({ userId, email, properties })` no auth hook após login
- [ ] `reset()` no logout
- [ ] Atributos em `properties` são estáveis (sem `lastActiveAt` etc.)

### Funil

- [ ] `useOnboarding().step(name, order)` em cada tela do onboarding, com `order` explícito
- [ ] `complete()` antes do paywall final
- [ ] Nenhum `track("$onboarding_*")` ou `track("$paywall_*")` manual

### Paywall

- [ ] Apresentação via Superwall (webhook configurado) **ou** UI própria via `useProducts` + `usePurchase`
- [ ] `syncSuperwallAttributes({ timeoutMs: 1500 })` chamado antes de todo `registerPlacement`, se usa Superwall
- [ ] Nenhuma chamada a `presentPaywall`/`presentCampaign`/`requireSubscriptionWithCampaign` (removidos, são stubs inertes)
- [ ] Botão "Restore" presente
- [ ] Testado em Sandbox iOS + Google Play License Tester

### A/B testing

- [ ] Flags críticas (primeira renderização) em `sessionFlags`
- [ ] Flags runtime via `getVariantCached` com `defaultValue`
- [ ] Variantes em `snake_case`, com `control` para baseline

### Push (se aplicável)

- [ ] `@react-native-firebase/app` instalado (Android)
- [ ] `google-services.json` configurado (Android); AppDelegate fiado (iOS)
- [ ] `identify({ userId })` chamado — sem isso, push por usuário vai para zero devices
- [ ] `requestPermission()`/`requestPermissionWithPrePrompt()` em momento contextual (não no boot)

### Qualidade

- [ ] `npm run check` passa (Prettier + ESLint + TS)
- [ ] Sem `any`, sem `console.log`, sem `StyleSheet.create` em componentes
- [ ] Componentes interativos têm `testID` + `accessibilityLabel`

---

## 9. Anti-patterns globais

| ❌ Não faça                                              | ✅ Faça                                                              |
| :--------------------------------------------------------- | :---------------------------------------------------------------------- |
| Singleton wrapper `class PaywalloService`                  | Importar `PaywalloClient` direto                                        |
| `init()` manual em `useEffect`                              | Deixar o `PaywalloProvider` cuidar disso                                |
| Sincronizar com RevenueCat / `react-native-iap`             | SDK Paywallo tem IAP nativo — é o único provider                        |
| `presentPaywall`/`presentCampaign`/`requireSubscriptionWithCampaign` | Superwall (`registerPlacement`) ou UI própria (`useProducts` + `usePurchase`) |
| `track("$paywall_*")` manual                                | Nada — SDK/bridge do Superwall rastreiam automaticamente                |
| `track("$onboarding_step", { stepIndex })`                  | `useOnboarding().step(name, order)`                                      |
| Custom Zustand persist para A/B variants                   | `sessionFlags` (sync) ou `getVariantCached` (com cache nativo)          |
| `PaywalloClient.identify({ email })` sem `userId`           | Sempre inclua `userId` se o app usa push por usuário ou webhooks        |
| `if (presentCampaign(...))` (booleano)                      | N/A — API de campanha é stub inerte; use o resultado de `purchase()`/`hasActiveSubscription()` |
| `await getVariant(...)` em primeira tela                    | Adicionar key em `sessionFlags`                                         |
| `console.log` para debug em produção                        | `debug: __DEV__` + `onError`                                            |

---

## 10. Quando algo dá errado

| Sintoma                                                    | Causa provável                                 | Fix                                                                          |
| :--------------------------------------------------------- | :--------------------------------------------- | :--------------------------------------------------------------------------- |
| `presentPaywall`/`presentCampaign` não faz nada             | Removidos na 2.10.0 — são stubs inertes por design | Use Superwall (`registerPlacement`) ou `useProducts` + `usePurchase`  |
| Variants `null` no primeiro boot                           | Sem rede + flag não em `sessionFlags`          | Mover para `sessionFlags` ou aceitar `null` com default                      |
| Eventos não aparecem no dashboard                          | Buffer não flushou, ou 4xx descartou o payload | Conferir `onError`; eventos `normal` perdidos por falha de rede agora têm retry (2.10.1) |
| `useSubscription().isActive` sempre `false`                 | Bug da 2.10.x: `SubscriptionManager` nunca recebe o `distinctId` | Use `usePaywallo().hasActiveSubscription()` (`usePremiumStatus`)            |
| `hasActiveSubscription()` ainda `false` logo após compra    | Webhook da loja/Superwall ainda não processou  | Revalide depois da compra; o crédito definitivo vem do webhook               |
| Push `sent: 0` / `no_active_tokens`                        | Token não vinculado a `userId`                 | `identify({ userId })` após login, em todo cold start logado                |
| Push token não registra no iOS                             | AppDelegate não fiado (`PaywalloAPNSTokenReceived` etc.) | Ver `docs/push-setup.md` do pacote, seção AppDelegate                        |
| `FirebaseApp is not initialized` (Android)                 | `google-services.json` ausente/plugin não aplicado | Conferir `android/app/google-services.json` + plugin Gradle                 |
| iOS reclama de privacy manifest                            | SDK ainda não embarca `PrivacyInfo.xcprivacy` (decisão pendente) | Aguardar atualização do SDK; sem workaround do lado do app                   |

---

## 11. Skills relacionadas

| Skill                                                          | Foco                                                                |
| :------------------------------------------------------------- | :------------------------------------------------------------------ |
| [`paywallo-sdk-setup.md`](./paywallo-sdk-setup.md)             | Instalação, Provider, identify, reset, onError                      |
| [`paywallo-paywall-skill.md`](./paywallo-paywall-skill.md)     | Paywall via Superwall ou UI própria (`useProducts`/`usePurchase`), status de assinatura, restore |
| [`paywallo-funnel-tracking.md`](./paywallo-funnel-tracking.md) | `useOnboarding`, eventos custom, identify enriquecido               |
| [`paywall-ab-testing.md`](./paywall-ab-testing.md)             | `sessionFlags`, `getVariantCached`, conditional flags               |

---

## 12. Referências

- **README do SDK:** `C:\Projects\paywallo\panel-sdk\README.md` (⚠️ Quick start/Imperative API ainda citam `requireSubscriptionWithCampaign` — removido na 2.10.0; confie no código/CHANGELOG, não nesses trechos)
- **CHANGELOG:** `C:\Projects\paywallo\panel-sdk\CHANGELOG.md`
- **Types completos:** `C:\Projects\paywallo\panel-sdk\src\core\types.ts`, `src\types\index.ts`, `dist\index.d.ts`
- **Push:** `C:\Projects\paywallo\panel-sdk\docs\push-setup.md`
- **Dashboard:** `https://paywallo.com.br/dashboard`
- **Documentação web:** `https://paywallo.com.br/docs?page=intro` (alta nível — para detalhes técnicos prefira os types/CHANGELOG do SDK)
