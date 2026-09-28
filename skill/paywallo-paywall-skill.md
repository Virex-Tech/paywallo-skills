# Skill: Paywallo SDK — Paywall (via Superwall), Gate de Conteúdo & Assinatura

> **SDK alvo:** `@virex-tech/paywallo-sdk` ^2.10.0 (+ `expo-superwall` para apresentar o paywall)
> **Pré-requisito:** [`paywallo-sdk-setup.md`](./paywallo-sdk-setup.md) já aplicado: `PaywalloProvider` montado e `identify` plugado.
> **Docs oficiais:** https://paywallo.com.br/docs/superwall · https://paywallo.com.br/docs/subscriptions · https://paywallo.com.br/docs/stores

Esta skill cobre como apresentar paywall, fazer o gate de features premium, verificar assinatura e restaurar compras num app React Native + Expo com Paywallo 2.10+.

> ⚠️ **A partir da 2.10.0 o Paywallo NÃO apresenta paywall.** O paywall próprio (WebView), a engine de campanhas e o preload no `init()` foram **removidos**. As APIs abaixo continuam exportadas só por compatibilidade e viraram **stubs `@deprecated` inertes**: não chamam rede, nunca lançam e resolvem um shape neutro. **Se o app seguir o padrão antigo, o paywall simplesmente nunca aparece**, sem erro nenhum.

### APIs removidas (stubs inertes) → o que usar no lugar

| API removida (2.10.0)                                                                                           | Retorno do stub                                              | Substituto                                                  |
| :-------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------- | :---------------------------------------------------------- |
| `presentPaywall`, `presentCampaign` (no `PaywalloClient` **e** no `usePaywallo()`)                              | `{ presented: false, purchased: false, cancelled: false, restored: false }` | `registerPlacement` do `expo-superwall` (seção 5)            |
| `requireSubscriptionWithCampaign`                                                                               | Só o status de assinatura atual, nunca apresenta nada        | `useSubscriptionGate(placement).unlock()` (seção 5)         |
| `gateContentWithCampaign`                                                                                       | Executa `content` só se já for assinante                     | `useSubscriptionGate` + a ação dentro do `if (granted)`     |
| `getPaywall`, `getCampaign`, `getPaywallConfig` (context)                                                        | `null`                                                       | Configurar o paywall no dashboard do Superwall              |
| `preloadCampaign`, `preloadPaywall(s)`, `getPreloadedCampaign`, `isPreloaded`, `isPaywallPreloaded`             | `{ success: false }` / `null` / `false`                      | Nada: o Superwall faz o próprio preload                     |
| `getAutoPreloadedPlacement`, `waitForAutoPreloadedPlacement`                                                    | `null`                                                       | Nada                                                        |
| `registerPaywallPresenter`, `registerCampaignPresenter`, `registerEmergencyPaywallHandler`                      | no-op                                                        | Nada                                                        |
| `getEmergencyPaywall`                                                                                           | `{ enabled: false }`                                         | Nada                                                        |
| `PaywallModal`, `usePaywallContext`                                                                             | Não renderiza UI                                             | Nada                                                        |

> ⚠️ **Armadilha silenciosa:** `PaywalloClient.requireSubscription(placement)` e `gateContent(fn, placement)` **não** estão marcados como `@deprecated`, mas o ramo que apresentava paywall agora chama o stub. Na prática viram "já é assinante? então sim; senão, não", e nenhum paywall aparece. Não use para gate.

> ⚠️ Os campos `presentPaywall`/`presentCampaign`/`preloadCampaign`/`getPaywallConfig` retornados por `usePaywallo()` também são stubs, e o tipo do context **não** os marca como deprecated (o editor não avisa). Busque por eles no código ao migrar.

**Migração:** procure no app por `presentCampaign`, `presentPaywall`, `requireSubscriptionWithCampaign`, `gateContentWithCampaign`, `requireSubscription(`, `gateContent(` e `preload` e troque todos pelo gate da seção 5.

---

## 1. Quem faz o quê

| Responsabilidade                                               | Quem faz                                                                  |
| :------------------------------------------------------------- | :------------------------------------------------------------------------ |
| Renderizar o paywall, A/B de paywall, audiences, compra        | **Superwall** (`expo-superwall`), configurado no dashboard do Superwall   |
| Eventos de paywall (`viewed`/`closed`/`purchased`) no Paywallo | **Automático**: bridge do SDK Paywallo (seção 3)                          |
| Evento de receita (`transaction`) no Paywallo                  | **Automático**: bridge, no `transactionComplete` do Superwall             |
| Atribuição (`pw_*`) disponível nas audiences do Superwall      | **Automático** + `syncSuperwallAttributes()` antes de cada register (seção 4) |
| Venda verificada, renovação, cancelamento, reembolso           | **Webhooks no servidor**: Superwall → Paywallo (seção 7) + lojas          |
| Checar se o usuário é assinante                                | `usePaywallo().hasActiveSubscription()` (seção 8)                          |
| Restaurar compras                                              | Botão de restore do paywall Superwall + botão em Settings (seção 9)        |

> ⚠️ **Não rastreie paywall nem compra manualmente.** Nada de `track("paywall", ...)`, `track("transaction", ...)`, `track("$paywall_viewed")` etc. O bridge já emite e o webhook já registra a venda. Tracking manual duplica o funil e a receita no dashboard.

---

## 2. Instalação e ordem dos providers

```bash
npx expo install expo-superwall
```

`expo-superwall` é peer **opcional** do SDK Paywallo: se estiver instalado, o bridge liga sozinho. Precisa de dev client / rebuild nativo (não roda no Expo Go).

**Ordem:** `PaywalloProvider` **por fora**, `SuperwallProvider` **dentro**. O Paywallo inicializa primeiro, liga o bridge e empurra os atributos `pw_*` antes do primeiro `registerPlacement`.

```tsx
// src/components/core/AppProviders/index.tsx
import React from "react";
import { SuperwallProvider } from "expo-superwall";
import { type PaywalloInitConfig, PaywalloProvider } from "@virex-tech/paywallo-sdk";

interface IAppProvidersProps {
  children: React.ReactNode;
}

const paywalloConfig: PaywalloInitConfig = {
  appKey: process.env.EXPO_PUBLIC_PAYWALLO_APP_KEY ?? "",
  debug: __DEV__,
};

const SUPERWALL_API_KEYS = {
  ios: process.env.EXPO_PUBLIC_SUPERWALL_IOS_API_KEY ?? "",
  android: process.env.EXPO_PUBLIC_SUPERWALL_ANDROID_API_KEY ?? "",
};

export const AppProviders: React.FC<IAppProvidersProps> = ({ children }) => (
  <PaywalloProvider config={paywalloConfig}>
    <SuperwallProvider apiKeys={SUPERWALL_API_KEYS}>
      {/* bridges de identidade (seção 6), ThemeProvider, QueryProvider... */}
      {children}
    </SuperwallProvider>
  </PaywalloProvider>
);
```

---

## 3. Bridge automático Superwall → Paywallo

Ao montar, o `PaywalloProvider` detecta o `expo-superwall` (import dinâmico; se não houver, é no-op) e assina **globalmente** os eventos de todos os `usePlacement` do app. Você não escreve nada:

| Evento do Superwall                     | Vira no Paywallo                                                                           |
| :-------------------------------------- | :----------------------------------------------------------------------------------------- |
| `onPaywallPresent`                      | `paywall` `{ type: "viewed", paywall_id, placement, variant_id, campaign_id }`              |
| `onPaywallDismiss`                      | `paywall` `{ type: "closed", close_reason: "purchased" \| "restored" \| "dismissed" }`      |
| `onPurchase`                            | `paywall` `{ type: "purchased", product_id }`                                               |
| `transactionComplete` (via `onSuperwallEvent`) | `transaction` (prioridade `critical`) com `transaction_id`, `product_id`, `amount`, `currency`; trial grátis vai com `amount: 0` e `full_price` |

- Restore **não** conta como conversão (`close_reason: "restored"`, não `"purchased"`).
- Eventos `paywall` são telemetria em lote (`normal`); o `transaction` sai na hora, com retry durável.
- Com `debug: true` no config, o bridge loga com o prefixo `[Paywallo:SuperwallBridge]`.

---

## 4. `syncSuperwallAttributes()`: obrigatório antes de cada `registerPlacement`

O Paywallo empurra a atribuição do usuário para o Superwall como user attributes com prefixo `pw_` (`pw_ad_network`, `pw_is_paid`, `pw_match_type`, `pw_utm_*`, `pw_fbclid`...). Com eles você cria **audiences** no Superwall (ex.: `pw_ad_network is meta` vê um paywall, orgânico `pw_is_paid is false` vê outro).

**Por que chamar antes do register:** o Superwall avalia as audiences **no momento do `register()`**, e a variante escolhida ali **gruda no usuário** até o assignment ser resetado. Parte da atribuição (deferred match) só resolve alguns segundos depois do primeiro open. Se o register rodar antes, quem veio de anúncio é avaliado como orgânico e fica assim para sempre.

```ts
import { syncSuperwallAttributes } from "@virex-tech/paywallo-sdk";

const outcome = await syncSuperwallAttributes({ timeoutMs: 1500 }); // "synced" | "timeout" | "skipped"
await registerPlacement({ placement: "feature_x_unlock" });
```

- **Nunca lança** e nunca segura o paywall além do `timeoutMs` (prazo total). Seguro chamar em **todo** register.
- `"skipped"`: `expo-superwall` ausente ou config do Superwall falhou. `"timeout"`: não deu tempo, o register segue sem os atributos novos. Logar o retorno é opcional.
- Disponível desde a 2.9.0. O hook da seção 5 já faz isso; se você chamar `registerPlacement` em outro lugar, repita o padrão.

---

## 5. Gate de feature premium: `useSubscriptionGate(placement)`

Um hook do app encapsula `usePlacement` do `expo-superwall` + sync + register e devolve uma Promise booleana, no lugar do antigo `requireSubscriptionWithCampaign`.

```ts
// src/lib/superwall/hooks/useSubscriptionGate.ts
import { useCallback, useMemo, useRef, useState } from "react";
import { usePlacement } from "expo-superwall";
import { syncSuperwallAttributes } from "@virex-tech/paywallo-sdk";

interface IUseSubscriptionGateReturn {
  unlock: () => Promise<boolean>;
  isUnlocking: boolean;
}

/**
 * Gate de feature premium via Superwall.
 *
 * `unlock()` registra o placement (configurado no dashboard do Superwall) e
 * resolve `true` quando o usuário tem acesso (já assinante, holdout/skip ou
 * compra/restore bem-sucedido) e `false` quando recusa o paywall ou dá erro.
 */
export const useSubscriptionGate = (placement: string): IUseSubscriptionGateReturn => {
  const [isUnlocking, setIsUnlocking] = useState<boolean>(false);
  const resolveRef = useRef<((granted: boolean) => void) | null>(null);

  const settle = useCallback((granted: boolean): void => {
    setIsUnlocking(false);
    if (resolveRef.current) {
      resolveRef.current(granted);
      resolveRef.current = null;
    }
  }, []);

  const { registerPlacement } = usePlacement({
    onDismiss: (_info, result) => {
      settle(result.type === "purchased" || result.type === "restored");
    },
    onSkip: () => {
      settle(true);
    },
    onError: () => {
      settle(false);
    },
  });

  // registerPlacement muda de identidade a cada render; estabiliza via ref.
  const registerPlacementRef = useRef(registerPlacement);
  registerPlacementRef.current = registerPlacement;

  const unlock = useCallback((): Promise<boolean> => {
    // Já existe um unlock em andamento: não empilha outro paywall.
    if (resolveRef.current) return Promise.resolve(false);

    setIsUnlocking(true);

    return new Promise<boolean>((resolve) => {
      resolveRef.current = resolve;
      // Atributos pw_* ANTES do register, senão a audience avalia sem eles
      // e a variante gruda errada. Nunca rejeita; no pior caso, timeout.
      void syncSuperwallAttributes({ timeoutMs: 1500 })
        .then(() =>
          registerPlacementRef.current({
            placement,
            feature: () => {
              settle(true);
            },
          }),
        )
        .catch(() => {
          settle(false);
        });
    });
  }, [placement, settle]);

  return useMemo(() => ({ unlock, isUnlocking }), [unlock, isUnlocking]);
};
```

### Uso numa tela

```tsx
import React from "react";
import { Pressable, Text } from "react-native";
import { router } from "expo-router";
import { useTranslation } from "react-i18next";
import { useQueryClient } from "@tanstack/react-query";

import { useSubscriptionGate } from "@/lib/superwall";
import { PREMIUM_STATUS_QUERY_KEY } from "@/lib/paywallo";

export const ExportButton: React.FC = () => {
  const { t } = useTranslation();
  const queryClient = useQueryClient();
  const { unlock, isUnlocking } = useSubscriptionGate("export_pdf");

  const handlePress = async (): Promise<void> => {
    const granted = await unlock();
    if (!granted) return; // recusou o paywall: não faz nada

    await queryClient.invalidateQueries({ queryKey: PREMIUM_STATUS_QUERY_KEY });
    router.push("/export");
  };

  return (
    <Pressable onPress={handlePress} disabled={isUnlocking}>
      <Text>{t("export.cta")}</Text>
    </Pressable>
  );
};
```

### Como cada desfecho resolve

| O que o Superwall fez                                            | Callback                          | `unlock()` |
| :--------------------------------------------------------------- | :-------------------------------- | :--------- |
| Usuário já tem entitlement, sem paywall                          | `feature()`                       | `true`     |
| Comprou no paywall                                               | `onDismiss` `purchased`           | `true`     |
| Restaurou no paywall                                             | `onDismiss` `restored`            | `true`     |
| Fechou sem comprar                                               | `onDismiss` `declined`            | `false`    |
| `Holdout` / `NoAudienceMatch` / `PlacementNotFound`              | `onSkip(reason)`                  | `true`     |
| Erro de apresentação                                             | `onError`                         | `false`    |

> ⚠️ **Marque o paywall como _Gated_ no Superwall** (feature gating da campanha). Num paywall _Non Gated_ o Superwall executa `feature()` mesmo sem compra, e o gate libera a feature de graça.

> ⚠️ `onSkip` libera por padrão, e isso inclui `PlacementNotFound` (placement com nome errado ou sem campanha). Se preferir falhar fechado, troque `onSkip: () => settle(true)` por `onSkip: (reason) => settle(reason.type !== "PlacementNotFound")`. Confira o nome do placement no dashboard antes do release.

**Regra de placement:** um placement por superfície de UI (`onboarding_end`, `home_unlock`, `export_pdf`). Paywall, preço e A/B ficam no dashboard do Superwall e trocam sem release. Não crie placements diferentes para variar preço.

---

## 6. Identidade: o MESMO id nos dois SDKs

O webhook do Superwall liga a venda ao usuário do Paywallo pelo `app user id` do Superwall, comparado com o `userId` que o Paywallo recebeu no `identify`. **Ids diferentes = venda sem dono** (cai como "Direto", sem atribuição do anúncio).

```ts
// dentro de um bridge montado abaixo dos dois providers
import { useEffect, useRef } from "react";
import { useUser } from "expo-superwall";
import { PaywalloClient } from "@virex-tech/paywallo-sdk";

import { useStore } from "@/store";

export const useBillingIdentitySync = (): void => {
  const user = useStore((state) => state.user);
  const { identify, signOut } = useUser();
  const previousUserIdRef = useRef<string | null>(null);

  useEffect(() => {
    const previousUserId = previousUserIdRef.current;
    const currentUserId = user?.id ?? null;
    if (previousUserId === currentUserId) return;

    if (currentUserId) {
      void identify(currentUserId); // Superwall
      void PaywalloClient.identify({ userId: currentUserId, email: user?.email ?? undefined }); // Paywallo
    } else if (previousUserId) {
      void signOut();
      void PaywalloClient.reset();
    }

    previousUserIdRef.current = currentUserId;
  }, [user, identify, signOut]);
};
```

- `identify({ userId })` é atalho para `properties.userId` (vira o `external_user_id` que o webhook usa).
- Se o app vende **antes** do cadastro, gere um id próprio (UUID salvo no device) antes do paywall, use-o nos dois `identify` e mantenha-o depois do cadastro.
- Na maioria dos apps o setup já tem um `usePaywalloAuthSync`. Basta um `useSuperwallAuthSync` equivalente usando o mesmo `user.id`.

---

## 7. Webhooks: sem eles, as assinaturas não aparecem no Paywallo

O SDK não credita venda. Uma transação só vira **Comprador** e entra na receita quando o servidor recebe o webhook e ela fica `verified`.

### 7.1 Superwall → Paywallo (obrigatório no caminho Superwall)

1. Paywallo: **Configurações → Integrações → Superwall**. Ative e copie a URL e o secret.
2. Superwall: **Settings → Webhooks**. Adicione a URL:
   `https://paywallo.com.br/api/webhook/superwall/{sua-app-key}`
3. Adicione o custom header:

| Header              | Valor                           |
| :------------------ | :------------------------------ |
| `x-paywallo-secret` | o secret gerado pelo Paywallo   |

   Sem o header correto o Paywallo responde `401`.
4. Marque **todos** os eventos obrigatórios:

| Evento                   | O que quebra se faltar                                  |
| :----------------------- | :------------------------------------------------------ |
| `initial_purchase`       | Primeira compra/trial não entra                         |
| `renewal`                | Renovações não entram na receita                        |
| `cancellation`           | Reembolso continua contando como venda                  |
| `uncancellation`         | Reativação não é registrada                             |
| `expiration`             | Assinatura segue "ativa" depois de vencida              |
| `billing_issue`          | Problema de cobrança não aparece                        |
| `non_renewing_purchase`  | Compras avulsas/lifetime não entram                     |

**Validar:** o botão "Testar Conexão" do Paywallo só confirma que a integração está ativa, não recebe evento. Para validar de verdade, faça uma compra sandbox e confira em **Transações** em até ~1 min. Compras sandbox são verificadas automaticamente.

### 7.2 Lojas (Apple / Google)

Configure também **Conectar lojas** no Paywallo (App Store Server Notifications V2 e Google RTDN, docs `stores-apple` / `stores-google`). É o que verifica as compras feitas fora do Superwall (caminho da seção 10) e mantém renovação e reembolso em dia.

---

## 8. Verificar assinatura sem apresentar paywall

Para mostrar ou esconder UI (badge premium, banner de upgrade), use `hasActiveSubscription()` do context. Ele confere as transações ativas **no próprio device** (StoreKit 2 / Play Billing) primeiro, depois o cache e por último o servidor (`/sdk/purchases/status` pelo `distinctId`). Funciona com compra feita pelo Superwall.

```ts
// src/lib/paywallo/hooks/usePremiumStatus.ts
import { useCallback } from "react";
import { useQuery } from "@tanstack/react-query";
import { usePaywallo } from "@virex-tech/paywallo-sdk";

export const PREMIUM_STATUS_QUERY_KEY = ["paywallo", "premium-status"] as const;

interface IUsePremiumStatusReturn {
  isPremium: boolean;
  isLoading: boolean;
  refresh: () => Promise<void>;
}

export const usePremiumStatus = (): IUsePremiumStatusReturn => {
  const { isInitialized, distinctId, hasActiveSubscription } = usePaywallo();

  const query = useQuery({
    queryKey: [...PREMIUM_STATUS_QUERY_KEY, distinctId],
    queryFn: (): Promise<boolean> => hasActiveSubscription(),
    enabled: isInitialized, // o SDK precisa ter terminado o init
    staleTime: 60_000,
  });

  const { refetch } = query;
  const refresh = useCallback(async (): Promise<void> => {
    await refetch();
  }, [refetch]);

  return { isPremium: query.data ?? false, isLoading: query.isLoading, refresh };
};
```

| Opção                                        | Quando usar                                                                                                                                   |
| :------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- |
| `usePaywallo().hasActiveSubscription()`      | **Padrão** em componentes (hook acima). Nunca lança, retorna `false` em erro.                                                                 |
| `PaywalloClient.hasActiveSubscription()`     | Fora de componente (store, service). **Lança** se o SDK não terminou o init. Faça `await PaywalloClient.waitUntilReady()` antes.              |
| `usePaywallo().getSubscription()` / `PaywalloClient.getSubscription()` | Detalhes: `{ productId, status, expiresAt, platform, autoRenewEnabled, inGracePeriod }` ou `null`. Vem do servidor, alimentado pelos webhooks. |
| `useUser().subscriptionStatus` (`expo-superwall`) | Alternativa reativa on-device no caminho Superwall: `status === "ACTIVE"`.                                                              |

> ⚠️ **Não use `useSubscription()` na 2.10.x.** O `SubscriptionManager` que ele usa nunca recebe o `distinctId` (`setUserId` não é chamado em lugar nenhum do SDK), então a consulta sai com `distinctId=` vazio, o servidor rejeita e `isActive` fica sempre `false`. O mesmo vale para `useSubscription().restore()`. Use o hook acima até isso ser corrigido no SDK.

> Não precisa de polling. Invalide `PREMIUM_STATUS_QUERY_KEY` depois de `unlock()` = `true` e depois de restore (seção 9).

---

## 9. Restaurar compras (Apple Guideline 3.1.1)

Restore é **obrigatório**. Precisa existir em dois lugares:

1. **No paywall:** ative o botão de restore no editor do Superwall. O resultado `restored` já resolve o gate como `true` e o bridge registra `close_reason: "restored"`.
2. **Em Settings** (ou login): botão explícito do usuário, **nunca** num efeito de abertura de tela (a Apple pode pedir a senha do Apple ID).

```ts
// src/lib/superwall/hooks/useRestorePurchases.ts
import { useCallback, useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { useSuperwall } from "expo-superwall";
import { usePaywallo } from "@virex-tech/paywallo-sdk";

import { PREMIUM_STATUS_QUERY_KEY } from "@/lib/paywallo";

type TRestoreOutcome = "restored" | "nothing_to_restore" | "failed";

interface IUseRestorePurchasesReturn {
  restore: () => Promise<TRestoreOutcome>;
  isRestoring: boolean;
}

export const useRestorePurchases = (): IUseRestorePurchasesReturn => {
  const restorePurchases = useSuperwall((state) => state.restorePurchases);
  const { hasActiveSubscription } = usePaywallo();
  const queryClient = useQueryClient();
  const [isRestoring, setIsRestoring] = useState<boolean>(false);

  const restore = useCallback(async (): Promise<TRestoreOutcome> => {
    setIsRestoring(true);
    try {
      const response = await restorePurchases();
      if (response.result === "failed") return "failed";

      const isActive = await hasActiveSubscription();
      await queryClient.invalidateQueries({ queryKey: PREMIUM_STATUS_QUERY_KEY });
      return isActive ? "restored" : "nothing_to_restore";
    } finally {
      setIsRestoring(false);
    }
  }, [restorePurchases, hasActiveSubscription, queryClient]);

  return { restore, isRestoring };
};
```

- **Sem Superwall** (UI própria, seção 10): use `usePaywallo().restorePurchases()`, que retorna `{ success, restoredProducts: string[], error? }`, ou `usePurchase().restore()`, que retorna `IAPPurchase[]`. Os dois releem as transações ativas do device e finalizam cada uma.
- ⚠️ Os dois engolem erro da loja e devolvem vazio, então "falhou" e "nada para restaurar" chegam iguais. Mostre uma mensagem neutra ("Nenhuma assinatura encontrada") nesse caso.
- Mostre o texto sempre via i18n (`t("settings.restore")`), nunca hardcoded.

---

## 10. Caminho alternativo: UI de paywall própria (sem Superwall)

Só use se o app **não** vai usar Superwall. Aqui o app desenha o paywall e o SDK faz a compra nativa (StoreKit 2 / Play Billing, precisa de dev client).

| Hook                         | Retorno                                                                                                    | Status na 2.10                                  |
| :--------------------------- | :--------------------------------------------------------------------------------------------------------- | :---------------------------------------------- |
| `useProducts(productIds?)`   | `{ products, formattedProducts, isLoading, error, loadProducts, getProduct, getFormattedProduct, getPriceInfo }` (preço localizado da loja) | ✅ Funcional                                     |
| `usePurchase()`              | `{ state, purchase(productId) → { status: "success" \| "cancelled" \| "failed", purchase?, error? }, restore() → IAPPurchase[], error, isPurchasing, isRestoring }` | ✅ Funcional (**recomendado** para comprar)     |
| `useOfferingPurchase()`      | `{ purchase(productId) → Promise<void>, isPurchasing }`                                                    | ⚠️ Não informa o desfecho: falha da loja resolve em silêncio, só lança em exceção. Prefira `usePurchase()` |
| `useOfferings(ids?)`         | `{ offerings: Offering[], isLoading }` (qual produto mostrar, trocável no dashboard sem release)            | ⚠️ Retorna `[]` sempre, a menos que o app inicialize o serviço (ver abaixo) |

```tsx
// src/features/paywall/components/CustomPaywall.tsx
import React from "react";
import { Pressable, Text, View } from "react-native";
import { useTranslation } from "react-i18next";
import { useProducts, usePurchase } from "@virex-tech/paywallo-sdk";

const PRODUCT_IDS = ["app_premium_monthly", "app_premium_annual"];

interface ICustomPaywallProps {
  onUnlocked: () => void;
}

export const CustomPaywall: React.FC<ICustomPaywallProps> = ({ onUnlocked }) => {
  const { t } = useTranslation();
  const { formattedProducts } = useProducts(PRODUCT_IDS);
  const { purchase, isPurchasing } = usePurchase();

  const handleBuy = async (productId: string): Promise<void> => {
    const outcome = await purchase(productId);
    if (outcome.status === "success") onUnlocked();
    // "cancelled": usuário fechou a sheet nativa, não faz nada
    // "failed": outcome.error é PurchaseError; mostre feedback genérico
  };

  return (
    <View>
      {formattedProducts.map((product) => (
        <Pressable key={product.productId} disabled={isPurchasing} onPress={() => void handleBuy(product.productId)}>
          <Text>{product.title} · {product.localizedPrice}</Text>
        </Pressable>
      ))}
    </View>
  );
};
```

- O `purchase()` do SDK emite sozinho `checkout_started` e `transaction`. **Não rastreie a compra manualmente.**
- A venda só fica **verificada** pelo webhook da loja no servidor (seção 7.2). O SDK não valida recibo desde a 2.10.0, e `POST /sdk/purchases/validate` foi removido.
- Inclua o botão de restore na própria UI (seção 9, variante sem Superwall).
- **`useOfferings` na 2.10.x:** o `offeringService` não é inicializado pelo `init()`. Se precisar das offerings do dashboard, inicialize uma vez depois do SDK pronto:

```ts
import { useEffect } from "react";
import { offeringService, PaywalloClient, usePaywallo } from "@virex-tech/paywallo-sdk";

export const useOfferingServiceBootstrap = (): void => {
  const { isInitialized } = usePaywallo();

  useEffect(() => {
    if (!isInitialized) return;
    const apiClient = PaywalloClient.getApiClient();
    if (apiClient) offeringService.init({ apiClient, debug: __DEV__ });
  }, [isInitialized]);
};
```

  O tipo `Offering` real é `{ id, identifier, name, description, metadata, product: { productId, billingPeriod, trialDays, priceUsd, ... } }`. Os campos `introOffer*` citados no README do SDK **não existem** no tipo. Para preço exibido, use sempre o preço localizado de `useProducts`, não `priceUsd`.

### RevenueCat

Se o app **já** usa RevenueCat, não precisa trocar: o Paywallo importa compras, renovações e reembolsos via webhook `https://paywallo.com.br/api/webhook/revenuecat/{sua-app-key}` (header `Authorization` com o secret, docs `revenuecat`). Não há bridge de eventos de paywall para RevenueCat. Use o mesmo id no `Purchases.logIn(userId)` e no `identify({ userId })`. **Não** combine RevenueCat + Superwall + compra do SDK no mesmo app sem necessidade: escolha **um** caminho de compra.

---

## 11. Checklist por feature

Para cada ponto do app que libera conteúdo premium:

- [ ] Nenhuma chamada a API removida (`presentCampaign`, `presentPaywall`, `requireSubscriptionWithCampaign`, `gateContentWithCampaign`, `requireSubscription(placement)`, `gateContent`, `preload*`)
- [ ] Placement criado no **dashboard do Superwall**, com nome estável (`feature_x_unlock`) e paywall **Gated**
- [ ] Gate via `useSubscriptionGate(placement).unlock()` em handler (`onPress`), tratando `false` como "não liberar"
- [ ] `syncSuperwallAttributes({ timeoutMs: 1500 })` antes de **todo** `registerPlacement` (já incluso no hook)
- [ ] `PaywalloProvider` por fora do `SuperwallProvider`
- [ ] Mesmo `userId` no `identify` do Superwall e do Paywallo; `signOut()` + `reset()` no logout
- [ ] Webhook Superwall → Paywallo com header `x-paywallo-secret` e os 7 eventos; lojas conectadas
- [ ] Status premium via `hasActiveSubscription()` (não `useSubscription()`), invalidado após unlock/restore
- [ ] Restore no paywall (Superwall) **e** em Settings
- [ ] Sem tracking manual de `paywall`/`transaction`/`$paywall_*`
- [ ] Compra sandbox testada (iOS Sandbox + Google Play License Tester) e visível em **Transações** no Paywallo

---

## 12. Anti-patterns

| ❌ Não faça                                                              | ✅ Faça                                                                 |
| :----------------------------------------------------------------------- | :---------------------------------------------------------------------- |
| `await presentCampaign("x")` / `requireSubscriptionWithCampaign("x")`    | `useSubscriptionGate("x").unlock()` (Superwall)                         |
| `usePaywallo().presentCampaign(...)` "porque ainda compila"              | É stub: resolve `presented: false` e nada aparece                       |
| `registerPlacement` sem `syncSuperwallAttributes` antes                  | `await syncSuperwallAttributes({ timeoutMs: 1500 })` e depois register  |
| `track("paywall", ...)` / `track("transaction", ...)` manual             | Nada: bridge e webhook cuidam disso                                     |
| `SuperwallProvider` por fora do `PaywalloProvider`                       | `PaywalloProvider` > `SuperwallProvider`                                 |
| Ids diferentes no Superwall e no Paywallo                                | Mesmo `userId` nos dois `identify`                                      |
| Esquecer o webhook do Superwall ("o SDK já manda a compra")              | Webhook com `x-paywallo-secret` + os 7 eventos                          |
| `useSubscription().isActive` para liberar conteúdo (2.10.x)              | `usePaywallo().hasActiveSubscription()` (hook `usePremiumStatus`)        |
| Paywall _Non Gated_ usado como gate                                      | Marque _Gated_ no Superwall                                             |
| `registerPlacement` / restore em `useEffect` de abertura de tela         | Em handler de evento (`onPress`, fim do onboarding)                     |
| Múltiplos placements para variar preço                                   | 1 placement por superfície; variantes/audiences no Superwall            |
| RevenueCat + Superwall + `usePurchase` juntos "por garantia"             | Um único caminho de compra                                              |
| Restore com `react-native-iap` ou lib paralela                           | Restore do Superwall, ou `restorePurchases()` do SDK na UI própria      |

---

## 13. Próximos passos

- Tracking de funil antes do paywall: [`paywallo-funnel-tracking.md`](./paywallo-funnel-tracking.md)
- A/B test de variantes do paywall: [`paywall-ab-testing.md`](./paywall-ab-testing.md)
- Visão geral consolidada: [`paywallo-full-skill.md`](./paywallo-full-skill.md)
