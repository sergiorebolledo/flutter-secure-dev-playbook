# ASO, Marketing de Lanzamiento y Analítica

## Cuándo usar este archivo

Al preparar la ficha de Play Store antes del primer release, al planear cómo conseguir los primeros usuarios sin presupuesto de marketing, o al decidir qué analítica instrumentar desde el día 1.

---

## 1. ASO (App Store Optimization) — el "SEO" de las tiendas de apps

La ficha de Play Store es lo único que decide si alguien instala tu app o sigue de largo. Elementos que sí mueven la aguja:

| Elemento | Por qué importa | Consejo práctico |
|---|---|---|
| **Título de la app** | Se indexa para búsqueda | Incluye 1 palabra clave relevante si es natural (ej. "Turnos — Calendario de Trabajo", no relleno de keywords) |
| **Descripción corta** (80 caracteres) | Se muestra en resultados de búsqueda | La frase de valor más clara posible: qué hace y para quién |
| **Descripción larga** | Se indexa parcialmente, y convence al usuario que ya llegó a la ficha | Primeras 2-3 líneas son las más importantes (antes del "leer más") |
| **Ícono** | Primera impresión visual en la lista de resultados | Simple, reconocible en tamaño pequeño, sin texto denso |
| **Capturas de pantalla** | El mayor factor de conversión de ficha→instalación | Muestra el valor real de la app, no solo pantallas vacías; agrega texto corto superpuesto explicando el beneficio |
| **Video preview** (opcional) | Mejora conversión, especialmente en apps con interacción visual | 15-30 segundos mostrando el flujo principal |
| **Categoría correcta** | Afecta en qué listados/rankings apareces | Verifica la categoría más específica que aplique, no la más genérica |
| **Rating y reviews** | Señal de confianza y factor de ranking | Pide review en el momento correcto (ver sección 3), nunca con popups agresivos al abrir |

## 2. Checklist de lanzamiento (antes de publicar a producción)

- [ ] Ficha de Play Store completa (título, descripciones, capturas, ícono de alta resolución)
- [ ] **Data Safety form** completado con precisión (qué datos recolecta la app — debe coincidir con la realidad del código, ver [security-owasp-masvs-asvs.md](security-owasp-masvs-asvs.md#masvs-privacy-privacidad-y-cumplimiento))
- [ ] Política de privacidad publicada (URL pública, obligatoria incluso para apps simples si recolectas cualquier dato)
- [ ] Build de release firmado y verificado (`flutter build appbundle --obfuscate --split-debug-info=...`)
- [ ] Probado en **Internal Testing track** de Play Console con al menos 1 dispositivo real (no solo emulador)
- [ ] `SECURITY.md`/canal de reporte de bugs si el proyecto es open source (ver [production-security-ops.md](production-security-ops.md))
- [ ] Analítica básica instrumentada (sección 4) — no se puede medir el lanzamiento sin datos

## 3. Cómo conseguir los primeros usuarios sin presupuesto

Para una primera app sin budget de ads pagados:

- **Comunidades específicas del nicho**: para un calendario de turnos, foros/grupos de Facebook o subreddits de trabajadores por turnos (salud, retail, manufactura) — mucho más efectivo que publicidad genérica.
- **Pide el review en el momento correcto**: usa `in_app_review` (paquete oficial que invoca el diálogo nativo de rating) **después de un momento de éxito claro** (ej. tras crear su 3er turno exitosamente), nunca al abrir la app por primera vez.
- **Página de aterrizaje simple**: incluso una sola página con capturas + link a Play Store ayuda a compartir en redes sin depender solo del listado de la tienda.
- **Feedback loop temprano**: agrega un canal de feedback fácil (email o formulario) visible en la app — los primeros usuarios reales son tu mejor fuente de qué priorizar.

## 4. Analítica mínima viable (sin sobre-instrumentar)

No necesitas 50 eventos trackeados desde el día 1. Con Firebase Analytics (gratis, ya integrado si usas Firebase Auth/Firestore) es suficiente para empezar:

| Evento | Por qué |
|---|---|
| `app_open` (automático) | Retención básica |
| `sign_up` / `login` | Funnel de activación |
| Evento de "acción de valor" (ej. `shift_created`) | Mide si el usuario realmente usa el core de la app, no solo la abre |
| `screen_view` en pantallas clave | Dónde abandonan los usuarios el flujo |

```dart
await FirebaseAnalytics.instance.logEvent(
  name: 'shift_created',
  parameters: {'shift_type': shiftType},
);
```

**Privacidad al instrumentar analítica**: no envíes PII (nombre, email, datos de sueldo) como parámetro de evento — usa IDs anónimos/hasheados si necesitas correlacionar. Esto conecta directo con [security-owasp-masvs-asvs.md](security-owasp-masvs-asvs.md#masvs-privacy-privacidad-y-cumplimiento) (data minimization).

## 5. Iteración post-lanzamiento

- Revisa reviews negativas cada semana las primeras semanas — son la señal más barata de bugs/UX confuso que tienes.
- Trackea el **crash-free rate** en Crashlytics (ver [testing-methodologies-tdd-bdd-qa.md](testing-methodologies-tdd-bdd-qa.md)) — una caída notable después de un release es la primera señal de que algo salió mal.
- No cambies el ícono/nombre de la app con frecuencia — Play Store penaliza el ranking y confunde a usuarios existentes buscándola.

## Real-world usage

Para el calendario de turnos: ficha de Play Store con 5 capturas mostrando el calendario, la creación de un turno y las notificaciones; política de privacidad publicada en una página estática simple; `in_app_review` disparado tras el 3er turno creado exitosamente; evento `shift_created` en Firebase Analytics sin PII; lanzamiento inicial compartido en 2-3 grupos de Facebook de trabajadores por turnos antes de cualquier inversión en ads.
