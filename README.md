# 🏖️ Vacation Flow — Automatización de Solicitudes de Tiempo Libre

Sistema de gestión de vacaciones y días libres construido en n8n, con validaciones automáticas, aprobación humana vía Discord y sincronización con Google Calendar.

---

## 🧩 El problema que resuelve

Los procesos manuales de solicitud de vacaciones generan fricción: correos perdidos, aprobaciones que tardan días, calendarios desactualizados y sin trazabilidad del saldo de días disponibles por empleado.

**Vacation Flow digitaliza y automatiza todo el ciclo**: desde que el empleado envía su solicitud hasta que RRHH aprueba, el calendario se actualiza y el saldo de días se descuenta automáticamente.

---

## 🏗️ Arquitectura

```
┌──────────────────────────────────────────────────────────────┐
│                      VACATION FLOW                           │
│                                                              │
│  ENTRADA (dos vías):                                         │
│  ┌─────────────────────┐   ┌──────────────────────────┐     │
│  │  Form Trigger n8n   │   │  Webhook POST autenticado │     │
│  │  (interfaz web)     │   │  (integración con APIs)   │     │
│  └──────────┬──────────┘   └────────────┬─────────────┘     │
│             └──────────────┬────────────┘                    │
│                            ▼                                 │
│              Extraer variables de la solicitud               │
│                            │                                 │
│             ┌──────────────▼──────────────┐                  │
│             │     VALIDACIONES EN CASCADA  │                  │
│             │                             │                  │
│             │  ¿Solicitud con 7+ días     │                  │
│             │  de anticipación?           │                  │
│             │           │                 │                  │
│             │  ¿Fechas son correctas?     │                  │
│             │  (fin > inicio)             │                  │
│             │           │                 │                  │
│             │  ¿Tiene saldo de días       │                  │
│             │  suficiente? (PostgreSQL)   │                  │
│             └──────────────┬──────────────┘                  │
│                    FALLA   │  PASA                           │
│                    ─────   │  ─────                          │
│               Email rechazo│                                 │
│               automático   ▼                                 │
│                    Notificación a RRHH (Discord)             │
│                            │                                 │
│                    Esperar 48 horas                          │
│                    (Human-in-the-loop)                       │
│                            │                                 │
│                   ¿RRHH aprueba?                             │
│                   NO ──────┼──── SÍ                         │
│                   │        │      │                          │
│              Email rechazo │   Crear evento Google Calendar  │
│                            │   Restar días en PostgreSQL     │
│                            │   Email confirmación            │
└──────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Stack tecnológico

| Componente | Tecnología |
|---|---|
| Orquestación | n8n |
| Base de datos | PostgreSQL (NeonTech) |
| Notificaciones internas | Discord Webhook |
| Calendario | Google Calendar API |
| Correos | Gmail (OAuth2) |
| Validación de entrada | JavaScript (nodo Code de n8n) |
| Autenticación webhooks | Header Auth |

---

## 🔄 Flujo detallado

### 1. Entrada — dos vías disponibles

**Vía A: Formulario nativo de n8n**
Interfaz web lista para usar, con campos: nombre completo, correo, fecha inicio, fecha de retorno y comentarios.

**Vía B: Webhook POST autenticado**
Para integrar con sistemas externos (HRIS, apps internas). Requiere header de autenticación y acepta JSON:

```json
{
  "Nombre completo": "Ana García",
  "Correo electrónico": "ana@empresa.com",
  "Fecha inicio": "2025-09-15",
  "Fecha de retorno": "2025-09-20",
  "Comentarios": "Vacaciones de verano"
}
```

El nodo de validación JS verifica campos requeridos, formato de fechas (YYYY-MM-DD) y longitud de comentarios antes de continuar.

### 2. Validaciones en cascada

El flujo ejecuta tres validaciones en orden. Si alguna falla, envía un correo de rechazo al empleado con el motivo específico y detiene el proceso.

| Validación | Condición | Error si falla |
|---|---|---|
| Anticipación | `fecha_inicio >= hoy + 7 días` | "Se debe solicitar con 7 días de antelación" |
| Fechas correctas | `días_solicitados > 0` | Detalla las fechas recibidas y el cálculo |
| Saldo disponible | `días_solicitados <= vacation_days` | Indica cuántos días tiene disponibles |

### 3. Aprobación humana (Human-in-the-loop)

Si pasa todas las validaciones, RRHH recibe un mensaje en Discord con los detalles de la solicitud y un enlace único para aprobar o rechazar. El flujo queda en pausa hasta **48 horas** esperando la respuesta.

```
## Solicitud nueva:
Empleado: Ana García
Correo: ana@empresa.com

Fecha de inicio: 2025-09-15
Fecha de retorno: 2025-09-20

### Comentarios:
Vacaciones de verano

## Aprobar o Rechazar:
Para continuar, [click aquí](https://...)
```

### 4. Post-aprobación (ejecución automática)

Si RRHH aprueba, el flujo ejecuta en secuencia:

1. **Google Calendar** — crea evento "Vacaciones de [Nombre]" en el calendario compartido del equipo
2. **PostgreSQL** — descuenta los días solicitados del saldo del empleado (`vacation_days - días_solicitados`)
3. **Gmail** — envía correo de confirmación al empleado con las fechas aprobadas

---

## ⚙️ Configuración

### Estructura de base de datos (PostgreSQL)

```sql
CREATE TABLE days_off (
  id            SERIAL PRIMARY KEY,
  email         VARCHAR(255) UNIQUE NOT NULL,
  vacation_days INTEGER NOT NULL DEFAULT 0,
  sick_days     INTEGER NOT NULL DEFAULT 0
);

-- Ejemplo: dar 15 días de vacaciones a un empleado
INSERT INTO days_off (email, vacation_days, sick_days)
VALUES ('ana@empresa.com', 15, 5);
```

### Credenciales requeridas en n8n

| Credencial | Descripción |
|---|---|
| `Postgres` | Conexión a tu base de datos PostgreSQL |
| `Discord Webhook` | URL del webhook del canal de RRHH |
| `Gmail OAuth2` | Cuenta de correo para notificaciones |
| `Google Calendar OAuth2` | Cuenta con acceso al calendario del equipo |
| `Header Auth Secret` | Clave secreta para autenticar el webhook POST |

### ID del calendario de Google

En el nodo **Crear evento de vacaciones del empleado**, reemplaza el valor del campo `calendar` con el ID de tu calendario de Google (disponible en Configuración del calendario → Integrar calendario).

---

## 🚀 Instalación

1. **Clona el repositorio**
   ```bash
   git clone https://github.com/Jairh1294/vacation-flow.git
   ```

2. **Importa el workflow en n8n**
   - Ve a **Workflows → Import from File**
   - Selecciona el archivo `.json` del repositorio

3. **Configura las credenciales**
   - En n8n: **Settings → Credentials**
   - Agrega cada una de las credenciales listadas arriba

4. **Crea la tabla en PostgreSQL**
   - Ejecuta el script SQL de la sección anterior
   - Inserta los registros de empleados con su saldo inicial de días

5. **Activa el workflow**
   - Haz clic en **Activate**
   - Comparte la URL del formulario o el endpoint del webhook con los empleados

---

## 📋 Ejemplo de solicitud vía curl

```bash
curl -X POST https://tu-instancia-n8n.com/webhook/vacation/ask/form/001 \
  -H "Content-Type: application/json" \
  -H "Authorization: tu-header-secret" \
  -d '{
    "Nombre completo": "Ana García",
    "Correo electrónico": "ana@empresa.com",
    "Fecha inicio": "2025-09-15",
    "Fecha de retorno": "2025-09-20",
    "Comentarios": "Vacaciones de verano"
  }'
```

**Respuesta exitosa (200):**
```json
{ "valid": true, "status": 200 }
```

**Respuesta con error de validación (400):**
```json
{
  "valid": false,
  "status": 400,
  "errors": ["Se debe de solicitar con 7 días de antelación"]
}
```

---

## 🔐 Seguridad

- El webhook POST requiere autenticación por header (`Header Auth Secret`)
- El endpoint de aprobación de RRHH también requiere autenticación por header
- Las credenciales de Google y Discord se gestionan de forma segura a través del sistema de credenciales de n8n
- El flujo nunca expone datos sensibles en los mensajes de Discord o correos de rechazo

---

## 📄 Licencia

MIT License — libre para uso comercial y personal.

---

*Construido con n8n · PostgreSQL · Discord · Google Calendar · Gmail*
