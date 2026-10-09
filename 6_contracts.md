# 6. Contratos de la API (Delta v2)

* Los endpoints siguen el mismo patrón RESTful de la v1 (`GET`, `POST`, `PUT`, `DELETE` en `/api/{tabla}`).
* **Nuevo código HTTP esperado:** La API ahora devolverá `409 Conflict` en lugar de `500 Internal Server Error` ante violaciones de integridad.

**Ejemplo de respuesta 409 (Borrado bloqueado por hijos activos):**
```json
{
  "estado": 409,
  "mensaje": "No se puede eliminar el grupo porque tiene semilleros activos. Retire primero el detalle."
}
