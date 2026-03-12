# Instrucciones del Agente — Proyecto

## Perfil del desarrollador
- Experto en **Java** y **APIs** (REST, diseño de contratos, integración).
- Preferencia por código limpio, convenciones estándar y documentación clara.

## Estilo de código (Java)
- Seguir convenciones de **Java** (nombres, paquetes, estructura).
- Usar **Java 17+** cuando el proyecto lo permita (records, sealed classes, pattern matching si aplica).
- Preferir inmutabilidad y APIs fluentes donde tenga sentido.
- Documentar APIs públicas con Javadoc (mínimo: propósito, parámetros, retorno, excepciones).
- Seguir principios de Clean Code

## APIs REST
- Diseño **RESTful**: recursos como sustantivos, verbos HTTP correctos, códigos de estado apropiados.
- Contratos claros: DTOs/POJOs bien definidos, sin exponer entidades internas.
- Validación en capa de entrada (Bean Validation, etc.) y mensajes de error consistentes.
- Documentación de API: OpenAPI/Swagger cuando el proyecto lo use.

## Arquitectura y capas
- Separar responsabilidades: controladores → servicios → repositorios/adapters.
- Mantener lógica de negocio en la capa de servicio.

## Especificaciones y documentación
- Las especificaciones de arquitectura expresas en `docs/spec/architecture.md`  son la fuente de verdad para arquitectura de la aplicación.
- Las especificaciones de API expresas en `docs/spec/api-contract.md`  son la fuente de verdad para el contrato de la API.
- Las especificaciones de requerimientos expresas en `docs/spec/requirements.md`  son la fuente de verdad para requisitos.
- Respetar decisiones de diseño y convenciones definidas en esos documentos al generar o modificar código.

## Referencias útiles
- Para reglas por tipo de archivo o flujos concretos, ver `.cursor/rules/`.
- Para crear o ajustar reglas del agente: usar `/create-rule` en el chat o editar archivos en `.cursor/rules/`.
