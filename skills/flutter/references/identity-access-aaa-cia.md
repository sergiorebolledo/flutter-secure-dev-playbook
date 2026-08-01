# Identidad, Acceso y la Tríada CIA en apps Flutter

## Cuándo usar este archivo

Al implementar login, roles de usuario (empleado vs. supervisor), o al decidir qué tan estricto debe ser el control de acceso a un dato. También como marco mental general para justificar decisiones de seguridad ("¿esto protege confidencialidad, integridad o disponibilidad?").

---

## 1. CIA Triad — el marco base

| Propiedad | Qué significa | Amenaza si falla | Control típico en Flutter/backend |
|---|---|---|---|
| **Confidentiality** (Confidencialidad) | Solo quien debe ver el dato lo ve | Un empleado ve el sueldo de otro | Autorización server-side, cifrado en tránsito/reposo |
| **Integrity** (Integridad) | El dato no fue alterado sin autorización | Alguien modifica un turno ya aprobado sin dejar rastro | Validación server-side, checksums, logs de auditoría, reglas de Firestore que impiden escrituras no autorizadas |
| **Availability** (Disponibilidad) | El sistema responde cuando se necesita | La app no carga los turnos del día porque el backend está caído | Manejo de errores con reintentos/backoff, modo offline con caché local, rate limiting bien calibrado (que no bloquee uso legítimo) |

Cada decisión de seguridad se puede clasificar contra estas tres. Ejemplo: cifrar tokens localmente protege Confidencialidad; validar `overlapsWith()` en el backend antes de guardar un turno protege Integridad; tener una caché local con Hive para ver turnos sin conexión protege Disponibilidad.

---

## 2. AAA — Authentication, Authorization, Accounting

### Authentication (¿quién eres?)

- Usa Firebase Auth (o similar) en vez de un sistema propio — implementar login seguro desde cero (hashing, reset de password, verificación de email) es una fuente enorme de vulnerabilidades evitables.
- **MFA** (Multi-Factor Authentication): Firebase Auth soporta segundo factor por SMS/TOTP. Para una app de turnos, MFA es opcional para usuarios normales, pero **recomendado obligatorio para cuentas de supervisor/admin** que pueden editar turnos de otros.
- Verifica el estado de `emailVerified` antes de dar acceso a funciones sensibles si usas login por email/password.

### Authorization (¿qué puedes hacer?)

Esta es la parte que con más frecuencia se rompe en apps Firebase — la autorización real vive en las **reglas de seguridad del backend**, no en si el botón se muestra o no en la UI:

```javascript
// firestore.rules — ejemplo para el calendario de turnos
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /shifts/{shiftId} {
      // Lectura: solo el dueño del turno o un supervisor
      allow read: if request.auth != null &&
        (resource.data.userId == request.auth.uid ||
         request.auth.token.role == 'supervisor');

      // Escritura: el dueño puede crear/editar los suyos;
      // solo supervisor puede editar turnos de otros
      allow create: if request.auth != null &&
        request.resource.data.userId == request.auth.uid;

      allow update, delete: if request.auth != null &&
        (resource.data.userId == request.auth.uid ||
         request.auth.token.role == 'supervisor');
    }
  }
}
```

**Error común (Elevation of Privilege, CWE-284)**: guardar el rol (`role: "supervisor"`) como un campo editable en el documento de perfil del propio usuario en Firestore. Si el usuario puede escribir su propio documento de perfil, puede auto-asignarse `supervisor`. El rol debe vivir en **Firebase Custom Claims**, seteado únicamente desde una Cloud Function con verificación de quién hace la petición (ej. otro admin ya autorizado).

```typescript
// Cloud Function — únicamente un admin puede asignar el rol
export const setSupervisorRole = functions.https.onCall(async (data, context) => {
  if (context.auth?.token.role !== 'admin') {
    throw new functions.https.HttpsError('permission-denied', 'No autorizado');
  }
  await admin.auth().setCustomUserClaims(data.targetUid, { role: 'supervisor' });
});
```

### Accounting (¿qué hiciste?)

- Guarda un log de auditoría inmutable para acciones sensibles (editar/borrar turno de otro usuario): `who`, `what`, `when`, `before/after` — una colección `audit_logs` separada, de solo-append (reglas que prohíben `update`/`delete` sobre ese log).
- Esto también cubre **Repudiation** de STRIDE (ver [threat-modeling-stride-pasta-cwe.md](threat-modeling-stride-pasta-cwe.md)): si un supervisor borra un turno y el empleado reclama, el log de auditoría es la prueba objetiva.

---

## 3. IAM/PAM en el contexto de un proyecto pequeño

No necesitas un IAM/PAM corporativo para tu app, pero los principios sí aplican a **tus propios accesos administrativos**:

- **Firebase Console / Google Play Console**: activa MFA en tu propia cuenta de Google (la que usaste para crear la cuenta de desarrollador) — es el punto de fallo más crítico: quien controle esa cuenta controla la app en producción y puede publicar actualizaciones maliciosas.
- **Principio de mínimo privilegio para colaboradores futuros**: si en algún punto agregas a alguien al proyecto de Firebase/Play Console, dale el rol mínimo necesario (ej. "Viewer" para alguien que solo necesita ver crash reports), no "Owner" por defecto.
- **Rotación de credenciales de servicio**: si usas una Service Account de Firebase Admin SDK en CI/CD, rota su clave periódicamente y revócala si sospechas exposición.

## Real-world usage

Para el calendario de turnos: Firebase Auth con email/password + Google Sign-In; rol `supervisor` como Custom Claim asignado solo vía Cloud Function; reglas de Firestore que verifican `request.auth.uid` en cada operación (nunca confiando en un `userId` enviado desde el cliente); colección `audit_logs` de solo-append para ediciones de turnos ajenos; MFA activado en la cuenta de Google usada para Play Console.
