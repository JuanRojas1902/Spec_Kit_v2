# Especificación — Versión 2: Investigación con llaves foráneas y maestro-detalle

> **Versión 2** del desarrollo incremental ([mapa de versiones](../0_mapa_versiones.md)).
> Rige la constitución del proyecto: [../../1_constitution.md](../../1_constitution.md).
> Parte del estado que dejó la v1: las seis tablas de catálogo, con API y
> pantalla, funcionando. La API y el front siguen construyéndose **a la vez**.
> PHP puro, sin framework.
>
> | Documento de esta versión | Contenido |
> |---|---|
> | **2_spec.md** (este) | QUÉ construir en v2 y sus criterios de aceptación |
> | [3_plan.md](3_plan.md) | CÓMO: estructura, capas y las piezas nuevas |
> | [4_research.md](4_research.md) | Decisiones y alternativas |
> | [5_data_model.md](5_data_model.md) | Las tablas, sus llaves foráneas y los procedimientos |
> | [6_api_contract.md](6_api_contract.md) | Los endpoints y las pantallas, con formatos exactos |
> | [7_quickstart.md](7_quickstart.md) | Arranque y pruebas |
> | [8_tasks.md](8_tasks.md) | Orden de construcción por fases |
> | [9_checklist.md](9_checklist.md) | Lista de cierre, firmada por una persona |
> | [GUIA_IA2.md](GUIA_IA2.md) | Los prompts usados y lo que se le corrigió a la IA |

> **Estado:** borrador para la Compuerta 1. Mientras quede un
> `[NECESITA ACLARACIÓN]`, esta versión no pasa a planeación (ver §8).

---

## 1. Propósito de la v2

La v1 dejó seis catálogos que no dependían de nadie. Esta versión mete al
sistema en el problema que empieza cuando las tablas se relacionan:

1. **Que una ficha apunte a otra.** Un semillero pertenece a un grupo. ¿Qué
   pasa si se apunta a un grupo que no existe? ¿Y si se quiere retirar un
   grupo que tiene semilleros?
2. **Que no se digiten llaves.** La persona no puede adivinar códigos: los
   escoge de un desplegable con nombres.
3. **Que una operación sean varias escrituras.** Guardar un grupo con sus
   semilleros es varias escrituras. Si una falla, no puede quedar un grupo
   a medias.
4. **Que parte del negocio viva en la base.** El maestro-detalle se guarda
   con un procedimiento almacenado.

## 2. Alcance

**Incluye:**

- **Las tablas con llave foránea del módulo**, con su CRUD y su pantalla:

  | Tipo | Tablas |
  |---|---|
  | Maestras | `docente`, `grupo_investigacion`, `semillero` |
  | Tablas puente | `participa_grupo`, `participa_semillero`, `grupo_linea`, `semillero_linea`, `ac_linea`, `ods_linea`, `aa_linea` |

  `[NECESITA ACLARACIÓN: confirmar esta lista contra el mapa de versiones de MI repositorio y contra el SQL oficial.]`

- **UN maestro-detalle completo: `grupo_investigacion → semillero`.** Un
  semillero no existe sin su grupo. Los cinco verbos pasan por procedimientos
  almacenados.
- **Desplegables** en todas las llaves foráneas, cargados desde la API.
- **La traducción de los errores de integridad** a un mensaje en español y a
  un código **409**, en un solo archivo (`repositorios/errores_de_integridad.php`).
- **El campo `activo`** en las tablas de la v2 que no lo tengan (con su `ALTER`).
- **La regresión de la v1**, completa.

**No incluye (y es deliberado):**

- Los otros maestro-detalle del módulo (`universidad → grupo_investigacion`
  y `linea_investigacion → docente`): quedan como CRUD corriente.
- Disparadores: no se piden en esta versión.
- Login, contraseñas, tokens, roles y el CRUD de `usuario`, `rol` y
  `rol_usuario` (v3).
- Consultas de varias tablas, dashboard e imagen corporativa (v4).
- Framework, Composer, otros motores y fábrica de repositorios.
- JavaScript de aplicación: las pantallas siguen siendo formularios
  corrientes.

## 3. Requisitos funcionales

### 3.A — La API

#### RF1 — Las seis tablas de la v1
Siguen respondiendo igual. Sus pruebas forman parte de los criterios de esta
versión.

#### RF2 — Las fichas nuevas
`docente` y cada tabla maestra nueva ofrecen las mismas seis operaciones que
en la v1, cada una en **su propia ruta** (`/api/docente`, etc.):

- Listar (`GET`, con `?limite`), obtener (`GET /{llave}`), crear (`POST`),
  reemplazar (`PUT`), actualizar parcial (`PATCH`) y eliminar (`DELETE`).
- Envoltura `{tabla, limite, total, datos}` y **204** cuando la tabla está vacía.
- `activo` nunca viaja en el body.

#### RF3 — Las tablas puente se asignan y se retiran
Cada tabla puente tiene una llave compuesta (dos llaves foráneas):

- `GET /api/<tabla>` lista las parejas activas.
- `POST /api/<tabla>` asigna una pareja.
- `DELETE /api/<tabla>/{clave1}/{clave2}` retira la pareja.
- Repetir una pareja responde **409**.
- Retirar una pareja que no existe responde **404**.
- `PUT` y `PATCH` responden **405**: una pareja existe o no existe, no se edita.

`[NECESITA ACLARACIÓN: nombres reales de las columnas de cada tabla puente y si llevan activo o se borran físicamente.]`

#### RF4 — El maestro-detalle: grupo con sus semilleros
`POST /api/grupo_investigacion` recibe el encabezado del grupo y la lista de
semilleros **en el mismo body**. Todo se guarda en una transacción dentro del
procedimiento. Devuelve **el grupo completo**, con su detalle.

- Si un semillero falla, **no se guarda nada**.
- Cada renglón se valida por separado y el error dice **cuál** falla.
- Un grupo puede crearse con detalle vacío `[NECESITA ACLARACIÓN: ¿se exige al menos un semillero?]`.

#### RF5 — Los cinco verbos del maestro-detalle van por procedimientos
| Operación | Procedimiento |
|---|---|
| Listar | `sp_listar_grupo_investigacion` |
| Consultar (con su detalle) | `sp_consultar_grupo_investigacion` |
| Crear | `sp_crear_grupo_investigacion` |
| Actualizar | `sp_actualizar_grupo_investigacion` |
| Eliminar | `sp_eliminar_grupo_investigacion` |

El repositorio **solo llama** (`CALL ... (?, ?)`): no escribe SQL a mano.

#### RF6 — Reemplazar el grupo reemplaza su detalle
`PUT /api/grupo_investigacion/{id}` reemplaza el encabezado y **todo** el
detalle. Los semilleros que ya no vienen quedan retirados (`activo = 0`).
No existe `PATCH` del grupo: un `PATCH` responde **405**.

#### RF7 — Retirar un grupo
`DELETE /api/grupo_investigacion/{id}` es un borrado lógico. **Si el grupo
tiene semilleros activos, se rechaza con 409**, con el mensaje del `SIGNAL`
del procedimiento: «Primero retire los semilleros de este grupo».

#### RF8 — La integridad la defiende la base, y la API la explica
| Lo que se intentó | Lo que responde |
|---|---|
| Crear con una llave que ya existe, o repetir una pareja | 409 «Ya existe …» |
| Apuntar a un código que no existe | 409 «Alguno de los códigos a los que apunta … no existe.» |
| Eliminar algo de lo que otros dependen | 409 «No se puede eliminar … porque hay otras fichas que dependen de ella.» |
| Romper una regla del procedimiento (`SIGNAL`) | 409 con el mensaje del procedimiento |

La API **no comprueba antes**: intenta y traduce el veredicto. Un 500 nunca
es la respuesta a un rechazo de integridad.

#### RF9 — Diagnóstico
`GET /` responde mensaje, versión (`"v2"`), la lista de recursos y la ruta
de los contratos.

### 3.B — El front

> Aquí no hay verbos ni códigos de estado: **el usuario no sabe qué es un 409**.

#### RF10 — Una sección por recurso
Cada recurso tiene su propia dirección, y el menú lleva a todos desde cualquier
pantalla.

#### RF11 — Las llaves foráneas se escogen, no se escriben
Cada campo que apunta a otra tabla es un **desplegable** cargado desde la API.

- Muestra el **nombre** y manda la **clave**.
- Solo ofrece registros **activos**.
- Si la llave es opcional, la primera opción es **«(ninguna)»** y manda `null`.
- Todo texto que viene de la base se imprime con `htmlspecialchars`.

#### RF12 — El grupo se ve con su detalle
Al abrir un grupo, **sus semilleros aparecen ahí mismo**, y se le pueden
agregar renglones sin salir de la pantalla. Todos los renglones nuevos viajan
en **un solo envío** (campos `detalle_xxx[]`).

El detalle se ofrece en **cinco renglones fijos** y los vacíos se ignoran. Es
una limitación de la pantalla (no usa JavaScript), no de la API.

#### RF13 — Las tablas puente
Se asignan y se retiran. No tienen botón de editar.

#### RF14 — Los rechazos se ven en español
Cuando la API responde 409, la pantalla muestra el motivo dentro de la
aplicación, sin números de estado ni jerga.

#### RF15 — Las listas vacías no rompen la pantalla
Un `204` se muestra como «todavía no hay datos». No se pierde lo que la
persona escribió cuando hay un error.

#### RF16 — La pantalla sobrevive a la API caída
Con la API apagada, las pantallas siguen respondiendo, con su menú y su aviso,
y **sin una sola fila**.

## 4. Requisitos no funcionales

- **RNF1 — Capas estrictas:** el controlador no toca SQL; el servicio no
  conoce HTTP; el repositorio no conoce HTTP. Con interfaces y inyección de
  dependencias (SOLID).
- **RNF2 — PHP puro:** sin framework, sin Composer, sin `vendor/`.
- **RNF3 — SQL siempre con parámetros** (`?`), nunca concatenado.
- **RNF4 — Errores uniformes:** el mismo sobre de la v1, más el 409.
- **RNF5 — El dialecto de MariaDB vive solo en las clases `*MariaDB`:** los
  `CALL` y los números de error no salen de ahí.
- **RNF6 — El front no toca la base ni el código de la API.**
- **RNF7 — El front no habla en jerga:** ninguna pantalla muestra «PUT»,
  «PATCH», «422», «FK», rutas `/api/` ni nombres de columna.
- **RNF8 — Interpretabilidad:** quien abre la carpeta entiende qué hay y qué
  hace cada archivo. Nombres claros, un README por carpeta y comentarios
  sencillos. La documentación y el código dicen lo mismo.
- **RNF9 — Sin anticipación:** nada de v3 ni v4.

## 5. Criterios de aceptación

| # | Criterio |
|---|---|
| 1 | `docker compose up -d --build` deja corriendo la base, la API y el front; `GET /` lista los recursos y el front abre su inicio |
| 2 | Las tablas de la v1 tienen CRUD completo y su pantalla (regresión) |
| 3 | Las tablas de la v2 tienen CRUD completo y su pantalla; las puente se asignan y se retiran |
| 4 | **Los cinco verbos del maestro-detalle pasan por procedimientos almacenados:** en el repositorio hay `CALL ... (?, ?)` y no hay SQL a mano |
| 5 | **El maestro-detalle funciona de punta a punta:** se abre un grupo, se le agregan tres semilleros y se guardan en **un solo envío**. Se mira la base: están los tres, o no está ninguno |
| 6 | Cada llave foránea se elige de un **desplegable lleno**, con nombres y solo activos |
| 7 | La llave foránea opcional acepta **«(ninguna)»** y manda `null` (no `0`) |
| 8 | Crear un hijo con un padre inexistente responde **409** en castellano. Un 500 es criterio fallado |
| 9 | Repetir una pareja en una tabla puente responde **409** |
| 10 | Retirar un grupo con semilleros activos responde **409** con el mensaje del procedimiento |
| 11 | El detalle se ve y se agrega **desde el maestro** |
| 12 | El borrado es lógico: el registro desaparece de la lista y **sigue en la base**. Listados y desplegables filtran `activo = 1` |
| 13 | Una tabla vacía responde `204` y la pantalla no se rompe |
| 14 | `PATCH /api/grupo_investigacion/{id}` responde **405** |
| 15 | Prueba de capas: los servicios se prueban con repositorios **falsos**, sin MariaDB |
| 16 | La interfaz no dice «PUT», «PATCH», «422» ni «FK» en ninguna pantalla |
| 17 | **La regresión de la v1 pasa completa** |
| 18 | Con la API apagada, las pantallas siguen respondiendo, sin filas |

## 6. Glosario mínimo

| Término | Significado |
|---|---|
| **Llave foránea** | Columna que solo acepta valores que existen en otra tabla |
| **Tabla puente** | Tabla que relaciona dos tablas; su llave está formada por dos llaves foráneas |
| **Maestro-detalle** | Dos tablas que no tienen sentido por separado: el grupo y sus semilleros |
| **Procedimiento almacenado** | Operación guardada en la base, que se llama por su nombre |
| **Transacción** | Varias escrituras que se guardan todas o ninguna |
| **Borrado lógico** | Retirar un registro poniendo `activo = 0`, sin borrar la fila |
| **409 Conflicto** | «Su petición está bien escrita, pero no cabe en los datos que hay». Distinto de un 500, que es «se rompió algo» |

## 7. Definición de TERMINADA

Los criterios pasan → `9_checklist.md` firmado → commit → tag `v2`. Hasta
entonces no se escribe la spec de la v3.

## 8. Clarificaciones (Compuerta 1)

| # | La pregunta | La respuesta acordada |
|---|---|---|
| C1 | ¿Qué maestro-detalle se hace? | **`grupo_investigacion → semillero`.** El semillero no existe sin su grupo. Una sola relación bien hecha enseña lo mismo que tres a medias. `[NECESITA ACLARACIÓN: confirmar con el SQL que semillero tiene la FK al grupo]` |
| C2 | Con borrado lógico, ¿se puede retirar un grupo con semilleros activos? | **No: 409.** El procedimiento revisa y lanza `SIGNAL SQLSTATE '45000'`. La regla también va en el servicio, porque la base no lo impide sola (es un `UPDATE`) |
| C3 | ¿La API comprueba antes que el grupo exista? | **No.** Se intenta y se traduce el veredicto. Comprobar antes no evita el problema y duplica la regla en PHP y en la base |
| C4 | ¿El grupo tiene `PATCH`? | **No.** El detalle se reemplaza entero con `PUT`; un `PATCH` responde 405 |
| C5 | ¿Cómo se retira un semillero suelto? | Con el `PUT` del grupo: los que no vienen quedan en `activo = 0`. `[NECESITA ACLARACIÓN: ¿o se necesita DELETE propio del semillero?]` |
| C6 | ¿Cuántos renglones ofrece el formulario? | **Cinco fijos**, los vacíos se ignoran. Es limitación de la pantalla, declarada, no de la API |
| C7 | ¿Qué pasa con las tablas puente y `activo`? | `[NECESITA ACLARACIÓN: decidir si llevan activo o se borran físicamente, y escribirlo en 5_data_model.md]` |
| C8 | ¿Se hacen disparadores? | **No.** No se piden en esta versión y no suman nota |
| C9 | ¿Cuál numeración de versiones vale? | `[NECESITA ACLARACIÓN: el mapa de mi repositorio dice que la v1 es solo area_conocimiento; la guía de la v2 dice que la v1 son todas las tablas sin FK. Confirmar con el profesor]` |
