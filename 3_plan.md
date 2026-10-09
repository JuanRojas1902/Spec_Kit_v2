# Plan técnico — v2: Investigación con relaciones

> **Estado:** borrador. Los `[NECESITA ACLARACIÓN]` se resuelven con el SQL
> oficial antes de empezar a programar.

## 1. Secuencia de desarrollo

1. **Base de datos:** procedimientos almacenados (CRUD) de las 10 tablas,
   los 3 triggers (propuesta A en `universidad`) y las reglas de negocio
   (bloqueo de retiro con `SIGNAL SQLSTATE '45000'`).
2. **Repositorios:** reemplazar el SQL directo por `CALL sp_...` y capturar
   los errores de MariaDB (1452, 1062, 1451 y `'45000'`).
3. **Servicios y controladores:** devolver `409 Conflict` con mensajes en
   español.
4. **Frontend:** pantallas Maestro-Detalle y `<select>` dinámicos en lugar de
   escribir las llaves foráneas.

## 2. Qué cambia respecto a la v1

La v2 **incluye** la v1: lo que ya funciona no se toca, solo se agrega.

| Capa | Cambio |
|---|---|
| Base de datos | `ALTER` para `activo`, procedimientos y triggers |
| Repositorios | Ahora hacen `CALL` en vez de SQL escrito en PHP |
| Servicios | Nueva excepción `ConflictoExcepcion` y reglas de retiro |
| Controladores | Nuevo caso: responder `409` |
| Frontend | Desplegables, maestro-detalle y manejo de la tabla vacía (`204`) |

La arquitectura por capas es la misma:

```text
Frontend → API → Controlador → Servicio → Repositorio → CALL sp_... → MariaDB
```

## 3. Archivos nuevos

```text
db/
├── init.sql                 ← esquema y datos (ya existe)
└── procedimientos.sql       ← ALTER, procedimientos y triggers (nuevo)

api_investigacion/
├── controladores/           ← un controlador por cada tabla nueva
├── servicios/               ← un servicio (y su interfaz) por tabla
├── repositorios/            ← un repositorio (y su interfaz) por tabla
├── modelos/                 ← una clase por tabla
├── excepciones/
│   └── ConflictoExcepcion.php   ← nueva
└── pruebas/
    └── prueba_capas_v2.php

front_investigacion/
└── vistas/                  ← lista, formulario y detalle por tabla
```

`[NECESITA ACLARACIÓN: nombres exactos de las 10 tablas y sus columnas, tomados del SQL oficial.]`

## 4. Base de datos

### 4.1 Cambios de esquema

- Agregar `activo TINYINT(1) NOT NULL DEFAULT 1` a las tablas que no lo tengan.
- Agregar a `universidad` la columna del contador del trigger (ver §5).

`[NECESITA ACLARACIÓN: ¿las tablas puente llevan activo o se borran físicamente?]`

### 4.2 Procedimientos almacenados

Cinco por tabla, con el nombre que exige la guía:

| Operación | Procedimiento |
|---|---|
| Listar | `sp_listar_<tabla>` |
| Ver uno | `sp_consultar_<tabla>` |
| Crear | `sp_crear_<tabla>` |
| Actualizar | `sp_actualizar_<tabla>` |
| Retirar | `sp_eliminar_<tabla>` |

Reglas:

- Se escriben con `DELIMITER $$` y `CREATE OR REPLACE PROCEDURE`, para poder
  correr el script varias veces.
- `listar` y `consultar` filtran `activo = 1`.
- `crear` devuelve la fila que quedó guardada. Si la llave es
  `AUTO_INCREMENT`, guarda `LAST_INSERT_ID()` en una variable antes de otra
  sentencia.
- `eliminar` hace `UPDATE ... SET activo = 0`, nunca `DELETE`.
- `PUT` y `PATCH` usan el mismo `sp_actualizar`: el servicio lee el registro
  actual, mezcla los campos enviados y llama al procedimiento con todos.

Ejemplo del bloqueo de retiro (grupo con semilleros activos):

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE sp_eliminar_grupo_investigacion(IN p_id INT)
BEGIN
    IF EXISTS (SELECT 1 FROM semillero
               WHERE grupo_id = p_id AND activo = 1) THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Primero retire los semilleros de este grupo.';
    END IF;

    UPDATE grupo_investigacion SET activo = 0 WHERE id = p_id AND activo = 1;
END$$

DELIMITER ;
```

`[NECESITA ACLARACIÓN: nombres reales de las columnas (grupo_id, id...).]`

## 5. Los 3 triggers (propuesta A: contador en `universidad`)

`universidad` guarda cuántos grupos activos tiene, y **se mantiene sola**
sin que la API la envíe.

En MariaDB un trigger atiende **un solo evento**, así que son tres, y los
tres llaman a un procedimiento común:

| Trigger | Cuándo | Qué hace |
|---|---|---|
| `trg_grupo_ai` | Después de `INSERT` en `grupo_investigacion` | Recalcula el total de esa universidad |
| `trg_grupo_au` | Después de `UPDATE` | Recalcula la universidad nueva y la anterior |
| `trg_grupo_ad` | Después de `DELETE` | Recalcula el total |

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE sp_recalcular_total_grupos(IN p_universidad INT)
BEGIN
    UPDATE universidad
    SET total_grupos = (SELECT COUNT(*) FROM grupo_investigacion
                        WHERE universidad_id = p_universidad AND activo = 1)
    WHERE id = p_universidad;
END$$

CREATE OR REPLACE TRIGGER trg_grupo_ai AFTER INSERT ON grupo_investigacion
FOR EACH ROW CALL sp_recalcular_total_grupos(NEW.universidad_id)$$

DELIMITER ;
```

Los otros dos siguen el mismo patrón.

Puntos importantes:

- El trigger de `UPDATE` es **imprescindible**: con borrado lógico, retirar un
  grupo es un `UPDATE`, no un `DELETE`.
- El trigger está sobre `grupo_investigacion` y modifica `universidad`. Si
  modificara su propia tabla fallaría (error 1442).
- Se comprueba **desde la pantalla**: se crea o retira un grupo y el contador
  de la universidad cambia solo.

`[NECESITA ACLARACIÓN: nombre de la columna contador y de la FK universidad_id.]`

## 6. Repositorios

El repositorio es el único que hace `CALL`, siempre con parámetros `?`:

```php
$stmt = $this->pdo->prepare('CALL sp_consultar_docente(?)');
$stmt->execute([$id]);
$fila = $stmt->fetch(PDO::FETCH_ASSOC);
$stmt->closeCursor();   // obligatorio, si no falla con el error 2014
```

Reglas:

- Nada de SQL armado concatenando texto.
- `closeCursor()` después de cada `CALL` que devuelva filas.
- Si la FK es opcional y llega como cadena vacía, se convierte a `null`
  (con `(int) ''` daría `0`).
- El repositorio captura el `PDOException` y lanza `ConflictoExcepcion`:

```php
catch (PDOException $e) {
    $sqlstate = $e->errorInfo[0];   // '23000', '45000'...
    $codigo   = $e->errorInfo[1];   // 1452, 1062, 1451...

    if ($sqlstate === '45000') {
        throw new ConflictoExcepcion($e->errorInfo[2]);
    }
    if ($codigo === 1452) {
        throw new ConflictoExcepcion('El registro relacionado no existe.');
    }
    if ($codigo === 1062) {
        throw new ConflictoExcepcion('Ese registro ya existe.');
    }
    if ($codigo === 1451) {
        throw new ConflictoExcepcion('No se puede borrar: otros registros dependen de este.');
    }
    throw $e;
}
```

> [!NOTE]
> `getCode()` de PDO no sirve para distinguir 1452, 1062 y 1451: los tres
> llegan como `'23000'`. Por eso se usa `errorInfo[1]`. El `SIGNAL` se
> identifica por `errorInfo[0] = '45000'`, no por el 1644.

## 7. Servicios y controladores

| Capa | Qué hace en la v2 |
|---|---|
| Servicio | Valida, aplica la regla de retiro y deja pasar `ConflictoExcepcion`. No conoce HTTP |
| Controlador | Convierte `ConflictoExcepcion` en `409` |

Respuesta de ejemplo:

```json
{
    "estado": 409,
    "mensaje": "Conflicto de integridad.",
    "detalle": "Primero retire los semilleros de este grupo."
}
```

Tabla de errores de la v2:

| Situación | Código |
|---|---|
| FK inexistente (1452) | `409` |
| Pareja o registro repetido (1062) | `409` |
| Borrar con dependientes (1451) | `409` |
| Regla del procedimiento (`'45000'`) | `409` |
| El resto (validación, no existe, etc.) | Igual que en la v1: `400`, `404`, `405`, `422` |

El usuario **nunca** ve el SQL ni el nombre de la tabla en un error.

## 8. Rutas

**Tablas maestras** (mismo patrón de la v1):

```http
GET    /api/docente
GET    /api/docente/{id}
POST   /api/docente
PUT    /api/docente/{id}
PATCH  /api/docente/{id}
DELETE /api/docente/{id}
```

**Tablas puente:** se asignan y se retiran, no se editan.

```http
GET    /api/grupo_linea
POST   /api/grupo_linea
DELETE /api/grupo_linea/{clave1}/{clave2}
```

- Repetir una pareja → `409`.
- Retirar una pareja que no existe → `404`.
- `405` si se intenta `PUT` o `PATCH`.

`[NECESITA ACLARACIÓN: columnas reales de cada tabla puente.]`

## 9. Frontend

### 9.1 Desplegables

- Cada FK es un `<select>` que **muestra el nombre y manda la clave**.
- Se llena con una llamada a la API (`cliente_api.php`), solo con `activo = 1`.
- Si la FK es opcional, la primera opción es **«(ninguna)»** y manda `null`.
- Todo texto que viene de la base se imprime con `htmlspecialchars`.

### 9.2 Maestro-Detalle

- La pantalla del maestro muestra su detalle y permite agregar renglones sin
  salir de ella.
- Varios renglones viajan en **un solo envío**, con campos `detalle_xxx[]` que
  llegan como arreglos paralelos.
- El repositorio los guarda dentro de una **transacción** (`beginTransaction`,
  `commit` y `rollback`): si una parte falla, no se guarda nada.

`[NECESITA ACLARACIÓN: confirmar los maestro-detalle (universidad → grupo, grupo → semillero, línea → docente) con la guía de la v2.]`

### 9.3 Mensajes

- Un `409` se muestra como mensaje claro en castellano.
- La tabla vacía (`204`) se muestra como «todavía no hay datos».
- La interfaz no usa las palabras «PUT», «PATCH», «422» ni «FK».
- Si hay un error, **no se pierde lo que la persona escribió**.

## 10. Pruebas

**Prueba de capas** (`pruebas/prueba_capas_v2.php`), sin base de datos:

- Un repositorio falso en memoria que lanza `ConflictoExcepcion`.
- El servicio deja pasar el conflicto.
- Retirar un maestro con hijos activos es rechazado.

**Pruebas por HTTP**, incluidas las que **deben fallar**:

- [ ] CRUD de cada tabla maestra.
- [ ] Asignar, repetir (`409`) y retirar en cada tabla puente.
- [ ] FK inexistente → `409`.
- [ ] Retirar un grupo con semilleros activos → `409`.
- [ ] Maestro-detalle con un renglón inválido → no se guarda nada.
- [ ] Tabla vacía → `204`.
- [ ] Trigger: crear y retirar un grupo cambia el contador de la universidad.

**Regresión:** todos los criterios de la v1 siguen en verde.

## 11. Orden de trabajo

1. Confirmar tablas y columnas con el SQL oficial.
2. Escribir
