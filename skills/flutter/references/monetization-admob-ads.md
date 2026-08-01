# Monetización con Publicidad: Google AdMob en Flutter

## Cuándo usar este archivo

Al decidir monetizar una app gratuita con anuncios, al integrar `google_mobile_ads`, o al preparar el cumplimiento de consentimiento (GDPR/UMP) antes de publicar en Play Store.

Complementa a [monetization-subscriptions-memberships.md](monetization-subscriptions-memberships.md) — ads y suscripciones no son mutuamente excluyentes (modelo freemium: anuncios en el tier gratuito, sin anuncios en el tier pago).

---

## 1. Paquete y formatos de anuncio

`google_mobile_ads` (paquete oficial de Google) soporta varios formatos — elige según UX, no solo por potencial de ingreso:

| Formato | Cuándo usarlo | Impacto en UX |
|---|---|---|
| **Banner** | Espacio persistente en pantallas de bajo compromiso (ej. debajo de una lista) | Bajo, pero ingreso por impresión también es bajo |
| **Interstitial** | Entre transiciones de contenido naturales (ej. después de guardar un turno, no en medio de llenar un formulario) | Alto si se muestra en mal momento — puede sentirse como que "castiga" una acción |
| **Rewarded** | Usuario elige verlo a cambio de un beneficio (ej. "ver anuncio para desbloquear el tema oscuro") | El mejor formato en percepción de UX — es opt-in |
| **Native** | Se integra visualmente en el diseño de una lista/feed | Requiere más trabajo de diseño para no ser confuso con contenido real |

**Regla de oro de UX**: nunca un interstitial inmediatamente al abrir la app o justo antes/después de una acción crítica (ej. justo al crear un turno) — sentirse "penalizado" por usar la app es la razón #1 de desinstalación por ads mal implementados.

## 2. Setup básico

```yaml
# pubspec.yaml
dependencies:
  google_mobile_ads: ^5.0.0   # revisar última versión en pub.dev
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
  <application>
    <meta-data
        android:name="com.google.android.gms.ads.APPLICATION_ID"
        android:value="ca-app-pub-XXXXXXXXXXXXXXXX~YYYYYYYYYY"/>
  </application>
</manifest>
```

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await MobileAds.instance.initialize();
  runApp(const MyApp());
}
```

**Usa IDs de prueba durante todo el desarrollo** — Google suspende cuentas por clics accidentales/simulados en anuncios reales durante testing:

```dart
// ID de banner de prueba oficial (Android) — nunca el ID real en desarrollo
const testBannerId = 'ca-app-pub-3940256099942544/6300978111';
```

## 3. Consentimiento (UMP / GDPR) — obligatorio, no opcional

Si tu app puede tener usuarios en el Espacio Económico Europeo/Reino Unido (Play Store es global por defecto salvo que restrinjas países), **debes** mostrar un formulario de consentimiento antes de cargar anuncios personalizados. Google provee el **User Messaging Platform (UMP) SDK** para esto — es parte del mismo paquete `google_mobile_ads`.

```dart
Future<void> _requestConsent() async {
  final params = ConsentRequestParameters();
  ConsentInformation.instance.requestConsentInfoUpdate(
    params,
    () async {
      if (await ConsentInformation.instance.isConsentFormAvailable()) {
        await _loadAndShowConsentForm();
      }
      await MobileAds.instance.initialize(); // solo después de resolver consentimiento
    },
    (error) {/* maneja el error, no bloquees la app por esto */},
  );
}

Future<void> _loadAndShowConsentForm() async {
  ConsentForm.loadConsentForm(
    (form) async {
      final status = await ConsentInformation.instance.getConsentStatus();
      if (status == ConsentStatus.required) {
        form.show((formError) {/* ... */});
      }
    },
    (formError) {/* ... */},
  );
}
```

**No mostrar el formulario cuando corresponde es una violación de política de Google/Play Store** que puede resultar en suspensión de la cuenta de AdMob — no es un detalle opcional de cumplimiento.

## 4. Cumplimiento con Play Store

- Declara el uso de publicidad en el **Data Safety form** de Play Console (qué datos recolecta el SDK de anuncios y con qué fin).
- **Apps dirigidas a niños** (Designed for Families) tienen reglas mucho más estrictas — sin anuncios personalizados, formatos limitados. Si tu app no es para niños, declara correctamente el rango de edad objetivo.
- Un anuncio no puede imitar la UI del sistema operativo ni de tu propia app de forma que confunda al usuario sobre qué es contenido real vs. publicidad (Google exige distinción clara, generalmente con la etiqueta "Anuncio"/"Ad").

## 5. Placement recomendado para una app de utilidad (no-juego)

Para una app como un calendario de turnos (no un juego con "niveles" naturales para interstitials):

- **Banner discreto** en la pantalla de lista de turnos (parte inferior, no interrumpe el scroll de contenido).
- **Nunca** en la pantalla de crear/editar turno (formularios necesitan foco sin distracción).
- Considera **rewarded** para features opcionales premium-lite (ej. "ver anuncio para exportar tu calendario a PDF") en vez de interstitials forzados — mejor percepción y de forma similar a un tier freemium sin necesidad de un sistema de suscripción completo.

## Real-world usage

Para el calendario de turnos: banner discreto en la lista de turnos (oculto para usuarios con suscripción activa, ver [monetization-subscriptions-memberships.md](monetization-subscriptions-memberships.md)); UMP SDK integrado y probado con la cuenta de Google Ads configurada en modo debug antes de cualquier build de producción; IDs de anuncio reales solo en `--dart-define` de builds de release, IDs de prueba en debug por defecto (evita el riesgo de clics accidentales durante desarrollo).
