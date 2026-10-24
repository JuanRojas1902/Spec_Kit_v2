# Cómo construir la versión 2 con IA — por chat o con un IDE agéntico

> Guía para trabajar la v2 con ayuda de IA por **cualquiera de los dos
> caminos**: un chat web (Gemini, DeepSeek, ChatGPT, Claude…) o un IDE
> agéntico (Antigravity, Cursor, Claude Code, Copilot en VS Code…).
> La clave es la misma en ambos: la IA no inventa — **sigue el spec kit**.
> Usted verifica; la IA propone (chat) o ejecuta bajo su supervisión (IDE).

---

## 0. Lo que cambia respecto de la guía de la v1

La v1 se construyó **desde cero**. La v2 **parte de la v1 funcionando**:

1. **El punto de partida es código que ya existe.** Las seis tablas de
   catálogo, con su API y su pantalla, siguen en pie.
2. **Hay que vigilar que lo viejo no se rompa.** El criterio 17 dice que la
   regresión de la v1 pasa completa. Es normal que una IA «mejore» de paso algo
   que funcionaba: eso es una regresión, y hay que devolverla.
3. **Aparecen tentaciones nuevas.** Como hay diez recursos nuevos y muchos se
   parecen, la IA va a proponer un enrutador genérico, o darle a todos los
   mismos seis verbos. El Artículo 10 lo prohíbe, y las reglas del prompt lo
   dicen explícito.

## 1. Los dos caminos, en una tabla

| | **Camino A: chat web** | **Camino B: IDE agéntico** |
|---|---|---|
| Herramientas | Gemini, DeepSeek, ChatGPT, Claude (web) | Antigravity, Cursor, Claude Code, Copilot agente |
| ¿Cómo conoce la spec? | Usted le **sube los 9 archivos** | El agente **lee `docs/spec_kit/` de su proyecto** |
| ¿Quién escribe los archivos? | Usted copia y pega lo que la IA propone | El agente |
| ¿Quién ejecuta los comandos? | Usted, y pega la salida | El agente (pidiendo permiso) |
| Su papel | Operador | Supervisor |
| Riesgo típico | La IA olvida el contexto en chats largos | El agente avanza rápido y agrega lo que no se pidió |

En ambos casos «terminado» significa lo mismo: **los 18 criterios de
aceptación de `2_spec.md` en verde**, verificados con `7_quickstart.md`,
corrido por usted.

---

## 2. Qué subirle (los 9 archivos)

| # | Archivo | Papel |
|---|---|---|
| 1 | `docs/spec_kit/1_constitution.md` | Las reglas permanentes (Artículos 1 a 12) |
| 2 | `.../v2_<nombre>/2_spec.md` | QUÉ construir y los 18 criterios |
| 3 | `.../v2_<nombre>/3_plan.md` | CÓMO: estructura, capas y piezas nuevas |
| 4 | `.../v2_<nombre>/4_research.md` | Las decisiones y sus alternativas descartadas |
| 5 | `.../v2_<nombre>/5_data_model.md` | Las tablas, sus relaciones y **quién escribe cada columna** |
| 6 | `.../v2_<nombre>/6_contratos.md` | Los endpoints y las pantallas, exactos |
| 7 | `.../v2_<nombre>/7_quickstart.md` | El smoke test de validación |
| 8 | `.../v2_<nombre>/8_tasks.md` | Las fases, en orden |
| 9 | `.../v1_<nombre>/2_spec.md` | **La spec de la v1**, para que la IA sepa qué NO debe romper |

**No suba** `9_checklist.md` (es para que lo pase una persona antes de
programar) ni el mapa de versiones: le revelaría lo que viene, y una versión no
anticipa a la siguiente.

Además de los documentos, hay **artefactos que NO se suben al chat ni los
genera la IA**: `db/init.sql` (la v1), `front_php/publico/bootstrap.min.css`,
`bootstrap.bundle.min.js` y `marca.css` (Art. 12). Se **copian tal cual**.

---

## 3. Prepare SU proyecto (ANTES de abrir el chat)

Parta de **su proyecto de la v1**, que ya funciona. No lo reconstruya.

### 3.1 Cree las carpetas nuevas

En la terminal integrada de VS Code (PowerShell), parado en la raíz:

```powershell
mkdir docs\spec_kit\versiones\v2_investigacion
```

Las carpetas `api_investigacion\` (con `controladores`, `servicios`,
`repositorios`, `modelos`, `excepciones`, `pruebas`), `front_php\` y `db\` ya
existen desde la v1.

### 3.2 Los archivos VACÍOS de la v2

**Usted los va llenando** uno a uno, pegando el código que la IA le entregue.
Que nazcan vacíos y con su nombre puesto da forma al trabajo: se ve cuántas
piezas son y dónde va cada una.

```powershell
# Los diez recursos nuevos de la API: modelo, interfaces, servicio, repositorio y controlador
$recursos = 'Docente','GrupoInvestigacion','Semillero','ParticipaGrupo','ParticipaSemillero','GrupoLinea','SemilleroLinea','AcLinea','OdsLinea','AaLinea'
foreach ($r in $recursos) {
    New-Item "api_investigacion\modelos\$r.php",
             "api_investigacion\repositorios\IRepositorio$r.php",
             "api_investigacion\repositorios\Repositorio${r}MariaDB.php",
             "api_investigacion\servicios\IServicio$r.php",
             "api_investigacion\servicios\Servicio$r.php",
             "api_investigacion\controladores\Controlador$r.php" -ItemType File
}

# Lo nuevo de la v2 que no es un recurso
New-Item api_investigacion\excepciones\ConflictoDeIntegridadExcepcion.php,
         api_investigacion\repositorios\errores_de_integridad.php,
         db\procedimientos_v2.sql,
         docs\spec_kit\versiones\v2_investigacion\GUIA_IA2.md -ItemType File
```

Las vistas del front (los nombres de las puente son una propuesta; deben
coincidir con `6_contratos.md` §10):

```powershell
$vistas = 'docentes','grupos','semilleros','docentes_grupo','docentes_semillero','lineas_grupo','lineas_semillero','areas_conocimiento_linea','ods_linea','areas_aplicacion_linea'
foreach ($v in $vistas) { New-Item "front_php\vistas\${v}_lista.php" -ItemType File }

# Formularios: semilleros no tiene (es solo lectura); grupos tiene además la pantalla de detalle
foreach ($v in $vistas | Where-Object { $_ -ne 'semilleros' }) { New-Item "front_php\vistas\${v}_formulario.php" -ItemType File }
New-Item front_php\vistas\grupos_detalle.php -ItemType File
```

> **Fíjese y cuente.** `semilleros` solo tiene lista (es de lectura), `grupos`
> tiene **una vista de más** (`grupos_detalle.php`) y las puente tienen lista y
> formulario pero ningún botón de editar. Esa asimetría no es un descuido: es lo
> que la versión viene a enseñar.
>
> **Hay front** porque la versión incluye su pantalla (Art. 1.1). Es la mitad
> del trabajo.

Un `README.md` por carpeta (Fase 9 de `8_tasks.md`) se crea al final, cuando ya
se sabe qué hay en cada una.

### 3.3 Los archivos que vienen DADOS: cópielos del repositorio del curso

| Del clon del curso | A su proyecto |
|---|---|
| `front_php\publico\bootstrap.min.css` y `bootstrap.bundle.min.js` | `front_php\publico\` (ya los tiene de la v1) |
| `front_php\publico\marca.css` y la carpeta `marca\` | `front_php\publico\` y la raíz (Art. 12) |
| `docs\spec_kit\1_constitution.md` | `docs\spec_kit\` |
| Los `.md` de la v2 (los suyos, de §2) | `docs\spec_kit\versiones\v2_investigacion\` |

El script `db\init.sql` de la v1 **no se toca**. Los cambios de la v2 van en
`db\procedimientos_v2.sql`.

### 3.4 Compruebe antes de empezar

- [ ] La v1 arranca: `docker compose up -d --build` y las seis pantallas
      funcionan.
- [ ] `docs\spec_kit\1_constitution.md` existe y tiene contenido.
- [ ] `docs\spec_kit\versiones\v2_investigacion\` tiene los documentos 2 a 9.
- [ ] `MARIADB_DATABASE` en `docker-compose.yml` es **`investigacion`** (igual
      que en `init.sql`).
- [ ] `front_php\publico\` tiene Bootstrap y `marca.css` con contenido.
- [ ] No quedan `[NECESITA ACLARACIÓN]` abiertos, o los que quedan los conoce
      y los dirá a la IA (regla 14 del prompt).
- [ ] Pasó `9_checklist.md` con una persona y el veredicto está en verde.

### 3.5 ¿Desde qué carpeta se corre cada comando?

| Comando | Se corre desde |
|---|---|
| `docker compose ...` | La **raíz** de su proyecto |
| `php -l archivo.php` · `php pruebas\prueba_capas.php` | `api_investigacion\` |
| `python pruebas_humo\humo_front.py` | La **raíz** de su proyecto |

**A la terminal SOLO se le pegan COMANDOS** (lo que viene en las cajitas de
código). Si pega texto de un mensaje, la terminal intentará ejecutar cada
palabra y llenará la pantalla de errores. Al chat, texto; a la terminal,
comandos.

---

## 4. Camino A — El prompt del chat (cópielo tal cual)

**Antes de enviar el primer mensaje, tres chequeos:**

1. **Los 9 adjuntos** aparecen todos.
2. **Active el modo de razonamiento** si el chat lo tiene: sigue mucho mejor
   las reglas estrictas.
3. **Apague la búsqueda web**: no se necesita y puede traer código de internet
   por fuera de la spec.

```
Actúa como mi asistente de programación para construir la VERSIÓN 2 de un
proyecto universitario. Te adjunto 9 documentos: una constitución (reglas
permanentes), el spec kit de la versión 2, y la spec de la versión 1 para que
sepas qué ya existe y no se puede romper.

El proyecto es PHP 8.3 puro (sin frameworks ni Composer) + MariaDB. Son DOS
programas que se construyen a la vez: la API (api_investigacion) y su front
(front_php), que habla con ella por HTTP. Si en tu respuesta aparece OTRO
lenguaje o framework, significa que no leíste los adjuntos: detente y dímelo.

Esta versión NO parte de cero: parte de la versión 1, que ya tiene seis
recursos de catálogo funcionando de punta a punta. Yo ya tengo ese código.

REGLAS DE TRABAJO (no negociables):

1. La especificación manda. No agregues NADA que los documentos no pidan: ni
   frameworks, ni Composer, ni librerías, ni tablas extra, ni motores extra,
   ni disparadores (triggers), ni fábricas "por si acaso", ni mejoras de tu
   cosecha. Si crees que falta algo, pregúntame antes.
2. NO ROMPAS LA v1. Las seis tablas de catálogo deben seguir respondiendo
   exactamente igual (criterio 17 de 2_spec.md). Si vas a tocar un archivo de
   la v1, dime primero por qué.
3. Vamos a seguir 8_tasks.md FASE POR FASE, en orden. En cada fase:
   a. Me explicas en 3-5 líneas qué vamos a hacer y por qué.
   b. Me entregas los archivos DE A UNO: primero la ruta exacta y el
      contenido COMPLETO de UN solo archivo, listo para copiar y pegar, con
      la cabecera y los comentarios didácticos en español que exige la
      constitución. Esperas mi "listo" y solo entonces sigues.
   c. Al cerrar la fase me dices su comando de verificación y qué salida
      esperar.
   NOTA: la estructura de carpetas y los archivos vacíos YA EXISTEN: no me
   des comandos para crearlos; tu trabajo es dictarme el CONTENIDO.
4. NADA GENÉRICO. Es el Artículo 10 de la constitución y aquí hay diez
   recursos nuevos que se parecen, así que la tentación es grande:
   - una ruta por recurso, escrita con su nombre (`/api/docente`), nunca
     `/api/{tabla}`;
   - una función por recurso en el ensamblador y en el cliente del front,
     nunca un `switch` ni nombres de clase armados con texto
     ("Repositorio{$recurso}MariaDB" está prohibido);
   - código repetitivo-pero-legible antes que compacto-pero-mágico.
5. NO TODO RECURSO ES UN CRUD, y no los fuerces a serlo:
   - `grupo_investigacion` es el ÚNICO maestro-detalle: sus cinco verbos
     pasan por procedimientos almacenados (`CALL sp_..._grupo_investigacion`),
     no tiene PATCH, el detalle viaja como UN parámetro JSON y el repositorio
     lee dos resultados con nextRowset() y llama closeCursor();
   - `semillero` es SOLO LECTURA: nace dentro del grupo;
   - las siete tablas puente solo se asignan y se retiran (sin PUT, PATCH ni
     columna `activo`), con las dos claves en la ruta;
   - el resto de las tablas usa SQL con parámetros `?` en su repositorio.
6. LA INTEGRIDAD LA DEFIENDE LA BASE. No escribas comprobaciones previas del
   tipo "¿existe ese grupo?" antes de insertar: se intenta, y se traduce el
   rechazo del motor a un 409 con mensaje en español, en UN solo archivo
   (errores_de_integridad.php). Se lee así: 1062, 1452 y 1451 en
   $e->errorInfo[1]; el SIGNAL se lee al revés, en $e->errorInfo[0]:
   '45000' -> 409 y '45001' -> 404. getCode() NO sirve (da '23000' para todos).
7. NO HAY DISPARADORES en esta versión. Si ves triggers, total_grupos o
   sp_recontar_grupos en algún lado, sobran: quítalos.
8. El código debe cumplir 6_contratos.md al pie de la letra: mismos verbos,
   rutas, códigos de estado y formatos, incluidos el 409 y el 405.
9. Todo en español: nombres, comentarios y mensajes. PHP 8.3 con
   declare(strict_types=1) en cada archivo.
10. Ninguna pantalla le habla al usuario en jerga: nada de verbos HTTP,
    códigos de estado, /api/, PDO, MariaDB ni nombres de columna en el texto
    que se ve. El borrado lógico se llama "Retirar", nunca "Borrar".
11. Bootstrap y marca.css ya están en front_php/publico/: enlaza esos
    archivos locales, NUNCA un CDN, y no me dictes su contenido. Los estilos
    usan var(--umv-...), no colores escritos a mano; los errores van en rojo
    alerta, no en ocre.
12. Los errores NO nos frenan. Si te pego un error, lo diagnosticas y me das
    el archivo completo corregido; si no sale rápido, seguimos y lo retomamos
    al final.
13. Yo trabajo en Windows con VS Code (terminal PowerShell) y Docker Desktop.
    Los puertos son los de la constitución: front 8110, API 8111, phpMyAdmin
    8105 y MariaDB 13330. La base se llama `investigacion`.
14. Las columnas de `docente`, `semillero` y las siete tablas puente son una
    PROPUESTA hasta contrastarlas con el esquema oficial. No las cambies por
    tu cuenta. Si encuentras un [NECESITA ACLARACIÓN] sin resolver, pregúntame.
15. Todo archivo PHP lleva una cabecera: qué es, por qué está donde está y qué
    NO hace.

Al final, la versión 2 está TERMINADA solo cuando pasan los 18 criterios de
aceptación de 2_spec.md, verificados con el smoke test de 7_quickstart.md.

Empieza: resume en máximo 10 líneas qué vamos a construir y en qué se
diferencia de la v1, y luego arranca con la Fase 0.
```

### 4.1 El método de la conversación

1. **Pegue primero, ejecute cuando quiera.** Lo obligatorio es pegar cada
   archivo en su ruta y responder «listo».
2. **No se quede varado en un error.** Anótelo, siga con las fases y retómelo
   al final.
3. **El punto de control real es el smoke test final.** Péguele CADA error tal
   cual salga, completo.
4. **Vigile las cuatro tentaciones de esta versión:**
   - un enrutador o un ensamblador genérico (regla 4);
   - un `if` que comprueba llaves foráneas antes de insertar (regla 6);
   - darle verbos que no le corresponden a un recurso: un `PATCH` al grupo, un
     `POST` suelto de `semillero`, un `PUT` a una tabla puente (regla 5);
   - triggers «para que el contador se mantenga solo» (regla 7).

   Las cuatro «funcionan» y las cuatro están mal por razones que los
   documentos explican. Si aparecen, cite la regla y pida el archivo
   corregido.
5. **Si la IA «mejora» algo de la v1**, devuélvalo: es una regresión.
6. **Si el chat pierde el contexto**, abra uno nuevo, vuelva a subir los 9
   documentos y agregue: «Ya tengo construidas las fases 0 a N; te pego el
   código actual. Continuemos en la fase N+1».
7. **Anote en la §7 lo que la IA hizo mal.** Es lo que se entrega, y lo que
   más vale.

---

## 5. Camino B — El prompt para el agente (cópielo tal cual)

### 5.1 Preparación

1. Abra SU carpeta (la que ya tiene la v1) en el IDE, con Docker Desktop
   corriendo.
2. Copie dentro los documentos de la v2 (misma estructura que en §3) y
   `marca.css`.
3. Active el modo agente.

### 5.2 El prompt

```
Construye la VERSIÓN 2 de este proyecto, partiendo de la versión 1 que ya
está en esta carpeta y funciona.

Primero lee, en este orden, los documentos que están bajo docs/spec_kit/:
1_constitution.md, y de versiones/v2_investigacion/ los archivos 2_spec,
3_plan, 4_research, 5_data_model, 6_contratos, 7_quickstart y 8_tasks. Lee
también versiones/v1_*/2_spec.md, para saber qué NO puedes romper. Después
resume en máximo 10 líneas qué vas a construir y espera mi confirmación antes
de tocar nada.

docs/spec_kit/ es solo lectura: no la modifiques.

REGLAS (no negociables):

1. La especificación manda. No agregues NADA que los documentos no pidan.
2. NO ROMPAS LA v1: las seis tablas de catálogo deben seguir respondiendo
   igual (criterio 17 de 2_spec.md).
3. Sigue 8_tasks.md FASE POR FASE. Al terminar cada fase, EJECUTA su
   verificación, muéstrame el resultado real, y espera mi OK.
4. NADA GENÉRICO (Artículo 10): una ruta y una función por recurso, con su
   nombre escrito. Prohibido armar nombres de clase con texto.
5. grupo_investigacion es el único maestro-detalle: sus cinco verbos pasan
   por procedimientos almacenados, sin PATCH, con el detalle como un solo
   parámetro JSON. semillero es solo lectura. Las siete tablas puente solo se
   asignan y se retiran, sin columna `activo`.
6. La integridad la defiende la base: no comprobaciones previas; se traduce
   el rechazo a 409 en errores_de_integridad.php (1062/1452/1451 en
   errorInfo[1]; SIGNAL '45000' -> 409 y '45001' -> 404 en errorInfo[0]).
7. Sin disparadores. Quita los triggers, total_grupos y sp_recontar_grupos
   que hoy trae procedimientos.sql.
8. Todo en español, PHP 8.3 con declare(strict_types=1), con cabecera en cada
   archivo (qué es, por qué está ahí y qué NO hace).
9. Esta versión incluye SU FRONT. El front no toca la base: nada de new PDO
   ahí, y NO le hagas require de ningún archivo de api_investigacion aunque
   los dos estén en PHP y funcionaría. En el compose, el servicio del front
   no lleva credenciales de base de datos ni depends_on de mariadb.
10. Bootstrap y marca.css ya están en front_php/publico/: enlaza esos
    archivos locales, nunca un CDN. Estilos con var(--umv-...); errores en
    rojo alerta. El botón de borrado lógico dice «Retirar».
11. Las columnas de docente, semillero y las puente son PROPUESTA: no las
    cambies sin avisarme.
12. Al final, corre el smoke test completo de 7_quickstart.md —incluida la
    prueba de apagar la API con la base encendida— y muéstrame la evidencia
    de los 18 criterios. La versión no está terminada hasta que los 18 estén
    en verde.
```

### 5.3 El método de supervisión

1. **Revise cada diff antes de aceptar.** Si un archivo no está en la
   estructura de `3_plan.md` §2, pregunte por qué existe.
2. **Exija la evidencia, no el relato.** «Ya pasa la fase 3» no vale: pida la
   salida real del comando.
3. **Vigile el alcance.** Si aparece un `composer.json`, una tabla `usuario`,
   un trigger o una fábrica multi-motor, el agente se salió de la v2.
4. **Vigile las cuatro tentaciones** de §4.1 punto 4.
5. **El cierre lo corre usted.** El recorrido a mano de `7_quickstart.md` §4.2
   **no lo puede hacer el agente**: juzga si la pantalla se entiende, y eso no
   lo mide ningún guion.

---

## 6. Por qué funciona (la lección del curso)

Esto ES spec-driven development: la misma IA que con «hazme una API de grupos
de investigación en PHP» produce cualquier cosa, con una constitución + spec +
plan + tareas produce EL sistema especificado.

Y la v2 agrega una lección que la v1 no podía dar: **la especificación también
sirve para decir que NO.** Las cuatro tentaciones de §4.1 —el enrutador
genérico, la comprobación previa de llaves foráneas, los verbos que no le
corresponden a un recurso, los triggers— son cosas que una IA propone con
naturalidad porque funcionan. Lo que las descarta no es el gusto de nadie: es
un documento escrito antes, con su razón al lado.

---

## 7. Registro de prompts y correcciones (lo que se entrega)

> **Esto es lo que más vale.** El spec kit dice **qué** construir; el código es
> **lo construido**; el prompt es **cómo se pasó de uno al otro**. Y lo que
> tuvo que corregirle a la IA dice más de usted que el código final: en la
> sustentación individual distingue a quien dirigió el trabajo de quien pegó
> una respuesta.
>
> Se llena **mientras** se trabaja, no al final. Una fila por cada prompt que
> de verdad usó.

**Herramienta y modalidad usadas:** _(chat o IDE agéntico, y cuál)_
**Integrante:** _(su nombre)_

### 7.1 Los prompts

| # | Fase | Prompt (resumido o completo) | Qué devolvió la IA | ¿Hubo que corregir? |
|---|---|---|---|---|
| 1 | 0 | El prompt de arranque de §4 o §5.2 | | |
| 2 | | | | |

### 7.2 Lo que tuvo que corregirle a la IA

Una fila por cada vez que la IA se equivocó o se salió de la spec. Anote **qué
pidió**, **qué salió mal**, **qué corrección lo arregló** y **qué regla de los
documentos lo explicaba**.

| # | Archivo | Qué salió mal | La corrección que lo arregló | Regla o documento |
|---|---|---|---|---|
| 1 | | | | |

**Ejemplos de lo que probablemente va a aparecer** (bórrelos si no le pasaron,
o déjelos si le pasaron de verdad):

| Qué suele salir mal | La corrección | Dónde está la regla |
|---|---|---|
| Un `PATCH` para el grupo, «por consistencia con las demás» | Quitarlo: debe responder 405 | `6_contratos.md` §2, D8 |
| Comprobar con un `SELECT` que el padre existe antes de insertar | Quitar la comprobación y traducir el error del motor | `4_research.md` D2 |
| Leer el error con `getCode()`, y todo cae en 500 | Usar `errorInfo[1]` y, para el `SIGNAL`, `errorInfo[0]` | `4_research.md` D6 |
| Olvidar `closeCursor()` tras el `CALL` | Agregarlo (error 2014 en la consulta siguiente) | `3_plan.md` §4.3 |
| Un `<select>` opcional que guarda `0` | Convertir `''` a `null` | `3_plan.md` §4.6 |
| Triggers y `total_grupos` en `procedimientos.sql` | Quitarlos: la v2 no los pide | `4_research.md` D12 |
| El bloque del grupo en `index.php` con `$pdo` e `$id` que no existen | Rehacerlo con el ensamblador y `$idStr` | `8_tasks.md` Fase 4 |
| `MARIADB_DATABASE: investigacion_php_local` | Cambiarlo a `investigacion` | `8_tasks.md` Fase 0 |
| El `PUT` de edición muestra solo cinco renglones y retira semilleros que no se veían | Mostrar todos los actuales más renglones vacíos | `6_contratos.md` §10.1 |
| Un enrutador `/api/{tabla}` o clases armadas con texto | Una ruta y una función por recurso | Art. 10 |

### 7.3 Lo que no se tocó

Anote aquí si la IA propuso cambiar algo de la v1 y usted lo devolvió (la
regresión que evitó):

-
-

### 7.4 Para la sustentación

Elija **un archivo** y practique explicarlo en voz alta. Dos sugeridos, por lo
que enseñan:

- `repositorios/errores_de_integridad.php`: por qué el `SIGNAL` se lee en
  `errorInfo[0]` y los demás en `errorInfo[1]`.
- `repositorios/RepositorioGrupoInvestigacionMariaDB.php`: cómo viaja el
  detalle como JSON, por qué `nextRowset()` y `closeCursor()`, y cómo se
  garantiza el «todo o nada».
