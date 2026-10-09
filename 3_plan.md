# 3. Plan de Acción (v2)

## Secuencia de Desarrollo
1. **Base de Datos:** Escribir los Procedimientos Almacenados (CRUD) para las 10 tablas, los 3 Triggers (Propuesta A en `universidad`) y las reglas de negocio (bloqueo de borrado con `SIGNAL SQLSTATE '45000'`).
2. **Repositorios:** Modificar los métodos en PHP para que no usen SQL directo, sino `CALL sp_...`. Capturar los errores de MariaDB (1452, 1062, 1451, 1644) analizando `$e->errorInfo[1]` y `$e->errorInfo[0]`.
3. **Servicios y Controladores:** Ajustar para que devuelvan el código `409 Conflict` con los mensajes en español.
4. **Frontend:** Construir las interfaces Maestro-Detalle y reemplazar los inputs de texto de las Llaves Foráneas por `<select>` dinámicos alimentados por la API.
