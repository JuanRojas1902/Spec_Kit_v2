# Especificación — v2: Investigación con relaciones

> **Estado:** borrador para la Compuerta 1. Los `[NECESITA ACLARACIÓN]` se
> resuelven antes de planear (ver §11).

## 1. Alcance

La v2 construye el CRUD de las **tablas con llave foránea** del módulo de
Investigación y **todo el acceso a datos pasa por procedimientos
almacenados**. La v2 **incluye** la v1: las seis tablas de catálogo siguen
en pie y ahora alimentan los desplegables.

| Tipo | Tablas |
|---|---|
| Maestras | `docente`, `grupo_investigacion`, `semillero` |
| Tablas puente | `participa_grupo`, `participa_semillero`, `grupo_linea`, `semillero_linea`, `ac_linea`, `ods_linea`, `aa_linea` |

`[NECESITA ACLARACIÓN: contrastar esta lista con la fila v2 de docs/spec_kit/versiones/0_mapa_versiones.md. El mapa de la metodología menciona también universidad → grupo_investigacion y linea_investigacion → docente.]`

## 2. Fuera de alcance

- Login, JWT, roles, CRUD de `usuario`, `rol` y `rol_usuario` (v3).
- Consultas multitabla, dashboard, imagen corporativa y publicación (v4).
- Cambiar las seis tablas de la v1 (solo se usan).

## 3. Requerimientos técnicos

- **Todo el CRUD por procedimientos almacenados:** los cinco verbos (listar,
  consultar, crear, actualizar, eliminar) de **todas** las tablas.
- **Cero SQL escrito a mano en PHP:** el repositorio solo hace `CALL` con
  parámetros `?`. Nunca SQL armado concatenando texto.
- **Cero FK digitadas:** cada llave foránea es un `<select>` cargado desde
  la API, que muestra el nombre y manda la clave.
- **Maestro-Detalle:** el detalle se ve y se agrega desde el maestro, y varios
  renglones viajan en **un solo envío** (todo o nada).
- **Integridad:** los errores de la base responden `409`, nunca `500`.
- **Borrado lógico:** `DELETE` es `activo = 0` y los listados y desplegables
  filtran `activo = 1`.
- **Al menos un disparador** funcionando (ver §7).
- **PHP 8.3 puro con PDO**, sin framework, front también en PHP.

## 4. Procedimientos almacenados

Un nombre por operación y por tabla:

| Operación | Procedimiento |
|---|---|
| Listar | `sp_listar_<tabla>` |
| Ver uno | `sp_consultar_<tabla>` |
| Crear | `sp_crear_<tabla>` |
| Actualizar | `sp_actualizar_<tabla>` |
| Retirar | `sp_eliminar_<tabla>` |

Reglas:

- Se escriben con `DELIMITER $$` y `CREATE OR REPLACE PROCEDURE`, para poder
  volver a correr el script sin borrar nada a mano.
- `sp_listar_*` y `sp_consultar_*` filtran `activo = 1`.
- `sp_crear_*` **devuelve la fila como quedó guardada** (MariaDB no tiene
  `RETURNING`). Si la llave es `AUTO_INCREMENT`, se guarda `LAST_INSERT_ID()`
  en una variable antes de otra sentencia.
- `sp_eliminar_*` hace `UPDATE ... SET activo = 0`, nunca `DELETE`.
- Después de cada `CALL` que devuelve filas, el repositorio llama
  `closeCursor()` (si no, falla con el error 2014).
- Los procedimientos viven en el script de la base, versionados.

> [!NOTE]
> Las tablas del módulo no traen la columna `activo`. Se agrega con `ALTER`
> en el script y se documenta en `5_data_model.md`. `[NECESITA ACLARACIÓN: ¿las tablas puente también llevan activo, o se borran físicamente? Decidirlo y escribirlo.]`

## 5. Rutas de la API

Un juego de endpoints **por tabla** (no una ruta genérica), igual que en la v1:

```http
GET    /api/docente
GET    /api/docente/{id}
POST   /api/docente
PUT    /api/docente/{id}
PATCH  /api/docente/{id}
DELETE /api/docente/{id}
```

Lo mismo para `grupo_investigacion` y `semillero`.

Las **tablas puente** tienen llave compuesta (dos claves foráneas). Una pareja
existe o no existe: se **asigna** y se **retira**, no se edita.

```http
GET    /api/grupo_linea
POST   /api/grupo_linea
DELETE /api/grupo_linea/{clave1}/{clave2}
```

`[NECESITA ACLARACIÓN: nombres reales de las columnas de cada tabla puente, tomados del SQL oficial (db_scripts/mysql/).]`

## 6. Desplegables (sin FK a mano)

El desplegable **muestra el nombre** y **manda la clave**. Solo ofrece
registros activos.

| Formulario | Desplegable de |
|---|---|
| Semillero | Grupos de investigación |
| Participa en grupo | Docentes y grupos |
| Participa en semillero | Docentes y semilleros |
| Grupo ↔ línea, semillero ↔ línea | Líneas de investigación |
| Área de conocimiento ↔ línea | `area_conocimiento` |
| ODS ↔ línea | `objetivo_desarrollo_sostenible` |
| Área de aplicación ↔ línea | `area_aplicacion` |

Reglas de PHP:

- Todo lo que viene de la base y se imprime pasa por `htmlspecialchars`.
- Si la FK es **opcional**, el desplegable tiene «(ninguna)» y llega como
  cadena vacía; se convierte a `null` (no con `(int) ''`, que da `0`).
- La API responde el sobre `{tabla, limite, total, datos}`; la tabla vacía
  responde `204` sin cuerpo y la pantalla debe manejarlo sin reventar.

## 7. Disparador (mínimo uno)

`[NECESITA ACLARACIÓN: elegir al menos uno de A–E de la guía de la v2 o proponer otro.]`

| | Propuesta | Para Investigación |
|---|---|---|
| A | Contador en el maestro | `grupo_investigacion.total_semilleros` se mantiene solo |
| B | Sello de modificación | `fecha_modificacion` se pone sola en cada `UPDATE` |
| C | Bitácora de lo retirado | Tabla `bitacora` |
| D | Validación entre tablas | Una regla que un `CHECK` no puede hacer |
| E | Estado derivado | Marcar el grupo «con semilleros» al recibir el primero |

Reglas de MariaDB:

- **Un disparador atiende un solo evento:** si se hace la opción A, son tres
  (`INSERT`, `UPDATE`, `DELETE`) que llaman a un procedimiento común.
- El `UPDATE` es imprescindible: con borrado lógico, retirar un hijo es un
  `UPDATE`.
- Un disparador no puede modificar su propia tabla (error 1442).
- **Se comprueba por la interfaz:** el dato cambia sin que la API lo envíe.

## 8. Maestro-Detalle

La pantalla del maestro muestra su detalle y permite agregarle renglones sin
salir de ella. Varios renglones van en **un solo envío**, con campos de nombre
`detalle_xxx[]` que llegan como arreglos paralelos. Si una parte falla, no se
guarda nada.

| Maestro | Detalle |
|---|---|
| `grupo_investigacion` | `semillero` (y sus líneas y docentes) |
| `semillero` | sus líneas y sus docentes participantes |
| `docente` | sus participaciones |

`[NECESITA ACLARACIÓN: confirmar los maestro-detalle contra el mapa de versiones del repositorio.]`

## 9. Integridad: 409, nunca 500

| Qué pasa | Cómo se detecta | Respuesta |
|---|---|---|
| Hijo con padre inexistente | `errorInfo[1] = 1452` | `409` |
| Pareja repetida en tabla puente | `errorInfo[1] = 1062` | `409` |
| Borrar padre con hijos (solo borrado físico) | `errorInfo[1] = 1451` | `409` |
| Regla propia del procedimiento (`SIGNAL`) | `errorInfo[0] = '45000'` | `409` con el mensaje del `SIGNAL` |

El `getCode()` de PDO no sirve para distinguir los tres primeros: todos llegan
como `'23000'`.

```json
{
    "estado": 409,
    "mensaje": "Conflicto de integridad.",
    "detalle": "No se puede retirar el grupo: primero retire sus semilleros."
}
```

Flujo: el repositorio captura `PDOException` → lanza `ConflictoExcepcion` → el
controlador responde `409`. El usuario nunca ve el SQL.

## 10. Clarificaciones (Compuerta 1)

**Pregunta:** con borrado lógico, ¿se puede retirar un maestro que aún tiene
detalle activo? (Ej.: un grupo con semilleros).

**Decisión: NO.** El procedimiento de retiro revisa si hay hijos activos y, si
los hay, lanza `SIGNAL SQLSTATE '45000'`. La API responde `409` pidiendo
retirar primero el detalle. Esta regla se pone también en el servicio.

Consecuencia: para retirar un grupo se retiran antes sus semilleros y
relaciones. Una fila de tabla puente sí se puede retirar sola.

## 11. Compuertas del método

1. **Clarificaciones:** ningún `[NECESITA ACLARACIÓN]` pendiente antes del plan.
2. **Chequeo de constitución:** al final de `3_plan.md`.
3. **Lista de requisitos:** `9_checklist.md` firmado por una persona.

## 12. Datos de referencia y pruebas

- Los datos de referencia se cargan desde el script de la base, no desde el
  frontend, para probar el maestro-detalle desde el primer día.
- Pruebas mínimas: CRUD de cada maestro; asignar, repetir (409) y retirar en
  cada tabla puente; FK inexistente (409); retirar maestro con hijos (409);
  maestro-detalle todo o nada; servicios con repositorios falsos; tabla vacía
  (204); y la regresión completa de la v1.

## 13. Qué se entrega

- Spec kit en `docs/spec_kit/versiones/v2_<nombre>/` con los nueve documentos
  (solo el **delta**) y la guía de IA.
- Script de la base con `ALTER`, procedimientos y disparador.
- Repositorios que llaman a los procedimientos.
- Interfaz gráfica de los recursos nuevos.
- Colección de pruebas, **incluidas las que deben fallar**.
- `GUIA_IA2.md` con los prompts usados **y lo que tuvieron que corregirle a la IA**.
- Tag `v2`.

## 14. v2 está lista cuando…

- [ ] Las tablas de la v1 y de la v2 tienen CRUD completo y su interfaz.
- [ ] Los cinco verbos de todas las tablas pasan por `CALL`; no hay SQL a mano.
- [ ] Al menos un disparador funciona, comprobado por la interfaz.
- [ ] Cada FK es un desplegable lleno que muestra nombres.
- [ ] La FK opcional acepta «(ninguna)» y manda `null`.
- [ ] Padre inexistente, pareja repetida y retiro con hijos responden `409` en castellano.
- [ ] Las tablas puente se asignan y retiran con sus dos claves.
- [ ] El detalle se ve y se agrega desde el maestro.
- [ ] El borrado es lógico y listados y desplegables filtran `activo = 1`.
- [ ] La interfaz no dice «PUT», «PATCH», «422» ni «FK».
- [ ] La regresión de la v1 pasa completa.
- [ ] Cero secretos en el código.
- [ ] Todo entró a `main` por Pull Request y se creó el tag `v2`.

## 15. Versiones siguientes

| Versión | Qué trae |
|---|---|
| v3 | Login, JWT, roles y control de acceso |
| v4 | Consultas multitabla, dashboard, imagen corporativa y publicación |
