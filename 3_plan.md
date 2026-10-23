# Plan técnico — Versión 2

> CÓMO construir lo de [2_spec.md](2_spec.md). Porqués en [4_research.md](4_research.md).

## 1. Stack

El mismo de la v1, sin dependencias nuevas.

| Pieza | Elección | Cambio vs v1 |
|---|---|---|
| Lenguaje | PHP 8.3, `declare(strict_types=1)` | nada |
| Acceso a datos | PDO con `?` | **+ `CALL`** solo para el grupo y su detalle |
| Front | PHP + Bootstrap local + `marca.css` | una sección por recurso nuevo |
| Motor | MariaDB 11 | nada |

## 2. Estructura

```text
(raíz)
├── docker-compose.yml        # mariadb 13330 · api 8111 · front 8110 · phpmyadmin 8105
├── db/
│   ├── init.sql              # v1 (cabecera numera los cambios al esquema, Art. 5)
│   └── procedimientos_v2.sql # tablas v2 + los 5 sp_ del grupo
├── api_investigacion/
│   ├── README.md
│   ├── index.php             # un bloque de rutas por recurso
│   ├── modelos/              # + Docente, Semillero, GrupoInvestigacion, 7 puentes
│   ├── controladores/
│   ├── servicios/            # IServicio*, Servicio*, ensamblador.php
│   ├── repositorios/         # IRepositorio*, Repositorio*MariaDB, errores_de_integridad.php  ← NUEVO
│   ├── excepciones/          # NoEncontradoExcepcion, ConflictoDeIntegridadExcepcion          ← NUEVA
│   └── pruebas/prueba_capas.php
├── front_php/
│   ├── README.md
│   ├── index.php
│   ├── cliente_api.php
│   ├── vistas/               # {recurso}_lista, {recurso}_formulario, grupos_detalle  ← NUEVA
│   └── publico/              # bootstrap, marca.css, estilos.css
└── pruebas_humo/humo_front.py
```

## 3. Flujo (no cambia)

```text
NAVEGADOR → front_php/index.php → cliente_api.php
   ↓ HTTP + JSON (aquí termina un proceso y empieza otro)
api_investigacion/index.php → Controlador → IServicio → IRepositorio → Repositorio*MariaDB → MariaDB
```

Lo nuevo es la flecha **de vuelta**: la base ahora también **dice que no**. Ese «no» sube por el repositorio, que lo traduce, y sale como `409`.

SOLID, para explicarlo en voz alta:

| Principio | Dónde se ve |
|---|---|
| **S** | El controlador habla HTTP, el servicio aplica reglas, el repositorio habla con la base |
| **O** | Una tabla nueva es un juego nuevo de archivos; no se toca lo que ya funciona |
| **L** | El repositorio falso de las pruebas reemplaza al de MariaDB |
| **I** | La interfaz de las puente no tiene `actualizar` |
| **D** | El servicio recibe `IRepositorio...` por el constructor, nunca hace `new Repositorio...MariaDB` |

## 4. Decisiones de diseño

**4.1 Tres clases de recurso**

| Clase | Ejemplos | Acceso | Verbos |
|---|---|---|---|
| Ficha | `docente`, `semillero` | SQL con `?` | los seis de la v1 |
| Puente | `grupo_linea`, ... | SQL con `?` | `GET`, `POST`, `DELETE` con dos claves; `PUT`/`PATCH` → `405` |
| Maestro-detalle | `grupo_investigacion` | **`CALL sp_*`** | todos menos `PATCH` (→ `405`) |

El `405` sale del enrutador porque la ruta no está escrita (Art. 10).

**4.2 `errores_de_integridad.php`.** Una función que traduce `PDOException` a `ConflictoDeIntegridadExcepcion`:

```php
if (($error->errorInfo[0] ?? '') === '45001') { /* no existe → NoEncontradoExcepcion */ }
if (($error->errorInfo[0] ?? '') === '45000') { /* regla propia → mensaje del SIGNAL */ }
$codigo = $error->errorInfo[1] ?? 0;          // 1062 repetido · 1452 padre no existe · 1451 hay dependientes
```

`getCode()` de PDO da `'23000'` para los tres primeros, así que no sirve. El `SIGNAL` se lee **al revés**: su errno siempre es 1644 y lo identifica el SQLSTATE en `errorInfo[0]`.

**4.3 Repositorio del grupo.** Solo llama:

```php
$s = $this->conexion->prepare('CALL sp_crear_grupo_investigacion(?, ?, ?, ?, ?, ?)');
$s->execute([$gruplac, $categoria, $convocatoria, $ambito, $universidadId, json_encode($semilleros)]);
$grupo   = $s->fetch(PDO::FETCH_ASSOC);   // 1.er resultado: el grupo
$s->nextRowset();
$detalle = $s->fetchAll(PDO::FETCH_ASSOC); // 2.º resultado: sus semilleros
$s->closeCursor();                         // IMPRESCINDIBLE: si no, error 2014
```

El detalle viaja como **un parámetro JSON**; el procedimiento lo recorre con `JSON_TABLE` y abre la transacción (`EXIT HANDLER` + `ROLLBACK` + `RESIGNAL`). El código está en `db/procedimientos_v2.sql`.

**4.4 Reglas de retiro.** Con borrado lógico la llave foránea no se entera. Por eso `sp_eliminar_grupo_investigacion` revisa los semilleros activos y lanza el `SIGNAL`. El servicio no repite la regla (una sola fuente de verdad).

**4.5 Puente.** Llave compuesta, ruta con las dos claves, sin `activo`: retirar es `DELETE FROM`. Repetir pareja → `1062` → `409`.

**4.6 Front.**

- Desplegables: nombre visible, clave enviada, solo activos.
- «(ninguna)» llega como cadena vacía y se convierte a `null` a mano; `(int) ''` daría `0`.
- Un `204` no trae JSON: `json_decode('')` da `null` y el `foreach` revienta si no se trata antes.
- `htmlspecialchars` en todo lo que viene de la base.
- Detalle en cinco renglones fijos `detalle_nombre[]`; el front los recorre a la vez, descarta vacíos y manda una lista en un solo `POST`.
- Estilos con `var(--umv-...)`; errores en rojo alerta.

**4.7 Interpretabilidad.** Cabecera en cada archivo PHP (qué es y qué NO hace), un `README.md` por carpeta, nombres que dicen lo que hacen.

## 5. Docker

```yaml
mariadb:            # mariadb:11 · 13330 · MARIADB_DATABASE: investigacion · ./db → initdb.d
api-investigacion:  # build ./api_investigacion · 8111
front-php:          # build ./front_php · 8110 · URL_API → http://api-investigacion
                    #   sin credenciales, sin depends_on de mariadb, sin pdo_mysql
phpmyadmin:         # 8105
```

Las credenciales siguen en el compose (excepción declarada del Art. 9). `MARIADB_DATABASE` debe ser **`investigacion`**, igual que en `init.sql`.

## 6. Chequeo de constitución (Compuerta 2)

| Artículo | Cómo lo cumple la v2 |
|---|---|
| **1** Didáctico | Cabeceras, README por carpeta, comentarios que explican decisiones |
| **1.1** Incluye su front | Cada recurso nuevo entrega su pantalla; con la API apagada la pantalla sigue en pie |
| **2** PHP puro | Sin framework ni Composer; Bootstrap descargado |
| **3** Tres capas | Controlador sin SQL, servicio sin HTTP, repositorio sin reglas; solo `ensamblador.php` hace `new` |
| **4** Un comando | `docker compose up -d --build` |
| **5** Esquema dado | Cambios al esquema declarados y numerados en la cabecera del script; las columnas propuestas se contrastan con el oficial |
| **6** Borrado lógico | `activo = 0`; la pantalla dice «retirar» |
| **7** Persistencia | Volumen; `down -v` vuelve a cero |
| **8** Convenciones | Todo en español, `snake_case` en JSON y columnas |
| **9** Seguridad | `?` siempre, lista blanca de columnas, `htmlspecialchars`; contraseña en el compose (excepción declarada) |
| **10** API específica | Una ruta por recurso; la prueba: las puente y el grupo no admiten `PATCH` |
| **11** Gemelo | El contrato no cambia de forma respecto al gemelo en Python; si cambia, se anota |
| **12** Identidad visual | Estilos con `marca.css`; errores en rojo alerta |

**Complejidad justificada:** `SQLSTATE '45001'` para «no existe» no está en la guía. Se prefiere a leer el texto del mensaje (frágil). Queda aislado en `errores_de_integridad.php`.

## 7. Pruebas

- **Capas** (`pruebas/prueba_capas.php`): servicios con repositorio falso, sin MariaDB.
- **HTTP**, incluidas las que **deben fallar**: padre inexistente, pareja repetida, grupo con semilleros, `PATCH` del grupo, detalle con renglón inválido.
- **Recorrido desde la pantalla** (`humo_front.py`).
- **Regresión** completa de la v1.

## 8. Riesgos

| Riesgo | Evitarlo |
|---|---|
| Script partido en el primer `;` | `DELIMITER $$` y volver a `;` |
| Error 2014 | `closeCursor()` tras cada `CALL` |
| Todo cae en 500 | Leer `errorInfo[0]` y `[1]`, no `getCode()` |
| FK opcional guarda `0` | Convertir `''` a `null` |
| Pantalla revienta con tabla vacía | Tratar `204` antes del `foreach` |
| Lista muestra retirados | `WHERE activo = 1` en listados y desplegables |
| Grupo a medias | `EXIT HANDLER` con `ROLLBACK` y `RESIGNAL` |
| Los `.sql` corren en otra base | `MARIADB_DATABASE: investigacion` |
