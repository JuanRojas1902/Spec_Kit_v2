# 4. Investigación y Decisiones (v2)

* **Triggers en MariaDB:** Se investigó que MariaDB no soporta múltiples eventos en un solo trigger (como PostgreSQL). Por lo tanto, para el recuento de grupos por universidad, se crearán 3 triggers separados (`AFTER INSERT`, `AFTER UPDATE`, `AFTER DELETE`) que llamarán a un único procedimiento de recálculo `sp_recontar_grupos`.
* **Manejo de Errores de Integridad:** Se descubrió que PDO devuelve el mismo SQLSTATE (`23000`) para llaves foráneas inexistentes, duplicados y borrado físico bloqueado. La distinción se hará leyendo el driver-specific error code en `$e->errorInfo[1]`.
* **Reglas propias (1644):** Para el bloqueo del borrado lógico si hay hijos activos, usaremos `SIGNAL SQLSTATE '45000'`. Este se lee al revés en PDO: se verifica el índice `[0]` del arreglo de errores.
