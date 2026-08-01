# Checklist de Code Review y Principio de "Evidencia antes que Afirmación"

## Cuándo usar este archivo

Antes de mergear tu propio PR (como desarrollador único, tú eres tu propio revisor), al preparar una feature grande para revisión, o como referencia rápida de qué mirar en cualquier cambio de código Flutter.

---

## 1. El principio central: nunca afirmes "funciona" sin evidencia fresca

La causa más común de bugs en producción no es no saber programar — es afirmar que algo "debería funcionar" o "se ve bien" sin haber corrido la verificación real en ese momento. Antes de decir "listo", "arreglado" o "funciona":

```
REGLA: si no corriste el comando de verificación EN ESTE momento, no puedes afirmar el resultado.
```

| Afirmación | Requiere haber corrido | No es suficiente |
|---|---|---|
| "El código está limpio" | `flutter analyze` → "No issues found!" | "Lo escribí bien" |
| "Los tests pasan" | `flutter test` → todos en verde | "Debería pasar" |
| "El build funciona" | `flutter build apk/ios/web` exitoso | "Analyze pasó" (analyze ≠ build) |
| "El bug está arreglado" | Reproducir el síntoma original + confirmar que ya no ocurre | "Cambié el código" |
| "El widget funciona" | Widget test o verificación manual real | "El código se ve correcto" |

**Frases de alerta** (si te sorprendes pensando/escribiendo esto, detente y corre el comando): *"debería funcionar ahora"*, *"probablemente pasa"*, *"se ve bien"*, *"confío en que está bien"*. Ninguna de estas es evidencia.

---

## 2. Checklist de arquitectura (Clean Architecture)

**Capa Domain:**
- [ ] Entidades son clases Dart puras (sin `import 'package:flutter/...'`)
- [ ] Interfaces de repositorio son `abstract interface class`
- [ ] Sin imports desde `data/` o `presentation/` (la dependencia va en un solo sentido)

**Capa Data:**
- [ ] Modelos/DTOs tienen `fromJson`/`toJson` (o equivalente) probados
- [ ] Manejo de errores mapea a fallos de dominio, no deja excepciones crudas de HTTP escapar hacia arriba
- [ ] Implementaciones de repositorio cumplen realmente el contrato de la interfaz (sin `UnimplementedError` en métodos usados)

**Capa Presentation:**
- [ ] Sin lógica de negocio dentro de un `Widget` — solo reacciona a `State`
- [ ] Estados de carga y error manejados explícitamente en la UI (no solo el "happy path")
- [ ] `Cubit`/`Bloc` depende de interfaces (`Repository` abstracto), no de implementaciones concretas

## 3. Checklist de calidad Dart/Flutter

- [ ] `flutter analyze` sin issues
- [ ] `const` usado donde aplica (reduce rebuilds)
- [ ] `StatelessWidget` preferido sobre `StatefulWidget` cuando no hay estado mutable real
- [ ] Sin árboles de widgets anidados más de ~5 niveles sin extraer a un widget nombrado
- [ ] `dispose()` limpia todo lo que se abrió (`StreamController`, `AnimationController`, listeners)

## 4. Checklist de seguridad (triage por severidad)

Al revisar un PR, clasifica cualquier hallazgo de seguridad en una de estas 3 categorías — te dice qué bloquea el merge y qué no:

| Severidad | Ejemplos | ¿Bloquea merge? |
|---|---|---|
| **Crítico** | API key hardcodeada; token en `SharedPreferences` sin cifrar; `badCertificateCallback` que ignora el error; dato sensible en un log | Sí, siempre |
| **Advertencia** | Falta certificate pinning en endpoint de auth; `Random()` usado para un ID de sesión; falta validación `formz` antes de llamar a la API; `android:allowBackup="true"` con datos sensibles | Sí, salvo justificación explícita documentada |
| **Nota** | Falta ofuscación en build de release; `dart pub outdated` muestra parches disponibles; dependencia transitiva con permisos amplios y pocos pub points | No bloquea, pero se registra como deuda técnica |

Ver [security-owasp-masvs-asvs.md](security-owasp-masvs-asvs.md) y [threat-modeling-stride-pasta-cwe.md](threat-modeling-stride-pasta-cwe.md) para el detalle de cada categoría.

## 5. Checklist de tests (por prioridad, no "todo o nada")

No todos los tests valen lo mismo — prioriza en este orden si el tiempo es limitado:

1. **Repositorio/DataSource**: la lógica que toca red o almacenamiento — aquí viven los bugs más costosos.
2. **Cubit/Bloc**: transiciones de estado, casos borde, estados de error.
3. **Widget** (opcional pero recomendado): interacciones críticas del usuario.
4. **Golden tests** (opcional): solo si hay regresiones visuales que realmente importen.

## 6. Formato de reporte de revisión (útil incluso auto-revisándote)

```markdown
## Resumen de Code Review

### Qué está bien
[reconoce lo que funciona antes de listar problemas]

### Cumplimiento de arquitectura
- Domain: ✅/❌ [detalle]
- Data: ✅/❌ [detalle]
- Presentation: ✅/❌ [detalle]

### Hallazgos
#### Crítico
[archivo:línea — descripción]
#### Advertencia
[archivo:línea — descripción]
#### Nota
[archivo:línea — descripción]

### Verificación (con evidencia real, no supuesta)
- `flutter analyze`: [resultado exacto pegado]
- `flutter test`: [resultado exacto pegado, ej. "34/34 passed"]

### Veredicto
**¿Listo para mergear?** Sí / No / Con correcciones
```

## Real-world usage

Para el calendario de turnos: antes de cada merge a `main`, se pega en la descripción del PR (o en un comentario propio si trabajas solo) el bloque de "Verificación" de arriba con la salida real de `flutter analyze` y `flutter test` — nunca se escribe "tests pasan" de memoria. Un hallazgo de severidad Crítico (ej. un token quedó en un `print()` de debug) bloquea el merge hasta corregirse; uno de severidad Nota (falta `--obfuscate` en un build de debug local) se anota pero no bloquea.
