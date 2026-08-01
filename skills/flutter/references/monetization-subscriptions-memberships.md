# Monetización con Suscripciones y Membresías en Flutter

## Cuándo usar este archivo

Al diseñar un modelo de negocio freemium/premium, al implementar el flujo de compra de una suscripción, o al decidir entre la API nativa de `in_app_purchase` y un servicio de terceros como RevenueCat.

Complementa a [in-app-purchase.md](in-app-purchase.md) (mecánica de compra a bajo nivel con `in_app_purchase`, ya en la skill base) con la capa de **estrategia de negocio y arquitectura de suscripciones**. Ver también [monetization-admob-ads.md](monetization-admob-ads.md) para el modelo híbrido freemium-con-anuncios.

---

## 1. Modelos de suscripción comunes

| Modelo | Cuándo aplica | Ejemplo |
|---|---|---|
| **Freemium con paywall de features** | La app es útil gratis, algunas features avanzadas son de pago | Calendario gratis; exportar a PDF, sincronización multi-dispositivo o recordatorios avanzados de pago |
| **Freemium con límite de uso** | El valor core es el mismo, pero limitado en cantidad | Máximo 1 calendario gratis, ilimitados en la versión paga |
| **Trial gratuito con conversión automática** | Se quiere que el usuario pruebe el valor completo antes de decidir | 7 días gratis de todas las features, luego cobra automáticamente salvo cancelación |
| **Solo-pago (sin free tier)** | Nicho específico donde el usuario ya sabe que quiere pagar | Apps B2B/profesionales especializadas |

Para una primera app, **freemium con paywall de features** suele ser la opción más simple de implementar y más fácil de justificar al usuario ("lo básico es gratis para siempre").

## 2. API nativa (`in_app_purchase`) vs. RevenueCat

| | `in_app_purchase` nativo | RevenueCat |
|---|---|---|
| Costo | Gratis (solo la comisión de Google/Apple: 15-30%) | Gratis hasta cierto volumen de ingresos, luego comisión adicional |
| Complejidad de implementación | Alta — debes manejar verificación de recibos, reconciliación entre plataformas, restauración de compras tú mismo | Baja — SDK unificado, dashboard de analíticas, webhooks listos |
| Verificación de recibos server-side | Debes construirla tú (Cloud Function que valida contra Google Play Developer API / App Store Server API) | Incluida — RevenueCat valida y expone el estado de "entitlements" vía API |
| Multiplataforma (iOS + Android + Web) | Requiere lógica separada por tienda | Unifica el estado de suscripción entre plataformas |

**Recomendación para un desarrollador solo en su primera app con monetización**: empieza con RevenueCat. La verificación de recibos server-side hecha a mano es una fuente significativa de bugs y de fraude si se hace mal (un cliente puede falsificar una respuesta de "compra exitosa" si solo confías en el cliente) — RevenueCat resuelve esto sin que tengas que escribir tu propio backend de validación.

## 3. Seguridad: nunca confíes en el cliente para el estado de la suscripción

Este es el error más costoso en monetización — ligado directamente a [threat-modeling-stride-pasta-cwe.md](threat-modeling-stride-pasta-cwe.md) (Tampering/CWE-284):

```dart
// Mal: el cliente decide si el usuario tiene acceso premium
class SubscriptionCubit extends Cubit<bool> {
  void unlockPremium() => emit(true); // cualquiera con acceso al binario puede forzar esto
}
```

```dart
// Bien: el estado de "premium" se verifica contra una fuente de verdad server-side
// (RevenueCat API, o tu propio backend que valida el recibo con Google/Apple)
final isPremium = await revenueCatCustomerInfo.entitlements.active.containsKey('premium');
```

Si construyes tu propio backend de suscripciones (sin RevenueCat), la regla de [identity-access-aaa-cia.md](identity-access-aaa-cia.md) aplica igual: el "entitlement" premium vive en una tabla/colección controlada solo por el backend tras verificar el recibo con Google/Apple — nunca como un campo que el cliente puede escribir directamente.

## 4. Diseño del paywall (UX)

- **Muestra el valor antes de pedir la tarjeta**: deja que el usuario use el free tier lo suficiente para sentir el valor antes de mostrar el paywall — un paywall inmediato al abrir la app por primera vez tiene conversión mucho más baja.
- **Precio y período claros, sin letra pequeña escondida**: el usuario debe saber exactamente cuánto y cuándo se le cobra antes de confirmar — además de ser buena práctica de UX, es requisito de política de Apple/Google.
- **Botón de cancelar/gestionar suscripción fácil de encontrar**: llévalo directo a la pantalla de gestión de suscripciones del sistema (`Uri.parse('https://play.google.com/store/account/subscriptions')` en Android, equivalente en iOS) — ocultar esto genera quejas y reportes a las tiendas.
- **Restaurar compras**: botón explícito "Restaurar compras" — obligatorio en iOS (requisito de App Store Review Guidelines) y buena práctica en Android para usuarios que cambian de dispositivo.

## 5. Políticas de las tiendas (obligatorio conocerlas antes de publicar)

- **Apple exige** usar exclusivamente el sistema de compras In-App de Apple para contenido digital consumido dentro de la app — no puedes redirigir a un pago externo (ej. Stripe directo) para desbloquear features dentro de la app en iOS.
- **Google es más flexible** en ciertas categorías, pero para la mayoría de apps de consumo aplica la misma regla vía Google Play Billing.
- Cambios de precio, período de prueba y política de cancelación deben reflejarse correctamente en la ficha de la tienda — inconsistencias entre lo mostrado en la app y lo declarado en la tienda son motivo de rechazo en revisión.

## 6. Métricas mínimas a trackear

No necesitas un dashboard de BI completo, pero sí estas métricas básicas desde el día 1 (RevenueCat las da automáticamente; si no, instrumenta manualmente):

- **Conversión de trial a pago**: cuántos de los que empiezan el trial terminan pagando.
- **Churn mensual**: cuántos suscriptores cancelan cada mes.
- **MRR (Monthly Recurring Revenue)**: ingreso recurrente mensual total.

## Real-world usage

Para el calendario de turnos: modelo freemium — calendario básico gratis para siempre, tier "Pro" (sincronización entre dispositivos, exportar a PDF, recordatorios avanzados) vía suscripción mensual/anual gestionada con RevenueCat; el estado `isPremium` se lee del `CustomerInfo` de RevenueCat, nunca de una variable local; el paywall se muestra solo al intentar usar una feature Pro específica (no al abrir la app), con botón de "Restaurar compras" visible en la pantalla de Perfil.
