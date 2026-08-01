# Política de Seguridad

## Alcance

Este repositorio es contenido de referencia/documentación para Claude Code (archivos `SKILL.md` y `references/*.md`) — no contiene código Dart/Flutter ejecutable propio. Las áreas relevantes para reportes de seguridad son:

- **Ejemplos de código inseguros**: cualquier snippet marcado como "✅ Bien" que en realidad contenga un patrón inseguro, o guía desactualizada/engañosa (especialmente en `security-owasp-masvs-asvs.md`, `identity-access-aaa-cia.md`, `threat-modeling-stride-pasta-cwe.md`).
- **Recomendaciones que contradicen buenas prácticas actuales** (ej. un paquete recomendado que fue deprecado o tiene vulnerabilidades conocidas).
- **Manifiesto del plugin** (`.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`): permisos o configuración excesivos.

## Cómo reportar

Preferido: abre un [issue público](https://github.com/sergiorebolledo/flutter-secure-dev-playbook/issues) describiendo el problema — dado que este repo es documentación (no código con secretos/producción en juego), no se requiere un canal privado en la mayoría de los casos.

Si el hallazgo es sensible (ej. una guía que podría facilitar explotar apps de terceros que sigan el ejemplo incorrecto), usa el [reporte privado de vulnerabilidades de GitHub](https://github.com/sergiorebolledo/flutter-secure-dev-playbook/security/advisories/new) en su lugar.

### Qué incluir

- Qué archivo/sección contiene el problema
- Por qué el ejemplo o recomendación es incorrecto/inseguro
- La corrección sugerida, si la tienes (con referencia a la fuente oficial: OWASP, CWE, docs de Flutter/Dart)

### Tiempos de respuesta

Este es un proyecto personal de un solo mantenedor — sin SLA formal, pero los issues se revisan de forma regular.

## Reconocimiento

Con gusto se reconoce a quien reporte un hallazgo válido en el commit/PR que lo corrige.
