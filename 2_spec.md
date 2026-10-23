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
> | [9_checklist.md](9_checklist.md) | Lista de cierre firmada |
> | [GUIA_IA2.md](GUIA_IA2.md) | Prompts usados y correcciones a la IA |

## 1. Propósito

La v1 dejó seis catálogos que no dependían de nadie. La v2 enseña qué pasa cuando las tablas se relacionan:

1. **Una ficha apunta a otra.** Un semillero pertenece a un grupo. ¿Qué pasa si el grupo no existe? ¿Y si se quiere retirar un grupo con semilleros?
2. **Nadie digita llaves.** Se escogen de un desplegable con nombres.
3. **Una operación son varias escrituras.** Guardar un grupo con sus semilleros es todo o nada.
4. **Parte del negocio vive en la base.** El maestro-detalle se guarda con procedimientos almacenados.

## 2. Alcance

**Incluye:**

| Tipo | Tablas |
|---|---|
| Maestras | `docente`, `grupo_investigacion`, `semillero` |
| Puente | `participa_grupo`, `participa_semillero`, `grupo_linea`, `semillero_linea`, `ac_linea`, `ods_linea`, `aa_linea` |

- **UN maestro-detalle completo: `grupo_investigacion → semillero`**, con sus cinco verbos por procedimientos almacenados.
- Desplegables en todas las llaves foráneas, cargados desde la API.
- Errores de integridad traducidos a **409** en español, en un solo archivo (`repositorios/errores_de_integridad.php`).
- La regresión completa de la v1.

**No incluye (deliberado):**

- Los otros maestro-detalle (`universidad → grupo_investigacion`, `linea_investigacion → docente`): quedan como CRUD corriente.
- Disparadores: la v2 no los pide.
- Login, JWT, roles y `usuario`/`rol`/`rol_usuario` (v3).
- Consultas multitabla, dashboard y publicación (v4).
- Framework, Composer, otros motores, JavaScript de aplicación.

## 3. Requisitos funcionales

### 3.A La API

- **RF1 — La v1 sigue igual.** Sus seis recursos responden como antes.
- **RF2 — Fichas.** `docente` y `semillero` ofrecen listar (`?limite`), ver uno, `POST`, `PUT`, `PATCH` y `DELETE` en su propia ruta. Sobre `{tabla, limite, total, datos}`; tabla vacía → `204`. `activo` nunca viaja en el body.
- **RF3 — Tablas puente.** `GET` lista, `POST` asigna una pareja, `DELETE /api/<tabla>/{clave1}/{clave2}` retira. Pareja repetida → `409`. Pareja que no existe → `404`. `PUT` y `PATCH` → `405`.
- **RF4 — Maestro-detalle.** `POST /api/grupo_investigacion` recibe el grupo y su lista de semilleros en el mismo body, y devuelve el grupo completo con su detalle. Si un semillero falla, **no se guarda nada**. El error dice **cuál** renglón falla.
- **RF5 — Procedimientos.** Listar, consultar, crear, actualizar y eliminar del grupo van por `CALL sp_..._grupo_investigacion(?, ...)`. El repositorio no escribe SQL a mano.
- **RF6 — Reemplazar.** `PUT /api/grupo_investigacion/{id}` reemplaza el encabezado y todo el detalle. `PATCH` → `405`.
- **RF7 — Retirar.** `DELETE` es borrado lógico. Con semilleros activos → `409` con el mensaje del `SIGNAL`.
- **RF8 — Integridad.** La base defiende, la API traduce; nunca `500`:

  | Caso | Respuesta |
  |---|---|
  | Llave o pareja repetida | `409` |
  | Apunta a algo que no existe | `409` |
  | Eliminar algo de lo que otros dependen | `409` |
  | Regla del procedimiento (`SIGNAL '45000'`) | `409` con su mensaje |
  | Grupo inexistente (`SIGNAL '45001'`) | `404` |

- **RF9 — Diagnóstico.** `GET /` responde `"version": "v2"` y la lista de recursos.

### 3.B El front (`front_php`)

- **RF10 — Una sección por recurso**, con menú desde cualquier pantalla.
- **RF11 — Desplegables.** Cada llave foránea es un `<select>` que muestra el nombre y manda la clave, solo con registros activos. Si es opcional, lleva «(ninguna)» y manda `null`. Todo texto de la base pasa por `htmlspecialchars` (Art. 9).
- **RF12 — El grupo se ve con su detalle.** Sus semilleros aparecen en la misma pantalla y se agregan ahí, en **un solo envío** (campos `detalle_nombre[]`). Son cinco renglones fijos; los vacíos se ignoran. Es una limitación de la pantalla, no de la API.
- **RF13 — Puente.** Se asignan y se retiran; no hay botón de editar.
- **RF14 — Los rechazos se leen en español**, sin números de estado ni jerga.
- **RF15 — Lista vacía.** Un `204` se muestra como «todavía no hay datos».
- **RF16 — La pantalla sobrevive a la API caída** (Art. 1.1): con su menú, su aviso y sin filas.
- **RF17 — Identidad visual.** Los estilos usan `var(--umv-...)` de `marca.css`; los errores van en rojo alerta, no en ocre (Art. 12).

## 4. Requisitos no funcionales

- **RNF1** — Capas estrictas con interfaces e inyección de dependencias; solo `ensamblador.php` hace `new` de clases concretas (Art. 3).
- **RNF2** — PHP puro, sin Composer (Art. 2).
- **RNF3** — SQL siempre con parámetros `?` (Art. 9).
- **RNF4** — El dialecto de MariaDB (`CALL`, números de error) vive solo en las clases `*MariaDB` y en `errores_de_integridad.php`.
- **RNF5** — El front no usa PDO, no depende de `mariadb` y no comparte código con la API (Art. 1.1).
- **RNF6** — La interfaz no dice «PUT», «PATCH», «422», «FK», rutas `/api/` ni nombres de columna. Dice «retirar», no «borrar» (Art. 6).
- **RNF7** — Interpretabilidad: cada archivo tiene su cabecera (qué hace y por qué está ahí) y cada carpeta un `README.md` (Art. 1).
- **RNF8** — Nada de v3 ni v4.

## 5. Criterios de aceptación

| # | Criterio |
|---|---|
| 1 | `docker compose up -d --build` deja base, API (8111) y front (8110) funcionando |
| 2 | Las tablas de la v1 siguen con CRUD completo y pantalla (regresión) |
| 3 | Las tablas de la v2 tienen CRUD y pantalla; las puente se asignan y se retiran |
| 4 | **Los cinco verbos del grupo pasan por procedimientos:** el repositorio tiene `CALL ... (?, ?)` y no SQL a mano |
| 5 | **Maestro-detalle completo:** se abre un grupo, se le agregan 3 semilleros y se guardan en un solo envío; en la base están los tres o ninguno |
| 6 | Cada llave foránea es un desplegable lleno, con nombres y solo activos |
| 7 | La llave opcional acepta «(ninguna)» y manda `null`, no `0` |
| 8 | Crear un hijo con padre inexistente → **409** en castellano (un 500 es criterio fallado) |
| 9 | Repetir una pareja en una puente → **409** |
| 10 | Retirar un grupo con semilleros activos → **409** con el mensaje del procedimiento |
| 11 | El detalle se ve y se agrega desde el maestro |
| 12 | Borrado lógico: desaparece de la lista y **sigue en la base** |
| 13 | Tabla vacía → `204` y la pantalla no se rompe |
| 14 | `PATCH /api/grupo_investigacion/{id}` → **405** |
| 15 | Prueba de capas con repositorios falsos, sin MariaDB |
| 16 | La interfaz no muestra jerga técnica |
| 17 | **La regresión de la v1 pasa completa** |
| 18 | Con la API apagada, las pantallas siguen en pie y sin filas |

## 6. Glosario mínimo

| Término | Significado |
|---|---|
| Llave foránea | Columna que solo acepta valores que existen en otra tabla |
| Tabla puente | Relaciona dos tablas; su llave son las dos llaves foráneas juntas |
| Maestro-detalle | Dos tablas que no tienen sentido por separado: grupo y semilleros |
| Procedimiento almacenado | Operación guardada en la base, que se llama por su nombre |
| Transacción | Varias escrituras que se guardan todas o ninguna |
| 409 Conflicto | «Su petición está bien, pero no cabe en los datos que hay» |

## 7. Definición de TERMINADA

Los 18 criterios pasan → `9_checklist.md` firmado → commit → tag `v2`.

## 8. Clarificaciones (Compuerta 1)

| # | Pregunta | Respuesta |
|---|---|---|
| C1 | ¿Qué maestro-detalle? | **`grupo_investigacion → semillero`.** Un semillero no existe sin su grupo. |
| C2 | ¿Se retira un grupo con semilleros activos? | **No: 409.** El procedimiento lanza `SIGNAL '45000'`; la base no lo impide sola porque es un `UPDATE`. |
| C3 | ¿La API comprueba antes que el padre exista? | **No.** Se intenta y se traduce el veredicto del motor. |
| C4 | ¿El grupo tiene `PATCH`? | **No.** El detalle se reemplaza entero con `PUT`. |
| C5 | ¿Cómo se retira un semillero suelto? | Con el `PUT` del grupo: los que no vienen quedan en `activo = 0`. |
| C6 | ¿Las puente llevan `activo`? | **No.** Se borran físicamente, porque volver a asignar una pareja retirada chocaría con la llave. |
| C7 | ¿Disparadores? | **No**, la v2 no los pide. |
| C8 | ¿Cuántos renglones ofrece el formulario? | **Cinco fijos**; los vacíos se ignoran. |
| C9 | ¿De dónde salen las columnas de `docente`, `semillero` y las puente? | `[NECESITA ACLARACIÓN]` Son propuestas hasta contrastar con `db_scripts/mysql/investigacion.sql` (Art. 5). |
