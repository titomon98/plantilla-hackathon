# Sistema de Gestión de Citas — Salón de Belleza

Agendamiento de citas para un salón de belleza con múltiples servicios y colaboradoras. Panel público sin login: el cliente elige servicio, colaboradora y horario disponible. Citas de **2 horas fijas**. Clientes identificados por número de celular.

Ver [CLAUDE.md](CLAUDE.md) para el detalle de dominio, reglas de negocio y estado del proyecto.

## Stack

- **Backend:** NestJS 12 + TypeORM + PostgreSQL (`backend/`)
- **Frontend:** React + Vite (`frontend/`)
- **Node:** 22

## Puesta en marcha

Requiere PostgreSQL local con una base `salon` (`CREATE DATABASE salon;`). Ver credenciales de desarrollo en [CLAUDE.md](CLAUDE.md#2-stack-tecnológico).

### Backend

```bash
cd backend
npm install --legacy-peer-deps
npm run start:dev
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

> El backend usa `--legacy-peer-deps` por un bug de resolución de peers de vitest 4 en npm 10.9.8.
