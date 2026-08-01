# Accesibilidad (WCAG 2.2) aplicada a Flutter

## Cuándo usar este archivo

Al construir cualquier widget interactivo, antes de una publicación en Play Store (Google evalúa señales de accesibilidad), o si algún usuario reporta dificultad usando la app con TalkBack/VoiceOver.

**Por qué importa incluso en una app pequeña**: accesibilidad no es "curro extra" — un contraste insuficiente afecta a cualquier usuario con luz solar directa en la pantalla, no solo a personas con baja visión. Los controles de esta guía mejoran la app para todos.

---

## 1. Niveles de conformidad WCAG 2.2

- **A**: elimina las barreras más críticas.
- **AA**: el estándar que la mayoría de regulaciones exige — objetivo razonable por defecto para cualquier app pública.
- **AAA**: exhaustivo, raro de alcanzar en toda la app; se aplica a flujos puntuales de alto impacto si acaso.

Para una primera app, apunta a **AA** en las pantallas principales — es alcanzable sin sobre-invertir.

---

## 2. Checklist por categoría (con el ID real del criterio WCAG)

### Semántica y lectores de pantalla

| Criterio | Regla | Código |
|---|---|---|
| 1.1.1 Non-text Content | Toda imagen con significado necesita `semanticLabel`; las decorativas usan `excludeFromSemantics: true` | `Image.asset('warn.png', semanticLabel: 'Advertencia de solapamiento')` |
| 4.1.2 Name, Role, Value | Botones de solo ícono necesitan `Tooltip` o `Semantics(label:)` — el lector de pantalla no tiene otra forma de saber su propósito | `IconButton(icon: Icon(Icons.delete), tooltip: 'Eliminar turno', onPressed: ...)` |
| 4.1.3 Status Messages | Cambios de estado asíncronos visibles deben anunciarse | `Semantics(liveRegion: true, child: Text('Turno guardado'))` |

```dart
// Mal: sin label, un GestureDetector no es alcanzable por teclado/switch access
GestureDetector(onTap: _delete, child: Icon(Icons.delete));

// Bien: widget interactivo real + label explícito
IconButton(
  icon: const Icon(Icons.delete),
  tooltip: 'Eliminar turno',
  onPressed: _delete,
);
```

**Nunca uses `GestureDetector` desnudo para un tap target** (criterio 2.1.1 Keyboard) — es solo-puntero, inalcanzable por teclado o switch access. Usa `InkWell`, `ElevatedButton`, `TextButton`, o `IconButton`.

### Tamaño de objetivos táctiles

| Criterio | Regla |
|---|---|
| 2.5.8 Target Size (Minimum, AA) | Mínimo 24x24 dp — hallazgos entre 24 y 48 dp son aceptables en AA pero se recomienda evitarlos |
| 2.5.5 Target Size (Enhanced, AAA) | Mínimo 44x44 dp |

Recomendación práctica (más estricta que el mínimo AA): apunta a **48x48 dp** para cualquier botón/ícono tocable — es el estándar de facto de Material Design y evita fallos por poco margen en dispositivos con pantallas densas.

### Contraste de color

| Criterio | Umbral |
|---|---|
| 1.4.3 Contrast (Minimum, AA) | Texto normal 4.5:1, texto grande 3:1 |
| 1.4.11 Non-text Contrast (AA) | Componentes de UI e indicadores de foco 3:1 |
| 1.4.1 Use of Color | El color nunca puede ser el único diferenciador — combínalo con un ícono, texto o forma |

```dart
// Mal: un turno "vencido" solo se distingue por el color rojo del texto
Text('Turno', style: TextStyle(color: isOverdue ? Colors.red : Colors.black));

// Bien: color + ícono + texto — accesible para daltonismo/bajo contraste
Row(children: [
  if (isOverdue) const Icon(Icons.warning, color: Colors.red, size: 16),
  Text(isOverdue ? 'Turno vencido' : 'Turno', style: TextStyle(color: isOverdue ? Colors.red : Colors.black)),
]);
```

Verifica el contraste real de tu `ColorScheme` con una herramienta (ej. el contrast checker de WebAIM) contra los colores de fondo/texto reales, no a ojo.

### Escalado de texto

| Criterio | Regla |
|---|---|
| 1.4.4 Resize Text (AA) | El texto debe escalar a 200% (300% en iOS con Larger Accessibility Sizes) sin perder contenido |

```dart
// Mal: contenedor de altura fija que recorta el texto al escalar la fuente del sistema
Container(height: 40, child: const Text('Turno de la mañana'));

// Bien: sin altura fija, o con minHeight — el contenedor crece con el texto
Container(constraints: const BoxConstraints(minHeight: 40), child: const Text('Turno de la mañana'));
```

Prueba tu app real con la configuración de accesibilidad del sistema puesta al máximo (Ajustes → Accesibilidad → Tamaño de fuente) — es el chequeo de 2 minutos que más bugs de este tipo revela.

### Animación y movimiento

| Criterio | Regla |
|---|---|
| 2.3.3 Animation from Interactions (AAA, buena práctica igual en AA) | Toda animación debe respetar la preferencia del sistema de "reducir movimiento" |

```dart
final reduceMotion = MediaQuery.of(context).disableAnimations;
AnimatedContainer(
  duration: reduceMotion ? Duration.zero : const Duration(milliseconds: 300),
  // ...
);
```

### Formularios

| Criterio | Regla |
|---|---|
| 1.3.5 Identify Input Purpose (AA) | Campos de datos personales estructurados necesitan `autofillHints` |
| 3.3.1 / 3.3.2 Error Identification / Labels | Errores identificados en texto (no solo color), todo campo con label visible |
| 3.3.8 Accessible Authentication (AA) | No exigir una prueba cognitiva sin alternativa; permitir pegar contraseñas, soportar gestores de contraseñas |

```dart
TextField(
  autofillHints: const [AutofillHints.email],
  keyboardType: TextInputType.emailAddress,
  decoration: const InputDecoration(labelText: 'Correo electrónico'),
);
```

**Anti-patrón común de autenticación**: deshabilitar "pegar" en el campo de contraseña "por seguridad" — en realidad empeora la seguridad, porque empuja a los usuarios a contraseñas más débiles y fáciles de recordar/escribir en vez de usar un gestor de contraseñas. Nunca lo hagas.

---

## 3. Widgets Cupertino: semántica más débil por defecto

Si usas widgets Cupertino (`CupertinoSwitch`, `CupertinoSlider`, `CupertinoSegmentedControl`), sus valores por defecto de accesibilidad son más pobres que sus equivalentes Material — envuélvelos explícitamente:

```dart
Semantics(
  label: 'Notificaciones de turno',
  value: _enabled ? 'activado' : 'desactivado',
  child: CupertinoSwitch(value: _enabled, onChanged: _onChanged),
);
```

## 4. Chequeo rápido de 10 minutos antes de cada release

1. Activa TalkBack (Android) o VoiceOver (iOS) y navega el flujo principal completo con los ojos cerrados.
2. Sube el tamaño de fuente del sistema al máximo y revisa que nada se recorte.
3. Revisa que cada `IconButton` tenga `tooltip`.
4. Revisa contraste de los 2-3 colores de texto más usados contra su fondo.
5. Confirma que ningún flujo dependa solo del color para transmitir información.

## Real-world usage

Para el calendario de turnos: cada turno vencido se marca con ícono + color (no solo color); el botón "Crear turno" mide 48x48dp mínimo; el campo de email en el login usa `autofillHints: [AutofillHints.email]`; las animaciones de transición entre vistas respetan `MediaQuery.disableAnimations`; antes de cada release a Play Store se corre el chequeo de 10 minutos de arriba con TalkBack activado en un dispositivo real.
