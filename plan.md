# MediaAgenda – plan.md
_“Agenda de pacientes con turnos online, gestión de disponibilidad y recordatorios”_  
Stack: **SvelteKit 2 / Svelte 5 (SSR)**, **TypeScript**, **Tailwind CSS 4**, **Prisma**, **PostgreSQL**, **Docker**, **Vercel/Fly/Railway**, **Vitest**, **Playwright**.

---

## 0) Objetivo y alcance
- Permitir a pacientes **reservar, reprogramar o cancelar** turnos online, evitando doble reserva.
- Panel **/admin** para consultorios: disponibilidad, feriados, buffers, bloqueos, agenda por profesional/consultorio, CRUD de pacientes y turnos, exportaciones y analíticas básicas.
- **Notificaciones** (email + opcional WhatsApp deep link) con plantillas editables.
- Calidad de producción: accesibilidad, seguridad, pruebas, CI/CD, monitoreo, backups y guías de despliegue.

---

## 1) Preparación del entorno (Día 0)
- [ ] Clonar repo y entrar al proyecto.
- [ ] Copiar `.env.example` → `.env` y completar:
  ```env
  DATABASE_URL=postgresql://user:pass@localhost:5432/mediagenda
  AUTH_SECRET=...
  TZ=America/Argentina/Buenos_Aires
  REMINDER_HOURS=24,2
  SMTP_HOST=...
  SMTP_PORT=...
  SMTP_USER=...
  SMTP_PASS=...
  PUBLIC_BASE_URL=http://localhost:5173
  ```
- [ ] Instalar dependencias `npm i`.
- [ ] Levantar stack de desarrollo: `docker compose up -d postgres` o `npm run dev:docker`.
- [ ] Inicializar DB: `npx prisma migrate dev` y `npx prisma db seed`.
- [ ] Correr app: `npm run dev`.

---

## 2) Estructura del repo
```
/src
  /routes                # pages, endpoints y actions
    /(public)
      +page.svelte
      appointment/
        +page.svelte     # flujo de reserva
        confirm/+server.ts
        reschedule/+page.svelte
        cancel/+page.svelte
    /admin
      +layout.server.ts
      +layout.svelte
      login/+page.svelte
      calendar/+page.svelte
      patients/+page.svelte
      appointments/+page.svelte
      settings/
        availability/+page.svelte
        templates/+page.svelte
    /api
      health/+server.ts
      version/+server.ts
  /lib
    /components          # UI
    /db                  # prisma client
    /scheduling          # lógica de slots, buffers, solapamientos
    /notifications       # email/whatsapp y plantillas
    /server              # auth, csrf, rate-limit, errors
    /utils               # helpers comunes
/prisma
  schema.prisma
/tests
  unit/                  # vitest
  e2e/                   # playwright
/docs
docker-compose.yml
```

---

## 3) Modelo de datos (Prisma)
Entidades: **User** (admin/profesional), **Patient**, **Service**, **Practitioner**, **ScheduleBlock**, **Appointment**, **Location** (opcional), **AuditLog**.

- [ ] Definir `schema.prisma` (ítems clave):
  - `Appointment`:
    - `id, practitionerId, patientId, serviceId, startsAt, endsAt, status (Reserved|Confirmed|Attended|NoShow|Cancelled), slotKey, notes`
    - Índices:
      - **Único** `(practitionerId, startsAt)` para prevenir doble reserva.
      - Index en `(startsAt)` para consultas por rango.
  - `ScheduleBlock` (disponibilidad/bloqueos/feriados/buffers):
    - `type: OPEN|BLOCK|HOLIDAY|BUFFER`, `rrule` opcional para recurrencia.
  - `AuditLog`:
    - `actorId, action, entityType, entityId, diff, at`.
- [ ] Crear migraciones: `npx prisma migrate dev`.
- [ ] Seeds (servicios, profesionales, pacientes demo y turnos): `npx prisma db seed`.

---

## 4) Seguridad y base de plataforma
- [ ] **Auth** para /admin (email+password o provider): sesiones httpOnly, rotación de tokens.
- [ ] **CSRF** en forms de acciones mutables (tokens por sesión + SameSite=strict).
- [ ] **Rate-limit** (p.ej. por IP+route) en acciones públicas: reservar, reprogramar, cancelar.
- [ ] **Validación** con Zod en cliente y servidor (schemas compartidos en `/lib/validators`).
- [ ] **CORS** y **Headers** de seguridad (CSP, HSTS en prod, X-Frame-Options).
- [ ] **Registro de auditoría** para operaciones sensibles (crear/editar/mover/eliminar turnos, ver historial).
- [ ] Backups automáticos de PostgreSQL (cron + retención).

---

## 5) Flujo público (pacientes)
### 5.1 Selección y disponibilidad
- [ ] Página de entrada: seleccionar **especialidad/profesional**, **servicio**, **día**.
- [ ] Cálculo de **slots disponibles** en `/lib/scheduling`:  
  1) Expandir disponibilidad por `ScheduleBlock` (OPEN, HOLIDAY, BLOCK, BUFFER).  
  2) Generar slots por servicio (duración + gap).  
  3) Restar turnos existentes (`Appointment` status activos).  
  4) Aplicar buffers (antes/después) y redondeos.  
  5) **Cachear** resultado (Redis opcional) por ventana (p.ej. 15 min).

### 5.2 Reserva con prevención de doble booking
- [ ] Formulario **Alta rápida de paciente** (nombre, email/teléfono) + Zod.
- [ ] **Transacción atómica**:  
  - Verificar slot libre con `SELECT ... FOR UPDATE` (o unique index catch).  
  - Insert `Appointment` con `status=Reserved` y `slotKey`.  
  - Manejar colisión por **índice único (practitionerId, startsAt)**.
- [ ] Confirmación: pantalla + (opcional) **email con .ics** (adjunto y/o link).

### 5.3 Reglas de reprogramación/cancelación
- [ ] Rutas dedicadas con token de turno (hash corto) + captchas si público.  
- [ ] Reglas:  
  - Reprogramar hasta X horas antes.  
  - Cancelar hasta Y horas antes.  
  - Cambios posteriores requieren contacto con el consultorio.  
- [ ] Actualizar estado a `Confirmed` en botón de confirmación del email.

### 5.4 Accesibilidad/SEO/Responsivo
- [ ] Componentes con etiquetas ARIA, foco visible, contraste AA+.  
- [ ] Metadatos y OG tags; sitemap; humano y liviano.  
- [ ] Responsive mobile-first (Tailwind 4).

---

## 6) Panel /admin (consultorio)
### 6.1 Login y protección
- [ ] Ruta `/admin/login` + guard en `+layout.server.ts`.
- [ ] Roles: **admin** y **practitioner** (scopes por vista y acciones).

### 6.2 Calendario (día/semana/mes)
- [ ] Vista por **profesional** y por **consultorio** (si `Location` se usa).
- [ ] Drag & drop de turnos (mover/cambiar duración) con validaciones de solapamiento.
- [ ] Búsqueda y filtros por paciente/estado/servicio/fecha.

### 6.3 Gestión de disponibilidad
- [ ] UI para crear **ScheduleBlock** (OPEN/BLOCK/HOLIDAY/BUFFER) + `rrule` (p.ej. “cada lu-mi-vi 9–13h”).  
- [ ] Vista de conflictos (overlaps) y previsualización de slots generados.

### 6.4 CRUD de pacientes
- [ ] Listado, alta/edición, **historial de turnos** y **notas breves** (solo texto).

### 6.5 CRUD de turnos y estados
- [ ] Crear/editar/mover/eliminar. Estados: **Reserved, Confirmed, Attended, NoShow, Cancelled**.  
- [ ] Acciones masivas (confirmar/cancelar).  
- [ ] Auditoría por acción.

### 6.6 Exportaciones y dashboard
- [ ] **CSV**: pacientes, turnos por rango y filtros.  
- [ ] **Dashboard**: ocupación (% por profesional/rango), ausentismo (NoShow), próximas 24/48h.

---

## 7) Notificaciones
- [ ] **Plantillas editables** (Mustache/Handlebars segura) guardadas en DB o filesystem.  
- [ ] **Email**: SMTP con cola (reintentos, DLQ).  
- [ ] **WhatsApp opcional**: generar **deep link** con texto prearmado (sin proveedor externo).  
- [ ] **Recordatorios**: enviar a `REMINDER_HOURS` (ej. 24 y 2 h antes).  
- [ ] **ICS**: generar archivo `text/calendar` con `METHOD:PUBLISH`, `UID`, `DTSTART/DTEND`, `SUMMARY`, `LOCATION`, `DESCRIPTION`, `URL`.  
- [ ] Tracking simple: guardar `NotificationLog` (opcional) con estado `SENT/FAILED` y error.

**Scheduler recomendado**  
- Worker separado (Node) o Jobs en plataforma (Railway cron / Fly Machines cron / Vercel cron + queue).  
- Reloj único (evitar doble ejecución).

---

## 8) Calidad, pruebas y DX
- [ ] **ESLint + Prettier** ya configurados.  
- [ ] **Vitest**:  
  - Reglas de agenda/solapamientos (slots, buffers, rrules, intersecciones).  
  - Prevención de doble reserva (simulación de carreras).  
- [ ] **Playwright (E2E)**: reservar → reprogramar → cancelar → confirmación.  
- [ ] **Manejo centralizado de errores** (hooks y `handle`): respuestas JSON limpias y páginas 4xx/5xx.  
- [ ] **Logs estructurados** (pino/winston) con request id.  
- [ ] **Fixtures** de datos para tests.  
- [ ] **Git Hooks** (lint-staged) y CI que corra lint, typecheck, unit y e2e smoke.

---

## 9) DevOps
- [ ] **Docker Compose**: app + postgres (y redis opcional).  
- [ ] `npm run dev:docker` → levanta servicios y aplica migraciones al inicio.  
- [ ] **.env.example** completo y **README** pro (setup, migraciones, seeds, scripts, troubleshooting).  
- [ ] Endpoints `/health` y `/version` (hash de commit + fecha).

**Backups & DR**  
- [ ] Snapshot diario + retención 7/30 días.  
- [ ] Restauración probada en entorno de staging.  

**Monitoreo**  
- [ ] Uptime (healthchecks), logs centralizados, métricas (p99 de reservas, mails/min).  
- [ ] Alertas (latencia, errores 5xx, jobs atascados).

---

## 10) Despliegue
- [ ] Guías para **Vercel / Fly.io / Railway** (1‑click si posible).  
- [ ] **Migraciones en prod**: `prisma migrate deploy` en step previo al arranque.  
- [ ] Variables de entorno en plataforma.  
- [ ] **Estrategia de colas** (si cron/worker): un proceso por worker, visibilidad y reintentos.

---

## 11) Extensiones futuras (opcional, roadmap)
- [ ] **Multi‑tenant (SaaS)**: agregar `organizationId` a tablas clave, aislamiento por `orgId`, branding por org.  
- [ ] **Sincronización Google/Outlook** (webhooks, mapeo bidireccional).  
- [ ] **PWA** para profesionales/pacientes (instalable + offline parcial).  
- [ ] **Stripe** para reservas de pago y no‑shows (depósito).  
- [ ] **IA**: predicción de ausentismo y recomendación de horarios.  
- [ ] **API pública** (OpenAPI) para integraciones.

---

## 12) Lista de verificación por funcionalidad
### Reserva pública
- [ ] UI selección (especialidad/profesional/servicio/día).  
- [ ] Generación de slots (con buffers y rrules).  
- [ ] Alta rápida de paciente + validaciones Zod.  
- [ ] Transacción atómica + índice único.  
- [ ] Confirmación (pantalla + email + .ics).  
- [ ] Reprogramar/cancelar con reglas.

### Panel /admin
- [ ] Login + roles.  
- [ ] Calendario D/W/M con drag&drop.  
- [ ] Disponibilidad (OPEN/BLOCK/HOLIDAY/BUFFER + RRULE).  
- [ ] CRUD pacientes (historial, notas).  
- [ ] CRUD turnos (estados, acciones masivas).  
- [ ] Export CSV, dashboard (ocupación, ausentismo, próximas 24/48h).

### Notificaciones
- [ ] Plantillas editables.  
- [ ] Email + deep link WhatsApp.  
- [ ] Recordatorios X horas antes.  
- [ ] ICS + logs de envío.

### Calidad & Ops
- [ ] A11y AA+, SEO básico.  
- [ ] Pruebas (Vitest + Playwright).  
- [ ] CI/CD, monitoreo, backups.  
- [ ] README y onboarding “clonar y correr”.

---

## 13) Comandos útiles
```bash
# Desarrollo
npm run dev               # app local
npm run dev:docker        # app + postgres por docker
npx prisma studio         # UI de DB

# Base de datos
npx prisma migrate dev
npx prisma migrate deploy
npx prisma db seed

# Pruebas
npm run test              # vitest
npm run test:e2e          # playwright

# Lint & types
npm run lint
npm run typecheck
```

---

## 14) API y Contratos (borrador)
- `POST /api/appointment` → reserva (body: patient, practitionerId, serviceId, startsAt).  
- `POST /api/appointment/:id/reschedule` → reprogramar.  
- `POST /api/appointment/:id/cancel` → cancelar.  
- `POST /api/appointment/:id/confirm` → confirmar.  
- `GET /api/availability?practitionerId&date=YYYY-MM-DD` → slots.  
- `GET /api/health` / `GET /api/version`.

**Errores estándar**: `code`, `message`, `fieldErrors`, `requestId`.

---

## 15) Consideraciones de rendimiento
- [ ] Índices correctos en `Appointment.startsAt`, `Appointment.practitionerId`.  
- [ ] Consultas por rango con paginación.  
- [ ] Cache de slots (clave: practitionerId+date+serviceId).  
- [ ] Minimizar N+1 en vistas de calendario (joins con proyección).

---

## 16) Privacidad y cumplimiento
- [ ] Políticas de privacidad y TDU.  
- [ ] Retención de datos (p.ej. borrar `NotificationLog` a 12 meses, anonimizaciones).  
- [ ] Consentimiento para notificaciones.  
- [ ] Exportación/eliminación de datos de paciente bajo solicitud.

---

## 17) “Definition of Done” (DoD) por release
- [ ] Funcionalidad completa y testeada (unit + e2e).  
- [ ] Docs actualizadas (README, plan.md, env).  
- [ ] Métricas saludables (error rate < 1%, p95 reservas < 400 ms).  
- [ ] Zero known P0/P1 abiertos.  
- [ ] Backup verificado y migraciones aplicadas.

---

## 18) Troubleshooting rápido
- Slots vacíos inesperados → revisar `ScheduleBlock` (RRULE/zonas horarias), buffers, solapamientos.  
- Doble reserva → verificar índice único y transacción.  
- Emails no salen → revisar cola, credenciales SMTP, logs y bloqueos de proveedor.  
- Playwright falla en CI → aumentar timeout o usar `--headed` en flakiness debug.

---

## 19) Roadmap sugerido (sprints de 2 semanas)
1. **MVP público + admin básico** (reserva → confirmación, calendario día, CRUD simple).  
2. **Disponibilidad avanzada** (rrules, buffers, feriados) + reprogramar/cancelar.  
3. **Notificaciones + .ics** + dashboard y exportaciones.  
4. **Monitoreo, backups, CI/CD**.  
5. **Opcional**: multi‑tenant y PWA.
