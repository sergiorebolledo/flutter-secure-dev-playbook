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

Cada paquete en `pubspec.yaml` es una superficie de ataque potencial (supply chain). Prácticas mínimas:

- `dart pub outdated` regularmente para ver paquetes desactualizados.
- Revisar el **pub.dev score** y "Popularity" antes de agregar un paquete nuevo — paquetes con 1 mantenedor y sin actividad reciente son mayor riesgo.
- Fijar versiones (`^x.y.z` con cuidado) y revisar el changelog antes de actualizar mayor versión, no solo `flutter pub upgrade --major-versions` a ciegas.
- Vigilar advisories: GitHub Dependabot funciona sobre `pubspec.yaml`/`pubspec.lock` si el repo está en GitHub — actívalo en **Settings → Security → Dependabot**.

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
