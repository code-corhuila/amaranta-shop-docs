# 06 - Data

Esta seccion define como Amaranta Shop almacena, protege, migra y relaciona sus datos.
El foco del MVP es **SQL relacional por microservicio**, usando PostgreSQL como motor
base segun `05-architecture/decisions/records/ADR-002-database-engine-and-orm.md`.

La regla principal es simple: **cada microservicio es propietario de su propia base de
datos**. Ningun servicio hace `JOIN` ni consulta directa contra tablas de otro servicio.
Cuando necesita informacion externa, la obtiene por API o por eventos.

---

## Archivos de esta seccion

| Archivo | Proposito | Estado |
|---------|-----------|--------|
| `models.md` | Esquema SQL por microservicio, tablas, relaciones, indices, seguridad basica y ownership | Completo para el MVP |
| `data-dictionary.md` | Diccionario detallado de campos criticos | Pendiente |
| `modeling-conventions.md` | Convenciones de nombres, tipos, auditoria y borrado logico | Pendiente |
| `normalization-assessment.md` | Justificacion de normalizacion y denormalizaciones | Pendiente |
| `migration-strategy.md` | Politica detallada de migraciones | Pendiente |

Mientras los archivos pendientes no existan, `models.md` es la fuente principal para el
modelo de datos.

---

## Respuestas que debe cubrir esta seccion

| Pregunta | Respuesta para Amaranta |
|----------|-------------------------|
| Que datos maneja cada microservicio? | Identity & Access (`identity-service` en arquitectura, `auth-service` en la carpeta actual) maneja usuarios, credenciales, roles y tokens; `catalog-service` productos y stock; `sales-service` ordenes e items; `scheduling-service` citas; `clinical-service` mascotas e historias clinicas; `notification-service` bitacora de envios. |
| Por que se eligio ese motor de base de datos? | PostgreSQL porque el dominio es relacional, requiere transacciones ACID, restricciones, indices, migraciones versionadas y buen soporte para Prisma. |
| Como se actualiza el esquema sin que el sistema falle? | Con migraciones SQL versionadas, forward-only, cambios compatibles primero, y cambios destructivos en dos fases. |
| Quien es el propietario de cada dato? | Cada dato tiene un unico owner. Los demas servicios guardan solo referencias UUID o snapshots necesarios para auditoria. |

---

## Mapa de datos por microservicio

| Microservicio | Base SQL | Datos propios | Referencias externas permitidas |
|---------------|----------|---------------|---------------------------------|
| `api-gateway` | Sin BD persistente | No posee datos de negocio | Consulta JWT y aplica politicas de entrada |
| `identity-service` / `auth-service` | PostgreSQL | Usuarios, credenciales, roles, tokens, auditoria de seguridad | Ninguna |
| `catalog-service` | PostgreSQL | Productos, precios, stock, movimientos de inventario | `created_by`, `source_order_id` como UUID externos |
| `sales-service` | PostgreSQL | Ordenes, items, estados de orden | `customer_id`, `created_by`, `product_id` como UUID externos |
| `scheduling-service` | PostgreSQL | Citas, agenda, historial de estado | `pet_id`, `owner_id`, `professional_id` como UUID externos |
| `clinical-service` | PostgreSQL | Mascotas, historias clinicas | `owner_id`, `appointment_id`, `veterinarian_id` como UUID externos |
| `notification-service` | PostgreSQL | Plantillas y log de entregas | `source_event_id`, `aggregate_id`, destinatarios |

---

## Correlaciones con otras secciones

| Esta seccion se alimenta de... | Y alimenta... |
|--------------------------------|---------------|
| `02-domain/entities-and-rules.md` - entidades, invariantes y value objects | Tablas SQL, columnas obligatorias, restricciones `CHECK` y relaciones internas |
| `02-domain/domain-events.md` - eventos y consumidores | Tablas `outbox_messages` y `processed_events` |
| `05-architecture/` - decisiones de motor, database per service, outbox y saga | Eleccion del motor y reglas de consistencia en `models.md` |
| `models.md` | `07-api/contracts/` - datos que expone cada servicio por API |
| `models.md` | `08-uml/` - diagramas ER por servicio |
| `models.md` | `09-microservices/[service]/data-model.md` - detalle operacional por servicio |

---

## Seguridad basica desde SQL

El modelo usa seguridad defensiva desde la base de datos:

- roles separados para migraciones y aplicacion: `*_migrator` y `*_app`;
- contrasenas y refresh tokens guardados solo como hash;
- restricciones `NOT NULL`, `UNIQUE`, `CHECK` y FK internas;
- borrado logico con `deleted_at`;
- tablas de auditoria e idempotencia;
- ausencia de permisos entre bases de datos de servicios distintos;
- consultas publicas por vistas o endpoints, no por acceso directo a tablas sensibles.

La autorizacion de negocio se valida en cada servicio con RBAC, segun
`00-governance/security-policy.md`.

---

## Como anexar informacion nueva

1. Identificar el owner del dato en la matriz anterior.
2. Agregar la tabla o columna en `models.md` dentro del microservicio propietario.
3. Si otro servicio necesita leer ese dato, actualizar el contrato en `07-api/contracts/`
   o publicar/consumir un evento en `02-domain/domain-events.md`.
4. Crear una migracion SQL nueva con nombre `V###__descripcion.sql`.
5. No modificar migraciones ya ejecutadas.
6. Actualizar el diagrama ER en `08-uml/` y el `data-model.md` del servicio en
   `09-microservices/services/`.
