## Arquitectura — API de Cuentas Bancarias
Este documento complementa las guías generales de AGENTS.md y es la fuente de verdad para la arquitectura de esta API

### Visión general

La aplicación será una **API REST Java** organizada en **tres capas** siguiendo el patrón **MVC**:

- **Capa de presentación (Controller / API)** — expone los endpoints REST, mapea DTOs, maneja códigos HTTP.
- **Capa de servicio (Service / Dominio)** — contiene la lógica de negocio (reglas de cuentas, movimientos, descubierto).
- **Capa de acceso a datos (Repository / Persistence)** — se encarga de la persistencia de entidades (BD relacional u otro almacenamiento).

Se recomienda usar un framework tipo Spring Boot (o similar) para simplificar configuración, inyección de dependencias y mapeo REST, aunque este documento es agnóstico al framework específico.

---

### Paquetes sugeridos

Estructura de paquetes a modo de guía:

- `com.banking.api.controller`
  - Controladores REST (`ClientController`, `AccountController`, `MovementController`).
- `com.banking.api.dto`
  - DTOs de entrada/salida (`ClientRequest`, `ClientResponse`, `AccountRequest`, `AccountResponse`, `MovementRequest`, `MovementResponse`, errores).
- `com.banking.domain.model`
  - Entidades de dominio (`Client`, `Account`, `Movement`, enums de tipo de cuenta, estado, tipo de movimiento).
- `com.banking.domain.service`
  - Servicios de negocio (`ClientService`, `AccountService`, `MovementService`).
- `com.banking.domain.exception`
  - Excepciones de dominio (`ClientNotFoundException`, `AccountNotFoundException`, `InsufficientFundsException`, `OverdraftLimitExceededException`, etc.).
- `com.banking.infrastructure.repository`
  - Repositorios (interfaces y adaptadores a la tecnología de persistencia).
- `com.banking.infrastructure.config`
  - Configuración técnica (BD, mapeos, etc.) si aplica.

---

### Capa Controller (Presentación)

- Responsabilidades:
  - Exponer los endpoints definidos en `api-contract.md`.
  - Recibir requests JSON → mapear a DTOs de entrada.
  - Delegar la lógica de negocio a servicios.
  - Mapear respuestas del dominio a DTOs de salida.
  - Devolver códigos HTTP correctos (`200`, `201`, `204`, `400`, `404`, `409`, etc.).
- No debe:
  - Contener lógica de negocio compleja.
  - Acceder directamente a la capa de persistencia.

Se recomienda un manejo centralizado de errores (ej. un `@ControllerAdvice` o equivalente) para mapear excepciones de dominio al modelo de error estándar definido en `api-contract.md`.

---

### Capa Service (Dominio / Negocio)

- Responsabilidades:
  - Implementar reglas de negocio de:
    - Creación de clientes.
    - Apertura de cuentas.
    - Validación de operaciones de ingreso/egreso.
    - Cálculo y validación de saldo/descubierto.
    - Cambio de límite de descubierto.
  - Orquestar operaciones entre repositorios (ej. lectura de cuenta, verificación de cliente, registro de movimiento y actualización de saldo).
  - Emitir excepciones de dominio cuando se violan reglas (ej. saldo insuficiente, cuenta no encontrada).

- Ejemplos de lógica clave:
  - **Caja de ahorro**:
    - Antes de un `EGRESO`, verificar `saldoActual >= monto`; si no, lanzar `InsufficientFundsException`.
  - **Cuenta corriente**:
    - Antes de un `EGRESO`, calcular `nuevoSaldo = saldoActual - monto` y verificar que `nuevoSaldo >= -overdraftLimit`; si no, lanzar `OverdraftLimitExceededException`.
  - En ambos casos, registrar un `Movement` y actualizar el saldo de la cuenta de forma atómica.

La capa de servicio debe estar bien testeada con pruebas unitarias de las reglas de negocio (idealmente).

---

### Capa Repository (Persistencia)

- Responsabilidades:
  - Proveer interfaces para operaciones CRUD sobre:
    - Clientes.
    - Cuentas.
    - Movimientos.
  - Encapsular detalles de implementación (ORM, SQL, NoSQL, etc.).
  - Garantizar consistencia de datos al registrar movimientos y actualizar saldos (transacciones).

Se recomienda usar:

- Una base de datos relacional (ej. PostgreSQL, MySQL) con al menos las siguientes tablas lógicas:
  - `clients` (id, document_id, full_name, email, phone, address, timestamps…).
  - `accounts` (id, account_number, client_id, type, currency, balance, overdraft_limit, status, timestamps…).
  - `movements` (id, account_id, type, amount, description, created_at, balance_after_movement…).

---

### Flujo principal — Registrar movimiento de egreso

1. El cliente de la API envía `POST /api/v1/accounts/{accountId}/movements` con un `MovementRequest` tipo `EGRESO`.
2. El controlador:
   - Valida el request.
   - Llama a `MovementService.registerMovement(accountId, request)`.
3. El servicio:
   - Recupera la cuenta desde `AccountRepository`.
   - Verifica tipo de cuenta y reglas de saldo/descubierto.
   - Calcula el nuevo saldo y construye un `Movement`.
   - Usa una transacción para:
     - Guardar el `Movement` en `MovementRepository`.
     - Actualizar el saldo de la cuenta en `AccountRepository`.
   - Devuelve el `Movement` de dominio.
4. El controlador mapea a `MovementResponse` y devuelve `201 Created`.
5. Si se viola alguna regla (saldo insuficiente, descubierto excedido), el servicio lanza una excepción específica:
   - El handler global mapea a `409 Conflict` con el modelo de error estándar.

---

### Versionado y extensibilidad

- El prefijo `/api/v1` permite introducir `/api/v2` en el futuro sin romper clientes existentes.
- El modelo de dominio y DTOs se diseñan para poder:
  - Agregar nuevos tipos de cuenta (ej. plazos fijos) sin romper esquemas actuales.
  - Incorporar operaciones adicionales (ej. transferencias entre cuentas, débitos automáticos) agregando nuevos servicios y endpoints.

---

### Consideraciones adicionales

- **Logging**:
  - Registrar operaciones relevantes (creación de cuentas, movimientos) con un nivel adecuado de detalle, cuidando no loguear datos sensibles.

- **Validación**:
  - Usar Bean Validation (o mecanismo equivalente) en DTOs y entidades donde aplique, de modo que los controladores reciban objetos ya validados.

- **Pruebas**:
  - Tests unitarios para la capa de servicio (reglas de negocio).
  - Tests de integración para los flujos principales de API (crear cliente, abrir cuenta, registrar movimientos, consultar).

