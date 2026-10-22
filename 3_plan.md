# Plan técnico — Versión 2: Investigación con llaves foráneas y maestro-detalle

> **Versión 2** · CÓMO construir lo especificado en [2_spec.md](2_spec.md).
> El porqué de cada decisión: [4_research.md](4_research.md) · contratos
> exactos: [6_api_contract.md](6_api_contract.md) · orden: [8_tasks.md](8_tasks.md).

> **Estado:** borrador para la Compuerta 2. Los `[NECESITA ACLARACIÓN]` se
> resuelven con el SQL oficial antes de programar.

---

## 1. Stack

El mismo de la v1, sin una dependencia nueva. Lo que cambia no es la
tecnología: es cuántas piezas hay y cómo se relacionan.

| Pieza | Elección | Qué cambia respecto de la v1 |
|---|---|---|
| Lenguaje | **PHP 8.3**, `declare(strict_types=1)` | nada |
| Framework / paquetes | **NINGUNO** | nada |
| Acceso a datos | **PDO** con parámetros `?` | **+ `CALL` a procedimientos almacenados** solo para `grupo_investigacion` y su detalle |
| Front | PHP puro, plantillas, Bootstrap local | una sección por recurso nuevo |
| Motor | **MariaDB**, y solo MariaDB | nada |

## 2. Estructura de carpetas

```text
(raíz del proyecto)
├── README.md                          # qué es el proyecto y cómo arrancarlo
├── docker-compose.yml                 # mariadb + api + front + phpmyadmin
├── db/
│   ├── init.sql                       # la BD completa, PROVISTA
│   └── procedimientos_v2.sql          # ALTER de activo + los 5 sp_ del grupo
│
├── api_investigacion/                 # puerto [NECESITA ACLARACIÓN]
│   ├── README.md                      # qué hay en cada carpeta
│   ├── index.php                      # front controller: un bloque de rutas por recurso
│   ├── modelos/
│   │   ├── (los 6 de la v1)
│   │   ├── Docente.php  Semillero.php
│   │   ├── GrupoInvestigacion.php     # el ENCABEZADO
│   │   └── ParticipaGrupo.php ...     # una clase por tabla puente (7)
│   ├── controladores/                 # uno por recurso
│   ├── servicios/
│   │   ├── IServicio*.php  Servicio*.php      # un par por recurso
│   │   └── ensamblador.php            # una función por recurso
│   ├── repositorios/
│   │   ├── IRepositorio*.php  Repositorio*MariaDB.php   # un par por recurso
│   │   └── errores_de_integridad.php  ← LO NUEVO: traduce el veredicto del motor
│   ├── excepciones/
│   │   ├── NoEncontradoExcepcion.php            # de la v1
│   │   └── ConflictoDeIntegridadExcepcion.php   ← LO NUEVO
│   └── pruebas/
│       └── prueba_capas.php           # servicios con repositorios falsos
│
├── front_investigacion/               # puerto [NECESITA ACLARACIÓN]
│   ├── README.md
│   ├── index.php                      # un bloque de pantallas por recurso
│   ├── cliente_api.php                # una función por operación
│   ├── vistas/
│   │   ├── plantilla.php  inicio.php  no_encontrada.php
│   │   ├── {recurso}_lista.php  {recurso}_formulario.php
│   │   ├── puente_lista.php ...       # las puente: lista y asignar, sin editar
│   │   └── grupos_detalle.php         ← la pantalla del maestro con sus semilleros
│   └── publico/                       # Bootstrap guardado aquí, sin CDN
│
└── pruebas_humo/
    └── humo_front.py                  # el recorrido desde la pantalla
```

`[NECESITA ACLARACIÓN: nombres exactos de las tablas y puertos. Unificar front_investigacion / front_php: en la v1 usé los dos.]`

**Fíjese en el patrón:** las fichas simples (`docente`, `semillero` suelto)
repiten las mismas piezas de la v1. Las tablas puente tienen las mismas
carpetas con contenidos distintos: sin `PUT` ni `PATCH`. Y el grupo es el
único recurso con **tres** diferencias: dos modelos que viajan juntos, sus
repositorios llaman procedimientos, y su pantalla muestra el detalle.

## 3. Arquitectura en capas (el flujo no cambió)

```text
NAVEGADOR
     → front_investigacion/index.php   (ruta → pantalla)
     → cliente_api.php                 (única pieza que habla HTTP hacia afuera)
     ↓  HTTP + JSON  — aquí termina un proceso y empieza otro
     → api_investigacion/index.php     (método + ruta → controlador)
     → Controlador{Recurso}            (valida la forma del body → 422)
     → IServicio{Recurso}              (interfaz — reglas de negocio)
     → IRepositorio{Recurso}           (interfaz — el servicio no sabe qué motor hay)
     → Repositorio{Recurso}MariaDB     (PDO; en el grupo, CALL a procedimientos)
     → MariaDB                         (llaves foráneas y procedimientos)
```

Lo que la v2 agrega es una flecha **de vuelta**: la base ya no solo guarda,
también **dice que no**. Ese «no» sube por el repositorio, que lo traduce, y
sale por el controlador como un `409`.

Principios SOLID, para poder explicarlos en voz alta:

| Principio | Dónde se ve |
|---|---|
| **S** — una sola responsabilidad | El controlador habla HTTP, el servicio aplica reglas, el repositorio habla con la base |
| **O** — abierto a extender | Una tabla nueva es un juego nuevo de archivos; no se toca lo que ya funciona |
| **L** — sustituibles | El repositorio falso de las pruebas reemplaza al de MariaDB sin que el servicio lo note |
| **I** — interfaces pequeñas | Las tablas puente tienen una interfaz sin `actualizar` |
| **D** — depender de abstracciones | El servicio recibe `IRepositorio...` por el constructor, nunca `new Repositorio...MariaDB` |

## 4. Decisiones de diseño clave

### 4.1 `errores_de_integridad.php`: dónde se traduce el «no» del motor

Un archivo con una función, que vive en `repositorios/`:

```php
function traducirErrorDeIntegridad(PDOException $error, string $ficha): never
{
    // El regla propia (SIGNAL) se lee en [0]; los demás, en [1]
    if (($error->errorInfo[0] ?? '') === '45000') { /* mensaje del procedimiento */ }

    $codigo = $error->errorInfo[1] ?? 0;
    if ($codigo === 1062) { /* llave o pareja repetida     */ }
    if ($codigo === 1452) { /* apunta a algo que no existe */ }
    if ($codigo === 1451) { /* otros dependen de esto      */ }

    throw $error;   // no lo reconozco: que se vuelva un 500
}
```

Los números 1062, 1452 y 1451 son de MariaDB. El repositorio es la única
capa que tiene derecho a conocerlos, porque es la única que sabe qué motor
hay detrás. Hacia arriba no sale un número: sale una
`ConflictoDeIntegridadExcepcion` con el problema dicho en español.

> **Detalle que conviene saber explicar:** `getCode()` de PDO devuelve
> `'23000'` para 1452, 1062 y 1451 por igual, así que no sirve para
> distinguirlos. El número que sirve está en `errorInfo[1]`. Y el `SIGNAL`
> se lee al revés: su errno es siempre 1644, así que lo que lo identifica es
> el SQLSTATE `'45000'` en `errorInfo[0]`.

El 1451 casi no se dispara con borrado lógico (es un `UPDATE`), pero se sigue
traduciendo por seguridad.

### 4.2 Tres clases de recurso, tres tratamientos

| Clase | Ejemplos | Acceso a datos | Verbos |
|---|---|---|---|
| **Ficha** | `docente`, `semillero` | SQL con parámetros `?` en el repositorio | `GET` lista, `GET` uno, `POST`, `PUT`, `PATCH`, `DELETE` |
| **Puente** | `grupo_linea`, `ac_linea`... | SQL con parámetros `?` en el repositorio | `GET` lista, `POST`, `DELETE` con dos claves. `PUT`/`PATCH` → `405` |
| **Maestro-detalle** | `grupo_investigacion` + `semillero` | **Procedimientos almacenados** | `GET`, `GET` uno, `POST`, `PUT`, `DELETE`. `PATCH` → `405` |

El `405` sale del enrutador porque la ruta no está escrita: no hay que
programar nada para que `PATCH` falle.

### 4.3 El repositorio del grupo llama procedimientos, no escribe SQL

Las fichas y las puente arman su `SELECT` y su `INSERT`. El repositorio del
grupo **solo llama**:

```php
$sentencia = $this->conexion->prepare('CALL sp_crear_grupo_investigacion(?, ?, ?)');
$sentencia->execute([$nombre, $universidad, json_encode($semilleros)]);

$grupo = $sentencia->fetch(PDO::FETCH_ASSOC);   // primer resultado: el encabezado
$sentencia->nextRowset();
$detalle = $sentencia->fetchAll(PDO::FETCH_ASSOC);   // segundo: los semilleros

$sentencia->closeCursor();   // IMPRESCINDIBLE: si no, la siguiente consulta da el error 2014
```

`[NECESITA ACLARACIÓN: parámetros reales del procedimiento, según las columnas del SQL oficial.]`

**Cómo viaja el detalle.** Los semilleros llegan al procedimiento como **un
solo parámetro JSON**, y el procedimiento los recorre con `JSON_TABLE`. Un
procedimiento no puede recibir una lista, y mandar cinco `CALL` seguidos
rompería el «todo o nada».

**Cómo se garantiza el «todo o nada».** El procedimiento abre la transacción
y la deshace si algo falla:

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE sp_crear_grupo_investigacion(
    IN p_nombre      VARCHAR(100),
    IN p_universidad INT,
    IN p_detalle     LONGTEXT          -- JSON con los semilleros
)
BEGIN
    DECLARE v_id INT;

    -- si CUALQUIER cosa falla: se deshace todo y el error sube igual
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    INSERT INTO grupo_investigacion (nombre, universidad)
    VALUES (p_nombre, p_universidad);
    SET v_id = LAST_INSERT_ID();       -- ANTES de otra sentencia

    INSERT INTO semillero (nombre, grupo)
    SELECT j.nombre, v_id
    FROM JSON_TABLE(p_detalle, '$[*]' COLUMNS (nombre VARCHAR(100) PATH '$.nombre')) AS j;

    COMMIT;

    -- MariaDB no tiene RETURNING: se devuelve lo que quedó guardado
    SELECT * FROM grupo_investigacion WHERE id = v_id;
    SELECT * FROM semillero WHERE grupo = v_id AND activo = 1;
END$$

DELIMITER ;
```

Es un esquema para entender la forma, no el código final: las columnas son
ilustrativas.

### 4.4 Distinguir «no existe» de «no se deja»

Los rechazos de un procedimiento llegan con `SIGNAL`, y no significan lo
mismo:

- «El grupo no existe» → debe ser **404**.
- «Primero retire los semilleros de este grupo» → debe ser **409**.

El ejemplo de clase los separa mirando el **texto** del mensaje, y lo declara
como una fragilidad. Aquí se evita con un dato más fiable: cada caso lleva su
propio SQLSTATE.

| Caso | SQLSTATE del `SIGNAL` | Respuesta |
|---|---|---|
| Regla de negocio rota | `'45000'` | `409` con el texto del mensaje |
| No existe o está retirado | `'45001'` | `404` |

`[NECESITA ACLARACIÓN: confirmar con el profesor que se puede usar '45001'. La guía solo nombra '45000'. Si no, se separa por el texto, como en el ejemplo, y se declara como fragilidad.]`

### 4.5 El controlador del grupo valida dos niveles

El body del grupo trae una lista adentro:

```json
{ "nombre": "Grupo de IA", "universidad": 1,
  "detalle": [ {"nombre": "Semillero de visión"},
               {"nombre": "Semillero de lenguaje"} ] }
```

Validar eso es validar que `detalle` sea una lista **y** que cada renglón
traiga sus campos con el tipo correcto. El error dice **cuál** renglón y por
qué, y se numeran desde 1: el renglón 0 no existe para quien mira la pantalla.

### 4.6 La regla de retiro vive en dos sitios

Con borrado lógico, retirar un grupo es un `UPDATE`, y la llave foránea no se
entera. Por eso hay dos defensas a distinta altura:

1. **El procedimiento** `sp_eliminar_grupo_investigacion` revisa si hay
   semilleros activos y lanza el `SIGNAL`. Protege a la base de cualquier
   programa que le escriba sin pasar por la API.
2. **El servicio** aplica la misma regla en PHP, para dar el mensaje antes.
   `[NECESITA ACLARACIÓN: confirmar con el profesor si se quiere esta doble defensa o solo la del procedimiento.]`

### 4.7 Las tablas puente

- La llave es compuesta: las dos llaves foráneas juntas.
- La ruta lleva las dos claves: `DELETE /api/grupo_linea/{clave1}/{clave2}`.
- Repetir una pareja da el 1062 → `409`.
- Retirar una pareja que no existe → `404`.
- La interfaz del repositorio **no tiene** `actualizar`: una pareja existe o
  no existe.
- En la pantalla no hay botón de editar.

`[NECESITA ACLARACIÓN: ¿llevan activo o se borran físicamente? Cambia el repositorio: UPDATE o DELETE.]`

### 4.8 `activo`: el `ALTER` de la v2

Las tablas del módulo no traen `activo`; solo `rol` y `usuario` lo tienen. Si
no se agregó en la v1, se agrega ahora en `procedimientos_v2.sql`:

```sql
ALTER TABLE docente ADD COLUMN IF NOT EXISTS activo TINYINT(1) NOT NULL DEFAULT 1;
```

`ADD COLUMN IF NOT EXISTS` es de MariaDB y permite correr el script varias
veces. La razón se escribe en `5_data_model.md`.

### 4.9 `ensamblador.php`: una función por recurso, no un `switch`

Una función por recurso (`crearServicioDocente()`, `crearServicioGrupo()`...),
y no una sola con un `switch` ni nombres de clase armados con texto:

- con funciones separadas, PHP verifica los tipos y el error sale al llamarlas;
- con nombres armados en texto, el error sale en producción, cuando alguien
  pida el recurso que nadie probó;
- quien lea el archivo ve el inventario completo sin ejecutar nada.

### 4.10 El front: un bloque por recurso y una excepción

`front_investigacion/index.php` repite un bloque de rutas por recurso. Es
largo, pero cada bloque se lee solo. La prueba de que repetir era correcto: el
bloque del grupo **no se parece a los demás**. No tiene «guardar solo lo que
cambié», y tiene una pantalla que muestra el detalle. Una tabla de
configuración habría tenido que crecer con excepciones hasta volverse ilegible.

### 4.11 Desplegables y llaves foráneas

Los formularios con llave foránea ofrecen un `<select>`. Cuesta una petición
más a la API por cada desplegable, y las vale.

- Muestra el **nombre** y manda la **clave**.
- Solo ofrece registros **activos**, porque el listado del que sale ya filtra.
- Si la llave es opcional, la primera opción es **«(ninguna)»** y se convierte
  a `null` a mano: un `<select>` vacío llega como cadena vacía, y `(int) ''`
  vale `0`.
- Todo texto que viene de la base se imprime con `htmlspecialchars`.
- La respuesta `204` (tabla vacía) no trae JSON: `json_decode('')` da `null`,
  y hay que tratarlo antes de recorrerlo.

**Lo que el desplegable NO hace es validar.** La lista pudo cargarse hace un
minuto y esa ficha pudo borrarse entretanto. Quien defiende la integridad es
la base.

### 4.12 Detalle en un solo envío

El formulario del grupo ofrece **cinco renglones fijos**, con campos
`detalle_nombre[]` que llegan a PHP como arreglos paralelos. El front los
recorre a la vez, descarta los vacíos y los manda como una lista en un solo
`POST`. Es una limitación de la pantalla, no de la API.

### 4.13 Interpretabilidad

Quien entra a la carpeta debe saber qué hay y qué hace cada cosa:

- Un `README.md` por carpeta principal, con una frase por archivo.
- Un comentario de apertura en cada archivo PHP: qué es y qué NO hace.
- Nombres que dicen lo que hacen (`ControladorGrupoInvestigacion`, no `Ctrl1`).
- Los documentos del spec kit dicen lo mismo que el código.

## 5. Docker: los mismos cuatro servicios

```yaml
services:
  mariadb:                 # mariadb:11 + db/init.sql + procedimientos_v2.sql · healthcheck
  api-investigacion:       # build ./api_investigacion · DB_DSN → mariadb:3306
  front-investigacion:     # build ./front_investigacion · URL_API → http://api-investigacion:...
                           #   sin credenciales de la base, sin depends_on de mariadb
  phpmyadmin:
volumes:
  mariadbdata:
```

`[NECESITA ACLARACIÓN: puertos de esta versión. Deben ser distintos a los de la v1 para poder tener las dos encendidas y compararlas.]`

Las variables de entorno son las mismas de la v1: `DB_HOST`, `DB_PORT`,
`DB_DATABASE`, `DB_USERNAME` y `DB_PASSWORD`.

## 6. Convenciones

Las de la constitución: todo en español, comentario de apertura por archivo,
clases PascalCase (prefijo `I` en interfaces), métodos y variables camelCase,
un archivo por clase y `declare(strict_types=1)` en todo archivo PHP.

## 7. Chequeo de constitución

> **La compuerta 2** del método: antes de pasar a `8_tasks.md` se revisa la
> constitución **artículo por artículo**.

> [!WARNING]
> Todavía no he visto `1_constitution.md`. Los nombres de los artículos salen
> del ejemplo de clase. Hay que contrastarlos con el archivo real antes de dar
> esta tabla por buena.

| Artículo | Cómo lo cumple esta versión |
|---|---|
| **1** — Propósito didáctico | Todo en español y comentado; explícito y repetitivo antes que compacto (§4.10) |
| **1.1** — Una versión incluye su front | Cada recurso nuevo entrega su pantalla, incluida la del grupo con su detalle |
| **2** — PHP puro | Sin framework, sin Composer. Bootstrap es un archivo copiado |
| **3** — Tres capas estrictas | El front no recibe credenciales ni depende de la base |
| **4** — Un solo comando | `docker compose up -d --build` deja los cuatro servicios funcionando |
| **5** — Independencia del motor | El dialecto (`CALL`, 1062, 1452, 1451, `'45000'`) vive solo en las clases `*MariaDB` y en `errores_de_integridad.php` |
| **6** — Persistencia | Volúmenes; `down -v` devuelve la base a su estado original |
| **7** — Recarga natural | PHP reinterpreta cada petición |
| **8** — Convenciones fijas | Sobre de respuesta y catálogo de errores, con el `409` agregado |
| **9** — Seguridad académica | Credenciales en `.env`; todo lo que el front pinta pasa por `htmlspecialchars` |
| **10** — API específica | Una ruta escrita por recurso; la prueba: las puente y el grupo no admiten `PATCH` y ninguna ruta genérica podría decirlo |
| **11** —
