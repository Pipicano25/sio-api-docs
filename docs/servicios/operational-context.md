# Operational Context API

> **Autor:** pipicano25 Anderson Daniel Pipicano Ruiz

[← Índice](../index.md) · [Autenticación](../guias/autenticacion.md) · [Permisos](../guias/permisos.md) · [Errores](../guias/errores.md) · [Parámetros](../parametros/README.md)

## Introducción

Operational Context define **quién puede operar dónde**: las empresas cliente de SIO, sus sucursales, los usuarios operativos (membresía de una persona en una empresa), las asignaciones informativas de usuarios a sucursales (sede habitual) y los asesores de cada empresa. Es la **única fuente de membresías**: todos los servicios de negocio le consultan, con el token del usuario, si una persona tiene acceso a una empresa o sucursal. Desde la enmienda 2026-11-19 la membresía `ACTIVE` da acceso a la empresa y a **todas** sus sucursales activas; lo que la persona puede hacer lo limitan solo sus permisos de Keycloak (grupos). Para el frontend es el primer servicio que se llama después del login.

## Responsabilidad

- Empresas (vinculadas a una identidad de organización de Party Identity) y sucursales, con ubicación validada en Geography.
- Usuarios operativos: membresía de una persona (cuenta de Keycloak + identidad de Party Identity) en una empresa; asignaciones usuario–sucursal (dato informativo: sede habitual y listados de empleados por sucursal; no conceden ni retiran acceso).
- Asesores de empresa (varios vigentes, uno principal).
- Contratos de acceso operativo: contexto propio (`/me/operational-context`) y comprobación de acceso (`/operational-access`).

## Datos del servicio

| Campo | Valor |
|---|---|
| URL base | `https://sio-operational-context-dev.onrender.com/api/v1` |
| Salud | `https://sio-operational-context-dev.onrender.com/health/live`, `/health/ready` |
| Swagger / OpenAPI | [/docs](https://sio-operational-context-dev.onrender.com/docs) · [/openapi.json](https://sio-operational-context-dev.onrender.com/openapi.json) |
| Formato de error | Problem Details con `code` `OPERATIONAL_CONTEXT.*` y `trace_id` |
| Operaciones en el contrato desplegado | 33 (31 de negocio + 2 de salud) |
| Verificación | OpenAPI de Render del 2026-10-19 cruzado con `services/operational-context-service` |

## Autenticación y permisos

Todas las rutas salvo `/health/*` exigen `Authorization: Bearer <ACCESS_TOKEN>`. Permisos con formato `perm:operational-context:{recurso}:{accion}`:

| Recurso | Acciones y operaciones | Grupos de la plantilla con lectura | Grupos con escritura |
|---|---|---|---|
| `companies` | `read` (listar, consultar), `create` (+ `platform:administer`), `update` (modificar, reactivar + `platform:administer`), `delete` (inactivar + `platform:administer`) | Todos (`admin`, `Compras`, `Ventas` e `Inventario` vía `perm:operational-context:all:read`) | `superadmin` (único con `platform:administer`); `admin` puede `PUT` (no exige `platform:administer`) |
| `branches` | `read`, `create`, `update` (modificar, reactivar), `delete` (inactivar) | Todos | `superadmin`, `admin` |
| `application_users` | `read` (listar, consultar, **`/me/operational-context`**, **`/operational-access`**), `create`, `update` (modificar, reactivar), `delete` (inactivar) | Todos | `superadmin`, `admin` |
| `user_branches` | `read`, `create`, `update` (reactivar), `delete` (retirar) | Todos | `superadmin`, `admin` |
| `company_advisor_assignments` | `read`, `create`, `delete` (cerrar), `create` + `delete` (cambiar principal) | Todos | `superadmin`, `admin` |
| `platform` | `administer` (ver y administrar todas las empresas sin membresía), `lookup` (lecturas por UUID sin membresía, para identidades técnicas) | — | Solo `superadmin` (vía `perm:all:all:all`) |

## Alcance de datos

El alcance se calcula en **cada solicitud** a partir de las membresías `ACTIVE` del `sub` del token, sin caché:

| Situación | Efecto |
|---|---|
| Usuario con `perm:operational-context:platform:administer` | Ve y administra todas las empresas y sucursales |
| Usuario con membresía `ACTIVE` | Ve y opera solo sus empresas; las listas (`GET /companies`) se filtran a ellas |
| Lecturas por UUID (`GET /companies/{id}`, `GET /branches/{id}`, `GET /application-users/{id}`, …) | Además, las permite `perm:operational-context:platform:lookup` sin membresía |
| Fuera de alcance | `404 OPERATIONAL_CONTEXT.NOT_FOUND` con el mismo mensaje que un UUID inexistente («La empresa solicitada no existe.») |

## Tabla de endpoints

| # | Método | Ruta relativa | Descripción | Permiso | Clase | Ficha |
|---|---|---|---|---|---|---|
| 1 | `GET` | `/health/live` (sin prefijo) | Proceso vivo | Ninguno | `READ` | [Ver](#1-get-healthlive) |
| 2 | `GET` | `/health/ready` (sin prefijo) | PostgreSQL disponible | Ninguno | `READ` | [Ver](#2-get-healthready) |
| 3 | `GET` | `/companies` | Listar empresas del alcance | `companies:read` | `READ` | [Ver](#3-get-companies) |
| 4 | `POST` | `/companies` | Registrar empresa | `companies:create` + `platform:administer` | `WRITE` | [Ver](#4-post-companies) |
| 5 | `GET` | `/companies/{company_id}` | Consultar empresa | `companies:read` | `READ` | [Ver](#5-get-companiescompany_id) |
| 6 | `PUT` | `/companies/{company_id}` | Modificar empresa | `companies:update` | `WRITE` | [Ver](#6-put-companiescompany_id) |
| 7 | `POST` | `/companies/{company_id}/deactivate` | Inactivar empresa | `companies:delete` + `platform:administer` | `HIGH_RISK` | [Ver](#7-post-companiescompany_iddeactivate) |
| 8 | `POST` | `/companies/{company_id}/reactivate` | Reactivar empresa | `companies:update` + `platform:administer` | `WRITE` | [Ver](#8-post-companiescompany_idreactivate) |
| 9 | `GET` | `/companies/{company_id}/branches` | Listar sucursales | `branches:read` | `READ` | [Ver](#9-get-companiescompany_idbranches) |
| 10 | `POST` | `/companies/{company_id}/branches` | Registrar sucursal | `branches:create` | `WRITE` | [Ver](#10-post-companiescompany_idbranches) |
| 11 | `GET` | `/branches/{branch_id}` | Consultar sucursal | `branches:read` | `READ` | [Ver](#11-get-branchesbranch_id) |
| 12 | `PUT` | `/branches/{branch_id}` | Modificar sucursal | `branches:update` | `WRITE` | [Ver](#12-put-branchesbranch_id) |
| 13 | `POST` | `/branches/{branch_id}/deactivate` | Inactivar sucursal | `branches:delete` | `HIGH_RISK` | [Ver](#13-post-branchesbranch_iddeactivate) |
| 14 | `POST` | `/branches/{branch_id}/reactivate` | Reactivar sucursal | `branches:update` | `WRITE` | [Ver](#14-post-branchesbranch_idreactivate) |
| 15 | `GET` | `/companies/{company_id}/application-users` | Listar usuarios operativos | `application_users:read` | `READ` | [Ver](#15-get-companiescompany_idapplication-users) |
| 16 | `POST` | `/companies/{company_id}/application-users` | Registrar usuario operativo | `application_users:create` | `HIGH_RISK` | [Ver](#16-post-companiescompany_idapplication-users) |
| 17 | `GET` | `/application-users/{application_user_id}` | Consultar usuario operativo | `application_users:read` | `READ` | [Ver](#17-get-application-usersapplication_user_id) |
| 18 | `PUT` | `/application-users/{application_user_id}` | Modificar datos de referencia | `application_users:update` | `WRITE` | [Ver](#18-put-application-usersapplication_user_id) |
| 19 | `POST` | `/application-users/{application_user_id}/deactivate` | Inactivar membresía | `application_users:delete` | `HIGH_RISK` | [Ver](#19-post-application-usersapplication_user_iddeactivate) |
| 20 | `POST` | `/application-users/{application_user_id}/reactivate` | Reactivar membresía | `application_users:update` | `HIGH_RISK` | [Ver](#20-post-application-usersapplication_user_idreactivate) |
| 21 | `GET` | `/application-users/{application_user_id}/user-branches` | Sucursales asignadas a un usuario (informativo) | `user_branches:read` | `READ` | [Ver](#21-get-application-usersapplication_user_iduser-branches) |
| 22 | `POST` | `/application-users/{application_user_id}/user-branches` | Asignar sucursal (informativo) | `user_branches:create` | `HIGH_RISK` | [Ver](#22-post-application-usersapplication_user_iduser-branches) |
| 23 | `GET` | `/branches/{branch_id}/user-branches` | Usuarios asignados a una sucursal (informativo) | `user_branches:read` | `READ` | [Ver](#23-get-branchesbranch_iduser-branches) |
| 24 | `GET` | `/user-branches/{user_branch_id}` | Consultar asignación | `user_branches:read` | `READ` | [Ver](#24-get-user-branchesuser_branch_id) |
| 25 | `POST` | `/user-branches/{user_branch_id}/deactivate` | Retirar asignación (informativo) | `user_branches:delete` | `HIGH_RISK` | [Ver](#25-post-user-branchesuser_branch_iddeactivate) |
| 26 | `POST` | `/user-branches/{user_branch_id}/reactivate` | Reactivar asignación (informativo) | `user_branches:update` | `HIGH_RISK` | [Ver](#26-post-user-branchesuser_branch_idreactivate) |
| 27 | `GET` | `/companies/{company_id}/advisor-assignments` | Historial de asesores | `company_advisor_assignments:read` | `READ` | [Ver](#27-get-companiescompany_idadvisor-assignments) |
| 28 | `POST` | `/companies/{company_id}/advisor-assignments` | Registrar asesor | `company_advisor_assignments:create` | `WRITE` | [Ver](#28-post-companiescompany_idadvisor-assignments) |
| 29 | `GET` | `/advisor-assignments/{assignment_id}` | Consultar asignación de asesor | `company_advisor_assignments:read` | `READ` | [Ver](#29-get-advisor-assignmentsassignment_id) |
| 30 | `POST` | `/advisor-assignments/{assignment_id}/close` | Cerrar asignación | `company_advisor_assignments:delete` | `WRITE` | [Ver](#30-post-advisor-assignmentsassignment_idclose) |
| 31 | `POST` | `/companies/{company_id}/advisor-assignments/change-primary` | Cambiar asesor principal | `company_advisor_assignments:create` + `delete` | `WRITE` | [Ver](#31-post-companiescompany_idadvisor-assignmentschange-primary) |
| 32 | `GET` | `/me/operational-context` | Contexto operativo propio | `application_users:read` | `READ` | [Ver](#32-get-meoperational-context) |
| 33 | `GET` | `/operational-access` | Comprobar acceso operativo | `application_users:read` | `READ` | [Ver](#33-get-operational-access) |
| 34 | `GET` | `/companies/{company_id}/branch-employees` | Empleados separados por sucursal | `user_branches:read` + `branches:read` + `application_users:read` | `READ` | [Ver](#34-get-companiescompany_idbranch-employees) |
| 35 | `POST` | `/companies:onboard` | Dar de alta una empresa con su administrador | `companies:create` + `application_users:create` + `platform:administer` | `HIGH_RISK` | [Ver](#35-post-companiesonboard) |
| 36 | `POST` | `/account/initial-password` | Cambiar la contraseña temporal en el primer ingreso | Ninguno (sin token) | `HIGH_RISK` | [Ver](#36-post-accountinitial-password) |
| 37 | `POST` | `/companies/{company_id}/employees:onboard` | Registrar un empleado: persona, cuenta y usuario operativo | `application_users:create` | `HIGH_RISK` | [Ver](#37-post-companiescompany_idemployeesonboard) |
| 38 | `PUT` | `/application-users/{application_user_id}/employee` | Actualizar un empleado: persona, cuenta de Keycloak y usuario operativo | `application_users:update` | `HIGH_RISK` | [Ver](#38-put-application-usersapplication_user_idemployee) |

Todos los permisos de la tabla empiezan por `perm:operational-context:`.

## Convenciones comunes

- Listas paginadas (`page` ≥ 1, `page_size` 1–100, por defecto 20) con la envoltura `items`, `page`, `page_size`, `total_records`, `total_pages`. Por defecto devuelven solo `status=ACTIVE`; `status=INACTIVE` muestra las inactivas.
- `PUT` es **parcial**: el cuerpo lleva solo los campos que cambian; un campo ausente no cambia; `null` borra un campo opcional y en uno obligatorio responde `422`. Cuerpo vacío `{}` o solo con `version` → `422`. Las validaciones (ubicación, unicidad, identidad) se aplican sobre el registro combinado; si nada cambia, no se escribe ni sube la versión.
- `version` (≥ 1) es **opcional** en todas las escrituras (`PUT`, `deactivate`, `reactivate`, `close`, `change-primary`): si se envía y no coincide → `409 OPERATIONAL_CONTEXT.VERSION_CONFLICT`; si no se envía, se aplica sobre lo vigente. Las operaciones de estado aceptan `{"version": n}`, cuerpo vacío `{}` o ningún cuerpo.
- No hay borrado físico: inactivar conserva el registro. Inactivar algo ya inactivo (o reactivar algo activo) → `409 INVALID_STATUS_TRANSITION`.
- Todos los modelos rechazan campos adicionales (`422`). Las cadenas en blanco en campos opcionales se tratan como `null`.
- Ubicación (`country_id`, `administrative_division_id`, `city_id`): se valida en Geography sin transacción abierta; valores inexistentes → `422 COUNTRY_NOT_FOUND` / `ADMINISTRATIVE_DIVISION_NOT_FOUND` / `CITY_NOT_FOUND`; inactivos → `422 LOCATION_INACTIVE`; incoherentes entre sí → `422 GEOGRAPHY_INCOHERENT`; división o ciudad sin país → `422 GEOGRAPHY_PARENT_REQUIRED`. `latitude`/`longitude` son decimales (número o texto) y se devuelven como cadenas.
- `422 IMMUTABLE_FIELD` cuando se envía un campo que no puede cambiar.
- Formato de 401/403/404/503 como en [Errores](../guias/errores.md); `503 DEPENDENCY_UNAVAILABLE` lleva `dependency` (`database`, `party-identity`, `geography`, `keycloak`).

Ejemplo de `PUT` parcial (solo cambia `phone`, sin `version`):

```bash
curl -sS -X PUT 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"phone": "+57 300 123 4567"}'
```

## Operaciones

### 1. `GET /health/live`

`READ`, sin autenticación. `https://sio-operational-context-dev.onrender.com/health/live` → `200` `{"status": "ok"}`.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/health/live'
```

### 2. `GET /health/ready`

`READ`, sin autenticación. `https://sio-operational-context-dev.onrender.com/health/ready` → `200` `{"status": "ok"}`; `503` si PostgreSQL no responde.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/health/ready'
```

### 3. `GET /companies`

| | |
|---|---|
| Clasificación | `READ` |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/companies` |
| Permiso | `perm:operational-context:companies:read` |
| Alcance | Solo las empresas con membresía `ACTIVE` del usuario; todas con `platform:administer` |

Parámetros de consulta: `page`, `page_size`, `status` (`ACTIVE` por defecto, `INACTIVE`), `organization_identity_id`, `country_id`, `city_id` (UUID).

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/companies?page_size=20' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```
Respuesta `200`: `PaginatedResponse[CompanyResponse]`. Ejemplo completo en [company_id](../parametros/company-id.md).

Ejemplo de respuesta `200` (`PaginatedResponse_CompanyResponse_`; valores ilustrativos):

```json
{
  "items": [
    {
      "id": "e6d7d0c7-a0be-4f4b-8bca-631867ff85c8",
      "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322111",
      "email": "contacto@laeconomia.com",
      "phone": "+57 300 123 4567",
      "address_line": "Calle 10 # 5-20",
      "country_id": "c0a80101-0000-4000-8000-000000000001",
      "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
      "city_id": "c0a80101-0000-4000-8000-000000000003",
      "postal_code": null,
      "latitude": "4.6097",
      "longitude": "-74.0817",
      "timezone": "America/Bogota",
      "default_currency": "COP",
      "status": "ACTIVE",
      "version": 1,
      "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "created_at": "2026-10-19T15:04:05Z",
      "updated_at": "2026-10-19T15:04:05Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_records": 1,
  "total_pages": 1
}
```

`CompanyResponse`: `id`, `organization_identity_id` ([identidad](../parametros/organization-identity-id.md) con la razón social), `email`, `phone`, `address_line`, `country_id`, `administrative_division_id`, `city_id`, `postal_code`, `latitude`, `longitude`, `timezone` (IANA, por ejemplo `America/Bogota`), `default_currency` (por ejemplo `COP`), `status`, `version`, `created_by`, `updated_by`, `created_at`, `updated_at`. **No tiene `name`.**

### 4. `POST /companies`

| | |
|---|---|
| Clasificación | `WRITE` · crea una empresa de la plataforma |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/companies` |
| Permisos | `perm:operational-context:companies:create` **y** `perm:operational-context:platform:administer` |

Cuerpo (`CompanyCreate`):

| Campo | Tipo | Obligatorio | Validación / valor por defecto | Cómo obtenerlo |
|---|---|---|---|---|
| `organization_identity_id` | UUID | Sí | Identidad `ACTIVE` en Party Identity (`422 ORGANIZATION_IDENTITY_NOT_FOUND` / `_INACTIVE`); una sola empresa por identidad (`409 DUPLICATE_ORGANIZATION_IDENTITY`) | [organization_identity_id](../parametros/organization-identity-id.md) |
| `email` | email \| null | No | Formato de correo | — |
| `phone` | string \| null | No | ≤ 40 | — |
| `address_line` | string \| null | No | — | — |
| `country_id`, `administrative_division_id`, `city_id` | UUID \| null | No | Coherentes y activos en Geography | [Geography](geography.md) |
| `postal_code` | string \| null | No | ≤ 20 | — |
| `latitude`, `longitude` | number \| string \| null | No | Decimales | — |
| `timezone` | string | No | 1–60; por defecto `America/Bogota` | — |
| `default_currency` | string | No | Por defecto `COP` | — |

```json
{
  "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322110",
  "email": "contacto@empresa-ejemplo.test",
  "phone": "+57 601 000 0000",
  "address_line": "Calle 1 # 2-3",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": "110111",
  "timezone": "America/Bogota",
  "default_currency": "COP"
}
```

Ejemplo de solicitud con el cuerpo anterior:

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322110",
    "email": "contacto@empresa-ejemplo.test",
    "phone": "+57 601 000 0000",
    "address_line": "Calle 1 # 2-3",
    "country_id": "c0a80101-0000-4000-8000-000000000001",
    "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
    "city_id": "c0a80101-0000-4000-8000-000000000003",
    "postal_code": "110111",
    "timezone": "America/Bogota",
    "default_currency": "COP"
  }'
```

Respuesta `201`: `CompanyResponse` (`status` `ACTIVE`, `version` 1). El `id` es el [company_id](../parametros/company-id.md). Dependencias: Party Identity y Geography.

Ejemplo de respuesta `201` (`CompanyResponse`; valores ilustrativos):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322111",
  "email": "contacto@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "default_currency": "COP",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 5. `GET /companies/{company_id}`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}`; permiso `companies:read`. Devuelve la empresa **en cualquier estado** si está en el alcance (membresía, `platform:administer` o `platform:lookup`). `404` si no.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Ejemplo de respuesta `200` (`CompanyResponse`; valores ilustrativos):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322111",
  "email": "contacto@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "default_currency": "COP",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 6. `PUT /companies/{company_id}`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}`; permiso `companies:update`. Cuerpo (`CompanyUpdate`, parcial): los campos de la operación 4, todos opcionales; `organization_identity_id`, `timezone` y `default_currency` no admiten `null`. `version` es opcional. Solo cambian los campos enviados; si nada cambia no escribe ni incrementa la versión. Una empresa `INACTIVE` no se modifica (`422 COMPANY_INACTIVE`). Errores: `409 VERSION_CONFLICT`, `409 DUPLICATE_ORGANIZATION_IDENTITY`, `422` de ubicación e identidad, `404`.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X PUT 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322111",
    "email": "contacto@laeconomia.com",
    "phone": "+57 300 123 4567",
    "address_line": "Calle 10 # 5-20",
    "country_id": "c0a80101-0000-4000-8000-000000000001",
    "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
    "city_id": "c0a80101-0000-4000-8000-000000000003",
    "latitude": "4.6097",
    "longitude": "-74.0817",
    "timezone": "America/Bogota",
    "default_currency": "COP",
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`CompanyResponse`; valores ilustrativos):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322111",
  "email": "contacto@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "default_currency": "COP",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 7. `POST /companies/{company_id}/deactivate`

`HIGH_RISK`: la empresa deja de ser operable para **todos** sus usuarios (las API de negocio responderán `404`/`403 COMPANY_INACTIVE`). URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/deactivate`; permisos `companies:delete` + `platform:administer`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo). No elimina ni modifica sus sucursales.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/deactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```
Respuesta `200` con `status: "INACTIVE"`; `409 INVALID_STATUS_TRANSITION` si ya estaba inactiva.

Ejemplo de respuesta `200` (`CompanyResponse`; valores ilustrativos):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322111",
  "email": "contacto@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "default_currency": "COP",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 8. `POST /companies/{company_id}/reactivate`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/reactivate`; permisos `companies:update` + `platform:administer`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo).

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/reactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```
Respuesta `200` con `status: "ACTIVE"`.

Ejemplo de respuesta `200` (`CompanyResponse`; valores ilustrativos):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322111",
  "email": "contacto@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "default_currency": "COP",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 9. `GET /companies/{company_id}/branches`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/branches`; permiso `branches:read`; alcance: membresía en la empresa. Parámetros: `page`, `page_size`, `search` (≤ 160; código y nombre), `status` (`ACTIVE` por defecto), `city_id`.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/branches?page_size=20' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```
Respuesta `200`: `PaginatedResponse[BranchResponse]`. Ejemplo completo en [branch_id](../parametros/branch-id.md).

Ejemplo de respuesta `200` (`PaginatedResponse_BranchResponse_`; valores ilustrativos):

```json
{
  "items": [
    {
      "id": "e6d7d0c7-a0be-4f4b-8bca-631867ff85c8",
      "company_id": "550e8400-e29b-41d4-a716-446655440000",
      "company_status": "ACTIVE",
      "code": "SUC-01",
      "name": "Sucursal Centro",
      "email": "centro@laeconomia.com",
      "phone": "+57 300 123 4567",
      "address_line": "Calle 10 # 5-20",
      "country_id": "c0a80101-0000-4000-8000-000000000001",
      "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
      "city_id": "c0a80101-0000-4000-8000-000000000003",
      "postal_code": null,
      "latitude": "4.6097",
      "longitude": "-74.0817",
      "timezone": "America/Bogota",
      "effective_timezone": "America/Bogota",
      "status": "ACTIVE",
      "version": 1,
      "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "created_at": "2026-10-19T15:04:05Z",
      "updated_at": "2026-10-19T15:04:05Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_records": 1,
  "total_pages": 1
}
```

`BranchResponse`: `id`, `company_id`, `company_status`, `code`, `name`, `email`, `phone`, `address_line`, `country_id`, `administrative_division_id`, `city_id`, `postal_code`, `latitude`, `longitude`, `timezone` (propia o `null`), `effective_timezone` (la propia o la de la empresa), `status`, `version`, trazabilidad.

### 10. `POST /companies/{company_id}/branches`

| | |
|---|---|
| Clasificación | `WRITE` |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/branches` |
| Permiso | `perm:operational-context:branches:create` |
| Alcance | Membresía en la empresa (o `platform:administer`); la empresa debe estar `ACTIVE` (`422 COMPANY_INACTIVE`) |

Cuerpo (`BranchCreate`): `code` (string, obligatorio, único por empresa: `409 DUPLICATE_BRANCH_CODE`), `name` (1–160, obligatorio), `email`, `phone` (≤ 40), `address_line`, `country_id`, `administrative_division_id`, `city_id`, `postal_code` (≤ 20), `latitude`, `longitude`, `timezone` (≤ 60; `null` hereda la de la empresa).

```json
{
  "code": "SUC-002",
  "name": "Sucursal Norte",
  "email": "norte@empresa-ejemplo.test",
  "phone": "+57 601 000 0002",
  "address_line": "Autopista Norte # 100-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "timezone": null
}
```

Ejemplo de solicitud con el cuerpo anterior:

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/branches' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "code": "SUC-002",
    "name": "Sucursal Norte",
    "email": "norte@empresa-ejemplo.test",
    "phone": "+57 601 000 0002",
    "address_line": "Autopista Norte # 100-20",
    "country_id": "c0a80101-0000-4000-8000-000000000001",
    "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
    "city_id": "c0a80101-0000-4000-8000-000000000003",
    "timezone": null
  }'
```

Respuesta `201`: `BranchCreatedResponse` = `BranchResponse` + `numbering` + `warnings`; su `id` es el [branch_id](../parametros/branch-id.md). Para que un usuario opere en ella hay que asignársela (operación 22).

**Numeración automática.** Guardada la sucursal, Operational Context pide a Purchase y a Sales que le dejen numeración, con el mismo token de quien la crea (`POST …/purchase-sequences:ensure-branch` y `POST …/sale-sequences:ensure-branch`). El prefijo sale del código de la sucursal: `OC-<código>` para compras y `FV-<código>` para ventas, separador `-`, 6 dígitos y sin reinicio (`OC-SUC-002-000001`, `FV-SUC-002-000001`). Para el prefijo, el código pasa a mayúsculas sin tildes, cada grupo de caracteres que no sean letras ni dígitos queda como un guion y se recorta a 20 caracteres en total.

| `numbering.purchases.result` / `numbering.sales.result` | Significado |
|---|---|
| `CREATED` | Se creó la numeración de la sucursal; trae `sequence_id` y `prefix` |
| `ALREADY_EXISTS` | La sucursal o la empresa ya tenía una secuencia activa (creada a mano): se conserva y tiene prioridad; trae la que aplica |
| `FAILED` | No se pudo (por ejemplo, el usuario no tiene `perm:purchase:purchase_sequences:create` o el servicio no respondió); `sequence_id` y `prefix` en `null` y un texto en `warnings` |

La sucursal **nunca** falla por la numeración: la escritura en los tres servicios no es atómica y un `FAILED` deja la sucursal creada. En ese caso la numeración se crea sola al confirmar la primera compra o venta de la sucursal. La numeración de ventas es interna: no lleva resolución de facturación de la DIAN (pendiente).

Ejemplo de respuesta `201` (`BranchCreatedResponse`; valores ilustrativos):

```json
{
  "id": "b361a654-58d4-422f-8ffe-9699a87e91e3",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "code": "SUC-01",
  "name": "Sucursal Centro",
  "email": "centro@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "effective_timezone": "America/Bogota",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z",
  "numbering": {
    "purchases": {
      "result": "CREATED",
      "sequence_id": "4e5f6a7b-8c9d-4e0f-9a1b-2c3d4e5f6a7b",
      "prefix": "OC-SUC-01"
    },
    "sales": {
      "result": "FAILED",
      "sequence_id": null,
      "prefix": null
    }
  },
  "warnings": [
    "No se pudo crear la numeración de ventas de la sucursal (FORBIDDEN). Se creará sola al confirmar la primera venta."
  ]
}
```

### 11. `GET /branches/{branch_id}`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/branches/{branch_id}`; permiso `branches:read`. Sucursal en cualquier estado, con `company_status`. `404` fuera de alcance.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/branches/7c9e6679-7425-40de-944b-e07fc1f90ae7' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Ejemplo de respuesta `200` (`BranchResponse`; valores ilustrativos):

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "code": "SUC-01",
  "name": "Sucursal Centro",
  "email": "centro@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "effective_timezone": "America/Bogota",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 12. `PUT /branches/{branch_id}`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/branches/{branch_id}`; permiso `branches:update`. Cuerpo (`BranchUpdate`, parcial): los campos de la operación 10, todos opcionales; `code` y `name` no admiten `null`. `version` es opcional. La empresa de la sucursal es **inmutable**. Geography solo se consulta si cambia la ubicación. Errores: `409 VERSION_CONFLICT`, `409 DUPLICATE_BRANCH_CODE`, `422`.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X PUT 'https://sio-operational-context-dev.onrender.com/api/v1/branches/7c9e6679-7425-40de-944b-e07fc1f90ae7' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "code": "SUC-01",
    "name": "Sucursal Centro",
    "email": "centro@laeconomia.com",
    "phone": "+57 300 123 4567",
    "address_line": "Calle 10 # 5-20",
    "country_id": "c0a80101-0000-4000-8000-000000000001",
    "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
    "city_id": "c0a80101-0000-4000-8000-000000000003",
    "latitude": "4.6097",
    "longitude": "-74.0817",
    "timezone": "America/Bogota",
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`BranchResponse`; valores ilustrativos):

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "code": "SUC-01",
  "name": "Sucursal Centro",
  "email": "centro@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "effective_timezone": "America/Bogota",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 13. `POST /branches/{branch_id}/deactivate`

`HIGH_RISK`: todos los miembros de la empresa pierden el acceso a esa sucursal (`allowed: false`, `BRANCH_INACTIVE`) y las ventas y compras de la sucursal fallan con `403 BRANCH_ACCESS_DENIED`; las existencias se conservan. URL `https://sio-operational-context-dev.onrender.com/api/v1/branches/{branch_id}/deactivate`; permiso `branches:delete`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo).

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/branches/7c9e6679-7425-40de-944b-e07fc1f90ae7/deactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`BranchResponse`; valores ilustrativos):

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "code": "SUC-01",
  "name": "Sucursal Centro",
  "email": "centro@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "effective_timezone": "America/Bogota",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 14. `POST /branches/{branch_id}/reactivate`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/branches/{branch_id}/reactivate`; permiso `branches:update`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo). Solo si la empresa está `ACTIVE` (`422 COMPANY_INACTIVE`).

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/branches/7c9e6679-7425-40de-944b-e07fc1f90ae7/reactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`BranchResponse`; valores ilustrativos):

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "code": "SUC-01",
  "name": "Sucursal Centro",
  "email": "centro@laeconomia.com",
  "phone": "+57 300 123 4567",
  "address_line": "Calle 10 # 5-20",
  "country_id": "c0a80101-0000-4000-8000-000000000001",
  "administrative_division_id": "c0a80101-0000-4000-8000-000000000002",
  "city_id": "c0a80101-0000-4000-8000-000000000003",
  "postal_code": null,
  "latitude": "4.6097",
  "longitude": "-74.0817",
  "timezone": "America/Bogota",
  "effective_timezone": "America/Bogota",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 15. `GET /companies/{company_id}/application-users`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/application-users`; permiso `application_users:read`. Parámetros: `page`, `page_size`, `status` (`ACTIVE` por defecto), `search` (≤ 160; nombre y correo de referencia), `keycloak_user_id` (≤ 255), `person_identity_id` (UUID).

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/application-users?page_size=20' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```
Respuesta `200`: `PaginatedResponse[ApplicationUserListItem]`: cada usuario operativo con los nombres de su
persona, leídos de Party Identity con el token de quien consulta y sin guardarlos. Si Party Identity no responde o
la persona no se encuentra, los nombres vienen en `null` y el listado sale igual. Dependencia: Party Identity
(una consulta por persona de la página).

Ejemplo de respuesta `200` (`PaginatedResponse_ApplicationUserListItem_`; valores ilustrativos):

```json
{
  "items": [
    {
      "id": "e6d7d0c7-a0be-4f4b-8bca-631867ff85c8",
      "company_id": "550e8400-e29b-41d4-a716-446655440000",
      "company_status": "ACTIVE",
      "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
      "display_name": "Ana Pérez",
      "reference_email": "compras@ejemplo.com",
      "status": "ACTIVE",
      "version": 1,
      "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "created_at": "2026-10-19T15:04:05Z",
      "updated_at": "2026-10-19T15:04:05Z",
      "first_name": "Ana",
      "middle_name": null,
      "last_name": "Pérez",
      "second_last_name": "Gómez"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_records": 1,
  "total_pages": 1
}
```

`ApplicationUserListItem` (los campos de `ApplicationUserResponse` más los nombres):

| Campo | Tipo | Significado |
|---|---|---|
| `id` | UUID | `application_user_id` (el que aparece en `/me/operational-context` y en `created_by_user_id` de ventas y compras) |
| `company_id`, `company_status` | — | Empresa de la membresía |
| `keycloak_user_id` | string | `sub` de la cuenta en Keycloak |
| `person_identity_id` | UUID | [Persona](../parametros/person-identity-id.md) en Party Identity |
| `display_name` | string (≤ 160) | Nombre para mostrar |
| `reference_email` | email \| null | Correo de referencia propio de SIO |
| `status`, `version`, trazabilidad | — | — |
| `first_name`, `middle_name`, `last_name`, `second_last_name` | string \| null | Nombres de la persona en Party Identity; solo en el listado (no en la consulta por id); `null` si Party Identity no respondió |

### 16. `POST /companies/{company_id}/application-users`

| | |
|---|---|
| Clasificación | `HIGH_RISK` · concede a una cuenta de Keycloak membresía (y, por tanto, acceso a los datos) de la empresa |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/application-users` |
| Permiso | `perm:operational-context:application_users:create` |
| Alcance | Membresía en la empresa (o `platform:administer`); empresa `ACTIVE` |

Cuerpo (`ApplicationUserCreate`):

| Campo | Tipo | Obligatorio | Validación | Cómo obtenerlo |
|---|---|---|---|---|
| `keycloak_user_id` | string \| null | Exactamente uno de los dos | 1–255; `sub` de la cuenta | El administrador de Keycloak, o `keycloak_user_id` de `/me/operational-context` del propio usuario |
| `username` | string \| null | Exactamente uno de los dos | 1–255; nombre de usuario en Keycloak | Keycloak |
| `person_identity_id` | UUID | Sí | Persona `ACTIVE` en Party Identity | [person_identity_id](../parametros/person-identity-id.md) |
| `display_name` | string | Sí | 1–160 | — |
| `reference_email` | email \| null | No | — | — |

```json
{
  "username": "operario.dao",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "display_name": "Ana Pérez",
  "reference_email": "operario.dao@ejemplo.test"
}
```

Ejemplo de solicitud con el cuerpo anterior:

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/application-users' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "operario.dao",
    "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
    "display_name": "Ana Pérez",
    "reference_email": "operario.dao@ejemplo.test"
  }'
```

Orden de validación (verificado): empresa visible y `ACTIVE` → duplicado por persona (`409 DUPLICATE_MEMBERSHIP`) → cuenta en Keycloak (`422 KEYCLOAK_USER_NOT_FOUND`, `422 KEYCLOAK_USER_DISABLED`, `422 TECHNICAL_ACCOUNT_NOT_ALLOWED` si es cuenta de servicio) → duplicado por `sub` → persona en Party Identity (`422 PERSON_IDENTITY_NOT_FOUND` / `_INACTIVE`) → inserción. Respuesta `201`: `ApplicationUserResponse` con `keycloak_user_id` resuelto. El servicio consulta la Admin API de Keycloak con su cuenta técnica. Dependencias: Keycloak, Party Identity.

Ejemplo de respuesta `201` (`ApplicationUserResponse`; valores ilustrativos):

```json
{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "display_name": "Ana Pérez",
  "reference_email": "ana.perez@ejemplo.com",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 17. `GET /application-users/{application_user_id}`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/application-users/{application_user_id}`; permiso `application_users:read`. Membresía en cualquier estado; `404` fuera de alcance.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/application-users/a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Ejemplo de respuesta `200` (`ApplicationUserResponse`; valores ilustrativos):

```json
{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "display_name": "Ana Pérez",
  "reference_email": "compras@ejemplo.com",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 18. `PUT /application-users/{application_user_id}`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/application-users/{application_user_id}`; permiso `application_users:update`. Cuerpo (`ApplicationUserUpdate`, parcial): `display_name` (1–160; no admite `null`), `reference_email` (`null` lo borra), `version` (opcional); envíe al menos uno de los dos primeros. Enviar `keycloak_user_id`, `person_identity_id` o `company_id` → `422 IMMUTABLE_FIELD`.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X PUT 'https://sio-operational-context-dev.onrender.com/api/v1/application-users/a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "display_name": "Ana Pérez",
    "reference_email": "ana.perez@ejemplo.com",
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`ApplicationUserResponse`; valores ilustrativos):

```json
{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "display_name": "Ana Pérez",
  "reference_email": "ana.perez@ejemplo.com",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 19. `POST /application-users/{application_user_id}/deactivate`

`HIGH_RISK`: el usuario pierde el acceso a esa empresa (`USER_MEMBERSHIP_INACTIVE`) en todos los servicios. URL `https://sio-operational-context-dev.onrender.com/api/v1/application-users/{application_user_id}/deactivate`; permiso `application_users:delete`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo). Solo inactiva esta membresía.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/application-users/a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d/deactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`ApplicationUserResponse`; valores ilustrativos):

```json
{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "display_name": "Ana Pérez",
  "reference_email": "compras@ejemplo.com",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 20. `POST /application-users/{application_user_id}/reactivate`

`HIGH_RISK` (restituye acceso). URL `https://sio-operational-context-dev.onrender.com/api/v1/application-users/{application_user_id}/reactivate`; permiso `application_users:update`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo). Revalida la cuenta en Keycloak y la persona en Party Identity (mismos `422` de la operación 16).

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/application-users/a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d/reactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`ApplicationUserResponse`; valores ilustrativos):

```json
{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_status": "ACTIVE",
  "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "display_name": "Ana Pérez",
  "reference_email": "compras@ejemplo.com",
  "status": "ACTIVE",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 21. `GET /application-users/{application_user_id}/user-branches`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/application-users/{application_user_id}/user-branches`; permiso `user_branches:read`. Parámetros: `status` (`ACTIVE` por defecto), `page`, `page_size`. Las asignaciones son informativas (sede habitual): el acceso operativo lo da la membresía `ACTIVE`, que cubre todas las sucursales activas de la empresa.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/application-users/a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d/user-branches?page_size=20' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```
Respuesta `200`: `PaginatedResponse[UserBranchResponse]`.

`UserBranchResponse`: `id` (`user_branch_id`), `application_user_id`, `branch_id`, `company_id`, `status`, `assigned_at`, `assigned_by` (`sub`), `deactivated_at`, `membership_status`, `branch_status`, `company_status`, `operational_access` (`{"allowed": boolean, "reason": string | null}`, acceso efectivo del usuario a la sucursal sin evaluar permisos; se calcula por membresía, empresa y sucursal, no por el estado de la asignación), `version`, trazabilidad.

```json
{
  "items": [
    {
      "id": "9f0a1b2c-3d4e-4f5a-8b6c-7d8e9f0a1b2c",
      "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
      "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "company_id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "ACTIVE",
      "assigned_at": "2026-01-15T15:00:00Z",
      "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "deactivated_at": null,
      "membership_status": "ACTIVE",
      "branch_status": "ACTIVE",
      "company_status": "ACTIVE",
      "operational_access": {"allowed": true, "reason": null},
      "version": 1,
      "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "created_at": "2026-01-15T15:00:00Z",
      "updated_at": "2026-01-15T15:00:00Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_records": 1,
  "total_pages": 1
}
```

### 22. `POST /application-users/{application_user_id}/user-branches`

`HIGH_RISK`. Registra la sucursal habitual del usuario (dato informativo): **no** concede acceso, porque la membresía `ACTIVE` ya da acceso a todas las sucursales activas de la empresa. URL `https://sio-operational-context-dev.onrender.com/api/v1/application-users/{application_user_id}/user-branches`; permiso `user_branches:create`. Cuerpo (`UserBranchCreate`): `{"branch_id": "…"}` ([branch_id](../parametros/branch-id.md)). Reglas: la sucursal debe ser de la misma empresa que la membresía (`422 BRANCH_COMPANY_MISMATCH`); membresía, sucursal y empresa `ACTIVE` (`422 MEMBERSHIP_INACTIVE`, `BRANCH_INACTIVE`, `COMPANY_INACTIVE`); no duplicar (`409 DUPLICATE_USER_BRANCH`). `assigned_by` es el `sub` del token.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/application-users/a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d/user-branches' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
  }'
```
Respuesta `201`: `UserBranchResponse`.

Ejemplo de respuesta `201` (`UserBranchResponse`; valores ilustrativos):

```json
{
  "id": "e8c5f9de-8af8-4aa7-8f96-536c03dc33ac",
  "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "ACTIVE",
  "assigned_at": "2026-10-19T15:04:05Z",
  "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "deactivated_at": null,
  "membership_status": "ACTIVE",
  "branch_status": "ACTIVE",
  "company_status": "ACTIVE",
  "operational_access": {
    "allowed": true,
    "reason": null
  },
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 23. `GET /branches/{branch_id}/user-branches`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/branches/{branch_id}/user-branches`; permiso `user_branches:read`. Parámetros: `status`, `page`, `page_size`. Lista los empleados asignados a la sucursal (dato informativo); no es la lista de quién puede operar en ella: puede hacerlo cualquier miembro `ACTIVE` de la empresa con los permisos necesarios.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/branches/7c9e6679-7425-40de-944b-e07fc1f90ae7/user-branches?page_size=20' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```
Respuesta `200`: `PaginatedResponse[UserBranchResponse]`.

Ejemplo de respuesta `200` (`PaginatedResponse_UserBranchResponse_`; valores ilustrativos):

```json
{
  "items": [
    {
      "id": "e6d7d0c7-a0be-4f4b-8bca-631867ff85c8",
      "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
      "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "company_id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "ACTIVE",
      "assigned_at": "2026-10-19T15:04:05Z",
      "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "deactivated_at": null,
      "membership_status": "ACTIVE",
      "branch_status": "ACTIVE",
      "company_status": "ACTIVE",
      "operational_access": {
        "allowed": true,
        "reason": null
      },
      "version": 1,
      "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "created_at": "2026-10-19T15:04:05Z",
      "updated_at": "2026-10-19T15:04:05Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_records": 1,
  "total_pages": 1
}
```

### 24. `GET /user-branches/{user_branch_id}`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/user-branches/{user_branch_id}`; permiso `user_branches:read`. `200` con `UserBranchResponse`; `404`.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/user-branches/fd88f6a7-3e56-466b-8b82-e0b19abfb3df' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Ejemplo de respuesta `200` (`UserBranchResponse`; valores ilustrativos):

```json
{
  "id": "fd88f6a7-3e56-466b-8b82-e0b19abfb3df",
  "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "ACTIVE",
  "assigned_at": "2026-10-19T15:04:05Z",
  "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "deactivated_at": null,
  "membership_status": "ACTIVE",
  "branch_status": "ACTIVE",
  "company_status": "ACTIVE",
  "operational_access": {
    "allowed": true,
    "reason": null
  },
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 25. `POST /user-branches/{user_branch_id}/deactivate`

`HIGH_RISK`. Retira la asignación informativa; **no** retira el acceso: mientras la membresía siga `ACTIVE`, el usuario conserva el acceso a la sucursal (`operational_access.allowed` sigue en `true` si la sucursal y la empresa están activas). URL `https://sio-operational-context-dev.onrender.com/api/v1/user-branches/{user_branch_id}/deactivate`; permiso `user_branches:delete`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo).

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/user-branches/fd88f6a7-3e56-466b-8b82-e0b19abfb3df/deactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```
Respuesta `200` con `status: "INACTIVE"` y `deactivated_at`.

Ejemplo de respuesta `200` (`UserBranchResponse`; valores ilustrativos):

```json
{
  "id": "fd88f6a7-3e56-466b-8b82-e0b19abfb3df",
  "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "INACTIVE",
  "assigned_at": "2026-10-19T15:04:05Z",
  "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "deactivated_at": "2026-10-19T15:04:05Z",
  "membership_status": "ACTIVE",
  "branch_status": "ACTIVE",
  "company_status": "ACTIVE",
  "operational_access": {
    "allowed": true,
    "reason": null
  },
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 26. `POST /user-branches/{user_branch_id}/reactivate`

`HIGH_RISK`. URL `https://sio-operational-context-dev.onrender.com/api/v1/user-branches/{user_branch_id}/reactivate`; permiso `user_branches:update`; cuerpo `{"version": n}` (`version` opcional; se acepta `{}` o ningún cuerpo). Exige membresía, sucursal y empresa `ACTIVE`. Solo vuelve a registrar la asignación informativa; no cambia el acceso.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/user-branches/fd88f6a7-3e56-466b-8b82-e0b19abfb3df/reactivate' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```

Ejemplo de respuesta `200` (`UserBranchResponse`; valores ilustrativos):

```json
{
  "id": "fd88f6a7-3e56-466b-8b82-e0b19abfb3df",
  "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "ACTIVE",
  "assigned_at": "2026-10-19T15:04:05Z",
  "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "deactivated_at": null,
  "membership_status": "ACTIVE",
  "branch_status": "ACTIVE",
  "company_status": "ACTIVE",
  "operational_access": {
    "allowed": true,
    "reason": null
  },
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 27. `GET /companies/{company_id}/advisor-assignments`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/advisor-assignments`; permiso `company_advisor_assignments:read`. Parámetros: `page`, `page_size`, `status` (`ACTIVE` = vigentes, `INACTIVE` = cerradas; sin valor, todas), `is_primary` (boolean), `person_identity_id` (UUID).

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/advisor-assignments?page_size=20' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```
Respuesta `200`: `PaginatedResponse[AdvisorAssignmentResponse]`.

Ejemplo de respuesta `200` (`PaginatedResponse_AdvisorAssignmentResponse_`; valores ilustrativos):

```json
{
  "items": [
    {
      "id": "e6d7d0c7-a0be-4f4b-8bca-631867ff85c8",
      "company_id": "550e8400-e29b-41d4-a716-446655440000",
      "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
      "is_primary": true,
      "status": "ACTIVE",
      "valid_from": "2026-10-01T00:00:00Z",
      "valid_to": null,
      "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "version": 1,
      "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
      "created_at": "2026-10-19T15:04:05Z",
      "updated_at": "2026-10-19T15:04:05Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_records": 1,
  "total_pages": 1
}
```

`AdvisorAssignmentResponse`: `id` (`assignment_id`), `company_id`, `person_identity_id`, `is_primary`, `status`, `valid_from`, `valid_to` (null si vigente), `assigned_by`, `version`, trazabilidad.

### 28. `POST /companies/{company_id}/advisor-assignments`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/advisor-assignments`; permiso `company_advisor_assignments:create`. Cuerpo (`AdvisorAssignmentCreate`): `person_identity_id` (UUID, obligatorio; persona `ACTIVE` en Party Identity), `is_primary` (boolean, por defecto `true`), `valid_from` (date-time, opcional; por defecto ahora). Reglas: varios asesores vigentes, **uno solo principal** (`409 PRIMARY_ADVISOR_EXISTS`); la misma persona no puede tener dos asignaciones vigentes (`409 DUPLICATE_ADVISOR_ASSIGNMENT`); `422 INVALID_VALIDITY_PERIOD`.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/advisor-assignments' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
    "is_primary": true,
    "valid_from": "2026-10-01T00:00:00Z"
  }'
```
Respuesta `201`.

Ejemplo de respuesta `201` (`AdvisorAssignmentResponse`; valores ilustrativos):

```json
{
  "id": "4af4ba18-a34d-462e-8392-d290e6c4d674",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "is_primary": true,
  "status": "ACTIVE",
  "valid_from": "2026-10-01T00:00:00Z",
  "valid_to": null,
  "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 29. `GET /advisor-assignments/{assignment_id}`

`READ`. URL `https://sio-operational-context-dev.onrender.com/api/v1/advisor-assignments/{assignment_id}`; permiso `company_advisor_assignments:read`. `200`; `404`.

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/advisor-assignments/93c4899b-7130-4feb-878a-e72c2e3b9d97' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Ejemplo de respuesta `200` (`AdvisorAssignmentResponse`; valores ilustrativos):

```json
{
  "id": "93c4899b-7130-4feb-878a-e72c2e3b9d97",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "is_primary": true,
  "status": "ACTIVE",
  "valid_from": "2026-10-01T00:00:00Z",
  "valid_to": null,
  "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 30. `POST /advisor-assignments/{assignment_id}/close`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/advisor-assignments/{assignment_id}/close`; permiso `company_advisor_assignments:delete`. Cuerpo (`AdvisorCloseRequest`, puede omitirse): `version` (opcional) y `valid_to` (opcional; por defecto ahora; ≥ `valid_from`, si no `422 INVALID_VALIDITY_PERIOD`). Una asignación cerrada es inmutable (no se reabre).

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/advisor-assignments/93c4899b-7130-4feb-878a-e72c2e3b9d97/close' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1
  }'
```
Respuesta `200` con `status: "INACTIVE"` y `valid_to`.

Ejemplo de respuesta `200` (`AdvisorAssignmentResponse`; valores ilustrativos):

```json
{
  "id": "93c4899b-7130-4feb-878a-e72c2e3b9d97",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
  "is_primary": true,
  "status": "ACTIVE",
  "valid_from": "2026-10-01T00:00:00Z",
  "valid_to": null,
  "assigned_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "version": 1,
  "created_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "updated_by": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "created_at": "2026-10-19T15:04:05Z",
  "updated_at": "2026-10-19T15:04:05Z"
}
```

### 31. `POST /companies/{company_id}/advisor-assignments/change-primary`

`WRITE`. URL `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/advisor-assignments/change-primary`; permisos `company_advisor_assignments:create` **y** `delete`. Cuerpo (`ChangePrimaryRequest`): `current_assignment_id` (UUID de la asignación principal vigente), `version` (su versión; opcional), `person_identity_id` (nueva persona, `ACTIVE`), `effective_at` (date-time, opcional). Cierra la vigente y crea la nueva en una sola transacción. Errores: `409 NO_ACTIVE_PRIMARY_ADVISOR`, `409 PRIMARY_ADVISOR_MISMATCH` (la indicada no es la principal vigente), `409 VERSION_CONFLICT`, `422`.

Ejemplo de solicitud (valores ilustrativos):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/advisor-assignments/change-primary' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "current_assignment_id": "78fd5a21-6769-49cb-89b8-313ec380c752",
    "version": 1,
    "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110"
  }'
```
Respuesta `201` (`ChangePrimaryResponse`): `{"closed": AdvisorAssignmentResponse, "created": AdvisorAssignmentResponse}`.

### 32. `GET /me/operational-context`

| | |
|---|---|
| Clasificación | `READ` · la primera llamada tras el login |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/me/operational-context` |
| Permiso | `perm:operational-context:application_users:read` |
| Parámetros | Ninguno |

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/me/operational-context' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Respuesta `200` (`MeOperationalContextResponse`): `keycloak_user_id` y `memberships[]` con `application_user_id`, `company_id`, `company_status`, `status` y `branches[]` (`branch_id`, `user_branch_status`, `branch_status`, `allowed`, `reason`). Ejemplo completo y uso en [company_id](../parametros/company-id.md) y [branch_id](../parametros/branch-id.md). En cada membresía, `branches[]` lista **todas** las sucursales de la empresa (activas e inactivas): la membresía `ACTIVE` da acceso a todas las activas. `user_branch_status` es la asignación informativa (`ACTIVE`, `INACTIVE` o `null` si el usuario no tiene asignación en esa sucursal) y no limita el acceso. `allowed`/`reason` se calculan en el orden membresía existente → membresía `ACTIVE` → empresa `ACTIVE` → sucursal `ACTIVE`; una sucursal inactiva aparece con `allowed: false` y `BRANCH_INACTIVE`, y una membresía inactiva con `USER_MEMBERSHIP_INACTIVE`. Devuelve también las membresías inactivas, para que la interfaz pueda explicarlo. Un usuario sin membresías recibe `memberships: []`.

Ejemplo de respuesta `200` (`MeOperationalContextResponse`; valores ilustrativos):

```json
{
  "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "memberships": [
    {
      "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
      "company_id": "550e8400-e29b-41d4-a716-446655440000",
      "company_status": "ACTIVE",
      "status": "ACTIVE",
      "branches": [
        {
          "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
          "user_branch_status": "ACTIVE",
          "branch_status": "ACTIVE",
          "allowed": true,
          "reason": null
        }
      ]
    }
  ]
}
```

### 33. `GET /operational-access`

| | |
|---|---|
| Clasificación | `READ` |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/operational-access` |
| Permiso | `perm:operational-context:application_users:read` |

Parámetros de consulta: `company_id` (UUID), `branch_id` (UUID), `permission` (≤ 255; `perm:{servicio}:{recurso}:{accion}`), `keycloak_user_id` (≤ 255; solo con `platform:lookup`, evalúa a otro usuario).

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/operational-access' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```
Respuesta `200` (`OperationalAccessResponse`): `allowed`, `reason`, `keycloak_user_id`, `company_id`, `branch_id`, `application_user_id`, `permission`. Catálogo de `reason` y ejemplo en [Permisos §6](../guias/permisos.md#6-comprobar-el-acceso-antes-de-actuar).

Ejemplo de respuesta `200` (`OperationalAccessResponse`; valores ilustrativos):

```json
{
  "allowed": true,
  "reason": null,
  "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
  "company_id": "550e8400-e29b-41d4-a716-446655440000",
  "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "permission": "perm:sales:sales:read"
}
```

Reglas: el acceso se evalúa por membresía (enmienda 2026-11-19): membresía existente → membresía `ACTIVE` → empresa `ACTIVE` → sucursal `ACTIVE` → permiso (por la vía técnica, empresa y sucursal se evalúan antes que la membresía). La asignación a sucursal no se evalúa: `BRANCH_NOT_ASSIGNED` y `USER_BRANCH_INACTIVE` siguen en el catálogo por compatibilidad, pero ya no se producen. Una denegación funcional responde `200` con `allowed: false`, nunca `403`. Por la vía propia (token del usuario), una empresa o sucursal ajena **o inexistente** responde `USER_NOT_MEMBER` sin revelar más. Por la vía técnica (`platform:lookup` + `keycloak_user_id`), un objetivo inexistente responde `404` y el permiso se evalúa con los roles efectivos del usuario objetivo leídos de Keycloak. Es la operación que invocan Sales, Purchase, Inventory, Commercial Relationship, Audit, Business Reporting y Product Inventory Query para decidir el alcance.

### 34. `GET /companies/{company_id}/branch-employees`

| | |
|---|---|
| Clasificación | `READ` · empleados de la empresa separados por sucursal |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/branch-employees` |
| Permisos | `user_branches:read` **y** `branches:read` **y** `application_users:read` |
| Alcance | Membresía en la empresa (o `platform:administer`); fuera del alcance → `404` como un UUID inexistente |
| Disponibilidad | Nueva (FR-025a); disponible tras el próximo despliegue de Operational Context |

Devuelve una página de **sucursales** (código y nombre) y, dentro de cada una, sus empleados asignados (asignación informativa; no determina quién puede operar en la sucursal) con el nombre y la identificación de la persona. El nombre y la identificación se consultan a Party Identity en el momento con el token del usuario (una vez por persona) y no se guardan; si Party Identity no responde o el token no puede leer personas, esos campos vienen en `null` y la respuesta sigue siendo `200`.

| Parámetro | Dónde | Tipo | Obligatorio | Validación / por defecto | Cómo obtenerlo |
|---|---|---|---|---|---|
| `company_id` | ruta | UUID | Sí | Empresa visible | [company_id](../parametros/company-id.md) |
| `branch_id` | query | UUID | No | Solo esa sucursal, en cualquier estado; de otra empresa → `404` | [branch_id](../parametros/branch-id.md) |
| `status` | query | `ACTIVE` \| `INACTIVE` | No | Estado de las sucursales; `ACTIVE` por defecto | — |
| `assignment_status` | query | `ACTIVE` \| `INACTIVE` | No | Estado de las asignaciones; `ACTIVE` por defecto (`INACTIVE` = empleados retirados de la sucursal) | — |
| `search` | query | string | No | ≤ 160; parcial en código y nombre de la sucursal | — |
| `page`, `page_size` | query | integer | No | 1–100; la paginación es de sucursales | — |

Ejemplo de solicitud (sin cuerpo):

```bash
curl -sS 'https://sio-operational-context-dev.onrender.com/api/v1/companies/550e8400-e29b-41d4-a716-446655440000/branch-employees' \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Ejemplo de respuesta `200` (`PaginatedResponse_BranchEmployeesResponse_`; valores ilustrativos):

```json
{
  "items": [
    {
      "branch_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "code": "PRINCIPAL",
      "name": "Sede Principal",
      "status": "ACTIVE",
      "employees": [
        {
          "application_user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
          "display_name": "Ana Pérez",
          "reference_email": "ana.perez@ejemplo.com",
          "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
          "person_identity_id": "a0e1d2c3-b4a5-4968-8776-655443322110",
          "identification_type": "CC",
          "identification_number": "1012345678",
          "first_name": "Ana",
          "middle_name": null,
          "last_name": "Pérez",
          "second_last_name": null,
          "membership_status": "ACTIVE",
          "user_branch_id": "e8c5f9de-8af8-4aa7-8f96-536c03dc33ac",
          "user_branch_status": "ACTIVE",
          "assigned_at": "2026-10-19T15:04:05Z",
          "allowed": true,
          "reason": null
        }
      ]
    }
  ],
  "page": 1,
  "page_size": 20,
  "total_records": 1,
  "total_pages": 1
}
```

| Campo de `employees[]` | Significado |
|---|---|
| `application_user_id`, `display_name`, `reference_email`, `keycloak_user_id`, `person_identity_id` | Usuario operativo (operación 15) |
| `identification_type`, `identification_number`, `first_name`, `middle_name`, `last_name`, `second_last_name` | Persona en Party Identity; `null` si no se pudo consultar |
| `membership_status` | Estado del empleado en la empresa |
| `user_branch_id`, `user_branch_status`, `assigned_at` | Asignación a la sucursal (operación 24) |
| `allowed`, `reason` | Acceso operativo efectivo, como `operational_access` de la operación 23 |

Un empleado asignado a varias sucursales aparece en cada una; una sucursal sin empleados trae `employees: []`. Errores: `404 NOT_FOUND` (empresa o sucursal), `403` si falta alguno de los tres permisos. Dependencias: Party Identity (opcional para la respuesta).

### 35. `POST /companies:onboard`

| | |
|---|---|
| Clasificación | `HIGH_RISK` · crea la organización, la empresa, la persona, la cuenta de Keycloak y el usuario operativo del administrador |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/companies:onboard` |
| Permisos | `perm:operational-context:companies:create` **y** `perm:operational-context:application_users:create` **y** `perm:operational-context:platform:administer` (solo `superadmin`) |
| Keycloak | El token del superadministrador debe traer el rol `manage-users` del cliente `realm-management` (lo tiene el grupo `superadmin`): la cuenta se crea con ese token |
| Disponibilidad | Nueva; disponible tras el próximo despliegue de Operational Context |

Reúne en una sola solicitud los pasos que antes se hacían uno por uno: crear la organización en Party Identity
(operación `POST /organization-identities`), la empresa (operación 4), la persona en Party Identity
(`POST /person-identities`), la cuenta en Keycloak y el usuario operativo (operación 16). Cada pieza que ya
existe se reutiliza (`FOUND`) y la que falta se crea (`CREATED`):

| Pieza | Se reutiliza si | Si no existe |
|---|---|---|
| `organization` | Se envía `organization_identity_id`, o ya hay una organización con el mismo `country_id` + `identification_type` + `identification_number` | Se crea en Party Identity con el token del superadministrador |
| `company` | La organización ya tiene empresa **sin usuarios activos** o con el administrador pedido entre ellos | Se crea con los datos de `company` |
| `person` | Se envía `person_identity_id`, o ya hay una persona con el mismo documento | Se crea en Party Identity |
| `account` | Ya hay una cuenta con ese `username`: **no se cambia su contraseña**; solo se le agregan los grupos que le falten | Se crea habilitada, con contraseña temporal y la acción obligatoria de cambiarla |
| `application_user` | La persona ya es usuario activo de la empresa | Se crea con `display_name` y `reference_email` |

Es todo o nada (Constitución D29). Antes de escribir se revisa que la cuenta existente sirva (no es una cuenta de
servicio ni está deshabilitada). Si un servicio no responde, el paso se reintenta hasta tres veces; un rechazo por
datos no se reintenta. Si un paso falla, se deshace en orden inverso lo que **esta solicitud** creó: la cuenta de
Keycloak (o los grupos que se le agregaron a una cuenta existente), la persona, la empresa y la organización; lo que
ya existía (`FOUND`) no se toca. La respuesta de error trae en `failed_step` el paso que falló, en `onboarding` el
resultado de cada pieza (`ROLLED_BACK` para lo deshecho, `FAILED`, `NOT_STARTED`, `FOUND`) y en `rollback`
`complete`, `undone` y `kept`. El mensaje termina en «no quedó ningún registro guardado» solo si es cierto; si
deshacer también falló, nombra lo que quedó guardado (`kept`, con el motivo). Al corregir la causa se repite la
misma solicitud y empieza desde cero.
Si la empresa ya tiene otros usuarios activos y ninguno es esa persona, responde `409 COMPANY_EXISTS` sin crear
nada: para agregar otro administrador se usan las operaciones de usuarios operativos.

Cuerpo (`CompanyOnboardingRequest`):

| Campo | Tipo | Obligatorio | Validación / valor por defecto | Cómo obtenerlo |
|---|---|---|---|---|
| `organization.organization_identity_id` | UUID \| null | No | Si se envía, los demás campos de `organization` se ignoran | [organization_identity_id](../parametros/organization-identity-id.md) |
| `organization.country_id` | UUID | Sí, sin id | País de registro | [Geography](geography.md) |
| `organization.identification_type` | string | Sí, sin id | 1–40; sigla con regla activa (p. ej. `NIT`) | [Party Identity](party-identity.md) |
| `organization.identification_number` | string | Sí, sin id | 1–40; el NIT va con guion y dígito de verificación (`900123456-8`) | — |
| `organization.legal_name` | string | Sí, sin id | 1–200; al menos una letra | — |
| `organization.trade_name` | string \| null | No | ≤ 200 | — |
| `organization.organization_type` | string \| null | No | ≤ 40 (p. ej. `SAS`) | — |
| `organization.registration_status` | string \| null | No | ≤ 30 (p. ej. `VIGENTE`) | — |
| `organization.website` | string \| null | No | ≤ 300; URL válida | — |
| `company.*` | | No | Los mismos campos de la operación 4 menos `organization_identity_id`: `email`, `phone`, `address_line`, `country_id`, `administrative_division_id`, `city_id`, `postal_code`, `latitude`, `longitude`, `timezone` (`America/Bogota`), `default_currency` (`COP`) | [Geography](geography.md) |
| `admin.person.person_identity_id` | UUID \| null | No | Si se envía, los demás campos de `person` se ignoran | — |
| `admin.person.identification_country_id` | UUID | Sí, sin id | País del documento | [Geography](geography.md) |
| `admin.person.identification_type` | string | Sí, sin id | 1–40 (p. ej. `CC`) | [Party Identity](party-identity.md) |
| `admin.person.identification_number` | string | Sí, sin id | 1–40 | — |
| `admin.person.first_name`, `last_name` | string | Sí, sin id | 1–100; solo letras, espacios, apóstrofo, punto o guion | — |
| `admin.person.middle_name`, `second_last_name` | string \| null | No | ≤ 100 | — |
| `admin.person.birth_date` | date \| null | No | Desde 1900-01-01 y no posterior a hoy | — |
| `admin.person.verification_status` | string | No | `UNVERIFIED` (por defecto), `VERIFIED`, `SUSPENDED` | — |
| `admin.account.username` | string | Sí | 3–60; minúsculas, dígitos, `.`, `_`, `-`, `@`; se guarda en minúsculas | — |
| `admin.account.email` | email | Sí | Único en Keycloak (`409 ACCOUNT_CONFLICT`) | — |
| `admin.account.temporary_password` | string \| null | No | Si se omite, se genera una de 16 caracteres. Política: 12+ caracteres, mayúscula, minúscula y dígito | — |
| `admin.account.groups` | string[] \| null | No | Por defecto `admin`, `Compras`, `Ventas`, `Inventario`. Solo esos cuatro; `superadmin` → `422 GROUP_NOT_ALLOWED` | — |
| `admin.display_name` | string \| null | No | 1–160; por defecto, nombres y apellidos de la persona | — |
| `admin.reference_email` | email \| null | No | Por defecto, `admin.account.email` | — |

Ejemplo de solicitud (los ids de país, departamento y ciudad se toman de Geography antes de enviarla):

```bash
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/companies:onboard' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "organization": {
      "organization_identity_id": null,
      "country_id": "'"$COUNTRY_ID"'",
      "identification_type": "NIT",
      "identification_number": "900123456-8",
      "legal_name": "FERRETERIA EJEMPLO S.A.S.",
      "trade_name": "Ferretería Ejemplo",
      "organization_type": "SAS",
      "registration_status": "VIGENTE",
      "website": "https://ferreteria-ejemplo.test"
    },
    "company": {
      "email": "contacto@ferreteria-ejemplo.test",
      "phone": "+57 602 000 0000",
      "address_line": "Calle 1 # 2-3",
      "country_id": "'"$COUNTRY_ID"'",
      "administrative_division_id": "'"$DIVISION_ID"'",
      "city_id": "'"$CITY_ID"'",
      "postal_code": "190001",
      "latitude": null,
      "longitude": null,
      "timezone": "America/Bogota",
      "default_currency": "COP"
    },
    "admin": {
      "person": {
        "person_identity_id": null,
        "identification_country_id": "'"$COUNTRY_ID"'",
        "identification_type": "CC",
        "identification_number": "1020304050",
        "first_name": "Laura",
        "middle_name": null,
        "last_name": "Gómez",
        "second_last_name": null,
        "birth_date": "1990-05-17",
        "verification_status": "UNVERIFIED"
      },
      "account": {
        "username": "admin.ferreteria",
        "email": "admin@ferreteria-ejemplo.test",
        "temporary_password": null,
        "groups": ["admin", "Compras", "Ventas", "Inventario"]
      },
      "display_name": "Laura Gómez",
      "reference_email": "admin@ferreteria-ejemplo.test"
    }
  }'
```

Respuesta `201` si se creó alguna pieza, `200` si todas ya existían. `Location` apunta a la empresa. La
contraseña temporal **solo** aparece cuando la cuenta se crea en esta solicitud: no se guarda en ningún servicio
ni se escribe en logs o auditoría, así que hay que entregársela al administrador por un canal seguro. Mientras no
la cambie, Keycloak no le entrega token (operación 36).

Ejemplo de respuesta `201` (valores ilustrativos):

```json
{
  "organization": { "result": "CREATED", "organization_identity_id": "0f1e2d3c-4b5a-4968-8776-655443322110" },
  "company": { "result": "CREATED", "company_id": "550e8400-e29b-41d4-a716-446655440000" },
  "person": { "result": "FOUND", "person_identity_id": "6a5b4c3d-2e1f-4a0b-9c8d-7e6f5a4b3c2d" },
  "account": {
    "result": "CREATED",
    "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
    "username": "admin.ferreteria",
    "groups": ["admin", "Compras", "Ventas", "Inventario"],
    "temporary_password": "<contraseña temporal de 16 caracteres>",
    "must_change_password": true
  },
  "application_user": { "result": "CREATED", "application_user_id": "7c6b5a49-3827-4615-a4b3-c2d1e0f9a8b7" },
  "next_step": "Entregue al administrador su usuario y su contraseña temporal por un canal seguro: no se vuelve a mostrar. ..."
}
```

Una cuenta que ya existía responde `"account": {"result": "FOUND", "keycloak_user_id": "…", "username": "…",
"groups": […], "groups_added": […]}`, sin contraseña.

| Respuesta | Causa | Qué hacer |
|---|---|---|
| `403 OPERATIONAL_CONTEXT.FORBIDDEN` | Falta `platform:administer`, o el token no tiene `manage-users` en Keycloak o permiso de alta en Party Identity | Solo `superadmin` |
| `409 …COMPANY_EXISTS` (con `company_id`) | La empresa ya tiene otros usuarios activos | Agregar el usuario con las operaciones 16 y siguientes |
| `409 …ACCOUNT_CONFLICT` | Otra cuenta de Keycloak ya usa ese correo | Usar otro correo |
| `422 …IDENTITY_REJECTED` (con `provider_code` y `provider_errors`) | Party Identity rechazó la organización o la persona (sigla sin regla, NIT con dígito inválido…) | Corregir el campo indicado en `errors[0].field` |
| `422 …ORGANIZATION_IDENTITY_INACTIVE` / `PERSON_IDENTITY_INACTIVE` | Ya existe con ese documento, pero inactiva | Reactivarla en Party Identity |
| `422 …GROUP_NOT_ALLOWED` / `PASSWORD_POLICY` | Grupo fuera de la lista o contraseña temporal débil | Corregir el cuerpo |
| `503` con `dependency` | Party Identity, Geography o Keycloak no respondió | Repetir la misma solicitud |

### 36. `POST /account/initial-password`

| | |
|---|---|
| Clasificación | `HIGH_RISK` · cambia la contraseña de una cuenta de Keycloak |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/account/initial-password` |
| Permisos | Ninguno: es pública, porque la cuenta todavía no puede obtener token |
| Límite | 5 intentos cada 15 minutos por usuario y por IP (`429 TOO_MANY_ATTEMPTS` con `Retry-After`); por debajo sigue la protección contra fuerza bruta de Keycloak (5 fallos bloquean la cuenta un tiempo) |
| Disponibilidad | Nueva; disponible tras el próximo despliegue de Operational Context |

El administrador creado con la operación 35 cambia aquí la contraseña temporal por la definitiva. El servicio
comprueba la contraseña temporal con Keycloak (cliente `sio-web`) y, si es correcta y la cuenta tiene el cambio
pendiente, fija la nueva con su cuenta de servicio (rol `manage-users`, que solo usa para esta operación). Después
el administrador inicia sesión normalmente (guía de [autenticación](../guias/autenticacion.md)).

También puede cambiarla en el navegador, en la página de cuenta de Keycloak
(`https://<keycloak>/realms/sio/account`): al iniciar sesión con la contraseña temporal, Keycloak le pide la nueva.

| Campo | Tipo | Obligatorio | Validación |
|---|---|---|---|
| `username` | string | Sí | El de la cuenta; se compara en minúsculas |
| `temporary_password` | string | Sí | La que entregó la operación 35 |
| `new_password` | string | Sí | 12+ caracteres, mayúscula, minúscula y dígito; distinta de la temporal |

```bash
read -s -p 'Contraseña temporal: ' TEMPORAL; echo
read -s -p 'Contraseña nueva: ' NUEVA; echo
curl -sS -X POST 'https://sio-operational-context-dev.onrender.com/api/v1/account/initial-password' \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "admin.ferreteria",
    "temporary_password": "'"$TEMPORAL"'",
    "new_password": "'"$NUEVA"'"
  }'
```

Respuesta `204` sin cuerpo.

| Respuesta | Causa | Qué hacer |
|---|---|---|
| `401 …INVALID_TEMPORARY_PASSWORD` | Usuario o contraseña temporal incorrectos, o la cuenta ya no tiene el cambio pendiente (el mensaje es el mismo para no revelar cuál) | Revisar los datos; si ya la cambió, iniciar sesión con la nueva |
| `422 …PASSWORD_POLICY` | La contraseña nueva no cumple la política o es igual a la temporal | El `detail` dice qué regla falta |
| `429 …TOO_MANY_ATTEMPTS` | Más de 5 intentos en 15 minutos | Esperar los segundos de `Retry-After` |
| `503` con `dependency` `keycloak` | Keycloak no respondió o el cliente `sio-web` no admite el flujo de contraseña | Reintentar; revisar la configuración del realm |

### 37. `POST /companies/{company_id}/employees:onboard`

| | |
|---|---|
| Clasificación | `HIGH_RISK` · crea la persona, la cuenta de Keycloak y el usuario operativo de un empleado |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/companies/{company_id}/employees:onboard` |
| Permisos | `perm:operational-context:application_users:create`, y membresía activa en la empresa o `platform:administer` (`superadmin` y `admin` de la empresa) |
| Party Identity | La persona se crea con el token de quien llama (el grupo `admin` tiene alta de personas) |
| Keycloak | La cuenta la crea la **cuenta de servicio** de Operational Context (roles `manage-users`, `view-users` y `query-groups` de `realm-management`); quien llama no necesita permisos en Keycloak |
| Disponibilidad | Nueva; disponible tras el próximo despliegue de Operational Context |

Reúne en una sola solicitud «crear la persona» (`POST /person-identities` de Party Identity), «crear la cuenta
en Keycloak» (que antes se hacía a mano) y «hacerla empleada» (operación 16). Cada pieza que ya existe se
reutiliza (`FOUND`) y la que falta se crea (`CREATED`):

| Pieza | Se reutiliza si | Si no existe |
|---|---|---|
| `person` | Se envía `person_identity_id`, o ya hay una persona con el mismo documento | Se crea en Party Identity |
| `account` | Ya hay una cuenta con ese `username`: **no se cambia su contraseña**; solo se le agregan los grupos que le falten (con los límites de abajo) | Se crea habilitada, con contraseña temporal y la acción obligatoria de cambiarla |
| `application_user` | La persona ya es usuario activo de la empresa | Se crea con `display_name` y `reference_email` |

Tres datos los fija **siempre el servidor**; el cliente no los envía (si llegan, `422`):

| Dato | Valor |
|---|---|
| Contraseña | Temporal, generada de 16 caracteres (política del realm), con cambio obligatorio al iniciar sesión |
| Grupos de la cuenta | Los del operario: `Compras`, `Ventas` e `Inventario` |
| `verification_status` de la persona | `UNVERIFIED` |

Límites, porque la cuenta la crea la identidad técnica del servicio:

- Nunca se asignan `admin` ni `superadmin`.
- La empresa se comprueba antes de crear nada: una empresa sin membresía de quien llama responde `404`, igual
  que un id que no existe.
- Sobre una cuenta que ya existe, el administrador de la empresa recibe `409 ACCOUNT_CONFLICT` si la cuenta
  está en el grupo `admin` o `superadmin`, o si trabaja en otra empresa y le faltan grupos (los grupos valen en
  todas sus empresas). En esos casos la registra un superadministrador.

Es todo o nada (Constitución D29), igual que el alta de la empresa: se revisa la cuenta antes de escribir, las
fallas pasajeras se reintentan y, si un paso falla, se deshace lo que esta solicitud creó (la cuenta o los grupos
agregados, y la persona). El error trae `failed_step`, `onboarding` (con `ROLLED_BACK`) y `rollback`; si deshacer
también falló, el mensaje dice qué quedó guardado.

Parámetro de ruta: `company_id` ([company_id](../parametros/company-id.md)).

Cuerpo (`EmployeeOnboardingRequest`):

| Campo | Tipo | Obligatorio | Validación / valor por defecto | Cómo obtenerlo |
|---|---|---|---|---|
| `person.*` | | Sí | Los mismos campos y reglas de `admin.person` de la operación 35 (`person_identity_id`, o documento, nombres, apellidos y `birth_date`), **sin** `verification_status` | [Party Identity](party-identity.md) |
| `account.username` | string | Sí | 3–60; minúsculas, dígitos, `.`, `_`, `-`, `@`; se guarda en minúsculas | — |
| `account.email` | email | Sí | Único en Keycloak (`409 ACCOUNT_CONFLICT`) | — |
| `display_name` | string \| null | No | 1–160; por defecto, nombres y apellidos de la persona | — |
| `reference_email` | email \| null | No | Por defecto, `account.email` | — |

Ejemplo de solicitud (la hace el administrador de la empresa con su propio token):

```bash
curl -sS -X POST "https://sio-operational-context-dev.onrender.com/api/v1/companies/$COMPANY_ID/employees:onboard" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "person": {
      "person_identity_id": null,
      "identification_country_id": "'"$COUNTRY_ID"'",
      "identification_type": "CC",
      "identification_number": "1000000002",
      "first_name": "Operario",
      "middle_name": null,
      "last_name": "Ejemplo",
      "second_last_name": null,
      "birth_date": "1995-08-20"
    },
    "account": {
      "username": "operario.ejemplo",
      "email": "operario@ferreteria-ejemplo.test"
    },
    "display_name": "Operario Ejemplo",
    "reference_email": "operario@ferreteria-ejemplo.test"
  }'
```

Respuesta `201` si se creó alguna pieza, `200` si todas ya existían. `Location` apunta al usuario operativo
(operación 17). Cuando la cuenta se crea, la respuesta trae **su usuario** (`account.username`) y **su contraseña
temporal** (`account.temporary_password`) una sola vez: no se guarda en ningún servicio ni se escribe en logs o
auditoría, así que hay que entregársela al empleado por un canal seguro. El empleado la cambia en su primer
ingreso (operación 36).

Ejemplo de respuesta `201` (valores ilustrativos):

```json
{
  "person": { "result": "CREATED", "person_identity_id": "6a5b4c3d-2e1f-4a0b-9c8d-7e6f5a4b3c2d" },
  "account": {
    "result": "CREATED",
    "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
    "username": "operario.ejemplo",
    "groups": ["Compras", "Ventas", "Inventario"],
    "temporary_password": "<contraseña temporal de 16 caracteres>",
    "must_change_password": true
  },
  "application_user": { "result": "CREATED", "application_user_id": "7c6b5a49-3827-4615-a4b3-c2d1e0f9a8b7" },
  "next_step": "Entregue al empleado su usuario (account.username) y su contraseña temporal (account.temporary_password) por un canal seguro: no se vuelve a mostrar. ..."
}
```

Una cuenta que ya existía responde `"account": {"result": "FOUND", "keycloak_user_id": "…", "username": "…",
"groups": […], "groups_added": […]}`, sin contraseña.

| Respuesta | Causa | Qué hacer |
|---|---|---|
| `403 OPERATIONAL_CONTEXT.FORBIDDEN` | Falta `application_users:create`, o el token no tiene alta de personas en Party Identity | Lo hacen `superadmin` y `admin` |
| `404 …NOT_FOUND` | La empresa no existe o quien llama no es miembro de ella | Usar una empresa propia |
| `409 …ACCOUNT_CONFLICT` | Otra cuenta ya usa ese correo, o la cuenta existente es de un administrador o trabaja en otra empresa y le faltan grupos | Usar otro correo o usuario, enviar solo los grupos que ya tiene, o pedírselo a un superadministrador |
| `409 …DUPLICATE_MEMBERSHIP` | Esa cuenta ya es usuario activo de la empresa con otra persona | Revisar el usuario y el documento |
| `422 …IDENTITY_REJECTED` (con `provider_code`) | Party Identity rechazó la persona | Corregir el campo indicado en `errors[0].field` (`person.…`) |
| `422 …PERSON_IDENTITY_INACTIVE` | Ya existe con ese documento, pero inactiva | Reactivarla en Party Identity |
| `422 …VALIDATION_ERROR` | Se envió `temporary_password`, `groups` o `verification_status`, que fija el servidor, u otro campo desconocido | Quitarlos del cuerpo |
| `503` con `dependency` | Party Identity o Keycloak no respondió | Repetir la misma solicitud |

### 38. `PUT /application-users/{application_user_id}/employee`

| | |
|---|---|
| Clasificación | `HIGH_RISK` · cambia la persona, la cuenta de Keycloak y el usuario operativo de un empleado |
| URL | `https://sio-operational-context-dev.onrender.com/api/v1/application-users/{application_user_id}/employee` |
| Permisos | `perm:operational-context:application_users:update`, y membresía activa en la empresa del empleado o `platform:administer` (`superadmin` y `admin` de la empresa) |
| Party Identity | La persona se cambia con el token de quien llama (el grupo `admin` puede modificar personas) |
| Keycloak | La cuenta la cambia la **cuenta de servicio** de Operational Context (`manage-users` y `view-users` de `realm-management`) |
| Disponibilidad | Nueva; disponible tras el próximo despliegue de Operational Context |

Reúne «Actualizar la persona» (`PUT /person-identities/{id}` de Party Identity) y «Actualizar el usuario
operativo» (operación 18), y además mantiene al día la copia que guarda Keycloak. Es un `PUT` parcial: solo
cambia lo enviado.

| Si se envía | Cambia en Party Identity | Cambia en Keycloak | Cambia en el usuario operativo |
|---|---|---|---|
| `person.first_name`, `middle_name`, `last_name` o `second_last_name` | La persona | `firstName` (nombres) y `lastName` (apellidos), con los nombres que quedaron en Party Identity | Nada (`display_name` no se recalcula) |
| `person.identification_*` o `person.birth_date` | La persona | Nada | Nada |
| `display_name` | — | Nada | El nombre visible |
| `reference_email` | — | El correo de la cuenta (queda sin verificar) | El correo de referencia |

Antes de cambiar nada se comprueba que la cuenta se pueda tocar. Un administrador de empresa recibe
`409 ACCOUNT_CONFLICT` si la cuenta del empleado es de otro administrador (la suya sí la puede cambiar) o si el
empleado también trabaja en otra empresa, porque el nombre y el correo de la cuenta valen en todas: en esos casos
lo hace un superadministrador. `verification_status`, el usuario de Keycloak (`username`) y el estado no se
cambian aquí.

Es todo o nada (Constitución D29): la cuenta se revisa antes de escribir, las fallas pasajeras se reintentan hasta
tres veces y, si un paso falla, la persona y la cuenta de Keycloak vuelven a sus valores anteriores. La respuesta de
error trae `failed_step`, `employee_update` con el resultado de cada pieza (`ROLLED_BACK`, `UNCHANGED`, `FAILED`,
`NOT_STARTED`) y `rollback` (`complete`, `undone`, `kept`); si deshacer también falló, el mensaje dice qué quedó
cambiado.

Parámetro de ruta: `application_user_id` ([application_user_id](../parametros/application-user-id.md)).

Cuerpo (`EmployeeUpdateRequest`, al menos un campo):

| Campo | Tipo | Validación |
|---|---|---|
| `person.identification_country_id`, `identification_type`, `identification_number` | UUID / string | No admiten `null`; el número debe cumplir la regla del tipo (`422 IDENTITY_REJECTED`) |
| `person.first_name`, `person.last_name` | string | 1–100; no admiten `null` |
| `person.middle_name`, `person.second_last_name` | string \| null | ≤ 100; `null` los borra |
| `person.birth_date` | date \| null | Desde 1900-01-01 y no posterior a hoy |
| `display_name` | string | 1–160; no admite `null` |
| `reference_email` | email \| null | Si se envía con valor, también reemplaza el correo de Keycloak; único en Keycloak (`409 ACCOUNT_CONFLICT`) |

Ejemplo de solicitud:

```bash
curl -sS -X PUT "https://sio-operational-context-dev.onrender.com/api/v1/application-users/$OPERARIO_USER_ID/employee" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "person": {
      "middle_name": "Andrés"
    },
    "display_name": "Operario Ejemplo - Caja 1",
    "reference_email": "caja1@ferreteria-ejemplo.test"
  }'
```

Ejemplo de respuesta `200` (valores ilustrativos):

```json
{
  "person": { "result": "UPDATED", "person_identity_id": "6a5b4c3d-2e1f-4a0b-9c8d-7e6f5a4b3c2d", "fields": ["middle_name"] },
  "account": {
    "result": "UPDATED",
    "keycloak_user_id": "3f2a9c1e-7b4d-4e8a-9c6f-1d2e3f4a5b6c",
    "username": "operario.ejemplo",
    "fields": ["first_name", "last_name", "email"]
  },
  "application_user": {
    "result": "UPDATED",
    "application_user_id": "7c6b5a49-3827-4615-a4b3-c2d1e0f9a8b7",
    "fields": ["display_name", "reference_email"]
  }
}
```

| Respuesta | Causa | Qué hacer |
|---|---|---|
| `403 OPERATIONAL_CONTEXT.FORBIDDEN` | Falta `application_users:update`, o el token no puede modificar personas en Party Identity | Lo hacen `superadmin` y `admin` |
| `404 …NOT_FOUND` | El usuario operativo no existe o es de otra empresa | Usar un empleado propio |
| `409 …ACCOUNT_CONFLICT` | Otra cuenta ya usa ese correo, o la cuenta es de otro administrador o trabaja en otra empresa | Usar otro correo, o pedírselo a un superadministrador |
| `422 …IDENTITY_REJECTED` (con `provider_code`) | Party Identity rechazó la persona | Corregir el campo de `errors[0].field` (`person.…`) |
| `422 …KEYCLOAK_USER_NOT_FOUND` | La cuenta del empleado ya no existe en Keycloak | Revisar la cuenta |
| `422 …VALIDATION_ERROR` | Cuerpo vacío, `null` donde no se admite, o campos que no se cambian aquí (`verification_status`, `username`, `version`…) | Corregir el cuerpo |
| `503` con `dependency` | Party Identity o Keycloak no respondió | Repetir la misma solicitud |

## Errores frecuentes

| Respuesta | Causa | Qué hacer |
|---|---|---|
| `404 OPERATIONAL_CONTEXT.NOT_FOUND` «La empresa solicitada no existe.» | UUID inexistente o sin membresía | Comprobar `/me/operational-context` |
| `403 OPERATIONAL_CONTEXT.FORBIDDEN` | Falta el permiso (p. ej. `platform:administer` al crear empresas) | Solo el grupo `superadmin` |
| `409 …DUPLICATE_BRANCH_CODE` / `DUPLICATE_MEMBERSHIP` / `DUPLICATE_USER_BRANCH` / `DUPLICATE_ORGANIZATION_IDENTITY` | Unicidad | Buscar el registro existente |
| `409 …VERSION_CONFLICT` / `INVALID_STATUS_TRANSITION` | Concurrencia o estado | Recargar |
| `422 …KEYCLOAK_USER_NOT_FOUND` / `KEYCLOAK_USER_DISABLED` / `TECHNICAL_ACCOUNT_NOT_ALLOWED` | La cuenta no existe, está deshabilitada o es técnica | Crear o habilitar la cuenta en Keycloak |
| `422 …PERSON_IDENTITY_NOT_FOUND` / `ORGANIZATION_IDENTITY_NOT_FOUND` / `_INACTIVE` | Identidad inválida en Party Identity | Crear o reactivar la identidad |
| `422 …GEOGRAPHY_INCOHERENT` / `LOCATION_INACTIVE` / `*_NOT_FOUND` | Ubicación inválida | Usar selectores en cascada de Geography |
| `422 …IMMUTABLE_FIELD` | Se envió un campo inmutable | Quitarlo del cuerpo |
| `503` con `dependency` `keycloak`, `party-identity` o `geography` | Dependencia caída | Reintentar |

## Dependencias

| Servicio | Uso |
|---|---|
| Party Identity | Crear o buscar por documento la organización y la persona en el alta de empresa (operación 35) y la persona en el registro y la actualización de empleados (operaciones 37 y 38). Validar identidades de organización (empresas) y de persona (usuarios, asesores), y leer nombre e identificación para `/me/operational-context` y el listado de empleados por sucursal (sin guardarlos), con el token del usuario |
| Geography | Validar país, división y ciudad activos y coherentes, con el token del usuario |
| Purchase, Sales | Dejar la numeración automática de la sucursal nueva (operación 10, `:ensure-branch`), con el token del usuario; si fallan, la sucursal se crea igual |
| Keycloak (Admin API) | Verificar la cuenta al crear o reactivar usuarios operativos y leer roles efectivos en la vía técnica, con la cuenta de servicio del propio Operational Context. En el alta de empresa (operación 35) crea la cuenta con el token del superadministrador; en el registro de empleados (operación 37) la cuenta de servicio crea la cuenta del empleado (solo grupos del operario) y en su actualización (operación 38) le copia nombres y correo; en el cambio de contraseña inicial (operación 36) la cuenta de servicio fija la contraseña nueva |
| Audit | Eventos de auditoría |

## Cobertura

| Verificación | Resultado |
|---|---|
| Operaciones en el OpenAPI desplegado | 33 (la 34 se agrega en el próximo despliegue) |
| Operaciones documentadas | 38 (de la 35 a la 38 se agregan en el próximo despliegue) |
| Permisos contrastados con el código | Sí (`api/companies.py`, `branches.py`, `application_users.py`, `user_branches.py`, `branch_employees.py`, `advisor_assignments.py`, `operational_access.py`) |
| Esquemas contrastados | Sí (`api/schemas/*.py`, `application/access_scope.py`, `domain/access.py`, `api/errors.py`) |
| Verificado | En asesores, el inicio de vigencia es el valor enviado o el instante actual; nunca puede ser futuro y debe llevar zona horaria |
