# Principios de Diseño UX/UI para Apps Flutter

## Cuándo usar este archivo

Al diseñar una pantalla nueva antes de escribir código, al decidir la jerarquía visual de una feature, o al revisar por qué una pantalla "se siente mal" aunque funcione correctamente. Complementa [theming-material3.md](theming-material3.md) (que cubre la implementación técnica de `ThemeData`) con las decisiones de diseño *antes* de llegar al código.

---

## 1. Las 10 heurísticas de Nielsen, aplicadas a Flutter

Marco clásico de usabilidad — cada heurística con su chequeo concreto en una app móvil:

| Heurística | Pregunta guía | Ejemplo en Flutter |
|---|---|---|
| Visibilidad del estado del sistema | ¿El usuario sabe qué está pasando? | `CircularProgressIndicator` mientras carga, no una pantalla congelada |
| Coincidencia con el mundo real | ¿El lenguaje es el del usuario, no el técnico? | "Turno de mañana" no "shift_type: MORNING" |
| Control y libertad del usuario | ¿Hay forma de deshacer/cancelar? | Botón "Deshacer" en un `SnackBar` tras borrar un turno |
| Consistencia y estándares | ¿Se comporta como el resto de apps de la plataforma? | Gestos de swipe-back en iOS, botón atrás de Android respetado |
| Prevención de errores | ¿Se puede evitar el error antes de que ocurra? | Deshabilitar el botón "Guardar" si el formulario es inválido, en vez de mostrar error después |
| Reconocer antes que recordar | ¿La opción está visible o hay que memorizarla? | Mostrar los turnos recientes en vez de exigir escribir la fecha de memoria |
| Flexibilidad y eficiencia de uso | ¿Hay atajos para usuarios frecuentes? | Duplicar el turno de la semana anterior con un botón |
| Diseño estético y minimalista | ¿Cada elemento en pantalla es necesario? | No mostrar 5 botones de acción cuando el usuario solo usa 1 |
| Ayudar a reconocer y recuperarse de errores | ¿El mensaje de error dice qué hacer? | "El turno se solapa con uno existente a las 14:00" no "Error 400" |
| Ayuda y documentación | ¿Hay ayuda contextual si se necesita? | Tooltip explicando qué significa un ícono poco obvio |

---

## 2. Arquitectura de la información (IA)

Antes de diseñar pantallas individuales, define la estructura de navegación completa:

- **Card sorting mental**: lista todas las pantallas que necesitas (Login, Calendario, Detalle de turno, Crear turno, Perfil, Notificaciones...) y agrúpalas por relación, no por orden de desarrollo.
- **Profundidad de navegación**: en móvil, prioriza pocas capas (2-3 niveles) sobre menús anidados profundos. Un calendario de turnos probablemente necesita: `Tab bar (Calendario | Notificaciones | Perfil)` → `Detalle de turno` → `Editar turno`, sin ir más profundo.
- **Navegación primaria**: `BottomNavigationBar`/`NavigationBar` (Material 3) para 3-5 destinos de nivel superior; evita un `Drawer` como navegación primaria en apps con pocas secciones — agrega un tap extra sin necesidad.

## 3. Design tokens: sistema de espaciado y tipografía

Además del `ColorScheme` que ya cubre [theming-material3.md](theming-material3.md), define una escala de espaciado consistente — evita "espaciado mágico" (`padding: 13.5`) esparcido por el código:

```dart
// lib/core/theme/app_spacing.dart
abstract final class AppSpacing {
  static const xxs = 4.0;
  static const xs = 8.0;
  static const sm = 12.0;
  static const md = 16.0;  // unidad base
  static const lg = 24.0;
  static const xl = 32.0;
  static const xxl = 48.0;
}

// Uso: EdgeInsets.symmetric/only — nunca EdgeInsets.fromLTRB con números sueltos
Padding(padding: const EdgeInsets.symmetric(horizontal: AppSpacing.md, vertical: AppSpacing.sm))
```

Misma lógica para tipografía: una escala nombrada (`displayLarge`, `headlineMedium`, `bodyLarge`...) integrada en `ThemeData.textTheme`, accedida vía `Theme.of(context).textTheme.bodyLarge`, nunca `TextStyle(fontSize: 15)` suelto en un widget.

**Por qué importa incluso solo**: sin estos tokens, cambiar el espaciado base de la app significa buscar y reemplazar números mágicos en decenas de archivos. Con tokens, es un cambio en un solo lugar.

---

## 4. Patrones de estado de pantalla (más allá de "cargando/error/éxito")

Toda pantalla con datos async necesita diseñar, no solo el happy path, sino:

| Estado | Qué mostrar | Error común |
|---|---|---|
| **Loading inicial** | Skeleton/shimmer o spinner centrado | Pantalla en blanco sin indicador |
| **Loading de refresco** | Indicador sutil (no bloquea la lista ya cargada) | Reemplazar toda la pantalla por un spinner al hacer pull-to-refresh |
| **Vacío (empty state)** | Ilustración/mensaje + acción clara ("Aún no tienes turnos. Crea el primero.") | Lista vacía sin ningún mensaje — el usuario piensa que es un bug |
| **Error** | Mensaje específico + botón de reintentar | "Ha ocurrido un error" genérico sin acción de recuperación |
| **Éxito con contenido** | El diseño "normal" | — |

El **empty state** es el más comúnmente olvidado — para una primera app, es literalmente el estado que todo usuario nuevo ve primero (antes de crear su primer turno).

## 5. Onboarding: cuándo SÍ y cuándo NO

- Para una app de utilidad simple (como un calendario de turnos), **evita un onboarding de múltiples pantallas explicando features** — es fricción que el usuario abandona. Prefiere un empty state accionable ("Crea tu primer turno") que enseña haciendo.
- Onboarding explícito (tour guiado, permisos explicados) solo se justifica cuando: (a) hay que pedir un permiso sensible (notificaciones, ubicación) y necesitas explicar el "por qué" antes del prompt nativo del sistema, o (b) el concepto central de la app no es obvio por sí solo.
- **Pedir el permiso de notificaciones con contexto primero**: en vez de pedirlo apenas abre la app (alta tasa de rechazo), muestra una pantalla propia explicando "te avisaremos 1h antes de cada turno" con un botón que dispara el prompt nativo — mejora medible en tasas de aceptación.

## 6. Jerarquía visual y contraste de acción

- Un máximo de **una acción primaria** por pantalla (botón `FilledButton`/`ElevatedButton`); acciones secundarias como `OutlinedButton`/`TextButton`. Dos botones "iguales" en la misma pantalla confunden al usuario sobre cuál es la acción esperada.
- Usa tamaño, peso de fuente y color — no solo color — para indicar importancia (esto también es un requisito de accesibilidad, ver [accessibility-wcag.md](accessibility-wcag.md)).

## Real-world usage

Para el calendario de turnos: navegación con `NavigationBar` de 3 destinos (Calendario, Notificaciones, Perfil); `AppSpacing`/`AppTextStyle` centralizados desde el día 1; el estado vacío del calendario muestra "Aún no tienes turnos esta semana" con un botón `FilledButton` "Crear turno" (única acción primaria visible); el permiso de notificaciones se pide después de mostrar una pantalla propia explicando el recordatorio de turno, no al abrir la app por primera vez.
