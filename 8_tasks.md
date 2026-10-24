# Tareas — Versión 2: llaves foráneas, integridad y maestro-detalle

> **Versión 2** · El orden de construcción, partiendo del estado que dejó la
> v1. Cada fase termina en algo **verificable**. Requisitos:
> [2_spec.md](2_spec.md) · técnica: [3_plan.md](3_plan.md) · contratos:
> [6_contratos.md](6_contratos.md) · validación final:
> [7_quickstart.md](7_quickstart.md).
>
> **El orden no es casual.** Se construye de abajo hacia arriba en el mapa de
> dependencias ([5_data_model.md](5_data_model.md) §2): primero lo que no
> depende de nadie, después lo que depende de alguien, y de últimas el
> maestro-detalle. Al revés no se puede probar nada: un grupo necesita una
> universidad que exista.

---

## Fase 0 — Partir de la v1 y arreglar la base
- [ ] Partir del código de la v1 funcionando: las seis tablas de catálogo con
      API y pantalla.
- [ ] `docker-compose.yml`: poner `MARIADB_DATABASE: investigacion` (hoy dice
      `investigacion_php_local`, y los `.sql` correrían en otra base).
- [ ] Renombrar `db/procedimientos.sql` a **`db/procedimientos_v2.sql`**. Docker
      ejecuta los archivos en orden alfabético y `init.sql` debe ir primero.
- [ ] Reescribir `procedimientos_v2.sql` según [5_data_model.md](5_data_model.md):
      **quitar** los triggers, `total_grupos` y `sp_recontar_grupos` (la v2 no
      los pide), **crear** las tablas que faltan y los cinco `sp_` del grupo
      con transacción y `SIGNAL`.
- [ ] Numerar los cambios al esquema (C1 a C3) en la cabecera del script
      (Art. 5).
- [ ] Agregar las semillas de la v2 (un grupo con dos semilleros y una pareja
      `grupo_linea`).

**Verificar:** `docker compose down -v && docker compose up -d --build`, y en
phpMyAdmin (<http://localhost:8105>) existen las 13 tablas y los 5
procedimientos. Correr el script **dos veces** no da error. **Si esto no
pasa, no siga.**

## Fase 1 — La pieza que hace posible todo lo demás: traducir el «no» de la base
- [ ] `excepciones/ConflictoDeIntegridadExcepcion.php`.
- [ ] `repositorios/errores_de_integridad.php` con
      `traducirErrorDeIntegridad()`: reconoce **1062** (repetido), **1452**
      (apunta a algo que no existe), **1451** (otros dependen) en
      `errorInfo[1]`, y los `SIGNAL` **`'45000'`** (→ conflicto, con su texto)
      y **`'45001'`** (→ `NoEncontradoExcepcion`) en `errorInfo[0]`.
      Deja pasar lo que no reconoce ([3_plan.md](3_plan.md) §4.2).

> Se hace PRIMERO, aunque todavía no haya quién la use. Si se deja para el
> final, cada repositorio inventa su propia forma de reportar el conflicto.

**Verificar:** `php -l` pasa. La prueba de verdad llega en las fases 2 y 4.

## Fase 2 — La ficha que no depende de nadie: `docente`
- [ ] Modelo `Docente.php` (propiedades privadas, getters, setters menos la
      cédula, `toArray()`).
- [ ] Interfaces `IRepositorioDocente` e `IServicioDocente`.
- [ ] `RepositorioDocenteMariaDB`: SQL con `?`, `activo = 1` en listado y
      consulta, `UPDATE ... SET activo = 0` al retirar, y el `try/catch` que
      llama a `traducirErrorDeIntegridad` en las escrituras.
- [ ] `ServicioDocente` y `ControladorDocente` (`POST`/`PUT` exigen todo,
      `PATCH` solo lo enviado; cédula no entera → `400`).
- [ ] Registrarlos en `ensamblador.php` (una función por recurso).

**Verificar:** `GET /api/docente` devuelve los 2, y **crear un docente con una
cédula que ya existe responde 409** — no 500
([7_quickstart.md](7_quickstart.md) §3.2).

## Fase 3 — El maestro-detalle: `grupo_investigacion` con sus semilleros
- [ ] Modelo `GrupoInvestigacion.php` (el encabezado) y `Semillero.php` (un
      renglón). **Sin `nombre`** en el grupo: se identifica por su `gruplac`.
- [ ] `IRepositorioGrupoInvestigacion`: **no son los métodos de siempre**.
      Lleva `obtenerTodos`, `obtenerPorId`, `crear`, `reemplazar` y `retirar`;
      **no lleva `actualizarParcial`** ([4_research.md](4_research.md) D8).
- [ ] `RepositorioGrupoInvestigacionMariaDB` solo con `CALL ... (?, ...)`, sin
      SQL escrito a mano:
      - el detalle viaja como **un parámetro JSON** (`json_encode`);
      - se leen **dos resultados** con `nextRowset()`;
      - **`closeCursor()` después de cada `CALL`**;
      - el `try/catch` con `traducirErrorDeIntegridad`.
- [ ] `ServicioGrupoInvestigacion`: delgado. La regla de retirar vive en el
      procedimiento, no se repite aquí (D7).
- [ ] `ControladorGrupoInvestigacion` con validación de **dos niveles**: los
      campos del grupo, y cada renglón de `detalle` por separado, diciendo
      **cuál** falla (numerado desde 1). `"detalle": []` es válido.
- [ ] Registrarlo en `ensamblador.php`.

**Verificar (criterios 4, 5, 10 y 14):** el ciclo de
[7_quickstart.md](7_quickstart.md) §3.3 y §3.4 — crear con 3 semilleros en una
petición, el fallo con universidad `999` que **no deja nada guardado**, el
renglón inválido con `422`, `DELETE` con semilleros → `409`, grupo inexistente
→ `404`.

## Fase 4 — El enrutador de la API (`index.php`)
Hoy el bloque del grupo **no funciona** y hay que rehacerlo:

- [ ] Quitar el bloque suelto que está **antes** del `switch`: usa `$pdo` e
      `$id`, que no existen.
- [ ] Agregar los `require_once` de los controladores nuevos.
- [ ] Quitar la línea `Content-Type` repetida.
- [ ] Cambiar `"version" => "v1"` por `"v2"` y listar los 16 recursos en `GET /`.
- [ ] Agregar un `case` por recurso, igual que los de la v1:

```php
case 'grupo_investigacion':
    $controlador = crearControladorGrupoInvestigacion();   // viene de ensamblador.php
    $controlador->procesarPeticion($metodo, $idStr);
    break;
```

- [ ] Las puente llevan **dos claves** en la ruta (`$partes[2]` y `$partes[3]`);
      pasarlas al controlador:

```php
case 'grupo_linea':
    $controlador = crearControladorGrupoLinea();
    $controlador->procesarPeticion($metodo, $partes[2] ?? null, $partes[3] ?? null);
    break;
```

- [ ] En cada controlador, un `else` final que responde **405**: el 405 de
      `PATCH` del grupo y de `PUT` de una puente sale de ahí, sin escribir el
      número en otro sitio.

**Verificar:** `GET /` dice `"version": "v2"`; `PATCH /api/grupo_investigacion/1`
responde `405`; una ruta inventada responde `404`.

## Fase 5 — `semillero` (solo lectura)
- [ ] Modelo, interfaces, `RepositorioSemilleroMariaDB` con **solo** `obtenerTodos`
      y `obtenerPorId`, `ServicioSemillero` y `ControladorSemillero`.
- [ ] `POST`, `PUT`, `PATCH` y `DELETE` responden **405**.

**Verificar:** `GET /api/semillero` lista los 2 con su `grupo_id`; `POST` da
`405` ([7_quickstart.md](7_quickstart.md) §3.6).

## Fase 6 — Las siete tablas puente
- [ ] Un modelo por tabla puente: `ParticipaGrupo`, `ParticipaSemillero`,
      `GrupoLinea`, `SemilleroLinea`, `AcLinea`, `OdsLinea`, `AaLinea`.
- [ ] La interfaz de repositorio **sin `actualizar`** (segregación de
      interfaces): `obtenerTodos`, `asignar`, `retirar`.
- [ ] Repositorios con `INSERT` y `DELETE FROM` (las puente no llevan `activo`)
      más el `try/catch` de integridad.
- [ ] Servicios y controladores: `POST` valida las dos claves (enteros > 0),
      `DELETE` lleva las dos en la ruta; retirar una pareja que no existe →
      `404`.
- [ ] Hacer **una** completa (`grupo_linea`), probarla y copiar el patrón a las
      otras seis.

**Verificar (criterios 3, 9 y 13):** el ciclo de
[7_quickstart.md](7_quickstart.md) §3.5 — asignar, repetir → `409`, línea
inexistente → `409`, retirar, retirar otra vez → `404`, `PUT` → `405`, y una
tabla vacía → `204`.

## Fase 7 — El front: las secciones nuevas
- [ ] `cliente_api.php`: una función por operación, con el nombre del recurso
      adentro (`listar_docentes()`, no `listar($tabla)`; Art. 10). Tratar el
      **`204`** antes de recorrer: `json_decode('')` da `null` y el `foreach`
      revienta. **El grupo no lleva `actualizar_parcial_grupo`.**
- [ ] `index.php`: un bloque de rutas por recurso, y el de grupos distinto
      ([6_contratos.md](6_contratos.md) §10.1). Los ayudantes de catálogos para
      los desplegables.
- [ ] Vistas de `docente`: `docentes_lista.php` y `docentes_formulario.php`.
- [ ] Vistas del grupo: `grupos_lista.php`, `grupos_formulario.php` (**un solo
      botón**, cinco renglones `detalle_nombre[]`; al **editar**, todos los
      semilleros actuales más renglones vacíos) y **`grupos_detalle.php`**, la
      pantalla del maestro con sus semilleros.
- [ ] `semilleros_lista.php` (solo lectura) y las vistas de las puente (lista
      con **nombres** y formulario con **dos desplegables**, sin botón de
      editar).
- [ ] Desplegables: muestran el nombre, mandan la clave, solo activos, todo con
      `htmlspecialchars`.
- [ ] Menú de `plantilla.php` e `inicio.php` con todas las secciones.
- [ ] Botones y avisos con «**Retirar**» (nunca «Borrar»); errores en **rojo
      alerta** y estilos con `var(--umv-...)` de `marca.css` (Art. 12).
- [ ] La pantalla **no dice** «PUT», «PATCH», «409», «FK», `/api/` ni nombres de
      columna.

**Verificar (criterios 6, 11, 16 y 18):** el recorrido a mano de
[7_quickstart.md](7_quickstart.md) §4.2 — incluido intentar retirar el grupo con
semilleros y apagar solo la API.

## Fase 8 — Las pruebas
- [ ] Ampliar `pruebas/prueba_capas.php` con los servicios nuevos y un
      repositorio falso en memoria: retirar un grupo con semilleros es
      rechazado, una pareja repetida da conflicto, un grupo inexistente da
      «no encontrado» (criterio 15).
- [ ] `pruebas_humo/humo_front.py`: las pantallas, los estáticos (`marca.css`,
      Bootstrap), la jerga, el ciclo de una ficha, el grupo con sus semilleros,
      **los rechazos de integridad** y la prueba de apagar la API.
- [ ] Una colección de pruebas HTTP **con las que deben fallar**: padre
      inexistente, pareja repetida, grupo con semilleros, `PATCH` del grupo,
      renglón inválido.

**Verificar:** `docker compose exec api-investigacion php pruebas/prueba_capas.php`
da `[OK]` y `python pruebas_humo/humo_front.py` termina en verde.

## Fase 9 — Que se entienda la carpeta (interpretabilidad)
- [ ] Una **cabecera** en cada archivo PHP: qué es, por qué está ahí y qué NO
      hace (Art. 1).
- [ ] Un **`README.md`** por carpeta (`controladores/`, `servicios/`,
      `repositorios/`, `modelos/`, `excepciones/`, `vistas/`), con una frase por
      archivo.
- [ ] Elegir **un archivo** (por ejemplo `errores_de_integridad.php` o
      `RepositorioGrupoInvestigacionMariaDB.php`) y practicar explicarlo en voz
      alta: la sustentación individual lo pide.

## Fase 10 — La regresión y los documentos
- [ ] **Regresión de la v1:** correr todos sus criterios. La v2 incluye la v1.
- [ ] Contrastar `docente`, `semillero` y las puente con
      `db_scripts/mysql/investigacion.sql` (Art. 5). Si el oficial difiere,
      manda el oficial: ajustar el script y los documentos 2 a 6.
- [ ] Corregir lo que quedó desfasado en la v1 (contraseña en el compose y no
      en `.env`, carpeta `front_php`, puertos reales, título del
      `7_quickstart.md`).
- [ ] Escribir `GUIA_IA2.md`: **los prompts que de verdad usó** y, sobre todo,
      **lo que tuvo que corregirle a la IA**.
- [ ] Revisar que **documentación y código dicen lo mismo**: cada ruta, código
      y nombre documentado existe.

## Fase 11 — Cierre de la versión
- [ ] Correr el smoke test completo de [7_quickstart.md](7_quickstart.md): los
      **18** criterios de [2_spec.md](2_spec.md) §5.
- [ ] Revisar y firmar [9_checklist.md](9_checklist.md) (la firma una persona).
- [ ] Commit, Pull Request y tag `v2`.

**La v2 está TERMINADA.** Solo ahora se escribe la spec de la v3.
