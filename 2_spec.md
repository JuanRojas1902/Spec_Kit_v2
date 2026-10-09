# 2. Especificación (Versión 2 - Investigación)

## 1. Alcance
Implementar el CRUD completo, usando Procedimientos Almacenados, para las 10 tablas con llaves foráneas del módulo de Investigación: `docente`, `grupo_investigacion`, `semillero`, y las tablas puente (`participa_grupo`, `participa_semillero`, `grupo_linea`, `semillero_linea`, `ac_linea`, `ods_linea`, `aa_linea`).

## 2. Requerimientos Técnicos
* **Cero SQL crudo en PHP**: Todas las peticiones a la base de datos se harán mediante `CALL sp_...`.
* **Cero FK digitadas a mano**: En el frontend, todas las llaves foráneas serán listas desplegables (`<select>`) cargadas mediante la API.
* **Maestro-Detalle**: Interfaz donde al abrir un maestro, se pueda agregar su detalle en un solo envío.
* **Integridad**: Interceptar los errores SQL (1452, 1062, 1451, 1644) y devolver `HTTP 409 Conflict`.

## 3. Clarificaciones (Compuerta 1)
**Pregunta:** Con borrado lógico (`activo = 0`), ¿se puede retirar un maestro que todavía tiene detalle activo? (Ej. borrar un grupo de investigación que tiene semilleros).
**Decisión:** **NO**. Se bloqueará la operación. Si se intenta hacer un borrado lógico de un maestro con hijos activos, la base de datos disparará una señal (`SIGNAL SQLSTATE '45000'`) y la API devolverá un error `HTTP 409 Conflict` indicando que primero deben retirarse los detalles.
