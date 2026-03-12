## Contrato de API — Cuentas Bancarias (v1)
Este documento complementa las guías generales de AGENTS.md y es la fuente de verdad para el contrato de esta API

### Convenciones generales

- **Base URL**: `/api/v1`
- **Formato**: JSON en requests y responses.
- **Autenticación**: No disponible en esta versión.
- **Códigos HTTP**:
  - `200 OK`: operación exitosa con respuesta.
  - `201 Created`: recurso creado.
  - `204 No Content`: operación exitosa sin cuerpo.
  - `400 Bad Request`: error de validación o parámetros inválidos.
  - `404 Not Found`: recurso no encontrado.
  - `409 Conflict`: conflicto de negocio (ej. saldo insuficiente).
  - `500 Internal Server Error`: error inesperado del servidor.

### Validaciones
- **Datos de entrada**: todos los datos de entrada deberan verficarse en un json schema para cada endpoint

### Modelo de error estándar

Todas las respuestas de error deben seguir la misma estructura:

```json
{
  "code": "BUSINESS_RULE_VIOLATION",
  "message": "Saldo insuficiente para realizar el egreso",
  "details": [
    "accountId: 123",
    "attemptedAmount: 1000.00",
    "availableLimit: 500.00"
  ],
  "timestamp": "2026-03-12T10:15:30Z"
}
```

- `code`: código de error legible por máquina (ej. `VALIDATION_ERROR`, `NOT_FOUND`, `INSUFFICIENT_FUNDS`).
- `message`: mensaje entendible por humanos (localizable).
- `details`: lista opcional con mensajes adicionales o errores de campo.
- `timestamp`: fecha/hora del error en ISO-8601.

---

### Recursos principales

- `/clients` — Gestión de clientes.
- `/accounts` — Gestión de cuentas.
- `/accounts/{accountId}/movements` — Registro y consulta de movimientos de una cuenta.

---

## DTOs — Objetos de transferencia

### Cliente

**ClienteRequest**

```json
{
  "documentId": "12345678",
  "fullName": "Juan Pérez",
  "email": "juan.perez@example.com",
  "phone": "+54-11-5555-5555",
  "address": "Calle Falsa 123"
}
```

**ClienteResponse**

```json
{
  "id": "cli_001",
  "documentId": "12345678",
  "fullName": "Juan Pérez",
  "email": "juan.perez@example.com",
  "phone": "+54-11-5555-5555",
  "address": "Calle Falsa 123",
  "createdAt": "2026-03-12T10:15:30Z"
}
```

### Cuenta bancaria

**AccountType**

- `CAJA_AHORRO`
- `CUENTA_CORRIENTE`

**AccountStatus**

- `ACTIVA`
- `BLOQUEADA`
- `CERRADA`

**AccountRequest**

```json
{
  "clientId": "cli_001",
  "type": "CUENTA_CORRIENTE",
  "currency": "ARS",
  "initialBalance": 0.00,
  "overdraftLimit": 10000.00
}
```

- `initialBalance`: opcional, por defecto `0.00`.
- `overdraftLimit`: requerido solo para `CUENTA_CORRIENTE`, ignorado o `0` para `CAJA_AHORRO`.

**AccountResponse**

```json
{
  "id": "acc_001",
  "accountNumber": "0001-00000001",
  "clientId": "cli_001",
  "type": "CUENTA_CORRIENTE",
  "currency": "ARS",
  "balance": 500.00,
  "overdraftLimit": 10000.00,
  "status": "ACTIVA",
  "createdAt": "2026-03-12T10:15:30Z"
}
```

### Movimiento

**MovementType**

- `INGRESO`
- `EGRESO`

**MovementRequest**

```json
{
  "type": "EGRESO",
  "amount": 1500.50,
  "description": "Pago de servicio"
}
```

**MovementResponse**

```json
{
  "id": "mov_001",
  "accountId": "acc_001",
  "type": "EGRESO",
  "amount": 1500.50,
  "description": "Pago de servicio",
  "createdAt": "2026-03-12T11:00:00Z",
  "balanceAfterMovement": -500.50
}
```

---

## Endpoints — Clientes

### Crear cliente

- **POST** `/api/v1/clients`
- **Descripción**: Crea un nuevo cliente.
- **Request body**: `ClienteRequest`.
- **Responses**:
  - `201 Created` — cuerpo `ClienteResponse`.
  - `400 Bad Request` — validación fallida (ej. email inválido, documento duplicado).

### Obtener cliente por id

- **GET** `/api/v1/clients/{clientId}`
- **Responses**:
  - `200 OK` — cuerpo `ClienteResponse`.
  - `404 Not Found` — cliente inexistente.

### Listar clientes

- **GET** `/api/v1/clients`
- **Query params**:
  - `page` (opcional, default 0).
  - `size` (opcional, default 20).
- **Responses**:
  - `200 OK` — lista paginada de `ClienteResponse`.

---

## Endpoints — Cuentas

### Crear cuenta

- **POST** `/api/v1/accounts`
- **Descripción**: Crea una nueva cuenta bancaria asociada a un cliente existente.
- **Request body**: `AccountRequest`.
- **Responses**:
  - `201 Created` — cuerpo `AccountResponse`.
  - `400 Bad Request` — datos inválidos (tipo de cuenta inválido, `overdraftLimit` negativo, etc.).
  - `404 Not Found` — `clientId` inexistente.

### Obtener cuenta por id

- **GET** `/api/v1/accounts/{accountId}`
- **Responses**:
  - `200 OK` — cuerpo `AccountResponse`.
  - `404 Not Found` — cuenta inexistente.

### Listar cuentas de un cliente

- **GET** `/api/v1/clients/{clientId}/accounts`
- **Responses**:
  - `200 OK` — lista de `AccountResponse`.
  - `404 Not Found` — cliente inexistente.

### Actualizar límite de descubierto (cuenta corriente)

- **PATCH** `/api/v1/accounts/{accountId}/overdraft`
- **Descripción**: Actualiza el límite de descubierto de una cuenta corriente.
- **Request body**:

```json
{
  "overdraftLimit": 20000.00
}
```

- **Responses**:
  - `200 OK` — cuerpo `AccountResponse` actualizado.
  - `400 Bad Request` — valor inválido (negativo, etc.).
  - `404 Not Found` — cuenta inexistente.
  - `409 Conflict` — la cuenta no es de tipo `CUENTA_CORRIENTE`.

---

## Endpoints — Movimientos

### Registrar movimiento en una cuenta

- **POST** `/api/v1/accounts/{accountId}/movements`
- **Descripción**: Registra un movimiento de ingreso o egreso sobre la cuenta.
- **Request body**: `MovementRequest`.
- **Reglas de negocio**:
  - Si `type = INGRESO`: se suma al saldo.
  - Si `type = EGRESO`:
    - Caja de ahorro: no permite saldo negativo → si importe > saldo, devolver `409 Conflict` (`INSUFFICIENT_FUNDS`).
    - Cuenta corriente: permite saldo negativo hasta \(-overdraftLimit\) → si se supera, devolver `409 Conflict` (`OVERDRAFT_LIMIT_EXCEEDED`).
- **Responses**:
  - `201 Created` — cuerpo `MovementResponse`.
  - `400 Bad Request` — validación de request.
  - `404 Not Found` — cuenta inexistente.
  - `409 Conflict` — violación de reglas de negocio (saldo/descubierto).

### Listar movimientos de una cuenta

- **GET** `/api/v1/accounts/{accountId}/movements`
- **Query params**:
  - `page` (opcional, default 0).
  - `size` (opcional, default 20).
  - `from` (opcional, fecha ISO-8601).
  - `to` (opcional, fecha ISO-8601).
- **Responses**:
  - `200 OK` — lista paginada de `MovementResponse`.
  - `404 Not Found` — cuenta inexistente.

---

## Consideraciones adicionales

- **Idempotencia**:
  - Se podría agregar un `requestId` opcional en la creación de movimientos para evitar duplicados en caso de reintentos (no obligatorio en esta primera versión, pero recomendado).

- **Auditoría**:
  - El modelo podría ampliarse con información de usuario que realizó la operación (ej. `performedBy`), según el esquema de autenticación que se defina más adelante.

- **Internacionalización**:
  - Los mensajes de error (`message`) deben poder traducirse; el `code` debe ser estable para clientes.

