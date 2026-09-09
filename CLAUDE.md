# CLAUDE.md — Sistema de Gestión de Citas para Salón de Belleza

Este archivo guía a Claude Code en el desarrollo de este proyecto. Léelo completo antes de generar código.

## 1. Descripción del proyecto

Sistema de agendamiento de citas para un salón de belleza con múltiples tipos de servicio (uñas, pestañas, pintado de pelo, cejas, etc.) y varias colaboradoras que atienden. Cada colaboradora tiene un horario propio de disponibilidad, que puede deshabilitarse manualmente en bloques puntuales (vacaciones, permisos, etc.). Las citas duran **2 horas fijas** y, al agendarse, bloquean ese espacio en el horario de la colaboradora correspondiente.

Existe un panel público (sin login de administrador) donde cualquier cliente puede agendar una cita automáticamente: elige servicio, colaboradora y horario disponible. El cliente se identifica por número de celular, para poder usar esa información más adelante en reportes (frecuencia de visitas, servicios más solicitados, clientes recurrentes, etc.).

No existe código previo: el proyecto se crea desde cero (backend y frontend).

## 2. Stack tecnológico

### Backend
- **NestJS** (últimas versiones estables)
- **TypeORM** como ORM
- **PostgreSQL** local
- **Node 22**

**Conexión a base de datos (desarrollo local):**
```
host: localhost
port: 5432
usuario: root
password: Jose2598@
database: salon
```
> Si la base de datos `salon` no existe, créala antes de correr las migraciones (`CREATE DATABASE salon;`).

Variables de entorno (`.env`) en el backend:
```
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=root
DB_PASSWORD=Jose2598@
DB_NAME=salon
PORT=3000
```

### Frontend
- **React** (Vite, sin frameworks de UI pesados — nada de Next.js ni librerías de componentes complejas)
- **Node 22**
- Fetch/axios simple para consumir la API, React Router para las vistas (panel público de agendamiento + vistas internas si se requieren luego)
- Mantener el frontend simple: componentes funcionales, estado con hooks nativos (`useState`, `useEffect`), sin Redux ni librerías de estado global salvo que el proyecto crezca y se justifique.

## 3. Arquitectura backend

Seguir **arquitectura en capas** (controller → service → repository), módulos por dominio, siguiendo el estilo que ya usas en otros proyectos NestJS (ver `farmacia-backend-tds` como referencia de organización). Mantener dependencias mínimas y evitar complejidad innecesaria (no hexagonal completo salvo que se necesite).

```
src/
  modules/
    clientes/
    colaboradoras/
    servicios/
    horarios/
    disponibilidad/      # bloqueos manuales de horario
    citas/
  common/
    filters/
    interceptors/
    utils/
  config/
    database.config.ts
  main.ts
  app.module.ts
```

Cada módulo con su propio `*.module.ts`, `*.controller.ts`, `*.service.ts`, `*.entity.ts`, y DTOs (`create-*.dto.ts`, `update-*.dto.ts`).

## 4. Modelo de dominio

### 4.1 Servicio
Representa el tipo de servicio ofrecido (uñas, pestañas, pintado de pelo, cejas, etc.)

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid/int | PK |
| nombre | string | Ej. "Uñas acrílicas" |
| categoria | string | Ej. "Uñas", "Pestañas", "Cabello", "Cejas" |
| duracion_minutos | int | Default **120** (2 horas) — fijo por ahora, pero modelable a futuro por servicio |
| precio | decimal | Opcional para v1, útil para reportes futuros |
| activo | boolean | Para desactivar servicios sin borrarlos |

### 4.2 Colaboradora
Persona que atiende (estilista, manicurista, etc.)

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid/int | PK |
| nombre | string | |
| telefono | string | opcional |
| activo | boolean | |
| servicios | relación M:N con Servicio | qué servicios puede atender cada colaboradora |

### 4.3 Horario (disponibilidad base recurrente)
Define la disponibilidad "por defecto" de cada colaboradora, por día de la semana.

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid/int | PK |
| colaboradora_id | FK | |
| dia_semana | enum/int | 0-6 (domingo-sábado) |
| hora_inicio | time | Ej. 09:00 |
| hora_fin | time | Ej. 18:00 |

### 4.4 BloqueoHorario (deshabilitación manual)
Permite deshabilitar manualmente franjas puntuales del horario de una colaboradora (permiso, vacaciones, imprevistos), sin tocar su horario recurrente.

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid/int | PK |
| colaboradora_id | FK | |
| fecha | date | Fecha específica del bloqueo |
| hora_inicio | time | |
| hora_fin | time | |
| motivo | string | Opcional, texto libre |

### 4.5 Cliente
Identificado principalmente por número de celular.

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid/int | PK |
| nombre | string | |
| telefono | string | **único**, identificador principal desde el panel público |
| created_at | timestamp | Para reportes de clientes nuevos vs recurrentes |

> Regla: al agendar, se busca cliente por teléfono. Si no existe, se crea automáticamente con el nombre proporcionado.

### 4.6 Cita
| Campo | Tipo | Notas |
|---|---|---|
| id | uuid/int | PK |
| cliente_id | FK | |
| colaboradora_id | FK | |
| servicio_id | FK | |
| fecha | date | |
| hora_inicio | time | |
| hora_fin | time | Calculada: hora_inicio + 2 horas |
| estado | enum | `confirmada`, `cancelada`, `completada` |
| created_at | timestamp | |

## 5. Reglas de negocio clave

1. **Duración fija de cita = 2 horas.** Al crear una cita, `hora_fin = hora_inicio + 2h`, sin excepción por ahora.
2. **Cálculo de disponibilidad de una colaboradora en una fecha dada:**
   - Partir de su `Horario` recurrente para ese día de la semana.
   - Restar los bloques ya ocupados por `Cita`s confirmadas ese día.
   - Restar los bloques de `BloqueoHorario` manual para esa fecha.
   - El resultado son los slots de 2 horas disponibles para mostrar en el panel público.
3. **Al confirmar una cita**, ese bloque de 2 horas deja de aparecer disponible para esa colaboradora (no se necesita bloqueo manual adicional; se calcula por cruce con la tabla `Cita`).
4. **El panel público de agendamiento** permite, sin autenticación:
   - Elegir un servicio (filtra colaboradoras que lo ofrecen).
   - Elegir colaboradora (o dejar que el sistema sugiera la primera disponible).
   - Ver horarios disponibles según las reglas anteriores.
   - Ingresar teléfono (y nombre si es cliente nuevo) para confirmar.
5. **Identificación de clientes por celular** es la base para reportes futuros (v2): clientes recurrentes, servicios más solicitados por cliente, frecuencia de visita, etc. No es necesario construir los reportes en v1, pero el modelo de datos debe soportarlo desde ahora (no perder histórico, no borrar citas físicamente — usar estado `cancelada` en vez de DELETE).

## 6. Endpoints sugeridos (v1)

```
GET    /servicios
GET    /colaboradoras
GET    /colaboradoras/:id/disponibilidad?fecha=YYYY-MM-DD
POST   /citas                # body: { cliente: {telefono, nombre}, colaboradora_id, servicio_id, fecha, hora_inicio }
GET    /citas?colaboradora_id=&fecha=
PATCH  /citas/:id/cancelar

POST   /horarios             # crear horario recurrente de una colaboradora
POST   /bloqueos             # deshabilitar manualmente un bloque de horario
```

## 7. Frontend (panel público)

Vistas mínimas en React:
1. **Selección de servicio** (lista de categorías/servicios).
2. **Selección de colaboradora** (filtrada por servicio elegido).
3. **Selección de fecha y horario disponible** (slots de 2h calculados por el backend).
4. **Datos de contacto** (teléfono + nombre) y confirmación de cita.
5. Pantalla de confirmación final.

Mantenerlo simple: sin librería de UI pesada, CSS propio o utilidades mínimas (Tailwind si se desea, pero no obligatorio). Priorizar componentes pequeños y reutilizables (`ServicioCard`, `SlotButton`, `StepWizard`, etc.).

## 8. Comandos de desarrollo

### Backend
```bash
# Instalar dependencias
npm install

# Levantar en modo desarrollo
npm run start:dev

# Generar migración (si se usa migraciones en vez de synchronize)
npm run typeorm migration:generate -- -n NombreMigracion
npm run typeorm migration:run
```

### Frontend
```bash
npm install
npm run dev
```

## 9. Convenciones de código

- Backend en NestJS: DTOs con `class-validator`, entidades TypeORM con nombres de tabla en snake_case, servicios con lógica de negocio (no dejarla en controllers).
- Nombrar entidades y variables en español (consistente con el dominio del negocio), igual que en otros proyectos (`farmacia-backend-tds`).
- Preferir soluciones directas y con mínimas dependencias, evitando sobre-ingeniería (no hexagonal completo salvo necesidad real).
- No hacer DELETE físico de citas; usar cambio de `estado`.
- Fechas y horas: todo el manejo de disponibilidad se calcula en el backend, el frontend solo consume slots ya calculados.

## Estado del proyecto

- [x] Backend NestJS inicializado (`backend/`, Nest 12, Node 22). Deps del stack instaladas: `@nestjs/typeorm`, `typeorm`, `pg`, `@nestjs/config`, `class-validator`, `class-transformer`. Compila (`npm run build`).
- [x] Frontend React + Vite inicializado (`frontend/`). Deps: `react-router-dom`, `axios`. Compila (`npm run build`).
- [ ] Config de conexión a PostgreSQL (`database.config.ts` + `.env`)
- [ ] Entidades y módulos por dominio (clientes, colaboradoras, servicios, horarios, disponibilidad, citas)
- [ ] Lógica de cálculo de disponibilidad
- [ ] Endpoints v1
- [ ] Vistas del panel público (frontend)

> Nota: `npm install` en el backend requirió `--legacy-peer-deps` (bug de resolución de peers de vitest 4 en npm 10.9.8).

## 10. Pendiente / fuera de alcance de v1

- Reportes de clientes (frecuencia, recurrencia, servicios más solicitados) — el modelo de datos ya lo soporta, pero la construcción de dashboards/reportes queda para una fase posterior.
- Autenticación de administrador para gestionar colaboradoras, horarios y bloqueos (v1 puede crearse vía endpoints directos o seed, sin panel admin todavía).
- Notificaciones (SMS/WhatsApp) de confirmación — no incluido en v1.
