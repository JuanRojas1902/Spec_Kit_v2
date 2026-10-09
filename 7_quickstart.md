# 7. Quickstart (Pruebas manuales v2)

Para comprobar que la v2 funciona:
1. Ejecutar el nuevo script SQL con los procedimientos almacenados y triggers en MariaDB.
2. Hacer `GET /api/universidad`. El campo `total_grupos` debe reflejar el conteo real.
3. Intentar crear un `grupo_investigacion` enviando el ID de una universidad que no existe. La API debe responder `409` (y no 500).
4. Intentar eliminar un `grupo_investigacion` que tenga semilleros activos. La API debe responder `409`.
5. En el frontend, abrir el formulario de "Nuevo Grupo" y verificar que el campo "Universidad" es un `<select>` con nombres y no un input numérico.
