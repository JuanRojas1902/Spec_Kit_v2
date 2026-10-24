# Lista de chequeo de requisitos — Versión 2

> **La compuerta 3** del método. Esta lista revisa **la ESPECIFICACIÓN, no el
> código**: se pasa cuando los documentos 2 a 8 de esta versión están escritos
> y ANTES de programar la primera línea.

## Cómo se usa

- **Las casillas las marca una persona.** Una IA puede ayudar a evaluar y a
  señalar dudas, pero **no puede auto-aprobarse**: quien firma es quien
  responde por la versión.
- Se marca `[x]` solo cuando el criterio se cumple **hoy, en el documento**,
  no "cuando lo arregle".
- **Con una sola casilla en rojo no se escribe código.** Se vuelve al
  documento que la causó, se corrige y se pasa la lista otra vez.
- Funciona mejor en pareja: quien revisa la spec es alguien que no la escribió.

---

## A. Claridad — ¿dice UNA sola cosa?

- [ ] Ningún requisito usa palabras sin definir: *rápido, amigable, eficiente,
      correcto, adecuado, robusto*.
- [ ] No queda ningún marcador `[NECESITA ACLARACIÓN: …]` sin resolver en
      ningún documento de la v2, empezando por las **Clarificaciones** de
      [2_spec.md](2_spec.md) §8.
- [ ] Cada RF explica UNA cosa. Si uno necesita un "y" para entenderse, se
      partió en dos.
- [ ] Los RF no mencionan tecnología ni nombres de clase: el QUÉ está separado
      del CÓMO, que vive en [3_plan.md](3_plan.md).
- [ ] Está escrita **la decisión sobre qué maestro-detalle se hace** (uno solo:
      `grupo_investigacion → semillero`) y por qué.

## B. Medible — ¿se puede verificar?

- [ ] Cada criterio de aceptación de [2_spec.md](2_spec.md) §5 dice un **valor
      concreto**: un número, un código de estado o un texto exacto.
- [ ] Cada criterio se puede comprobar con **un comando o un paso** de
      [7_quickstart.md](7_quickstart.md). Si no hay comando posible, no es
      criterio.
- [ ] Los códigos de error están dichos por su número (400, 404, 405, 409, 422,
      500), no como "responde con un error".
- [ ] Los totales de las pruebas (2 por tabla, grupo 1 con 2 semilleros, pareja
      `(1, 1)`) coinciden con los datos de [5_data_model.md](5_data_model.md) §5.

## C. Completitud — ¿falta algo?

- [ ] Los RF cubren todo lo que promete el propósito: nada del alcance quedó sin
      requisito.
- [ ] [2_spec.md](2_spec.md) tiene su **NO incluye** explícito (triggers, otros
      maestro-detalle, login, consultas multitabla).
- [ ] Cada entrada de [6_contratos.md](6_contratos.md) documenta sus desenlaces
      de **ERROR**, no solo el camino feliz.
- [ ] [6_contratos.md](6_contratos.md) dice el nombre EXACTO de cada campo **en
      el JSON**, incluido el renglón del `detalle`.
- [ ] Cada rechazo de integridad que la especificación promete (1062, 1452,
      1451, `'45000'`, `'45001'`) tiene su **mensaje en español** escrito, no
      solo su código.
- [ ] [5_data_model.md](5_data_model.md) trae los **datos exactos** de los que
      dependen los comandos del smoke test.
- [ ] **La versión tiene su front especificado**, no solo su API: hay requisitos
      de lo que la persona ve y hace, y [6_contratos.md](6_contratos.md) dice
      qué pantalla existe y qué le pide a la API (Art. 1.1).
- [ ] Está dicho **qué hace la pantalla cuando la API no responde**.
- [ ] Está dicho **qué hace la pantalla cuando no hay filas** (`204`), y que eso
      no es un error.
- [ ] Está dicho **qué pasa con lo escrito** cuando una validación falla: no se
      pierde.
- [ ] Cada decisión de [4_research.md](4_research.md) tiene al menos una
      alternativa descartada con su razón.
- [ ] Está dicho cómo se cumple el **Art. 12** (identidad visual: `marca.css`,
      errores en rojo alerta) y el **Art. 6** («retirar», no «borrar»).

## D. Coherencia — ¿los documentos dicen lo mismo?

- [ ] Todo RF de [2_spec.md](2_spec.md) aparece en
      [6_contratos.md](6_contratos.md).
- [ ] Todo lo que promete [6_contratos.md](6_contratos.md) tiene una tarea que
      lo construye en [8_tasks.md](8_tasks.md).
- [ ] Todo criterio de aceptación tiene su comando en
      [7_quickstart.md](7_quickstart.md), **con el mismo número**.
- [ ] Cada pantalla de [6_contratos.md](6_contratos.md) §10 usa un endpoint que
      existe en las secciones de arriba del mismo documento, y ninguna inventa
      uno.
- [ ] Los ejemplos de [6_contratos.md](6_contratos.md) usan datos que existen en
      [5_data_model.md](5_data_model.md).
- [ ] Los **verbos de cada recurso** son los mismos en
      [2_spec.md](2_spec.md), [3_plan.md](3_plan.md) y
      [6_contratos.md](6_contratos.md) (en especial `semillero` y el grupo).
- [ ] Los **puertos y carpetas** son los mismos en todos los documentos: front
      `front_php` 8110, API `api_investigacion` 8111, phpMyAdmin 8105, MariaDB
      13330.
- [ ] El **nombre de la base** es `investigacion` en el script, en el compose y
      en los documentos.
- [ ] [3_plan.md](3_plan.md) no nombra ningún archivo que ninguna tarea
      construya, y ninguna tarea construye un archivo que el plan no liste.
- [ ] El **Chequeo de constitución** de [3_plan.md](3_plan.md) está completo:
      artículos 1 a 12, sin saltarse ninguno.
- [ ] Los documentos de la v1 no contradicen a la v2 ni a la constitución
      (contraseña en el compose, no en `.env`; carpeta `front_php`).

## E. Alcance — ¿no se está anticipando?

- [ ] Ningún documento nombra entidades, motores o pantallas fuera del alcance
      declarado en [2_spec.md](2_spec.md).
- [ ] **Ninguna ruta es genérica**: no aparece `/api/{tabla}` ni nada que reciba
      el nombre del recurso como parámetro, ni en la API ni en el cliente del
      front (Art. 10). *(Las rutas `/{recurso}` de la tabla de pantallas son una
      forma de escribir el patrón: cada recurso tiene la suya escrita.)*
- [ ] **Ningún recurso tiene operaciones que no le corresponden.** El grupo no
      tiene `PATCH`, el semillero no se crea suelto, la pareja de una puente no
      se edita.
- [ ] **Hay UN solo maestro-detalle** con procedimientos almacenados, y las demás
      tablas usan SQL con parámetros `?` en su repositorio.
- [ ] **No hay disparadores** en ningún documento ni en el script (la v2 no los
      pide).
- [ ] **El dialecto del motor no aparece fuera de la capa de datos.** Ni un
      `SELECT`, ni un `CALL`, ni un número de error del motor en un servicio, un
      controlador o una vista.
- [ ] **Está dicho quién escribe cada columna** (cliente, base o procedimiento),
      incluido `grupo_id` del semillero y `activo`.
- [ ] Ningún documento anticipa una versión futura (login, JWT, consultas
      multitabla, otros motores).
- [ ] Las dependencias que nombra [3_plan.md](3_plan.md) son exactamente las que
      permite la constitución: sin framework y sin Composer.

## F. Interpretabilidad — ¿se entiende la carpeta?

- [ ] El plan y las tareas piden una **cabecera** en cada archivo PHP y un
      **`README.md`** por carpeta.
- [ ] Quien firma puede **explicar en voz alta**: por qué el `SIGNAL` se lee en
      `errorInfo[0]`, por qué el desplegable no valida y cómo se garantiza el
      «todo o nada».
- [ ] Está previsto el entregable `GUIA_IA2.md` (prompts usados y correcciones).

---

## Resultado

| | |
|---|---|
| **Revisada por** | *(nombre de quien firma)* |
| **Fecha** | |
| **Casillas en rojo** | |
| **Veredicto** | ⬜ En verde: puede empezar el código · ⬜ En rojo: vuelve a la spec |

> Si el veredicto es rojo, anote aquí qué documento hay que corregir y por qué.

**Documentos a corregir:**

&nbsp;

&nbsp;

**Firma:** _______________________________________
