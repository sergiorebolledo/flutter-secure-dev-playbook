# 🚀 Flutter Secure Dev Playbook

Guía de referencia exhaustiva y práctica para el desarrollo de aplicaciones móviles (y web) con **Flutter** y **Dart**, empaquetada como **skill/plugin instalable para [Claude Code](https://claude.com/claude-code)**. Compila estándares de arquitectura, patrones de Flutter, diseño UX/accesibilidad, marcos de ciberseguridad, y monetización/marketing — desde la idea hasta la app publicada — en un solo repositorio.

Este proyecto es un **fork extendido** de [Arcturus91/claude-flutter-skill](https://github.com/Arcturus91/claude-flutter-skill) (base de 19 archivos de referencia sobre Flutter/Dart), al que se agregó una capa completa de **Secure SDLC / DevSecOps / arquitectura**, con contenido original nuevo — informado además por conceptos revisados en [VeryGoodOpenSource/vgv-ai-flutter-plugin](https://github.com/VeryGoodOpenSource/vgv-ai-flutter-plugin) (seguridad estática específica de paquetes, cumplimiento de licencias, gates de calidad) y [vp-k/flutter-craft](https://github.com/vp-k/flutter-craft) (checklist de revisión de arquitectura, principio de verificación antes de afirmar completado). El texto es original — no se copió contenido literal de ninguno de los tres, solo los temas/conceptos que cubren. Se mantiene la licencia MIT original y el aviso de copyright del autor base.

---

## Instalación (como plugin de Claude Code)

```text
/plugin marketplace add sergiorebolledo/flutter-secure-dev-playbook
/plugin install flutter-secure-dev-playbook@flutter-secure-dev-playbook
```

Actualizar más adelante:

```text
/plugin marketplace update flutter-secure-dev-playbook
/plugin update flutter-secure-dev-playbook@flutter-secure-dev-playbook
```

### Alternativa: instalar como skill personal (sin plugin)

```bash
git clone https://github.com/sergiorebolledo/flutter-secure-dev-playbook
cp -R flutter-secure-dev-playbook/skills/flutter ~/.claude/skills/flutter
```

---

Este no es un repositorio solo de seguridad: cubre el ciclo completo — **diseño UX, desarrollo, arquitectura, seguridad, testing, monetización y marketing** — desde la idea hasta la app publicada.

## 📌 Índice de Contenidos

1. [Desarrollo Flutter/Dart (base)](#1-desarrollo-flutterdart-base)
2. [Principios de Diseño y Código Limpio (Clean Code)](#2-principios-de-diseño-y-código-limpio-clean-code)
3. [Gestión de Estado y Patrones en Flutter](#3-gestión-de-estado-y-patrones-en-flutter)
4. [Diseño UX/UI y Accesibilidad](#4-diseño-uxui-y-accesibilidad)
5. [Ciberseguridad en el Ciclo de Desarrollo (Shift Left)](#5-ciberseguridad-en-el-ciclo-de-desarrollo-shift-left)
6. [Seguridad y Operaciones en Producción](#6-seguridad-y-operaciones-en-producción)
7. [Ciclo de Vida, DevOps y Automatización](#7-ciclo-de-vida-devops-y-automatización)
8. [Calidad de Software y Metodologías de Pruebas](#8-calidad-de-software-y-metodologías-de-pruebas)
9. [Resiliencia Operativa y Continuidad](#9-resiliencia-operativa-y-continuidad)
10. [Monetización: Publicidad y Suscripciones](#10-monetización-publicidad-y-suscripciones)
11. [ASO, Marketing de Lanzamiento y Analítica](#11-aso-marketing-de-lanzamiento-y-analítica)
12. [El Ciclo de Vida Completo (SDLC), App y Web](#12-el-ciclo-de-vida-completo-sdlc-app-y-web)

---

## 1. Desarrollo Flutter/Dart (base)

Heredado de la skill original, es un **reference skill**: `SKILL.md` enruta a Claude al archivo correcto; solo se carga el archivo necesario para la tarea actual.

| Área | Archivo |
|---|---|
| Dart esencial | `skills/flutter/references/dart-essentials.md` |
| Async / streams / isolates | `skills/flutter/references/async-streams-isolates.md` |
| Widgets & layout | `skills/flutter/references/widgets-and-layout.md` |
| Gestión de estado (BLoC/Cubit) | `skills/flutter/references/state-management.md` |
| Navegación (flow_builder, go_router) | `skills/flutter/references/navigation-and-routing.md` |
| Formularios (formz) | `skills/flutter/references/forms-and-input.md` |
| Theming & Material 3 | `skills/flutter/references/theming-material3.md` |
| Networking & REST | `skills/flutter/references/networking-rest.md` |
| WebSockets & realtime | `skills/flutter/references/websockets-realtime.md` |
| Firebase core & auth | `skills/flutter/references/firebase-core-auth.md` |
| FCM & notificaciones locales | `skills/flutter/references/firebase-messaging-fcm.md` |
| Mapas & ubicación | `skills/flutter/references/maps-and-location.md` |
| Media, archivos & assets | `skills/flutter/references/media-files-assets.md` |
| Compras in-app | `skills/flutter/references/in-app-purchase.md` |
| Animaciones | `skills/flutter/references/animations.md` |
| Testing (mecánica) | `skills/flutter/references/testing.md` |
| Performance & DevTools | `skills/flutter/references/performance-and-devtools.md` |
| Tooling, build & deploy | `skills/flutter/references/tooling-build-deploy.md` |
| Convenciones de arquitectura del proyecto | `skills/flutter/references/architecture-conventions.md` |

## 2. Principios de Diseño y Código Limpio (Clean Code)

📄 [`skills/flutter/references/architecture-principles-solid-clean.md`](skills/flutter/references/architecture-principles-solid-clean.md)

* **SOLID** aplicado con ejemplos reales en Dart (inyección de dependencias en un `Cubit`, interfaces mínimas).
* **DRY, KISS, YAGNI**: cuándo compiten entre sí y regla práctica de "tolera 2 repeticiones, extrae en la 3ª".
* **DDD**: qué partes valen la pena en una app de un solo desarrollador (Value Objects, lenguaje ubicuo) y cuáles no (bounded contexts completos).
* **Clean Architecture**: capas `data/domain/presentation`, cuándo omitir la capa de casos de uso en apps pequeñas.
* **GRASP**: Information Expert, Creator, Controller, Low Coupling/High Cohesion, Protected Variations.
* **MVVM vs BLoC/Cubit**: por qué no son excluyentes.

## 3. Gestión de Estado y Patrones en Flutter

* **BLoC/Cubit**: separación UI/lógica mediante eventos y estados — ver `skills/flutter/references/state-management.md`.
* **MVVM**: mapeo de conceptos a BLoC/Cubit — ver la sección correspondiente en `architecture-principles-solid-clean.md`.
* **DI (Dependency Injection)**: constructor injection manual vs. `get_it`/`injectable` según escala del proyecto.

## 4. Diseño UX/UI y Accesibilidad

| Archivo | Contenido |
|---|---|
| [`skills/flutter/references/ux-design-principles.md`](skills/flutter/references/ux-design-principles.md) | Heurísticas de Nielsen, arquitectura de la información, design tokens (espaciado/tipografía), estados de pantalla (loading/empty/error), onboarding |
| [`skills/flutter/references/accessibility-wcag.md`](skills/flutter/references/accessibility-wcag.md) | WCAG 2.2 aplicado a Flutter con IDs de criterio reales — semántica, tamaño de objetivos táctiles, contraste, escalado de texto, autenticación accesible |

## 5. Ciberseguridad en el Ciclo de Desarrollo (Shift Left)

| Marco | Archivo |
| :--- | :--- |
| **MASVS / MASTG / ASVS** (OWASP) — almacenamiento, cripto, auth, red, permisos, ofuscación, privacidad | [`skills/flutter/references/security-owasp-masvs-asvs.md`](skills/flutter/references/security-owasp-masvs-asvs.md) |
| **STRIDE** — modelo de amenazas por componente | [`skills/flutter/references/threat-modeling-stride-pasta-cwe.md`](skills/flutter/references/threat-modeling-stride-pasta-cwe.md) |
| **PASTA** — modelado de ataques centrado en riesgo de negocio | mismo archivo, sección 2 |
| **CWE** — catálogo de debilidades mapeadas a errores comunes de Dart/Flutter | mismo archivo, sección 3 |
| **CIA** — Confidencialidad, Integridad, Disponibilidad | [`skills/flutter/references/identity-access-aaa-cia.md`](skills/flutter/references/identity-access-aaa-cia.md) |
| **AAA** — Autenticación, Autorización, Auditoría (Accounting) | mismo archivo, con ejemplos reales de reglas de Firestore y Custom Claims |

Cada archivo incluye ejemplos de código Dart/Firestore reales (no solo teoría) y una tabla de checklist accionable.

## 6. Seguridad y Operaciones en Producción

📄 [`skills/flutter/references/production-security-ops.md`](skills/flutter/references/production-security-ops.md)

Cubre WAF/WAAP, RASP, IAM/PAM, MFA, SIEM/SOC y un **Incident Response Plan (IRP) mínimo viable**, con el equivalente realista de cada control enterprise para un proyecto de un solo desarrollador (ej. Firebase App Check como sustituto accesible de un WAF completo).

## 7. Ciclo de Vida, DevOps y Automatización

📄 [`skills/flutter/references/secure-sdlc-devsecops.md`](skills/flutter/references/secure-sdlc-devsecops.md)

* **SDLC / SSDLC**: shift-left de seguridad por fase.
* **DevSecOps**: pipeline de GitHub Actions completo con `flutter analyze`, tests, y gestión de secretos.
* **SAST / DAST / SCA**: herramientas y reglas de lint concretas para Dart.
* Gestión de `.gitignore`, firma de release y keystores en CI.

## 8. Calidad de Software y Metodologías de Pruebas

📄 [`skills/flutter/references/testing-methodologies-tdd-bdd-qa.md`](skills/flutter/references/testing-methodologies-tdd-bdd-qa.md)

* **TDD**: ciclo Red/Green/Refactor con ejemplo real (`Shift.overlapsWith()`).
* **BDD**: escenarios Gherkin `Given/When/Then` como documentación viva.
* **QA**: checklist de PR y beta interna en Google Play Console.
* **KPIs**: code coverage, crash-free rate, deuda técnica — con advertencia sobre coverage inflado.

**Adicionales**:

| Archivo | Contenido |
|---|---|
| [`skills/flutter/references/license-compliance.md`](skills/flutter/references/license-compliance.md) | Auditoría de licencias de dependencias (permisiva vs. copyleft) — obligación legal, no solo vulnerabilidad |
| [`skills/flutter/references/code-review-checklist.md`](skills/flutter/references/code-review-checklist.md) | Checklist de revisión de arquitectura + triage de severidad de seguridad + principio de "evidencia antes que afirmación" |

Este propio repositorio incluye un [`SECURITY.md`](SECURITY.md) como ejemplo de política de divulgación de vulnerabilidades aplicada a sí mismo.

## 9. Resiliencia Operativa y Continuidad

📄 [`skills/flutter/references/resilience-bcp-drp.md`](skills/flutter/references/resilience-bcp-drp.md)

* **RTO/RPO**: valores realistas para una app pequeña, no cifras de infraestructura enterprise copiadas sin sentido.
* **BCP**: modo offline-first con persistencia de Firestore.
* **DRP**: exportaciones automáticas de backups y recuperación de cuentas de infraestructura (Firebase/Play Console).

## 10. Monetización: Publicidad y Suscripciones

| Archivo | Contenido |
|---|---|
| [`skills/flutter/references/monetization-admob-ads.md`](skills/flutter/references/monetization-admob-ads.md) | Formatos de AdMob (banner/interstitial/rewarded/native), consentimiento UMP/GDPR, política de Play Store, placement UX |
| [`skills/flutter/references/monetization-subscriptions-memberships.md`](skills/flutter/references/monetization-subscriptions-memberships.md) | Modelos freemium, `in_app_purchase` nativo vs. RevenueCat, seguridad del estado de suscripción (nunca confiar en el cliente), diseño de paywall, políticas de tienda |

## 11. ASO, Marketing de Lanzamiento y Analítica

📄 [`skills/flutter/references/aso-marketing-launch.md`](skills/flutter/references/aso-marketing-launch.md)

* **ASO**: qué elementos de la ficha de Play Store realmente mueven la conversión.
* **Checklist de lanzamiento**: de la ficha de tienda a la política de privacidad.
* **Adquisición sin presupuesto**: comunidades de nicho, momento correcto para pedir reviews.
* **Analítica mínima viable**: eventos esenciales sin sobre-instrumentar, sin filtrar PII.

## 12. El Ciclo de Vida Completo (SDLC), App y Web

📄 [`skills/flutter/references/sdlc-full-lifecycle.md`](skills/flutter/references/sdlc-full-lifecycle.md)

Archivo **capstone** que amarra los 34 archivos restantes en las 7 fases clásicas del SDLC (Planificación → Análisis → Diseño → Implementación → Pruebas → Despliegue → Mantenimiento), señalando en cada fase qué archivo leer. Aplica tanto a una app móvil como a un proyecto web (Flutter Web u otro), con una tabla explícita de diferencias entre ambos canales (distribución, descubribilidad, actualización, monetización, permisos, accesibilidad).

---

## Cómo funciona (integración con Claude Code)

`SKILL.md` describe cuándo la skill es relevante; Claude Code carga solo el/los archivo(s) de referencia necesarios para la tarea — mantiene el uso de contexto liviano incluso con 35 archivos de referencia disponibles.

## Filosofía: opinionado pero transferible

Los ejemplos reflejan un stack de producción real (Cubit/BLoC + `equatable` + `formz`, Firebase auth + FCM, `flutter_map`) **más** un enfoque de seguridad aplicable a cualquier app Flutter con datos de usuario, no solo a la app de ejemplo (un calendario de turnos). Adapta los patrones; los marcos de seguridad (OWASP, STRIDE, CWE) son estándares de industria, no específicos de este proyecto.

## Contribuir

Issues y PRs bienvenidos. Mantén los ejemplos de código verificables contra versiones de paquete reales y cita la fuente oficial (OWASP, CWE, Microsoft STRIDE) cuando agregues un nuevo control.

## Créditos

- Base de referencia Flutter/Dart (fork): [Arturo Barrantes Vásquez](https://github.com/Arcturus91) — [claude-flutter-skill](https://github.com/Arcturus91/claude-flutter-skill) (MIT).
- Temas de seguridad estática por paquete, cumplimiento de licencias y gates de calidad — conceptos revisados en [VeryGoodOpenSource/vgv-ai-flutter-plugin](https://github.com/VeryGoodOpenSource/vgv-ai-flutter-plugin) (Very Good Ventures), reescritos de forma original.
- Checklist de revisión de arquitectura y principio de verificación antes de afirmar completado — conceptos revisados en [vp-k/flutter-craft](https://github.com/vp-k/flutter-craft), reescritos de forma original.
- Capa de arquitectura y seguridad (SOLID, OWASP MASVS/ASVS, STRIDE/PASTA, CWE, DevSecOps, AAA/CIA, resiliencia, licencias, code review): [Sergio Rebolledo López](https://github.com/sergiorebolledo).

## Licencia

[MIT](LICENSE) © Arturo Barrantes Vasquez — contribuciones adicionales © Sergio Rebolledo López
