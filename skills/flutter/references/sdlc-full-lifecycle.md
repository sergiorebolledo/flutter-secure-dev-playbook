# El Ciclo de Vida Completo: de la Idea a la App en Producción (App y Web)

## Cuándo usar este archivo

Al empezar un proyecto nuevo (mapea qué archivo de este repositorio leer en cada etapa), o cuando no sabes "qué sigue" en el desarrollo. Es el archivo capstone que amarra todos los demás — cada fase enlaza al archivo específico correspondiente.

Este ciclo aplica tanto a una **app móvil** (Android/iOS vía Flutter) como a un **proyecto web** (Flutter Web u otro framework) — las fases son las mismas; lo que cambia es el canal de distribución y algunos detalles marcados explícitamente como **[Solo App]** o **[Solo Web]**.

---

## Las 7 fases del SDLC, aplicadas

```
1. Planificación → 2. Análisis → 3. Diseño → 4. Implementación → 5. Pruebas → 6. Despliegue → 7. Mantenimiento
                                        ↑___________________________________________________________|
                                                    (el ciclo se repite en cada nueva versión)
```

### 1. Planificación

**Qué se decide**: qué problema resuelve la app, para quién, y el alcance del MVP (versión mínima viable) — no todas las features del día 1.

- Define el objetivo de negocio en una frase: "Ayudar a trabajadores por turnos a no perder de vista su horario."
- Lista funcionalidades y sepáralas en MVP vs. futuro — resiste la tentación de construir todo antes de tener un solo usuario real.
- Define métricas de éxito desde ahora (ver [aso-marketing-launch.md](aso-marketing-launch.md) sección 4) — sin esto, no sabrás si el lanzamiento funcionó.

**[Solo App] vs [Solo Web]**: una app requiere considerar desde ya si target es Android, iOS o ambos (afecta cuenta de desarrollador, costos, tiempos de revisión). Un proyecto web no tiene revisión de tienda, pero sí necesitas decidir hosting/dominio desde esta fase.

### 2. Análisis (requisitos + amenazas)

- Requisitos funcionales: qué debe hacer la app (historias de usuario simples).
- Requisitos no funcionales: rendimiento esperado, offline-first o no, cuántos usuarios concurrentes.
- **Threat modeling ligero** para cualquier feature que toque datos sensibles — ver [threat-modeling-stride-pasta-cwe.md](threat-modeling-stride-pasta-cwe.md). Hacerlo aquí, no después de programar, es literalmente el significado de "shift-left" (ver [secure-sdlc-devsecops.md](secure-sdlc-devsecops.md)).
- Decide el modelo de monetización si aplica, ahora — afecta arquitectura (ver [monetization-admob-ads.md](monetization-admob-ads.md) / [monetization-subscriptions-memberships.md](monetization-subscriptions-memberships.md)).

### 3. Diseño

Dos sub-fases en paralelo:

**Diseño de producto/UX** — antes de tocar código:
- Arquitectura de la información y flujos de pantalla — ver [ux-design-principles.md](ux-design-principles.md).
- Accesibilidad considerada desde el diseño, no agregada después — ver [accessibility-wcag.md](accessibility-wcag.md).
- Sistema de diseño (colores, tipografía, espaciado) — ver [theming-material3.md](theming-material3.md) y la sección de design tokens en [ux-design-principles.md](ux-design-principles.md).

**Diseño técnico**:
- Arquitectura de capas (Clean Architecture, dónde va cada responsabilidad) — ver [architecture-principles-solid-clean.md](architecture-principles-solid-clean.md) y [architecture-conventions.md](architecture-conventions.md).
- Modelo de datos y elección de backend (Firebase gestionado vs. backend propio).
- Reglas de autorización diseñadas junto con el modelo de datos, no después — ver [identity-access-aaa-cia.md](identity-access-aaa-cia.md).

### 4. Implementación

- Elección de paquetes evaluada por licencia y seguridad antes de agregarlos — ver [license-compliance.md](license-compliance.md) y la sección SCA de [secure-sdlc-devsecops.md](secure-sdlc-devsecops.md).
- Sigue las convenciones de estado/navegación/networking de la skill base ([state-management.md](state-management.md), [networking-rest.md](networking-rest.md), etc.).
- Almacenamiento y comunicación seguros desde la primera línea, no como "hardening" al final — ver [security-owasp-masvs-asvs.md](security-owasp-masvs-asvs.md).
- Revisión continua contra el checklist de [code-review-checklist.md](code-review-checklist.md) en cada PR/commit propio.

**[Solo Web]**: agrega consideraciones de SEO técnico (meta tags, renderizado, sitemap) desde esta fase — no es post-lanzamiento, se diseña en el routing.

### 5. Pruebas

- TDD para lógica de negocio core, BDD como documentación de comportamiento — ver [testing-methodologies-tdd-bdd-qa.md](testing-methodologies-tdd-bdd-qa.md).
- Mecánica de tests (unit/widget/golden) — ver [testing.md](testing.md).
- Gates de CI en orden correcto (analyze → format → test → coverage) — ver [secure-sdlc-devsecops.md](secure-sdlc-devsecops.md).
- Auditoría de accesibilidad con lector de pantalla real antes de considerar "listo" — ver [accessibility-wcag.md](accessibility-wcag.md) sección 4.
- Verificación con evidencia real, nunca "debería funcionar" — ver el principio central de [code-review-checklist.md](code-review-checklist.md).

### 6. Despliegue

- Checklist de release: firma, ofuscación, ficha de tienda/página web — ver [aso-marketing-launch.md](aso-marketing-launch.md) sección 2.
- Pipeline de CI/CD con gestión segura de secretos y keystore — ver [secure-sdlc-devsecops.md](secure-sdlc-devsecops.md).

**[Solo App]**: Internal Testing track de Play Console antes de producción; tiempos de revisión de la tienda (horas a días) a considerar en la planificación de fechas.
**[Solo Web]**: despliegue puede ser continuo (cada merge a `main` publica), sin espera de revisión externa — pero requiere sus propios gates de calidad más estrictos porque no hay "red de seguridad" de una revisión humana externa antes de llegar a usuarios.

### 7. Mantenimiento

- Monitoreo de errores y disponibilidad — ver [production-security-ops.md](production-security-ops.md).
- Backups y plan de continuidad — ver [resilience-bcp-drp.md](resilience-bcp-drp.md).
- Actualización de dependencias y vigilancia de nuevos CVEs — ver la sección SCA de [secure-sdlc-devsecops.md](secure-sdlc-devsecops.md).
- Iteración basada en reviews/analítica real, no en suposiciones — ver [aso-marketing-launch.md](aso-marketing-launch.md) sección 5.
- El ciclo vuelve a la fase 1 (Planificación) para la siguiente versión — cada iteración es más rápida porque la base (arquitectura, seguridad, CI) ya existe.

---

## Diferencias clave: App móvil vs. Proyecto Web

| Aspecto | App (Android/iOS) | Web |
|---|---|---|
| Distribución | Play Store / App Store, con revisión humana/automatizada previa | Deploy directo a un servidor/CDN, sin revisión externa |
| Descubribilidad | ASO (ficha de tienda) | SEO (motores de búsqueda) |
| Actualización | El usuario debe actualizar (o auto-update si lo permite) | Instantánea para todos los usuarios en el próximo request |
| Monetización con ads/IAP | Sujeta a políticas estrictas de Google/Apple (comisión de tienda) | Más libertad (Stripe directo, sin comisión de plataforma) — pero sin la confianza implícita de "está en la tienda oficial" |
| Permisos de dispositivo | Modelo de permisos del SO (cámara, ubicación, notificaciones) | Modelo de permisos del navegador, generalmente más limitado |
| Accesibilidad | TalkBack/VoiceOver | Lectores de pantalla de escritorio (NVDA, JAWS) + los mismos principios WCAG |

## Real-world usage

Para el calendario de turnos (app Android, con Flutter Web como posible expansión futura): el proyecto siguió las 7 fases en orden — planificación definió el MVP (crear/ver turnos, notificaciones) dejando sincronización multi-dispositivo para v2; threat modeling en fase de análisis identificó que los datos de turnos de otros empleados necesitan autorización server-side desde el diseño de las reglas de Firestore, no como parche posterior; el pipeline de CI (fase de implementación/pruebas) se configuró antes de escribir la primera pantalla, no al final.
