# 7. Quickstart (Pruebas manuales v2)

Para comprobar que los criterios de aceptación de la v2 funcionan correctamente, ejecute la siguiente secuencia de pruebas:

### 1. Preparación de la Base de Datos
1. Ejecutar el script SQL actualizado que contiene el `ALTER TABLE` de la tabla `universidad`, los procedimientos almacenados (`sp_...`) y los triggers en MariaDB.

### 2. Pruebas de Lógica de Base de Datos y API
2. **Validación del Disparador (Trigger):** Hacer `GET /api/universidad` y anotar el valor de `total_grupos` de una universidad. Crear un `grupo_investigacion` nuevo asociado a ella y repetir el `GET`; el contador debe haber subido exactamente en 1. Luego, eliminar lógicamente ese grupo y confirmar que el contador baja.
3. **Validación Padre Inexistente (Error 1452):** Intentar crear un `grupo_investigacion` enviando por POST el ID de una universidad que no existe (ej. `99999`). La API debe atrapar el error y responder `409 Conflict` (y no 500).
4. **Validación de Duplicidad en Puente (Error 1062):** Asignar un docente a un grupo de investigación (`POST` a `participa_grupo`). Intentar enviar la misma petición exacta por segunda vez. La API debe bloquearlo y responder `409 Conflict`.
5. **Validación de Regla de Negocio (Señal 1644):** Intentar eliminar (`DELETE`) un `grupo_investigacion` que tenga registros de `semillero` activos. La API debe denegar la operación devolviendo `409 Conflict` con el mensaje en castellano de que deben retirarse los detalles primero.

### 3. Pruebas de Interfaz (Frontend)
6. **Desplegables (Llaves Foráneas):** En el frontend, abrir el formulario de "Nuevo Grupo" y verificar que el campo "Universidad" es un `<select>` que muestra explícitamente los **nombres** de las universidades y no campos de texto para digitar el ID numérico.
7. **Filtrado Lógico en Desplegables:** Eliminar lógicamente una universidad (v1). Al volver al formulario de "Nuevo Grupo", esa universidad ya no debe aparecer como opción seleccionable en el `<select>`.
8. **Interfaz Maestro-Detalle:** Abrir la vista de un maestro (ej. `grupo_investigacion`) y comprobar que su detalle (`semillero`) se visualiza y se puede agregar en la misma pantalla en un único formulario, sin tener que navegar a menús separados.

### 4. Regresión
9. **Regresión obligatoria v1:** Validar que el CRUD de las tablas iniciales (`area_conocimiento`, `termino_clave`, etc.) siga funcionando sin errores.
