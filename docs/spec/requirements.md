## Requisitos de negocio — API de Cuentas Bancarias
Este documento complementa las guías generales de AGENTS.md y es la fuente de verdad para los requerimientos / historias de usuario de esta API

### Contexto general

El sistema expone una API REST para la gestión de **clientes**, **cuentas bancarias** y **movimientos**.  
El objetivo es permitir a aplicaciones externas (ej. frontend web, mobile, integraciones) crear y consultar clientes, abrir cuentas y registrar movimientos de ingreso/egreso de dinero.

### Entidades principales

- **Cliente**
  - Identificador único.
  - Datos personales mínimos: nombre completo, documento de identidad, email, teléfono (opcional), dirección (opcional).
  - Relación: un cliente puede tener **múltiples cuentas**.

- **Cuenta bancaria**
  - Identificador único.
  - Número de cuenta legible (ej. string con formato configurable).
  - Tipo de cuenta:
    - `CAJA_AHORRO`.
    - `CUENTA_CORRIENTE`.
  - Saldo actual.
  - Moneda (ej. `ARS`, `USD`) — se puede comenzar con una sola moneda y extender luego.
  - Estado: activa, bloqueada, cerrada (al menos activa/inactiva).
  - Relación con cliente (propietario principal; se puede extender a cotitulares en el futuro).
  - Para cuentas corrientes:
    - Monto de **descubierto** configurable (límite máximo de saldo negativo permitido).

- **Movimiento**
  - Identificador único.
  - Referencia a una cuenta.
  - Tipo de movimiento: `INGRESO` (crédito) o `EGRESO` (débito).
  - Importe (positivo, con validación de decimales).
  - Fecha/hora de registro.
  - Descripción u observación opcional.
  - Saldo resultante luego de aplicar el movimiento (para trazabilidad).

### Reglas funcionales

- **Creación de clientes**
  - Debe ser posible crear un cliente con datos mínimos válidos.
  - No se deben permitir documentos de identidad duplicados (regla de negocio configurable).

- **Creación de cuentas**
  - Cada cuenta debe asociarse a **un cliente existente**.
  - Se debe validar el tipo de cuenta (`CAJA_AHORRO` o `CUENTA_CORRIENTE`).
  - El saldo inicial puede ser:
    - Cero por defecto.
    - Opcionalmente configurable en la petición (si se permite).
  - Para cuentas corrientes:
    - El descubierto permitido debe configurarse al crear la cuenta (puede modificarse luego con otro endpoint).
    - El saldo mínimo permitido es \(-descubierto\).

- **Movimientos de ingreso**
  - Un movimiento de tipo **INGRESO** aumenta el saldo de la cuenta.
  - Debe quedar registrado con fecha/hora y descripción opcional.

- **Movimientos de egreso**
  - Un movimiento de tipo **EGRESO** disminuye el saldo.
  - Para **caja de ahorro**:
    - No se permite que el saldo resulte negativo. Si el importe supera el saldo disponible, el movimiento debe ser rechazado.
  - Para **cuenta corriente**:
    - Se permite saldo negativo hasta el límite de descubierto configurado.
    - Si el movimiento genera un saldo inferior al límite permitido (más negativo que \(-descubierto\)), el movimiento debe ser rechazado.

- **Consulta de información**
  - Debe ser posible:
    - Listar clientes.
    - Obtener detalle de un cliente con sus cuentas.
    - Listar cuentas de un cliente.
    - Consultar detalle de una cuenta.
    - Consultar el historial de movimientos de una cuenta (con paginación básica).

### Requisitos no funcionales (mínimos)

- **Seguridad**
  - La autenticación/autorización no se detalla en esta versión del spec, pero la API debe diseñarse de forma que pueda integrarse fácilmente con mecanismos estándar (ej. JWT, OAuth2).
  - No exponer datos sensibles innecesarios en las respuestas.

- **Validación y errores**
  - Las entradas deben validarse (campos obligatorios, formatos, rangos).
  - Los errores deben devolverse con mensajes claros y códigos HTTP consistentes (ver `api-contract.md`).

- **Versionado**
  - La API debe estar versionada (ej. prefijo `/api/v1` en las rutas).

- **Extensibilidad**
  - El diseño debe permitir:
    - Agregar nuevos tipos de cuenta.
    - Agregar nuevas operaciones (ej. transferencias entre cuentas) sin romper contratos existentes.

### Historias de usuario (ejemplos)

- **HU-001 — Alta de cliente**
  - Como *operador del sistema*  
    Quiero registrar un nuevo cliente con sus datos básicos  
    Para poder asociarle cuentas bancarias y operar con ellas.

- **HU-002 — Apertura de cuenta**
  - Como *operador del sistema*  
    Quiero crear una cuenta bancaria de tipo caja de ahorro o cuenta corriente para un cliente existente  
    Para que ese cliente pueda operar con depósitos y extracciones.

- **HU-003 — Ingreso de dinero**
  - Como *aplicación de front-office*  
    Quiero registrar un movimiento de ingreso en una cuenta  
    Para reflejar depósitos de efectivo o transferencias recibidas.

- **HU-004 — Egreso de dinero con validación de saldo**
  - Como *aplicación de front-office*  
    Quiero registrar un movimiento de egreso en una cuenta  
    Para registrar extracciones y pagos,  
    Y que el sistema valide que no se sobrepase el saldo (en caja de ahorro) o el descubierto (en cuenta corriente).

- **HU-005 — Consulta de movimientos**
  - Como *cliente* (vía canal digital)  
    Quiero ver el detalle de mis cuentas y sus movimientos recientes  
    Para poder controlar mis operaciones y saldos.

