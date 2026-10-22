# Investigación y decisiones — Versión 2

> **Versión 2** · **Lectura opcional** (el porqué de las decisiones del plan,
> con las alternativas que se evaluaron y descartaron). Complementa a
> [3_plan.md](3_plan.md); el orden de trabajo está en [8_tasks.md](8_tasks.md).

> **Estado:** borrador. Los `[NECESITA ACLARACIÓN]` se resuelven con el SQL
> oficial y con el profesor antes de programar.

---

## D1 — El rechazo del motor sale como 409, no como 500

**Alternativas descartadas:** dejarlo como 500 (lo que hace la v1) o
convertirlo en 400.
**Decisión:** **409 Conflicto**, con el motivo en español.
**Por qué:**

En la v1 el 500 estaba bien decidido: la integridad no era tema de esa
versión y traducirla habría sido lógica que nadie pidió. En la v2 la
integridad **es** el tema, así que la balanza cambia. Los tres códigos dicen
cosas distintas:

| | Qué le comunica al que llama |
|---|---|
| **500** | «Se nos dañó algo». Es mentira: el sistema funcionó bien |
| **400** | «Su petición está mal escrita». También es mentira: estaba bien escrita |
| **409** | «Su petición está bien, pero no cabe en los datos que hay» |

La diferencia se nota en la pantalla: con 500 la persona lee «error interno»;
con 409 lee «primero retire los semilleros de este grupo» y sabe qué hacer.

**Precio asumido:** hay que reconocer códigos del motor, y eso ata un archivo
a MariaDB. Se paga aislándolo en `errores_de_integridad.php`.

## D2 — No se pregunta antes: se intenta y se traduce el veredicto

**Alternativa descartada:** que el servicio comprobara primero («¿existe el
grupo 5?») y solo entonces insertara.
**Decisión:** intentar la escritura y traducir el «no» del motor.
**Por qué:**

1. **La comprobación previa no evita el problema, lo hace más raro.** Entre
   el `SELECT` y el `INSERT`, otra persona puede retirar ese grupo. Un error
   que falla siempre se arregla; uno que falla a veces se vuelve leyenda.
2. **Duplica una regla que ya está en la base.** La llave foránea vive allí.
   Repetirla en PHP es mantener el mismo negocio en dos idiomas.

**Precio asumido:** el mensaje sale del código del motor, no del caso
concreto, así que es más general («alguno de los códigos a los que apunta no
existe», sin decir cuál).

## D3 — Un solo maestro-detalle: `grupo_investigacion → semillero`

**Alternativas descartadas:** `universidad → grupo_investigacion`,
`linea_investigacion → docente`, o hacer los tres.
**Decisión:** se hace **uno**, completo.
**Por qué:**

- La guía de la v2 lo pide así: un maestro-detalle bien hecho enseña todo lo
  que hay que aprender; tres a medias enseñan lo mismo, peor hecho.
- Un semillero **no existe sin su grupo**, que es justo lo que define a un
  detalle. `linea_investigacion → docente` es más débil: un docente existe
  por sí solo.
- `universidad → grupo_investigacion` sirve, pero la universidad ya viene de
  la v1 y su pantalla tendría que cambiar. Escoger el grupo deja la v1 intacta.

**Precio asumido:** los otros dos quedan como CRUD corriente, sin pantalla de
maestro con detalle.

`[NECESITA ACLARACIÓN: confirmar con el SQL que semillero tiene una FK al grupo.]`

## D4 — Procedimientos almacenados solo en el maestro-detalle

**Alternativa descartada:** procedimientos en las 10 tablas, que era lo que
mis borradores anteriores proponían.
**Decisión:** los cinco verbos del grupo (`sp_listar_`, `sp_consultar_`,
`sp_crear_`, `sp_actualizar_`, `sp_eliminar_`) van por procedimientos. El resto
usa SQL preparado en el repositorio, como en la v1.
**Por qué:** el procedimiento gana algo de verdad donde hay varias
escrituras: **recibe el detalle completo y lo guarda en una transacción**. Un
CRUD de una sola tabla no gana nada con un procedimiento; se escribe igual de
bien con una sentencia preparada.

**Precio asumido:** el proyecto tiene dos estilos de repositorio. Se compensa
con que el patrón es siempre el mismo: la interfaz no cambia, solo lo que hay
dentro de la clase `*MariaDB`.

## D5 — El detalle viaja como un parámetro JSON

**Alternativas descartadas:** varios `CALL` seguidos (uno por semillero) o un
parámetro por cada semillero.
**Decisión:** el repositorio manda los semilleros como **un solo texto JSON**
y el procedimiento los recorre con `JSON_TABLE`.
**Por qué:**

- Un procedimiento no puede recibir una lista.
- Varios `CALL` rompen el «todo o nada»: si falla el tercero, quedan dos
  guardados.
- Un parámetro por semillero fija cuántos caben (y la pantalla ya limita a
  cinco; no hay que limitar también la API).

**Cómo se garantiza el «todo o nada»:** el procedimiento abre la transacción y
tiene un `EXIT HANDLER FOR SQLEXCEPTION` que hace `ROLLBACK` y `RESIGNAL`.

**Precio asumido:** `JSON_TABLE` pide MariaDB 10.6 o superior (la imagen
`mariadb:11` lo cumple), y es una solución mía que el ejemplo de clase no usa.
`[NECESITA ACLARACIÓN: confirmar con el profesor si acepta JSON_TABLE.]`

## D6 — El SIGNAL se lee al revés, y «no existe» tiene su propio SQLSTATE

**Hallazgo 1.** El `getCode()` de PDO devuelve `'23000'` para 1452, 1062 y
1451 por igual, así que con eso no se distingue un padre inexistente de una
pareja repetida. El número que sirve está en `errorInfo[1]`.

**Hallazgo 2.** Un `SIGNAL` propio llega con el errno genérico 1644, y lo que
lo identifica es el SQLSTATE `'45000'` en `errorInfo[0]`. Si solo se mira el
`[1]`, las reglas que uno mismo escribió en el procedimiento acaban en el 500.

| Qué pasa | Dónde se lee | Respuesta |
|---|---|---|
| Llave o pareja repetida | `errorInfo[1] = 1062` | `409` |
| Apunta a algo que no existe | `errorInfo[1] = 1452` | `409` |
| Otros dependen de esto (borrado físico) | `errorInfo[1] = 1451` | `409` |
| Regla propia del procedimiento | `errorInfo[0] = '45000'` | `409` con el mensaje del `SIGNAL` |
| El grupo no existe o está retirado | `errorInfo[0] = '45001'` | `404` |

**Alternativa descartada:** distinguir «no existe» de «no se deja» leyendo el
**texto** del mensaje. Es frágil: si alguien reescribe el procedimiento con
otras palabras, deja de funcionar.
**Decisión:** un SQLSTATE aparte (`'45001'`) para «no existe».
**Precio asumido:** la guía solo nombra `'45000'`, así que `'45001'` es una
decisión mía. Está aislada en `errores_de_integridad.php`; si el profesor no
la acepta, se cambia ahí por la lectura del texto, declarándolo como
fragilidad.

El 1451 casi no se dispara con borrado lógico (es un `UPDATE`), pero se sigue
traduciendo por seguridad.

## D7 — No se retira un grupo que tiene semilleros activos

**Alternativas descartadas:** retirar también los semilleros en cascada, o
dejarlos huérfanos.
**Decisión:** **se rechaza con 409**: «Primero retire los semilleros de este
grupo».
**Por qué:** con `DELETE FROM`, la base protegía a los hijos con la llave
foránea (error 1451). Con borrado lógico solo hay un `UPDATE`, y la llave
foránea no se entera. Sin una regla propia quedarían semilleros activos de un
grupo retirado, y los listados mostrarían datos incoherentes. La cascada
descartada es peor: la persona perdería datos sin quererlo.

**Dónde vive la regla:** en dos sitios, a distinta altura.

1. **El procedimiento** `sp_eliminar_grupo_investigacion` revisa y lanza el
   `SIGNAL`. Protege a la base de cualquier programa que escriba sin pasar
   por la API.
2. **El servicio** la aplica también, para dar el mensaje antes.

**Precio asumido:** la misma regla en dos idiomas, que es lo que D2 critica.
La diferencia es que aquí la regla de negocio **no** la sabe la llave
foránea, así que alguien tiene que decirla. `[NECESITA ACLARACIÓN: confirmar si se quiere la doble defensa o solo la del procedimiento.]`

## D8 — El grupo no tiene PATCH: se reemplaza entero

**Alternativa descartada:** darle a `IRepositorioGrupoInvestigacion` los mismos
métodos que a las demás fichas.
**Decisión:** `PUT` reemplaza el encabezado y **todo** el detalle; los
semilleros que ya no vienen quedan en `activo = 0`. Un `PATCH` responde 405.
**Por qué:** cambiar un semillero toca lo que el grupo contiene, y un «cambiar
solo un pedacito» del detalle deja dudas: ¿los que no menciono se quedan o se
borran? Con `PUT` la respuesta es una sola.

El 405 sale del enrutador, porque la ruta no está escrita. No hay que
programar nada para que `PATCH` falle.

**Esto demuestra por qué la API es específica:** una ruta genérica
`/api/{tabla}` habría dado `PATCH` al grupo por tener las mismas operaciones
que todas.

`[NECESITA ACLARACIÓN: ¿se necesita DELETE propio del semillero suelto, o basta retirarlo con el PUT del grupo?]`

## D9 — Las tablas puente se asignan y se retiran, no se editan

**Alternativa descartada:** darles `PUT` y `PATCH` como a cualquier ficha.
**Decisión:** `GET` lista, `POST` asigna, `DELETE` con las dos claves retira.
`PUT` y `PATCH` responden 405, y la pantalla no tiene botón de editar.
**Por qué:** su llave **es** la pareja. Cambiar un lado es otra pareja:
retirar una y asignar otra. Editar no tiene sentido.

- Repetir una pareja da el 1062 → `409`.
- Retirar una pareja que no existe → `404`.
- La interfaz del repositorio **no tiene** `actualizar`. Es el principio de
  segregación de interfaces (la «I» de SOLID): no obligar a una clase a
  implementar lo que no hace.

`[NECESITA ACLARACIÓN: ¿las tablas puente llevan activo o se borran físicamente? Cambia el repositorio (UPDATE o DELETE) y el 404 de retirar.]`

## D10 — Los desplegables muestran nombres y mandan claves

**Alternativa descartada:** una casilla de texto donde se teclea el código.
**Decisión:** un `<select>` cargado desde la API.
**Por qué:** teclear un código que hay que ir a buscar a otra pantalla es de
las cosas que hacen odiar un sistema. Cuesta una petición más por
desplegable, y las vale.

Detalles de PHP que costaron una corrida cada uno:

- **Solo ofrece activos**, porque el listado del que sale ya filtra. Ofrecer
  un padre retirado es ofrecer una opción que la base va a rechazar.
- **«(ninguna)» manda `null`.** Un `<select>` vacío llega como cadena vacía, y
  `(int) ''` vale `0`, que no es `null`. La base lo rechaza buscando el
  registro 0.
- **`htmlspecialchars` en todo lo que viene de la base.** Un apellido con
  apóstrofo o con un `<` rompe el HTML sin dar ningún error.
- **El `204` no trae JSON.** `json_decode('')` da `null` y el `foreach`
  revienta en PHP 8.

**Lo que el desplegable NO hace es validar.** La lista pudo cargarse hace un
minuto y esa ficha pudo retirarse; y cualquiera puede mandar un `POST` sin
pasar por la pantalla. Quien defiende la integridad es la base. Confundir «la
pantalla no deja escoger eso» con «eso no puede pasar» es uno de los errores
más caros del software.

## D11 — El formulario del grupo tiene cinco renglones fijos

**Alternativas descartadas:** un botón «agregar renglón» con JavaScript, o una
pantalla de dos pasos (primero el grupo, después los semilleros).
**Decisión:** cinco casillas (`detalle_nombre[]`); las vacías se ignoran.
**Por qué:** un formulario que crezca necesita JavaScript, y esta versión no
lo usa. La pantalla de dos pasos resolvía eso, pero rompía lo que la versión
enseña: **el maestro y su detalle se guardan juntos, en una sola operación.**
Partirla permitiría un grupo a medias.

**Precio asumido, declarado en la spec y no escondido:** un grupo de seis
semilleros no cabe en una sola pasada. Es una limitación **de la pantalla**; la
API acepta los que sean.

## D12 — No se hacen disparadores

**Alternativa descartada:** el contador `total_grupos` con tres triggers y
`sp_recontar_grupos`, que era lo que mis borradores anteriores incluían.
**Decisión:** **ningún disparador.**
**Por qué:** la guía de la v2 dice que no se piden y que no suman nota. El
ejemplo de clase los usa porque su factura calcula totales y mueve stock; el
grupo de investigación no tiene nada así que lo necesite. Copiarlos «para
cumplir» un criterio que no existe es sumar complejidad sin razón.

**Precio asumido:** se pierde la oportunidad de practicar triggers. Si más
adelante se quiere uno, se hace aparte, sin tocar lo que ya funciona.

## D13 — Se agrega `activo` a las tablas de la v2 que no lo tengan

**Alternativa descartada:** borrado físico (`DELETE FROM`).
**Decisión:** `ALTER TABLE ... ADD COLUMN IF NOT EXISTS activo TINYINT(1) NOT NULL DEFAULT 1`
en `db/procedimientos_v2.sql`, y todo `DELETE` pasa a ser `UPDATE ... SET activo = 0`.
**Por qué:** las tablas del módulo no traen la columna; solo `rol` y `usuario`
la tienen. Sin ella no hay borrado lógico, y la v2 entera (listados,
desplegables, `sp_eliminar_*`) está montada sobre él.

**Precio asumido:** cada listado y cada desplegable tienen que acordarse del
`WHERE activo = 1`. Olvidarlo es de los errores más frecuentes, y se nota
porque lo retirado sigue apareciendo.

`ADD COLUMN IF NOT EXISTS` es de MariaDB (en MySQL no funciona). Permite correr
el script varias veces sin error.

## D14 — La documentación y el código dicen lo mismo

**Alternativa descartada:** escribir el spec kit al final, como informe.
**Decisión:** el spec kit se escribe **antes**, y el `9_checklist.md` incluye
una revisión de que cada ruta, código y nombre documentado existe en el
código.
**Por qué:** un spec kit escrito al final es un informe de lo que se hizo, y
no sirvió para decidir nada. Y la sustentación pide explicar un archivo en voz
alta: solo se puede explicar bien lo que se entiende, y lo que se entiende es
lo que está escrito antes.

Para que quien abra la carpeta sepa qué hay: un `README.md` por carpeta con
una frase por archivo, un comentario de apertura en cada archivo PHP (qué es y
qué NO hace) y nombres que dicen lo que hacen (`ControladorGrupoInvestigacion`,
no `Ctrl1`).

## Pendientes

- [ ] El SQL oficial: columnas, llaves y FK de las tablas de la v2.
- [ ] Si `semillero` apunta a `grupo_investigacion` (D3).
- [ ] Si las tablas puente llevan `activo` (D9).
- [ ] Si el profesor acepta el SQLSTATE `'45001'` (D6) y `JSON_TABLE` (D5).
- [ ] Si el semillero necesita `DELETE` propio (D8).
- [ ] Si se quiere la doble defensa del retiro (D7).
- [ ] Si el script oficial tiene errores que corregir. Si los hay, cada
      corrección se escribe aquí con su motivo y cómo se encontró, en una
      decisión nueva (D15 en adelante).
