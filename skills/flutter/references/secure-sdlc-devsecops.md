# Secure SDLC y DevSecOps para proyectos Flutter

## Cuándo usar este archivo

Al configurar CI/CD para un proyecto Flutter (GitHub Actions, Codemagic, Fastlane), al decidir qué controles automatizados agregar antes de mergear a `main`, o al preparar el pipeline de build/firma para producción.

---

## 1. SDLC vs SSDLC

Un SDLC tradicional (Planificación → Diseño → Desarrollo → Pruebas → Despliegue → Mantenimiento) trata la seguridad como una fase final ("pentest antes de lanzar"). Un **SSDLC (Secure SDLC)** integra controles de seguridad en cada fase — esto es lo que en la industria se llama **"shift left"**: mover la seguridad lo más temprano posible en el ciclo, porque un bug de seguridad encontrado en diseño cuesta órdenes de magnitud menos que uno encontrado en producción.

| Fase | Control de seguridad shift-left |
|---|---|
| Planificación | Threat modeling ligero (STRIDE, ver [threat-modeling-stride-pasta-cwe.md](threat-modeling-stride-pasta-cwe.md)) para features con datos sensibles |
| Diseño | Revisión de reglas de autorización (Firestore rules, endpoints) antes de escribir el código de UI |
| Desarrollo | Linters estrictos + pre-commit hooks; nunca commitear secretos |
| Pruebas | SAST automático en cada PR; SCA sobre `pubspec.lock` |
| Despliegue | Firma de release, ofuscación, revisión de permisos del manifest |
| Mantenimiento | Monitoreo de nuevas CVEs en dependencias, actualización periódica del SDK de Flutter |

---

## 2. DevSecOps: seguridad como código, no como gate manual

DevSecOps significa que los controles de seguridad son **automatizados y viven en el repo** (pipeline, config), no un checklist manual que alguien revisa una vez al mes.

### Pipeline mínimo recomendado (GitHub Actions) para un proyecto Flutter solo

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request, push]

jobs:
  analyze-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.44.8'
          channel: 'stable'
      - run: flutter pub get

      # SAST: análisis estático — detecta anti-patrones y algunos problemas de seguridad
      - run: flutter analyze --fatal-infos

      # Formato consistente (no es seguridad, pero mantiene el diff revisable)
      - run: dart format --output=none --set-exit-if-changed .

      # Tests unitarios/widget — evita regresiones en lógica de autorización local
      - run: flutter test --coverage

      # SCA: auditoría de dependencias vulnerables conocidas
      - run: dart pub outdated --mode=null-safety
      - run: dart pub deps --json > deps.json
```

### Orden de los quality gates (por qué importa el orden)

No corras todos los checks a la vez esperando que "todo pase": hazlo en este orden, porque cada gate hace que el siguiente sea significativo:

1. **Analyze** (`flutter analyze`) — un analizador en rojo puede hacer que los tests ni siquiera compilen; arregla esto primero.
2. **Format** (`dart format --set-exit-if-changed`) — corre después de analyze, para que el formateo no compita con los cambios que un fix de analyze introduce.
3. **Test** (`flutter test`) — solo tiene sentido evaluarlo si analyze está limpio.
4. **Coverage** — solo es significativo si los tests ya pasan; nunca calcules cobertura sobre una corrida con fallos de compilación.

**Cobertura**: un objetivo de 100% es alcanzable (no una utopía) si excluyes del denominador el código generado, no el código real:

```yaml
# ejemplo de exclude para genhtml/lcov
exclude_coverage: "**/*.{g,freezed,gen}.dart"   # código generado por build_runner
```

Perseguir 100% incluyendo archivos `.g.dart`/`.freezed.dart` generados automáticamente es una meta sin sentido — exclúyelos explícitamente y mide cobertura solo sobre el código que tú escribiste a mano.

### SAST (Static Application Security Testing)

- `flutter analyze` con `analysis_options.yaml` estricto es tu primera línea de SAST — activa reglas como `avoid_print`, `close_sinks`, `unsafe_html`.
- Para SAST más profundo orientado a seguridad, herramientas como **Semgrep** (con reglas de la comunidad para Dart) detectan patrones como uso de `Random()` no seguro o `badCertificateCallback` que ignora errores.

```yaml
# analysis_options.yaml — reglas relevantes para seguridad
linter:
  rules:
    - avoid_print                 # evita fugas de datos sensibles en logs
    - close_sinks                 # evita leaks de recursos (StreamController sin dispose)
    - unsafe_html                 # relevante en Flutter Web
    - avoid_web_libraries_in_flutter
```

### SCA (Software Composition Analysis)

Cada paquete en `pubspec.yaml` es una superficie de ataque potencial (supply chain) — el código de un paquete se compila directamente dentro de tu binario, así que una dependencia vulnerable o maliciosa afecta a todos tus usuarios en todas las plataformas (esto es OWASP Mobile Top 10, categoría M2: Inadequate Supply Chain Security).

**Detección de advisories conocidos**:

```bash
# dart pub get ya muestra advisories del GitHub Advisory Database al resolver:
# http 0.13.0 (affected by advisory: GHSA-4rgh-jx4f-xxxx, 1.2.0 available)

# osv-scanner escanea pubspec.lock contra la base de datos OSV (más CVEs que el Advisory Database)
osv-scanner --lockfile=pubspec.lock
```

Si decides ignorar un advisory (falso positivo confirmado, código afectado no usado), documenta por qué — nunca lo silencies sin justificación:

```yaml
# pubspec.yaml
ignored_advisories:
  - GHSA-4rgh-jx4f-xxxx # No aplica: no usamos el constructor de http.Client afectado
```

**Caso real para tener en mente**: el paquete `flutter_downloader` (con alta popularidad en pub.dev) tuvo vulnerabilidades de inyección SQL y escritura arbitraria de archivos antes de la versión 1.11.2, que permitían robar tokens de sesión en apps bancarias y gubernamentales que lo usaban. Popularidad alta no es sinónimo de seguro — revisa el historial de advisories antes de confiar en un paquete solo por su adopción.

**Señales de typosquatting** (paquetes maliciosos que imitan nombres populares) al agregar una dependencia nueva:
- El nombre difiere de un paquete conocido por un carácter, un guion/guion bajo intercambiado, o letras transpuestas (ej. `flutter-secure-storage` en vez de `flutter_secure_storage`).
- Sin "verified publisher" en pub.dev y pocos pub points, pero pidiendo permisos sensibles (red, cámara, Keychain/Keystore).
- La URL del repositorio declarado no coincide con el homepage del paquete.

**Permisos que se cuelan por dependencias transitivas**: revisa `AndroidManifest.xml` por permisos que tu código de primera mano no usa realmente — pueden venir fusionados silenciosamente por un paquete de terceros:

```bash
flutter pub deps --style=tree   # rastrea qué paquete introdujo un permiso inesperado
```

**Otras prácticas mínimas**:
- `dart pub outdated` regularmente para ver parches de seguridad no anunciados como tales en el changelog.
- Revisar el **pub.dev score** y "Popularity" antes de agregar un paquete nuevo — paquetes con 1 mantenedor y sin actividad reciente son mayor riesgo (independiente del check de typosquatting).
- Fijar versiones (`^x.y.z` con cuidado) y revisar el changelog antes de actualizar mayor versión, no solo `flutter pub upgrade --major-versions` a ciegas.
- GitHub Dependabot sobre `pubspec.yaml`/`pubspec.lock` — actívalo en **Settings → Security → Dependabot**.

**Cumplimiento de licencias** (relacionado, pero distinto de vulnerabilidades): ver [license-compliance.md](license-compliance.md) — un paquete con licencia copyleft fuerte (GPL) puede obligarte legalmente a liberar tu propio código, incluso si el paquete no tiene ninguna vulnerabilidad.

### DAST (Dynamic Application Security Testing)

DAST tradicional (escanear una app corriendo) aplica más a la **API/backend** que a la app Flutter en sí:
- Si tienes backend propio (Cloud Functions, Node), herramientas como OWASP ZAP pueden escanear los endpoints en un ambiente de staging.
- Para la app móvil, el equivalente es testing dinámico con MASTG (proxy con mitmproxy/Burp para inspeccionar tráfico real y confirmar que no hay texto plano ni certificate pinning roto).

---

## 3. Gestión de secretos

- **Nunca** commitear `google-services.json`, `GoogleService-Info.plist` con configuración de producción si el repo es público, o usar `.gitignore` + inyectar en CI desde secrets.
- `.gitignore` mínimo para un repo Flutter público:

```gitignore
# Secretos y config sensible
.env
**/google-services.json
**/GoogleService-Info.plist
android/key.properties
android/app/*.jks
ios/Runner/GoogleService-Info.plist

# Build
/build/
.dart_tool/
```

- Variables de entorno en build: `flutter build apk --dart-define=API_BASE_URL=https://api.example.com` o `--dart-define-from-file=env.json` (con `env.json` fuera de git). En GitHub Actions, usar `secrets.API_KEY` inyectado como variable de entorno del job.
- **Firma de release Android**: `key.properties` y el `.jks` **nunca** van al repo. En CI, decodificar el keystore desde un secret base64:

```yaml
- name: Decode keystore
  run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > android/app/release.jks
```

---

## 4. Gates de seguridad antes de merge

Reglas mínimas razonables para un repo de un solo desarrollador (evita que la "seguridad" se vuelva burocracia sin sentido):

- [ ] `flutter analyze` sin errores (bloquea merge vía CI required check).
- [ ] `flutter test` pasa al 100%.
- [ ] Ningún archivo de `.gitignore` (secretos) aparece en el diff del PR — revisar con `git diff --stat` antes de commitear.
- [ ] Si el PR toca reglas de Firestore/autorización: revisión manual explícita de la regla (son la parte más fácil de romper por accidente, ver [threat-modeling-stride-pasta-cwe.md](threat-modeling-stride-pasta-cwe.md)).

## Real-world usage

Para el calendario de turnos: pipeline de GitHub Actions corre `flutter analyze` + `flutter test` en cada PR; Dependabot activado sobre `pubspec.yaml`; `google-services.json` real vive solo en un secret de CI y en el `.gitignore` local; el keystore de firma de release se genera una vez, se guarda cifrado fuera del repo (ej. gestor de contraseñas) y se inyecta en CI como secret base64 al hacer un release hacia Play Store.
