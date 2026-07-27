# Notas de rendimiento y metodología — structured-CoT en el Ralph loop

Fecha: 2026-07-23. Contexto: probar si structured-CoT (pre-trigger grammar)
acelera el loop agéntico de Hermes sobre Ternary-Bonsai-27B en la RTX 3080.

## Tabla resumen — todas las ejecuciones y su configuración

Todo el proyecto en una tabla; debajo, la **leyenda** numerada (los superíndices ¹–⁸ de las cabeceras/valores enlazan con ella).

**Común a todas las filas:** modelo Ternary-Bonsai-27B-Q2_0 (2-bit) en RTX 3080 · contexto servido `-c` = **64K** (uniforme) · `reasoning_budget` = -1 (OFF) salvo `ralph13` · Harness = Hermes salvo las filas `atomic-agent` y `pi`.

| Run          | Harness          | Binario¹        | scot (plantilla)²   | Refuerzo AGENTS.md³ | Config clave / cambio⁴                               | Cap GPU (W)⁵ | Cap max_tok⁶ | Idioma | Tareas           | Tiempo total | Turnos            | Spiral⁷ | Resultado / hallazgo                                  |
| ------------ | ---------------- | --------------- | ------------------ | --- | --------------------------------------------------- | --- | --- | ------ | ---------------- | ------------ | ----------------- | --- | ----------------------------------------------------- |
| ralph2       | Hermes           | Prebuilt        | — (no-scot)        | — | baseline histórico · sin proxy | 280 | 8192 | ES     | 10               | ~20m         | —                 | — | 10/10, juego funcional                                |
| ralph3       | Hermes           | Parcheado       | GOAL/PLAN/CHECK    | — | 1er scot · sin proxy | 280 | 8192 | ES     | 10               | 44m24s       | —                 | B | 10/10; **death-spiral** t9=1168s (plantilla mala)     |
| ralph4       | Hermes           | Parcheado       | ASSESS/ACTION | — | baseline scot | 280 | 8192 | ES     | 10               | **15m06s**   | 100               | — | 10/10 — resultó **suerte** (ver ralph8)               |
| ralph5       | Hermes           | Parcheado       | — (no-scot)        | — | control (sin scot) | 280 | 8192 | ES     | 10               | 34m06s       | 202               | — | 10/10; scot-A ≈ ½ turnos vs esto                      |
| ralph6       | Hermes           | Parcheado       | ASSESS/ACTION | v1 | — | 280 | 8192 | ES     | 10               | 22m58s       | 122               | B? | v1 fragmentó → peor; adherencia parada 77%            |
| ralph7       | Hermes           | Parcheado       | ASSESS/ACTION | v2 | — | 280 | 8192 | ES     | 10               | 18m32s       | 117               | B | v2 > v1 pero no bate a ralph4                         |
| ralph8       | Hermes           | Parcheado       | ASSESS/ACTION | — | réplica de ralph4 | 280 | 8192 | ES     | 10               | 29m56s       | 131               | B | **varianza ~2×**: ralph4 fue suerte                   |
| ralph9       | Hermes           | Parcheado       | ASSESS/ACTION | — | **tail troceado** (init→3, verify→3) | 280 | 8192 | ES     | 14               | **15m33s**   | 105               | — | 14/14; mata spiral del tail (estructural)             |
| ralph10      | Hermes           | Parcheado       | ASSESS/ACTION | — | réplica de ralph9 | 280 | 8192 | ES     | 14               | 22m26s       | 116               | B | verify estable 83s, pero spiral **migra** a lógica    |
| ralph11      | Hermes           | Parcheado       | ASSESS/ACTION | — | lógica atomizada | 280 | 8192 | ES     | 18               | 22m30s       | 149               | B | 18/18; spiral migra a **frases** (368s)               |
| ralph12      | Hermes           | Parcheado       | ASSESS/ACTION | — | (lectura de trazas) | 280 | 8192 | ES     | 20               | —            | —                 | A | **Spiral A**: turno "Done."×4094 = 24k chars            |
| ralph13      | Hermes           | Parcheado       | ASSESS/ACTION | — | **DRY⁸ + reasoning_budget=64** | 280 | 8192 | ES     | 20               | (abortado)   | —                 | B | budget=64 **backfire** (thrash task1)                 |
| ralph14      | Hermes           | Parcheado       | ASSESS/ACTION | — | **DRY⁸ solo** (budget -1) | 280 | 8192 | ES     | 20               | (parado)     | —                 | B | DRY mata Spiral A; **Spiral B persiste** (task3)          |
| atomic-agent | **atomic-agent** | Parcheado :8080 | — (grammar propia) | — | scot no porta | 280 | 8192 | ES     | tarea 01         | —            | —                 | B | **no crea ni el fichero** (Spiral B ×3)                 |
| pi           | **pi**           | Parcheado :8080 | — (grammar propia) | — | PTY · thinkingFormat deepseek | 280 | 8192 | ES     | tarea 01         | 166s (crear) | —                 | B | crea OK; **edición falla** por rutas (traza)          |
| ralph-en     | Hermes           | Parcheado       | ASSESS/ACTION | — | dir con **COLISIÓN** de nombre | 320 | 8192 | **EN** | 20 (parado 3/20) | —            | —                 | B | cascada de path-fails, ~185s/tarea                    |
| **kbd**      | Hermes           | Parcheado       | ASSESS/ACTION | PATH | **dir neutro + cebos apartados** | 280 | 8192 | **EN** | 20               | **60m45s**   | 290 ct / 133 stop | B | **20/20, 0 fallos, 0 path-fails**; lento (frases+cap) |
| kbd / hot-01 | Hermes           | Parcheado       | ASSESS/ACTION | PATH | GPU **caliente** | 280 | 8192 | EN     | tarea 01         | **196s**     | —                 | — | vs **638s en frío** → 3,3× (aísla warmup)             |

### Leyenda

**¹ Binario** — `Prebuilt` = release histórico del fork; `Parcheado` = fork PrismML recompilado con el parche `pre-trigger-grammar` (lo que permite scot). Diferencia de velocidad ≈5% (dentro del ruido).

**² scot (plantilla)** — *structured-CoT*: gramática GBNF que obliga a razonar con un formato fijo. `ASSESS/ACTION` (de **decisión**, buena → ½ turnos) · `GOAL/PLAN/CHECK` (de **planificación**, mala → death-spiral) · `— (no-scot)` = sin gramática · `— (grammar propia)` = el harness (atomic-agent/pi) usa su propia gramática de tools y **no admite scot**.

**³ Refuerzo AGENTS.md** — ediciones de *prompt engineering* del `AGENTS.md` (eje aparte de scot y del proxy): `v1` ("1 tool/turno" → **empeoró**, fragmentó) · `v2` ("mín. turnos" → mejor que v1, no bate al baseline) · `PATH` (regla de ruta: "usa el nombre relativo, nunca rutas absolutas") · `—` = sin refuerzo.

**⁴ Config clave / cambio** — el cambio distintivo del run. Vocabulario:
- *proxy* = `scot_proxy.py` intercalado (:8080→:8081) que **solo observa** (vuelca el `reasoning_content` a un log para poder leer las trazas); **no influye** en la generación, ni siquiera en el control. `sin proxy` = sin captura de trazas.
- *control* = run **sin scot**, para comparar.
- *tail troceado · lógica atomizada · réplica* = decisiones de **decomposición** de tareas.
- *DRY⁸ · reasoning_budget* = ajustes de **sampler** (ver ⁸).
- *PTY* = envoltura `script -qfec` que da un pseudo-terminal (pi se cuelga sin él).
- *colisión · dir neutro + cebos apartados* = fixes del **atractor de rutas** (dir de trabajo que el 2-bit confundía con otro que tenía un `typing-game.html` de cebo).

**⁵ Cap GPU (W)** — power limit de la 3080 (`nvidia-smi -pl`). **280 W** en todos salvo `ralph-en` (**320**: el reboot reseteó el cap al default; por eso se paró ese run y se re-fijó 280 en `kbd`).

**⁶ Cap max_tok** — tope de *salida* por respuesta (`max_tokens`) = **8192** en todos. Es el "cap 8192" que provoca el **Spiral A** (el modelo genera hasta toparlo).

**⁷ Spiral** — tipo de enrosque cuando el 2-bit no cierra: **A** = repetición degenerada en UNA generación (ej. `"Done."`×4094; la mata el DRY) · **B** = thrash agéntico (muchos turnos, tools distintas, confusión de estado/rutas; **techo de inteligencia del 2-bit**, ninguna palanca de inferencia lo cura). Trace-verificados solo en `ralph8/11/12` y `kbd`; el resto **inferido**. `B?` = probable · `—` = sin enrosque. **A solo es posible sin DRY** (≤`ralph12`).

**⁸ DRY** — *Don't Repeat Yourself*, sampler de llama.cpp que **penaliza repetir secuencias** ya generadas (castigo exponencial pasado `allowed-length`). Config: `--dry-multiplier 0.8 --dry-base 1.75 --dry-allowed-length 2`. Mata el **Spiral A** a coste cero; no toca el B. ON por defecto desde `ralph13`.

## El camino del experimento (trazabilidad)

Se lee de arriba abajo: cada prueba **mitiga el error de la anterior**. Resultado: 🟢 mejor · 🔴 peor/fallo · 🟡 mixto/igual · ⚪ baseline/diagnóstico.

Convención: **`ralphN`** = iteración N del *Ralph loop* (una tarea por proceso limpio, cola en carpetas `todo/done/fail`); el nombre de cada run es también el de su directorio de trabajo. Las filas van agrupadas en las **3 fases** del proyecto.

| # | Run | Qué se prueba (mitigación) | Contra el error de | Resultado | Aprendizaje |
|--|--|--|--|--|--|
| | **▸ Fase 1 · Experimento scot (Hermes, ES)** | | | | |
| 1 | ralph2 | baseline (prebuilt, sin scot) | — (punto de partida) | ⚪ ~20m, 10/10 | referencia inicial |
| 2 | ralph3 | meter **scot** (plantilla GOAL/PLAN/CHECK) | lentitud/varianza del baseline | 🔴 44m, death-spiral | plantilla de *planificación* → spiral |
| 3 | ralph4 | plantilla **ASSESS/ACTION** (decisión) | death-spiral de GOAL/PLAN/CHECK | 🟢 15m, ½ turnos | scot ayuda **si es de decisión** |
| 4 | ralph5 | **control sin scot** (mismo binario/proxy) | aislar el efecto real de scot | ⚪ 34m, 202 turnos | confirma scot ≈ **½ turnos** |
| 5 | ralph6 | refuerzo **v1** del `AGENTS.md` ("1 tool/turno") | baja adherencia de parada en ralph4 | 🔴 23m | microgestión **fragmenta** → peor |
| 6 | ralph7 | refuerzo **v2** ("mín. turnos") | fragmentación de v1 | 🟡 18.5m | mejor que v1, **no bate al baseline** |
| 7 | ralph8 | **replicar** ralph4 exacto | ¿los 15m eran reales? | 🟡 30m | ralph4 fue **suerte**; varianza ~2× |
| 8 | ralph9 | **atomizar el tail** (init/verify → 6 tareas) | spiral en el tail (verify 459s) | 🟢 15.5m, estructural | atomizar **mata el spiral local** |
| 9 | ralph10 | replicar ralph9 | confirmar el fix del tail | 🟡 22.5m | tail estable, pero **spiral migra** a lógica |
| 10 | ralph11 | **atomizar la lógica** (18 tareas) | spiral en render-finish | 🟡 22.5m | spiral migra a **frases** (whack-a-mole) |
| 11 | ralph12 | **leer las trazas** del proxy | ¿qué es el spiral por dentro? | ⚪ diagnóstico | identifica **Spiral A** (repetición) |
| 12 | ralph13 | **DRY** + reasoning_budget=64 | Spiral A + B | 🔴 backfire | budget=64 **empeora B**; abortado |
| 13 | ralph14 | **DRY solo** (budget -1) | backfire del budget | 🟢🟡 | DRY **mata A**; **Spiral B persiste** (= techo) |
| | **▸ Fase 2 · Comparativa de harness (test tarea 01)** | | | | |
| 14 | atomic-agent | cambiar de **harness** (atomic-agent) | ¿otro harness evita el techo? | 🔴 ni crea el fichero | mismo techo B, superficie peor |
| 15 | pi | cambiar de **harness** (pi) | fallo de atomic-agent | 🟡 crea, falla edición | mismo techo B; **no se recupera** |
| | **▸ Fase 3 · Inglés + fix de rutas (Hermes, EN)** | | | | |
| 16 | ralph-en | **traducir a inglés** (vuelta a Hermes) | ¿el español degrada al 2-bit? | 🔴 path-fails | dir con **colisión** = atractor (error nuestro) |
| 17 | **kbd** | **dir neutro + cebos apartados + regla PATH** | path-fails de ralph-en | 🟢 20/20, 0 path-fails | **fix de rutas** real y transferible |
| 18 | kbd / hot-01 | tarea 01 con **GPU caliente** | ¿los 638s eran arranque en frío? | 🟢 196s | confirma **cold-start** (3,3×) |

**El hilo conductor:** cada peldaño verde destapó el siguiente cuello de botella —
plantilla → varianza → tail → lógica → frases → repetición (Spiral A) → thrash (Spiral B) →
harness → idioma → rutas. Los fixes de *estrategia* (scot, atomizar, DRY, dir limpio) se
agotan en **Spiral B**, que es el techo de inteligencia del 2-bit.

## Aprendizajes

### ✅ Positivos — la estrategia SÍ mitiga el error

| Estrategia | Mitiga (error) | Evidencia |
|--|--|--|
| **scot plantilla ASSESS/ACTION** | el 2-bit divaga → dobla turnos | ~½ turnos vs no-scot (robusto, n=4) |
| **Tareas atómicas** (edits pequeños < cap 8192) | death-spirals de tareas gordas/abiertas | tail 738s → 174s; evita el caso de 1168s |
| **DRY sampler** | **Spiral A** (repetición degenerada) | `"Done."`×4094 no vuelve; un flag, coste 0 |
| **Verify/conteo FUERA del loop** (determinista) | tareas que disparan **Spiral B** | elimina el over-verify (459s) por diseño |
| **Dir neutro + cebos apartados + regla PATH** | cascada de path-fails (Spiral B por rutas) | `kbd`: **0 path-fails** en 20 tareas |
| **Loop-detection de Hermes** (`same_tool_failure`) | fallo de ruta que no se cierra | permite recuperarse y **completar** (vs pi) |

### ➖ Neutros — sin impacto claro (o confundido por el ruido)

| Cambio | Impacto | Nota |
|--|--|--|
| **Traducir tareas a inglés** | **ninguno medible** en velocidad/calidad | confundido por térmica + spirals + n=1; fiabilidad OK |
| **Refuerzo v2 del `AGENTS.md`** | no bate al baseline sin refuerzo | retorno incierto, tapado por la varianza ~2× |
| **Longitud del thinking por turno** | scot **no** la acorta | el ahorro es por nº de turnos, no por comprimir |
| **Proxy de logging** | **cero** (solo observa) | transparente; reenvía byte a byte |
| **Binario parcheado vs prebuilt** | ~5% tok/s (53 vs 56) | dentro del ruido de medición |

### ❌ Negativos — empeora o no funciona

| Cambio | Efecto | Por qué |
|--|--|--|
| **Cambiar de harness** (atomic-agent / pi) | **peor**: no completan | pierden la **gramática scot** (turnos decisivos) y la recuperación de bucles → el 2-bit va a pelo |
| **Plantilla scot GOAL/PLAN/CHECK** | death-spiral (44m) | *planificar* cada turno en vez de *decidir* → no cierra |
| **Refuerzo v1** ("1 tool/turno") | +turnos, peor | microgestión → fragmenta el uso de tools |
| **reasoning_budget agresivo** (=64) | empeora **Spiral B** | corta el razonamiento → no puede recuperarse de un error |
| **Cualquier palanca de inferencia vs Spiral B** | no lo cura | DRY/budget/temp no tocan el *thrash* agéntico = techo del 2-bit |


## El objetivo y el prompt de tarea

En **todos** los runs el objetivo fue el mismo: construir `typing-game.html` — un juego de mecanografía de **un solo fichero** (HTML + CSS + JS vanilla, sin CDNs), con 50 frases, resaltado de aciertos/errores letra a letra y estadísticas de WPM / precisión / tiempo. Lo único que cambiaba entre runs era **en cuántas tareas se troceaba** el trabajo y **el idioma**.

Cada tarea es un prompt corto y atómico que **lee el fichero actual y añade una pieza pequeña**. Ejemplo real (`09-render`):

```
Abre `typing-game.html`. El `<script>` ya tiene `loadSentence()`. Añade SOLO la función
`render(typed)` (edición pequeña) que recorre los spans `.char` de `#sentence` y asigna a
cada índice i la className:
- `char correct`   si typed[i] === current[i];
- `char incorrect` si existe typed[i] y difiere;
- `char current`   si i === typed.length;
- `char`           en el resto.
Nada más. Guarda y confirma.
```

## Sets de tarea (muchos menos que los runs)

17 directorios de trabajo (18 runs contando el sub-test `hot-01`), pero solo **5 sets de tarea distintos** — porque casi siempre relanzábamos el mismo objetivo con otra granularidad o idioma:

| Set | Nº tareas | Idioma | Decomposición | Runs que lo usaron |
|--|--|--|--|--|
| A | **10** | ES | original | ralph2 – ralph8 (7 runs) |
| B | **14** | ES | *tail* troceado | ralph9, ralph10 |
| C | **18** | ES | lógica atomizada | ralph11 |
| D | **20** | ES | frases troceadas | ralph12, ralph13, ralph14 · atomic-agent (solo tarea 01) |
| E | **20** | EN | traducción de D | ralph-en, kbd, kbd/hot-01 · pi (solo tarea 01) |

La app final es siempre la misma; el nº de tareas solo refleja **cómo de fino se partió el trabajo** (más tareas = edits más pequeños, para no pasar del cap 8192⁶ y acotar los spirals⁷).

## AGENTS.md — versiones (el contrato del proyecto)

`AGENTS.md` es el contrato que el agente lee al empezar cada tarea: objetivo, contrato de nombres (ids/clases/funciones) y reglas de edición. Hubo **5 variantes** (agrupadas por su contenido real):

| Variante | Idioma | Qué añade respecto a la base | Runs |
|--|--|--|--|
| **Base** | ES | objetivo + contrato de nombres + reglas ("edits pequeños, lee antes de editar, HTML válido") | ralph2–5, ralph8–14 |
| **+ Refuerzo v1** | ES | bloque `ASSESS/ACTION` + *"haz UNA sola llamada a herramienta y para"* | ralph6 |
| **+ Refuerzo v2** | ES | bloque `ASSESS/ACTION` + *"MÍNIMOS turnos, no re-verifiques lo ya comprobado"* | ralph7 |
| **Base EN** | EN | traducción del contrato base | ralph-en |
| **EN + PATH** | EN | + bloque **CRITICAL — FILE PATH** al frente (regla de ruta) | kbd, kbd/hot-01 |

**Extracto del contrato base (objetivo):**
```
App web de UN SOLO fichero `typing-game.html` (HTML + CSS + vanilla JS, sin CDNs).
Muestra frases aleatorias de una lista de 50; el usuario teclea y se resaltan
aciertos/errores letra a letra, con WPM, precisión y tiempo.
Se construye de forma INCREMENTAL: cada tarea LEE el fichero y AÑADE una pieza pequeña.
```

**Refuerzo v1 (ralph6) — la cláusula que fragmentó el trabajo (empeoró):**
```
ACTION: call-tool → aún falta algo: haz UNA sola llamada a herramienta y para.
```

**Refuerzo v2 (ralph7) — quita esa cláusula, orienta a mínimos turnos:**
```
Tu meta es completar la tarea en los MÍNIMOS turnos posibles.
Fíate del resultado de tus herramientas; no releas ni re-verifiques lo ya comprobado.
```

**Bloque PATH (kbd) — el que arregló las rutas (0 path-fails):**
```
# CRITICAL — FILE PATH
- ALWAYS use the bare relative name: `typing-game.html`
- NEVER use `/home/...`, NEVER absolute paths, NEVER look in other dirs.
- Ignore any other typing-game.html elsewhere on the system.
```
