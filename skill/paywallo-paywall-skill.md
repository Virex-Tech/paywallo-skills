# Skill: Paywallo SDK — Apresentação de Paywalls & Gate de Conteúdo

> **Pré-requisito:** [`paywallo-sdk-setup.md`](./paywallo-sdk-setup.md) já aplicado — Provider montado e `identify` plugado.
> **Doc oficial:** https://paywallo.com.br/docs/present-paywalls

Esta skill cobre como apresentar paywalls, verificar status de assinatura e restaurar compras usando os hooks do SDK. **O SDK renderiza a UI do paywall** — o app só dispara a chamada do hook.

> ⚠️ **Não rastreie eventos `$paywall_viewed`, `$paywall_purchased` manualmente.** O SDK emite esses eventos automaticamente. Tracking manual gera duplicação no funil do dashboard.

---

## 1. Os hooks principais

| Hook                | O que faz                                                      | Retorno                                |
| :------------------ | :------------------------------------------------------------- | :------------------------------------- |
| `usePaywallo()`     | Controlador de paywall — apresenta por ID ou por campanha       | `{ presentPaywall, presentCampaign }`  |
| `useSubscription()` | Status reativo da assinatura                                    | `{ isActive }`                         |
| `usePurchase()`     | Restaurar compras                                               | `{ restore, isRestoring }`             |

---

## 2. Apresentar paywall

### Por ID

Use quando você quer um paywall específico (criado no dashboard com aquele ID fixo):

```tsx
import { usePaywallo } from "@virex-tech/paywallo-sdk";

const paywallo = usePaywallo();
const result = await paywallo.presentPaywall("paywall_onboarding");
```

### Por campanha (recomendado)

Use **placement** em vez de paywall ID. O dashboard associa o placement a uma campanha e decide qual paywall mostrar — assim você troca o paywall (ou faz A/B test) sem release:

```tsx
const result = await paywallo.presentCampaign("placement_hard_paywall");
```

---

## 3. Tratando o resultado

```tsx
import { usePaywallo } from "@virex-tech/paywallo-sdk";
import { useNavigation } from "@react-navigation/native";

function SettingsScreen() {
  const paywallo = usePaywallo();
  const navigation = useNavigation();

  async function handleSubscribe() {
    const result = await paywallo.presentPaywall("plans");

    if (result.purchased) {
      // compra confirmada, navega pra tela premium
      navigation.navigate("PremiumHome");
    } else if (result.skippedReason === "subscriber") {
      // já era assinante, vai direto
      navigation.navigate("PremiumHome");
    }
    // se nenhum dos dois: user cancelou (fechou o paywall) — não faz nada
  }

  return (
    <TouchableOpacity onPress={handleSubscribe}>
      <Text>Ver planos</Text>
    </TouchableOpacity>
  );
}
```

### Campos do `result`

- `purchased: boolean` — `true` quando a compra foi confirmada e validada
- `skippedReason?: "subscriber"` — SDK detectou que o user já é assinante e pulou a apresentação

Se ambos forem falsy, o user cancelou (fechou o paywall sem comprar).

---

## 4. `forceShow` — apresentar mesmo para assinantes

Em telas tipo "ver planos" ou "gerenciar assinatura" você quer mostrar o paywall **mesmo** se o user já tem sub. Use `forceShow: true`:

```tsx
const result = await paywallo.presentCampaign("plans", { forceShow: true });
```

Sem `forceShow`, o SDK pula o paywall para assinantes e retorna `skippedReason: "subscriber"`.

---

## 5. Callbacks inline

Alternativa ao `await result` — útil pra reagir a eventos específicos sem montar lógica em cima do retorno:

```tsx
await paywallo.presentPaywall("onboarding", {
  onDismiss: () => console.log("fechou"),
  onPurchase: (productId) => analytics.track("purchase", { productId }),
  onError: (err) => reportError(err),
});
```

> Os callbacks **não substituem** o `await` — você ainda recebe o `result`. Use callbacks para instrumentação; use `await` para controle de fluxo.

---

## 6. Verificar assinatura (sem apresentar paywall)

Use `useSubscription()` em screens. É reativo: o SDK ouve transações nativas (StoreKit 2 / Google Play Billing) e atualiza sozinho quando uma compra, renovação ou refund acontece.

```tsx
import { useSubscription } from "@virex-tech/paywallo-sdk";

function Screen() {
  const { isActive } = useSubscription();

  if (isActive) {
    return <PremiumContent />;
  }
  return <FreeContent />;
}
```

> Não precisa polling, `useEffect` de refresh nem orquestração manual. Só consuma `isActive`.

---

## 7. Restaurar compras

Botão **obrigatório** nas lojas (Apple Guideline 3.1.1). Adicione em paywall custom ou em Settings:

```tsx
import { usePurchase } from "@virex-tech/paywallo-sdk";

function RestoreButton() {
  const { restore, isRestoring } = usePurchase();

  async function handleRestore() {
    const purchases = await restore();
    if (purchases.length > 0) {
      // assinatura restaurada — useSubscription() reflete automaticamente
    } else {
      // nada pra restaurar
    }
  }

  return (
    <TouchableOpacity onPress={handleRestore} disabled={isRestoring}>
      <Text>Restaurar compras</Text>
    </TouchableOpacity>
  );
}
```

> O paywall remoto do Paywallo já tem botão de restore embutido (configurável no dashboard). Só implemente manualmente se for usar UI 100% custom.

---

## 8. Placement vs Paywall ID — qual usar?

| Conceito       | O que é                                                       | Quando usar                                              |
| :------------- | :------------------------------------------------------------ | :------------------------------------------------------- |
| **Paywall ID** | Identificador fixo de um paywall específico no dashboard       | Você quer aquele paywall, sempre, sem variação           |
| **Placement**  | String que identifica **onde** no app o paywall é apresentado | Default — permite trocar paywall e fazer A/B sem release |

**Regra:** prefira placement. Crie um por superfície de UI distinta (`onboarding_end`, `home_unlock`, `feature_x_gate`). O dashboard cuida das variantes.

---

## 9. Checklist por feature

Para cada local que apresenta paywall:

- [ ] Placement criado no dashboard com nome estável (`feature_x_unlock`)
- [ ] Hook próprio do app (ex: `useFeatureXGate` em `features/featureX/hooks/`)
- [ ] Tratamento de `result.purchased` e `result.skippedReason === "subscriber"`
- [ ] Sem tracking manual de `$paywall_*` (SDK rastreia automaticamente)
- [ ] Botão de "Restaurar compras" em Settings (Apple Guideline 3.1.1)
- [ ] Testado em Sandbox iOS + Google Play License Tester antes do release

---

## 10. Anti-patterns

| ❌ Não faça                                                  | ✅ Faça                                          |
| :----------------------------------------------------------- | :----------------------------------------------- |
| `track("$paywall_viewed", ...)` antes de `presentCampaign`   | Nada — SDK emite automaticamente                 |
| `presentCampaign` em `useEffect` direto na render            | Em handler de evento (`onPress`, `onComplete`)   |
| Múltiplos placements para variar preço                       | 1 placement, A/B test no dashboard               |
| Implementar restore com `react-native-iap`                   | `usePurchase().restore()` do SDK                 |
| Tratar o retorno como booleano (`if (await presentX())`)     | `const r = await ...; if (r.purchased) ...`      |

---

## 11. Próximos passos

- Tracking de funil antes do paywall: [`paywallo-funnel-tracking.md`](./paywallo-funnel-tracking.md)
- A/B test de variantes do paywall: [`paywall-ab-testing.md`](./paywall-ab-testing.md)
- Visão geral consolidada: [`paywallo-full-skill.md`](./paywallo-full-skill.md)
