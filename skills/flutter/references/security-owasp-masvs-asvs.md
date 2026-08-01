# Seguridad Mobile: OWASP MASVS, MASTG y ASVS aplicados a Flutter

## Cuándo usar este archivo

Antes de manejar tokens, credenciales, almacenamiento local, comunicación de red, biometría, o antes de una auditoría/pentest de la app. También al preparar la app para producción o para subirla a Play Store con cuenta de desarrollador.

## Marcos de referencia

- **OWASP MASVS** (Mobile Application Security Verification Standard): checklist de requisitos de seguridad para apps móviles, dividido en categorías MASVS-STORAGE, MASVS-CRYPTO, MASVS-AUTH, MASVS-NETWORK, MASVS-PLATFORM, MASVS-CODE, MASVS-RESILIENCE, MASVS-PRIVACY.
- **OWASP MASTG** (Mobile Application Security Testing Guide): cómo verificar cada requisito del MASVS en la práctica (con Frida, jadx, mitmproxy, etc.).
- **OWASP ASVS** (Application Security Verification Standard): equivalente para backends/APIs — relevante porque toda app Flutter con backend hereda estos requisitos en su API.
- **OWASP Mobile Top 10**: lista de las 10 categorías de vulnerabilidad móvil más frecuentes.

---

## MASVS-STORAGE: almacenamiento local

**Regla de oro**: nada sensible (tokens, contraseñas, PII, claves de API privadas) va en `SharedPreferences` ni en archivos planos. `SharedPreferences` en Android es un XML sin cifrar en `/data/data/<package>/shared_prefs/`; en un dispositivo rooteado es legible directamente.

| Qué guardar | Dónde | Paquete Flutter |
|---|---|---|
| Tokens de sesión, refresh tokens | Keystore (Android) / Keychain (iOS), cifrado | `flutter_secure_storage` |
| Datos de negocio no sensibles (preferencias de tema, último tab visto) | `SharedPreferences` | `shared_preferences` |
| Datos de negocio sensibles offline (ej. turnos con datos de sueldo) | Base local cifrada | `sqflite` + `sqlcipher_flutter_libs`, o `Hive` + `HiveAesCipher` |
| Caché de imágenes/archivos temporales | Directorio temporal, limpiar en logout | `path_provider` (`getTemporaryDirectory`) |

```dart
// Mal
final prefs = await SharedPreferences.getInstance();
await prefs.setString('auth_token', token);

// Bien
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
);
await storage.write(key: 'auth_token', value: token);
```

**Checklist MASVS-STORAGE**:
- [ ] Ningún `print()`/`log()` imprime tokens, contraseñas o PII en producción (usa `kDebugMode` para gatear logs, o mejor, un logger que redacta campos sensibles).
- [ ] `android:allowBackup="false"` en `AndroidManifest.xml` si los datos locales son sensibles (evita que un backup de ADB exfiltre la base de datos).
- [ ] Limpiar almacenamiento seguro y caché al hacer logout (`storage.deleteAll()`).
- [ ] Sin capturas de pantalla del contenido sensible en el selector de apps recientes: `FLAG_SECURE` en Android vía `flutter_windowmanager` si aplica (ej. si hay datos de sueldo visibles).

---

## MASVS-CRYPTO: criptografía

- Nunca implementes tu propio algoritmo de cifrado o hash. Usa `package:cryptography` o las APIs nativas expuestas por `flutter_secure_storage`.
- Hashing de contraseñas **no se hace en el cliente** salvo que el backend lo requiera explícitamente por diseño (lo normal es enviar la contraseña por TLS y que el backend haga `bcrypt`/`argon2`).
- Si generas identificadores o nonces, usa `Random.secure()`, nunca `Random()` (el default no es criptográficamente seguro).

```dart
// Mal: no es criptográficamente seguro
final id = Random().nextInt(999999).toString();

// Bien
final bytes = List<int>.generate(16, (_) => Random.secure().nextInt(256));
final id = base64UrlEncode(bytes);
```

**Hashing de contraseñas local (caso específico)**: si tu app necesita guardar localmente un hash de algo tipo-contraseña (ej. un PIN de acceso rápido a la app, no la contraseña de la cuenta), no uses `sha256` de `package:crypto` — es un hash *rápido*, diseñado para integridad de datos, no para contraseñas: eso lo hace vulnerable a fuerza bruta si el hash se filtra. Usa un algoritmo lento y salteado como SHA-512-crypt (`package:dart_crypt`) o Argon2/bcrypt vía un binding nativo.

```dart
// Mal: sha256 es rápido, no está diseñado para contraseñas
import 'package:crypto/crypto.dart';
final hash = sha256.convert(utf8.encode(pin)).toString();

// Bien: algoritmo lento y salteado, diseñado para credenciales
import 'package:dart_crypt/dart_crypt.dart';
final hashed = Crypt.sha512(pin);   // al crear/cambiar el PIN
final isValid = hashed.match(inputPin); // al verificar
```

---

## MASVS-AUTH: autenticación y gestión de sesión

- Usa un proveedor de identidad probado (Firebase Auth, Auth0, Cognito) en vez de implementar login propio — ver [identity-access-aaa-cia.md](identity-access-aaa-cia.md).
- **Expiración de sesión**: los tokens de acceso (access token) deben expirar en minutos/horas, no días; usa `refresh token` de vida más larga guardado en `flutter_secure_storage` para renovar sin re-login.
- **Biometría** (`local_auth`): solo como capa adicional de conveniencia sobre una sesión ya autenticada — nunca como único mecanismo de autorización contra el backend, porque la verificación biométrica ocurre 100% en el dispositivo y no prueba nada al servidor por sí sola. El patrón correcto es: biometría desbloquea localmente un token/clave ya almacenada cifrada.
- **Cierre de sesión real**: invalidar el refresh token en el backend al hacer logout, no solo borrarlo del cliente (si no, un token robado antes del logout sigue siendo válido indefinidamente).

---

## MASVS-NETWORK: seguridad de red {#network-security}

- **TLS obligatorio**: nunca `http://` en producción. En Android, `usesCleartextTraffic="false"` (default desde API 28) — no lo reactives salvo necesidad real documentada.
- **Certificate pinning** para apps con datos sensibles (financieras, salud, datos de nómina): usa `HttpClient` con `badCertificateCallback` personalizado o el paquete `http_certificate_pinning` / `dio` con interceptor de pinning. Mitiga MITM incluso si el atacante controla una CA confiada por el sistema (ej. proxy corporativo o CA maliciosa instalada por malware).
- **Validar certificados server-side**, no confiar en que el sistema operativo siempre tenga el store de CAs correcto (dispositivos rooteados a veces tienen CAs de usuario inyectadas para interceptar tráfico).

```dart
// Ejemplo de interceptor dio con pinning simplificado
final dio = Dio();
dio.httpClientAdapter = IOHttpClientAdapter(
  createHttpClient: () {
    final client = HttpClient();
    client.badCertificateCallback = (cert, host, port) {
      final expectedSha256 = 'AA:BB:...'; // fingerprint del cert esperado
      return _sha256Fingerprint(cert) == expectedSha256;
    };
    return client;
  },
);
```

---

## MASVS-PLATFORM: uso correcto de permisos y componentes

- **Principio de mínimo privilegio en permisos**: pide solo los permisos de `AndroidManifest.xml` que la app realmente usa (`CAMERA`, `ACCESS_FINE_LOCATION`, `POST_NOTIFICATIONS`, etc.). Google Play rechaza o marca apps con permisos no justificados en el listado de la Play Store.
- **Deep links**: valida el origen de cualquier `intent-filter` con `autoVerify="true"` (App Links verificados) en vez de esquemas custom sin verificar, que pueden ser interceptados por apps maliciosas.
- **WebView**: si usas `webview_flutter`, deshabilita JavaScript salvo que sea estrictamente necesario, y nunca cargues URLs no confiables/generadas por el usuario sin sanitizar (riesgo de XSS local con acceso a JS bridges).

---

## MASVS-CODE: calidad y empaquetado

- **Ofuscación en release**: `flutter build apk --obfuscate --split-debug-info=./debug-symbols` — dificulta ingeniería inversa del código Dart compilado a AOT.
- **R8/ProGuard** habilitado en Android (`minifyEnabled true` en `build.gradle`) para reducir superficie de análisis estático.
- **No hardcodear secretos** (API keys, client secrets) en el código Dart — terminan en el binario y son extraíbles con `strings`/reversing. Usa variables de entorno inyectadas en build time (`--dart-define`) para claves públicas, y **nunca** pongas secretos que requieran confidencialidad real en el cliente — esos van solo en el backend.

---

## MASVS-RESILIENCE: anti-tampering (para apps de alto riesgo)

Relevante si la app maneja pagos, datos médicos o es blanco de fraude (ej. apps bancarias, apuestas). Para una primera app de calendario de turnos, normalmente **no** es prioritario, pero documentado para cuando escales:

- Detección de root/jailbreak (`safe_device` o similar) como señal, no como bloqueo absoluto (falsos positivos existen).
- `package:freerasp` detecta root/jailbreak, debugger adjunto y repaquetado de la app en tiempo de ejecución en un solo paquete — más completo que combinar varios detectores sueltos.
- Play Integrity API (reemplazo de SafetyNet) para verificar que el APK no fue modificado y corre en un dispositivo certificado.

---

## MASVS-PRIVACY: privacidad y cumplimiento

- **Data minimization**: solo recolecta los datos que el feature necesita. Para un calendario de turnos, probablemente no necesitas ubicación GPS precisa, solo la ciudad/zona horaria del usuario.
- **Play Data Safety form**: al publicar en Play Store con tu cuenta de desarrollador, debes declarar exactamente qué datos recolecta la app y con qué fin — que coincida con la realidad del código es una verificación real que hace Google.
- Si guardas datos de terceros (ej. nombres de compañeros de turno), evalúa si aplica alguna ley de protección de datos local (en Chile, Ley 19.628 / futura Ley de Protección de Datos Personales).

---

## OWASP ASVS — cuando la app tiene backend propio

Si construyes tu propia API (Node/Firebase Functions/etc.) en vez de usar solo Firebase gestionado, el backend debe cumplir ASVS nivel 1 como mínimo:
- Validación de input server-side (nunca confíes solo en la validación del cliente Flutter — `formz` valida para UX, no para seguridad).
- Rate limiting en endpoints de login (mitiga fuerza bruta y credential stuffing).
- Autorización verificada en cada endpoint, no solo autenticación (un usuario autenticado no debería poder leer los turnos de otro usuario cambiando un ID en la URL — IDOR, ver [threat-modeling-stride-pasta-cwe.md](threat-modeling-stride-pasta-cwe.md)).

## Real-world usage

Para el calendario de turnos: tokens de Firebase Auth en `flutter_secure_storage`, `usesCleartextTraffic="false"`, permisos de `AndroidManifest.xml` limitados a `INTERNET` y `POST_NOTIFICATIONS` (para recordatorios de turno) — sin `ACCESS_FINE_LOCATION` salvo que agregues geofencing real. Reglas de Firestore server-side que verifican `request.auth.uid == resource.data.userId` en cada turno (autorización real, no solo ocultar el botón en la UI).
