# Metodologías de Testing: TDD, BDD, QA y KPIs

## Cuándo usar este archivo

Al decidir el flujo de trabajo para escribir tests, al definir qué métricas de calidad trackear en el repo, o antes de escribir tests de una feature con reglas de negocio no triviales (ej. validar solapamiento de turnos).

Complementa a [testing.md](testing.md), que cubre la mecánica concreta (`bloc_test`, mocktail, golden tests, widget tests). Este archivo cubre **cuándo y por qué** aplicar cada metodología.

---

## 1. TDD (Test-Driven Development)

Ciclo: **Red → Green → Refactor**. Escribe el test que falla primero, el mínimo código para pasarlo, luego refactoriza con la red de seguridad del test ya en verde.

**Dónde vale la pena en una app Flutter**: lógica de negocio pura en `domain/` (sin Flutter, sin red) — es barata de testear con TDD porque no requiere mocks pesados.

```dart
// 1. Red: el test primero (falla porque overlapsWith no existe aún)
test('un turno que empieza durante otro turno se considera solapado', () {
  final a = Shift(start: DateTime(2026, 8, 1, 9), end: DateTime(2026, 8, 1, 17));
  final b = Shift(start: DateTime(2026, 8, 1, 15), end: DateTime(2026, 8, 1, 23));
  expect(a.overlapsWith(b), isTrue);
});

// 2. Green: código mínimo para pasar
class Shift {
  Shift({required this.start, required this.end});
  final DateTime start;
  final DateTime end;

  bool overlapsWith(Shift other) => start.isBefore(other.end) && other.start.isBefore(end);
}

// 3. Refactor: con el test en verde, puedes limpiar sin miedo a romper la regla
```

**Dónde NO vale la pena forzar TDD estricto**: en la capa de `Widget`/UI donde el diseño visual todavía está cambiando rápido — ahí un widget test *después* de estabilizar el diseño es más pragmático.

---

## 2. BDD (Behavior-Driven Development)

BDD describe comportamiento en lenguaje natural (formato Gherkin: `Given/When/Then`), pensado para que sea legible por alguien no técnico (útil si en el futuro trabajas con un product owner o cliente).

```gherkin
Feature: Crear turno de trabajo
  Scenario: Un empleado crea un turno sin solapamiento
    Given el empleado "Ana" no tiene turnos el 2026-08-05
    When Ana crea un turno de 09:00 a 17:00 ese día
    Then el turno se guarda exitosamente
    And Ana recibe una notificación de confirmación

  Scenario: Un empleado intenta crear un turno solapado
    Given el empleado "Ana" ya tiene un turno de 09:00 a 17:00 el 2026-08-05
    When Ana intenta crear un turno de 15:00 a 20:00 ese día
    Then la app muestra un error "Este turno se solapa con uno existente"
    And el turno no se guarda
```

En Dart, `bdd_widget_test` o simplemente convertir cada escenario Gherkin a un `testWidgets`/`blocTest` con nombre descriptivo logra el 90% del valor sin agregar un framework Gherkin completo — para un proyecto de un solo desarrollador, generalmente basta con nombrar los tests siguiendo la estructura Given/When/Then en el `test('...')` description.

**Cuándo BDD aporta valor real** (más allá de TDD normal): cuando necesitas que los criterios de aceptación sean el mismo artefacto que los tests — por ejemplo, si documentas requisitos en el repo para un futuro colaborador o para un README de portafolio, los escenarios Gherkin sirven de documentación viva.

---

## 3. QA (Quality Assurance) — proceso, no solo tests

QA es el conjunto de prácticas organizacionales para asegurar calidad, más allá de "correr los tests":

- **Checklist de PR** (aunque seas el único dev, fuerza disciplina):
  - [ ] `flutter analyze` limpio
  - [ ] Tests nuevos para la lógica agregada
  - [ ] Probado manualmente en al menos 1 dispositivo/emulador real
  - [ ] Sin `TODO`/`print` de debug olvidados
- **Testing manual exploratorio** antes de cada release: casos borde que los tests automatizados no cubren (rotación de pantalla, modo oscuro, permisos denegados, sin conexión).
- **Beta interna**: usar el **Internal Testing track** de Google Play Console (ya tienes cuenta de desarrollador) antes de producción — te permite instalar builds de release en tu propio dispositivo real vía un link privado, detectando problemas que el emulador no muestra.

---

## 4. KPIs de calidad — qué trackear sin obsesionarse

| KPI | Cómo medirlo en Flutter | Meta razonable para un proyecto solo |
|---|---|---|
| Code coverage | `flutter test --coverage` + `genhtml coverage/lcov.info` | 70-80% en `domain/` y `cubit/`; no persigas 100% en `widget/view` |
| Crash-free rate | Firebase Crashlytics | >99% de sesiones sin crash antes de considerar "estable" |
| Tiempo de build CI | Duración del job en GitHub Actions | Bajo control, no necesita optimizarse hasta que moleste (>5 min) |
| Deuda técnica visible | `// TODO` y `flutter analyze --fatal-infos` warnings | Cero warnings ignorados silenciosamente en `main` |

**Advertencia práctica**: coverage alto no significa tests útiles — un test que llama a un método sin `expect()` significativo infla coverage sin dar valor real. Prioriza calidad de aserciones sobre porcentaje.

## Real-world usage

Para el calendario de turnos: `Shift.overlapsWith()` desarrollado con TDD estricto (regla de negocio central, alto riesgo de bug sutil); features de UI cubiertas con widget tests escritos después de estabilizar el diseño; escenarios Gherkin documentados en `docs/features/*.feature` como referencia legible, convertidos a `blocTest` reales; coverage mínimo 75% en `lib/features/shifts/domain` y `cubit`, sin meta de coverage en `lib/features/shifts/view`.
