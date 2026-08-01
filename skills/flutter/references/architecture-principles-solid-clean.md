# Principios de Diseño y Arquitectura (SOLID, Clean Architecture, DDD, GRASP)

## Cuándo usar este archivo

Al decidir cómo estructurar una feature nueva, cómo dividir responsabilidades entre capas, o al revisar código que "huele mal" (clases gigantes, lógica de negocio mezclada con widgets, dependencias difíciles de testear).

Complementa a [architecture-conventions.md](architecture-conventions.md), que documenta las convenciones concretas de este proyecto (feature-first, `services/`, DI). Este archivo cubre los **principios** detrás de esas convenciones.

---

## 1. SOLID aplicado a Dart/Flutter

| Principio | Definición | Antipatrón típico en Flutter | Cómo aplicarlo |
|---|---|---|---|
| **S**ingle Responsibility | Una clase, una razón para cambiar | `HomeScreen` que hace fetch HTTP, parsea JSON, valida forms y pinta UI | Separar en `Widget` (solo UI) + `Cubit` (orquestación) + `Repository` (datos) |
| **O**pen/Closed | Abierto a extensión, cerrado a modificación | `switch` gigante sobre `UserType` repetido en 10 archivos | Polimorfismo con clases selladas (`sealed class`) + pattern matching, o Strategy pattern |
| **L**iskov Substitution | Una subclase debe poder sustituir a su superclase sin romper el contrato | `MockApiService` que lanza `UnimplementedError` en métodos que sí se usan en tests | Definir interfaces (`abstract interface class`) mínimas y honestas; no heredar solo para reusar código |
| **I**nterface Segregation | Interfaces pequeñas y específicas mejor que una gigante | Un `Repository` con 40 métodos que ningún consumidor usa completo | Dividir en `UserReader`, `UserWriter`, etc. cuando los consumidores difieren |
| **D**ependency Inversion | Depender de abstracciones, no de implementaciones concretas | `Cubit` que instancia `http.Client()` directamente | Inyectar `ApiClient` (interfaz) vía constructor; en producción pasa la implementación real, en tests un mock |

### Ejemplo: Dependency Inversion en un Cubit

```dart
// Mal: acoplado a la implementación concreta, imposible de testear sin red real
class ShiftsCubit extends Cubit<ShiftsState> {
  ShiftsCubit() : super(ShiftsInitial());

  Future<void> load() async {
    final response = await http.get(Uri.parse('https://api.example.com/shifts'));
    // ...
  }
}

// Bien: depende de una abstracción inyectada
abstract interface class ShiftsRepository {
  Future<List<Shift>> fetchShifts();
}

class ShiftsCubit extends Cubit<ShiftsState> {
  ShiftsCubit(this._repository) : super(ShiftsInitial());
  final ShiftsRepository _repository;

  Future<void> load() async {
    emit(ShiftsLoading());
    try {
      final shifts = await _repository.fetchShifts();
      emit(ShiftsLoaded(shifts));
    } catch (e) {
      emit(ShiftsError(e.toString()));
    }
  }
}
```

Esto es lo que hace que `bloc_test` funcione sin mockear HTTP: el test inyecta un `FakeShiftsRepository`.

---

## 2. DRY, KISS, YAGNI — cuándo se rompen entre sí

Estos tres principios compiten en la práctica:

- **DRY** (no te repitas) empuja a abstraer código repetido.
- **KISS** (simplicidad) empuja a no crear abstracciones prematuras.
- **YAGNI** (no lo vas a necesitar) empuja a no construir flexibilidad que nadie pidió.

**Regla práctica**: tolera 2 repeticiones. Al llegar a la 3ª instancia del mismo patrón, extrae. Extraer en la 1ª o 2ª repetición casi siempre produce la abstracción equivocada porque aún no sabes qué varía realmente.

Ejemplo de violación de YAGNI real en apps Flutter: crear un `GenericRepository<T>` con caché, retry, offline-first y sincronización delta **antes** de tener una segunda entidad además de `Shift`. Empieza con `ShiftsRepository` concreto; generaliza cuando exista `EmployeesRepository` y el patrón repetido sea evidente.

---

## 3. Clean Architecture (3 capas)

```
lib/
  core/                     # compartido entre features: theming, network client, errores
  features/
    shifts/
      data/                 # DTOs, fuentes de datos (API, SQLite/Hive), implementación de repositorios
        shift_dto.dart
        shifts_remote_datasource.dart
        shifts_repository_impl.dart
      domain/                # PURO Dart, sin imports de Flutter ni de paquetes de infraestructura
        shift.dart            # entidad
        shifts_repository.dart # interfaz (contrato)
        get_upcoming_shifts.dart # caso de uso (opcional en apps pequeñas)
      presentation/
        cubit/
        view/
        widgets/
```

**Regla que sí importa hacer cumplir**: `domain/` no debe importar `package:flutter/...` ni `package:http/...`. Si domain necesita `DateTime`, usa `dart:core`. Esto es lo que permite testear reglas de negocio (p. ej. "un turno no puede solaparse con otro") sin levantar el framework de widgets.

**Cuándo NO vale la pena la capa `domain/` separada con casos de uso**: en una app de 1 desarrollador con pocas reglas de negocio (como una primera app de calendario de turnos), es razonable fusionar `domain` dentro de `data` y mantener solo `Repository` + `Entity`. Añade casos de uso explícitos cuando una misma regla de negocio se invoque desde 2+ lugares distintos (UI y un background job, por ejemplo).

---

## 4. MVVM vs BLoC/Cubit en Flutter

MVVM y BLoC no son excluyentes — BLoC/Cubit *es* una implementación del patrón ViewModel:

| Concepto MVVM | Equivalente en BLoC/Cubit |
|---|---|
| Model | Entidades de `domain/` + `Repository` |
| ViewModel | `Cubit`/`Bloc` (expone `State`, recibe eventos/llamadas de método) |
| View | `Widget` (solo reacciona a `State` vía `BlocBuilder`, nunca contiene lógica de negocio) |

**Test de humo para saber si una `View` está limpia**: si pudieras convertir el widget a un mockup estático sin borrar ningún `if`/lógica de negocio real (solo estados de carga/error/UI), está bien separado.

---

## 5. Dependency Injection (DI)

En apps Flutter medianas, no siempre hace falta un framework de DI (`get_it`, `injectable`, `riverpod` como DI). Opciones por escala:

- **App pequeña (1–2 devs, pocas features)**: constructor injection manual desde `main.dart`, pasando repositorios hacia abajo con `RepositoryProvider`/`BlocProvider` de `flutter_bloc`.
- **App mediana/grande**: `get_it` como service locator + `injectable` para generar el registro automáticamente, evita el "prop drilling" de dependencias por 5 niveles de widgets.

**Por qué importa para seguridad, no solo para testing**: inyectar el cliente HTTP autenticado (en vez de instanciarlo donde se usa) es lo que permite rotar tokens, forzar `certificate pinning` o interceptar todas las llamadas salientes desde un solo punto — ver [security-owasp-masvs-asvs.md](security-owasp-masvs-asvs.md#network-security).

---

## 6. GRASP (resumen aplicado)

| Patrón GRASP | Pregunta que responde | Ejemplo Flutter |
|---|---|---|
| Information Expert | ¿Quién tiene los datos para hacer esto? | `Shift.overlapsWith(other)` vive en la entidad `Shift`, no en el `Cubit` |
| Creator | ¿Quién debería crear este objeto? | `ShiftsRepositoryImpl` crea `Shift` a partir del DTO, no el `Cubit` |
| Controller | ¿Quién recibe eventos del sistema? | El `Cubit`/`Bloc`, nunca el `Widget` directamente |
| Low Coupling / High Cohesion | ¿Esta clase depende de demasiadas otras? ¿Hace una sola cosa bien? | Guía general para decidir si dividir una clase |
| Protected Variations | ¿Qué punto de variación necesita una interfaz estable? | `PaymentGateway` interfaz, para poder cambiar de Stripe a otro proveedor sin tocar el `Cubit` |

---

## 7. Domain-Driven Design (DDD) — cuándo aplica realmente

DDD completo (bounded contexts, agregados, value objects, eventos de dominio) tiene sentido en backends complejos con múltiples equipos. En una app Flutter de un desarrollador, toma prestado solo lo útil:

- **Value Objects** para invariantes: en vez de pasar `String email` suelto, usa un tipo `Email` que valida el formato en el constructor (esto es literalmente lo que ya hace `formz` con `Email extends FormzInput`).
- **Ubiquitous Language**: nombra clases y variables como el negocio las nombra. Si el usuario dice "turno" y "descanso", el código debe decir `Shift` y `RestPeriod`, no `Item1` y `Item2`.

## Real-world usage

Para la app de calendario de turnos: `domain/shift.dart` define `Shift` con un método `overlapsWith(Shift other)` (Information Expert). `ShiftsRepository` es la interfaz; `ShiftsRepositoryImpl` en `data/` la implementa contra Hive/SQLite local o Firestore. `ShiftsCubit` en `presentation/` depende solo de la interfaz — así el test `shifts_cubit_test.dart` usa un `FakeShiftsRepository` sin tocar base de datos real.
