# 6. Contratos de la API (Delta v2)

* Los endpoints mantienen el patrón RESTful establecido en la v1 (`GET`, `POST`, `PUT`, `DELETE` en `/api/{tabla}`).
* **Manejo de Claves Foráneas (FK):** En las peticiones `POST` y `PUT`, las claves foráneas viajan obligatoriamente como valores numéricos (ID). Si una llave foránea es opcional y el usuario no selecciona ninguna, el cliente debe enviar el valor `null` (nunca `0` ni cadena vacía `""`).
* **Envío Maestro-Detalle:** La creación de un maestro junto con múltiples registros de detalle se procesará en una **única petición** HTTP para garantizar la atomicidad (todo se guarda o nada se guarda).
* **Nuevo código HTTP esperado:** La API ahora interceptará las violaciones de integridad del motor de base de datos (MariaDB) y devolverá `409 Conflict` estructurado en lugar de un genérico `500 Internal Server Error`.

### Catálogo de Respuestas 409 (Conflict)

**1. Borrado bloqueado por hijos activos (Regla de negocio propia - SQLSTATE '45000'):**
Ocurre cuando se intenta hacer un borrado lógico de un maestro que aún conserva detalles activos (Ej. un grupo de investigación con semilleros).
```json
{
  "estado": 409,
  "mensaje": "No se puede eliminar el grupo porque tiene semilleros activos. Retire primero el detalle."
}

{
  "estado": 409,
  "mensaje": "El registro maestro al que intenta vincularse no existe en la base de datos."
}

{
  "estado": 409,
  "mensaje": "Esta asignación ya existe. No se pueden duplicar vínculos."
}
