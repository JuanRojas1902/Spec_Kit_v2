# Investigación y decisiones — v2

Aquí se anotan las decisiones que tomamos en la v2 y por qué. Lo que se
descubrió investigando está marcado como **Hallazgo**.

> **Estado:** borrador. Los `[NECESITA ACLARACIÓN]` se resuelven con el SQL
> oficial antes de programar.

## Resumen de decisiones

| # | Decisión | Por qué |
|---|---|---|
| 1 | Todo el CRUD por **procedimientos almacenados** (`CALL sp_...`) | Cero SQL escrito a mano en PHP |
| 2 | **3 triggers** separados que llaman a un solo procedimiento | MariaDB no permite varios eventos en un trigger |
| 3 | Distinguir errores con **`errorInfo[1]`** | PDO da el mismo `23000` para tres errores distintos |
| 4 | Reglas propias con **`SIGNAL SQLSTATE '45000'`** | Permite rechazar con un mensaje claro |
| 5 | **No se retira** un maestro con detalle activo | Evita hijos huérfanos con el borrado lógico |
| 6 | Errores de integridad responden **`409`**, nunca `500` | Son un conflicto, no una falla del servidor |
| 7 | `closeCursor()` después de cada `CALL` | Si no, falla con el error 2014 |
| 8 | Scripts con **`CREATE OR REPLACE`** | Se pueden correr varias veces |
| 9 | Un solo **`sp_actualizar`** para `PUT` y `PATCH` | Menos procedimientos; el servicio mezcla los datos |
| 10 | Cada FK es un **`<select>`** cargado desde la API | Cero llaves foráneas digitadas |
| 11 | FK opcional: opción **«(ninguna)»** que manda `null` | `(int) ''` daría `0` |
| 12 | Maestro-detalle en **un solo envío** con transacción | Todo o nada |
| 13 | Tablas puente: se **asignan y retiran**, no se editan | Una pareja existe o no existe |
| 14 | La v2 **incluye la v1** (regresión) | Lo que ya funcionaba debe seguir funcionando |

## Detalles de cada decisión

### Triggers en MariaDB (decisiones 2)

**Hallazgo:** MariaDB no soporta varios eventos en un solo trigger (como sí
hace PostgreSQL). Un trigger atiende un único evento.

**Decisión:** para contar los grupos por universidad se crean **3 triggers**
que llaman al mismo procedimiento:

| Trigger | Evento | Qué hace |
|---|---|---|
| `trg_grupo_ai` | `AFTER INSERT` | Recuenta la universidad del grupo nuevo |
| `trg_grupo_au` | `AFTER UPDATE` | Recuenta la universidad nueva y la anterior |
| `trg_grupo_ad` | `AFTER DELETE` | Recuenta la universidad del grupo borrado |

El procedimiento común es `sp_recontar_grupos`.

**Alternativas descartadas:**

- Un solo trigger para los tres eventos: MariaDB no lo permite.
- Calcular el total en cada consulta con `COUNT(*)`: no sería un trigger, y la
  rúbrica pide al menos uno.

**Puntos importantes:**

- El trigger de `UPDATE` es **imprescindible**: con borrado lógico, retirar un
  grupo es un `UPDATE`, no un `DELETE`.
- El trigger está en `grupo_investigacion` y modifica `universidad`. Si
  modificara su propia tabla fallaría (error 1442).
- Se comprueba **desde la pantalla**: se crea o retira un grupo y el contador
  cambia solo.

`[NECESITA ACLARACIÓN: nombre de la columna contador en universidad y de la FK universidad_id.]`

### Errores de integridad (decisión 3)

**Hallazgo:** PDO devuelve el mismo `SQLSTATE` (`23000`) para tres casos
distintos, así que `getCode()` no sirve para distinguirlos:

| Qué pasa | `errorInfo[0]` | `errorInfo[1]` |
|---|---|---|
| FK inexistente | `23000` | `1452` |
| Registro o pareja repetida | `23000` | `1062` |
| Borrado físico bloqueado | `23000` | `1451` |

**Decisión:** se distingue leyendo el código propio del driver en
`$e->errorInfo[1]`.

```php
catch (PDOException $e) {
    $codigo = $e->errorInfo[1];

    if ($codigo === 1452) { /* la FK no existe */ }
    if ($codigo === 1062) { /* repetido */ }
    if ($codigo === 1451) { /* hay dependientes */ }
}
```

### Reglas propias con `SIGNAL` (decisión 4)

Para rechazar una operación con un mensaje propio, el procedimiento lanza:

```sql
SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Primero retire los semilleros de este grupo.';
```

**Hallazgo:** este error se lee al revés que los anteriores. Aquí **no** se
mira `errorInfo[1]`, sino el índice `[0]`:

| Índice | Qué trae en un `SIGNAL` |
|---|---|
| `errorInfo[0]` | `'45000'`, que es la señal |
| `errorInfo[1]` | `1644` (código de MariaDB para una señal propia) |
| `errorInfo[2]` | El texto de `MESSAGE_TEXT` |

El spec dice «interceptar 1644», pero en PHP se detecta por
`errorInfo[0] === '45000'`, y el mensaje que ve el usuario sale de
`errorInfo[2]`.

```php
if ($e->errorInfo[0] === '45000') {
    throw new ConflictoExcepcion($e->errorInfo[2]);
}
```

### No retirar un maestro con detalle activo (decisión 5)

**Pregunta (Compuerta 1):** con borrado lógico, ¿se puede retirar un maestro
que todavía tiene detalle activo?

**Decisión: NO.**

- El procedimiento de retiro revisa si hay hijos activos.
- Si los hay, lanza `SIGNAL SQLSTATE '45000'`.
- La API responde `409`.

**Por qué:** con `DELETE FROM`, la base protegía los hijos con la llave
foránea (error 1451). Con borrado lógico solo se hace un `UPDATE`, y la FK no
se entera: quedarían semilleros activos de un grupo retirado. El
procedimiento hace ese trabajo.

**Alternativas descartadas:**

- Retirar también los hijos en cascada: la persona podría perder datos sin
  querer.
- Dejar los hijos huérfanos: los listados mostrarían datos incoherentes.

Una fila de tabla puente sí se puede retirar sola.

> [!NOTE]
> El error 1451 solo aparece con borrado físico. Con el borrado lógico de la
> v2 casi nunca se dispara, pero se sigue capturando por seguridad.

### `409 Conflict` y no `500` (decisión 6)

Un error de integridad no es una falla del servidor, es un conflicto con los
datos que ya existen. Por eso responde `409`.

| Situación | Código |
|---|---|
| FK inexistente (1452) | `409` |
| Repetido (1062) | `409` |
| Borrado con dependientes (1451) | `409` |
| Regla del procedimiento (`'45000'`) | `409` |
| Lo demás (validación, no existe, etc.) | Igual que en la v1: `400`, `404`, `405`, `422` |

Flujo: el repositorio captura `PDOException` → lanza `ConflictoExcepcion` → el
controlador responde `409`. El usuario nunca ve SQL ni nombres de tablas.

### Procedimientos almacenados (decisiones 1, 7, 8 y 9)

- Cinco por tabla: `sp_listar_<tabla>`, `sp_consultar_<tabla>`,
  `sp_crear_<tabla>`, `sp_actualizar_<tabla>` y `sp_eliminar_<tabla>`.
- Se escriben con `DELIMITER $$` y `CREATE OR REPLACE PROCEDURE`, para poder
  correr el script otra vez sin borrar nada a mano.
- **Hallazgo:** MariaDB no tiene `RETURNING`, así que `sp_crear_*` devuelve la
  fila con un `SELECT` final. Si la llave es `AUTO_INCREMENT`, se guarda
  `LAST_INSERT_ID()` en una variable antes de otra sentencia.
- **Hallazgo:** después de un `CALL` que devuelve filas hay que llamar
  `$stmt->closeCursor()`. Si no, la siguiente consulta falla con el error
  2014.
- `PUT` y `PATCH` usan el mismo `sp_actualizar`: el servicio lee el registro
  actual, mezcla los campos enviados y llama al procedimiento con todos.

### Desplegables y FK opcionales (decisiones 10 y 11)

- El `<select>` **muestra el nombre y manda la clave**.
- Solo ofrece registros con `activo = 1`.
- Si la FK es opcional, la primera opción es **«(ninguna)»** y llega como
  cadena vacía.
- **Hallazgo:** `(int) ''` vale `0`, y `0` no es «sin valor»: la base
  rechazaría esa FK. Hay que convertir `''` a `null` explícitamente.

```php
$grupoId = ($_POST['grupo_id'] ?? '') === '' ? null : (int) $_POST['grupo_id'];
```

- Todo texto que viene de la base se imprime con `htmlspecialchars`.

### Maestro-detalle (decisión 12)

- La pantalla del maestro muestra su detalle y permite agregar renglones sin
  salir de ella.
- Varios renglones viajan en **un solo envío**, con campos `detalle_xxx[]`
  que llegan a PHP como arreglos paralelos.
- El repositorio los guarda dentro de una transacción:

```php
$this->pdo->beginTransaction();
try {
    // CALL sp_crear_... por cada renglón
    $this->pdo->commit();
} catch (PDOException $e) {
    $this->pdo->rollBack();
    throw $e;
}
```

Si un renglón falla, **no se guarda ninguno**.

`[NECESITA ACLARACIÓN: confirmar los maestro-detalle (universidad → grupo, grupo → semillero, línea → docente) con la guía de la v2.]`

### Tablas puente (decisión 13)

- Su llave es compuesta (dos claves foráneas).
- Se **asignan** (`POST`) y se **retiran** (`DELETE` con las dos claves). No
  llevan botón de editar, porque cambiar una pareja es retirar una y asignar
  otra.
- Repetir una pareja → `409` (error 1062).
- Retirar una pareja que no existe → `404`.
- `PUT` o `PATCH` → `405`.

`[NECESITA ACLARACIÓN: ¿las tablas puente llevan activo o se borran físicamente?]`

### Regresión de la v1 (decisión 14)

La v2 **incluye** la v1. Los criterios de la v1 deben seguir en verde:
`PUT` exige todo, `PATCH` vacío da `400`, el segundo `DELETE` da `404`, y los
registros retirados no salen en los listados.

## Alternativas consideradas (resumen)

| Tema | Elegida | Descartadas |
|---|---|---|
| Acceso a datos | Procedimientos almacenados | SQL directo en PHP |
| Contador de grupos | 3 triggers | `COUNT(*)` en cada consulta |
| Detectar el error | `errorInfo[1]` y `errorInfo[0]` | `getCode()` |
| Retiro con hijos | Bloquear con `SIGNAL` | Cascada; dejar huérfanos |
| FK en pantalla | `<select>` | Campo de texto |
| Detalle | Un envío con transacción | Un envío por renglón |

## Pendientes por confirmar

- [ ] Numeración de versiones: el mapa del repositorio pone las tablas sin FK
      en la v2 y las que tienen FK en la v3. Preguntar al profesor cuál vale.
- [ ] SQL oficial de las 10 tablas (columnas, llaves y FK).
- [ ] ¿Las tablas puente llevan `activo`?
- [ ] Nombre de la columna contador en `universidad`.
- [ ] Maestro-detalle exactos según la guía de la v2.

## Qué viene después

- **v3:** login, JWT y roles.
- **v4:** consultas de varias tablas, dashboard y publicación.
