# Skill: Paywallo SDK — Setup & Core Integration

> **SDK alvo:** `@virex-tech/paywallo-sdk` ^2.10.0
> **Stack alvo:** React Native + Expo SDK 54
> **Stack pressuposta:** Expo Router, Zustand, React Query, i18next, theme tokens

Setup mínimo **funcional** do SDK Paywallo no seu app. Esta skill cobre instalação, configuração, montagem do Provider, identificação de usuário em login/logout e wiring de erros para observability.

> ⚠️ **A partir da 2.10.0, o Paywallo não apresenta paywall.** As APIs de paywall/campanha próprias do SDK (`presentPaywall`, `presentCampaign`, `getPaywall`, `getCampaign`, `preload*`, `isPreloaded`, `requireSubscriptionWithCampaign`, `gateContentWithCampaign`, `register*Presenter`, o paywall de emergência) viraram **stubs inertes**: continuam exportadas (compat de tipo/runtime), não fazem chamada de rede, nunca lançam, e resolvem um shape neutro. A apresentação do paywall é feita pelo **Superwall** (`expo-superwall`, com bridge automático de compra — ver `paywallo-paywall-skill.md`) ou por UI própria via `useProducts` + `usePurchase`. O `PaywalloProvider` **não monta mais modais de paywall** — ele cuida de identidade, sessão, flags, push e tracking.
>
> ⚠️ **Não envolva o SDK em um Singleton wrapper customizado.** O `PaywalloClient` exportado pelo pacote já é singleton. Wrappers adicionam latência (Promise race / timeout) e duplicam lógica que o SDK já fornece (`waitUntilReady`, `onError`).

---

## 1. Requisitos

| Plataforma   | Versão mínima                                      |
| :----------- | :------------------------------------------------- |
| React Native | recente (0.7x+)                                    |
| iOS          | 15.0+                                              |
| Android      | API 24 (Android 7.0+)                              |
| Node         | 18+                                                |
| Expo         | qualquer com development build (não há config plugin) |

---

## 2. Instalação

```bash
npm install @virex-tech/paywallo-sdk
npx expo install expo-application expo-device expo-localization
```

> Equivalentes: `yarn add` / `pnpm add`.

`expo-application`, `expo-device` e `expo-localization` são **peer deps obrigatórias**, importadas estaticamente pelo SDK — a ausência de qualquer uma quebra o build (não é um degrade silencioso, é erro de compilação).

**iOS:**

```bash
cd ios && pod install && cd ..
```

Autolinking do RN 0.60+ resolve o resto. Sem config extra em `Podfile`, `build.gradle` ou `MainApplication`.

**Expo:**

```bash
npx expo prebuild       # gera /ios e /android
cd ios && pod install   # depois disso
```

> Não existe config plugin Expo — use prebuild. Requer **development build** (não funciona no Expo Go).

O SDK traz módulos nativos próprios (StoreKit 2, Google Play Billing, secure storage, device info). **Não é necessário** RevenueCat, `react-native-iap` ou similar.

> ⚠️ **Rebuild nativo obrigatório a cada atualização do SDK.** Atualizações OTA (CodePush, EAS Update) **não bastam** — módulos nativos novos só entram num binário recompilado.

### Peer deps opcionais

| Pacote                                        | Quando instalar                                                                 |
| :--------------------------------------------- | :------------------------------------------------------------------------------- |
| `expo-tracking-transparency`                  | App usa Meta Ads / AppsFlyer / qualquer atribuição que precise do IDFA (iOS).    |
| `@react-native-firebase/app`                  | Push notifications no **Android** (FCM). No iOS o SDK usa APNS direto — sem Firebase. |
| `@react-native-async-storage/async-storage`   | Melhora persistência de cache local (carregado via `import()` dinâmico).         |
| `expo-superwall`                              | Apresentação de paywall via Superwall (bridge automático — ver `paywallo-paywall-skill.md`). |

Se nenhum desses casos se aplica, **não instale**. O SDK detecta a ausência e desliga o subsistema correspondente. Não é necessário `@react-native-firebase/messaging` nem `@notifee/react-native` — push usa o módulo nativo próprio do SDK (detalhe em `docs/push-setup.md` do pacote, ou peça a skill de push se o seu app tiver uma).

---

## 3. Variáveis de ambiente

Adicione em `.env.example` e `.env`:

```env
EXPO_PUBLIC_PAYWALLO_APP_KEY=pk_xxxxxxxx
```

> Formato: `pk_<uuid-sem-hifen>`. Pegue no dashboard do Paywallo (seu app → aba **SDK**). Cole sem espaços.

> Variáveis com prefixo `EXPO_PUBLIC_` ficam embutidas no bundle do app — **não** coloque secrets do servidor aqui.

---

## 4. Plugando o `PaywalloProvider`

O Provider faz automaticamente ao montar:

- chama `PaywalloClient.init(config)` internamente (você **não** chama `init()` manualmente)
- inicia a sessão (`autoStartSession: true` por default)
- pré-resolve `sessionFlags` declaradas na config
- registra os listeners nativos de transação, push e SKAdNetwork

Ele **não** apresenta paywall — se o app usa Superwall, o `SuperwallProvider` entra **dentro** do `PaywalloProvider` (modelo híbrido). Plugue no arquivo do seu app que monta os Providers (ex: `src/components/core/AppProviders/index.tsx`), **acima** dos providers que dependem de identidade do usuário (React Query, navegação):

```tsx
// src/components/core/AppProviders/index.tsx
import React from "react";

import {
  PaywalloProvider,
  type PaywalloInitConfig,
} from "@virex-tech/paywallo-sdk";

import { QueryClientProvider } from "@/lib/react-query";
import { I18nProvider } from "@/i18n";
import { ThemeProvider } from "@/theme";

import { IAppProvidersProps } from "./types";

const PAYWALLO_APP_KEY = process.env.EXPO_PUBLIC_PAYWALLO_APP_KEY ?? "";

// Declarar como variável tipada `PaywalloInitConfig` (não literal inline).
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

### Config completo (`PaywalloInitConfig`, fonte: `src/core/types.ts`)

```ts
interface PaywalloInitConfig {
  appKey: string; // obrigatório
  apiUrl?: string; // default: https://serverjs.paywallo.com.br — só para teste local
  debug?: boolean; // default: false — use __DEV__
  environment?: "Production" | "Sandbox"; // default: deduzido de __DEV__
  autoStartSession?: boolean; // default: true
  sessionConfig?: { sessionTimeoutMs?: number; trackAppState?: boolean };
  sessionFlags?: string[]; // flags pré-resolvidas no boot (síncronas via getSessionFlag)
  subscriptionCacheTTL?: number; // ms — cache do status de subscription
  timeout?: number; // ms — timeout de requests
  errorStrings?: { title?: string; retry?: string; close?: string };
  onError?: (error: unknown) => void;
  notifications?: boolean; // default: auto-detect (liga se o módulo nativo de push estiver linkado)
  skan?: boolean; // default: true — SKAdNetwork conversion value (iOS). false se outro SDK já for dono do valor
}
```

> ⚠️ **Não passe esses campos como literal inline** dentro de `<PaywalloProvider config={{ ... }}>`. O type público aceito pelo Provider é `PaywalloConfig` (subset menor). Para usar campos como `sessionFlags`, `onError`, `sessionConfig`, declare uma variável tipada `PaywalloInitConfig` separada e passe-a por referência — o SDK aceita em runtime.

> **Campos aceitos mas ignorados (stubs `@deprecated`, saem na v3.0.0): `offlineQueueEnabled`, `autoPreloadCampaign`.** Não configure — não têm efeito.

> **`requestATT` foi removido (2.8.0).** O SDK nunca pediu/pede o ATT sozinho. Se o app precisa do IDFA (Meta Ads, AppsFlyer), chame `requestTrackingPermissionsAsync()` de `expo-tracking-transparency` **antes** de montar o `PaywalloProvider`, e declare `NSUserTrackingUsageDescription` no `Info.plist`:

```tsx
import * as TrackingTransparency from "expo-tracking-transparency";
import { Platform } from "react-native";

async function bootstrap() {
  if (Platform.OS === "ios") {
    await TrackingTransparency.requestTrackingPermissionsAsync();
  }
  // só então renderize <PaywalloProvider>
}
```

Se o Provider montar antes do ATT ser resolvido, o install sobe com `attStatus: 'undetermined'` (sem IDFA), degradando a atribuição.

> **`environment`**: o SDK deduz `Sandbox` quando `__DEV__ === true` e `Production` caso contrário. Só sobrescreva se precisar testar Production em dev build, ou Sandbox em release build.

---

## 5. Identify após login

`PaywalloClient.identify(...)` associa o `distinctId` anônimo (criado no primeiro boot) ao seu usuário identificado. **Pode ser chamado antes ou depois do init** — chamadas pré-init são enfileiradas automaticamente.

Plugue no hook de auth do seu app (ex: `src/features/auth/hooks/useAuth.ts`, ou onde a session é hidratada):

```tsx
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

import { useStore } from "@/store";

export const useAuthSync = (): void => {
  const user = useStore((state) => state.user);

  useEffect(() => {
    if (!user) return;

    void PaywalloClient.identify({
      userId: user.id,
      email: user.email,
      properties: {
        plan: user.plan,
        signup_at: user.createdAt,
        // qualquer atributo segmentável — não envie PII além do necessário
      },
    });
  }, [user]);
};
```

### `IdentifyOptions` (fonte: `src/types/index.ts`)

```ts
interface IdentifyOptions {
  userId?: string; // atalho para properties.userId — ver regra abaixo
  email?: string;
  firstName?: string;
  lastName?: string;
  phone?: string;
  dateOfBirth?: string; // YYYY-MM-DD
  gender?: "m" | "f";
  zipCode?: string;
  city?: string;
  state?: string;
  properties?: Record<string, string | number | boolean | null | Record<string, ...>>;
}
```

**Regras:**

- **`userId` é o que liga o usuário.** No nível raiz, é um **atalho** para `properties.userId` — o único campo que o servidor usa para gerar o `external_user_id` (usado em webhooks como `app_user_id`, e como `recipients` no envio de push por usuário). Se você passar os dois (`userId` raiz **e** `properties.userId`) e forem diferentes, **`properties.userId` vence**. Sem `userId`, o push por usuário manda para zero devices (`skippedReason: "no_active_tokens"`).
- **Chame em todo launch logado, não só uma vez.** `identify` é idempotente e o SDK **deduplica payload idêntico por até 24h** (fingerprint do payload em `SecureStorage`) — repetir é barato e cura devices que perderam uma tentativa anterior.
- O merge de `properties` é **não-destrutivo**: chamar `identify` de novo adiciona/atualiza campos sem apagar os existentes.
- Em `properties`, use **valores estáveis** (plano, cohort, `signup_at`). Atributos voláteis (última atividade) não devem ir aqui — use `track`.

### Atualizando properties sem mudar o resto

```tsx
await PaywalloClient.identify({
  properties: { plan: "premium" },
});
```

### Reset no logout

```tsx
await PaywalloClient.reset();
```

`reset()` desfaz o vínculo identificado e gera novo `distinctId`/`anonId`. Use no logout, antes de limpar o estado de auth do app. Depois do próximo login, chame `identify` de novo — senão os tokens/eventos novos ficam anônimos.

> **Não use `fullReset()` em logout normal** — é destrutivo e existe para casos extremos (troca de `appKey`).

> **`deleteUserData()` (LGPD/GDPR)** apaga PII local e gira o `distinctId` — **não sinaliza o servidor** (a rota de exclusão nunca existiu, ver CHANGELOG 2.10.0). É um "esqueça-me" no device, não no backend.

---

## 6. Wiring de erros para observability

O SDK silencia a maioria dos erros de rede/SDK (não derruba o app), mas chama o callback `onError` para você logar:

```tsx
const paywalloConfig: PaywalloInitConfig = {
  appKey: PAYWALLO_APP_KEY,
  debug: __DEV__,
  onError: (error: unknown): void => {
    if (__DEV__) return; // SDK já loga em dev via debug:true
    const err = error instanceof Error ? error : new Error(String(error));
    Sentry?.captureException(err, { tags: { source: "paywallo-sdk" } });
  },
};
```

---

## 7. Verificando que está funcionando

Em dev, com `debug: __DEV__`, o SDK loga no console com prefixo `[Paywallo ...]`. Confira:

- `appKey` copiada sem espaços (formato `pk_xxxxxxxx`)
- `pod install` rodou sem erro (iOS)
- Gradle sync completo (Android)
- As três peer deps obrigatórias (`expo-application`, `expo-device`, `expo-localization`) instaladas

Para confirmar identidade:

```tsx
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

const distinctId = PaywalloClient.getDistinctId(); // string
const email = PaywalloClient.getEmail(); // string | null
```

E no dashboard do Paywallo: o usuário deve aparecer em **Users** após o primeiro `identify`.

---

## 8. Checklist de integração

- [ ] `npm install @virex-tech/paywallo-sdk` + peer deps obrigatórias (`expo-application`, `expo-device`, `expo-localization`) + `pod install` no iOS
- [ ] `EXPO_PUBLIC_PAYWALLO_APP_KEY` no `.env` (e `.env.example` com placeholder)
- [ ] `<PaywalloProvider>` montado em `AppProviders/index.tsx` envolvendo todos os outros providers que dependem de identidade
- [ ] Se o app usa ATT: `requestTrackingPermissionsAsync()` chamado **antes** do Provider montar
- [ ] `identify({ userId, email, properties })` no hook de auth, após o user logar
- [ ] `reset()` no fluxo de logout
- [ ] `onError` plugado no Sentry / logger central
- [ ] Rebuild nativo feito após instalar/atualizar o SDK
- [ ] Em dev, log `[Paywallo INIT]` aparece no console
- [ ] Dashboard mostra o user após primeiro `identify`

---

## 9. O que **não fazer**

| ❌ Anti-pattern                                              | ✅ Faça em vez disso                                              |
| :------------------------------------------------------------ | :------------------------------------------------------------------ |
| `class PaywalloService { ... }` wrapper customizado           | Importe `PaywalloClient` direto                                    |
| `await PaywalloClient.init(...)` manual em `useEffect`        | Use `<PaywalloProvider>` — ele faz init no mount                    |
| `presentPaywall`/`presentCampaign`/`requireSubscriptionWithCampaign` para mostrar paywall | Superwall (`expo-superwall`) ou UI própria via `useProducts` + `usePurchase` — ver `paywallo-paywall-skill.md` |
| `track("login", { email })` como evento manual                | Use `identify` — login não é evento, é estado                      |
| `identify({ email, properties: { plan } })` sem `userId`       | Sempre inclua `userId` se o app faz push por usuário ou usa webhooks de assinatura |
| `console.log` em produção                                     | Use `onError` callback                                              |

Próximas skills:

- [`paywallo-paywall-skill.md`](./paywallo-paywall-skill.md) — apresentar paywall via Superwall e/ou UI própria com offerings
- [`paywallo-funnel-tracking.md`](./paywallo-funnel-tracking.md) — tracking de onboarding e eventos custom
- [`paywall-ab-testing.md`](./paywall-ab-testing.md) — A/B testing com `sessionFlags` e flags
- [`paywallo-full-skill.md`](./paywallo-full-skill.md) — guia consolidado (master)
