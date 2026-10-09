# 8. Tareas (v2)

### 1. Capa de Base de Datos (MariaDB)
- [ ] Ejecutar `ALTER TABLE universidad` para agregar la columna `total_grupos`.
- [ ] Escribir el procedimiento `sp_recontar_grupos` y los 3 Triggers (`AFTER INSERT`, `AFTER UPDATE`, `AFTER DELETE`) en la tabla `grupo_investigacion`.
- [ ] Crear Procedimientos Almacenados (CRUD completo: listar, consultar, crear, modificar, eliminar) para las tablas principales: `docente`, `grupo_investigacion` y `semillero`.
- [ ] Incluir la regla `SIGNAL SQLSTATE '45000'` dentro de los procedimientos `sp_eliminar_universidad` y `sp_eliminar_grupo_investigacion` para bloquear el borrado si tienen hijos activos.
- [ ] Crear Procedimientos Almacenados (Asignar y Retirar) para las 7 tablas puente (ej. `participa_grupo`, `grupo_linea`, etc.).

### 2. Capa Backend (API en PHP)
- [ ] Refactorizar los Repositorios de la v2 para eliminar el SQL crudo y utilizar exclusivamente sentencias preparadas con `CALL sp_...`.
- [ ] Implementar la captura de excepciones PDO en los Repositorios, leyendo `$e->errorInfo[1]` (para errores 1452, 1062, 1451) y `$e->errorInfo[0]` (para el 1644/45000 propio).
- [ ] Ajustar los Servicios y Controladores para asegurar que ante una violación de integridad devuelvan el código `HTTP 409 Conflict` con un mensaje descriptivo en español (nunca 500).
- [ ] Configurar el backend para recibir y mapear correctamente valores `null` en llaves foráneas opcionales.

### 3. Capa Frontend (Interfaz y JavaScript)
- [ ] Reemplazar todos los *inputs* numéricos de Llaves Foráneas por listas desplegables (`<select>`).
- [ ] Programar peticiones `GET` mediante Fetch en el frontend para poblar dichos `<select>` mostrando los nombres (ej. nombre de la universidad) y enviando el ID en el `value`.
- [ ] Controlar que si un `<select>` opcional se deja vacío, el JavaScript envíe explícitamente `null` en el JSON y no una cadena vacía `""`.
- [ ] Construir la interfaz Maestro-Detalle (ej. Grupo y Semillero), garantizando que los datos de ambos se envíen al backend en una sola petición (un solo JSON).
- [ ] Implementar el manejo de respuestas 409 en el frontend, mostrando alertas (ej. `alert()` o modales de Bootstrap) con el mensaje exacto que devuelve la API.

### 4. Verificación y Cierre
- [ ] Ejecutar paso a paso el `7_quickstart.md` para probar triggers, errores 409 y maestro-detalle.
- [ ] Ejecutar la **regresión completa de la v1** para comprobar que el código nuevo no rompió lo anterior.
- [ ] Etiquetar el commit final con el tag `v2` en la rama `main`.
