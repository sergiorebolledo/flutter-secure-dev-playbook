# Seguridad en Producción: WAF, RASP, SIEM/SOC, IRP

## Cuándo usar este archivo

Al preparar el paso a producción de una app con backend propio, al decidir qué nivel de monitoreo agregar, o al escribir un plan de respuesta a incidentes antes de publicarla en Play Store.

**Nota de escala honesta**: la mayoría de estos controles nacieron para infraestructura empresarial con equipos de SOC 24/7. Como desarrollador independiente en tu primera app, no vas a montar un SIEM completo — pero conocer el concepto y su equivalente "hecho a mano" te sirve tanto para producción real como para entrevistas de trabajo.

---

## 1. WAF / WAAP (Web Application & API Protection)

Si tu backend es Cloud Functions/Cloud Run detrás de Firebase Hosting o un API Gateway:

- **Equivalente accesible**: Cloud Armor (GCP), AWS WAF, o Cloudflare (plan gratuito incluye protección básica contra bots y DDoS) delante de tu API.
- Reglas mínimas útiles: rate limiting por IP en endpoints de login/registro, bloqueo de patrones conocidos de inyección en query params.
- Si usas solo Firestore + Firebase Auth (sin API HTTP propia), gran parte de esta capa la absorbe la infraestructura de Google — tu control principal pasa a ser las **reglas de seguridad de Firestore** (ver [identity-access-aaa-cia.md](identity-access-aaa-cia.md)).

## 2. RASP (Runtime Application Self-Protection)

Protección embebida en el binario que detecta manipulación en tiempo de ejecución:

- Para Flutter: detección de root/jailbreak (`safe_device`, `flutter_jailbreak_detection`) y verificación de integridad con **Play Integrity API** en Android.
- Úsalo como **señal de riesgo** (ej. reduce funcionalidad o pide reautenticación) en vez de bloqueo absoluto — hay falsos positivos en dispositivos legítimos rooteados por el propio dueño.
- Prioridad baja para una primera app sin pagos ni datos regulados; documenta la decisión de no implementarlo aún (aceptación de riesgo consciente, no omisión accidental).

## 3. IAM / PAM

Ver [identity-access-aaa-cia.md](identity-access-aaa-cia.md#3-iampam-en-el-contexto-de-un-proyecto-pequeño) — MFA en tu cuenta de Google/Play Console es el control de mayor retorno con menor esfuerzo.

## 4. MFA

Cubierto en AAA — aplicar tanto a **usuarios de la app** (opcional para empleados, recomendado para supervisores) como a **tus propias cuentas de infraestructura** (Firebase, GitHub, Play Console — este último es obligatorio de facto: perder esa cuenta significa perder el control de publicación de la app).

## 5. SIEM / SOC — versión "hecha a mano" para un proyecto pequeño

Un SIEM centraliza logs de múltiples fuentes y un SOC los monitorea humanamente. La versión realista para ti:

| Componente enterprise | Equivalente accesible |
|---|---|
| SIEM (Splunk, Sentinel) | Firebase Crashlytics + Cloud Logging (GCP) centralizando logs de Cloud Functions |
| Alertas de SOC | Alertas de Cloud Monitoring (ej. tasa de errores 5xx anómala, picos de lecturas de Firestore) enviadas a tu correo/Slack |
| Dashboard de amenazas | Firebase App Check para detectar tráfico de clientes no genuinos (bots, apps modificadas) hablando con tu backend |

**Firebase App Check** merece mención especial: verifica que las requests a Firestore/Functions vengan realmente de tu app (Play Integrity/DeviceCheck), no de un script que copió tu API key. Es la protección más costo-efectiva contra abuso automatizado para un proyecto Firebase pequeño.

## 6. IRP (Incident Response Plan) — versión mínima viable

No necesitas un documento de 20 páginas. Necesitas responder 4 preguntas **antes** de que ocurra un incidente, no durante:

1. **¿Cómo me entero?** (Crashlytics, alertas de Cloud Monitoring, reporte de usuario por email/Play Store review)
2. **¿Qué hago primero?** Para una fuga de datos: revocar tokens comprometidos (Firebase Auth permite invalidar sesiones), rotar API keys expuestas, desplegar regla de Firestore corregida.
3. **¿A quién aviso?** Si hay datos personales de usuarios afectados, evalúa obligación de notificación según la ley aplicable (en Chile, ver Ley 19.628 y sus actualizaciones).
4. **¿Cómo evito que se repita?** Post-mortem breve: causa raíz, qué control lo hubiera prevenido (mapear a CWE/STRIDE ya documentado en el repo), agregarlo a este mismo repositorio como lección aprendida.

Plantilla mínima para guardar en `docs/incidents/` cuando ocurra algo real:

```markdown
# Incidente: <título corto> — <fecha>

## Qué pasó
## Cómo se detectó
## Impacto (usuarios afectados, datos expuestos)
## Causa raíz (CWE/STRIDE si aplica)
## Acción inmediata tomada
## Prevención a futuro
```

## Real-world usage

Para el calendario de turnos: Firebase App Check activado en Firestore/Functions; Crashlytics + Cloud Logging centralizan errores; una alerta simple de Cloud Monitoring avisa por correo si la tasa de errores 5xx supera un umbral; existe un `docs/incidents/README.md` vacío con la plantilla lista, por si alguna vez hace falta usarla.
