# Guía rápida — Versión 2

> **Versión 2** · Validación rápida de la v2 ya construida. Si aún no hay
> nada, empiece por [8_tasks.md](8_tasks.md).

---

## 1. Arrancar TODO (un solo comando)

```powershell
# desde la raíz del proyecto (terminal integrada de VS Code):
docker compose up -d --build
docker compose ps
```

| | Dónde | Qué es |
|---|---|---|
| **La pantalla** | <http://localhost:8110> | Por donde se usa el sistema |
| La API | <http://localhost:8111> | JSON; el front habla con ella |
| phpMyAdmin | <http://localhost:8105> | Para mirar la base por dentro |
| MariaDB | `localhost:13330` | Usuario y clave en `docker-compose.yml` (excepción declarada del Art. 9) |

**Empiece por el 8110**, que es como lo ve alguien que no programó esto.

> **Los puertos son distintos a los del gemelo en Python.** Es a propósito:
> así se pueden tener los dos encendidos y compararlos.

## 2. Preparar PowerShell

En PowerShell, `curl` no sirve bien para JSON. Usamos `Invoke-RestMethod`
con el mismo ayudante de la v1 (solo cambia el puerto):

```powershell
$API = "http://localhost:8111"

function pedir($metodo, $ruta, $cuerpo) {
    try {
        if ($null -eq $cuerpo) {
            Invoke-RestMethod -Method $metodo -Uri ($API + $ruta) -ContentType "application/json"
        }
        else {
            Invoke-RestMethod -Method $metodo -Uri ($API + $ruta) -ContentType "application/json" -Body $cuerpo
        }
    }
    catch {
        $r = $_.Exception.Response
        if ($null -ne $r) {
            $lector = New-Object IO.StreamReader($r.GetResponseStream())
            "HTTP " + [int]$r.StatusCode + " -> " + $lector.ReadToEnd()
        }
        else { $_.Exception.Message }
    }
}
```

Un `200` imprime la respuesta; un `204` **no imprime nada**; un error imprime
`HTTP <código> -> <cuerpo>`.

## 3. Smoke test de la API

### 3.1 Diagnóstico y regresión de la v1 (criterios 1 y 2)

```powershell
# Deben aparecer los 16 recursos y "version": "v2"
pedir GET /

# La v1 sigue igual
(pedir GET /api/area_conocimiento).total       # 2
(pedir GET /api/universidad).total             # 2
(pedir GET /api/linea_investigacion).total     # 2

# PUT incompleto → 422; PATCH con un campo → 200 (el contraste de la v1)
pedir PUT   /api/area_conocimiento/1 '{"area": "Ingeniería de Sistemas"}'
pedir PATCH /api/area_conocimiento/1 '{"disciplina": "Ingeniería de software"}'
```

> Los totales son los de las semillas de `init.sql` (2 por tabla). Si cargó
> el dataset oficial (218, 17, 21 y 6), use esos números.

### 3.2 Una ficha nueva: `docente` (criterio 3)

```powershell
(pedir GET /api/docente).total                 # 2
pedir POST   /api/docente '{"cedula": 1030456, "nombres": "Marta", "apellidos": "Gil Ríos"}'   # 200
pedir PATCH  /api/docente/1030456 '{"apellidos": "Gil Restrepo"}'                              # 200
pedir DELETE /api/docente/1030456                                                              # 200
pedir DELETE /api/docente/1030456                                                              # 404, ya está retirado

pedir POST /api/docente '{"cedula": 1017245, "nombres": "Repetida", "apellidos": "X"}'         # 409, la cédula ya existe
pedir GET  /api/docente/abc                                                                    # 400, no 404
```

### 3.3 EL MAESTRO-DETALLE: el grupo con sus semilleros (criterios 4 y 5)

```powershell
# El grupo 1 de las semillas trae sus 2 semilleros adentro
pedir GET /api/grupo_investigacion/1

# Crear un grupo CON 3 semilleros, en UNA petición.
# Fíjese: el body NO lleva id ni grupo_id, y la respuesta SÍ trae los ids.
$g = pedir POST /api/grupo_investigacion '{"gruplac": "COL0002", "categoria": "B", "convocatoria": "Convocatoria 2025", "ambito": "Regional", "universidad_id": 1, "detalle": [{"nombre": "Robótica"}, {"nombre": "Datos"}, {"nombre": "Redes"}]}'
$g.grupo                                       # el grupo COMPLETO, con sus 3 semilleros
$id = $g.grupo.id
```

**Todo o nada.** Estas dos peticiones deben fallar y **no dejar nada guardado**:

```powershell
# 409: la universidad 999 no existe → no queda ni el grupo ni sus semilleros
pedir POST /api/grupo_investigacion '{"gruplac": "COL0003", "categoria": "C", "convocatoria": "X", "ambito": "Y", "universidad_id": 999, "detalle": [{"nombre": "Fantasma"}]}'

# 422: el renglón 2 no tiene nombre → el error dice CUÁL renglón
pedir POST /api/grupo_investigacion '{"gruplac": "COL0004", "categoria": "C", "convocatoria": "X", "ambito": "Y", "universidad_id": 1, "detalle": [{"nombre": "Bueno"}, {"nombre": ""}]}'

(pedir GET /api/grupo_investigacion).total     # debe ser 2 (el 1 y el recién creado), no 3 ni 4
```

> Para ver la transacción con sus propios ojos: abra phpMyAdmin y compruebe
> que en `semillero` **no** hay una fila llamada «Fantasma».

**PUT reemplaza el detalle entero** y **PATCH no existe** (criterios 5 y 14):

```powershell
# El grupo queda con 1 solo semillero; los otros 2 quedan retirados
pedir PUT /api/grupo_investigacion/$id '{"gruplac": "COL0002", "categoria": "B", "convocatoria": "Convocatoria 2025", "ambito": "Regional", "universidad_id": 1, "detalle": [{"nombre": "Robótica"}]}'

pedir PATCH /api/grupo_investigacion/$id '{"categoria": "A"}'    # 405
```

### 3.4 Los rechazos de integridad (criterios 8, 9 y 10)

```powershell
# 10. Retirar un grupo CON semilleros → 409 con el texto del procedimiento
pedir DELETE /api/grupo_investigacion/1
#   → "Primero retire los semilleros de este grupo."

# Un grupo sin semilleros SÍ se retira, y la segunda vez da 404 (SIGNAL 45001)
$v = pedir POST /api/grupo_investigacion '{"gruplac": "COL0009", "categoria": "C", "convocatoria": "X", "ambito": "Y", "universidad_id": 1, "detalle": []}'
pedir DELETE /api/grupo_investigacion/$($v.grupo.id)             # 200
pedir DELETE /api/grupo_investigacion/$($v.grupo.id)             # 404
pedir GET    /api/grupo_investigacion/9999                       # 404, no 409 ni 500
```

**NINGUNO debe responder 500.** Si alguno lo hace, `errores_de_integridad.php`
no está atrapando ese caso. Recuerde: el `SIGNAL` se lee en `errorInfo[0]`
(`'45000'` o `'45001'`), y los demás en `errorInfo[1]`.

### 3.5 Las tablas puente (criterios 3 y 9)

```powershell
(pedir GET /api/grupo_linea).total                                # 1, la pareja (1, 1) de las semillas
pedir POST   /api/grupo_linea '{"grupo_id": 1, "linea_id": 2}'    # 200, asigna
pedir POST   /api/grupo_linea '{"grupo_id": 1, "linea_id": 2}'    # 409, la pareja ya existe (1062)
pedir POST   /api/grupo_linea '{"grupo_id": 1, "linea_id": 999}'  # 409, la línea no existe (1452)
pedir DELETE /api/grupo_linea/1/2                                 # 200, retira
pedir DELETE /api/grupo_linea/1/2                                 # 404, ya no está
pedir PUT    /api/grupo_linea/1/1 '{"linea_id": 2}'               # 405, una pareja no se edita
pedir GET    /api/ods_linea                                       # 204: tabla vacía, no imprime nada (criterio 13)
```

Repita el ciclo (asignar, repetir, retirar) con las otras seis puente. Cada
una tiene su propia pareja de claves; están en [6_contratos.md](6_contratos.md) §4.

### 3.6 Semillero es solo lectura

```powershell
pedir GET  /api/semillero                       # 200, con grupo_id
pedir POST /api/semillero '{"nombre": "Suelto"}'   # 405: un semillero nace dentro de su grupo
```

### 3.7 Prueba de capas (criterio 15, sin MariaDB)

Los servicios instanciados con repositorios **falsos** en memoria:

```powershell
docker compose exec api-investigacion php pruebas/prueba_capas.php
```

Deben salir líneas `[OK]`, incluidas las del retiro de un grupo con semilleros
y las del `409` de una pareja repetida.

## 4. La pantalla (criterios 16 a 18)

### 4.1 Automático

```powershell
python pruebas_humo/humo_front.py
```

Recorre las pantallas, que los estilos lleguen de verdad (`marca.css`,
Bootstrap), el menú, que no haya jerga, el ciclo de una ficha, el grupo con
sus semilleros, **los rechazos de integridad explicados en español** y la
prueba de apagar la API.

### 4.2 A mano, que es lo que un guion no ve

Abra <http://localhost:8110> y haga esto **leyendo lo que dice la pantalla**.
Lo que se juzga aquí no es si funciona —eso ya lo dijo el guion— sino si se
entiende:

1. **El menú.** Todas las secciones, y cada una con su dirección propia.
2. **Una ficha sencilla.** Cree un docente, edítelo con los dos botones y
   retírelo. Es lo mismo que en la v1: si algo se siente distinto, es un error.
   Fíjese en que el botón dice **«Retirar»**, nunca «Borrar».
3. **El grupo con su detalle.** Cree un grupo. En el formulario hay
   **«Universidad» como lista con nombres** (no una casilla para teclear `1`)
   y cinco renglones de semillero. Llene tres y guarde: va a la pantalla del
   grupo, con sus tres semilleros ahí mismo.
4. **Editar no pierde semilleros.** Abra ese grupo con «Editar»: debe mostrar
   los tres semilleros actuales más renglones vacíos. Guarde sin tocar nada y
   compruebe que siguen los tres.
5. **La integridad, que es el corazón de esta versión.** Intente retirar el
   grupo que tiene semilleros. La pantalla debe decir, en español, «primero
   retire los semilleros de este grupo» — sin ningún número raro ni la
   palabra «error interno».
6. **Las puente.** Entre a «Líneas por grupo»: las parejas se leen con
   **nombres**, no con claves. Asigne una, intente asignar la misma otra vez
   (debe explicar que ya existe) y retírela. **No hay botón de editar.**
7. **La lista vacía.** Entre a una puente sin parejas: dice «todavía no hay
   datos» y ofrece agregar. No se rompe.
8. **Si un campo falla, no se pierde lo escrito.** Deje vacío el nombre de un
   semillero que sí tiene otros datos: la pantalla muestra el motivo y
   conserva todo lo demás.
9. **Los colores.** Los avisos de error van en **rojo alerta**, no en ocre; el
   verde es el de la marca (Art. 12).
10. **La prueba de los dos procesos.** Con la base de datos encendida, apague
    solo la API:

    ```powershell
    docker compose stop api-investigacion
    ```

    Refresque cualquier pantalla. Sigue en pie, con su menú y con un aviso de
    que el servicio no está disponible — **y sin una sola fila**. Si
    aparecieran datos, sería porque el front llegó a la base por su cuenta.

    ```powershell
    docker compose start api-investigacion
    ```

## 5. Volver al punto de partida

Las pruebas de arriba crean y retiran registros. Para dejar la base como
recién nacida:

```powershell
docker compose down -v && docker compose up -d --build
```

`down -v` borra el volumen, así que `init.sql` y `procedimientos_v2.sql` vuelven
a ejecutarse (en ese orden).

Para comprobar el borrado lógico, mire la fila **en la base**:

```powershell
docker compose exec mariadb mariadb -uinvestigacion -p investigacion -e "SELECT id, gruplac, activo FROM grupo_investigacion;"
```

> Le pedirá la clave: está en `docker-compose.yml`. Un grupo retirado debe
> aparecer con `activo = 0`, no desaparecer.

## 6. Si algo falla

| Síntoma | Causa probable |
|---|---|
| Un rechazo de integridad responde **500** en vez de 409 | `errores_de_integridad.php` no reconoce ese código. El número útil está en `$e->errorInfo[1]`, **no** en `$e->getCode()` (da `'23000'` para todos). El `SIGNAL` se lee en `errorInfo[0]` ([3_plan.md](3_plan.md) §4.2) |
| «Procedure … does not exist» | Falta el `DELIMITER $$` en el script y se partió en el primer `;`: las **tablas sí se crearon** y los procedimientos no |
| Los procedimientos o las tablas de la v2 no existen | Los `.sql` corrieron en otra base: `MARIADB_DATABASE` debe ser **`investigacion`**, igual que en `init.sql`. Recree con `down -v` |
| `procedimientos_v2.sql` falla en el primer `ALTER` o `CREATE` | Corrió antes que `init.sql`, o `init.sql` falló. Los archivos se ejecutan en orden alfabético: revise los nombres |
| «Cannot execute queries while other unbuffered queries are active» | Falta el `closeCursor()` después del `CALL` (error 2014) |
| `POST` del grupo guarda el grupo pero no el detalle | Falta `nextRowset()` al leer, o el JSON del detalle no llega al procedimiento (`JSON_TABLE` pide MariaDB 10.6 o superior) |
| Un grupo inexistente da 409 en vez de 404 | Falta distinguir `'45001'` de `'45000'` en `errores_de_integridad.php` |
| Un grupo con semilleros se retira sin error | El procedimiento no revisa los semilleros activos, o los semilleros se crearon con `activo = 0` |
| Un desplegable sale vacío | La API respondió `204` y no se trató antes del `foreach`; o el listado que lo llena no filtra bien |
| El desplegable ofrece cosas retiradas | Falta `WHERE activo = 1` en el listado que lo llena |
| «(ninguna)» guarda `0` | Se usó `(int) ''`: convierta la cadena vacía a `null` antes |
| Un nombre con `<` o `'` rompe la pantalla | Falta `htmlspecialchars` al imprimir |
| El `PUT` del grupo retira semilleros que no se veían | El formulario de edición mostró solo cinco renglones: debe mostrar todos los actuales |
| La pantalla se ve sin estilos | El router se está tragando los archivos estáticos: falta el `return false` cuando el archivo existe; revise también `publico/marca.css` |
| 500 en todos los endpoints | MariaDB apagada o DSN mal apuntado (Docker: host `mariadb`; local: `localhost:13330`) |
| El `index.php` de la API da error fatal en `/api/grupo_investigacion` | Falta el `require_once` del controlador, o el bloque usa variables que no existen (`$pdo`, `$id`) |
| Puerto 8110, 8111, 8105 o 13330 ocupado | Otro proyecto del curso está corriendo: apáguelo (`docker compose down` en ESE proyecto) |

## 7. La guía está cumplida cuando…

- [ ] `docker compose up -d --build` deja los cuatro servicios funcionando.
- [ ] `GET /` dice `"version": "v2"` y lista los 16 recursos.
- [ ] La v1 sigue igual (§3.1).
- [ ] El grupo con 3 semilleros se guarda en una petición y devuelve sus `id` (§3.3).
- [ ] Un fallo a mitad (universidad inexistente) no deja nada guardado.
- [ ] `PATCH` del grupo da `405`; `DELETE` de un grupo con semilleros da `409`.
- [ ] Repetir una pareja da `409`; retirar una inexistente da `404`.
- [ ] Ningún rechazo de integridad responde `500`.
- [ ] La pantalla pasa el recorrido a mano de §4.2 y la prueba de la API apagada.
- [ ] La prueba de capas pasa sin MariaDB.
