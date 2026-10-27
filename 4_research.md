# Decisiones — Versión 2

> ¿Por qué? del plan, con alternativas descartadas. Lectura opcional.

| # | Decisión | Alternativa descartada | Por qué |
|---|---|---|---|
| D1 | Rechazo de integridad → **409** | 500 (v1) o 400 | En la v2 la integridad es el tema. 500 dice «se dañó algo» y 400 dice «mal escrito»; ambos son mentira. 409 dice «no cabe en los datos que hay» |
| D2 | **Se intenta y se traduce** | Comprobar antes con un `SELECT` | Entre comprobar y escribir otro puede borrar el padre; y duplica una regla que ya está en la base |
| D3 | **Un maestro-detalle: grupo → semillero** | Los tres, o `linea → docente` | Un detalle no existe sin su maestro; el docente existe solo. Uno bien hecho enseña más que tres a medias |
| D4 | **Procedimientos solo en el maestro-detalle** | En las 10 tablas | Ganan donde hay varias escrituras en una transacción; un CRUD de una tabla no gana nada |
| D5 | **Detalle como un parámetro JSON** (`JSON_TABLE`) | Varios `CALL`, o un parámetro por semillero | Un procedimiento no recibe listas; varios `CALL` rompen el «todo o nada» |
| D6 | **`45000` → 409, `45001` → 404** | Leer el texto del mensaje | El texto es frágil. `45001` es decisión propia (la guía solo nombra `45000`) |
| D7 | **No se retira un grupo con semilleros activos** | Cascada, o dejar huérfanos | La cascada hace perder datos; los huérfanos muestran datos incoherentes |
| D8 | **Grupo sin `PATCH`** | Interfaz igual a las demás fichas | Cambiar un renglón del detalle deja dudas sobre los que no se mencionan; `PUT` las resuelve |
| D9 | **Puente: asignar y retirar, sin editar ni `activo`** | `PUT`/`PATCH`, o borrado lógico | Su llave es la pareja; con `activo`, volver a asignar chocaría con la llave |
| D10 | **Desplegables** que muestran nombre y mandan clave | Casilla de texto | Nadie adivina códigos. Pero **no validan**: la lista pudo cambiar y cualquiera puede mandar un `POST` sin la pantalla |
| D11 | **Cinco renglones fijos** | Botón «agregar» con JS, o dos pasos | Sin JS; y los dos pasos permitirían un grupo a medias |
| D12 | **Sin disparadores** | `total_grupos` con tres triggers | La v2 no los pide ni suman nota; no se copian «para cumplir» |
| D13 | **Contraseña en el compose** | `.env` | Art. 9: excepción declarada; la v1 decía `.env` y se corrige |
| D14 | **Estilos con `marca.css`** | Colores propios | Art. 12: el manual se cumple, no se discute |

## Detalles que hay que saber explicar

**Cómo se lee un error del motor (D6):**

| Qué pasa | Dónde se lee | Respuesta |
|---|---|---|
| Repetido | `errorInfo[1] = 1062` | `409` |
| Padre no existe | `errorInfo[1] = 1452` | `409` |
| Hay dependientes (solo borrado físico) | `errorInfo[1] = 1451` | `409` |
| Regla propia | `errorInfo[0] = '45000'` | `409` con su mensaje |
| Grupo inexistente | `errorInfo[0] = '45001'` | `404` |

**El desplegable no valida (D10).** Confundir «la pantalla no deja escoger eso» con «eso no puede pasar» es uno de los errores más caros del software. Quien defiende la integridad es la base.

**Todo o nada (D5).** El procedimiento abre la transacción y tiene un `EXIT HANDLER FOR SQLEXCEPTION` que hace `ROLLBACK` y `RESIGNAL`. `JSON_TABLE` pide MariaDB 10.6 o superior (la imagen `mariadb:11` lo cumple).

**Precio asumido en el `PUT` (D8):** retirar y reinsertar los semilleros les cambia el `id`. Si algún día `participa_semillero` apunta a un semillero que se reemplaza, habrá que hacerlo con más cuidado.

## Pendientes

- [ ] Contrastar `docente`, `semillero` y las puente con `db_scripts/mysql/investigacion.sql` (Art. 5).
- [ ] Preguntar al profesor si acepta `'45001'` y `JSON_TABLE`.
- [ ] Si el script oficial tiene errores que corregir, cada corrección se documenta aquí con su motivo y cómo se encontró.
