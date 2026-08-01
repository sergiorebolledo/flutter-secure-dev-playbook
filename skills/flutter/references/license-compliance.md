# Cumplimiento de Licencias de Dependencias

## Cuándo usar este archivo

Antes de publicar un repo público, antes de agregar un paquete nuevo a `pubspec.yaml` que hace algo "demasiado conveniente" (parece resolver mucho trabajo con poco código — vale la pena revisar su licencia), o al preparar una app para uso comercial/Play Store.

Complementa el apartado de SCA en [secure-sdlc-devsecops.md](secure-sdlc-devsecops.md): SCA busca *vulnerabilidades*, este archivo cubre *obligaciones legales* — un paquete puede estar libre de CVEs y aun así crear una obligación legal si su licencia es incompatible con tu proyecto.

---

## Por qué importa incluso en un proyecto personal

Si tu app es de código cerrado (la subes a Play Store sin publicar el código fuente) y usa sin saberlo un paquete con licencia **copyleft fuerte** (GPL), técnicamente estás obligado a liberar el código fuente completo de tu app bajo esa misma licencia si distribuyes binarios que la incluyen. La mayoría de los desarrolladores nunca lo revisan porque casi todo el ecosistema pub.dev es permisivo (MIT/BSD/Apache) — pero vale la pena el chequeo de 5 minutos antes de publicar.

## Categorías de licencia

| Categoría | Licencias comunes | Riesgo | Qué significa para ti |
|---|---|---|---|
| **Permisiva** | MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0 | Bajo | Puedes usarla en código cerrado sin obligación de publicar tu código |
| **Copyleft débil** | LGPL-2.1, LGPL-3.0, MPL-2.0 | Medio | Segura si solo *enlazas* la librería (uso normal de un paquete pub); revisar si la modificas directamente |
| **Copyleft fuerte** | GPL-2.0, GPL-3.0, AGPL-3.0 | Alto | Puede obligarte a liberar el código fuente completo de tu app bajo la misma licencia |
| **Desconocida/Ausente** | Sin licencia declarada | Alto | Ausencia de licencia significa "todos los derechos reservados" por defecto — **no** significa "de uso libre"; trátalo como no autorizado hasta confirmar |

## Cómo revisar

```bash
# Ver las licencias declaradas por cada paquete usado (incluye transitivas)
flutter pub deps --style=compact

# En el propio pubspec.lock puedes ver la fuente de cada paquete;
# la licencia real se revisa en la página de pub.dev de cada uno:
# https://pub.dev/packages/<nombre> → pestaña "License"
```

No existe (a agosto de 2026) un comando oficial de `flutter`/`dart` que audite licencias automáticamente para todo el árbol de dependencias — la verificación manual vía pub.dev sigue siendo necesaria para paquetes nuevos o poco comunes. Si tu flujo de CI lo requiere, herramientas de terceros (ej. `license_checker` en pub.dev) pueden automatizar parte de esto.

## Proceso de auditoría (antes de cada release público)

1. Lista las dependencias directas de `pubspec.yaml` (no hace falta revisar cada transitiva salvo que una directa sea copyleft — en ese caso, revisa de qué depende ella).
2. Para cada una, confirma la licencia en su página de pub.dev.
3. Clasifícala según la tabla de arriba.
4. Cualquier copyleft fuerte o licencia ausente/desconocida: reemplázala o busca una excepción de licencia explícita del autor antes de publicar.

### Ejemplo de reporte

```markdown
## Reporte de Cumplimiento de Licencias — flutter_turnos_app

### Resumen
- Dependencias directas escaneadas: 14
- Conformes: 14
- Marcadas: 0

### Dependencias conformes
Todas usan licencias permisivas (MIT, BSD, Apache-2.0): flutter_bloc, formz,
flutter_secure_storage, firebase_core, firebase_auth, cloud_firestore,
firebase_messaging, http, equatable, intl, flutter_local_notifications,
go_router, cached_network_image, shared_preferences.
```

## Real-world usage

Para el calendario de turnos: antes de publicar el repo (código de la app, no el playbook) como público, se corre el proceso de auditoría manual sobre las ~15 dependencias directas del `pubspec.yaml`; el resultado se guarda como `docs/license-compliance-report.md` con fecha, para tener evidencia de la revisión si en el futuro se agrega un colaborador o se plantea monetizar la app.
