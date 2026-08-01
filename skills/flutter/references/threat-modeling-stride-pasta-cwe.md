# Threat Modeling: STRIDE, PASTA y CWE aplicados a apps Flutter

## Cuándo usar este archivo

Al diseñar una feature nueva que toca datos sensibles o dinero (pagos, autenticación, datos de otros usuarios), antes de una revisión de seguridad, o al documentar por qué se tomó una decisión de diseño de seguridad (útil para portafolio/entrevistas).

---

## 1. STRIDE — modelo de amenazas por diseño (Microsoft)

STRIDE se aplica por **componente** del sistema (cliente Flutter, API, base de datos), no a la app completa de una sola vez.

| Letra | Amenaza | Pregunta guía | Ejemplo en una app de turnos | Mitigación |
|---|---|---|---|---|
| **S** | Spoofing (suplantación) | ¿Puede alguien hacerse pasar por otro usuario o por el servidor? | Un atacante intercepta y reenvía un login con credenciales robadas | Auth con MFA, TLS + cert pinning para evitar servidores falsos |
| **T** | Tampering (manipulación) | ¿Puede alguien modificar datos en tránsito o en reposo sin autorización? | Modificar el `userId` en un request para editar el turno de otro empleado | Autorización server-side por sesión, no confiar en IDs enviados por el cliente |
| **R** | Repudiation (repudio) | ¿Puede alguien negar haber hecho una acción? | Un supervisor borra un turno y niega haberlo hecho | Logs de auditoría con user ID + timestamp inmutables (ver [production-security-ops.md](production-security-ops.md)) |
| **I** | Information Disclosure (divulgación) | ¿Puede alguien ver datos que no debería? | Un endpoint `/shifts/{id}` devuelve turnos de cualquier ID sin verificar dueño (IDOR) | Verificar `auth.uid` contra el dueño del recurso en cada request |
| **D** | Denial of Service | ¿Puede alguien tumbar el servicio o agotar recursos? | Un usuario hace miles de requests de creación de turnos por segundo | Rate limiting, cuotas de Firestore/backend |
| **E** | Elevation of Privilege | ¿Puede un usuario normal obtener permisos de admin? | Un campo `role: "admin"` editable desde el cliente en el perfil de usuario | Roles gestionados solo server-side (Firebase Custom Claims, nunca un campo de Firestore editable por el propio usuario) |

### Ejercicio práctico: STRIDE en 15 minutos para una feature nueva

Antes de programar "editar turno de otro empleado (para supervisores)":
1. Dibuja el flujo: `Widget → Cubit → Repository → API/Firestore`.
2. Para cada flecha, pregúntate las 6 letras de STRIDE.
3. La que más frecuentemente se olvida en apps Flutter/Firebase: **Elevation of Privilege** vía reglas de seguridad mal escritas (`allow write: if true;` dejado de una prueba).

---

## 2. PASTA (Process for Attack Simulation and Threat Analysis)

PASTA es más formal que STRIDE y está orientado a riesgo de negocio, útil para documentar decisiones en un repo/portafolio. 7 etapas, resumidas para una app móvil:

| Etapa | Qué se hace | Ejemplo para el calendario de turnos |
|---|---|---|
| 1. Definir objetivos de negocio | ¿Qué pasa si esto falla? | "Si un empleado ve el sueldo de otro, hay incumplimiento legal y pérdida de confianza" |
| 2. Definir alcance técnico | Qué componentes están en juego | App Flutter, Firebase Auth, Firestore, Cloud Functions |
| 3. Descomposición de la app | Diagramar flujos de datos (DFD) | Login → token → lectura de turnos propios → notificación push |
| 4. Análisis de amenazas | Aplicar STRIDE a cada flujo del DFD | (tabla de arriba) |
| 5. Análisis de vulnerabilidades | Mapear a CWE conocidos | Ver sección 3 |
| 6. Modelado de ataques | ¿Cómo explotaría esto un atacante paso a paso? | "Interceptar request → cambiar `userId` en el body → recibir datos de otro empleado" |
| 7. Análisis de impacto/riesgo | Probabilidad × impacto, priorizar mitigación | IDOR en datos de sueldo = alto impacto, prioridad 1 |

**Diferencia clave con STRIDE**: STRIDE es rápido y técnico (por componente); PASTA es más lento y conecta la amenaza con impacto de negocio — úsalo para features de alto riesgo (pagos, datos de nómina), no para cada botón nuevo.

---

## 3. CWE — Common Weakness Enumeration, mapeo práctico a Dart/Flutter

CWE es un diccionario de *tipos* de debilidad de código (no vulnerabilidades específicas). Los más relevantes para apps Flutter con backend:

| CWE | Nombre | Dónde aparece en Flutter/Dart | Ejemplo |
|---|---|---|---|
| **CWE-798** | Uso de credenciales hardcodeadas | API keys/secrets en el código Dart | `const apiKey = "sk_live_..."` commiteado al repo |
| **CWE-311** | Falta de cifrado de datos sensibles | Tokens en `SharedPreferences` sin cifrar | Ver [security-owasp-masvs-asvs.md](security-owasp-masvs-asvs.md#masvs-storage-almacenamiento-local) |
| **CWE-319** | Transmisión de datos sensibles en texto claro | `http://` en vez de `https://` | Login enviando password por HTTP |
| **CWE-284 / CWE-639** | Control de acceso inadecuado / IDOR | Backend no valida ownership del recurso | `GET /shifts/{id}` sin chequear `auth.uid` |
| **CWE-330** | Uso de valores aleatorios insuficientemente aleatorios | `Random()` en vez de `Random.secure()` para tokens/nonces | Generar un "código de invitación" predecible |
| **CWE-20** | Validación de input incorrecta | Confiar solo en la validación de `formz` del cliente | Backend acepta un `email` sin `@` porque no revalida |
| **CWE-522** | Credenciales insuficientemente protegidas | Password en logs (`print('login: $email / $password')`) | Log accidental en debug que llega a Crashlytics |
| **CWE-489** | Funcionalidad de debug activa en producción | `kDebugMode` mal usado, endpoints de prueba expuestos | Un botón "seed test data" visible en build de producción |
| **CWE-295** | Validación de certificado incorrecta | `badCertificateCallback` que retorna `true` siempre (ignora el error en vez de pinning real) | Copiar-pegar un fix de "ignora error SSL" de StackOverflow |

**Cómo usar esta tabla en revisiones de código propio**: cuando encuentres un bug de seguridad, anota su CWE en el mensaje de commit/PR (ej. `fix: valida ownership antes de editar turno (CWE-639, IDOR)`). Esto documenta tu proceso de forma estándar de industria — útil tanto para aprendizaje como para mostrar en un portafolio.

---

## 4. Priorización simple (para no paralizarte)

Como desarrollador único en tu primera app, no necesitas ejecutar PASTA completo en cada feature. Regla práctica:

1. **Siempre** (cualquier feature con datos de usuario): pregunta las 6 letras de STRIDE mentalmente, 2 minutos.
2. **Antes de producción**: revisa la tabla de CWE contra tu código de auth/storage/network.
3. **Solo si manejas dinero, salud o datos de terceros sensibles**: aplica PASTA completo y considera SAST/DAST automatizados (ver [secure-sdlc-devsecops.md](secure-sdlc-devsecops.md)).

## Real-world usage

Para "supervisor edita turno de un empleado": STRIDE identificó Elevation of Privilege como el riesgo principal → mitigación = Firebase Custom Claims (`role: supervisor` seteado solo desde Cloud Function con verificación admin, nunca editable desde el cliente) + regla de Firestore `allow update: if request.auth.token.role == 'supervisor'`.
