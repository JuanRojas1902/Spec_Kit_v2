# Contratos — Versión 2: los endpoints y las pantallas

> **Versión 2** · En PHP puro no hay Swagger automático: **este documento ES
> el contrato publicado** (y el endpoint `/` lo enlaza).
>
> | Parte | Base | Secciones |
> |---|---|---|
> | **La API** (lo que devuelve) | `http://localhost:8111` | §0 a §9 |
> | **El front** (lo que se ve) | `http://localhost:8110` | §10 y §11 |
>
> Las dos partes están en el mismo documento a propósito: la §11 es la tabla
> de traducción entre una y otra, y separarlas haría que se desincronizaran.

> **Estado:** los campos de `docente`, `semillero` y las puente son
> **[PROPUESTA]** hasta contrastar con el esquema oficial (Art. 5).

---

## 0. Convenciones

### 0.1 Las tres vías de envío, y los verbos

| Vía | Dónde viaja | Ejemplo | Se usa para |
|---|---|---|---|
| Parámetro de **ruta** | en la URL | `/api/grupo_investigacion/1` | Identificar UNA ficha |
| **Query string** | después de `?` | `/api/docente?limite=3` | Opciones de la consulta |
| **Body** JSON | en el cuerpo | `{"nombres": "…"}` | Los datos de la ficha |

| Verbo | Semántica | Dónde se usa en la v2 |
|---|---|---|
| GET | Leer (nunca modifica) | listar · obtener · diagnóstico |
| POST | Crear / asignar | `docente` · grupo con su detalle · parejas de las puente |
| PUT | **Reemplazar completo** | `docente` · el grupo entero (con su detalle) |
| PATCH | **Actualizar parcial** | `docente` — **el grupo NO** |
| DELETE | Retirar | `docente` · grupo · parejas |

### 0.2 La envoltura de los listados

```json
{ "tabla": "docente", "limite": 1000, "total": 2, "datos": [ … ] }
```

Y **204 sin cuerpo** cuando la tabla está vacía. Un 204 no es un error: es la
respuesta correcta a «no hay filas».

### 0.3 El sobre de los errores

```json
{ "estado": 404, "mensaje": "Grupo no encontrado.", "detalle": "…" }
{ "estado": 422, "mensaje": "Datos inválidos.", "errores": [ "…", "…" ] }
```

| Origen | HTTP |
|---|---|
| Body inválido según la validación del controlador | **422** con `errores: [ … ]` |
| Regla de negocio (`limite ≤ 0`, `PATCH` con body vacío, id no numérico) | 400 |
| La ficha pedida no existe (`SIGNAL '45001'` incluido) | 404 |
| **La base rechaza por integridad** (repetido, referencia rota, dependencias, `SIGNAL '45000'`) | **409** ← nuevo en la v2 |
| El verbo no aplica a esa ruta | 405 |
| Error del motor o falla real | 500 |

**El 409 es lo nuevo:** un 500 dice «se nos dañó algo»; un 409 dice «su
petición está bien escrita, pero no cabe en los datos que hay». En la v1 estos
casos daban 500, y estaba bien para esa versión; el porqué del cambio está en
[4_research.md](4_research.md) D1.

### 0.4 Los campos de cada recurso, con el nombre EXACTO del JSON

| Recurso | Llave | La escribe | Campos del body |
|---|---|---|---|
| `docente` | `cedula` | el cliente | `nombres` texto(100) · `apellidos` texto(100) |
| `grupo_investigacion` | `id` | **la base** | `gruplac` texto(255) · `categoria` texto(50) · `convocatoria` texto(255) · `ambito` texto(255) · `universidad_id` entero · `detalle` lista de `{nombre}` |
| `semillero` | `id` | **la base** | *(solo lectura: nace dentro del grupo)* |
| puentes | las dos claves | el cliente | las dos claves (ver §7) |

**Lo que NINGÚN body lleva:** `activo` (lo cambia `DELETE`) y el `id` de un
recurso con llave generada. Si llegan, se ignoran.

**El `grupo_id` de un semillero tampoco viaja:** el procedimiento lo pone con
`LAST_INSERT_ID()`. El renglón del detalle trae solo el `nombre`.

---

## 1. `/api/docente` — Docentes

Las personas que investigan. Es el padre de `participa_grupo` y
`participa_semillero`.

```
GET /api/docente[?limite=N]
→ 200 { "tabla": "docente", "limite": 1000, "total": N, "datos": [ … ] }
→ 204 si la tabla está vacía
→ 400 si limite <= 0

GET /api/docente/1017245
→ 200 { "cedula": 1017245, "nombres": "Ana", "apellidos": "Torres Gómez" }
→ 400 si la cédula no es un entero mayor que cero
→ 404 si no existe

POST /api/docente
body { "cedula": 1030456, "nombres": "Marta", "apellidos": "Gil Ríos" }
→ 200 { "estado": 200, "mensaje": "Docente creado exitosamente.", "filasAfectadas": 1 }
→ 422 con la lista de errores si el body no cumple
→ 409 si la cédula ya existe

PUT /api/docente/1030456      body con TODOS los campos (menos la cédula)
→ 200 { "estado": 200, "mensaje": "…", "filasAfectadas": 1 }
→ 422 si falta un obligatorio · 404 si no existe

PATCH /api/docente/1030456    body con SOLO los campos a cambiar
→ 200 · 400 si el body viene vacío · 404 si no existe

DELETE /api/docente/1030456
→ 200 { "estado": 200, "mensaje": "…", "filasAfectadas": 1 }
→ 404 si no existe o ya está retirado
```

- La cédula va en la URL y **no se modifica** con `PUT` ni `PATCH`.
- **409** al retirar un docente que participa en grupos o semilleros `[NECESITA ACLARACIÓN: ¿se bloquea o se permite? Hoy las puente no tienen activo, así que la base no lo impide. Decisión del equipo.]`

## 2. `/api/grupo_investigacion` — Grupos (el maestro-detalle)

**Éste es el recurso que no es un CRUD corriente.** Sus cinco verbos pasan por
procedimientos almacenados, no tiene `PATCH`, y sus respuestas devuelven el
grupo **con su detalle**.

```
GET /api/grupo_investigacion
→ 200 { "tabla": "grupo_investigacion", "limite": null, "total": N, "datos": [ … ] }
→ 204 si no hay ninguno
```

**`limite` viene en `null` a propósito.** `sp_listar_grupo_investigacion()` no
recibe parámetros, así que este recurso **no acepta `?limite`**. Aceptarlo y
no usarlo sería peor que decir que no existe. El listado trae solo el
encabezado, sin detalle.

```
GET /api/grupo_investigacion/1
→ 200 {
      "id": 1,
      "gruplac": "COL0001",
      "categoria": "A",
      "convocatoria": "Convocatoria 2024",
      "ambito": "Nacional",
      "universidad_id": 1,
      "detalle": [
        { "id": 1, "nombre": "Semillero de Visión Artificial" },
        { "id": 2, "nombre": "Semillero de Lenguaje Natural" }
      ]
    }
→ 400 si el id no es un entero mayor que cero
→ 404 si no existe o está retirado
```

```
POST /api/grupo_investigacion
body {
  "gruplac": "COL0002",
  "categoria": "B",
  "convocatoria": "Convocatoria 2025",
  "ambito": "Regional",
  "universidad_id": 1,
  "detalle": [
    { "nombre": "Semillero de Robótica" },
    { "nombre": "Semillero de Datos" }
  ]
}
→ 200 { "estado": 200, "mensaje": "Grupo creado exitosamente.",
        "grupo": { …el grupo COMPLETO, igual que el GET, con su id… } }
→ 422 si un campo falta o un renglón está mal:
      "errores": [ "El renglón 2: el nombre es obligatorio." ]
→ 409 si universidad_id apunta a una universidad que no existe
```

**Todo o nada.** El grupo y sus semilleros se guardan en una transacción: si
el segundo semillero falla, no queda ni el grupo. Un grupo sin semilleros
(`"detalle": []`) **sí es válido**.

**Se devuelve el grupo completo, no solo su número.** Quien lo creó necesita
el `id` que generó la base y los `id` de sus semilleros, datos que no mandó.

**El error de un renglón dice CUÁL renglón**, numerado desde 1.

```
PUT /api/grupo_investigacion/1        body igual al del POST
→ 200 { "estado": 200, "mensaje": "Grupo reemplazado.", "grupo": { … } }
→ 422 si falta un campo o un renglón está mal
→ 404 si el id no existe
→ 409 si universidad_id no existe
```

`PUT` reemplaza el encabezado y **TODO el detalle**: los semilleros que no
vienen quedan retirados y los que vienen se guardan de nuevo. **Consecuencia:**
los semilleros cambian de `id` en cada `PUT`.

```
PATCH /api/grupo_investigacion/1
→ 405 Método no permitido
```

**El 405 sale del enrutador sin que el número esté escrito en ninguna parte:**
no hay un `elseif` para `PATCH` en ese bloque. No es un olvido: cambiar un
renglón del detalle deja dudas sobre los que no se mencionan, así que el
detalle se reemplaza entero o no se toca.

```
DELETE /api/grupo_investigacion/1
→ 200 { "estado": 200, "mensaje": "Grupo retirado.", "filasAfectadas": 1 }
→ 404 si no existe o ya está retirado (SIGNAL '45001')
→ 409 si tiene semilleros activos (SIGNAL '45000'):
      "detalle": "Primero retire los semilleros de este grupo."
```

**La regla del 409 la sabe el procedimiento, no la llave foránea.** Con borrado
lógico, retirar un grupo es un `UPDATE` y la llave no se entera. Sin el
`SIGNAL`, quedarían semilleros activos de un grupo retirado.

## 3. `/api/semillero` — Semilleros (solo lectura)

**El detalle no se crea suelto.** Nace y se retira por medio del grupo
(`POST` y `PUT` de `/api/grupo_investigacion`). Este recurso existe para
leer: lo usan los desplegables de las puente `participa_semillero` y
`semillero_linea`.

```
GET /api/semillero[?limite=N]
→ 200 { "tabla": "semillero", "limite": 1000, "total": N, "datos": [
        { "id": 1, "nombre": "Semillero de Visión Artificial", "grupo_id": 1 } ] }
→ 204 si no hay ninguno · 400 si limite <= 0

GET /api/semillero/1
→ 200 { "id": 1, "nombre": "…", "grupo_id": 1 }
→ 404 si no existe o está retirado

POST /api/semillero · PUT · PATCH · DELETE
→ 405 Método no permitido
```

## 4. Las tablas puente: asignar y retirar

Siete tablas, **un mismo patrón**. Una pareja existe o no existe: se asigna y
se retira, **no se edita**.

| Recurso | Clave 1 | Clave 2 |
|---|---|---|
| `/api/participa_grupo` | `docente_cedula` | `grupo_id` |
| `/api/participa_semillero` | `docente_cedula` | `semillero_id` |
| `/api/grupo_linea` | `grupo_id` | `linea_id` |
| `/api/semillero_linea` | `semillero_id` | `linea_id` |
| `/api/ac_linea` | `area_conocimiento_id` | `linea_id` |
| `/api/ods_linea` | `ods_id` | `linea_id` |
| `/api/aa_linea` | `area_aplicacion_id` | `linea_id` |

El ejemplo usa `grupo_linea`; las otras seis son iguales cambiando los nombres.

```
GET /api/grupo_linea
→ 200 { "tabla": "grupo_linea", "limite": 1000, "total": 1, "datos": [
        { "grupo_id": 1, "linea_id": 1 } ] }
→ 204 si no hay ninguna pareja

POST /api/grupo_linea
body { "grupo_id": 1, "linea_id": 2 }
→ 200 { "estado": 200, "mensaje": "Pareja asignada.", "filasAfectadas": 1 }
→ 422 si falta una clave o no es un entero mayor que cero
→ 409 si la pareja ya existe              (error 1062)
→ 409 si el grupo o la línea no existen   (error 1452)

DELETE /api/grupo_linea/1/2
→ 200 { "estado": 200, "mensaje": "Pareja retirada.", "filasAfectadas": 1 }
→ 400 si alguna clave no es un entero mayor que cero
→ 404 si esa pareja no existe

PUT /api/grupo_linea/1/2 · PATCH /api/grupo_linea/1/2
→ 405 Método no permitido
```

- **Las dos claves van en la ruta**, en el orden de la tabla de arriba.
- **Retirar es un `DELETE FROM`**: las puente no llevan `activo`. Si lo
  llevaran, volver a asignar una pareja retirada chocaría con su propia llave.
- **No hay `GET` de una sola pareja:** una pareja no tiene más datos que sus
  dos claves.
- Para `participa_grupo` y `participa_semillero`, la clave 1 es `docente_cedula`.

## 5. Las seis de la v1

`area_conocimiento`, `objetivo_desarrollo_sostenible`, `area_aplicacion`,
`termino_clave`, `universidad` y `linea_investigacion` **responden igual que en
la v1**. Su contrato está en el `6_contratos.md` de la v1 y sus pruebas forman
parte de los criterios de esta versión. Ahora, además, son **los padres que
alimentan los desplegables**.

## 6. `GET /` — Diagnóstico

```
→ 200 { "mensaje": "API de Investigación funcionando", "version": "v2",
        "recursos": ["/api/area_conocimiento", "/api/objetivo_desarrollo_sostenible",
                     "/api/area_aplicacion", "/api/termino_clave",
                     "/api/universidad", "/api/linea_investigacion",
                     "/api/docente", "/api/grupo_investigacion", "/api/semillero",
                     "/api/participa_grupo", "/api/participa_semillero",
                     "/api/grupo_linea", "/api/semillero_linea",
                     "/api/ac_linea", "/api/ods_linea", "/api/aa_linea"],
        "contratos": "docs/spec_kit/versiones/v2_<nombre>/6_contratos.md" }
```

## 7. Los 405 que salen gratis

Ninguna ruta declara qué verbos NO admite. El 405 aparece solo, porque el
enrutador tiene un `else` al final de cada bloque:

| Petición | Respuesta | Por qué |
|---|---|---|
| `PATCH /api/grupo_investigacion/1` | 405 | El detalle se reemplaza entero |
| `POST /api/semillero` | 405 | Un semillero nace dentro de su grupo |
| `PUT /api/grupo_linea/1/2` | 405 | Una pareja no se edita |
| `PUT /api/docente` (la colección) | 405 | Reemplazar «todos los docentes» no significa nada |

La lista de verbos que un recurso admite **es** la lista de sus `elseif`.

## 8. Lo que traduce el repositorio (el «no» de la base)

| Qué pasa | Dónde se lee | Respuesta |
|---|---|---|
| Llave o pareja repetida | `errorInfo[1] = 1062` | `409` «Ya existe …» |
| Apunta a algo que no existe | `errorInfo[1] = 1452` | `409` «Alguno de los códigos a los que apunta … no existe.» |
| Hay dependientes (solo borrado físico) | `errorInfo[1] = 1451` | `409` «No se puede retirar … porque hay otras fichas que dependen de ella.» |
| Regla del procedimiento | `errorInfo[0] = '45000'` | `409` con el texto del `SIGNAL` |
| El grupo no existe | `errorInfo[0] = '45001'` | `404` |

`getCode()` de PDO da `'23000'` para los tres primeros, así que no sirve para
distinguirlos. El `SIGNAL` se lee al revés: su errno es siempre 1644 y lo
identifica el SQLSTATE.

## 9. Estabilidad de este contrato

Los endpoints de las seis tablas de la v1 **no cambiaron**, y eso es parte de
los criterios de aceptación. Y hay una consecuencia que se puede comprobar:
**cambiar de motor no debería tocar ni una línea de este documento.**

---

## 10. Las pantallas del front (`http://localhost:8110`)

Aquí no hay verbos ni códigos de estado: **el usuario no sabe qué es un 409**.

| Dirección | Qué ve la persona | Qué le pide a la API |
|---|---|---|
| `GET /` | El inicio, con todas las secciones | *(nada)* |
| `GET /{recurso}` | La tabla del recurso | `GET /api/{recurso}?limite=1000` |
| `GET /{recurso}/nuevo` | El formulario vacío | los catálogos, si tiene llaves foráneas |
| `POST /{recurso}/nuevo` | Vuelve al listado con «Se agregó», o el formulario con lo escrito y los motivos | `POST /api/{recurso}` |
| `GET /{recurso}/{llave}/editar` | El formulario con la ficha | `GET /api/{recurso}/{llave}` |
| `POST …/editar` con **«Guardar la ficha completa»** | Vuelve al listado | `PUT` con **todos** los campos |
| `POST …/editar` con **«Guardar solo lo que cambié»** | Igual | `PATCH` con **solo lo diligenciado** |
| `POST /{recurso}/{llave}/retirar` | Confirma y vuelve al listado | `DELETE /api/{recurso}/{llave}` |
| `GET /publico/…` | Hojas de estilo, Bootstrap y `marca.css` | *(las sirve el propio front)* |
| cualquier otra | «Esa página no existe», con el marco puesto | *(nada)* |

Donde `{recurso}` es `docentes`, y los seis de la v1 con su dirección de la v1.
Los botones dicen **«Retirar»**, nunca «Borrar» (Art. 6).

### 10.1 Los grupos tienen sus propias pantallas

| Dirección | Qué ve la persona | Qué le pide a la API |
|---|---|---|
| `GET /grupos` | El listado de grupos | `GET /api/grupo_investigacion` |
| `GET /grupos/nuevo` | Encabezado y **cinco renglones de semillero** | `GET /api/universidad` (desplegable) |
| `POST /grupos/nuevo` | Va a la pantalla del grupo creado | `POST /api/grupo_investigacion` |
| **`GET /grupos/{id}`** | **El grupo con sus semilleros, ahí mismo** | `GET /api/grupo_investigacion/{id}` |
| `GET /grupos/{id}/editar` | El formulario cargado, con sus semilleros actuales | `GET /api/grupo_investigacion/{id}` + universidades |
| `POST /grupos/{id}/editar` | **UN solo botón: «Guardar»** | `PUT /api/grupo_investigacion/{id}` |
| `POST /grupos/{id}/retirar` | Confirma avisando que se retiran solo grupos sin semilleros | `DELETE /api/grupo_investigacion/{id}` |

- **Los renglones del detalle** son campos `detalle_nombre[]`. El front los
  recorre a la vez, **descarta los vacíos** y manda una lista en **un solo**
  `POST` o `PUT`.
- **Al editar,** el formulario muestra **todos los semilleros actuales** más
  tres renglones vacíos. Si mostrara solo cinco y el grupo tuviera siete, el
  `PUT` retiraría los dos que no se ven.
- **El grupo no tiene «Guardar solo lo que cambié»** y sí una pantalla de ver
  el detalle. Si el front tuviera una sola pantalla genérica parametrizada por
  recurso, no podría expresar ninguna de las dos diferencias.

### 10.2 Semilleros y puentes

| Dirección | Qué ve la persona |
|---|---|
| `GET /semilleros` | Lista de solo lectura, con el grupo al que pertenece. Sin botones de agregar, editar ni retirar |
| `GET /{puente}` | Las parejas, **con nombres** (no claves), y un botón «Retirar» por pareja |
| `GET /{puente}/nueva` | **Dos desplegables** (uno por lado) y el botón «Asignar» |
| `POST /{puente}/nueva` | Vuelve al listado con «Se asignó», o el formulario con el motivo |
| `POST /{puente}/retirar` | Confirma y vuelve al listado |

**Las puente no tienen botón de editar.** Cambiar una pareja es retirarla y
asignar otra. Los nombres de pantalla son en castellano («Docentes por grupo»,
«Líneas por grupo»…), nunca el nombre de la tabla.

## 11. Cómo traduce el front lo que responde la API

Un solo archivo del front (`cliente_api.php`) conoce esta tabla. Ninguna vista
la conoce, y por eso el día que la API cambie el sobre se cambia en un sitio.

| Lo que responde la API | Lo que hace el front | Lo que ve la persona |
|---|---|---|
| `200` con `{tabla, limite, total, datos}` | Se queda con `datos` | La tabla llena |
| **`204`** (tabla vacía) | `ok = true` con la lista **vacía** | «Todavía no hay datos» y el botón de agregar |
| `200` de una escritura | `ok = true` | El aviso verde en el listado |
| `400` / `404` / `500` con `{estado, mensaje, detalle}` | Junta `mensaje` y `detalle` | El aviso en rojo alerta, en español |
| **`409`** con su `detalle` | **Lo mismo que un 400** | «Primero retire los semilleros de este grupo» |
| `422` con `{estado, mensaje, errores:[…]}` | Se queda con la lista `errores` | Los motivos, uno por línea, **sin perder lo escrito** |
| **Nada** (la API no responde) | `null`, que no es lo mismo que un error | «El servicio no está disponible», y la pantalla sigue en pie |

**Fíjese en la fila del 409: el front NO la distingue de un 400.** Es
deliberado: a quien usa la pantalla le importa el texto, no el número. El
número le importa a quien programa contra la API, y está en la §0.3.

### 11.1 Los desplegables

| Formulario | Campo | De dónde salen las opciones | Lo que se ve | Lo que viaja |
|---|---|---|---|---|
| Grupo | Universidad | `GET /api/universidad` | `nombre` | `universidad_id` |
| Docentes por grupo | Docente · Grupo | `GET /api/docente` · `GET /api/grupo_investigacion` | nombres y apellidos · `gruplac` | `docente_cedula` · `grupo_id` |
| Docentes por semillero | Docente · Semillero | `GET /api/docente` · `GET /api/semillero` | nombres y apellidos · `nombre` | `docente_cedula` · `semillero_id` |
| Líneas por grupo | Grupo · Línea | grupos · `GET /api/linea_investigacion` | `gruplac` · `nombre` | `grupo_id` · `linea_id` |
| Líneas por semillero | Semillero · Línea | semilleros · líneas | `nombre` · `nombre` | `semillero_id` · `linea_id` |
| Áreas de conocimiento por línea | Área · Línea | `GET /api/area_conocimiento` · líneas | `disciplina` · `nombre` | `area_conocimiento_id` · `linea_id` |
| ODS por línea | ODS · Línea | `GET /api/objetivo_desarrollo_sostenible` · líneas | `nombre` · `nombre` | `ods_id` · `linea_id` |
| Áreas de aplicación por línea | Área · Línea | `GET /api/area_aplicacion` · líneas | `nombre` · `nombre` | `area_aplicacion_id` · `linea_id` |

- Solo ofrecen registros **activos**, porque el listado del que salen ya filtra.
- Todo texto que viene de la base pasa por `htmlspecialchars`.
- Ninguna llave foránea de la v2 es opcional (todas son `NOT NULL`). Si el
  esquema oficial trae una opcional, lleva «(ninguna)» y manda `null`, no `0`.
- **El desplegable no valida:** la lista pudo cambiar y cualquiera puede mandar
  un `POST` sin la pantalla. Quien defiende la integridad es la base.

## 12. El contrato está cumplido cuando…

- [ ] Los recursos nuevos responden como dice este documento (incluidos los `405`).
- [ ] Las seis de la v1 siguen respondiendo igual.
- [ ] `POST /api/grupo_investigacion` con 3 semilleros devuelve el grupo con sus 3 y sus `id`.
- [ ] Un renglón inválido da `422` que dice **cuál**, y no deja nada guardado.
- [ ] `DELETE` de un grupo con semilleros da `409` con el texto del procedimiento.
- [ ] Un grupo inexistente da `404`,
