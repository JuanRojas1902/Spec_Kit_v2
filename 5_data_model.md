# 5. Cambios en el Modelo de Datos (v2)

## 1. Agregado de columna para Trigger (Propuesta A)
Para cumplir con el criterio de contar con al menos un disparador funcionando, se modificará la tabla `universidad` (Maestro) para que mantenga automáticamente el conteo de sus grupos de investigación activos (Detalle).

* **Tabla modificada:** `universidad`
* **Columna agregada:** `total_grupos INT NOT NULL DEFAULT 0`
* **Justificación:** Evita tener que usar un `COUNT()` en cada consulta, optimizando las lecturas. El valor será recalculado por un Procedimiento Almacenado invocado por 3 Triggers (`AFTER INSERT`, `AFTER UPDATE`, `AFTER DELETE` en la tabla `grupo_investigacion`).

## 2. Implementación de Procedimientos Almacenados
Toda operación (Listar, Consultar, Crear, Actualizar, Eliminar) para las 10 tablas de la V2 pasará por procedimientos almacenados nombrados bajo la convención `sp_<accion>_<tabla>`. El borrado lógico se mantendrá mediante `UPDATE ... SET activo = 0`.
