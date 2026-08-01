# Resiliencia Operativa: BCP/DRP, RTO/RPO

## Cuándo usar este archivo

Al decidir tu estrategia de backups, al diseñar el modo offline de la app, o al documentar qué pasa si Firebase (o tu backend) tiene una caída.

---

## 1. RTO y RPO — las dos métricas que importan

- **RTO (Recovery Time Objective)**: cuánto tiempo puede estar el servicio caído antes de que sea inaceptable.
- **RPO (Recovery Point Objective)**: cuántos datos puedes permitirte perder, medido en tiempo (ej. "máximo 1 hora de datos perdidos").

Para una app de calendario de turnos usada por un equipo pequeño, valores realistas y honestos:

| Métrica | Valor razonable | Por qué |
|---|---|---|
| RTO | Horas, no minutos | No es una app crítica de tiempo real (a diferencia de, ej., un sistema de pagos) |
| RPO | Minutos (gracias a Firestore, que replica en tiempo real) | Firestore ya ofrece alta disponibilidad gestionada; tu RPO real depende más de si el usuario alcanzó a sincronizar localmente antes de un fallo de red |

Definir esto **antes** de un incidente evita decisiones de pánico — si tu RTO objetivo es "horas", no necesitas guardia 24/7, pero sí necesitas saber cómo restaurar un backup en menos de ese tiempo.

---

## 2. BCP (Business Continuity Plan) — versión app pequeña

Qué mantiene el negocio/servicio funcionando ante una falla:

- **Modo offline-first**: cachear los turnos ya cargados localmente (Hive/sqflite) para que la app siga siendo usable (lectura) aunque no haya red o Firebase esté caído. Firestore además tiene persistencia offline nativa (`FirebaseFirestore.instance.settings = Settings(persistenceEnabled: true)`), que cubre gran parte de esto gratis.
- **Degradación elegante**: si falla el envío de notificaciones push (FCM caído), la app debe seguir funcionando para ver/crear turnos — las notificaciones son un *feature*, no una dependencia dura de la funcionalidad principal.

```dart
// Habilitar persistencia offline de Firestore (BCP casi gratis)
await FirebaseFirestore.instance.enablePersistence(
  const PersistenceSettings(synchronizeTabs: true),
);
```

## 3. DRP (Disaster Recovery Plan) — backups reales

- **Firestore**: exportaciones programadas (`gcloud firestore export`) a Cloud Storage, con una Cloud Function en cron (Cloud Scheduler) — configúralo desde el día 1, no cuando ya perdiste datos.
- **Cuenta de Google/Firebase/Play Console**: el verdadero "desastre" más probable para un desarrollador solo no es un fallo de Google, es perder acceso a tu propia cuenta (contraseña, 2FA perdido). Guarda los códigos de recuperación de MFA en un gestor de contraseñas, no en una nota suelta.
- **Código fuente**: obvio pero real — el repo en GitHub ya es tu backup de código; asegúrate de que el repo no sea el único lugar (ten un remoto adicional o al menos backups locales periódicos si trabajas con features grandes sin pushear).

## Real-world usage

Para el calendario de turnos: persistencia offline de Firestore activada desde el MVP; exportación automática semanal de Firestore a Cloud Storage vía Cloud Scheduler + Cloud Function; códigos de recuperación de la cuenta de Google guardada en un gestor de contraseñas; RTO objetivo documentado como "resuelto en menos de 24h" dado que no es una app de misión crítica en su primera versión.
