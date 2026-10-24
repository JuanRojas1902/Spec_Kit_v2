# Especificación — Versión 2: Investigación con llaves foráneas y maestro-detalle

> **Versión 2.** Rige la [constitución](../../1_constitution.md).
> Parte de la v1: seis catálogos con API y pantalla, funcionando.
> API y front se construyen **a la vez** (Art. 1.1). PHP puro, sin framework (Art. 2).
>
> | Documento | Contenido |
> |---|---|
> | **2_spec.md** (este) | QUÉ construir y sus criterios |
> | [3_plan.md](3_plan.md) | CÓMO |
> | [4_research.md](4_research.md) | Decisiones y alternativas |
> | [5_data_model.md](5_data_model.md) | Tablas, llaves y procedimientos |
> | [6_contratos.md](6_contratos.md) | Endpoints y pantallas |
> | [7_quickstart.md](7_quickstart.md) | Arranque y pruebas |
> | [8_tasks.md](8_tasks.md) | Orden de construcción |
> | [9_checklist.md](9_checklist.md) | Lista de chequeo de la spec, firmada |
> | [GUIA_IA2.md](GUIA_IA2.md) | Prompts usados y correcciones a la IA |

## 1. Propósito

La v1 dejó seis catálogos que no dependían de nadie. La v2 enseña qué pasa cuando las tablas se relacionan:

1. **Una ficha apunta a otra.** Un semillero pertenece a un grupo. ¿Qué pasa si el grupo no existe? ¿Y si se quiere retirar un grupo con semilleros?
2. **Nadie digita llaves.** Se escogen de un desplegable con nombres.
3. **Una operación son varias escrituras.** Guardar un grupo con sus semilleros es todo o nada.
4. **Parte del negocio vive en la base.** El maestro-detalle se guarda con procedimientos almacenados.
5. **No todo recurso es un CRUD.** El grupo no tiene `PATCH`, el semillero no se crea suelto y una pareja de una tabla puente no se edita.

## 2. Alcance

**Incluye:**

| Clase | Tablas | Qué admite |
|---|---|---|
| Ficha | `docente` | Los seis verbos de la v1 |
| Maestro-detalle | `grupo_investigacion` (con su detalle `semillero`) | Todo menos `PATCH`; por procedimientos almacenados |
| Solo lectura | `semillero` | Solo `GET`: nace y se retira por medio del grupo |
| Puente | `participa_grupo`, `participa_semillero`, `grupo_linea`, `semillero_linea`, `ac_linea`, `ods_linea`, `aa_linea` | Listar, asignar y retirar |

- **UN maestro-detalle completo: `grupo_investigacion → semillero`**, con sus cinco verbos por procedimientos almacenados.
- Desplegables en todas las llaves foráneas, cargados desde la API.
- Errores de integridad traducidos a **409** en español, en un solo archivo (`repositorios/errores_de_integridad.php`).
- La regresión completa de la v1.

**No incluye (deliberado):**

- Los otros maestro-detalle (`universidad → grupo_investigacion`, `linea_investigacion → docente`): quedan como CRUD corriente de sus tablas.
- Disparadores: la v2 no los pide.
- Login, JWT, roles y `usuario`/`rol`/`rol_usuario` (v3).
- Consultas multitabla, dashboard y publicación (v4).
- Framework, Composer, otros motores, JavaScript de aplicación.

## 3. Requisitos funcionales

### 3.A La API

- **RF1 — La v1 sigue igual.** Sus seis recursos responden como antes.
- **RF2 — Ficha `docente`.** Ofrece listar (`?limite`), ver uno, `POST`, `PUT`, `PATCH` y `DELETE` en `/api/docente`. Sobre `{tabla, limite, total, datos}`; tabla vacía → `204`. Cédula no entera → `400`. `activo` nunca viaja en el body.
- **RF3 — Tablas puente.** `GET` lista, `POST` asigna una pareja, `DELETE /api/<tabla>/{clave1}/{clave2}` retira. Pareja repetida → `409`. Pareja que no existe → `404`. `PUT` y `PATCH` → `405`.
- **RF4 — Maestro-detalle.** `POST /api/grupo_investigacion` recibe el grupo y su lista de semilleros en el mismo body y devuelve el grupo completo con su detalle y sus `id`. Si un semillero falla, **no se guarda nada**. El error dice **cuál** renglón falla. Un grupo sin semilleros (`"detalle": []`) es válido.
- **RF5 — Procedimientos.** Listar, consultar, crear, actualizar y eliminar del grupo van por `CALL sp_..._grupo_investigacion(?, ...)`. El repositorio no escribe SQL a mano.
- **RF6 — Reemplazar.** `PUT /api/grupo_investigacion/{id}` reemplaza el encabezado y todo el detalle. `PATCH` → `405`.
- **RF7 — Retirar.** `DELETE` del grupo es borrado lógico. Con semilleros activos → `409` con el mensaje del `SIGNAL`. Grupo inexistente o ya retirado → `404`.
- **RF8 — Semillero de solo lectura.** `GET /api/semillero` y `GET /api/semillero/{id}` leen; `POST`, `PUT`, `PATCH` y `DELETE` → `405`. Un semillero nace y se retira solo por medio del grupo.
- **RF9 — Integridad.** La base defiende, la API traduce; nunca `500`:

  | Caso | Respuesta |
  |---|---|
  | Llave o pareja repetida | `409` |
  | Apunta a algo que no existe | `409` |
  | Eliminar algo de lo que otros dependen | `409` |
  | Regla del procedimiento (`SIGNAL '45000'`) | `409` con su mensaje |
  | Grupo inexistente (`SIGNAL '45001'`) | `404` |

- **RF10 — Diagnóstico.** `GET /` responde `"version": "v2"` y la lista de los 16 recursos.

### 3.B El front (`front_php`)

- **RF11 — Desplegables.** Cada llave foránea es un `<select>` que muestra el nombre y manda la clave, solo con registros activos. Todo texto de la base pasa por `htmlspecialchars` (Art. 9). Si alguna llave fuera opcional, llevaría «(ninguna)» y mandaría `null`; hoy ninguna lo es.
- **RF12 — El grupo se ve con su detalle.** Sus semilleros aparecen en la misma pantalla. Se agregan en **un solo envío** (campos `detalle_nombre[]`), con cinco renglones fijos; los vacíos se ignoran. Al editar, el formulario muestra **todos** los semilleros actuales más renglones vacíos. Es una limitación de la pantalla, no de la API.
- **RF13 — Semilleros y puente.** La pantalla de semilleros es de solo lectura. Las puente muestran las parejas con **nombres**, se asignan con dos desplegables y se retiran; no hay botón de editar.
- **RF14 — Una sección por recurso**, con menú desde cualquier pantalla.
- **RF15 — Los rechazos se leen en español**, sin números de estado ni jerga, y sin perder lo escrito.
- **RF16 — Lista vacía.** Un `204` se muestra como «todavía no hay datos» y no es un error.
- **RF17 — La pantalla sobrevive a la API caída** (Art. 1.1): con su menú, su aviso y sin filas.
- **RF18 — Identidad visual.** Estilos con `var(--umv-...)` de `marca.css`; errores en rojo alerta, no en ocre; el borrado lógico dice «Retirar» (Arts. 6 y 12).

## 4. Requisitos no funcionales

- **RNF1** — Capas estrictas con interfaces e inyección de dependencias; solo `ensamblador.php` hace `new` de clases concretas (Art. 3).
- **RNF2** — PHP puro, sin Composer (Art. 2).
- **RNF3** — SQL siempre con parámetros `?` (Art. 9).
- **RNF4** — El dialecto de MariaDB (`CALL`, números de error) vive solo en las clases `*MariaDB` y en `errores_de_integridad.php`.
- **RNF5** — El front no usa PDO, no depende de `mariadb` y no comparte código con la API (Art. 1.1).
- **RNF6** — La interfaz no dice «PUT», «PATCH», «409», «FK», rutas `/api/` ni nombres de columna.
- **RNF7** — Interpretabilidad: cada archivo tiene su cabecera (qué es, por qué está ahí y qué NO hace) y cada carpeta un `README.md` (Art. 1).
- **RNF8** — Ninguna ruta es genérica (Art. 10).
- **RNF9** — Nada de v3 ni v4.

## 5. Criterios de aceptación

La columna «Comando» dice dónde se comprueba en [7_quickstart.md](7_quickstart.md).

| # | Criterio | Comando |
|---|---|---|
| 1 | `docker compose up -d --build` deja base, API (`8111`) y front (`8110`) funcionando, y `GET /` responde `"version": "v2"` con 16 recursos | §1, §3.1 |
| 2 | Las seis tablas de la v1 siguen con CRUD completo (regresión) | §3.1 |
| 3 | `docente` tiene CRUD (cédula repetida → `409`, cédula `abc` → `400`) y las puente se asignan y se retiran | §3.2, §3.5 |
| 4 | **Los cinco verbos del grupo pasan por procedimientos:** el repositorio tiene `CALL ... (?, ...)` y no SQL a mano | §3.3 + revisar el repositorio |
| 5 | **Maestro-detalle completo:** un grupo con 3 semilleros se guarda en una petición y devuelve sus `id`; con `universidad_id` 999 no queda nada guardado | §3.3 |
| 6 | Cada llave foránea es un desplegable lleno, con nombres y solo activos | §4.2 (paso 3 y 6) |
| 7 | Un recurso no admite los verbos que no le corresponden: `POST /api/semillero` → `405` y `PUT` a una puente → `405` | §3.5, §3.6 |
| 8 | Crear con un padre inexistente → **409** en castellano (un `500` es criterio fallado) | §3.3, §3.5 |
| 9 | Repetir una pareja en una puente → **409** | §3.5 |
| 10 | Retirar un grupo con semilleros activos → **409** con «Primero retire los semilleros de este grupo»; un grupo inexistente → **404** | §3.4 |
| 11 | El detalle se ve y se agrega desde el maestro, y editar no pierde semilleros | §4.2 (pasos 3 y 4) |
| 12 | Borrado lógico: desaparece de la lista y **sigue en la base** con `activo = 0` | §5 |
| 13 | Tabla vacía → `204` y la pantalla no se rompe | §3.5, §4.2 (paso 7) |
| 14 | `PATCH /api/grupo_investigacion/{id}` → **405** | §3.3 |
| 15 | Prueba de capas con repositorios falsos, sin MariaDB | §3.7 |
| 16 | La interfaz no muestra jerga técnica y dice «Retirar» | §4.1, §4.2 |
| 17 | **La regresión de la v1 pasa completa** | §3.1 |
| 18 | Con la API apagada, las pantallas siguen en pie y sin filas | §4.2 (paso 10) |

## 6. Glosario mínimo

| Término | Significado |
|---|---|
| Llave foránea | Columna que solo acepta valores que existen en otra tabla |
| Tabla puente | Relaciona dos tablas; su llave son las dos llaves foráneas juntas |
| Maestro-detalle | Dos tablas que no tienen sentido por separado: grupo y semilleros |
| Procedimiento almacenado | Operación guardada en la base, que se llama por su nombre |
| Transacción | Varias escrituras que se guardan todas o ninguna |
| Borrado lógico | Retirar un registro con `activo = 0`, sin borrar la fila |
| 409 Conflicto | «Su petición está bien, pero no cabe en los datos que hay» |

## 7. Definición de TERMINADA

Los 18 criterios pasan → `9_checklist.md` firmado → commit → tag `v2`.

## 8. Clarificaciones (Compuerta 1)

| # | Pregunta | Respuesta |
|---|---|---|
| C1 | ¿Qué maestro-detalle? | **`grupo_investigacion → semillero`.** Un semillero no existe sin su grupo. |
| C2 | ¿Se retira un grupo con semilleros activos? | **No: 409.** El procedimiento lanza `SIGNAL '45000'`; la base no lo impide sola porque es un `UPDATE`. |
| C3 | ¿La API comprueba antes que el padre exista? | **No.** Se intenta y se traduce el veredicto del motor. |
| C4 | ¿El grupo tiene `PATCH`? | **No.** El detalle se reemplaza entero con `PUT`; `PATCH` → `405`. |
| C5 | ¿Cómo se crea o retira un semillero? | **Solo por medio del grupo:** el `POST` lo crea y el `PUT` retira los que ya no vienen (`activo = 0`). `semillero` es de solo lectura. |
| C6 | ¿Las puente llevan `activo`? | **No.** Se borran físicamente, porque volver a asignar una pareja retirada chocaría con la llave. |
| C7 | ¿Disparadores? | **No**, la v2 no los pide. |
| C8 | ¿Cuántos renglones ofrece el formulario? | **Cinco fijos** al crear; al editar, todos los actuales más vacíos. |
| C9 | ¿De dónde salen las columnas de `docente`, `semillero` y las puente? | `[NECESITA ACLARACIÓN]` Son propuestas hasta contrastar con `db_scripts/mysql/investigacion.sql` (Art. 5). |
| C10 | ¿Se retira un docente que participa en grupos o semilleros? | **Sí se permite.** Las puente no llevan `activo`, así que la base no lo impide; el docente retirado deja de aparecer en los desplegables. Decisión del equipo, sujeta a confirmación del profesor. |
| C11 | ¿Alguna llave foránea es opcional? | **No.** Todas son `NOT NULL`; por eso no hay criterio de «(ninguna)». Si el esquema oficial trae una opcional, se agrega el criterio. |
