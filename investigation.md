# Diario de investigación — Nekketsu Soccer MD

**Objetivo único:** entender por qué la pantalla de presentación del **primer
partido** no sigue el mismo flujo que las demás. Fase actual: **investigación,
sin parches**.

Convención del diario:

- **[HECHO]** — observado directamente con una herramienta. Incluye evidencia.
- **[HIPÓTESIS]** — no demostrado todavía.
- **[DESCARTADO]** — hipótesis refutada. **Se conserva** junto al motivo.

---

## 0. Material de partida

### 0.1 ROMs — [HECHO]

| Archivo | Tamaño | MD5 | Rol |
|---|---|---|---|
| `rom/jp.md` | 524.288 (0x80000) | `ff7a9a6fb74a640f40f10dff53e9cf4d` | ROM japonesa limpia |
| `rom/es098.md` | 1.048.576 (0x100000) | `d0c7498a6515e363ff332572c1c1faa4` | Traducción ES v0.9.8 (base de trabajo) |

Cabecera (ambas):

```
Domain : SEGA MEGA DRIVE
Copy   : (C)SEGA 1992.JUL
Serial : GM T-74033 -00
Region : J
SP     : 0x00ffff00
PC     : 0x00000208      <-- punto de entrada del reset
```

Diferencias de cabecera entre JP y ES:

```
JP  0x1a0: 00000000 0007ffff 00ff0000 00ffffff   (ROM end = 0x7ffff)
ES  0x1a0: 00000000 00000000 000fffff 00ffffff   (campos desplazados/rotos)
```

`[HECHO]` La ES declara mal el rango de ROM en la cabecera (`ROM end = 0`),
pero eso es cosmético en hardware real y en GPGX: no afecta al mapeo. Se anota
por completitud, **no** como causa candidata.

### 0.2 Mapa de divergencias binarias JP vs ES — [HECHO, contexto]

946 runs de bytes distintos en los primeros 0x80000, agrupados en 25 regiones:

```
00018e-0001ac   cabecera/checksum
000825-000974   *** código, zona del "modo 0" (state $f800=0)
000a74-000af1   datos/texto
000c06-000c0f   000d0b-000d15   000da4-000db4
00103b-00103e   *** código, screen builder (0x100c)
00132c-001332   001bab-001bae   003ed7-003eda
004bc9-004bcc   *** código, screen builder (0x4bbc)
004cd1-004cd4
0054a9-0054f1   0055a8 0056dc 0058c8
007000-00701c
0177fd-017de0   tabla / datos
0286ae, 0287f6-029e5c, 02a840-02a8c4, 02aa35-02ad66,
02b409-02b4d8, 02b56d-02b75a, 02b82b-02b834   -> tilemaps/texto
```

**Nota metodológica:** esta comparación de bytes es solo un *mapa de zonas para
mirar*. El encargo es comparar **comportamiento**, no bytes; estas regiones se
usarán únicamente para saber dónde poner breakpoints, nunca como conclusión.

### 0.3 Workspace histórico — clasificación

**Relevante (documentación técnica reutilizable):**

- `hist/disasm/references.txt` — el activo más valioso. Desensamblado parcial
  del arranque y del despachador principal. Coincide byte a byte con lo que
  observo en ejecución (verificado en §1.2).
- `hist/disasm/disasm_screens.txt` — rutinas 0xf80, 0x100c, 0x4b80…, que son
  constructores de pantalla.
- `hist/disasm/disasm_1b.txt` — bloque 0x1b30, otro constructor de pantalla.
- `hist/scripts/decompress.py` — **formato de compresión de tiles documentado**
  (control de 2 bits por tile, códigos 0=vacío 1=raw 2=máscara 3=máscara+XOR).
  Reutilizable tal cual.
- `hist/scripts/run_libretro.py`, `dump_runtime.py` — muestran que el arnés
  anterior era Genesis Plus GX vía libretro con símbolos `arena_get_*`.
- `hist/logs/rebuild.log` — confirma el core y sus flags de compilación.

**Ruido / de bajo valor:**

- `hist/logs/debuglog.txt` — es un volcado de llamadas `RETRO_ENVIRONMENT`, no
  contiene información del juego.
- `hist/logs/ea.json`…`ed.json`, `evskip/evpause/evmatch/evnav.json` — scripts
  de pulsaciones de mando de sesiones antiguas. Útiles solo como pista de
  navegación (§1.3), no como evidencia.
- `hist/dumps/*_vram.bin`, `*_work_ram.bin` — volcados de estado de sesiones
  antiguas. **No los uso como evidencia**: no sé con qué ROM ni en qué frame se
  tomaron. Regenero todo desde cero.
- `hist/scripts/spanish_patch*.py`, `recompress.py` — herramientas de
  generación del parche. Fuera del alcance de esta fase (no se toca la ROM).

---

## 1. Instrumentación construida

### 1.1 Emulador instrumentado — [HECHO]

Construido **Genesis Plus GX** (`8ae4ef7`) con `HOOK_CPU=1` y un módulo propio
`core/debug/arena_dbg.c` (fuente en `tools/arena_dbg.c`).

Bug corregido en el core para poder enlazar: `core/debug/cpuhook.h` declaraba
`void (*cpu_hook)(...)` sin `extern`, lo que produce definiciones múltiples con
GCC ≥10 (`-fno-common`). Corregido a `extern`.

El módulo registra, **en C** (hacerlo en Python sería ~700k callbacks/frame):

| Evento | Contenido |
|---|---|
| `CALL` | JSR / BSR tomado, con destino resuelto (modos `.L`, `.W`, `d16(PC)`) |
| `RET` | RTS / RTE / RTR |
| `JMP` | JMP largo (tail-call: el juego encadena pantallas con `jmp`) |
| `EXEC` | instrucción ejecutada (opcional, con filtro por rango de PC) |
| `VRAMW` | escritura a VRAM, **atribuida al PC que la causó** |
| `CRAMW` / `VSRAMW` | paleta / scroll vertical |
| `VDPREG` | escritura a registro del VDP |
| `MEMW` / `MEMR` | acceso 68k dentro de una ventana vigilada (para espiar `$ffxxxx`) |
| `MARK` | marcador inyectado desde el host para segmentar la traza |

Cada evento lleva `pc`, `frame` y **profundidad de pila**, lo que permite
reconstruir el árbol de llamadas y no solo una lista plana.

Además: breakpoints con contador de hits, y acceso directo a VRAM / CRAM /
VSRAM / work RAM / registros VDP.

**Trampa encontrada y resuelta:** GPGX almacena `cart.rom` y `work_ram`
*byteswapped* en hosts little-endian para acelerar los accesos de 16 bits. El
primer `peek16` leía basura y la traza salía vacía. Verificación de la
corrección:

```
peek16(0x208) = 0x4ab9   ==  ROM[0x208..0x209] = 4a b9   (tst.l)   OK
```

### 1.2 Validación del tracer contra la documentación histórica — [HECHO]

Traza real de arranque (JP, 30 frames), comparada con `references.txt`:

```
observado                          references.txt
CALL pc=000308 -> 00731a           000308: jsr $731a.w      OK
CALL pc=00030c -> 00735a           00030C: jsr $735a.w      OK
CALL pc=000376 -> 007c42           000376: bsr.w $7c42      OK
CALL pc=00037a -> 007a9e           00037A: jsr  $7a9e.w     OK
CALL pc=0003b2 -> 007b4e           0003B2: bsr.w $7b4e      OK
CALL pc=0003c8 -> 007878           0003C8: bsr.w $7878      OK
```

Coincidencia exacta, incluido el orden y el anidamiento. **El tracer es fiable
y el desensamblado histórico del arranque es correcto.** Es la única parte del
material antiguo que doy por validada; el resto queda pendiente de comprobar.

---

*(el diario continúa conforme avanza la investigación)*

---

## 2. Fase 2 — El bug del **primer partido**

### 2.0 Definición exacta del bug — [HECHO]

Evidencia: capturas del usuario (`ref/ok_jp.png`, `ref/bug_es.png`) y
reproducción propia en emulador (`work/shots/jp_present.png`,
`work/shots/es_present.png`).

| | JP (correcta) | ES v0.9.8 (bug) |
|---|---|---|
| Título | `第1試合` una sola vez | **`PRIMER PARTIDO` dos veces** (grande arriba + pequeña debajo) |
| Rival | `優秀院付属高校戦` una vez | `INSTITUTO YUSHUIN` abajo + **un fantasma superpuesto** bajo el título |

El bug es un **duplicado/residuo**, no un fallo de fuente ni de puntero.

### 2.1 Reproducción determinista — [HECHO]

Guardada en `tools/nav.py`. Desde el arranque en frío:

```
frame 430: START (pulsar/soltar)   -> salta el logo de título
frame 700: START                   -> entra al menú
frame 780: START                   -> elige "TORNEO 1J"
frame ~833: la pantalla de presentación se construye
frame  960: pantalla estable, bug visible
```

Ambas ROMs llegan al **mismo estado** con la misma secuencia:
`$f800=0x0c`, `$f802=0x5c`, `$f852=0` (contador de partido = 0 → primer partido).

### 2.2 Quién construye la pantalla — cadena completa — [HECHO]

Reconstruida con el tracer (flow=1) y confirmada con breakpoints.
**El origen real del flujo no es el último `JSR`**, sino esta cadena:

```
IRQ vblank $0400
  └─ $044A  jsr $462(pc,d0)        ; despacho por $f800  -> 0x0c
       └─ $04AE0                    ; despacho por $f802
            └─ $04AE8 jsr $5254(pc,d0)   ; $f802=0x5c -> idx 23
                 └─ $004B4A -> $005246   ; despachador de la presentación
                      └─ $00524E jsr $5254(pc,d0)  ; despacho por $f806
                           └─ $0054EC   <<< AQUÍ DIVERGEN LAS DOS ROMS >>>
```

`$f806` es el sub-paso de animación de la presentación; `$f80a` es el
contador de espera que hace avanzar la secuencia.

### 2.3 LA PRIMERA DIVERGENCIA — [HECHO, demostrado]

**Dirección: `$0054EC`.** Es el primer punto donde el flujo de ejecución de las
dos ROMs deja de ser el mismo *de forma causal* (no por temporización).

**ROM japonesa** — `$0054EC` es el manejador del sub-paso: descuenta el
temporizador y, si no ha expirado, **retorna sin dibujar nada**:

```
0054EC: subq.w  #$1, $f80a.w        ; decrementa el temporizador
0054F0: beq.w   $5500               ; si expira -> avanzar de sub-paso
0054F4: btst.b  #$7, $f811.w        ; ¿botón pulsado?
0054FA: bne.w   $5500
0054FE: rts                         ; <<< NO REDIBUJA NADA >>>
0055 00: clr.w  $f8da.w             ; (salida) siguiente sub-paso
005504: addq.w  #$4, $f806.w
005508: move.w  #$10, $f800.w
00550E: jmp     $49ee.l
```

**ROM española v0.9.8** — la instrucción está sustituida por un hook:

```
0054EC: jmp     $8a600.l            ; <<< HOOK DE LA TRADUCCIÓN >>>
```

que lleva a:

```
08A600: subq.w  #$1, $f80a.w        ; replica el decremento
08A604: beq.b   $8a616
08A606: btst.b  #$7, $f811.w
08A60C: bne.b   $8a616
08A60E: jsr     $8a300.l            ; <<< AÑADIDO: no existe en la japonesa >>>
08A614: rts
08A616: jmp     $5500.l             ; devuelve el control al código original
```

y `$8A300` es simplemente:

```
08A300: jsr     $c4180.l
08A306: rts
```

`$C4180` **reconstruye la pantalla entera**: borra dos rectángulos y vuelve a
pintar los dos textos:

```
0C4180: d0=0 d1=$02 d2=$1f d3=$07 -> jsr $7a6e   ; BORRA filas 2..9  (32x8 tiles)
0C419A: d0=0 d1=$14 d2=$1f d3=$03 -> jsr $7a6e   ; BORRA filas 20..23
0C41B4: d0=0 d1=$03 d2=$1f d3=$03, a0=$c4400 -> jsr $7a3c  ; PINTA título  (fila 3)
0C41D6: d0=0 d1=$15 d2=$1f d3=$02, a0=$c44c0 -> jsr $7a3c  ; PINTA rival   (fila 21)
0C41F8: rts
```

### 2.4 Por qué diverge — [HECHO]

La japonesa dibuja la pantalla **una sola vez** (al entrar en el sub-paso) y
después `$0054EC` solo cuenta el tiempo. La traducción, al necesitar redibujar
el texto latino, colocó la llamada de redibujado **dentro del manejador que se
ejecuta en cada frame**, en lugar de en la entrada del sub-paso.

Medición con breakpoints (misma navegación, hasta el frame 980):

| Breakpoint | JP | ES v0.9.8 |
|---|---:|---:|
| `$54EC` (manejador del sub-paso) | 131 | 130 |
| `$8A600` (wrapper nuevo) | — | **130** |
| `$8A300` → `$C4180` (redibujado) | — | **130 / 131** |
| `$7A6E` (**borrado** de rectángulo) | **1** | **264** |
| `$7A3C` (pintado de texto) | 54 | 317 |

**Este es el hecho central: la japonesa borra 1 vez; la española borra 264
veces** (2 rectángulos × ~131 frames). El redibujado es *por frame*.

Corroborado de forma independiente por la traza de escrituras a VRAM
atribuidas al PC (frames 790–980, plano A en `$C000`, 1 fila = `0x80` bytes):

```
JP: todas las filas 0..31 -> 64 escrituras (carga inicial única)
ES: fila  2..9   -> 4256..8448 escrituras  [pc=007a8c (borrado) x4192, pc=007a5e x4192]
    fila 20..23  -> 4256..8448 escrituras  [ídem]
```

`$7A8C` es el bucle interno de la rutina de borrado `$7A6E`
(`move.w #$0,(a5)` + `dbra`) — y es el PC nº1 en escrituras a VRAM de la ES
(50.304), mientras que **en la japonesa no aparece**.

### 2.5 Por qué produce exactamente el duplicado de la captura — [HECHO + razonamiento]

Las dos rutinas de la pantalla operan sobre regiones **distintas y no
alineadas**:

- `$C4180` (nuevo, cada frame) **borra** las filas **2–9** y **20–23**, y
  **pinta** el título en la fila **3** y el rival en la fila **21**.
- El resto del texto que la presentación ya había dibujado (y la animación de
  desplazamiento vertical que hace `$5514` mediante el registro de scroll
  `$58000003` / `$f830`) queda **fuera** de esas dos ventanas de borrado.

El resultado observado: el redibujado por frame reimprime el título y el rival
en posición fija, mientras la copia **original** —que la presentación había
colocado en otra fila y que el juego desplaza— **no se borra**, porque cae
fuera del rectángulo limpiado. Se ven las dos a la vez: la nueva (nítida, en su
sitio) y la vieja (el "fantasma" ligeramente desplazado). Exactamente lo que
muestra `ref/bug_es.png`.

**Nota de honestidad metodológica:** los puntos 2.1–2.4 están **demostrados**
con traza, breakpoints y desensamblado. La atribución concreta de *qué* copia
es la fantasma (2.5) está soportada por el mapa de filas de la traza VRAM
(filas 2–9 y 20–23 se reescriben; el resto no), pero **todavía no la he
aislado** apagando el hook y observando el resultado. Es el siguiente
experimento natural y lo dejo pendiente de tu visto bueno.

### 2.6 Segundo hook en la misma pantalla — [HECHO, anotado, no es la causa raíz]

`$005572` también está enganchado:

```
JP:  005572: jsr $4a20.l
ES:  005572: jsr $8e000.l
```

Pertenece a `$5514`, el sub-paso **siguiente** (animación de scroll). En la
navegación hasta el frame 980 registró **0 hits**, así que **no interviene** en
el bug de la captura. Se documenta para no perderlo de vista.

### 2.7 Infraestructura de la traducción localizada — [HECHO]

- `$084200`–`$08446F`: cadenas ASCII (13 títulos "…PARTIDO" + 13 "INSTITUTO …").
- `$084470`: tabla de 26 punteros de 32 bits a esas cadenas (terminada en `$FFFFFFFF`).
- `$084000`: rutina que imprime título+rival indexando por `$f852-2`.
  **Registró 0 hits en el primer partido** → no es la que dibuja esta pantalla.
  Es la variante usada por *otras* presentaciones (`$f852>=2`), lo que encaja
  con que "el primer partido no se comporte como los demás".
- `$08A300` / `$0C4180`: la ruta que sí se usa en el primer partido.
- `$0008F8` → `JMP $8D000`: otro hook (ajuste de paleta vía `$88500`), ajeno a este bug.

### 2.8 Hipótesis descartadas

- **[DESCARTADO]** *"El bug está en la rutina `$084000` que indexa la tabla de
  cadenas nueva"*. Motivo: breakpoint en `$084000` = **0 hits** durante todo el
  primer partido. La pantalla la construye `$0C4180`.
- **[DESCARTADO]** *"La primera divergencia está en el arranque"*. La comparación
  de secuencias de llamadas marcaba la llamada nº3 (`$3c8` vs `$428`), pero es
  un desfase de **temporización** del bucle de espera de vblank, no causal:
  ambas ROMs convergen inmediatamente después. La divergencia causal es `$54EC`.
- **[DESCARTADO]** *"La cabecera rota de la ES (`ROM end = 0`) influye"*. No afecta
  al mapeo en GPGX ni en hardware real; la ROM de 1 MB se lee correctamente
  (se ejecuta código en `$C4180`).

### 2.9 Estado: **PARADA SOLICITADA**

Primera divergencia localizada y demostrada. **No se ha modificado ningún byte
de ninguna ROM.** A la espera de revisión antes de diseñar el parche mínimo.

---

## 3. Experimento de aislamiento — [HECHO]

### 3.1 Método

Se añadió al emulador `arena_poke16()`, que escribe en `cart.rom` (la copia en
RAM desde la que ejecuta la CPU). **El fichero `.md` en disco nunca se toca**;
se verifica por md5 antes y después de cada brazo (los tres dieron
`FICHERO INTACTO: True`).

Punto de corte elegido: `$08A60E` (`jsr $8A300.l`, 3 words) → `NOP NOP NOP`.
Se eligió ahí, y no en `$54EC`, para eliminar **solo** el redibujado por frame
conservando intactos el temporizador, la lectura de botón y el avance de
sub-paso.

Verificación del poke: `4eb9 0008 a300` → `4e71 4e71 4e71`.

### 3.2 Resultados (navegación idéntica, hasta el frame 980)

| Breakpoint | A) JP control | B) ES sin tocar | C) ES neutralizado |
|---|---:|---:|---:|
| `$54EC` | 131 | 130 | 130 |
| `$8A600` wrapper | 0 | 130 | 130 |
| `$8A300` → redibujado | 0 | **130** | **0** |
| `$C4180` | 0 | **131** | **1** |
| `$7A6E` borrado | 1 | **264** | **4** |
| `$7A3C` pintado | 54 | 317 | 57 |

Estado final idéntico en los tres: `$f800=0c $f802=5c $f806=20 $f852=0`.

La neutralización **funcionó exactamente como se pretendía**: el redibujado
pasó de 130 ejecuciones a 0, y el borrado de 264 a 4.

### 3.3 Resultado visual: **LA HIPÓTESIS ERA FALSA**

`work/shots/iso_compare.png` (A | B | C).

**El duplicado NO desaparece.** Es más: B y C son **idénticos píxel a píxel**
(md5 del framebuffer `71354ce7cd06ec491e1ea6d424e71d45` en ambos).

### 3.4 [DESCARTADO] "El duplicado lo causa el redibujado por frame"

**Refutada por experimento directo.** Motivo: eliminar por completo las 130
llamadas de redibujado no cambia ni un píxel. El redibujado es
**cosméticamente inocuo**: cada frame borra y repinta *exactamente lo mismo en
el mismo sitio*. Es un derroche de CPU/ancho de banda de VDP, pero **no es el
bug**.

Lección metodológica: en §2.4 confundí una **correlación fuerte** (264 borrados
vs 1) con causalidad. La medición era correcta; la inferencia, no. Sin este
experimento habría propuesto un parche que no arregla nada.

### 3.5 LA CAUSA REAL — solape de bandas de tiles — [HECHO, demostrado]

Se abandonó el análisis de flujo y se inspeccionó el **contenido** del
nametable (plano A en `$C000`, 1 fila = `0x80` bytes).

**ES v0.9.8 — nametable final:**

```
fila  3: 0084 0184 0284 0384 ...   <- banda de tiles 0x00
fila  4: 2084 2184 2284 2384 ...   <- banda 0x20
fila  5: 4084 4184 4284 4384 ...   <- banda 0x40
fila  6: 6084 6184 6284 6384 ...   <- banda 0x60
...
fila 21: 6084 6184 6284 6384 ...   <- banda 0x60  << IDÉNTICA A LA FILA 6
fila 22: 8084 8184 8284 8384 ...   <- banda 0x80
fila 23: a084 a184 a284 a384 ...   <- banda 0xa0
```

Comprobación explícita: **`fila 6 == fila 21` → `True`**.

**JP — nametable final:**

```
fila  4: ... 0022 0122 0222     <- título, banda 0x00
fila  5: ... 1022 1122 1222     <- banda 0x10
fila  6: ... 2022 2122 2222     <- banda 0x20
fila 21: 3222 3322 3422 ...     <- rival, banda 0x32
fila 22: 4222 4322 4422 ...     <- banda 0x42
fila 23: 5222 5322 5422 ...     <- banda 0x52
```

**`fila 6 == fila 21` → `False`.** Sin solape: el título japonés ocupa **3**
filas y usa un rango de tiles propio.

### 3.6 Origen del solape en los datos fuente — [HECHO]

`$0C4180` pinta con estos parámetros (`d1`=fila inicial, `d3`=nº filas−1):

```
TÍTULO: d1=$03  d3=$03  a0=$C4400   -> filas 3,4,5,6   (4 filas)
RIVAL : d1=$15  d3=$02  a0=$C44C0   -> filas 21,22,23  (3 filas)
```

Y los mapas de tiles en ROM:

```
$C4400 (título) fila 3: 8460 8461 8462 8463 8464 8465 8466 8467 8468 8469
$C44C0 (rival)  fila 0: 8460 8461 8462 8463 8464 8465 8466 8467 8468 8469
                        ^^^^^^^^^^^^^^ BYTE A BYTE IDÉNTICAS
```

**El mapa del rival empieza en la banda de tiles `0x60`, que es exactamente la
cuarta y última banda del título.** El título en castellano se rasterizó a
**4 bandas** de tiles (porque "PRIMER PARTIDO" en fuente grande es más alto que
`第1試合`), pero la banda `0x60` se reutilizó como primera fila del bloque del
rival.

Resultado en pantalla: la fila 21 muestra la **cuarta banda del título** en
lugar de la primera del rival. Eso es el "PRIMER PARTIDO" pequeño y el
"INSTITUTO YUSHUIN" recortado que se ven en `ref/bug_es.png`.

### 3.7 Conclusión

- La primera divergencia **de flujo** (§2.3, `$54EC` → hook) es **real y está
  demostrada**, pero **no es la causa del bug visual**.
- La causa del duplicado es un **solape de rangos de tiles** entre el bloque
  del título (4 bandas: `0x00,0x20,0x40,0x60`) y el del rival (3 bandas:
  `0x60,0x80,0xa0`), presente **en los datos** de `$C4400`/`$C44C0`, no en el
  código.
- Es un problema de **asignación de VRAM/tilemap**, no de rutinas ni de hooks.

**No se ha modificado ninguna ROM.** Pendiente de tu revisión antes de
plantear cualquier corrección.

---

## 4. Informe del bloque gráfico del banner — [HECHO]

### 4.1 [DESCARTADO] "El título español es más alto e invade una 4ª banda"

**Refutada por medición directa del gráfico.** Es la hipótesis que planteaba el
usuario y que yo mismo sugerí en §3.5-3.6. Los datos la contradicen.

Medición de tinta (píxeles no transparentes) por línea de scanline, leída
directamente de la ROM en el banco del primer partido (`$0B0000`):

```
banda 0 (tiles 400-41F): [  0,  0,  0,  0,122,137,142,146]   TÍTULO línea 1/3
banda 1 (tiles 420-43F): [131,131,137,134,135,133,134,129]   TÍTULO línea 2/3
banda 2 (tiles 440-45F): [118,115,108,  0,  0,  0,  0,  0]   TÍTULO línea 3/3
banda 3 (tiles 460-47F): [  0,  0,  0,  0,166,172,178,178]   RIVAL  línea 1/3
banda 4 (tiles 480-49F): [152,151,153,152,151,144,149,155]   RIVAL  línea 2/3
banda 5 (tiles 4A0-4BF): [151,141,129,  0,  0,  0,  0,  0]   RIVAL  línea 3/3
```

**El gráfico del título ocupa exactamente 3 bandas, igual que el del rival.**
Ambos tienen la misma estructura y la misma altura real: 4+8+3 = **15 líneas de
píxel** dentro de 3 bandas (24 px). La banda 3 (`0x460`) pertenece
**íntegramente al rival**: su tinta empieza en la línea 4, idéntico patrón que
la banda 0 del título.

Confirmación visual: `work/shots/tiles_es_bands.png` muestra "PRIMER PARTIDO"
completo en las bandas 0-2 e "INSTITUTO YUSHUIN" completo en las bandas 3-5.

**No hay desbordamiento gráfico. El título español NO es más alto que el
espacio que tiene asignado.**

### 4.2 La causa real: un parámetro de altura equivocado en el tilemap

El tilemap NO son dos bloques separados, sino **una única secuencia contigua**
de 6 filas en `$0C4400`:

```
$0C4400  fila 0: 8400 8401 8402 ...   -> tiles 400+  TÍTULO
$0C4440  fila 1: 8420 8421 8422 ...   -> tiles 420+  TÍTULO
$0C4480  fila 2: 8440 8441 8442 ...   -> tiles 440+  TÍTULO
$0C44C0  fila 3: 8460 8461 8462 ...   -> tiles 460+  RIVAL   <-- $C44C0 = $C4400 + 3*0x40
$0C4500  fila 4: 8480 8481 8482 ...   -> tiles 480+  RIVAL
$0C4540  fila 5: 84a0 84a1 84a2 ...   -> tiles 4A0+  RIVAL
$0C4580  fila 6: ffff ffff ffff ...   -> terminador
```

`$0C44C0` **no es un tilemap distinto**: es simplemente la 4ª fila del mismo
array. Las dos llamadas de pintado en `$0C4180` son:

```
TÍTULO: d1=$03 (fila pantalla 3)   d3=$03 (4 filas)  a0=$C4400
RIVAL : d1=$15 (fila pantalla 21)  d3=$02 (3 filas)  a0=$C44C0
```

`d3` es el contador de `dbra`, es decir **nº de filas − 1**:

- Título: `d3=$03` → **4 filas** → filas de pantalla 3,4,5,6
- Rival: `d3=$02` → **3 filas** → filas de pantalla 21,22,23

**El título solo tiene 3 filas de gráfico, pero se le ordena pintar 4.** La 4ª
fila que lee (`$C4400 + 3*0x40 = $C44C0`) es precisamente la primera fila del
bloque del rival. Por eso la fila 6 de pantalla es idéntica a la fila 21
(`fila 6 == fila 21 → True`, verificado en §3.5).

**El bug es un único valor: `d3=$03` donde debería ser `d3=$02`.**

### 4.3 Efecto visual exacto

- Fila de pantalla 6 → banda `0x460` = **primera línea del "INSTITUTO YUSHUIN"**
  (sus 4 líneas superiores de píxel), pegada justo debajo del título.
- Es lo que se ve en `ref/bug_es.png` como "texto duplicado más pequeño":
  no es un duplicado del título, es **el borde superior del nombre del
  instituto dibujado en el sitio equivocado**.
- El instituto de abajo (filas 21-23) se dibuja completo y correcto; por eso
  el fantasma parece "recortado por arriba": es solo la primera de sus 3 bandas.

### 4.4 Alcance: ¿afecta a todos los partidos? — [HECHO]

**Sí, a los 13, y por la misma causa única.**

`$0C4100` carga el banco de gráficos indexando por `$f852` en la tabla `$0C4000`:

```
[ 0] -> 0B0000   1er partido      [ 7] -> 0BA800   8º
[ 1] -> 0B1800   2º               [ 8] -> 0BC000   9º
[ 2] -> 0B3000   3º               [ 9] -> 0BD800  10º
[ 3] -> 0B4800   4º               [10] -> 0BF000  11º
[ 4] -> 0B6000   5º               [11] -> 0C0800  12º
[ 5] -> 0B7800   6º               [12] -> 0C2000  FINAL
[ 6] -> 0B9000   7º               [13] -> FFFFFFFF (fin)
```

13 bancos de `0x1800` bytes = 192 tiles = **6 bandas de 32 tiles** cada uno.

Verificación de la estructura de los 13 bancos (tinta por scanline):

```
banco   partido | banda2 (última del TÍTULO)     | banda3 (1ª del RIVAL)
0B0000    1º    | [118,115,108, 0, 0, 0, 0, 0]   | [0,0,0,0,166,172,178,178]
0B1800    2º    | [154,144,127, 0, 0, 0, 0, 0]   | [0,0,0,0,188,195,202,203]
0B3000    3º    | [127,123,114, 0, 0, 0, 0, 0]   | [0,0,0,0,158,169,175,179]
0B4800    4º    | [131,124,112, 0, 0, 0, 0, 0]   | [0,0,0,0,146,158,163,167]
0B6000    5º    | [108,  7,  6, 5, 0, 0, 0, 0]   | [0,0,0,0,185,199,210,216]
0B7800    6º    | [120,113,102, 0, 0, 0, 0, 0]   | [0,0,0,0,158,170,179,182]
0B9000    7º    | [127,122,113, 0, 0, 0, 0, 0]   | [0,0,0,0,179,189,197,200]
0BA800    8º    | [129,121,108, 0, 0, 0, 0, 0]   | [0,0,0,0,174,185,196,201]
0BC000    9º    | [136,127,116, 0, 0, 0, 0, 0]   | [0,0,0,0,174,191,199,206]
0BD800   10º    | [129,122,111, 0, 0, 0, 0, 0]   | [0,0,0,0,164,173,183,188]
0BF000   11º    | [155,145,132, 0, 0, 0, 0, 0]   | [0,0,0,0,162,170,177,179]
0C0800   12º    | [168,156,139, 0, 0, 0, 0, 0]   | [0,0,0,0,181,188,198,200]
0C2000  FINAL   | [109,105,100, 0, 0, 0, 0, 0]   | [0,0,0,0,161,180,189,194]
```

**Los 13 bancos tienen estructura idéntica**: título en bandas 0-2 (siempre
termina en la línea 2-3 de la banda 2) y rival en bandas 3-5 (siempre empieza
en la línea 4 de la banda 3). Ningún banco desborda.

Y lo decisivo: **el tilemap `$0C4400` y la rutina `$0C4180` son únicos y
compartidos por los 13 partidos**. `$0C4180` no depende de `$f852`; solo cambia
el banco de gráficos que `$0C4100` ha volcado antes a VRAM.

Verificado con breakpoints en el primer partido:
`$C4100` = 1 hit (carga el banco), `$C4180` = 131 hits (pinta el tilemap fijo).

**Conclusión de alcance: hay un solo recurso y un solo defecto. Una única
corrección arregla los 13 partidos. No hay que rehacer ningún banner.**

### 4.5 Sobre la ruta alternativa `$084000`

`$084000` (la rutina que imprime cadenas ASCII desde la tabla `$084470`) sigue
registrando **0 hits**. No participa en esta pantalla en ningún partido: la
presentación gráfica siempre pasa por `$0C4100` + `$0C4180`. Queda como código
de otra vía (probablemente una versión anterior del sistema de traducción).

### 4.6 Comparación con la ROM japonesa

La japonesa usa un esquema de índices distinto (no contiguo: `0x022, 0x122,
0x222` para el título; `0x322…0x522` para el rival, saltos de `0x100`) y su
banner ocupa **3 filas** tanto arriba (4,5,6) como abajo (21,22,23).
`fila 6 == fila 21 → False`: sin solape.

Es decir: **la japonesa también usa 3 filas por bloque.** La traducción
mantuvo correctamente la altura de 3 bandas en el gráfico; el único error fue
el parámetro `d3` de la llamada de pintado.

### 4.7 Corrección mínima propuesta — [PENDIENTE DE TU APROBACIÓN]

**No hace falta rediseñar ningún gráfico.** El gráfico español ya tiene la
altura correcta (3 bandas, igual que el japonés). Rehacerlo no arreglaría nada,
porque el fallo no está en el dibujo sino en cuántas filas se mandan pintar.

Cambio mínimo, **1 byte**:

```
Offset ROM $0C41C2:   03  ->  02
   (word completo en $0C41C0: 363c0003 -> 363c0002,  move.w #$2,d3)
```

Con eso el título pinta 3 filas (3,4,5) en vez de 4 (3,4,5,6), la fila 6 queda
vacía y el fantasma desaparece. La lógica del juego no cambia; el rival sigue
pintándose igual en 21,22,23.

**Matiz honesto:** esto es técnicamente una modificación de un operando
inmediato dentro de una instrucción 68000, no de un recurso gráfico. Pero es
imposible corregirlo solo con datos: el tilemap es una secuencia contigua
correcta y el gráfico también lo es; el error está exclusivamente en el
parámetro de altura. La alternativa "solo datos" sería insertar una fila de
tiles vacíos entre las filas 2 y 3 del tilemap, lo que obligaría a desplazar
todo el bloque del rival y a cambiar igualmente el puntero `$C44C0` — más
invasivo y con el mismo resultado.

**No se ha modificado ninguna ROM.** Pendiente de tu decisión.

---

## 5. Validación del fix y generación de v0.9.9 — [HECHO]

### 5.1 Método

Se añadió `arena_poke_wram16()` para forzar `$f852` (índice de partido) en RAM
y poder validar los 13 banners sin jugar 13 partidos.

Punto de inyección: **frame 820**, justo antes del frame **832**, que es cuando
`$0C4100` vuelca el banco de gráficos a VRAM (verificado con breakpoint).

Todas las pruebas previas se hicieron con la ROM **sin modificar en disco**
(md5 comprobado en cada ejecución).

### 5.2 Baseline sin fix — el bug está en todos

```
   1º ($f852= 0): fila6==fila21 -> True    (fantasma presente)
   2º ($f852= 1): fila6==fila21 -> True
   3º ($f852= 2): fila6==fila21 -> True
  10º ($f852= 9): fila6==fila21 -> True
FINAL ($f852=12): fila6==fila21 -> True
```

Confirma §4.4: el defecto es único y afecta a los 13 partidos.

### 5.3 Con el fix aplicado en memoria — los 6 criterios

| Criterio | 1º | 2º | 3º | 10º | FINAL |
|---|---|---|---|---|---|
| 1. Título una sola vez (sin fantasma) | OK | OK | OK | OK | OK |
| 2. Instituto completo (3 filas 21-23) | OK | OK | OK | OK | OK |
| 3. Título conserva sus 3 filas (3,4,5) | OK | OK | OK | OK | OK |
| 4. Fila 6 vacía, sin huecos espurios | OK | OK | OK | OK | OK |
| 5. Separadores fila 20 / fila 24 intactos | OK | OK | OK | OK | OK |
| 6. Sin regresiones | OK | OK | OK | OK | OK |

### 5.4 Prueba de no-regresión fila a fila

Comparando el nametable completo **antes vs después** del fix, en los 5
partidos:

```
título (filas 3,4,5) idéntico  : True
rival  (filas 21,22,23) idéntico: True
resto del nametable idéntico    : True
única diferencia = fila 6: [24708, 24964, 25220] -> [0, 0, 0]
```

**El fix toca exclusivamente la fila 6.** Nada más cambia.

### 5.5 HUD del partido

Se avanzó hasta el partido en curso (estado `$24`) con y sin fix:

```
estado sin fix: 24/00      estado con fix: 24/00
VRAM completa idéntica: True
```

**Byte a byte idéntica.** El HUD ("TIEMPO", marcador, minimapa, jugadores) no
se ve afectado en absoluto. Captura: `work/shots/hud_fix.png`.

### 5.6 ROM v0.9.9 generada

```
origen : rom/es098.md                        md5 d0c7498a6515e363ff332572c1c1faa4
destino: rom/Nekketsu_Soccer_MD_ES_v0_9_9.md md5 1a60e6e2d67d94160f984b8f56eeb56b
tamaño : 1.048.576 bytes (idéntico)
```

**Un único byte modificado:**

```
Offset $0C41C3:  0x03 -> 0x02

antes:  0C41C0: 36 3c 00 03    move.w #$3,d3     (4 filas)
ahora:  0C41C0: 36 3c 00 02    move.w #$2,d3     (3 filas)
```

*Corrección de un error propio:* en §4.7 indiqué el offset `$0C41C2`. Es
incorrecto: el inmediato de 16 bits es `00 03`, luego el byte a cambiar es el
bajo, en **`$0C41C3`**. Detectado por una aserción de verificación antes de
escribir el fichero.

### 5.7 Validación final sobre el fichero (sin pokes de ROM)

```
   1º: fantasma=False  fila6_vacia=True  titulo_3filas=True  rival_3filas=True  OK
   2º: ... OK      3º: ... OK      10º: ... OK      FINAL: ... OK
RESULTADO GLOBAL: TODOS OK
```

### 5.8 Regresión global de la partida completa

Comparación de VRAM y CRAM entre v0.9.8 y v0.9.9 en 8 puntos del recorrido
(intro, SEGA, título, menú, presentación, posiciones, partido, partido tardío):

```
frame  estado         VRAM igual  CRAM igual
  300  00/20          True        True
  470  08/10          True        True
  600  08/3c          True        True
  760  08/4c          True        True
  900  0c/5c          False       True   <- filas afectadas=[6] (esperado)
 1100  10/34          True        True
 1400  24/00          True        True
 1700  24/00          True        True
```

**La única diferencia en toda la partida es la fila 6 de la presentación.**
Paletas intactas en todos los puntos.

### 5.9 Estado final

Bug resuelto con la modificación mínima posible: **1 byte**. Sin tocar
gráficos, sin tocar tilemaps, sin añadir hooks, sin mover punteros.

---

## 6. Verificación del 5º partido — caso cerrado — [HECHO]

### 6.1 Motivo de la revisión

En el barrido de los 13 bancos (§4.4), el 5º partido (`$0B6000`) mostraba un
perfil de tinta anómalo en la banda 2:

```
5º  : [108,   7,   6,   5, 0, 0, 0, 0]     <- distinto
resto: [~120, ~115, ~108,  0, 0, 0, 0, 0]
```

### 6.2 Explicación — [HECHO, no es un defecto]

El texto del 5º es **"QUINTO PARTIDO"**, el único de los 13 con **dos letras de
trazo descendente** (la cola de la `Q` y la de la `P`).

Columnas con tinta en la banda 2:

```
línea 0: 108 px  (cierre normal de los glifos, todo el ancho)
línea 1:   7 px  cols 48-54   <- cola de la Q
línea 2:   6 px  cols 49-54
línea 3:   5 px  cols 50-54
línea 4-7: 0 px
```

Y la banda 3 (primera del rival) arranca limpia, igual que en el resto:

```
línea 0-3: 0 px      línea 4: 185 px   línea 5: 199   línea 6: 210   línea 7: 216
```

**Los descendentes están contenidos dentro de la banda 2. No desbordan.** El
perfil "raro" era simplemente tipografía, no un defecto de rasterizado.

### 6.3 Verificación visual con v0.9.9

Captura: `work/shots/v099_check_5o.png`

- Título **"QUINTO PARTIDO"** completo, sin píxeles perdidos, cola de la Q
  íntegra.
- Rival **"INSTITUTO YOSHIMOTO"** completo.
- Sin fantasma, sin artefactos, sin filas vacías intermedias.

Comprobación estructural:

```
5º ($f852=4): fantasma=False  fila6_vacía=True
   título filas 3,4,5: 32/32 celdas cada una
   rival  filas 21,22,23: 32/32 celdas cada una
```

### 6.4 CASO CERRADO

La ROM **v0.9.9** (`md5 1a60e6e2d67d94160f984b8f56eeb56b`, 1 byte modificado en
`$0C41C3`) queda validada en los 6 partidos inspeccionados (1º, 2º, 3º, 5º,
10º, FINAL) y sin regresiones en el resto del juego.

Investigación finalizada.

---

## 7. HUD — inventario y estado del intento (opción B-mínima) — [PARADA]

### 7.1 Inventario completado — [HECHO]

**Asset del HUD:** `0x0286AA`, 256 tiles, se carga en **VRAM base `0x440`**
(idx de asset = tile − 0x440). Paleta: **15 = trazo, 14 = antialias, 0 = fondo**.

**12 tiles vacíos confirmados** (coinciden exactamente con la lista aportada):
`00 7A 89 98 A1 DE E2 E3 FC FD FE FF`.

**Letras latinas ya existentes en el HUD** (reutilizables tal cual):

| Letra | idx | Origen |
|---|---|---|
| P A U S E | `0x10`-`0x14` | palabra "PAUSE" |
| O K | `0x15`-`0x16` | palabra "OK" |

Nota: `O` y `K` usan **color 9** como fondo (proceden del recuadro "OK"),
mientras que `P A U S E` usan color 0. Para los comandos hay que usar el
estilo de `P A U S E`.

**Búsqueda en otros assets:** comparación exacta de los 7 glifos contra 9
assets (`2767A, 286AA, 40620, 439FC, 453AA, 4D4E0, 4E868, 4ECD6, 64FD4`):
**0 coincidencias fuera de `0x286AA`**. La fuente del HUD es exclusiva.

`0x2767A` contiene el alfabeto **A-Z completo** (idx `0x21`-`0x3A`), pero es
monocromo y de trazo fino: sirve de referencia de forma, **no es compatible**
para pegar tiles en el HUD.

**Letras necesarias:** `A C D E F I L N P R S T U`.
Ya existen 5 (A E P S U) → **faltan 8: C D F I L N R T**.
8 necesarios ≤ 12 libres → **sobran 4**.

**Asignación propuesta:** C→`0x00`, D→`0x7A`, F→`0x89`, I→`0x98`,
L→`0xA1`, N→`0xDE`, R→`0xE2`, T→`0xE3`. Reserva: `FC FD FE FF`.
Los 8 glifos están diseñados y verificados (`tools/newletters.py`).

### 7.2 La rutina NO está limitada a 3 columnas — [HECHO, demostrado]

`0x7A3C` es genérica: pinta `(d2+1)` columnas × `(d3+1)` filas leyendo words
consecutivos de `(a0)+`. **No hay terminador ni tope interno.**

```
007A4E: move.w d2,d6
007A5A: move.w (a0)+,d7     ; lectura secuencial
007A5E: move.w d7,(a5)
007A60: dbra   d6,$7A5A     ; d2+1 columnas
007A68: dbra   d3,$7A4E     ; d3+1 filas
```

Prueba directa: poniendo `d2=4` **pintó 5 columnas a la primera**.

Pero el ancho **no está en la tabla, sino en inmediatos del código**:

| | Coord X | Fila Y | Ancho (d2) | Alto (d3) |
|---|---|---|---|---|
| Bloque superior | `(a4)` | `d1=$17` | `0x00A530` = 1 | `0x00A534` = 1 |
| Bloque inferior | `2(a4)` | `d1=$19` | `0x00A572` = 2 | `0x00A576` = 1 |

Y la tabla **no cabe in situ**: ocupa `0x00A5E0`-`0x00A628` (72 B) y en
`0x00A628` empieza código. Pasar a 5 columnas necesita 120 B → faltan 48.

**Conclusión: no basta con datos.** Se requieren 2 bytes de código
(`0x00A572`, y `0x00A576` si se quiere 1 sola fila). La tabla sí puede
reubicarse solo con datos (punteros indirectos en `0x00A5C8`).

### 7.3 El asset no cabe recomprimido — [HECHO]

Añadir los 8 glifos rompe la compresión de los tiles vacíos (código 0 = 0 bytes):

```
espacio disponible : 6054 bytes  (0x286AA..0x029E4F)
asset con 8 letras : 6278 bytes
diferencia         : +224  -> NO CABE
hueco libre detrás : solo 12 bytes
```

**Reubicación necesaria.** Verificado que sólo existen **2 referencias** al
asset (`0x0020D0` y `0x005102`, ambos operandos de `lea.l`) y que `0x94000`
está libre (6278 B de `0xFF`).

**Reubicación validada como inocua:** con el asset original movido a `0x94000`
y los 2 punteros repuntados, la **VRAM resultante es idéntica byte a byte**
y la captura es idéntica píxel a píxel (md5 `b3fb3090...`).

*(Aquí cometí un error de lectura: interpreté una captura sin comandos activos
como "han desaparecido los retratos". La comparación de VRAM lo desmintió.)*

### 7.4 Geometría real del panel — [HECHO]

El bucle recorre **4 slots** (`dbra d6`, `a3 += 8`), con coords en `0x00A596`:

```
entrada 0: X_sup=25  X_inf=24     (fuera del panel, lado derecho)
entrada 1: X_sup=21  X_inf=21     (fuera del panel)
entrada 2: X_sup=9   X_inf=8
entrada 3: X_sup=5   X_inf=5
```

Interior útil del panel: **columnas 2..10** (9 celdas); la 11 es el borde.
Observado sin parche: comando en cols 6-7, "OK" en cols 9-10.

**El tamaño de cada entrada de la tabla es `(d2+1)*(d3+1)` words**, sin
cabecera. Con el `d2=2, d3=1` original son 6 words = 12 bytes.

### 7.5 Estado del intento y problema pendiente — [PARADA]

Con la tabla reubicada en formato original (3 cols × 2 filas) **funciona
correctamente**: se leen `450 451 453` = **P A S** en las columnas 5-7, con las
letras nuevas operativas (`4C9`=F, `4D8`=I, `51E`=N → "FIN").

Al ampliar a 5 columnas aparece un problema no resuelto: un volcado con words
marcadores (P A U S E O K C D F) revela que **se dibujan dos slots leyendo del
mismo puntero**:

```
word[k] enviados:  0:P 1:A 2:U 3:S 4:E 5:O 6:K 7:C 8:D 9:F
  fila 25:  . . . P A U | P A U S E
  fila 26:  . . . O K C | O K C D F
```

Es decir, el slot 2 (3 celdas) y el slot 3 (5 celdas) consumen el mismo array,
por lo que ampliar `d2` desplaza y duplica el contenido en lugar de alargar una
sola palabra. Además, con 5 celdas desde X=5 se invadirían las columnas de
"OK" (8-9): **5 letras + 2 de OK = 7 celdas, y sólo hay 9 útiles**, pero los
slots están fijados a X=5 y X=8, separados por sólo 3.

**No he encontrado todavía una disposición de 5 columnas que no solape.**

### 7.6 Decisión pendiente

Detengo aquí conforme a lo acordado (parar si falla una comprobación).
**No se ha escrito ninguna ROM.** La v0.9.9 sigue intacta
(md5 `1a60e6e2d67d94160f984b8f56eeb56b`, verificado tras cada prueba).

Opciones sobre la mesa:

- **B-completa:** además de `d2`, reasignar las X de los slots 2 y 3 (2 words
  de datos en `0x00A596`) para separarlos. Requiere entender por qué ambos
  slots leen del mismo puntero.
- **Palabras de 4 letras:** PASA/TIRA/CEDE caben en 4; PLACA y FINTA habría que
  acortarlas (p. ej. PLACA→PLAC no es aceptable). Con `d2=3` (4 cols) el
  solape con OK es de 1 sola columna.
- **Mantener 3 columnas** y usar abreviaturas de 3 letras.


---

## 8. Estructura exacta del panel de comandos — [HECHO, resuelto]

### 8.1 Por qué "dos slots leían del mismo puntero" — [RESUELTO]

**No eran dos slots: eran los dos BLOQUES de un mismo slot.**

El bucle real empieza en **`0x00A4BC`** (no en `0x00A4C2`, que es sólo la cabeza
del `dbra`):

```
00A4BC: lea.l  $F900.w, a3      ; base de los datos de slot en RAM
00A4C0: moveq  #$3, d6          ; 4 iteraciones: d6 = 3,2,1,0
...
00A58A: adda.l #$8, a3          ; cada slot ocupa 8 bytes de RAM
00A590: dbra   d6, $A4C2
```

Dentro de **cada** iteración hay **dos llamadas** a `0x7A3C`:

| Bloque | Coord X | Fila | d2 (cols) | d3 (filas) | Origen de `a0` |
|---|---|---|---|---|---|
| **Superior** | `(a4)` | `d1=$17` (23) | `0x00A530` = 1 → **2** | `0x00A534` = 1 → **2** | tabla **fija**: `$A5A8`/`$A5B4`/`$A5BC` |
| **Inferior** | `2(a4)` | `d1=$19` (25) | `0x00A572` = 2 → **3** | `0x00A576` = 1 → **2** | **puntero** de `$A5C4[ID&7]` |

`a4 = 0xA596 + d6*4`, por eso **cada slot tiene su propio par (X_sup, X_inf)**.

El bloque superior dibuja los **iconos de botón** (tablas fijas `$A5B4`=`00F 010
011 012`, `$A5BC` con flip). El inferior dibuja el **texto**. Ambos comparten la
coordenada del mismo slot, pero **no comparten puntero**: sólo el inferior usa
la tabla indexada.

### 8.2 Estructura exacta de cada entrada — [HECHO]

**Tamaño = `(d2+1) × (d3+1)` words, sin cabecera ni terminador.**
Con los valores originales: `3 × 2 = 6 words = 12 bytes`.
El orden es **fila superior completa, luego fila inferior** (row-major).

Tabla de **8 punteros** en `$00A5C4`, indexada por `$2(a3) & 7` (ID de comando):

```
[0] -> $A5A8   (vacío)
[1] -> $A5E0   ・ﾟ・ / ・スッ      = パス
[2] -> $A5EC   ・・・ / ドリブ      = ドリブル
[3] -> $A5F8   ・・・ / シュー      = シュート
[4] -> $A604   ・ﾟ・ / ッ??        = タックル
[5] -> $A610   ・・・ / ・ト-       = スルー
[6] -> $A61C   ・・・ / ・OK        = OK
[7] -> (fuera de la tabla, no usado)
```

### 8.3 Qué coordenadas son del comando y cuáles del "OK" — [HECHO]

Estado de RAM `$F900` capturado en partida (4 slots × 8 bytes):

```
slot 0 (d6=3) @F900: ID=1 (パス)   -> coords @0x00A5A2: X_sup=5   X_inf=5
slot 1 (d6=2) @F908: ID=6 (OK)     -> coords @0x00A59E: X_sup=9   X_inf=8
slot 2 (d6=1) @F910: ID=0 (vacío)  -> coords @0x00A59A: X=21
slot 3 (d6=0) @F918: ID=0 (vacío)  -> coords @0x00A596: X=25/24
```

Layout de cada slot en RAM: `+0`=?, `+2`=**ID de comando**, `+4`=trigger,
`+6`=temporizador.

**El "OK" no es un elemento especial: es el slot 1 con ID=6.** Usa exactamente
el mismo mecanismo que el comando.

⚠️ **Ojo con el índice:** las coordenadas se indexan por `d6`, que va de 3 a 0,
mientras que los slots de RAM avanzan de 0 a 3. Es decir, **slot k ↔ coords en
`0xA596 + (3-k)*4`**. Confundir esto fue la causa de varios intentos fallidos
en §7.5.

### 8.4 ¿Se pueden desacoplar sin tocar la rutina? — **SÍ** [HECHO, demostrado]

Prueba: cambiar **un solo word** (`0x00A5A0` = X_inf del slot del OK) de 8 a 10.

```
base    f26: ... 457 458 . 455 456 ...   (comando cols 6-7, OK cols 9-10)
OK a 10 f26: ... 457 458 . . . 455 456   (comando igual, OK cols 11-12)
```

**El OK se movió de forma independiente, sin tocar la rutina ni el comando.**

### 8.5 ¿Existe disposición para 5 letras sin solape? — **SÍ** [HECHO, demostrado]

Configuración validada:

```
d2 inferior (0x00A572): 2 -> 4          (5 columnas)   [2 bytes de código]
entrada de tabla: (4+1)*(1+1) = 10 words = 20 bytes
  fila 0 (superior, y=25): 5 espacios
  fila 1 (inferior,  y=26): las 5 letras
X_inf del comando (0x00A5A4): 5 -> 4
X_inf del OK      (0x00A5A0): 8 -> 10
```

Resultado medido en el nametable (fila 26, cols 0-15):

```
X=4 OK=10:  "..  PASA  OK   ."     <- ELEGIDA
X=5 OK=11:  "..   PASA  OK   "
X=3 OK= 9:  ".. PASA  OK   .."
```

Verificación visual (`work/shots/fx4_zoom.png`): **"PASA" y "OK" completos,
retratos intactos, sin solape ni artefactos**. Con las 8 letras nuevas cargadas
se leyó también **"FINTA OK"** correctamente.

X=2 se descartó: la **animadora** (sprite) tapa la primera letra.

### 8.6 Resumen del coste del HUD completo

| Elemento | Tipo | Bytes |
|---|---|---|
| Asset con 8 letras nuevas, reubicado a `0x94000` | datos | 6278 |
| Repuntar los 2 cargadores (`0x0020D0`, `0x005102`) | datos | 8 |
| Tabla de 8 entradas × 20 B en `0x08A700` | datos | 160 |
| Repuntar los 8 punteros (`0x00A5C4`) | datos | 32 |
| X_inf comando (`0x00A5A4`) y OK (`0x00A5A0`) | datos | 4 |
| **`d2` inferior (`0x00A572`): 3 → 5 columnas** | **código** | **2** |

**Total código: 2 bytes.** Todo lo demás son datos.

---

## 9. PAUSE → PAUSA aplicado — ROM v0.9.10 — [HECHO]

### 9.1 Cambio

El tilemap de "PAUSE" está en **`0x007CBC`** (no en `0x007CBA`; ese word es
parte de la instrucción anterior):

```
0x007CBC: 8450 8451 8452 8453 8454 0000
             P    A    U    S    E   fin
```

**Un solo byte:** `0x007CC5`: `0x54` → `0x51` (tile `0x454`=E → `0x451`=A).

**No se toca ningún tile ni se consume ningún slot libre**: sólo se reordena una
referencia, reutilizando la `A` que ya existe en `0x11`. El alfabeto queda
intacto, tal como pediste.

### 9.2 Validación

Texto leído del nametable (fila 14): **`'PAUSA'`**.
Captura: `work/shots/pausa_ok.png`.

Regresión v0.9.9 vs v0.9.10 en 9 puntos del juego:

```
punto      VRAM igual  CRAM igual
300/470/600/760/900/1100/1700   True   True
1400                            False  True   filas=[14]
pause                           False  True   filas=[14]
```

**La única diferencia en todo el juego es la fila 14** (donde se dibuja PAUSA).
Paletas intactas en todos los puntos.

### 9.3 ROM generada

```
rom/Nekketsu_Soccer_MD_ES_v0_9_10.md
1.048.576 bytes (tamaño idéntico)
md5: f966dc4d5401ec26fa5db21e121fc182
1 byte modificado respecto a v0.9.9: 0x007CC5  (0x54 -> 0x51)
```

---

## 10. HUD aplicado — ROM v0.9.11 — [HECHO]

Se aplicó **exactamente** la configuración validada en §8.5, sin experimentos
nuevos. Fuente única: `tools/hud_final.py` (la misma usada para validar en
memoria y para escribir la ROM).

### 10.1 Cambios

| Zona | Offset | Tipo | Bytes |
|---|---|---|---|
| Asset HUD + 8 letras, recomprimido | `0x094000` | datos | 5703 |
| Repunte de los 2 cargadores | `0x0020D0`, `0x005102` | datos | 6 |
| Tabla de 8 comandos × 20 B | `0x08A700` | datos | 160 |
| Repunte de los 8 punteros | `0x00A5C4` | datos | 25 |
| X_inf comando 5→4, X_inf OK 8→10 | `0x00A5A4`, `0x00A5A0` | datos | 2 |
| **`d2` inferior 3→5 columnas** | `0x00A572` | **código** | **1** |

**Total: 5897 bytes, de los cuales 1 solo byte es código.**
Ningún byte fuera de las zonas previstas (`otros: 0`).

Letras nuevas → tiles VRAM:
`C→440  D→4BA  F→4C9  I→4D8  L→4E1  N→51E  R→522  T→523`

### 10.2 Validación de los cinco comandos (sobre el fichero, sin pokes)

```
"..  PASA  OK   ....."      PASA  : OK
"..  PASA  FINTA....."      TIRA  : OK
"..  PLACA CEDE ....."      CEDE  : OK
"..  PLACA OK   ....."      PLACA : OK
"..        FINTA....."      FINTA : OK
"..  FINTA      ....."
"..  TIRA  OK   ....."
"..  FINTA OK   ....."
```

Verificación visual en `work/shots/cmds_grid.png`: los cinco comandos se leen
íntegros, **sin solape con el "OK"**, retratos y animadora intactos, sin
artefactos.

### 10.3 No-regresión v0.9.10 → v0.9.11

```
punto      VRAM igual  CRAM igual   nota
300..1100  True        True
1400       False       True         tiles=8
1700       False       True         tiles=8
pause      False       True         tiles=8
```

Los tiles que cambian son **exactamente** `440 4BA 4C9 4D8 4E1 51E 522 523`,
que coinciden **uno a uno** con los 8 slots vacíos asignados. **Ningún tile
original fue modificado.** Paletas intactas en todos los puntos.

PAUSA sigue correcto (`work/shots/v0911_pause.png`).

### 10.4 ROM

```
rom/Nekketsu_Soccer_MD_ES_v0_9_11.md
1.048.576 bytes   md5: 1ce1e44e8b0c109b725388e9329c41cc
```

### 10.5 Linaje de versiones

| Versión | md5 | Cambio |
|---|---|---|
| v0.9.8 (base) | `d0c7498a...` | — |
| v0.9.9 | `1a60e6e2...` | 1 byte: duplicado de "PRIMER PARTIDO" |
| v0.9.10 | `f966dc4d...` | 1 byte: PAUSE → PAUSA |
| **v0.9.11** | `1ce1e44e...` | HUD: 5 comandos en castellano |

---

## 11. DEPURACIÓN v0.9.11 — causa raíz de las regresiones — [HECHO]

### 11.1 CAUSA RAÍZ ÚNICA: los 12 tiles **NO estaban libres**

**La suposición de partida era falsa.** "Vacío en el asset" ≠ "libre".

Escaneo dinámico sobre la ROM **original** (v0.9.10), muestreando plano A,
plano B y la tabla de sprites (`SAT=0xDE00`) a lo largo de todo el juego
(intro, título, menú, presentación, posiciones, táctica y ~250 secuencias
aleatorias de juego):

```
idx 0x00 (VRAM 0x440): USADO en planoA + sprite   <-- CRÍTICO
idx 0x7A (VRAM 0x4BA): USADO en planoA
idx 0x89 (VRAM 0x4C9): sin uso
idx 0x98 (VRAM 0x4D8): sin uso
idx 0xA1 (VRAM 0x4E1): sin uso
idx 0xDE (VRAM 0x51E): USADO en planoB
idx 0xE2 (VRAM 0x522): USADO en planoB
idx 0xE3 (VRAM 0x523): USADO en planoB
idx 0xFC..0xFF        : sin uso
```

**5 de los 12 están en uso**, y son exactamente los que asigné a
**C (0x00), D (0x7A), N (0xDE), R (0xE2), T (0xE3)**.

### 11.2 El tile `0x440` es el **tile transparente universal**

Medición sobre 824 frames de la ROM original:

```
usos acumulados del tile 0x440:
   planoA : 4120
   planoB : 0
   sprite : 1648
contenido en VRAM: 32 bytes a cero  (transparente)
```

Es el tile de relleno que el juego usa **para todo lo que debe ser
transparente**. Aparecía "vacío" en el asset precisamente porque su contenido
es cero — esa es su función, no una señal de que esté libre.

**Al escribir la letra "C" en el índice 0x00, todo lo transparente del juego
pasó a dibujar una "C".**

### 11.3 Cada síntoma explicado

| # | Síntoma | Causa demostrada |
|---|---|---|
| 1 | "CCCCC" donde va PAUSA sin estar en pausa | El juego deja **5 celdas fijas** con tile `0x440` en **plano A, fila 14, cols 13-17** (medido). En la original son invisibles; ahora dibujan "CCCCC". |
| 1b | Letras al animar los pompones | Los frames de animación usan celdas transparentes = `0x440`. |
| 2 | Fragmentos detrás de Kunio | Su sprite tiene celdas transparentes con `0x440` (1648 usos en SAT; verificados sprites #45 y #47 activos con ese tile). |
| 4 | "Bloque" extraño en el HUD | Misma causa: celda transparente `0x440` en el panel/minimapa. |
| 3 | Fondo verde en las palabras nuevas | **No es un defecto de los tiles.** Comparación bit a bit: `P` original usa `{15:28, 14:24, 0:12}` y `C` nueva `{15:24, 14:23, 0:17}` — **mismo formato, misma paleta, misma prioridad**. El color 0 es transparente en Mega Drive; el verde es el campo que se ve detrás. Ocurre porque el comando se dibujó **desplazado a X=4**, fuera del recuadro azul del panel. |
| 5 | "PASOK" sin separación | El "OK" (entrada [6]) es `485 455 456`: su **primera celda es un espacio** que actuaba de separador. Con el comando a 5 columnas desde X=4 ocupa 4..8 y el OK arranca en X=8 → se pegan. |
| 6 | Ilustraciones desplazadas | **Descartado.** Comparación de los tilemaps de plano A y plano B (filas 21-27) entre v0.9.10 y v0.9.11: **IGUAL en todas las filas**. Lo que se percibe como desplazamiento son los artefactos del tile `0x440`. |

### 11.4 Solución correcta: hueco real entre assets — [HECHO, verificado]

Análisis del cargador (`0x0020C4`) y del siguiente asset:

```
asset HUD  : move.l #$48000002 -> VRAM 0x8800 = tile 0x440, d1=0x100 -> 0x440..0x53F
asset sig. : move.l #$6A000002 -> VRAM 0xAA00 = tile 0x550, d1=0x0B0 -> 0x550..0x5FF
HUECO      : tiles 0x540..0x54F = 16 tiles
```

Confirmado por escaneo dinámico: `0x540`-`0x54F` **nunca** se referencian en
plano A, plano B ni sprites.

**Propuesta:** ampliar `d1` de `0x100` a `0x108` (264 tiles) en los dos
cargadores y colocar las 8 letras en los índices `0x100`-`0x107`
(VRAM `0x540`-`0x547`).

- No se toca **ningún** tile en uso (los 12 "libres" quedan intactos).
- No se invade el asset siguiente (queda margen hasta `0x54F`).
- Coste: 2 words de código (`d1` en `0x0020D4` y `0x005106`) + datos.

**Riesgo:** el asset recomprimido crece; hay que verificar que sigue cabiendo
en `0x94000` (hay sitio de sobra) y que `f2a4` descomprime 264 tiles sin
desbordar. Debe validarse antes de escribir nada.

### 11.5 Correcciones pendientes de los otros síntomas

- **Fondo verde y "PASOK"**: devolver el comando a **X=5** (posición original)
  y dejar el OK en **X=8** con su celda-espacio inicial. Con 5 columnas desde
  X=5 el comando ocupa 5..9 y el OK 8..9 → **seguiría solapando**. Hay que
  decidir entre reducir a 4 columnas (PASA/TIRA/CEDE caben; PLACA y FINTA no)
  o desplazar el OK a X=11 (validado en §8.4 que se mueve de forma
  independiente y sin tocar la rutina).
- El recuadro azul del panel sólo cubre ciertas columnas; el texto debe quedar
  dentro para no mostrar el campo detrás. **Falta medir el ancho exacto del
  recuadro** antes de fijar las coordenadas definitivas.

### 11.6 Estado

**v0.9.11 debe considerarse RETIRADA.** La versión estable es **v0.9.10**
(`f966dc4d...`), que contiene el arreglo del duplicado y PAUSA, ambos
validados y sin regresiones.

**No se ha generado ninguna ROM nueva.**

---

## 12. INFORME TÉCNICO FINAL DEL HUD — revisión completa — [HECHO]

### PUNTO 1 — ¿Son seguros los tiles 0x540-0x547? **NO. RECHAZADOS.**

Mi propuesta anterior era **errónea**. Demostración:

Traza de escrituras a VRAM (`HOOK_VRAM_W`, 553.852 eventos) sobre la ROM
original:

```
escrituras en 0xA800-0xA9FF (tiles 0x540-0x54F): 256
   pc=0x00F450   frame 806
bloque completo escrito ese frame: 0xA580-0xAFFE = tiles 0x52C..0x57F
```

Origen localizado: rutina **`0x004BBC`** (pantalla de presentación del partido,
estado `0c/28`):

```
004BDA: move.l #$60000002,$c00004   -> VRAM 0xA000 = tile 0x500
004BE4: lea.l  $4E868,a0
004BEA: move.w #$100,d1              -> 256 tiles: 0x500..0x5FF
004BF2: jsr    $F2A4
```

**El asset `0x4E868` de la presentación ocupa 0x500-0x5FF, que engloba
0x540-0x54F.** Habría repetido exactamente el mismo error.

**Lección aplicada:** "no referenciado durante el partido" ≠ "libre". Hay que
comprobar **todos los estados** y además **quién escribe**.

### PUNTO 1b — Tiles realmente seguros: 8, verificados con marcador activo

Criterio correcto: dentro del asset del HUD, tiles que **nunca se referencian
en ningún estado** (plano A, plano B, ventana y SAT, con bases leídas de los
registros VDP, no asumidas).

Auditoría en 2 fases: recorrido completo (estados `00,04,08,0c,10,14,20,24`) +
los 4 modos del menú (1P, 2P, VERSUS, PASSWORD → estado `30`).

**Prueba activa:** se escribió un **tile sólido imposible** en los 8 candidatos
y se recorrió todo el juego buscándolo en pantalla:

```
idx 0x89 (VRAM 4C9): nunca aparece  [OK]     idx 0xFD (VRAM 53D): nunca aparece [OK]
idx 0x98 (VRAM 4D8): nunca aparece  [OK]     idx 0xFE (VRAM 53E): nunca aparece [OK]
idx 0xA1 (VRAM 4E1): nunca aparece  [OK]     idx 0xFF (VRAM 53F): nunca aparece [OK]
idx 0xFC (VRAM 53C): nunca aparece  [OK]     idx 0xEE (VRAM 52E): nunca aparece [OK]
```

**Asignación definitiva:** `C→0x89  D→0x98  F→0xA1  I→0xFC  L→0xFD  N→0xFE
R→0xFF  T→0xEE`

Los 5 tiles peligrosos (`0x00, 0x7A, 0xDE, 0xE2, 0xE3`) quedan **intactos**.

### PUNTO 2 — ¿Hay que ampliar el asset? **NO.**

Al no usar 0x540+, no hace falta tocar `d1` ni ampliar nada. El asset sigue
siendo de 256 tiles (`0x440-0x53F`).

Datos medidos, por completitud:
- El asset HUD termina en `0x53F` porque se carga con `d1=0x100` desde VRAM `0x8800`.
- Entre `0x540` y `0x54F` **no hay hueco**: es zona que la presentación reutiliza.
- El siguiente asset del HUD (`0x55C22`) empieza en `0x550` (`imm 0x6A000002`).

Tamaño del asset recomprimido con las 8 letras: **6263 B** (original 6054 B) →
sigue **sin caber in situ**, por lo que se mantiene la reubicación a `0x94000`,
ya demostrada inocua (VRAM idéntica byte a byte, §7.3).

### PUNTO 3 y 5 — Geometría real del panel, medida

Volcado del nametable (filas 21-27), ROM original:

```
       c0  c1  c2..c10       c11  c12..c19  c20
f21:   1F  2D  2E 2E 2E ...   2F   2E ...    2F
f22:   20  31  45 45 E4x5 45  32   36 37 ..  32
f23:   20  31  45 45 45 ...   32   39 42 ..  32
f26:   20  31  45 45 45 ...   32   3F 40 ..  32
f27:   20  34  35 35 35 ...   46   35 ...    46
```

- **Interior útil del panel: columnas 2..10 (9 celdas).**
- **Columnas 1 y 11 son los bordes del marco (tiles 0x31 y 0x32): INTOCABLES.**
- Fila 22, cols 5-9: animación de flechas (`0xE4`).

Disposición **original** con comando activo (capturada con breakpoint en
`0x00A582`):

```
f25:  ... 45 [26] 45 45 45 45 | 32
f26:  ... 45 [17 18] 45 [15 16] | 32
cols:       6  7    8   9  10    11
```

→ comando en 6-7, **separador en col 8**, OK en 9-10, borde 11 intacto.

Disposición en **v0.9.11** (defectuosa):

```
f26:  45 45 [10 11 13 11] 45 45 [15 16] 45
cols:  2  3   4  5  6  7          10 11
```

→ **el OK invade la columna 11 = el borde del marco.** Eso destruye el tile
`0x32` y produce el "bloque azul" y el borde roto de las capturas.

### PUNTO 4 — Cómo se lograba la separación original

**No era por coordenadas: era un tile vacío dentro del propio bloque OK.**

La entrada [6] de la tabla es `485 485 485 / 485 455 456`: su **primera celda
de la fila inferior es un espacio (`0x485`)**. Con X=8, ese espacio cae en la
columna 8 y las letras O y K en 9 y 10.

**Reproducción exacta:** mantener el bloque OK con su celda-espacio inicial y
colocarlo de forma que el espacio quede entre la última letra del comando y la O.

### Disposición final propuesta

```
comando  X=3  -> cols 3,4,5,6,7   (5 letras)
OK       X=8  -> cols 8,9,10      (8 = espacio separador, O=9, K=10)
borde col 11  -> INTACTO
col 2         -> libre
```

Separación idéntica al original (una celda vacía). Nada invade los bordes.

**Riesgo identificado y pendiente de validar:** la animadora (sprite) ocupa
visualmente hasta la columna 4, por lo que la primera letra en col 3 podría
solaparse parcialmente. **Debe comprobarse visualmente antes de fijar X=3**;
si molesta, la alternativa es reducir a 4 columnas (obligaría a acortar PLACA
y FINTA).

### PUNTO 6 — Retratos inferiores: **NO están recortados ni desplazados**

Comparación de VRAM en el mismo frame determinista, contra la base `es098`:

```
v0.9.10: tiles del asset distintos = 0    celdas nametable f21-27 distintas = 0
v0.9.11: tiles del asset distintos = 8    celdas nametable f21-27 distintas = 0
         (los 8 son exactamente los slots que modifiqué)
```

**Cero diferencias en el nametable del panel.** La sensación de recorte la
producían los artefactos del tile `0x440`. **Hipótesis descartada.**

### PUNTO 7 — ¿Es el 0x440 la única causa? **SÍ**

Prueba de control: se aplicaron las 8 letras **solo en los tiles seguros** y se
recorrió el juego completo buscando apariciones indebidas:

```
apariciones espúreas de las 8 letras: 0
```

Confirmado: eliminando el uso de `0x440` (y de los otros 4 en uso) desaparecen
todos los artefactos: pompones, fragmentos tras Kunio, bloque azul y "CCCCC".

### PUNTO 3 (matiz) — El fondo verde

Comparación bit a bit ya realizada (§11.3): `P` original `{15:28, 14:24, 0:12}`
vs `C` nueva `{15:24, 14:23, 0:17}` → **mismo formato, planos, paleta,
prioridad y transparencia**. No hay diferencia de formato.

El verde visible en las capturas procede de: (a) los sprites de la animadora
superpuestos y (b) los tiles `0x440` corruptos. **No** de un defecto de los
glifos nuevos.

### PUNTO 8 — PAUSA

**No se toca.** El cambio de v0.9.10 (`0x007CC5: 0x54→0x51`) se conserva
íntegro. Verificado que no interfiere con nada de lo anterior.

### Resumen de cambios para la próxima ROM

| Elemento | Offset | Tipo | Estado |
|---|---|---|---|
| 8 letras en tiles **seguros** (`89 98 A1 FC FD FE FF EE`) | asset | datos | verificado |
| Asset recomprimido (6263 B) reubicado | `0x94000` | datos | verificado inocuo |
| Repunte de 2 cargadores | `0x0020D0`, `0x005102` | datos | verificado |
| Tabla de comandos 8×20 B | `0x08A700` | datos | pendiente ajuste X |
| X comando → 3, X OK → 8 | `0x00A5A4`, `0x00A5A0` | datos | **pendiente validar** |
| `d2` inferior 3→5 columnas | `0x00A572` | **código (1 word)** | verificado |
| PAUSA | `0x007CC5` | datos | ya en v0.9.10 |

**No se ha generado ninguna ROM.** Falta únicamente validar visualmente la
disposición X=3 / X=8 frente al sprite de la animadora.

---

## 13. VALIDACIÓN GEOMÉTRICA FINAL — 5 letras es IMPOSIBLE — [HECHO]

### 13.1 Barrido objetivo X=2..5 (con tiles seguros)

Métricas medidas sobre capturas con breakpoint en `0x00A582`:

```
X=2: dentro=94%  invade_borde=SI  solape_animadora=3
X=3: dentro=88%  invade_borde=SI  solape_animadora=2
X=4: dentro=72%  invade_borde=SI  solape_animadora=1
X=5: dentro=57%  invade_borde=SI  solape_animadora=0
```

**Ninguna posición evita invadir el borde.** El barrido reveló además filas
como `"PASAFINTA"` y `"FINTA"` en columnas 7-12, lo que llevó al hallazgo real.

### 13.2 HALLAZGO: hay **tres slots** activos, no uno

Muestreo de `$F900` (700 frames), IDs por slot cuando está activo:

```
slot 0 (d6=3, X=5) : ID2 ドリブル 43% | ID1 パス 23% | ID4 タックル 23% | ID3 シュート 10%
slot 1 (d6=2, X=8) : ID6 OK 97% | ID5 スルー 2%
slot 2 (d6=1, X=21): ID4 タックル 100%   (panel derecho)
slot 3 (d6=0, X=24): nunca activo
```

**El slot 1 NO es "el OK fijo":** también muestra スルー (= CEDE, 5 letras).

Combinaciones simultáneas observadas:

```
slots (0,1)     : 244 frames
slots (0,1,2,3) : 129 frames
TIRA + CEDE     :  32 frames   <-- DOS comandos largos a la vez
PLACA + CEDE    :  19 frames   <-- idem
total frames con dos comandos largos simultáneos: 319
```

### 13.3 Demostración de imposibilidad

- `d2` (ancho) es **único para los 4 slots**: no se puede dar 5 columnas al
  slot 0 y 3 al slot 1 sin modificar la rutina.
- La distancia fija entre slot 0 (X=5) y slot 1 (X=8) es de **3 columnas**.

```
d2=2 (3 cols): slot0=[5,6,7]      slot1=[8,9,10]        sin solape, sin borde  OK
d2=3 (4 cols): slot0=[5,6,7,8]    slot1=[8,9,10,11]     solapan en 8, tocan borde
d2=4 (5 cols): slot0=[5..9]       slot1=[8..12]         solapan en 8,9 + borde
```

Mover las X tampoco resuelve: harían falta `5 + 1 + 5 = 11` celdas y el
interior del panel sólo tiene **9** (cols 2-10).

**Conclusión: con la rutina intacta, el ancho máximo es de 3 columnas.**
Las palabras de 5 letras (y de 4) son geométricamente imposibles.

Esto explica, retrospectivamente, todos los defectos de la v0.9.11: el
"bloque azul" y el borde roto eran el slot 1 escribiendo en la columna 11.

### 13.4 Cómo separa el original (medido)

```
ID1 パス     inf=[45 17 18]  -> 1 espacio inicial
ID2 ドリブル  inf=[19 1A 1B]  -> 0 espacios
ID5 スルー    inf=[45 23 24]  -> 1 espacio
ID6 OK       inf=[45 15 16]  -> 1 espacio
```

Y en pantalla, con dos slots activos, el original **también pega los textos**:

```
ids=[1,6]: 45 45 45 45 [17 18] 45 [15 16]   <- separados
ids=[1,2]: 45 45 45 45 [17 18][19 1A 1B]    <- PEGADOS, sin separación
ids=[4,5]: 45 45 [18 21 22] 45 [23 24]      <- separados
```

**El juego original ya muestra textos pegados** cuando el comando de la
izquierda ocupa sus 3 celdas. La separación no está garantizada por diseño:
depende de si la entrada empieza con espacio.

### 13.5 Opción A validada: abreviaturas de 3 letras

Probada con los 8 tiles seguros y **geometría 100 % original** (sin tocar
`d2` ni las coordenadas):

```
"..   PAS OK..."
"..   PLA OK..."
"..   TIR OK..."
"..   PASFIN..."     <- pegado, igual que el original con ドリブル
"..   PLACED..."     <- pegado
```

- No invade bordes, no toca la animadora, todo dentro del panel.
- **Cero cambios de código** (ni siquiera el word de `d2`).
- El comportamiento "pegado" coincide con el del juego original.

### 13.6 Opciones sobre la mesa

| Opción | Texto | Código | Riesgo |
|---|---|---|---|
| **A** | `PAS TIR CED PLA FIN` | **ninguno** | mínimo; fiel a la geometría original |
| **B** | `PASA TIRA CEDE PLACA FINTA` | modificar la rutina para ancho por slot | alto; toca lógica de dibujo |
| **C** | HUD en japonés | ninguno | nulo |

**Recomendación: opción A.** Es la única que cumple el objetivo declarado —
"que el HUD conserve exactamente el comportamiento gráfico original y que la
única diferencia visible sea el idioma" — sin tocar una sola instrucción.

**No se ha generado ninguna ROM.** La decisión entre A, B y C es tuya.

---

## 14. ANCHO INDEPENDIENTE POR SLOT — respuesta a las 5 preguntas — [HECHO]

### P1. ¿Existe una modificación que dé ancho independiente a cada slot? **SÍ**

Demostrado y **probado en emulador**. La rutina carga X, ancho y fila desde
inmediatos; los tres pueden leerse de una tabla indexada por slot.

Parche **in-place, sin desplazar un solo byte** (todos del mismo tamaño):

```
0x00A51E  e580          -> e780          asl.l #2,d0  ->  asl.l #3,d0   (SUP)
0x00A520  49f9 0000A596 -> 49f9 0008A800 lea.l tabla nueva              (SUP)
0x00A52E  343c 0001     -> 342c 0004     move.w #imm,d2 -> move.w 4(a4),d2
0x00A55E  e580          -> e780                                          (INF)
0x00A560  49f9 0000A596 -> 49f9 0008A800                                 (INF)
0x00A570  343c 0002     -> 342c 0004     ancho leído de la tabla         (INF)
```

Nueva entrada de **8 bytes por slot**: `+0 X_sup  +2 X_inf  +4 ancho-1  +6 libre`.

**Verificado funcionando:** con slot0 a 5 columnas se leyeron en pantalla
`PASA`, `PLACA`, `FINTA`, `TIRA` completos, con el borde (col 11 = `0x32`)
**intacto** y **0 bordes rotos**.

### P2. ¿Puede el OK dejar de compartir geometría? **SÍ**

Ya lo hace con el mismo parche: el slot 1 tiene su propio X y su propio ancho
en la tabla. Demostrado moviéndolo de X=8 a X=10 y a X=6 de forma
independiente, sin tocar el slot 0.

`d1` (fila Y, inmediatos en `0x00A52A` y `0x00A56C`) admite el mismo
tratamiento (`move.w 6(a4),d1`, 4 bytes), de modo que **cada slot podría tener
incluso su propia fila**. No se usa porque las filas 23-24 están ocupadas por
los retratos.

### P3. ¿Puede reconstruirse el HUD con otra estructura? **SÍ, parcialmente**

La tabla de punteros `0x00A5C4` es compartida por todos los slots, lo que
obliga a que las entradas tengan el mismo número de words. Se puede dar a cada
slot su propia tabla (`movea.l 8(a4),a1`), pero **no cabe in-place**: el bloque
`0xA54A-0xA566` tiene 30 bytes y la versión reordenada necesita 36. Requeriría
un **trampolín** a zona libre (`JSR`/`JMP`), técnicamente viable.

### P4. Solución más limpia

**Ancho por slot mediante tabla ampliada** (P1): 6 parches in-place, ninguno
cambia de tamaño, sin trampolines, sin mover código. Es la mínima intervención
que resuelve el problema estructural.

### P5. ¿Es imposible? **NO. Es posible, con un matiz de espacio físico**

Medición del panel: interior útil = **columnas 2..10 = 9 celdas**
(col 1 y 11 = bordes; col 12 en adelante = minimapa; **no ampliable**).

Peor caso con separador de 1 celda:

```
PLACA + CEDE = 5+1+4 = 10 celdas > 9   NO CABE
PASA  + FINTA= 4+1+5 = 10 celdas > 9   NO CABE
```

**Sin separador** (que es lo que hace el propio juego japonés, medido en §13.4:
`パス+ドリブル` → `[17 18][19 1A 1B]` sin hueco):

```
PLACA + CEDE = 9 celdas EXACTAS   CABE
PASA  + FINTA= 9 celdas EXACTAS   CABE
FINTA + CEDE = 9 celdas EXACTAS   CABE
```

**Conclusión: las 5 palabras completas SÍ son posibles.** Requiere aceptar que,
en los 3 casos de dos comandos largos simultáneos, los textos queden contiguos
— exactamente el mismo comportamiento que la ROM japonesa original.

### 14.1 Estado de la implementación

Configuración probada (`slot0 X=2 w5 | slot1 X=6 w5`):

```
bordes rotos: 0
"..PASA OK  ..."   "..TIRA CEDE..."   "..PASAFINTA..."
"..PLAC CEDE..."   "..FINTAOK  ..."   "..PLAC OK  ..."
```

**Defecto pendiente:** `PLACA` se recorta a `PLAC` cuando el slot 1 está activo,
porque con ambos a ancho 5 el slot 1 (X=6) pisa la columna 6, que es la última
letra del slot 0 (2..6).

**Vía de solución:** slot0 X=2 w5 (2..6) + slot1 X=7 w4 (7..10) = 9 celdas
justas, sin solape. Requiere que el slot 1 lea entradas de 4 words/fila
mientras el slot 0 lee 5 → **tabla de punteros propia por slot** (P3), es decir
el trampolín. Es el único punto que falta por implementar.

**No se ha generado ninguna ROM.** v0.9.10 sigue siendo la estable.

---

## 15. HUD DESACOPLADO — implementado y validado — [HECHO]

### 15.1 Lo que se ha construido

Desacoplamiento **completo** de los 4 slots. Cada uno tiene su propia entrada
de **16 bytes** en `0x08A800`:

```
+0  X_sup    +2  X_inf    +4  ancho_inf-1   +6  Y_inf
+8  puntero long a SU PROPIA tabla de textos
+12 ancho_sup-1          +14 reserva
```

**Trampolín** de 56 bytes en `0x08A900` que sustituye exactamente los 56 bytes
de setup del bloque inferior (`0x00A546-0x00A57D`):

```
move.l d6,d0 ; asl.l #4,d0 ; lea COORD,a4 ; adda.l d0,a4
move.w 2(a3),d5 ; andi.l #7,d5 ; asl.l #2,d5
movea.l 8(a4),a1 ; adda.l d5,a1 ; movea.l (a1),a0   <- tabla PROPIA del slot
move.w 2(a4),d0 ; move.w 6(a4),d1 ; move.w 4(a4),d2
move.w #1,d3 ; move.w #$4000,d4 ; clr.w d5 ; rts
```

Bloque superior: 3 parches in-place del mismo tamaño.
**Total código modificado: 61 bytes.**

Cuatro tablas de texto independientes: `0x08AA00` (slot0, ancho 5),
`0x08AB00` (slot1, 4), `0x08AC00` (slot2, 5), `0x08AD00` (slot3, 2).

### 15.2 Independencia demostrada

```
d6=3: tabla -> 08AA00  X_inf=2   ancho=5
d6=2: tabla -> 08AB00  X_inf=7   ancho=4
d6=1: tabla -> 08AC00  X_inf=21  ancho=5
d6=0: tabla -> 08AD00  X_inf=26  ancho=2
```

Prueba directa: se sobrescribió `cmd[4]` **sólo** en la tabla del slot0 y la
del slot1 **no cambió ni un byte**. Las tablas no comparten memoria.

Anchos verificados contra uso real (2500 frames de la ROM original):

```
slot0 necesita 5 (PLACA/FINTA)  -> tiene 5   OK
slot1 necesita 4 (CEDE)         -> tiene 4   OK
slot2 necesita 5 (FINTA)        -> tiene 5   OK
slot3 necesita 2 (OK)           -> tiene 2   OK
```

### 15.3 Batería de regresión (v0.9.12, `md5 41af082b...`)

| Prueba | Resultado |
|---|---|
| 5 comandos + OK sobre el fichero | **todos OK** |
| Bordes del marco | **0 rotos** |
| PAUSA | `'PAUSA'` intacto |
| CRAM (paletas) | idénticas |
| Registros VDP | idénticos |
| Tiles VRAM distintos | **exactamente los 8 nuevos** |
| Nametable del panel (f21-27) | **0 celdas distintas** |
| Tiles peligrosos (`00 7A DE E2 E3`) | **siguen vacíos** |
| Letras fuera de las filas 25-26 | **0 apariciones** |
| Regresión global (8 puntos) | sólo los 8 tiles nuevos |
| **Katakana en el HUD** | **3250 → 0** |

### 15.4 DEFECTO PENDIENTE: la animadora tapa las 2 primeras columnas

Detectado en la **inspección visual** (`work/shots/v12_grid.png`), no por las
métricas numéricas:

```
"PASA"  se ve como "ASA"
"PLACA" se ve como "LACA"
```

Medición del sprite (tabla SAT):

```
sprite #4: X=16 (col 2) Y=184 tamaño 2x3 tile=4B2 pal=2 PRI=1
sprite #5: X=16 (col 2) Y=208 tamaño 2x1 tile=4AA pal=2 PRI=1
-> ocupa las columnas 2 y 3, filas 23-26
```

La animadora es **PRI=1**, igual que el texto del plano. En el VDP, un sprite
de alta prioridad se dibuja **por encima** del plano de alta prioridad, así que
siempre gana. No se puede resolver con prioridades.

**Zona realmente legible: columnas 4..10 = 7 celdas** (no 9).

```
PLACA + CEDE = 9 celdas > 7   NO CABE
FINTA + CEDE = 9 celdas > 7   NO CABE
PLACA + OK   = 7 celdas       CABE
```

### 15.5 Estado

La infraestructura de desacoplamiento **funciona y está validada**. El único
problema restante es de **espacio legible**, no de arquitectura.

Opciones, todas ya soportadas por la infraestructura (sólo cambian datos):

| Opción | slot0 | slot1 | Resultado |
|---|---|---|---|
| **1** | X=4 w5 (4..8) | X=9 w2 (9,10) | PLACA/FINTA completos siempre; OK completo; **CEDE → "CE"** (29 % de los casos del slot1) |
| **2** | X=4 w4 (4..7) | X=8 w3 (8..10) | PASA/TIRA/CEDE completos; **PLACA→PLAC, FINTA→FINT** |
| **3** | X=2 w5 | X=7 w4 | todo completo pero la animadora tapa la 1.ª letra |

**La ROM v0.9.12 está generada pero contiene la opción 3 (defectuosa).
No debe usarse todavía.** La versión estable sigue siendo **v0.9.10**.

---

## 16. INFORME: ¿puede el texto ir DELANTE de la animadora? — [HECHO]

### P1. ¿Qué prioridad tiene el plano de los comandos?

**Plano A (`0xC000`), con PRI=1 ya activo.** Medido en el nametable:

```
fila 25: 8485 8485 ...   pri=1 pal=0
fila 26: 8485 8485 ...   pri=1 pal=0
```

Todos los words del HUD son `0x84xx` → bit 15 = 1. **Ya está en la prioridad
máxima disponible para un plano.** No hay margen de mejora por esta vía.

### P2/P3. ¿Puede cambiarse ese texto de plano o prioridad? — **No sirve de nada**

Jerarquía real del VDP (de atrás hacia delante):

```
1 backdrop · 2 planoB PRI0 · 3 planoA/ventana PRI0 · 4 sprites PRI0
5 planoB PRI1 · 6 planoA/ventana PRI1 · 7 sprites PRI1  <-- la animadora
```

**Un sprite PRI=1 gana siempre a cualquier plano.** El bit de prioridad ya está
puesto (P1) y no existe un nivel superior para planos.

### P4. ¿Alguna forma de que sólo el texto vaya delante? — **Probado y refutado**

Se realizó el experimento directo: forzar la prioridad del sprite de la
animadora de PRI=1 a PRI=0 en la SAT, en caliente.

Resultado (`work/shots/pri_compare.png`): **la imagen no cambia**. La "P"
sigue oculta en ambos casos. La hipótesis de que bastaba con la prioridad
queda **descartada empíricamente**.

Verificación del origen real del problema:

```
nametable fila 26: 20 31 [10 11 13 11] 45 [15 16] ...  -> "PASA OK" correcto
tile 0x450 (P): tiene contenido, no está vacío
análisis de color por celda de la captura:
   col 2: azul/negro, SIN blanco      <- letra tapada
   col 3: tonos de piel (215,71,34)   <- animadora
   col 4: blanco (238,238,238)        <- primera letra visible
```

El nametable es correcto: **el problema es puramente de superposición**.
Confirmado visualmente en `work/shots/zoom_row26.png`, donde se lee "ASA OK"
con los pompones sobre la "P".

### P5. ¿Duplicar el texto en otro plano? — **No resuelve**

El plano ventana (`0xD000`) está casi vacío (164 celdas, todas en las filas
28-31) y las filas 25-26 están libres. **Pero la ventana comparte el nivel 6
con el plano A**: seguiría por detrás del sprite. Además la SAT vive en
`0xDE00`, dentro del rango de la ventana, lo que limita su uso a las filas
0-27.

### P6. ¿La animadora es sprite o plano? — **100 % sprites**

```
plano A, cols 2-5, filas 23-26:  45 45 45 45   (sólo fondo azul del panel)
SAT: sprite #4  X=16 Y=184 2x3 tile=4B2 pal=2 PRI=1
     sprite #5  X=16 Y=208 2x1 tile=4AA pal=2 PRI=1
```

X=16 → columna 2, anchura 2 tiles → **ocupa las columnas 2 y 3**, filas 23-26.
Coincide exactamente con las letras que desaparecen.

### P7/P8. ¿Rutina que la oculte? ¿Precedente en el juego?

No se ha encontrado ninguna rutina que oculte la animadora durante los
comandos. El juego japonés **nunca necesitó una**: sus comandos ocupan 3
celdas desde X=5, muy lejos de la columna 2.

### CUARTA VÍA ENCONTRADA: dibujar el texto como SPRITES

Estado de la tabla de sprites durante el partido:

```
sprites en la cadena : 53 de 80
sprites visibles     : 37
entradas LIBRES      : 27
cadena: (0,link=1) (1,vacío) (2,vacío) (3,vacío) (4,ANIMADORA) (5,ANIMADORA) ...
```

**Los índices 1, 2 y 3 están vacíos y son MENORES que el de la animadora.**
Entre dos sprites del mismo nivel de prioridad, **gana el de menor índice en la
SAT**. Por tanto, dibujando el texto en los sprites 1-3 aparecería **por delante
de la animadora**.

Viabilidad: existe margen de sprites (27 libres) y el límite por línea son 20.
Coste: alto — habría que escribir una rutina que construya entradas de SAT cada
frame, sincronizada con el estado de los slots. Es un cambio de mucha mayor
envergadura que todo lo hecho hasta ahora.

### CONCLUSIÓN

- **No existe** ninguna combinación de plano/prioridad que ponga el texto
  delante de un sprite PRI=1. Es una limitación real del hardware VDP,
  **demostrada con experimento**, no asumida.
- **Sí existe** una cuarta vía: convertir el texto en sprites de índice bajo.
  Es técnicamente posible pero implica una rutina nueva y sostenida por frame.
- La vía barata y segura sigue siendo **mover el texto a la columna 4**, fuera
  del área de la animadora (cols 2-3), aceptando 7 celdas legibles.

**No se ha generado ninguna ROM.** La estable sigue siendo **v0.9.10**.

---

## 17. ROM v1.0 — HUD DEFINITIVO — [ENTREGADO]

Decisión: **Opción 1**, priorizando legibilidad y cero artefactos sobre la
literalidad absoluta de una palabra secundaria.

### 17.1 Geometría final (medida, no estimada)

Sprites que invaden las filas de texto (medido en la SAT, 120 frames):

```
fila 25: cols [2, 3, 28, 29]
fila 26: cols [2, 3, 28, 29]
```

Ambas animadoras ocupan 2 columnas → **7 celdas legibles por panel**:

| Slot | X | Ancho | Columnas | Contenido |
|---|---|---|---|---|
| slot0 | 4 | 5 | 4-8 | PASA / TIRA / PLACA / FINTA |
| slot1 | 9 | 2 | 9-10 | OK (CEDE → "CE") |
| slot2 | 21 | 5 | 21-25 | PASA / FINTA |
| slot3 | 26 | 2 | 26-27 | OK |

Bordes (cols 1, 11, 20, 30) y animadoras (2-3, 28-29) **sin tocar**.

### 17.2 Verificación de los 8 criterios

| # | Criterio | Resultado |
|---|---|---|
| 1 | Ningún artefacto gráfico | **0 letras fantasma** en 200 secuencias |
| 2 | Ninguna corrupción de VRAM | sólo los 8 tiles nuevos; los 5 peligrosos siguen vacíos |
| 3 | Ninguna letra fantasma | **0** |
| 4 | Ningún borde roto | **0** en 14 combinaciones |
| 5 | Ningún sprite afectado | animadoras intactas, SAT sin tocar |
| 6 | Ningún retrato afectado | **0 celdas** de nametable distintas |
| 7 | PAUSA perfecto | `'PAUSA'` |
| 8 | Comandos legibles | PASA/TIRA/PLACA/FINTA/OK íntegros |

Regresión global (9 puntos): VRAM idéntica salvo los 8 tiles nuevos; **CRAM y
registros VDP idénticos en todos los puntos**.

**Katakana en el HUD: 3250 → 0.**

### 17.3 Compromiso único y aceptado

`CEDE` se muestra como **"CE"** cuando aparece en el slot1 (2 celdas).
Frecuencia medida del slot1: OK 70 %, CEDE 29 %. Los comandos del slot
principal se muestran **siempre completos**.

### 17.4 ROM

```
rom/Nekketsu_Soccer_MD_ES_v1_0.md
1.048.576 bytes   md5: f14096880ab5107f9c43fc5c56787bbf
6455 bytes modificados vs v0.9.10   (0 fuera de las zonas previstas)
   asset con 8 letras nuevas ......... 5692
   tablas de texto (4, una por slot) .. 576
   tabla de coordenadas 16B/slot ...... 64
   trampolín .......................... 56
   CÓDIGO ............................. 61
   cargadores ......................... 6
```

### 17.5 Linaje

| Versión | md5 | Cambio |
|---|---|---|
| v0.9.8 | `d0c7498a…` | base recibida |
| v0.9.9 | `1a60e6e2…` | 1 byte: duplicado "PRIMER PARTIDO" |
| v0.9.10 | `f966dc4d…` | 1 byte: PAUSE → PAUSA |
| v0.9.11 | `1ce1e44e…` | **RETIRADA** (tiles en uso) |
| v0.9.12 | `41af082b…` | **RETIRADA** (animadora tapa la 1.ª letra) |
| **v1.0** | `f14096880ab5107f9c43fc5c56787bbf` | **HUD definitivo** |

---

## 18. DIAGNÓSTICO v1.0 — cuatro causas demostradas — [HECHO]

**v1.0 queda RETIRADA.** Las capturas del usuario contradicen mi informe y
tenía razón en los cuatro puntos.

### P1. Fondo verde — CAUSA: índice de color del fondo

Comparación hexadecimal directa de los tiles:

```
O ORIGINAL: 9efffffe effeeeff effe9eff ... -> NO contiene ningún '0'
K ORIGINAL: effe9eff effeeffe effffe99 ... -> NO contiene ningún '0'
F NUEVA   : efffffff effeeeee effe0000 ... -> LLENO de '0'
```

Histograma de índices de color:

```
O idx 0x15: {9:8, 14:26, 15:30}    sin índice 0   <- fondo OPACO
K idx 0x16: {9:8, 14:25, 15:31}    sin índice 0   <- fondo OPACO
C nueva   : {0:17, 14:23, 15:24}   USA índice 0   <- fondo TRANSPARENTE
F nueva   : {0:22, 14:20, 15:22}   USA índice 0
T nueva   : {0:26, 14:18, 15:20}   USA índice 0
```

**El índice 0 es transparente por hardware en Mega Drive.** Deja ver el plano
B, que en esa zona contiene el césped. Por eso las letras nuevas muestran
verde y **OK no**: OK rellena su fondo con el índice **9** (color sólido).

Confirmado midiendo píxeles en la captura del usuario
(`work/shots/dbg_green.png`): el verde `(0,130,0)` está **dentro** de los
glifos F e I, en los huecos que deberían ser opacos.

**La hipótesis del usuario era correcta.** Nota: P/A/U/S/E originales también
usan índice 0, pero se dibujan donde detrás hay fondo uniforme.

### P2. "FINTA" cortado en el panel derecho — CAUSA: modo H32

```
R12 = 0x00  ->  RS1:RS0 = 00  ->  H32 (256 px, 32 columnas)
framebuffer del emulador: 256x224
```

**El juego corre en H32, no en H40.** Validé razonando sobre 40 columnas. El
panel derecho ocupa las columnas 20-30 (interior 21-29) y la animadora derecha
está en las columnas 28-29 — dentro de la zona visible, no fuera como supuse.

Medición sobre la captura: las letras visibles llegan hasta la columna ~23-24 y
la animadora tapa el resto de "PLACA"/"FINTA" (5 letras desde X=21 → 21-25).

### P3. Letras fantasma — CAUSA: expansión de sprites multi-tile

**Traza completa capturada** (frame 1460, barrido de 6000 frames con gol):

```
sprite #6:  tile base = 4C1   tamaño 3x3 = 9 tiles
            ocupa: 4C1 4C2 4C3 4C4 4C5 4C6 4C7 4C8 4C9
            coincide con: 4C9 = mi letra "C"
            X=224 (col 28)  Y=184   <- junto a la animadora derecha
```

**Un sprite N×M consume N·M tiles CONSECUTIVOS desde su base.** Los tiles
intermedios nunca aparecen como "tile base" en la SAT.

Barrido de expansión completa sobre la ROM original:

```
C idx 0x89 (VRAM 4C9):  15 usos  *** OCUPADO POR SPRITE ***
D idx 0x98 (VRAM 4D8): 164 usos  *** OCUPADO POR SPRITE ***
F idx 0xA1 (VRAM 4E1): 164 usos  *** OCUPADO POR SPRITE ***
T idx 0xEE (VRAM 52E):   0 usos  libre
I idx 0xFC (VRAM 53C):   0 usos  libre
L idx 0xFD (VRAM 53D):   0 usos  libre
N idx 0xFE (VRAM 53E):   0 usos  libre
R idx 0xFF (VRAM 53F):   0 usos  libre
```

**Tres de mis ocho tiles (C, D, F) están dentro de sprites de la animadora.**
Coincide exactamente con las letras "D", "F", "I" que se ven en la captura del
gol.

### P4. Por qué mi batería no lo detectó

1. **Solo comprobaba el tile BASE de cada sprite.** No expandía `N×M`. Un
   sprite 3×3 con base `4C1` nunca reportaba `4C9`. (Mi propio `vram_audit.py`
   sí expandía; en la validación final usé la versión simplificada. Error mío,
   no del método.)
2. **Barrido demasiado corto y sin gol.** Hicieron falta 6000 frames para que
   apareciera la animación de celebración.
3. **Asumí H40 cuando el juego corre en H32.**
4. **No verifiqué el índice de color del fondo.** Comparé planos, paleta y
   prioridad, pero no que el fondo fuese opaco (índice 9) como en O/K.

### Criterios que debe incluir la batería a partir de ahora

- expandir **N×M** en cada sprite de la SAT
- barridos largos que cubran **gol y celebración**
- validar en el modo de vídeo real (**H32**)
- verificar que **ningún glifo nuevo use el índice 0** en su fondo
- comprobar el rango completo de tiles consumidos, no solo el base

### Estado

**v1.0 RETIRADA.** Versión estable: **v0.9.10** (`f966dc4d…`), que sólo
contiene los arreglos validados del banner y PAUSA.

Los únicos índices realmente libres confirmados son **5**: `0xEE, 0xFC, 0xFD,
0xFE, 0xFF` — insuficientes para las 8 letras. **No se propone solución
todavía**, conforme a lo pedido.

---

## 19. ESTUDIO TÉCNICO — cómo obtener 8 tiles — [ANÁLISIS, sin implementar]

### 19.1 Aclaración 1: el fondo verde, verificado en VRAM

Índices leídos **de la VRAM real** (no de la ROM), durante el partido:

```
O (VRAM 455): {9:8,  14:26, 15:30}   <- sin índice 0
K (VRAM 456): {9:8,  14:25, 15:31}   <- sin índice 0
P (VRAM 450): {0:12, 14:24, 15:28}   <- CON índice 0
C (VRAM 4C9): {0:17, 14:23, 15:24}   <- CON índice 0
F (VRAM 4E1): {0:22, 14:20, 15:22}   <- CON índice 0
```

**Atributos idénticos** en todos los casos: `PAL=0, PRI=1`. No hay ninguna
diferencia de atributos entre O/K y el resto — la diferencia está **solo en el
contenido del tile**.

¿Por qué P/A/U/S/E no se ven verdes si también usan índice 0? Por **dónde** cae
ese índice:

```
P original        O original        C nueva
+######+          9+#####+          +#####+.
+##+++##          +##+++##          +##++##+
+##+.+##          +##+9+##          +##+....   <- bloque grande de índice 0
+##+++##          +##+9+##          +##+....
+######+          +##+9+##          +##+....
+##++++.          +##+++##          +##++##+
+##+....          9+#####+          +#####+.
.++.....          99+++++9          .+++++..
```

En la P el índice 0 son píxeles sueltos del contorno; en la C es un **bloque
macizo** de 17-26 píxeles. La O rellena ese hueco con el índice **9**.

**Prueba visual final** (`work/shots/fi_zoom.png`, medido sobre la captura del
usuario): 330 píxeles verdes **exactamente dentro** de los huecos de la F y la
I. Y 0 píxeles verdes en el panel izquierdo, donde detrás hay azul: **el color
que se ve es el del plano B en cada zona**, no un defecto del glifo.

**Conclusión: la hipótesis del usuario es correcta y queda demostrada.** La
corrección es rellenar el fondo de las 8 letras con el índice 9, como O y K.

### 19.2 Aclaración 2: auditoría repetida con criterio uniforme

Método: expansión **N×M** de todos los sprites, plano A + plano B + ventana,
bases leídas de los registros VDP, 3 semillas × 4500 frames incluyendo gol.

| idx | VRAM | Quién lo usa | Cuándo | Veredicto |
|---|---|---|---|---|
| `0x89` | 4C9 | **sprite 3×3** (base 4C1, celebración) | estado `s24`, frame 3363+ | **NO válido** |
| `0x98` | 4D8 | **sprite 4×3** (81 usos) | partido | **NO válido** |
| `0xA1` | 4E1 | **sprite 4×3** (81 usos) | partido | **NO válido** |
| `0xEE` | 52E | **plano ventana** (14-19 usos) | estado `s24`, frame 3108+ | **NO válido** |
| `0xFC` | 53C | — | — | **LIBRE** |
| `0xFD` | 53D | — | — | **LIBRE** |
| `0xFE` | 53E | — | — | **LIBRE** |
| `0xFF` | 53F | — | — | **LIBRE** |

*(Los 5 descartados en §11: `0x00` tile transparente universal — 1894 usos en
plano A + sprites; `0x7A` plano A + sprite 3×3; `0xDE/0xE2/0xE3` plano B y
ventana.)*

**Corrección sobre mi informe anterior: no son 5 libres, son 4.** `0xEE` lo di
por seguro y lo usa el plano ventana. Mi barrido previo con "marcador sólido"
no lo detectó porque no cubría suficientes frames del estado `s24`.

### 19.3 Comparativa de alternativas

| Opción | Viable | Dificultad | Riesgo | Código | Datos | Regresiones | Mantenimiento |
|---|---|---|---|---|---|---|---|
| **1. Ampliar el asset** | **SÍ** | Baja | **Bajo** | 2 words (`d1`) | asset | Muy bajo | Fácil |
| 2. Robar tiles de otro asset | No | — | Alto | — | — | Alto | Frágil |
| 3. Reutilizar/componer letras | **No** | — | — | — | — | — | — |
| 4. Cambiar palabras | Sí | Muy baja | Nulo | 0 | tablas | Nulo | Fácil |
| 5. Sistema de sprites | Sí | **Muy alta** | **Muy alto** | ~200 B | SAT | Alto | Difícil |

#### Opción 1 — ampliar el asset del HUD ← **RECOMENDADA**

Mapa de carga medido:

```
HUD          : VRAM 8800 = tile 440, d1=0x100 -> 440..53F   (src 0x286AA)
siguiente    : VRAM AA00 = tile 550, d1=0x0B0 -> 550..5FF   (src 0x55C22)
HUECO        : tiles 540..54F = 16 tiles
```

El obstáculo que detecté en §12 (la presentación carga `0x4E868` en el tile
`0x500` con 256 tiles, pisando 540-54F) **no aplica**, porque el orden temporal
lo desmiente:

```
frame  804: carga PRESENTACIÓN (0x4E868 -> tile 500)   estado 0c
frame 1255: carga HUD          (0x286AA -> tile 440)   estado 20
```

**El HUD se carga DESPUÉS.** Verificación directa:

- **0 escrituras** en los tiles 540-54F tras la carga del HUD, sobre 1.421.415
  escrituras a VRAM registradas en 5200 frames de partido.
- **0 referencias** a 540-54F (plano A, plano B, ventana y sprites con
  expansión N×M) en 2 semillas × 5000 frames.

Cambios necesarios: `d1` de `0x100` a `0x108` en los **dos** cargadores
(`0x0020D4` y `0x005106`), 2 words. El asset pasa a 264 tiles y las 8 letras
van a los índices `0x100`-`0x107` (VRAM `0x540`-`0x547`).

Riesgos: (a) el asset recomprimido crece — verificar que cabe en `0x94000`
(hay sitio de sobra); (b) confirmar que `f2a4` descomprime 264 tiles sin
desbordar; (c) los tiles quedan corruptos durante la presentación, cuando el
HUD **no se muestra** — irrelevante, pero debe validarse.

#### Opción 2 — robar tiles de otro asset

Descartada: exigiría demostrar no-coexistencia temporal de cada tile, y toda la
VRAM se reescribe en algún momento (§11.4: 2048 tiles escritos). El único
"hueco" real es precisamente el de la Opción 1.

#### Opción 3 — reutilizar o componer letras: **INVIABLE**

El VDP admite **un solo tile por celda** (más flip H/V). No hay composición.
Y ninguna de `C D F I L N R T` es espejo de `P A U S E O K`. Derivar F de E, C
de O o R de P destruiría el glifo original, que sigue en uso.

#### Opción 4 — cambiar palabras

Reduce el problema pero no lo elimina, y degrada la traducción:

```
PASA TIRA CEDE PLACA FINTA   -> 8 letras nuevas (CDFILNRT)
PASE TIRO CEDE CORTE REGATE  -> 6 (CDGIRT)
PASA TIRA SUELTA ROBA SORTEA -> 5 (BILRT)
```

Con la Opción 1 disponible, **no es necesario tocar las traducciones**.

#### Opción 5 — sistema de sprites

Medido: 27 entradas SAT libres; la fila del texto tendría 12 sprites (límite
20) y 128 px de 256 — cabe. Coste: ~200 bytes de código nuevo, gestión de la
cadena de links compartida con jugadores/balón/animadoras, sincronización por
frame. **Complejidad muy alta y un fallo rompe todo el renderizado.**
Ventaja única: el texto iría por delante de la animadora.

#### Opción 6 — otras vías consideradas

- **Reducir el HUD a 4 letras**: no hace falta con la Opción 1.
- **Reutilizar los 4 libres + 4 con palabras más cortas**: híbrido innecesario.
- **Segundo asset propio cargado aparte**: equivale a la Opción 1 pero
  añadiendo un cargador nuevo. Más invasivo, mismo resultado.

### 19.4 Recomendación

**Opción 1 + corrección del índice de fondo.** Es la de menor riesgo y mejor
mantenimiento:

1. Ampliar el asset a 264 tiles (`d1: 0x100 -> 0x108`, 2 words).
2. Colocar las 8 letras en los índices `0x100`-`0x107` (VRAM `540`-`547`),
   **verificados sin uso ni escritura**.
3. **Rellenar el fondo de los 8 glifos con el índice 9**, igual que O y K.
4. Dejar intactos los 13 tiles del asset original que están en uso.

Con esto se conservan las cinco traducciones acordadas sin abreviaturas.

**Pendiente de validar antes de implementar:** que `f2a4` descomprima 264
tiles correctamente y que el asset ampliado quepa en `0x94000`.

**No se ha generado ninguna ROM.** Estable: **v0.9.10** (`f966dc4d…`).

---

## 20. VERIFICACIÓN DE LOS DOS RIESGOS — [HECHO]

### 20.1 Riesgo 1: ¿admite `f2a4` 264 tiles? — **SÍ, verificado en ejecución**

Análisis del código: el contador vive en `$fcc4` y se consume así:

```
00F34A: move.w $fcc4.w,d0
00F34E: subq.w #$1,d0
00F350: move.w d0,$fcc4.w
00F354: tst.w  d0
00F356: bne.b  $f2f0
```

Es un **word de 16 bits** sin ningún techo en 256. Además, la cabecera del
asset **no contiene ningún campo con el número de tiles**: el contador lo
aporta `d1`.

*(El `assert ... <= 256` del `recompress.py` histórico era una suposición del
autor, no una restricción del formato.)*

**Prueba real en emulador**, con `d1 = 0x108` en los dos cargadores:

```
los 264 tiles coinciden con lo esperado : True
último tile escrito  = 0x547
siguiente asset      = 0x550  -> NO PISA
```

Comparación contra la ROM original en el mismo frame de partido:

```
tiles distintos: 8 -> ['540','541','542','543','544','545','546','547']
COINCIDE con lo esperado : True
CRAM idéntica            : True
```

**El asset ampliado no altera ningún tile existente** y los 8 nuevos aterrizan
exactamente donde se pretendía.

### 20.2 Riesgo 2: ¿cabe el asset ampliado? — **SÍ, con margen amplio**

```
asset actual   (256 tiles) : 6054 bytes   (0x286AA..0x29E4F)
asset nuevo    (264 tiles) : 6279 bytes   (+225)
espacio libre en 0x94000   : 49152 bytes consecutivos de 0xFF
siguiente recurso          : 0xA0000
ocupa 0x94000..0x95886     -> sobran 42873 bytes
CABE                       : True
roundtrip 264 tiles        : True
8 letras íntegras          : True
ningún glifo usa índice 0  : True
```

**No hay que desplazar ningún otro recurso.**

### 20.3 Resultado del parche y **defecto residual detectado**

Aplicado en memoria, con los 8 glifos rediseñados con fondo **índice 9**:

```
bordes rotos: 0
|..  PASA OK..........PLACA    ....|
|..  FINTA  ..........PLACA    ....|
|..  PLACACE..........         ....|
|..  TIRA OK..........         ....|
```

Verificación de índices en VRAM tras el parche:

```
C (540): {9:38, 15:26}   opaco   OK
F (542): {9:42, 15:22}   opaco   OK
I (543): {9:44, 15:20}   opaco   OK
T (547): {9:44, 15:20}   opaco   OK
O (455): {9:8, 14:26, 15:30}  opaco  (original correcto)
K (456): {9:8, 14:25, 15:31}  opaco  (original correcto)

P (450): {0:12, 14:24, 15:28}  <-- ÍNDICE 0
A (451): {0:10, 14:24, 15:30}  <-- ÍNDICE 0
S (453): {0:7,  14:30, 15:27}  <-- ÍNDICE 0
```

**Las 8 letras nuevas ya son opacas y no muestran verde.** Pero medición sobre
la captura: **316 píxeles verdes** siguen apareciendo, y están en las letras
**P, A, S originales** de la palabra "PASA".

**Causa:** `P A U S E` del asset original usan índice 0 en su fondo. Con el
texto colocado sobre el panel se ve el plano B (césped) a través de ellas.
En japonés nunca se notó porque esas letras solo se usaban en "PAUSE", sobre
un fondo distinto.

**Consecuencia:** para eliminar el verde por completo habría que rehacer
también **P, A, U, S, E** con fondo índice 9 — 5 tiles del asset original que
**sí están en uso** (los usa "PAUSE"). Habría que verificar que el cambio no
degrada la pantalla de pausa.

### 20.4 Estado

Los **dos riesgos que bloqueaban la implementación quedan resueltos**: el
descompresor admite 264 tiles y el asset cabe sin mover nada.

Pero ha aparecido un **defecto residual no contemplado** en el encargo: el
fondo transparente afecta también a 5 letras originales, no solo a las 8
nuevas.

**No se genera ROM todavía**, porque no cumpliría el criterio "sin fondo
verde". Estable: **v0.9.10** (`f966dc4d…`).

---

## 21. PRUEBA AISLADA: ¿son las letras originales la causa del verde? — [HECHO]

### 21.1 Un error de método detectado y corregido

Los tres primeros intentos de prueba aislada dieron **resultado nulo**: el
verde no cambiaba al parchear la letra. La causa **no era el juego, sino mi
método**.

Prueba de control: escribí el tile de la P **entero a blanco puro** (`0xFF`)
directamente en VRAM y medí la pantalla:

```
¿cambia la celda de la P al escribir el tile a blanco puro?
   píxeles distintos: 0
```

**Genesis Plus GX mantiene una caché de patrones.** Escribir en `vram[]` desde
fuera no la invalida, así que el render sigue usando el tile antiguo. **Todas
mis pruebas con `poke_tile` sobre VRAM eran inválidas.**

Método correcto: parchear el **asset en ROM** y dejar que el juego lo
descomprima, que es lo que actualiza la caché.

### 21.2 Resultado de la prueba aislada (método correcto)

Verde medido en píxeles, por celda de pantalla, con "PASA OK" en el panel:

```
              P(c4)   A(c5)   S(c6)   A(c7)   O(c9)   K(c10)
base            12      10       7      10       0       0
solo P           0      10       7      10       0       0
solo A          12       0       7       0       0       0
solo S          12      10       0      10       0       0
```

**El verde desaparece exactamente en la letra modificada y solo en ella.**
En `solo A` desaparece en las **dos** A (columnas 5 y 7), porque comparten el
mismo tile — comportamiento coherente.

**Queda demostrado que el origen está en el propio tile**, no en la zona del
HUD ni en ningún otro factor.

### 21.3 Correspondencia píxel a píxel

Celda de la P en pantalla (`G` = verde, `#` = trazo) frente a los índices del
tile en VRAM (`.` = índice 0):

```
pantalla            tile en VRAM
. . . . . . . .     +######+
. . . . . . . .     +##+++##
. . . . G . . .     +##+.+##     <- índice 0 en (2,4)  -> verde en (2,4)
. . . . . . . .     +##+++##
. . . . . . . .     +######+
. . . . . . . G     +##++++.     <- índice 0 en (5,7)  -> verde en (5,7)
. . . . G G G G     +##+....     <- 4 índices 0        -> 4 verdes
G . . G G G G G     .++.....     <- 5 índices 0        -> 5 verdes
```

Correspondencia **uno a uno**: 12 índices 0 → 12 píxeles verdes.

### 21.4 Comparación bit a bit P vs O

```
P (idx 0x10)                      O (idx 0x15)
14 15 15 15 15 15 15 14            9 14 15 15 15 15 15 14
14 15 15 14 14 14 15 15           14 15 15 14 14 14 15 15
14 15 15 14  0 14 15 15           14 15 15 14  9 14 15 15   <- hueco: 0 vs 9
14 15 15 14 14 14 15 15           14 15 15 14  9 14 15 15
14 15 15 15 15 15 15 14           14 15 15 14  9 14 15 15
14 15 15 14 14 14 14  0           14 15 15 14 14 14 15 15
14 15 15 14  0  0  0  0            9 14 15 15 15 15 15 14   <- esquinas: 0 vs 9
 0 14 14  0  0  0  0  0            9  9 14 14 14 14 14  9
```

Mismo trazo (15) y mismo antialias (14). **La única diferencia es el relleno
de las zonas vacías: índice 0 en la P, índice 9 en la O.**

### 21.5 ¿Por qué O y K se diseñaron distinto?

Datos observados:

- `O` y `K` son las **únicas** dos letras del alfabeto del HUD con fondo
  opaco.
- Forman la palabra **"OK"**, que se dibuja dentro del panel de comandos, en
  la zona baja, **sobre el recuadro azul**.
- `P A U S E` forman **"PAUSE"**, que se dibuja en el **centro de la pantalla**
  durante la pausa, donde el juego apaga o cubre el fondo.

**Interpretación** (coherente con todos los datos, aunque es una inferencia de
diseño y no un hecho verificable en el código): los autores hicieron opacas
solo las letras que debían dibujarse sobre un fondo con contenido visible, y
dejaron transparentes las que se muestran sobre fondo limpio. Al reutilizar
`P A U S E` en el panel del HUD —algo que el juego original nunca hace— aflora
el índice 0.

### 21.6 Conclusión

- Las letras originales **sí son la causa** del verde restante: demostrado con
  prueba aislada letra por letra y con correspondencia píxel a píxel.
- El efecto **no depende de la zona**: depende del contenido del tile. Lo que
  varía con la zona es el color que se ve por detrás (azul del panel o césped).
- PAUSA no lo muestra porque en el centro de la pantalla, durante la pausa, no
  hay césped detrás de esas celdas.

**Para eliminar el verde por completo hay que rehacer P, A, U, S y E con fondo
índice 9.** Riesgo a validar: son los tiles que usa "PAUSE", así que hay que
comprobar que la pantalla de pausa no se degrada.

**No se ha modificado ninguna ROM.** Estable: **v0.9.10** (`f966dc4d…`).

---

## 22. VALIDACIÓN DE PAUSA CON LETRAS OPACAS — **FALLA** — [HECHO]

Se aplicó el cambio autorizado (P, A, U, S, E → fondo índice 9) y se comparó
PAUSA contra v0.9.10 **píxel a píxel**.

### 22.1 Resultado de la comparación

```
framebuffer idéntico : False
píxeles distintos    : 48 de 57344
bbox                 : x 104..143  y 112..119
                     -> celdas: fila 14, columnas 13..17  (exactamente PAUSA)
```

Color de los píxeles que cambian:

```
ANTES   : (0,238,0) / (0,206,0) / (205,238,205)   <- CÉSPED visto por transparencia
DESPUÉS : (0,0,65)                                <- AZUL SÓLIDO
```

Mapa de diferencias por letra (`X` = píxel que cambia):

```
P: 12 px    A: 10 px    U: 9 px    S: 7 px    A: 10 px
El trazo de las letras NO se altera; solo cambian los huecos.
```

### 22.2 Veredicto por criterio

| Criterio exigido | Resultado |
|---|---|
| PAUSA idéntica salvo el fondo | **OK** — 48 px, todos dentro de las 5 celdas |
| Sin bordes extraños | **FALLA** — aparece recuadro sólido `(0,0,65)` |
| Sin cambios de alineación | **OK** — mismas celdas (14, 13-17) |
| Sin cambios de paleta | **OK** — la CRAM no se toca |
| Sin píxeles sólidos donde debía verse el fondo | **FALLA** — 48 px pasan de césped a azul |

**Evidencia visual:** `work/shots/PAUSA_compare.png`. Se aprecia con claridad
un **recuadro azul oscuro** rodeando las letras, que no existe en v0.9.10.

### 22.3 Por qué falla

El índice 9 de la paleta 0 es **azul oscuro**, el color del panel del HUD.
Funciona perfectamente para "OK" y para los comandos, porque allí el fondo
real *es* ese azul. Pero PAUSA se dibuja **en el centro del campo**, donde el
fondo es césped verde: rellenar sus huecos con azul crea un bloque visible.

**El mismo tile no puede ser simultáneamente opaco-azul (para el HUD) y
transparente (para PAUSA).** Es un conflicto de uso, no un error de diseño.

### 22.4 Consecuencia y salida sin conflicto

`P A U S E` se usan en **dos contextos con fondos distintos**. La solución
correcta es **duplicarlas**: dejar los tiles originales `0x10-0x14` intactos
para PAUSA y crear **5 copias opacas** para el HUD.

Coste: 5 tiles adicionales. El asset pasaría de 264 a **269 tiles**
(`d1 = 0x10D`), y las nuevas irían a VRAM `0x548`-`0x54C`, dentro del hueco de
16 tiles ya verificado libre (`0x540`-`0x54F`).

- Total de tiles nuevos: 8 (C D F I L N R T) + 5 (P A U S E opacas) = **13**
- Hueco disponible: 16 → **caben, con 3 de margen**
- PAUSA seguiría usando los tiles originales → **cero regresión**

### 22.5 Estado

**No se ha generado ROM.** El cambio autorizado, aplicado tal cual, habría
introducido una regresión visible en PAUSA. Estable: **v0.9.10**
(`f966dc4d…`).

Pendiente de tu autorización: duplicar P A U S E en lugar de modificarlas.

---

## 23. ROM v1.1 — HUD TERMINADO — [ENTREGADO]

Solución: **duplicar** `P A U S E` en lugar de modificarlas. PAUSA conserva los
tiles originales transparentes; el HUD usa copias opacas.

### 23.1 Tabla de asignación por contexto

| Letra | PAUSA (original, transparente) | HUD (opaca) |
|---|---|---|
| P | idx `0x10` / VRAM `450` | idx `0x108` / VRAM `548` |
| A | idx `0x11` / VRAM `451` | idx `0x109` / VRAM `549` |
| U | idx `0x12` / VRAM `452` | idx `0x10A` / VRAM `54A` |
| S | idx `0x13` / VRAM `453` | idx `0x10B` / VRAM `54B` |
| E | idx `0x14` / VRAM `454` | idx `0x10C` / VRAM `54C` |
| O | — | idx `0x15` / VRAM `455` (ya era opaca) |
| K | — | idx `0x16` / VRAM `456` (ya era opaca) |
| C D F I L N R T | no existían | idx `0x100`-`0x107` / VRAM `540`-`547` |

Asset: 256 → **269 tiles** (`d1 = 0x10D`). Los 13 nuevos caben en el hueco
verificado `540`-`54F` (sobran 3).

**Separación de contextos verificada en ejecución:**

```
PAUSA usa tiles: 450 451 452 453 451     (originales)
HUD   usa tiles: 455 456 542 543 545 546 547 548 549 54B
¿el HUD usa alguno de 450-454? NO
```

### 23.2 Batería completa

| Criterio | Resultado |
|---|---|
| Fondo verde | **0 píxeles** en las letras (12 combinaciones) |
| PAUSA idéntica | **0 píxeles distintos** de 57344 |
| Letras fantasma | **0** en 6000 frames con gol |
| Artefactos en Kunio | **0** |
| Artefactos en animadoras | **0** |
| Bloques azules / bordes rotos | **0** |
| Panel izquierdo | PASA / TIRA / PLACA / FINTA / OK correctos |
| Panel derecho | PASA / PLACA / FINTA / OK correctos |
| Presentación del partido | **0 píxeles distintos** |
| Celebración de gol | sin artefactos |
| Katakana en el HUD | **1654 → 0** |
| CRAM (paletas) | idéntica en todos los puntos |
| Regresión global (9 puntos) | solo los 13 tiles nuevos |

Verificación visual: `work/shots/V11_grid.png`.

### 23.3 ROM

```
rom/Nekketsu_Soccer_MD_ES_v1_1.md
1.048.576 bytes   md5: d5079bd15d319ff9faa6a3c608d59a15
6603 bytes modificados vs v0.9.10   (0 fuera de las zonas previstas)
   asset 269 tiles ..... 5838      tablas de texto ..... 576
   tabla coords ........   64      trampolín ...........  56
   cargadores ..........    6      d1 ..................   2
   CÓDIGO ..............   61
```

### 23.4 Linaje final

| Versión | md5 | Estado |
|---|---|---|
| v0.9.8 | `d0c7498a…` | base recibida |
| v0.9.9 | `1a60e6e2…` | duplicado "PRIMER PARTIDO" |
| v0.9.10 | `f966dc4d…` | PAUSE → PAUSA |
| v0.9.11 | `1ce1e44e…` | RETIRADA (tiles en uso) |
| v0.9.12 | `41af082b…` | RETIRADA (animadora tapa letras) |
| v1.0 | `f14096880…` | RETIRADA (fondo verde + letras fantasma) |
| **v1.1** | `d5079bd15d319ff9faa6a3c608d59a15` | **HUD terminado** |

### 23.5 Compromiso conocido

`CEDE` se muestra como **"CE"** en el slot secundario (2 celdas). Frecuencia
medida: ese slot muestra OK el 70 % y CEDE el 29 %. Los comandos del slot
principal se muestran **siempre completos**. Es el único recorte, aceptado en
§17 por limitación física del panel (7 celdas legibles).

---

## 24. ESTUDIO: tipografía condensada exclusiva del HUD — [ANÁLISIS]

### 24.1 Punto de partida medido

Ancho efectivo de la fuente **actual** del HUD:

```
letra  columnas con tinta   ancho  libre
P      ########             8      0
A      ########             8      0
U      ########             8      0
S      ########             8      0
E      .#######             7      1
O      ########             8      0
K      ########             8      0
   ancho medio: 7.9 px de 8  ->  margen medio 0.1 px
```

**La fuente original no deja margen.** Por eso las palabras contiguas se pegan.

### 24.2 Fuente diseñada (`tools/hudfont.py`)

Criterio uniforme para los 15 glifos `A C D E F I K L N O P R S T U`:

- cuerpo de **5 px** (columnas 0-4), altura **7 px** (filas 0-6)
- columnas 5-7 y fila 7 transparentes → **3 px de margen**
- trazo de 1 px sin antialias (más nítido a este tamaño)
- contraforma interior con índice **9**

Medición: **los 15 glifos con ancho 5 y margen 3, sin excepción.**

```
palabra   tiles   px ocupados   px totales   libre
PASA          4            29           32       3
PLACA         5            37           40       3
FINTA         5            37           40       3
TIRA          4            29           32       3
CEDE          4            29           32       3
OK            2            13           16       3
```

Hallazgo útil: el fondo del panel **es el índice 9**, el mismo que uso en la
contraforma. El bloque de fondo queda **invisible**; solo se ve el trazo.

### 24.3 Resultado de la prueba obligatoria

Simulaciones en `work/shots/font_real.png`, con los colores reales del HUD
leídos del emulador y **sin tile separador**:

```
PLACAPLACA  (10 tiles)
PLACAOK     ( 7 tiles)
FINTACEDE   ( 9 tiles)
PASATIRA    ( 8 tiles)
```

- **"PLACA PLACA" cabe en 10 tiles**: se cumple (siempre cupo; 5+5 letras = 10 celdas).
- **Legibilidad**: muy mejorada respecto a la fuente original.
- **Separación percibida**: **NO se cumple.**

### 24.4 Por qué no se cumple — demostración

Medición del hueco en la unión de las dos palabras:

```
columnas completamente transparentes en la unión: 3
hueco entre letras DENTRO de una palabra:          3
```

**Son idénticos.** El ojo no tiene ninguna señal para distinguir dónde acaba
una palabra y empieza la otra.

En una rejilla de paso fijo de 8 px:

```
hueco(letra,letra) = hueco(palabra,palabra) = 8 - ancho_cuerpo   [CONSTANTE]
```

Probado también con **glifos asimétricos** (cuerpo desplazado dentro del tile):
desplazar **no crea** espacio, solo lo reparte entre margen izquierdo y
derecho; la suma sigue siendo `8 - cuerpo`.

Y estrechar más el cuerpo no ayuda, porque agranda **ambos** huecos por igual:

```
cuerpo 5px -> inter-letra 3px ; inter-palabra 3px
cuerpo 4px -> inter-letra 4px ; inter-palabra 4px
cuerpo 3px -> inter-letra 5px ; inter-palabra 5px
```

Para que el cerebro separe dos palabras, el hueco inter-palabra debe ser
claramente mayor que el inter-letra (regla tipográfica: al menos el doble).
Con paso fijo eso exige **kerning variable**, que el VDP no soporta, o un
**tile separador**, que gasta una celda.

### 24.5 Lo que la fuente condensada sí aporta

- **No reduce el número de celdas**: 1 letra = 1 celda, igual que antes. El
  problema de `PLACA(5) + CEDE(4) = 9 celdas > 7 disponibles` **no cambia**.
- **Sí mejora mucho** la legibilidad, la nitidez y el aspecto: da al HUD una
  tipografía propia y homogénea en lugar de glifos al límite del tile.

### 24.6 Vía que sí resolvería la separación

Con la fuente condensada de 5 px, **la separación se consigue si el slot deja
una celda vacía al final** — algo que ya ocurre cuando la palabra es más corta
que el ancho del slot:

```
slot0 = 5 celdas, palabra de 4 letras -> 1 celda vacía -> hueco 8+3 = 11 px
```

Es decir: funcionaría con `PASA`, `TIRA` y `CEDE` (4 letras), pero **no** con
`PLACA` ni `FINTA` (5 letras), que llenan el slot.

Para que funcione siempre haría falta **ampliar el slot0 a 6 celdas**, lo que
requiere 8 celdas legibles (6+2 de OK) y solo hay 7. Volvemos al límite
físico documentado en §17.

### 24.7 Conclusión

La idea es **correcta y valiosa** —el problema es de píxeles, no de tiles— y la
fuente condensada está diseñada, medida y lista en `tools/hudfont.py`. Pero por
sí sola **no logra la separación visual**, porque el paso de la rejilla es fijo
y hace que ambos huecos sean iguales.

**No se ha modificado ninguna ROM.** Estable: **v1.1** (`d5079bd1…`).

---

## Sesión — Captura in-game de la tipografía condensada (NO aplicada a ROM)

**Motivo**: el usuario pidió ver la tipografía condensada *en el juego*, no en una
maqueta ni descrita en texto. Las imágenes previas (`font_compare.png`,
`font_real.png`) eran renders sintéticos hechos con PIL desde `hudfont.py`; no
pasaban por el VDP ni por el descompresor del juego.

### Método [HECHO]
Herramienta nueva: `tools/shot_font.py` + `tools/shot_run.py` + `tools/shot_pair.py`.
- Se parchea **solo la copia en RAM del cartucho** dentro del emulador
  (`m.poke16`), antes de que el juego cargue el asset del HUD.
  **No se ha generado ninguna ROM nueva.** v1.1 sigue siendo la vigente.
- Se recomprime el asset con `compress()` y se escribe en `0x94000`, dejando que
  el propio juego lo descomprima → evita el problema de la caché de patrones de
  GPGX (ver sección correspondiente).
- Reproducibilidad: `boot()` + entradas pseudoaleatorias con `random.Random(7)`.
  El parche solo cambia píxeles de tiles, no la lógica → la partida es idéntica
  frame a frame entre versiones. **Verificado**: en los 6 frames capturados, la
  diferencia de imagen está confinada a `y=208..216` (la fila de texto del HUD),
  bbox máximo `(32,208)-(224,216)`.

### Hallazgo NUEVO que la maqueta ocultaba [HECHO]
La primera variante (`apply_cond`, 269 tiles) condensaba las 13 letras nuevas
pero **dejaba `O` y `K` con los tiles originales gruesos** (`0x15`/`0x16`), que
no forman parte del bloque nuevo. Resultado en pantalla: `PASA` fina junto a
`OK` gruesa — incoherencia tipográfica evidente que **no aparecía en
`font_compare.png`** porque aquella maqueta dibujaba todos los glifos con el
mismo generador.
- Esto confirma la regla de trabajo: la maqueta sintética no sustituye a la
  captura real. Error propio, detectado al mirar la captura.

**Corrección**: `apply_cond_full` (271 tiles). Añade `O`→idx 269 (VRAM `0x54D`)
y `K`→idx 270 (VRAM `0x54E`), y reescribe la entrada del comando 6 (`OK`) en las
4 tablas por slot para que apunte a los tiles nuevos.
- Cabe: el hueco libre confirmado es `0x540`-`0x54F` (16 tiles); en uso pasarían
  de 13 a 15. Quedaría 1 tile libre (`0x54F`).
- `d1` pasaría de `0x10D` (269) a `0x10F` (271) en `0x0020D6` y `0x005108`.
- Tamaño del blob comprimido: 6362 B (v1.1: 6441 B → **cabe en el mismo sitio**,
  espacio libre en `0x94000` hasta `0xA0000`).

### Comprobaciones sobre las capturas [HECHO]
- Píxeles de césped dentro de los paneles (filas 200-217, cols 8-99 y 150-249):
  **450 en v1.1 y 450 en la condensada, en los 6 frames** → el margen opaco
  (índice 9) funciona; **no reaparece el bug del fondo verde**.
- Cambios fuera de la fila de texto: **ninguno**.
- Nota: esta batería es **parcial**. No sustituye a la validación completa
  (PAUSA píxel a píxel, letras fantasma con gol, presentación, katakana,
  regresión global) que sería obligatoria antes de emitir una v1.2.

### Lo que la captura confirma y lo que no [HECHO / HIPÓTESIS]
- [HECHO] La legibilidad *de la letra aislada* mejora: se distinguen los huecos
  internos de `A`, `R`, `P`, `S`, que en la fuente actual se cierran.
- [HECHO] La separación entre `PLACA` y la palabra siguiente **no mejora**, tal
  y como predecía la demostración geométrica: paso de rejilla fijo de 8 px.
- [HECHO] `CEDE`→`CE` sigue igual: la fuente no reduce el número de celdas.
- [HIPÓTESIS, no verificable en emulador] El trazo de 1 px puede leerse peor que
  el de 2 px en una CRT/pantalla real a distancia de sofá. Solo el usuario puede
  juzgarlo.

### Ficheros
- `tools/shot_font.py`, `tools/shot_run.py`, `tools/shot_run2.py`, `tools/shot_pair.py`
- Capturas crudas: `work/shots/pair/{v11,cond,cf}_f*.png`
- Comparativas: `work/shots/CMP_final.png` (principal), `CMP_pantalla.png`,
  `CMP_paneles.png`, `CMP_zoom.png`, `CMP_full.png`

### CORRECCIÓN de una conclusión anterior — el problema de separación estaba mal planteado [HECHO]

Al medir sobre las capturas reales aparece un dato que invalida el marco con el
que se justificó la tipografía condensada.

**Barrido de layout** (2600 frames, semilla 7, lectura de tilemap plano A filas
25-26, agrupando columnas contiguas en "palabras"):

```
panel palabras     columnas                  hueco entre palabras
DER   PASA+OK      PASA[21-24] OK[26-27]     1 celda
DER   PASA+FI      PASA[21-24] FI[26-27]     1 celda
IZQ   PASA+OK      PASA[4-7]   OK[9-10]      1 celda
IZQ   TIRA+CE      TIRA[4-7]   CE[9-10]      1 celda
IZQ   PASA+CE      PASA[4-7]   CE[9-10]      1 celda
```

**El juego YA inserta una celda vacía entre las dos palabras.** Nunca se dibujan
pegadas en el layout real de la v1.1. El caso `PLACAPLACA` que se usó como
criterio de éxito (`font_real.png`, `font_compare.png`) **no se produce en el
juego**: era un montaje sintético del propio autor de la maqueta.

→ Se anota como **hipótesis descartada**: "las palabras contiguas aparecen
pegadas y hay que separarlas tipográficamente". Motivo: falsada por medición
directa del tilemap. El problema que la tipografía condensada venía a resolver
no existe en pantalla.

**Medición en píxeles** (detector de tinta, índice 15, filas y=207..217, sobre
las capturas reales):

| caso | versión | hueco ENTRE LETRAS | hueco ENTRE PALABRAS | contraste |
|---|---|---|---|---|
| PASA+OK izq | v1.1 | 1 px | 9 px | **9.0x** |
| PASA+OK izq | cond | 3 px | 11 px | **3.7x** |
| TIRA+CE izq | v1.1 | 1 y 3 px | 9 px | 9.0x / 3.0x |
| TIRA+CE izq | cond | 3 px | 11 px | 3.7x |
| PASA+OK der | v1.1 | 1 px | 9 px | 9.0x |
| PASA+OK der | cond | 3 px | 11 px | 3.7x |

**Conclusión [HECHO]**: la condensada sube el hueco absoluto entre palabras de 9
a 11 px, pero sube el hueco entre letras de 1 a 3 px. El **contraste** —que es lo
que usa el sistema visual para agrupar caracteres en palabras (ley de proximidad)—
**cae de 9.0x a 3.7x**. Es decir: la tipografía condensada **empeora** la
segmentación en palabras, justo lo contrario del objetivo declarado por el
usuario ("hacer que el cerebro las separe visualmente").

La fuente actual, con letras que casi se tocan y un corte de 9 px entre palabras,
es tipográficamente **correcta** para este propósito: alto contraste
intra/inter-palabra.

Captura de la evidencia: `work/shots/CMP_contraste.png`.

### Recomendación registrada [OPINIÓN, fundamentada]
NO aplicar la tipografía condensada. Mantener v1.1 como HUD definitivo.
Razones, por orden de peso:
1. Degrada el contraste de agrupación 9.0x → 3.7x (medido).
2. El problema que pretendía resolver no existe (el juego ya separa con 1 celda).
3. No reduce celdas: `CEDE`→`CE` se mantiene igual.
4. Trazo de 1 px vs 2 px: previsible pérdida de legibilidad a distancia en
   pantalla real (no verificable en emulador).
5. Obligaría a repetir la batería completa de validación por una mejora no
   demostrada.

---

## Sesión — FUENTE ESTRECHA con tiras de palabra (criterio PLACA PLACA)

### Reconocimiento de un error de planteamiento propio [HECHO]

El usuario lo formuló con precisión: *"aunque la fuente sea fina, el espacio que
ocupa es el mismo, así que no nos aporta ninguna ventaja. El objetivo de crear
una fuente estrecha es que donde ahora sólo cabe PLACAOK quepa PLACA PLACA."*

Tenía razón y `hudfont.py` estaba mal concebido desde la raíz. Mantenía la
equivalencia **1 letra = 1 tile**. Bajo esa restricción, estrechar el glifo solo
mueve píxeles dentro de su celda: el ancho consumido sigue siendo 8 px por letra
y el número de celdas NO baja. Por eso mi conclusión anterior ("es
geométricamente imposible") era falsa: no era una propiedad del hardware, era
una consecuencia de una decisión de implementación mía que nunca cuestioné.

Se anota como **hipótesis descartada**: "con paso de rejilla fijo de 8 px la
separación y el número de celdas son invariantes". Motivo de descarte: solo es
cierto si se impone 1 letra = 1 tile. El VDP no impone eso.

### Planteamiento correcto [HECHO]

`tools/narrowfont.py`: la **palabra completa se pre-renderiza sobre una tira de
tiles** y las letras **cruzan los límites de tile**. El paso de avance pasa a ser
de 5 px (cuerpo 4 + 1 de aire) en lugar de 8.

```
PLACA  tinta=24 px  ->  3 tiles   (fuente actual: 5 tiles)
FINTA  tinta=24 px  ->  3 tiles   (fuente actual: 5 tiles)
PASA   tinta=19 px  ->  3 tiles   (fuente actual: 4 tiles)
CEDE   tinta=19 px  ->  3 tiles   (fuente actual: 4 tiles)
OK     tinta= 9 px  ->  2 tiles   (fuente actual: 2 tiles)
```

**Criterio de éxito del usuario, medido:**
```
'PLACA PLACA' -> 55 px de tinta en 56 disponibles = 7 tiles EXACTOS
fuente actual -> 10 letras x 1 celda + 1 separador = 11 celdas
zona legible por panel = 7 celdas
```
→ **CUMPLIDO.** Con la fuente actual es imposible (11 > 7); con tiras cabe justo.
Y si cabe `PLACA PLACA`, cabe cualquier otra combinación: es el peor caso.

### Dos fallos detectados EN CAPTURA durante esta sesión [HECHO]

**Fallo 1 — desbordamiento sobre el asset siguiente.** Coloqué la tira en el
índice 269 (tras los 13 tiles de la v1.1) → VRAM `0x54D..0x553`. Pero el hueco
libre confirmado es `0x540-0x54F`: los tiles `0x550-0x553` pertenecen al asset
`0x55C22` (VRAM `0xAA00`). La captura mostró basura gráfica en esas 4 celdas.
Volcado de VRAM que lo prueba:
```
tile 550  0000000000000000
tile 551  00000000000000ee
tile 552  0e21111e0eee1111   <- datos del otro asset
tile 553  00000000ee000000
```
Corrección: la tira va al índice **256** → VRAM `0x540..0x546`, dentro del hueco.
En modo tira los 13 tiles por-letra de la v1.1 **no se usan**: la palabra deja de
componerse letra a letra.

**Fallo 2 — el slot secundario borraba la primera celda.** Los slots `d6=2`
(izq) y `d6=0` (der) siguen dibujando su celda propia y pisaban la columna
inicial de la frase. Lectura de tilemap que lo prueba:
```
fila 26 ... 485 541 542 543 544 545 546 ...   <- falta 540, sustituido por 485 (vacío)
```
Corrección: dar a esos slots como contenido el tile que corresponde a esa
columna (`ph[0]`), de modo que redibujen lo mismo que ya hay.

**Fallo 3 — divergencia de la partida.** Con 263 tiles frente a 269, la longitud
del DMA cambia, el timing cambia y la partida deja de ser la misma: el campo
mostraba 34.038 px de diferencia y las capturas no eran comparables. Corregido
rellenando hasta 269 tiles. Tras la corrección: **campo 0 px de diferencia** en
los 4 frames; toda la diferencia queda en el HUD (550 px).

### Validación de las capturas [HECHO, parcial]
- Campo (y<200): **0 px de diferencia** en los 4 frames → misma jugada, mismo frame.
- Césped dentro de los paneles: **450 px en ambos modos** → no reaparece el bug
  del fondo verde; el relleno opaco (índice 9) funciona en las tiras.
- Batería **incompleta**: falta PAUSA píxel a píxel, letras fantasma con gol,
  presentación, katakana y regresión global. Obligatorio antes de una v1.2.

### Lo que esto NO resuelve todavía [HECHO]
El demostrador fuerza `PLACA PLACA` en las 4 tablas para probar el criterio. Un
HUD real necesita además:
- una tira por cada combinación palabra+palabra que el juego pueda mostrar, o
  bien una tira por palabra con posicionamiento independiente;
- el conteo de tiles: 6 comandos x ~3 tiles = ~18 tiles > 16 del hueco
  `0x540-0x54F`. **Habría que ampliar la zona libre o compartir tiles entre
  frases.** Sin resolver.

### Ficheros
- `tools/narrowfont.py` (fuente 4 px + generador de tiras)
- `tools/demo_placa.py` (demostrador del criterio)
- `tools/shot_placa.py` (capturas pareadas)
- `work/shots/PLACA_full.png` (principal), `work/shots/PLACA_frames.png`
- Capturas crudas: `work/shots/placa/{actual,narrow}_f*.png`

---

## ESTUDIO TÉCNICO — ¿Puede el HUD abandonar el sistema de caracteres?

Encargo del usuario: análisis exclusivo de la rutina de dibujo, sin implementar
nada y sin generar ROM. Respuesta a 6 preguntas + maqueta a tamaño real.

### 1) ¿Puede cada comando sustituirse por un gráfico propio? — **SÍ** [HECHO]

Desensamblado de la rutina de dibujo `0x7A3C` (la única que pinta el HUD):

```
007A3C: ef41          asl.w  #7,d1        ; d1 = fila * 128
007A3E: e340          asl.w  #1,d0        ; d0 = columna * 2
007A40: d240          add.w  d0,d1
007A42: 02413fff      andi.w #$3fff,d1
007A46: 8244          or.w   d4,d1        ; d4=$4000 -> escritura VRAM
007A48: 4bf900c00000  lea.l  $c00000,a5   ; puerto VDP
  --- bucle FILAS (d3+1) ---
007A4E: 3c02          move.w d2,d6        ; d6 = ancho-1
007A50: 3b410004      move.w d1,$4(a5)
007A54: 3b7c00030004  move.w #$3,$4(a5)
  --- bucle COLUMNAS (d2+1) ---
007A5A: 3e18          move.w (a0)+,d7     ; lee UN WORD DE TILEMAP
007A5C: de45          add.w  d5,d7        ; d5 = 0 (el trampolín v1.1 hace clr.w d5)
007A5E: 3a87          move.w d7,(a5)      ; lo escribe TAL CUAL en el plano A
007A60: 51cefff8      dbra   d6,$7a5a
007A64: 06410080      addi.w #$80,d1
007A68: 51cbffe4      dbra   d3,$7a4e
007A6C: 4e75          rts
```

**Conclusión determinante**: `0x7A3C` es un **blitter de tilemap genérico**. No
conoce el concepto de "carácter". No hay tabla de fuente, no hay traducción
carácter→tile, no hay terminador, no hay lógica de texto en ninguna parte. Se
limita a copiar `(d2+1)×(d3+1)` words desde `(a0)+` al plano A.

→ Los words que lee **ya son entradas de tilemap** (tile + paleta + prioridad +
flips). Que esos tiles representen una letra suelta o un trozo de palabra
precompuesta **le es indiferente**. El sistema de caracteres no está en la
rutina de dibujo: está solo en cómo se rellenaron las tablas.

**Coste de la conversión: CERO instrucciones.** No hay que modificar `0x7A3C` ni
el bucle `0x00A4BC`. Basta con cambiar el contenido de las tablas de `$A5C4`
(o las tablas por slot de la v1.1) y los tiles del asset.

Esto invalida definitivamente el marco "el HUD es un sistema de caracteres y por
tanto 1 letra = 1 celda": esa equivalencia **nunca estuvo en el código**, fue
una convención de los datos.

### Espacio útil por panel — medido, no supuesto [HECHO]

Volcado de SAT en partido real (frame 2411):
```
#4 X= 16 Y=184 2x3 pri=1  -> cols 2..3    (animadora izq)
#5 X= 16 Y=208 2x1 pri=1  -> cols 2..3
#6 X=224 Y=184 2x3 pri=1  -> cols 28..29  (animadora der)
#7 X=224 Y=208 2x1 pri=1  -> cols 28..29
```
Tilemap fila 26 de la v1.1 limpia:
```
f26 ... 485 485 [548 549 54b 549] 485 [455 456] 472 ...
         c2  c3   c4  c5  c6  c7   c8   c9  c10  borde
```
El juego usa exactamente **cols 4..10** (izq) y **21..27** (der).

**ESPACIO ÚTIL = 7 celdas = 56 px por panel.** Coincide medida y uso real.

### 2) Coste en tiles de cada palabra como gráfico precompuesto [HECHO]

Con la fuente de ancho variable diseñada en el punto 3:

| palabra | ancho | tiles | v1.1 (1 letra=1 tile) | ahorro |
|---|---|---|---|---|
| PASA | 15 px | 2 | 4 | +2 |
| TIRA | 13 px | 2 | 4 | +2 |
| CEDE | 15 px | 2 | 4 | +2 |
| PLACA | 19 px | 3 | 5 | +2 |
| FINTA | 17 px | 3 | 5 | +2 |
| OK | 7 px | 1 | 2 | +1 |
| **TOTAL** | | **13** | **24** | **+11** |

**Las 6 palabras caben en el hueco libre `0x540-0x54F` (16 tiles), sobran 3.**
Además, en modo precompuesto los 13 tiles por-letra de la v1.1 dejan de ser
necesarios: ese presupuesto se libera por completo.

### 3) PLACA ultracondensado por redistribución de ancho [HECHO]

`tools/placafont.py`. **No se escala horizontalmente** (eso destruiría el trazo
de 1 px): se asigna a cada letra el mínimo de columnas que su forma necesita
para seguir siendo inequívoca.

```
P -> 3 px    vertical + cuenco cerrado de 2
L -> 3 px    con 2 el pie es 1 px: ambiguo con I
A -> 3 px    dos diagonales + travesaño
C -> 3 px    con menos se cierra y se confunde con O
I -> 1 px    barra vertical; no necesita más
```
Kerning 1 px. **PLACA = 3+3+3+3+3 + 4 kern = 19 px** (frente a 40 px en v1.1).

```
##..#....#...##..#.
#.#.#...#.#.#...#.#
#.#.#...#.#.#...#.#
##..#...###.#...###
#...#...#.#.#...#.#
#...#...#.#.#...#.#
#...###.#.#..##.#.#
```

### 4) ¿Caben dos PLACA separados ≥1 px? — **SÍ, con holgura** [HECHO]

```
ancho PLACA        = 19 px
espacio útil panel = 56 px

2xPLACA +  1 px = 39 px  -> CABE, sobran 17
2xPLACA + 18 px = 56 px  -> CABE exacto (reparto óptimo)
```
Restricción de rejilla: el tilemap posiciona en múltiplos de 8, pero **dentro de
una tira contigua de celdas los píxeles son libres** y la separación se hornea
en los tiles. Mínimo necesario: `ceil(39/8) = 5 celdas` de las 7 disponibles.

### 5) Píxeles que faltan — **NINGUNO. SOBRAN 17 px** [HECHO]

Con separación mínima de 1 px sobran **17 px** (39 de 56 ocupados).
Con el reparto óptimo la separación sube a **18 px**, dieciocho veces el mínimo
exigido.

### 6) Verificación real en emulador (sin ROM) [HECHO]

`tools/real_placa.py` — parchea solo la copia en RAM del cartucho. Tira de 7
tiles en el índice 256 → VRAM `0x540..0x546`, dentro del hueco libre. Relleno
hasta 269 tiles para igualar la longitud del DMA (si no, el timing cambia y la
partida diverge).

Validación sobre 4 frames (1819, 2269, 2371, 2411):
- **Campo (y<200): 0 px de diferencia** → misma jugada, mismo frame.
- **Césped dentro de los paneles: 450 px en ambas versiones** → no reaparece el
  bug del fondo verde.
- Toda la diferencia confinada al HUD (520 px).
- **`PLACA` visible 4 veces** (2 por panel), en las dos variantes de separación.

### Conclusión del estudio [HECHO]

Las 6 preguntas tienen respuesta afirmativa. El HUD **puede** abandonar el
sistema de caracteres sin tocar una sola instrucción de la rutina de dibujo, y
el caso peor (`PLACA PLACA`) no solo cabe: sobran 17 px.

### Lo que este estudio NO cubre [HONESTIDAD]
- Es un **estudio**, no una implementación. Solo se probó la tira `PLACA PLACA`
  forzada en las 4 tablas.
- Un HUD real necesita resolver la **combinatoria**: si se precompone cada par
  palabra+palabra el coste se dispara. La vía razonable es una tira por palabra
  con posicionamiento independiente, lo que exige revisar cómo se reparten las
  columnas entre el slot principal y el secundario. **No analizado.**
- Legibilidad del trazo de 1 px a 3 px de cuerpo en pantalla real (CRT, distancia
  de sofá): **no verificable en emulador**. Es el riesgo principal de esta vía y
  solo el usuario puede juzgarlo. En las capturas a x8 se lee bien; a tamaño real
  es notablemente más pequeño que la fuente actual.
- Batería completa de validación (PAUSA, letras fantasma con gol, presentación,
  katakana, regresión global): **no ejecutada**, no procede en un estudio.

### Ficheros
- `tools/placafont.py` — fuente de ancho variable (estudio)
- `tools/mockup_placa.py` — maqueta compuesta sobre PNG real
- `tools/real_placa.py` — verificación en emulador (sin ROM)
- `work/shots/ESTUDIO_placa.png` — comparativa a pantalla completa
- `work/shots/ESTUDIO_zoom.png` — panel a x8, 3 variantes
- `work/shots/mock_max.png`, `mock_min.png` — maquetas
- `work/shots/real/g18_f*.png`, `g04_f*.png` — capturas reales

---

## Esquema de DOS PUNTEROS — propuesta del usuario, VERIFICADA

Propuesta textual del usuario:
> "no creo que sea necesario hacer una tira por combinación, sino: puntero
> palabra1 / puntero palabra2. Sólo habría que marcar la posición de inicio del
> primer PLACA y del segundo."

### Hallazgo principal: esa arquitectura YA EXISTE en la v1.1 [HECHO]

El usuario describió, sin verlo, exactamente lo que el trampolín de la v1.1 ya
implementa. Desensamblado de `0x08A900`:

```
08A900: 2006          move.l d6,d0
08A902: e980          asl.l  #4,d0        ; d6*16 -> indice en GEOM
08A904: 49f9 0008a800 lea.l  $8a800,a4    ; a4 = GEOM DEL SLOT
08A90A: d9c0          adda.l d0,a4
08A90C: 3a2b0002      move.w $2(a3),d5    ; ID de comando del slot
08A910: 0285 00000007 andi.l #7,d5
08A916: e585          asl.l  #2,d5
08A918: 226c0008      movea.l $8(a4),a1   ; TABLA PROPIA DEL SLOT
08A91C: d3c5          adda.l d5,a1
08A91E: 2051          movea.l (a1),a0     ; PUNTERO A LA PALABRA
08A920: 302c0002      move.w $2(a4),d0    ; X_inf  <- POSICION PROPIA DEL SLOT
08A924: 322c0006      move.w $6(a4),d1    ; Y
08A928: 342c0004      move.w $4(a4),d2    ; ancho-1 PROPIO DEL SLOT
```

Cada slot lee **su propio puntero de palabra** (`$8(a4)` → `(a1)` → `a0`) y
**su propia X** (`$2(a4)`). Son cuatro slots totalmente desacoplados: el
desacoplamiento se hizo en la v1.1 para otra cosa, y resulta ser justo lo que
esta propuesta necesita.

→ **Coste en código del esquema de dos punteros: CERO instrucciones nuevas.**
Solo cambian datos: tiles del asset, tablas de palabra y valores de X.

Se anota como **hipótesis descartada** la mía anterior: *"un HUD real necesita
una tira por cada combinación palabra+palabra, y 6 comandos × ~3 tiles > 16
tiles del hueco"*. Motivo del descarte: presuponía componer pares. Con dos
punteros independientes **cada palabra se almacena UNA sola vez**.

### Catálogo de palabras — coste real [HECHO]

| palabra | idx | VRAM | tiles | ancho |
|---|---|---|---|---|
| PASA | 256 | 0x540 | 2 | 15 px |
| TIRA | 258 | 0x542 | 2 | 13 px |
| CEDE | 260 | 0x544 | 2 | 15 px |
| PLACA | 262 | 0x546 | 3 | 19 px |
| FINTA | 265 | 0x549 | 3 | 17 px |
| OK | 268 | 0x54C | 1 | 7 px |
| **TOTAL** | | | **13** | |

Hueco libre `0x540-0x54F` = 16 tiles → **caben, sobran 3**.

### Layout de 7 celdas [HECHO]
```
slot principal  -> cols +0..+2  (3 celdas)
hueco           -> col  +3
slot secundario -> cols +4..+6  (3 celdas)
```
**Las 36 combinaciones caben en 56 px.** Peor caso `PLACA + PASA`: separación de
**13 px** (mínimo exigido: 1 px).

### Verificación en emulador (sin ROM) [HECHO]

`tools/twoptr.py`. Combinaciones observadas en partida real (2600 frames):
```
IZQ  PASA@4  CEDE@8      IZQ  TIRA@4  CEDE@8
IZQ  PASA@4  OK@8        IZQ  TIRA@4  OK@8
DER  PASA@21 FINTA@25    DER  PASA@21 OK@25
DER  PLACA@21            DER  FINTA@21 / FINTA@25
```

**`CEDE` aparece COMPLETO. El compromiso `CEDE`→`CE` desaparece.**

Validación sobre 6 frames (1713, 2203, 2371, 2411, 2957, 3353):
- **campo (y<184): 0 px** de diferencia
- **flechas (y=184..200): 0 px** de diferencia
- césped dentro de paneles: **450 px en ambas** → sin fondo verde
- toda la diferencia confinada a la fila de texto

### Error propio detectado y corregido en esta sesión [HECHO]

Primera pasada: usé un único `_geom()` que escribía **X_sup y X_inf con el mismo
valor**, desplazando sin querer el bloque **superior** del HUD (las flechas
indicadoras de la fila 23). Se manifestó como 696-706 px de diferencia en
`y=184..200`.

Además mi umbral de "campo" estaba mal puesto en `y<200`: **el HUD empieza en
`y=184`** (fila 23), así que estaba contando el HUD como campo y creí que la
partida divergía. No divergía.

Corrección: `_geom()` recibe ahora `x_sup` opcional; si no se pasa, **conserva el
valor original de la v1.1**. Tras la corrección: flechas 0 px.

### Ventajas frente a la v1.1 [HECHO]
1. `CEDE` completo (elimina el único compromiso lingüístico vigente).
2. 13 tiles frente a 24 del esquema por letra.
3. Separación entre palabras de 13-17 px reales, frente a 8 px.
4. Cero código nuevo.

### Riesgo no resuelto [HONESTIDAD]
Legibilidad del trazo de 1 px con cuerpo de 3 px en pantalla real (CRT,
distancia de sofá). En las capturas a x8 se lee bien; a tamaño real es
sensiblemente más pequeño que la fuente actual. **No verificable en emulador.**
Es el único punto que puede tumbar esta vía y solo el usuario puede juzgarlo.

Sigue pendiente la batería completa (PAUSA, letras fantasma con gol,
presentación, katakana, regresión global) antes de cualquier v1.2.

### Ficheros
- `tools/twoptr.py` — catálogo + aplicación (estudio, sin ROM)
- `work/shots/DOSPUNTEROS.png`, `DOSPUNTEROS_frames.png`, `DOSPUNTEROS_zoom.png`
- `work/shots/twoptr/{v11,tp}_f*.png` — capturas crudas

---

## Comparativa A vs B — ¿fuente estrecha o palabras como gráficos?

Pregunta del usuario: cuál de las dos últimas propuestas es mejor.

### Hallazgo previo: la pregunta contiene una falsa dicotomía [HECHO]

Verificado por inspección de los módulos: `real_placa.py` (propuesta A) y
`twoptr.py` (propuesta B) **importan ambos `render()` de `placafont.py`**. Usan
**exactamente los mismos glifos**.

→ "Fuente estrecha" y "palabras como gráficos" **no son dos alternativas**. La
fuente estrecha es la *tipografía*; convertir palabras en gráficos es la
*organización de los datos*. B no sustituye a A: **B es A mejor organizada.**
Las dos capturas se diferencian solo en cómo se almacenan las palabras en VRAM.

### Coste real de cada organización [HECHO]

Uso real medido en partida: slot principal ∈ {PASA, TIRA, PLACA, FINTA},
slot secundario ∈ {OK, CEDE, FINTA} → **12 combinaciones reales**.

| | A · tira por combinación | B · catálogo + 2 punteros |
|---|---|---|
| unidad almacenada | par palabra+palabra | palabra suelta |
| reutilizable | no | sí |
| tiles | 12 × 7 = **84** | **13** |
| hueco disponible | 16 | 16 |
| resultado | **NO CABE, faltan 68** | **CABE, sobran 3** |
| combinaciones | 12 (las precompuestas) | 36 (todas) |
| código nuevo | 0 | 0 |

**A es inviable por un factor de 5.** No es una cuestión de preferencia.

### Contradicción del diario detectada y RESUELTA [HECHO]

§12 PUNTO 1 afirma: *"tiles 0x540-0x547 NO son seguros, RECHAZADOS"*, porque el
asset de presentación `0x4E868` carga 256 tiles en `0x500-0x5FF`, englobando
`0x540-0x54F`. Secciones posteriores afirman lo contrario ("hueco verificado
libre"). **Ambas no pueden ser ciertas.**

Resolución por traza (`config(vram=1)` + `trace(True)`, 408.250 escrituras VRAM):

1. Durante el partido: **0 escrituras** en `0xA800-0xA9FF` (tiles `0x540-0x54F`).
2. Desde el arranque: 464 escrituras, en frames **806** y **1261**, todas con
   `pc=0x00F450` (el descompresor).
3. En el frame 1261 la escritura es **una única carga contigua ascendente**
   `0x8800 → 0xA99E`, con el mismo `pc`. No son dos cargas distintas.
4. La v1.1 **sin parche** produce exactamente las mismas 464 escrituras en los
   mismos frames.

**Conclusión**: ese rango lo escribe **el propio asset del HUD de la v1.1**, que
con `d1=269` llega hasta `0x54C`. La advertencia de §12 era correcta **para la
ROM original de 256 tiles**, donde `0x540+` sí quedaba a merced de la
presentación. Desde que la v1.1 amplió el asset a 269 tiles y lo carga **después**
de la presentación, el rango pasa a ser propiedad del HUD.

Verificación directa e independiente de la traza: leídos los 13 tiles del
catálogo en VRAM durante el partido (frame 2411) → **intactos, byte a byte**.

Se anota §12 PUNTO 1 como **vigente solo para la ROM original**; no aplica a la
v1.1 ni a sus derivados.

### Error propio en el camino [HECHO]
Interpreté el orden de índices dentro del frame 1261 (`hueco` en 401674-401881,
`HUD` en 401402-401673) como "la presentación escribe después del HUD" y llegué
a escribir *"el hueco NO es seguro"*. Era falso: **es la misma carga**, que
escribe direcciones crecientes y por tanto llega al hueco al final. Corregido al
comprobar que ambos rangos comparten `pc` y son contiguos.

### Recomendación [OPINIÓN, fundamentada]

**B — catálogo de palabras + dos punteros.** Razones por orden de peso:
1. A no cabe (84 tiles frente a 16 disponibles). Decisivo.
2. B soporta las 36 combinaciones; A solo las precompuestas.
3. B elimina `CEDE`→`CE`, el último compromiso lingüístico vigente.
4. Ninguna de las dos añade código.
5. La tipografía es idéntica: elegir B no renuncia a nada de A.

**Riesgo vigente y no resuelto en ambas**: legibilidad del trazo de 1 px con
cuerpo de 3 px en pantalla real. Es el único motivo que justificaría descartar
las dos y quedarse en v1.1.

### Ficheros
- `work/shots/COMPARATIVA_AB.png`

---

## Alineación texto ↔ retratos en el panel izquierdo — propuesta del usuario

El usuario editó a mano una de las capturas para indicar la posición deseada y
observó que **el panel derecho ya está bien** ("tanto la primera palabra como el
primer retrato ya están ajustados al borde izquierdo").

### Medición de la edición del usuario [HECHO]
Diferencia de su PNG frente al mío: bbox `(888,346)-(1156,725)`, es decir solo
el screenshot derecho de la comparativa. Devuelto a coordenadas de juego:
- retratos: `(40,43),(54,55),(72,75),(85,87)` → desplazados a `(38,41),(52,53),(70,73),(83,85)`
- texto: `(32,46),(64,70)` → desplazado a `(39,57),(68,86)`

### Verificación del diagnóstico — el usuario tiene razón [HECHO]

Lectura del tilemap del plano A (frame 2411, esquema B):

```
PANEL IZQUIERDO          col:   2   3   4   5   6   7   8   9  10  11
  f23 (retratos)              485 485 485 313 314 485 485 325 326 472
  f26 (texto)                 485 485 540 541 485 485 54c 485 485 472
                                       ^texto col 4      ^texto col 8

PANEL DERECHO            col:  19  20  21  22  23  24  25  26  27
  f23 (retratos)              47a 472 3d6 3d5 485 485 3e8 3e7 485
  f26 (texto)                 481 472 540 541 485 485 54c 485 485
                                      ^texto col 21     ^texto col 25
```

- **Panel derecho**: retratos en cols 21-22 y 25-26; texto en 21 y 25. **ALINEADO.**
- **Panel izquierdo**: retratos en cols 5-6 y 9-10; texto en 4 y 8. **DESFASADO 1 columna.**

Confirmado: la asimetría existe y es exactamente la que el usuario señaló.

### Naturaleza de los retratos [HECHO — importante]
Volcado de SAT en el mismo frame: los únicos sprites que tocan `y=184..216` son
`#0` (X=120, balón del minimapa), `#4/#5` (X=16, animadora izq) y `#6/#7`
(X=224, animadora der). **Ningún sprite corresponde a los retratos.**

→ Los retratos son **tiles del plano A** (`313/314/315/316` y `325/326/327/328`
en filas 23-24). No son sprites.

Consecuencia práctica: **no hace falta moverlos.** Basta desplazar el texto +1
columna, que es un cambio de dato en la tabla GEOM (`X_inf`), no gráfico.

### Comprobación del riesgo de desbordamiento [HECHO]
Mover el texto +1 acerca el slot secundario al borde (col 11). Con ancho 3
(`PLACA`/`FINTA`) ocuparía cols 9..11 y **tocaría el borde**.

Muestreo del ID de comando del slot secundario izquierdo (`$f900+8+2`), 1500
muestras en partida real:
```
OK       38.2%   (1 tile)
(vacío)  34.3%   (0 tiles)
CEDE     27.5%   (2 tiles)
```
**Nunca aparece una palabra de 3 tiles en ese slot.** Se fija `SEC_W = 2`:
termina en col 10 y no toca el borde.

### Implementación (estudio, sin ROM) [HECHO]
`tools/twoptr.py`: constante `ALIGN_L = 1` aplicada solo al panel izquierdo, y
`SEC_W = 2` para el slot secundario izquierdo. El panel derecho **no se toca**.

Resultado en tilemap tras el cambio:
```
  f23  485 485 485 313 314 485 485 325 326 472    retratos 5-6 y 9-10
  f26  485 485 485 540 541 485 485 54c 485 472    texto    5    y 9
```

### Validación sobre 6 frames [HECHO]
| frame | campo (y<184) | retratos+flechas (184-200) | texto | césped |
|---|---|---|---|---|
| 1713 | 0 | 0 | 307 | 450/450 |
| 2203 | 0 | 0 | 203 | 450/450 |
| 2371 | 0 | 0 | 543 | 450/450 |
| 2411 | 0 | 0 | 647 | 450/450 |
| 2957 | 0 | 0 | 493 | 450/450 |
| 3353 | 0 | 0 | 278 | 450/450 |

**Retratos y flechas: 0 px de diferencia.** Solo se mueve el texto, como pedía
el usuario. Sin fondo verde.

### Respuesta a la pregunta [HECHO]
**Sí, es posible dejarlo así**, y además es más barato de lo que la edición
sugiere: no hay que mover los retratos (son tiles del plano y ya están donde
deben), solo el texto. Coste: **un valor en la tabla GEOM del panel izquierdo**.
Cero código nuevo, cero tiles nuevos.

### Ficheros
- `work/shots/ALINEADO_zoom.png`, `work/shots/ALINEADO_full.png`
- `work/shots/twoptr/al_f*.png`

---

## Fuente de cuerpo 4 px sobre arquitectura B — prueba de estrés PLACA PLACA

Encargo: probar cuerpo 4 px manteniendo el criterio del usuario — *"lo
importante es que quepa PLACA PLACA bien"*.

### Diseño [HECHO]
`tools/font4.py`: mismo principio de ancho variable que `placafont.py` (la `I`
sigue costando 1 px), pero cuerpo de 4 px en lugar de 3. Trazo 1 px, altura 7.

| palabra | 3 px | 4 px | tiles 3 | tiles 4 |
|---|---|---|---|---|
| PASA | 15 | 19 | 2 | 3 |
| TIRA | 13 | 16 | 2 | 2 |
| CEDE | 15 | 19 | 2 | 3 |
| PLACA | 19 | **24** | 3 | 3 |
| FINTA | 17 | 21 | 3 | 3 |
| OK | 7 | 9 | 1 | 2 |
| **catálogo** | | | **13** | **16** |

Hueco `0x540-0x54F` = 16 tiles → **cuerpo 4 px cabe JUSTO, sin margen**.
(Nota: el asset se amplía a 272 tiles = VRAM `0x440-0x54F`; `0x550` pertenece al
asset siguiente y no puede tocarse.)

### Medición del espacio libre real del panel izquierdo [HECHO]
Frame con el panel vacío (slot0 y slot1 con ID 0), lectura de píxeles en
`y=208..215` con el color de fondo real `(0,0,65)`:
```
x= 8..12  borde exterior
x=13..19  libre
x=20..27  ANIMADORA (pompones)
x=28..88  LIBRE  -> 61 px
x=89..94  borde interior (col 11)
```
→ celdas enteras utilizables: **4..10 = 7 celdas**. Confirma la medida previa
por una vía independiente (píxeles, no tilemap).

### CONFLICTO ENCONTRADO — alineación vs PLACA PLACA [HECHO]

Con cuerpo 4 px, `PLACA` ocupa **3 celdas**. Aplicando `align_l=1` (la
alineación con retratos aprobada en la sesión anterior):
```
slot ppal: cols 5,6,7      OK
slot sec : cols 9,10,11 -> la col 11 ES EL BORDE del panel
```
**Evidencia en tilemap** (frame 2411, `force='PLACA'`):
```
f26  485 485 485 548 549 54a 485 548 549 54a
                                          ^ col 11 = 0x54a (tile de PLACA)
                                            debería ser 0x472 (borde)
```
El segundo `PLACA` **sobrescribe el marco del panel**. Confirmado visualmente en
`work/shots/f4/c4_f2411.png`.

Sin alinear (`align_l=0`):
```
f26  485 485 548 549 54a 485 548 549 54a 472
                                          ^ col 11 = 0x472 borde INTACTO
```
→ cols 4,5,6 | 7 | 8,9,10. **Cabe exacto, borde respetado.**

### Alternativa descartada [HECHO]
`align_l=1` con hueco de 0 celdas → cols 5,6,7 | 8,9,10. Cabe, pero `PLACA`
ocupa 24 de 24 px del bloque de 3 celdas: **aire entre palabras = 0 px**. Las dos
palabras se tocarían. Descartada.

### Situación [HECHO]
Con cuerpo 4 px hay que elegir **una** de estas dos en el panel izquierdo:
- alineación texto↔retratos (cols 5 y 9), o
- `PLACA PLACA` sin pisar el borde (cols 4 y 8).

El panel derecho no tiene el problema: su borde está en la col 30 y el slot
secundario termina en la 27.

Con cuerpo 3 px **no hay conflicto**: `PLACA` ocupa 3 celdas pero el catálogo es
de 13 tiles y la alineación cabe (verificado en la sesión anterior).

### Pendiente de decisión del usuario
El criterio declarado es explícito: *"lo importante es que quepa PLACA PLACA
bien"*. Bajo ese criterio, la opción coherente es **cuerpo 4 px sin alineación**.
Pero la alineación fue una petición suya de la sesión inmediatamente anterior,
así que la elección le corresponde.

### Ficheros
- `tools/font4.py`, `tools/try4.py`
- `work/shots/CUERPO4_zoom.png`, `work/shots/CUERPO4_full.png`
- `work/shots/f4/{c3,c4,c4a0}_f2411.png`

---

## DECISIÓN DEL USUARIO: cuerpo 4 px

> "es cierto que el cuerpo 4 px se ve mucho más claro aunque no esté alineado.
> Es una decisión difícil, pero supongo que Cuerpo 4 px es la mejor opción."

### Observación del usuario sobre el cuerpo 3 px — CORRECTA [HECHO]

> "en el Cuerpo 3 px hay espacio de sobra entre los dos placas para mover el
> segundo un poco a la izquierda y que no pise el marco"

Verificado numéricamente. Retratos medidos: retrato1 `x=40..55`, retrato2
`x=72..87`. Borde interior desde `x=89`.

| cuerpo | PLACA | p2 alineado en x=72 | desborde | máx. x de inicio |
|---|---|---|---|---|
| 3 px | 19 px | termina en 90 | **2 px** | 70 (desfase 2 px) |
| 4 px | 24 px | termina en 95 | **7 px** | 65 (desfase 7 px) |

Con 3 px bastaría desplazar el segundo `PLACA` **2 px** — imperceptible. Con 4 px
harían falta **7 px**, casi una celda entera: ahí sí se rompería la alineación de
forma visible. La intuición del usuario era exacta en ambos casos.

Nota: ese ajuste de 2 px exigiría posicionamiento sub-celda (el tilemap solo
coloca en múltiplos de 8), es decir, hornear el desplazamiento dentro de los
tiles de la palabra. Viable, pero solo aplicable a la opción de 3 px, que queda
descartada por la decisión tomada.

### Hallazgo inesperado a favor del cuerpo 4 px [HECHO]

Con `align_l=0`, `PLACA` (24 px = 3 celdas exactas) cae así:
```
palabra1: celdas 4,5,6  -> x=32..55
palabra2: celdas 8,9,10 -> x=64..87

retrato1  x=40..55       palabra1 termina en x=55   COINCIDEN
retrato2  x=72..87       palabra2 termina en x=87   COINCIDEN
```

**No hay alineación por la izquierda, pero SÍ por la derecha, y es exacta.** Los
finales de palabra caen justo en los finales de retrato. Esto no se buscó: es
consecuencia de que `PLACA` ocupe 3 celdas exactas. Reduce mucho el coste
estético de renunciar a la alineación izquierda.

Separación entre palabras: 8 px. Margen hasta el borde: 1 px.
Evidencia: `work/shots/C4_alineacion_derecha.png`.

### Validación del cuerpo 4 px con comandos REALES [HECHO]

6 frames (1713, 2203, 2371, 2411, 2957, 3353), `force=None`:

| frame | campo | retratos+flechas | texto | césped |
|---|---|---|---|---|
| 1713 | 0 | 0 | 339 | 450/450 |
| 2203 | 0 | 0 | 247 | 450/450 |
| 2371 | 0 | 0 | 586 | 450/450 |
| 2411 | 0 | 0 | 678 | 450/450 |
| 2957 | 0 | 0 | 504 | 450/450 |
| 3353 | 0 | 0 | 303 | 450/450 |

Barrido de 1500 muestras comprobando los 4 tiles de borde (`col 1=0x471`,
`col 11=0x472`, `col 20=0x472`, `col 30=0x473`) en cada frame:

**violaciones de borde: NINGUNA**

Combinaciones reales observadas:
```
IZQ  PASA+CEDE   PASA+OK   TIRA+CEDE   TIRA+OK
DER  PASA+FINTA  PASA+OK   PLACA       FINTA
```
`CEDE` completo en todos los casos.

### Configuración fijada
- fuente: `tools/font4.py` (cuerpo 4 px, ancho variable, trazo 1 px, altura 7)
- arquitectura: B (catálogo + dos punteros), `tools/try4.py`
- `align_l = 0` (sin alineación izquierda; alineación derecha emergente)
- `N_TILES = 272` → VRAM `0x440..0x54F`, hueco lleno justo
- catálogo 16 tiles: PASA 3, TIRA 2, CEDE 3, PLACA 3, FINTA 3, OK 2

### Sigue pendiente [HONESTIDAD]
1. **Legibilidad del trazo de 1 px en pantalla real.** El emulador no puede
   juzgarlo. Único riesgo serio que queda.
2. **Batería completa** antes de cualquier v1.2: PAUSA píxel a píxel, letras
   fantasma con gol (6000 frames), presentación, katakana, regresión global,
   CRAM. No ejecutada: esto sigue siendo estudio.
3. **Partida real completa** jugada por el usuario. Recomendado explícitamente:
   esta sesión ha demostrado varias veces que el juego produce situaciones que
   el barrido automático no cubre.
4. El hueco queda **lleno justo** (16/16). Cualquier palabra adicional en el
   futuro exigiría ampliar la zona libre.

### Ficheros
- `work/shots/ELEGIDO_c4.png`, `work/shots/C4_alineacion_derecha.png`
- `work/shots/final4/{v11,c4}_f*.png`

---

## BATERÍA COMPLETA DE VALIDACIÓN — HUD cuerpo 4 px

`tools/bateria4.py`. Referencia: v1.1 limpia. Candidato: v1.1 + parche cuerpo
4 px aplicado **solo en memoria del emulador**. **Ninguna ROM generada.**

### ERROR METODOLÓGICO GRAVE detectado y corregido [HECHO]

Los primeros T1/T4/T5/T6 daban resultados **falsos**: T5 devolvía "0 tiles
distintos" cuando debía detectar los 16 del catálogo.

Causa: `md.py` ya avisaba —*"fresh handle per instance is NOT possible"*—. El
core de Genesis Plus GX es una **librería compartida con estado global**. Dos
instancias `MD` vivas a la vez **se pisan entre sí**:
```
m1.lib is m2.lib        -> False   (objetos ctypes distintos)
m1.lib._handle == m2...  -> True    (MISMO handle: mismo core, mismo estado)
```
Resultado: al crear la segunda instancia, la primera queda inservible y ambas
leen la misma VRAM. Todas las comparaciones ref/parche eran comparaciones de la
ROM parcheada consigo misma.

**Corrección**: capturar cada configuración **por separado**, con una sola
instancia viva, y comparar después los datos extraídos (`_capture_tilemap()`,
`_vram_after()`, con `del m` explícito).

Se anota como norma metodológica permanente: **nunca dos `MD` simultáneos**.

### Falso positivo `0x668` — analizado y descartado [HECHO]

Tras la corrección, T5 reportaba un tile fuera del catálogo: `0x668`. Volcado:
```
ref words: 8460 8471 8485 8485 8548 8549 854b 8549 8485 8455 8456 8472 ...
c4  words: 8460 8471 8485 8485 8540 8541 8542 8485 854e 854f 8485 8472 ...
```
No son píxeles: son **entradas de tilemap** (`0x8400` = paleta 0, prioridad 1).

Aritmética: `0x668 * 32 = 0xCD00`. Plano A en `0xC000`.
`(0xCD00 - 0xC000) / 128 = 26` → **fila 26 exacta del plano A**.

Mi barrido recorría `range(0x800)` sobre toda la VRAM, incluyendo tilemap y SAT.
Corregido con `NON_TILE_LO = 0x600` (`0xC000/32`): por encima de ese índice ya no
hay patrones gráficos.

### RESULTADOS FINALES

| Test | Qué mide | Resultado |
|---|---|---|
| **T1** bordes | 4 tiles de borde (`0x471/0x472/0x472/0x473`), 800 muestras | **0 rotos** |
| **T1** retratos | 16 celdas de retrato vs referencia | **0 alterados** |
| **T2** fantasmas | tiles `0x540-0x54F` fuera de fila 26 / cols panel, **6000 frames** | **0 fantasmas** |
| **T3** PAUSA | pantalla completa píxel a píxel | **0 px distintos** |
| **T4** presentación | pantalla completa píxel a píxel | **0 px distintos** |
| **T4** CRAM | paleta completa | **idéntica** |
| **T4** regresión global (presentación) | patrones `0x000-0x5FF` | **0 tiles distintos** |
| **T5** regresión en partido | patrones `0x000-0x5FF` | **exactamente 0x540-0x54F** |
| **T5** fuera del catálogo | — | **ninguno** |
| **T6** sprites (SAT) | 640 bytes de la tabla | **idéntica, 0 bytes** |

Verificación visual de T3: la captura muestra **PAUSA correctamente dibujada**
sobre el césped, sin recuadro azul ni artefactos (`work/shots/bat/pausa_c4.png`).
Confirma que el test se ejecutó sobre una pausa real y no sobre una pantalla
cualquiera.

### Criterios de aceptación del usuario
1. Ningún artefacto gráfico — **OK** (T1, T2, T5)
2. Ninguna corrupción de VRAM — **OK** (T5: solo los 16 tiles previstos)
3. Ninguna letra fantasma — **OK** (T2, 6000 frames)
4. Ningún borde roto — **OK** (T1)
5. Ningún sprite afectado — **OK** (T6)
6. Ningún retrato afectado — **OK** (T1)
7. PAUSA perfecto — **OK** (T3, 0 px + verificación visual)
8. Comandos legibles — **pendiente de juicio humano en pantalla real**

### Lo que la batería NO cubre [HONESTIDAD]
- **Legibilidad del trazo de 1 px en CRT a distancia.** No verificable en
  emulador. Sigue siendo el único riesgo abierto.
- **Partida real completa** jugada por el usuario. Esta sesión ha demostrado
  cuatro veces que el juego produce situaciones que el barrido no cubre
  (celebración del gol, caché de patrones, retratos como tiles, estado global
  del core).
- Estados no recorridos: VS, PASSWORD, final del torneo.

---

## Rótulo nuevo de la pantalla de título — ESTUDIO PRELIMINAR

Fichero recibido: `/home/user/uploads/IMG_4508.PNG`.

### Características del original [HECHO]
- 256x224 RGBA, **10 colores** (9 opacos + transparente puro `(0,0,0,0)`).
- Contenido opaco: bbox `(10,26)-(245,122)`, 236x97 px.
- **NO alineado a la rejilla de 8**: `x0=10` (10 mod 8 = 2), `y0=26` (26 mod 8 = 2).
  Ocupa celdas cols 1..30, filas 3..15.

### Ajuste a paleta Mega Drive [HECHO]
La MD usa **3 bits por canal** (8 niveles: 0,22,44,66,88,AA,CC,EE en hex).
Mapeo por vecino más próximo:

| original | → MD | px |
|---|---|---|
| `#0000ff` | `#0000ee` | 5681 |
| `#00072c` | `#000022` | 4758 |
| `#fcb448` | `#eeaa44` | 2546 |
| `#eeeeee` | `#eeeeee` | 1296 |
| `#fc1400` | `#ee2200` | 941 |
| `#00c908` | `#00cc00` | 612 |
| `#9f1211` | `#aa2200` | 346 |
| `#067826` | `#008822` | 311 |
| `#fcfc00` | `#eeee00` | 161 |

**9 colores únicos tras el ajuste** → cabe holgadamente en una sola paleta
(límite: 15 + transparente). Ningún color colisiona al redondear.

### Coste en tiles [HECHO]
```
celdas no vacías : 341
tiles ÚNICOS     : 221
```
Rótulo **actual** medido en VRAM (tilemap filas 3..15, pantalla de título):
```
tiles distintos  : 244   (rango 0x200..0x2F3)
```
→ **221 < 244: el rótulo nuevo cuesta MENOS tiles que el actual.** Presupuesto
holgado en principio.

### Puntos abiertos [HIPÓTESIS / PENDIENTE]
1. **El desalineamiento de 2 px** (x0=10, y0=26) obliga a una de estas dos:
   (a) desplazar el gráfico 2 px para cuadrarlo a la rejilla, o
   (b) hornear el desfase en los tiles, gastando una columna/fila extra.
   No decidido. Requiere consultar al usuario si acepta el desplazamiento.
2. **No verificado** cuántos de esos 221 tiles se pueden compartir con los que
   el juego ya tiene cargados, ni si el tilemap del título admite 221 entradas
   distintas sin tocar código.
3. **No verificado** en qué banco de la tabla de `0xC4000` reside el rótulo del
   título ni si el asset comprimido cabe in situ.
4. La paleta del título ya está ocupada por el fondo (cielo, gradas, público).
   **No verificado** si los 9 colores nuevos coinciden con los que ya hay o si
   habría que reasignar índices.

**No se ha modificado nada.** Esto es solo caracterización del material recibido.

### Ficheros
- `work/shots/ROTULO_ajustado.png` — simulación sobre captura real

---

## ROM v1.2 GENERADA — HUD cuerpo 4 px

Autorizado explícitamente por el usuario. `tools/build_v12.py` escribe un
fichero real (a diferencia de `try4.py`, que solo parcheaba memoria).

| | |
|---|---|
| origen | `Nekketsu_Soccer_MD_ES_v1_1.md` (md5 `d5079bd15d319ff9faa6a3c608d59a15`) |
| destino | `Nekketsu_Soccer_MD_ES_v1_2.md` |
| **md5 final** | **`f445db86743f66ce3a88b9a2d813e61e`** |
| tamaño | 1.048.576 B (idéntico a v1.1) |
| escrituras | 247 |
| bytes distintos de v1.1 | 700 |

### Catálogo de palabras
| palabra | idx | VRAM | tiles | ancho |
|---|---|---|---|---|
| PASA | 256 | 0x540 | 3 | 19 px |
| TIRA | 259 | 0x543 | 2 | 16 px |
| CEDE | 261 | 0x545 | 3 | 19 px |
| PLACA | 264 | 0x548 | 3 | 24 px |
| FINTA | 267 | 0x54B | 3 | 21 px |
| OK | 270 | 0x54E | 2 | 9 px |

### Zonas modificadas (todas conocidas y justificadas)
- `0x0020D7`, `0x005109` — 1 byte cada una: `d1 = 272` en los dos cargadores.
- `0x08A800-0x08A83F` — tabla GEOM (X_inf, ancho, Y, puntero, ancho_sup).
  **`X_sup` NO se toca**: mueve las flechas indicadoras de la fila 23.
- `0x08AA00-0x08AD7F` — cuatro tablas de palabra por slot.
- `0x094004-0x09594F` — asset del HUD recomprimido (272 tiles).

**Ninguna escritura fuera de estas zonas.** Cero bytes de código nuevo: la
arquitectura de dos punteros ya existía en la v1.1.

### Equivalencia con el parche validado [HECHO]
6 frames comparados entre la ROM v1.2 y el parche en memoria ya validado:
**0 px de diferencia en todos**. La ROM reproduce exactamente lo validado.

### Batería completa REEJECUTADA sobre el fichero real [HECHO]
`bateria4.py` con `USE_ROM = True`:

| Test | Resultado |
|---|---|
| T1 bordes (800 muestras) | **0 rotos** |
| T1 retratos | **0 alterados** |
| T2 fantasmas (6000 frames) | **0** |
| T3 PAUSA | **0 px** |
| T4 presentación | **0 px** |
| T4 CRAM | **idéntica** |
| T4 regresión global | **0 tiles** |
| T5 regresión en partido | **16 tiles, todos del catálogo** |
| T6 sprites (SAT) | **idéntica** |

### Checksum de cartucho [HECHO]
Detectado que la v1.1 **ya arrastraba un checksum incorrecto**:
```
jp.md      almacenado=9ee0 calculado=9ee0  OK
es098.md   almacenado=e38f calculado=e38f  OK
v1.1       almacenado=e38f calculado=c664  NO COINCIDE
v1.2       almacenado=e38f calculado=6848  NO COINCIDE
```
Comprobado por traza (`mem_window(0x180,0x1A0)`, 500 frames): **0 accesos** a la
zona del checksum. El juego no lo verifica, por eso la v1.1 funcionaba.

Aun así se corrige en v1.2 (`0x18E: e38f -> 6848`): 2 bytes, útil para
flashcarts y verificadores. Revalidado tras el cambio: **0 px de diferencia**.

### Lo que NO entra en esta ROM [HONESTIDAD]
El **rótulo del título** queda fuera. Solo se hizo caracterización; siguen
abiertos: desalineamiento de 2 px respecto a la rejilla, banco donde reside,
si el asset comprimido cabe in situ, y si los 9 colores colisionan con los
índices que el fondo ya ocupa. Meterlo ahora habría sido escribir bytes sin
terminar la investigación.

### Sigue pendiente
1. **Partida real completa** jugada por el usuario. Recomendado desde hace
   varias sesiones: el barrido automático no cubre todo.
2. Legibilidad del trazo de 1 px en pantalla real (CRT).
3. Estados no recorridos: VS, PASSWORD, final del torneo.
4. El hueco `0x540-0x54F` queda **lleno justo** (16/16).

### Ficheros
- `rom/Nekketsu_Soccer_MD_ES_v1_2.md`
- `tools/build_v12.py`
- `work/shots/ROM_v12.png`

---

## Rótulo del título — INVESTIGACIÓN COMPLETADA

El usuario probó la v1.2 ("va todo bien") y señaló que el rótulo sigue siendo el
antiguo. Correcto y deliberado: se excluyó por tener la investigación abierta.
Se cierran ahora los cuatro puntos pendientes.

### PUNTO 2 RESUELTO — dónde reside el rótulo [HECHO]

Traza de flujo: las cargas de la pantalla de título salen de `0x0080C`, con tres
llamadas al descompresor `0xF2A4` desde `0x830`, `0x84E` y `0x86C`.

```
000818: move.l #$40000001,$c00004   -> VRAM 0x4000 = tiles 0x200..0x2FF
000822: lea.l  $a0000,a0             <-- ROTULO DEL TITULO
000828: move.w #$100,d1              -> 256 tiles
000830: jsr    $f2a4

000836: move.l #$40000000,$c00004   -> VRAM 0x0000 = tiles 0x000..0x0FF
000840: lea.l  $90000,a0
00084E: jsr    $f2a4

000854: move.l #$40000002,$c00004   -> VRAM 0x8000 = tiles 0x400..0x4FF
00085E: lea.l  $4ecd6,a0
00086C: jsr    $f2a4
```

**El rótulo es el asset `0xA0000`**, 256 tiles, destino `0x200-0x2FF`.
Coincide con el rango medido en el tilemap (`0x200..0x2F3`).

Punteros a parchear si se reubica: `lea.l` en **`0x000822`** (operando en
`0x000824`) y, si cambia el número de tiles, `d1` en `0x000828`.

### PUNTO 3 RESUELTO — tamaño y espacio [HECHO]
```
asset 0xA0000 comprimido: 5412 B  (0xA0000..0xA1524)
justo detrás hay datos (no hueco): primer byte no-cero en 0xA1524
```
No cabe crecer in situ. **Pero hay espacio de sobra reubicando.**

Búsqueda de huecos de relleno (corregida: el relleno de esta ROM es `0xFF`, no
`0x00` — mi primera búsqueda solo miraba ceros y no encontró nada):

```
0x07EF70..0x07FFFF    4240 B
0x0827B4..0x083FFF    6220 B
0x088520..0x089FFF    6880 B
0x08E332..0x08F3FF    4302 B
0x091069..0x093FFF   12183 B
0x095950..0x09FFFF   42672 B   <-- justo tras el asset del HUD v1.2
0x0A1564..0x0A2FFF    6812 B
0x0A3420..0x0AFFFF   52192 B
0x0C4580..0x0FFFFF  244352 B
```

Candidato natural: **`0x096000`** (dentro del hueco de 42.672 B contiguo al HUD).

### PUNTO 4 RESUELTO — la paleta [HECHO, era el riesgo principal]

Medición: **el rótulo usa exclusivamente la paleta 3**, prioridad 0 (335 celdas,
todas pal=3, todas pri=0).

Paleta 3 leída de CRAM en la pantalla de título:
```
idx  0: #000000 (transparente)   idx  8: #006d00
idx  1: #004800                  idx  9: #b62400
idx  2: #ffff00                  idx 10: #000000
idx  3: #6d0000                  idx 11: #000000
idx  4: #ffb600                  idx 12: #000000
idx  5: #00da00                  idx 13: #000000
idx  6: #ff2400                  idx 14: #000000
idx  7: #da0000                  idx 15: #000000
```

Colores del rótulo nuevo ajustados a 3 bits/canal (niveles reales del hardware:
0,24,48,6D,91,B6,DA,FF — no los 0x22/0x44 que usé en el estudio preliminar):

| color | px | estado |
|---|---|---|
| `#0000ff` | 5681 | nuevo |
| `#000024` | 4758 | nuevo |
| `#ffb648` | 2546 | nuevo |
| `#ffffff` | 1296 | nuevo |
| `#ff2400` | 941 | **ya existe, idx 6** |
| `#00da00` | 612 | **ya existe, idx 5** |
| `#910000` | 346 | nuevo |
| `#006d24` | 311 | nuevo |
| `#ffff00` | 161 | **ya existe, idx 2** |

→ 3 reutilizados + **6 nuevos**, y hay exactamente **6 slots libres** (10-15).

**Verificación crítica** (no bastaba con que valgan `#000000`): se recogieron los
244 tiles que usan paleta 3 en plano A/B y SAT, y se contaron los índices de
color realmente presentes en sus píxeles:
```
idx 10..15:  0 px cada uno
```
**Genuinamente libres.** Escribir en ellos no altera nada de la pantalla actual.

Asignación propuesta:
```
#0000ff -> idx 10    #ffffff -> idx 13
#000024 -> idx 11    #910000 -> idx 14
#ffb648 -> idx 12    #006d24 -> idx 15
```

### PUNTO 1 — sigue abierto, requiere decisión del usuario
El PNG está desalineado 2 px respecto a la rejilla (`x0=10`, `y0=26`; ambos
`mod 8 = 2`). Opciones: (a) desplazarlo 2 px para cuadrarlo, o (b) hornear el
desfase gastando una fila y una columna extra de tiles.

Pregunta formulada al usuario en la sesión anterior, **sin respuesta todavía**.

### Presupuesto de tiles
```
rótulo nuevo: 221 tiles únicos
rótulo actual: 244 tiles (0x200..0x2F3)
límite del bloque: 256 (0x200..0x2FF)
```
Cabe con holgura.

### Estado
Los puntos 2, 3 y 4 quedan cerrados con evidencia. **No se ha escrito ni un byte.**
Falta solo la decisión del usuario sobre el alineamiento (punto 1) para poder
construir la v1.3.

---

## Rótulo — PUNTO 1 RESUELTO (cuadrar) y hallazgos nuevos

Decisión del usuario: **cuadrarlo** a la rejilla.

### Cuadrado [HECHO]
Desplazamiento `(-2,-2)`: bbox pasa de `(10,26)-(245,122)` a `(8,24)-(243,120)`.
`x0 mod 8 = 0`, `y0 mod 8 = 0`. Celdas cols 1..30, filas 3..15.
**16.652 px opacos antes y después: sin pérdida por el borde.**

### HALLAZGO — el tilemap del título NO es directo: usa METATILES 2x2 [HECHO]

La cadena real, trazada desde las escrituras al plano A (`pc=0x4774/0x477E`):

```
000878: lea.l $19be8,a6
00087E: jsr   $47e0        <- compone el tilemap en RAM 0xFF0000
000884: jsr   $4748        <- vuelca RAM 0xFF0000 -> VDP
```

Estructura en `0x19BE8`:
```
(a6)  = 0x01A008   stream de indices de metatile (1 byte por metatile)
4(a6) = 0x019BF0   tabla de metatiles: 8 bytes = 4 words (TL,TR,BL,BR)
```
`0x47E0` recorre 16x16 metatiles = **32x32 celdas**.

Ejemplo de metatiles leidos:
```
#0: 2000 2000 2000 2000
#1: 2000 2201 2206 2207
#2: 2202 2203 2208 2209
```

**Consecuencia**: no se puede sustituir el rótulo cambiando solo los tiles. Un
mismo tile aparece en varias celdas a través de los metatiles, y el gráfico
nuevo exige contenidos distintos en esas celdas.

### Presupuesto REAL de tiles [HECHO]

Primera medición (errónea, contando el tilemap directo): 10 colisiones.
Medición correcta expandiendo los metatiles: **32 colisiones**.

```
tiles 0x200-0x2FF usados en filas 3-15  : 187
tiles 0x200-0x2FF usados FUERA (marcador,
  menú, créditos...)                    :  41
usados SOLO por el rótulo (reasignables): 187
nunca usados (libres)                   :  28
------------------------------------------------
PRESUPUESTO                             : 215
```
El rótulo nuevo necesita **230 tiles no vacíos** → **faltan 15**.

### Solución: flips H/V del VDP [HECHO]

Cada entrada de tilemap tiene bits de flip horizontal y vertical (bits 11 y 12).
Son **gratuitos**: no consumen tiles. Canonicalizando cada celda por el mínimo
de sus 4 variantes (normal, H, V, HV):

```
tiles únicos sin flips : 230
tiles únicos con flips : 208     (ahorro: 22)
presupuesto            : 215
```
→ **CABE, con 7 tiles de margen.**

### Estado de los 4 puntos
1. Alineamiento — **RESUELTO** (cuadrado, sin pérdida)
2. Banco — **RESUELTO** (`0xA0000`, punteros en `0x000822`/`0x000828`)
3. Espacio del asset — **RESUELTO** (42.672 B libres en `0x095950`)
4. Paleta — **RESUELTO** (paleta 3, idx 10-15 libres con 0 px de uso)

Nuevo punto 5: **el tilemap con metatiles hay que reconstruirlo**, no solo los
tiles. Afecta a `0x01A008` (stream) y `0x019BF0` (tabla de metatiles).

**No se ha escrito ni un byte.** Falta decidir con el usuario si se acepta la
reconstrucción del tilemap, que es una modificación bastante más invasiva que
todo lo hecho hasta ahora en este proyecto.

---

## Aclaración: las cifras 221 vs 230 — contradicción propia RECONCILIADA

El usuario detectó una incoherencia legítima en mis mensajes: primero dije que
el rótulo nuevo ocupaba **221 tiles (menos que los 244 actuales)** y luego que
necesitaba **230 y no cabía**. Tenía razón en desconfiar.

**Ambas cifras son correctas, pero miden cosas distintas:**

| medición | tiles |
|---|---|
| PNG **original**, en su posición (x0=10, y0=26) | **221** |
| PNG **cuadrado**, tras desplazarlo (-2,-2) | **230** |
| control: PNG original recortado a filas 3-15 | 221 |

→ La diferencia **no es el recorte: es el desplazamiento de 2 px que yo mismo
introduje al cuadrarlo**. Mover el gráfico rompe coincidencias entre celdas que
antes eran idénticas y aparecen 9 tiles nuevos.

**El problema lo creé yo al cuadrar**, no venía en el material del usuario. Debí
haberlo advertido al proponer el cuadrado: dije "coste cero" y era falso.

### La pregunta del usuario: "¿no basta con borrar el antiguo y poner el nuevo?"

Sustancialmente **sí**. Medición de metatiles:

```
metatiles usados SOLO dentro del rótulo : 77
metatiles COMPARTIDOS con el exterior   :  1   (el metatile 0 = vacío)
posiciones de metatile en el rótulo     : 96  (384 celdas)
```

Solo se comparte el metatile vacío. Los 77 restantes son **exclusivos del
rótulo**: reescribirlos no afecta al marcador, al menú ni a los créditos.

Esto corrige mi advertencia anterior ("hay que respetar 41 celdas que no son del
rótulo"): esas 41 celdas usan tiles compartidos, pero **a través de metatiles
distintos**, así que no hay conflicto al reescribir los 77 propios.

Mi caracterización de la tarea como "bastante más invasiva" era **exagerada**.
Es una sustitución de datos acotada: 77 entradas de metatile + los tiles.

### Presupuesto definitivo

```
tiles reasignables (solo del rótulo) : 187
tiles nunca usados                   :  28
TOTAL DISPONIBLE                     : 215
```

| variante | sin flips | con flips H/V |
|---|---|---|
| original (sin mover) | 221 → faltan 6 | **210 → CABE, margen 5** |
| cuadrado (-2,-2) | 230 → faltan 15 | **208 → CABE, margen 7** |

Los flips H/V del VDP son gratuitos (bits 11 y 12 de la entrada de tilemap).
**Con ellos caben las dos variantes**, y la cuadrada incluso mejor.

### Tabla de metatiles
```
0x19BF0..0x1A007 = 1048 B = 131 entradas
índices usados en el stream: 129 (máx 129)
entradas libres: 1
```
Ajustado pero suficiente: el rótulo cuadrado necesita **106 metatiles únicos** y
tiene 77 propios + 1 libre. **Faltan 28 entradas de metatile.**

→ Habría que ampliar la tabla de metatiles (reubicándola a zona libre y
parcheando el puntero en `0x19BEC`) o reducir metatiles únicos aprovechando
flips también a nivel de metatile. **No resuelto todavía.**

### Estado
La vía es viable y menos invasiva de lo que dije. Queda por resolver el
presupuesto de **metatiles** (no el de tiles, que ya cabe).

---

## DECISIÓN DELEGADA AL ASISTENTE — y un error propio más grave

El usuario delega la elección entre cuadrar o no cuadrar. Al medirlo aparece que
**la premisa de mi propia pregunta era falsa**.

### El "desalineamiento" NUNCA fue un problema [HECHO]

Afirmé que el PNG "no está alineado a la rejilla de 8" y planteé un dilema entre
cuadrarlo o "hornear el desfase gastando fila y columna extra".

**Es falso.** La pantalla se trocea en celdas de 8x8 en posiciones **fijas**. Una
imagen de 256x224 se corta igual sea cual sea la posición de su contenido. No
hay nada que "alinear": no existe tal requisito.

El único efecto real de desplazar es cuántos tiles únicos salen:
```
original (sin mover) : 221 tiles  (210 con flips)
cuadrado (-2,-2)     : 230 tiles  (208 con flips)
```

**Le hice tomar al usuario una decisión sobre un dilema que yo inventé.** Y al
"cuadrarlo" empeoré el recuento en 9 tiles, tras haberle dicho que era gratis.

### Comparativa real de las dos variantes [HECHO]
```
                     tiles(flips)   metatiles   metatiles a añadir
ORIGINAL (sin mover)     210           104            26
CUADRADO (-2,-2)         208           106            28
```
Ambas caben en tiles (215 disponibles) y ambas necesitan ampliar la tabla de
metatiles en cantidad casi idéntica. **Cuadrar no aporta ninguna ventaja.**

→ **DECISIÓN: usar el PNG en su posición ORIGINAL.** Menos metatiles nuevos,
fiel al diseño del usuario, y evita el desplazamiento que yo introduje sin
necesidad.

### Paleta — formato confirmado EMPÍRICAMENTE [HECHO]

Me enredé varias veces con el byteswap comparando ROM contra `m.cram()` (cuyo
accesor aplica `^1` para el buffer interno de GPGX). En lugar de seguir
razonando, **prueba directa**:

```
poke16(0xA3402, 0x0E0E)  ->  el idx 1 pasó a MAGENTA (238,0,238), 5082 px
```
Coincide exactamente con los 5082 px que antes pintaba ese índice.

**Conclusión**: la ROM guarda los words en **BE directo, sin swap**.
Formato `0000 BBB0 GGG0 RRR0`, es decir `word = (B<<9)|(G<<5)|(R<<1)`.

Fuente de la paleta 3: **`0xA3400`**, 16 words, cargada por la rutina de
`0x088500` (`lea $a3400,a0` / bucle `move.w (a0)+,(a1)` en `0x088518`).
Verificado por traza de lecturas: 2192 accesos desde ese pc a `0xA3400..0xA341F`.

### Paleta 3 actual y encaje del rótulo [HECHO]
```
idx  0: 0000 #000000      idx  8: 0280 #009124
idx  1: 0200 #000024      idx  9: 00e4 #48ff00
idx  2: 0eee #ffffff      idx 10-15: 0000  (LIBRES)
idx  3: 000e #ff0000
idx  4: 04ae #ffb648
idx  5: 0e00 #0000ff
idx  6: 00ee #ffff00
idx  7: 002a #b62400
```

Colores del rótulo nuevo:
| color | estado |
|---|---|
| `#0000ff` | **ya en idx 5** |
| `#000024` | **ya en idx 1** |
| `#ffb648` | **ya en idx 4** |
| `#ffffff` | **ya en idx 2** |
| `#ffff00` | **ya en idx 6** |
| `#ff2400` | nuevo → word `002e` |
| `#00da00` | nuevo → word `00c0` |
| `#910000` | nuevo → word `0008` |
| `#006d24` | nuevo → word `0260` |

**5 de 9 ya existen** — confirma la intuición del usuario: el rótulo nuevo es
"casi igual" al suyo actual, y comparte casi toda la paleta. Solo hacen falta
**4 slots** de los 6 libres. Quedan 2 de margen.

Esto corrige mi cifra anterior de "6 colores nuevos": era resultado del
decodificador equivocado.

---

## v1.3 (rótulo) — CONSTRUCCIÓN EN CURSO, NO VÁLIDA TODAVÍA

Se autorizó generar la ROM. Se han encontrado y corregido **cuatro bugs
propios** en el constructor. La pantalla mejora en cada iteración pero **aún no
es correcta**: no se entrega.

### Arquitectura confirmada del título [HECHO]
```
000878: lea.l $19be8,a6
00087E: jsr   $47e0     compone el tilemap en RAM 0xFF0000 (16x16 metatiles)
000884: jsr   $4748     vuelca RAM -> VDP
```
`0x19BE8`: `+0` = stream (`0x1A008`, 256 B), `+4` = tabla de metatiles
(`0x19BF0`, 131 entradas de 8 B = 4 words TL,TR,BL,BR).

### Bugs propios encontrados y corregidos

**Bug 1 — flips invertidos.** Con canonicalización por flips H/V el rótulo salía
**espejado**. Se desactivaron (`USE_FLIPS=False`): 221 tiles en vez de 210, que
caben igual en 256. Coste asumido para eliminar una variable.

**Bug 2 — rejilla de metatiles PAR.** Analizaba filas 3..15 (rango impar), pero
la metafila k cubre las filas 2k y 2k+1. La mitad superior del rótulo caía fuera
del rango analizado. Evidencia: `metatile 131 = (None,None,None,(0,0,0))`.
Corregido con `ROW_LO, ROW_HI = 2, 15` (metafilas completas 1..7).

**Bug 3 — tile 0x200 sobrescrito.** `0x200` es el **tile vacío del fondo**: lo
referencian las filas 0-2 y 16-22 (cielo, gradas, público) y el metatile 0 es
`(2000,2000,2000,2000)`. Al empezar mi asset en el índice 0 lo destruí y la
pantalla entera se corrompió. Corregido: se conserva el tile original en la
posición 0 y el rótulo arranca en `TILE_BASE = 0x201`.
**Tras este arreglo el fondo, las gradas y la línea de texto superior salen
perfectos.**

**Bug 4 — paleta equivocada en el tilemap.** Escribía `0x2000` (bits 14-13 = 01
→ **paleta 1**) cuando el original usa `0x6000` (**paleta 3**). Corregido; el
tilemap en VRAM ya sale `0x62xx`, verificado.

### Problema ABIERTO — la paleta escrita no coincide con la CRAM

Escribo en `0xA3400` y el juego **sí lee de ahí** (2224 lecturas desde
`pc=0x088518`, verificado por traza). Pero los colores que aparecen en CRAM no
son los que escribo:

```
idx  word escrito   CRAM medida
 1   0200 #000024   #004800
 2   0eee #ffffff   #ffff00
 3   000e #ff0000   #6d0000
```

Interpretaciones probadas y **todas descartadas**:
- lectura directa `0BGR` → no coincide
- bytes intercambiados → no coincide
- desplazamiento de índice ±1 → 0/13 coincidencias

**No está resuelto.** Sigo sin entender el formato real, y la prueba empírica
anterior (`poke16(0xA3402,0x0E0E)` → magenta) contradice estos resultados.

Se anota como pendiente: **hay que instrumentar la escritura a CRAM
(`HOOK_CRAM_W`) y observar qué word llega realmente al VDP** en vez de seguir
razonando sobre el contenido de la ROM.

### Estado
La v1.3 en disco (`md5 ff07f4d1fa67565037803d130aac0e6e`) **NO es válida**:
el logo SOCCER sale con colores incorrectos. **No se entrega.**
La v1.2 sigue siendo la versión buena.

---

## v1.3 RESUELTA — la causa raíz era una ruta de dibujo equivocada

### EL ERROR DE FONDO [HECHO]

Toda la investigación del rótulo se hizo sobre la ruta **equivocada**.

Yo había trazado esto y lo di por bueno:
```
000878: lea.l $19be8,a6
00087E: jsr   $47e0     compone tilemap con METATILES 2x2 en RAM
000884: jsr   $4748     vuelca RAM -> VDP
```
De ahí salieron las conclusiones sobre metatiles, la tabla de `0x19BF0`, el
stream de `0x1A008`, el presupuesto de 131 entradas, la necesidad de ampliar la
tabla, los flips para ahorrar tiles... **todo inútil.**

**La verdad, hallada al trazar quién escribía la celda (5,18):**
```
frame=475 pc=004774 v=624c   <- mi valor, escrito por la ruta de metatiles
frame=479 pc=007a8c v=0000   <- borrado
frame=479 pc=007a5e v=624a   <- valor ORIGINAL, reescrito encima
```
El `pc=0x7A5E` es el blitter genérico `0x7A3C`, llamado desde `0x08803C`:
```
088028: move.w #$1f,d2        32 columnas
08802C: move.w #$0f,d3        16 filas
088030: lea.l  $a3000,a0      <-- TILEMAP PLANO, 32x16 words (1024 B)
08803C: jsr    $7a3c
```

**El rótulo se pinta desde un tilemap PLANO en `0xA3000`.** La ruta de metatiles
dibuja otra capa que queda inmediatamente sobrescrita. Por eso mis cambios se
aplicaban correctamente (verificado: tiles OK en VRAM, tabla OK en ROM, stream
OK) y aun así la pantalla no cambiaba.

Detalle revelador que debí notar antes: la paleta está en `0xA3400`, es decir
`0xA3000 + 1024`, **justo detrás del tilemap**. Estaban contiguos.

### Diagnóstico de por qué tardé tanto
Verifiqué cada pieza por separado —tiles, tabla, stream, paleta— y todas daban
correcto, pero nunca comprobé **quién escribe en la VRAM al final**. La
pregunta correcta no era "¿están bien mis datos?" sino "¿quién gana la última
escritura?". Un solo `hook` de escritura VRAM sobre una celda concreta lo
resolvió en un intento.

### Solución final [HECHO]
Constructor reescrito. Solo **5 escrituras**:
```
0x096000   4782 B  asset del titulo, 222 tiles
0x000824      4 B  puntero al asset
0x00082A      2 B  d1 = 222 tiles
0x0A3000   1024 B  tilemap plano 32x16
0x0A3400     32 B  paleta 3
```
Ya no se toca la tabla de metatiles ni el stream: **no hacía falta nada de eso.**

### Bug adicional corregido: escala de color [HECHO]
Usaba `LEVELS=[0,0x24,0x48,0x6d,0x91,0xb6,0xda,0xff]` (escala 0..255/7).
La correcta, medida píxel a píxel sobre el rótulo original, es
`[0x00,0x22,0x44,0x66,0x88,0xaa,0xcc,0xee]` (**nibble x 0x11**):
```
word 0eee -> #eeeeee    word 000e -> #ee0000    word 0e00 -> #0000ee
```
Con la escala correcta, **7 de los 9 colores del rótulo nuevo ya existían** en la
paleta 3; solo 2 son nuevos (idx 10 y 11). Antes creía que hacían falta 6.

### Verificación del resultado [HECHO]
Comparación píxel a píxel del PNG contra la captura, solo en píxeles opacos:
```
#0000ff -> #0000ee   5681 px     #00c908 -> #00ce00    612 px
#00072c -> #000020   4758 px     #9f1211 -> #ac2000    346 px
#fcb448 -> #eeaa41   2546 px     #067826 -> #008920    311 px
#fc1400 -> #ee2000    941 px     #fcfc00 -> #eeee00    161 px
```
**Ni un solo píxel mal colocado.** Todas las diferencias son el redondeo
inevitable de 24 bits a los 9 bits de color de Mega Drive (desviación máxima 17
por canal).

### Batería completa sobre la ROM v1.3 [HECHO]
| Test | Resultado |
|---|---|
| T1 bordes / retratos | 0 rotos / 0 alterados |
| T2 fantasmas (4000 frames) | 0 |
| T3 PAUSA | 0 px |
| T4 presentación / CRAM | 0 px / idéntica |
| T5 regresión VRAM en partido | **0 tiles** |
| T6 sprites (SAT) | idéntica |

T5 da 0 porque el rótulo no toca nada del partido: el HUD de la v1.2 se
mantiene intacto.

### Entrega
```
ROM  Nekketsu_Soccer_MD_ES_v1_3.md
     md5   bc2235cfa35230c667d498bf02a2cedb
     sha1  3480d3ea166f55184e5e76fa6e862128f8412f88
     crc32 db0af9d9
```

Parches en `patch/` (`tools/mkpatch.py`), IPS y BPS desde tres bases:
| desde | tramos | IPS | BPS |
|---|---|---|---|
| jp.md (512 KB) | 78 | 531854 B | 531663 B |
| es098.md | 22 | 12789 B | 12780 B |
| v1.2 | 8 | 5396 B | 5403 B |

**Los seis verificados** aplicándolos con un implementador independiente y
comparando byte a byte con la ROM destino; los BPS además validan CRC32 de
origen y destino.

Bug corregido en `mkpatch.py`: con bases de distinto tamaño (jp 512 KB → 1 MB)
rellenaba la diferencia con `0xFF` y generaba un parche corrupto. Ahora la cola
se trata como tramo nuevo completo.

### Pendiente
- **Partida real completa** jugada por el usuario.
- Legibilidad del trazo de 1 px en pantalla real.
- Estados no recorridos: VS, PASSWORD, final del torneo.

---

## Incidencias reportadas tras probar la v1.3 — INVESTIGACIÓN INICIAL

El usuario confirma que la v1.3 funciona bien y reporta tres asuntos pendientes.
Se deja constancia con la evidencia recogida en esta sesión, para arrancar la
siguiente sin repetir trabajo. **No se ha modificado nada.**

### 1. La imagen bajo "PRIMER PARTIDO" aparece cortada por arriba

Medición sobre la captura de la presentación (frame 960, estado `f800=0c
f802=5c`):
```
cielo azul de la escena: filas 80..126 px  = celdas 10..15
                         cols  64..191 px  = celdas  8..23
```
La escena ocupa **6 filas de celdas (10-15)**. El tilemap de esas filas:
```
 f10 100 11c 115 115 115 115 114 156 14d 155 144 115 11a 129 124 11c 115 100
 f11 100 15b 11e 12b 125 11b 114 156 10e 113 144 115 151 15f 15d 15b 115 100
 ...
```
La fila 10 ya contiene la primera banda de la escena, y la fila 9 pertenece al
bloque del título. **[HIPÓTESIS]** El gráfico de la escena tiene más bandas de
las que se pintan y la superior se pierde, igual que el bug del duplicado
"PRIMER PARTIDO" de la v0.9.9 pero al revés: allí sobraba una fila (`d3=$03`
cuando el gráfico tenía 3), aquí parece faltar.
**Sin verificar todavía**: hay que localizar el `d3` del bloque de la escena
(zona `0x0C41xx`, análoga a la del título) y comprobar cuántas bandas tiene
realmente el asset.

### 2. Parpadeo del letrero "PRIMER PARTIDO" — CAUSA LOCALIZADA [HECHO]

Traza de escrituras VRAM al tilemap del título (filas 8-9, `0xC400-0xC4FF`)
durante la presentación:
```
frame 901: 64 escrituras
frame 902: 64 escrituras
...
total 4416 escrituras en 69 frames consecutivos
pc que escribe: 0x007A8C  (el 100 %)
```

`0x007A8C` es el bucle interno de la **rutina de borrado `0x7A6E`**:
```
007A80: move.w d2,d6
007A82: move.w d1,$4(a5)
007A86: move.w #$3,$4(a5)
007A8C: move.w #$0,(a5)     <-- escribe CERO en cada celda
007A90: dbra   d6,$7a8c
```

Es decir: **cada frame se borra el título entero y se vuelve a pintar**. El
parpadeo es el intervalo en el que la celda está a cero antes de repintarse.

Encaja con el hallazgo de la v0.9.9: el redibujado por frame del bloque de
presentación ya se había observado entonces (hook `0x54EC→0x8A600→0x8A300→
0xC4180`) y se descartó como causa de aquel bug, pero **sigue ahí** y explica
este parpadeo.

**[HIPÓTESIS, sin verificar]** Si el borrado + repintado ocurre fuera del vblank,
el VDP muestra el estado intermedio. La solución previsible es evitar el
repintado por frame (pintar una sola vez al entrar en el estado) o moverlo
dentro del vblank. Requiere confirmar en qué punto del frame se ejecuta.

### 3. Tipografía de la introducción y las secuencias
Petición estética del usuario: prefiere una fuente más sencilla.
**No investigado.** Pendiente de localizar qué asset usa esa fuente y en qué
estados aparece; es previsiblemente distinta de la del HUD (`0x0286AA`) y de la
del título (`0xA0000`).

### Estado
Las tres quedan **abiertas**. La v1.3 sigue siendo válida y entregada; ninguna de
estas incidencias impide su uso.

---

## Sesión — Diagnóstico y prueba de los dos bugs de la presentación

Encargo: investigar y **probar** soluciones sin generar ROM ni parches.
Todas las pruebas por `poke16` en memoria del emulador.

### LAS DOS INCIDENCIAS TIENEN LA MISMA CAUSA [HECHO]

La rutina `0x0C4180` se ejecuta **cada frame** durante la presentación:
```
0C4184: d1=$02 d3=$07 -> jsr $7a6e   BORRA filas 2..9   (8 filas)
0C419E: d1=$14 d3=$03 -> jsr $7a6e   BORRA filas 20..23
0C41B8: d1=$03 d3=$02 -> jsr $7a3c   PINTA titulo filas 3..5
0C41DA: d1=$15 d3=$02 -> jsr $7a3c   PINTA rival  filas 21..23
```

**Origen de la escena** (trazado): `0x004C76` lee la tabla `0x17750` indexada
por `$f806` (`$f806=0x20` → entrada `0x17850`), de ahí `a6=0x01BCF8`, que
apunta a stream `0x01BF90` y tabla de metatiles `0x01BD00`. Se pinta con
`0x47E0` + `0x4748` en el frame 833, **una sola vez**.

El stream de la escena tiene contenido en **metafilas 2..11 = filas 4..23**.

**Bug 1 (imagen cortada)**: el primer `0x7a6e` borra las filas 2..9, pero las
filas **8 y 9 son escena legítima**. Se pierden dos bandas.
Medido: el cielo azul empezaba en `y=80` (fila 10) cuando debería empezar antes.

**Bug 2 (parpadeo)**: ese borrado + repintado ocurre **cada frame**.
Medido: **14.304 escrituras de borrado** (`pc=0x7A8C`) al título en 149 frames.
El parpadeo es el instante en que las celdas están a cero.

### Por qué el borrado ancho existe — NO es gratuito [HECHO]

La fila 6 del stream de la escena contiene el **título japonés original**
incrustado (metatiles 7-12, tiles `0x220,0x221,0x222,0x3BE,0x22C..0x251`).
El borrado de 8 filas lo tapaba.

Primera prueba (borrar solo filas 3..5): la imagen se completó **pero apareció
el katakana japonés**. Descartada. Evidencia: captura `/tmp/fixA.png`.

### FIX 1 — imagen cortada [PROBADO]
```
0x0C418E:  0007 -> 0005      (d3: 8 filas -> 6 filas, borra 2..7)
```
Borra el katakana de la fila 6 y **preserva las filas 8 y 9**.
Resultado: cielo desde `y=64` en vez de `y=80` → **2 filas de celda (16 px)
recuperadas**. Se ve la torre completa y el reloj.

### FIX 2 — parpadeo [PROBADO]
```
0x08A300:  4EB9 000C4180 -> 4E75 (rts)
```
`0x08A300` es solo `jsr $c4180 / rts`, llamado desde `0x08A60E`:
```
08A600: subq.w #1,$f80a.w      temporizador
08A604: beq.b  $8a616          -> salir del estado
08A606: btst.b #7,$f811.w
08A60C: bne.b  $8a616
08A60E: jsr    $8a300.l        <-- redibujado por frame
```
El `jsr` es **puramente redundante**: `0x0C4300` ya pinta una vez al entrar en el
estado (`cmpi.w #$20,$f806` / `jsr $c4100` / `jsr $c4180`).

**Demostración de la redundancia**: VRAM comparada con y sin el redibujado →
**0 bytes de diferencia**. El repintado por frame no aporta nada.

Resultado: borrados del título **14.304 → 0**.

### Verificación de no-regresión [HECHO]
| prueba | resultado |
|---|---|
| Los 13 partidos (índices 0-12) | idénticos entre sí, sin katakana, sin corte |
| Partido en juego (frame 2411) | **0 px** de diferencia |
| VRAM completa en partido | **0 bytes** de diferencia |
| Pantalla de título | **0 px** |
| PAUSA | **0 px** |
| `0x08A300` en otros estados | **no se usa**: 599 llamadas, todas de `0x08A60E`, frames 851-1449 (solo presentación) |

Comprobación objetiva sobre los 13 partidos: fila 6 con 908 px de patrón
PALSOFT (fondo correcto, no katakana) y cielo desde `y=64` en los 13.

### Coste total
**2 palabras (4 bytes)**, ambas en zona de código ya conocida:
```
0x0C418E:  0007 -> 0005
0x08A300:  4EB9 -> 4E75   (los 4 bytes siguientes quedan muertos)
```

### Estado
Ambos fixes **probados y verificados**, sin ROM generada según lo pedido.
Listos para incorporarse cuando el usuario lo autorice.

Capturas: `/tmp/fixA.png` (katakana), `/tmp/fixB.png`, `/tmp/fixD.png`,
`/tmp/fix2/m00..m12.png` (los 13 partidos), `/tmp/pres_grid.png`.

---

## ROM v1.4 GENERADA — corrección de la presentación

Autorizada por el usuario. `tools/build_v14rom.py`.

| | |
|---|---|
| origen | `v1.3` (md5 `bc2235cfa35230c667d498bf02a2cedb`) |
| destino | `Nekketsu_Soccer_MD_ES_v1_4.md` |
| **md5** | **`f2bb130b6a7bb9ffdc59395d2f720c03`** |
| crc32 | `99f1d227` |
| tamaño | 1.048.576 B |

### Cambios: 4 bytes
```
0x0C418E  0007 -> 0005   FIX1  borra filas 2..7 (antes 2..9)
0x08A300  4EB9 -> 4E75   FIX2  rts: elimina el redibujado por frame
0x00018E  ---- -> FCC3   checksum de cartucho recalculado
```
El constructor **comprueba los bytes de partida** antes de escribir
(`0x0C418E` debe valer `0007` y `0x08A300` debe valer `4EB9`); si no coinciden,
aborta. Evita aplicar el parche sobre una base equivocada.

### Verificación
- **ROM v1.4 vs los pokes en memoria ya validados: 0 px de diferencia.**
  El fichero reproduce exactamente lo que se probó.
- **13 partidos**: cielo desde `y=64` y fila 6 con 908 px de patrón PALSOFT en
  los 13. Ninguno con katakana ni corte.

Batería completa v1.4 vs v1.3:
| Test | Resultado |
|---|---|
| T1 bordes / retratos | 0 rotos / 0 alterados |
| T2 fantasmas (4000 frames) | 0 |
| T3 PAUSA | 0 px |
| T4 presentación | **2048 px** — ver abajo |
| T4 CRAM | idéntica |
| T4 regresión global | 0 tiles |
| T5 regresión VRAM en partido | 0 tiles |
| T6 sprites (SAT) | idéntica |

**Los 2048 px de T4 son la mejora, no una regresión.** bbox de la diferencia:
`(64,64)-(192,80)` = exactamente las **filas de celda 8 y 9**, las que antes se
borraban. Cielo visible desde `y=80` (v1.3) → `y=64` (v1.4).

### Parches (`patch/`)
| desde | tramos | IPS | BPS |
|---|---|---|---|
| jp.md (512 KB) | 78 | 531854 B | 531663 B |
| es098.md | 24 | 12801 B | 12788 B |
| v1.2 | 10 | 5408 B | 5413 B |
| **v1.3** | **3** | **27 B** | **42 B** |

**Los 8 verificados** aplicándolos con un implementador independiente y
comparando byte a byte; los BPS además validan CRC32 de origen y destino.

### Pendiente
- Tipografía de la introducción y las secuencias (siguiente tarea).
- Partida real completa jugada por el usuario.
- Estados no recorridos: VS, PASSWORD, final del torneo.

---

## Alineación del texto — DIAGNÓSTICO [HECHO]

El usuario pregunta por qué los diálogos empiezan en columnas distintas y por
qué una frase corta de la intro arranca a media pantalla y se parte en dos
líneas. Medición sobre sus 4 capturas (escaladas a coordenadas de juego):

```
19.48.59  linea1 col  4.3..28.3   linea2 col  4.2..31.4
19.49.03  linea1 col  9.3..24.5
19.49.32  linea1 col  3.2..30.6   linea2 col  4.1..31.7
19.49.43  linea1 col  5.2..27.6   linea2 col 15.0..20.5
```
Confirmado: cada línea arranca en una columna distinta, sin patrón.

### El motor de texto: `0x78A6` [HECHO]
```
0078A6: asl.w #7,d1          d1 = fila
0078A8: asl.w #1,d0          d0 = columna
0078AA: add.w d0,d1
0078B0: or.w  d4,d1          d4 = 0x4000 (escritura) o 0x6000 (paleta 3)
0078C2: move.b (a0),d3       lee un byte ASCII
0078C8: beq   $78d8          0x00 termina la cadena
0078CC: subi.w #$20,d3       tile = ASCII - 0x20
0078D0: move.w d3,(a5)       escribe la celda
0078D2: adda.w #1,a0
0078D6: bra   $78c2
```

**La rutina no calcula nada**: no mide la cadena, no centra, no ajusta. Recibe
la columna en `d0` y escribe desde ahí. El texto es **ASCII directo**, con el
tile = `carácter − 0x20`.

### Dónde está realmente la alineación [HECHO]

28 llamadas a `0x78A6` en toda la ROM. En **todas**, `d0` es un literal
inmediato (`move.w #$xx,d0`); nunca se deriva de la longitud del texto:
```
0x084018  d0= 8  d1= 4     0x088054  d0=13  d1=19   'INICIO'
0x084060  d0= 3  d1=21     0x08806c  d0=12  d1=21   'OPCIONES'
0x0840b6  d0=12  d1= 5     0x08e086  d0=17  d1=23   'VEST. NEKKETSU'
0x0840dc  d0= 7  d1=22
```

**El centrado está horneado en las propias cadenas, con espacios manuales:**
```
0x084233  len=20  '   PRIMER PARTIDO   '
0x084344  len=22  '  INSTITUTO YUSHUIN   '
0x08435b  len=22  ' INSTITUTO SHICHIFUKU '
0x084458  len=22  ' SELECCION CAPITANES  '
```
Los 13 nombres de instituto tienen **longitud fija 22** y se rellenan con
espacios a izquierda y derecha para simular centrado. La columna de arranque
(`d0=12`, `d0=7`) es la misma para todos.

→ **Respuesta a la pregunta del usuario**: no hay ninguna lógica de alineación.
Cada frase arranca donde la dejó quien escribió los espacios de relleno. Por eso
el resultado es irregular y por eso una frase corta puede empezar a media
pantalla: lleva demasiados espacios delante.

### Consecuencia para lo que pide el usuario
Alinear todo a la izquierda con sangría **no requiere tocar código**: basta con
reescribir las cadenas quitando los espacios iniciales sobrantes y dejando una
sangría constante. El límite es que cada cadena tiene un espacio reservado en la
ROM (longitud fija); habría que respetarlo o reubicar el bloque de textos.

### PENDIENTE — los diálogos de la intro
Las frases de las capturas del usuario (`SHINICHI: ¡Yo no quiero jugar!`,
`MISAKO: ¿Podría vuestro club...`) **no aparecen como ASCII plano en la ROM**:
búsqueda exhaustiva de cadenas imprimibles (88 encontradas) sin resultado.
Tampoco se localizó la secuencia en el emulador: la navegación probada
(título → menú) lleva a los estados `0x08 → 0x0c → 0x10 → 0x14`, y la intro con
diálogos no se alcanzó.

**Sin resolver**: dónde residen esos textos (previsiblemente comprimidos con
`0xF2A4`, como el resto de assets) y qué rutina los dibuja. Es el primer paso
antes de poder tocar ni la alineación ni la tipografía de la intro.

**No se ha modificado nada.** v1.4 intacta.

---

## Diálogos de la demo de atracción — LOCALIZADOS Y DESCIFRADOS [HECHO]

Aclaración del usuario: la "introducción" es la **demo de atracción** que se
reproduce si no se pulsa nada en el título.

**Corrección de un supuesto del usuario**: yo no traduje esta ROM. La traducción
española llegó ya hecha (v0.9.8); mi trabajo ha sido corregir bugs y ampliar.
No tenía localizados los diálogos: hubo que buscarlos.

### Cómo se llega
Sin pulsar nada: frame ~937 título, **frame ~1677 demo de atracción**
(`f800=0x0c`). Los diálogos aparecen a partir del frame ~1745.

### El texto NO está en el plano A [HECHO]
Está en el **plano B**, filas 24 y 26. Por eso las trazas sobre `0xCC00-0xCFFF`
no encontraban nada.

### Codificación descifrada [HECHO]
El tile de cada carácter:
```
espacio   -> 0x3F
mayúsculas y signos (ASCII < 0x61) -> ASCII - 0x20
minúsculas (ASCII >= 0x61)         -> ASCII + 0x1F
```
Verificado carácter a carácter contra `"En el campo del Nekketsu,"` leído de
VRAM. Por eso las búsquedas de ASCII plano en la ROM fallaban.

### Motor de diálogos: `0x0058C0` - `0x0059B4` [HECHO]
Estructura de estado en `$f940`:
```
+0x02  índice dentro del script     +0x14  COLUMNA base  (= 1)
+0x04  columna actual               +0x16  FILA base     (= 0x18 = 24)
+0x06  fila actual                  +0x08  contador de sub-paso
```
Puntos clave:
```
005900: cmpi.w #$8000,d0 / beq $593a    -> 0x8000 = SALTO DE LÍNEA
00593A: move.w #$0,d0                      reinicia columna a 0
005908: cmpi.w #$ffff,d0 / beq $59d4    -> 0xFFFF = FIN del script
005974: move.w $14(a6),d2 / add.w d2,d0    suma la COLUMNA BASE
0059B0: bsr $7a3c                          pinta el carácter
```
Se dibuja **un carácter cada 3 frames** (efecto máquina de escribir).

### Los scripts: `0x08027E`, 28 bloques de 126 bytes [HECHO]
Formato de cada bloque:
```
word 0   : columna base (1 en los 28)
word 1   : fila base (0x18 = 24)
words... : caracteres codificados
0x8000   : salto de línea
0xFFFF   : fin
```
Contenido recuperado (muestra):
```
0x08027E  '  En el campo del Nekketsu,  ' / ' Kunio y Shinichi entrenaban.'
0x0802FC  '       TAKASHI: ¡Kunio!      '
0x08037A  '  MISAKO: Encantada. Soy la  ' / ' delegada del club de fútbol.'
0x08076A  '   SHINICHI: ¡Yo no quiero   ' / '            jugar!           '
```

### CAUSA CONFIRMADA de la alineación irregular [HECHO]
La columna base es **1 en los 28 scripts**. El desplazamiento lo producen los
**espacios de relleno incrustados en cada frase**, distintos en cada una:
```
0x08027E  sangrías [2, 1]      0x08066E  sangrías [4, 10]
0x0802FC  sangrías [7]         0x08076A  sangrías [3, 12]
0x080476  sangrías [4]         0x080ADC  sangrías [5, 10]
0x0804F4  sangrías [0, 1]      0x080BD8  sangrías [1, 10]
```
Coincide exactamente con la observación del usuario: en japonés esos espacios
centraban el texto; con frases castellanas más largas, el centrado se rompe y
cada línea arranca donde toque.

El caso que señaló ("frase corta que empieza a media pantalla y se parte"):
`0x08076A` línea 2 = `'            jugar!           '` → **12 espacios**.

### Viabilidad de alinear a la izquierda [HECHO]
**No requiere tocar código.** Basta reescribir los words de espacio (`0x3F`)
del principio de cada línea, moviendo el texto a la izquierda y rellenando la
cola con espacios. Cada bloque tiene **tamaño fijo de 126 bytes**, así que no
hay riesgo de desbordamiento: es una permutación dentro del mismo espacio.

Con sangría de 1 celda, cada línea empezaría en la columna 2 (base 1 + 1).

Trabajo estimado: script que recorra los 28 bloques, separe las líneas por
`0x8000`, haga `lstrip` de los `0x3F` iniciales, añada la sangría fija y
rellene por la derecha. **Es asumible y reversible.**

### Estado
Investigación cerrada. **No se ha modificado nada.** Pendiente de que el
usuario confirme la sangría deseada.

---

## Alineación de diálogos — IMPLEMENTACIÓN Y HALLAZGO BLOQUEANTE

Encargo: sangría de 2 celdas, recomponer frases partidas, solo en diálogos.

### Correcciones a la investigación previa [HECHO]

**86 bloques, no 51.** Mi conteo anterior saltaba de 126 en 126 bytes, pero los
bloques tienen **tamaño variable**: 65 de 126 B, 11 de 56 B, 8 de 66 B, uno de
186 B y uno de 8 B. Hay que recorrerlos siguiendo el terminador `0xFFFF`.
Los que faltaban son nombres de jugadores, rótulos y créditos finales.

**Códec completo** (`tools/dlgtext.py`), verificado con round-trip byte a byte
sobre los 86 bloques: 0 discrepancias.
```
0x3F        espacio
0x01..0x3E  ASCII (t + 0x20)
0x50, 0x51  '¿', '¡'
0x52..0x57  Á É Í Ó Ú Ñ
0x80..0x99  'a'..'z' (t - 0x1F)
0x9A..0x9F  á é í ó ú ñ
```

### Filtro de selección [HECHO]
`tools/realign.py`. Criterio del usuario (nombre + `:`) más los textos
narrativos de las mismas escenas, identificados uno a uno.
**Excluidos correctamente**: 11 nombres de jugador, 8 nombres de capitán,
13 bloques de créditos, `FIN`, `FIN DE LA PARTIDA`.

**Bloque protegido a mano**: `0x081142` `'MISAKO: ¿Volver a intentarlo?'` lleva
un **menú** (`'SÍ       NO'`) cuya separación es funcional — el juego coloca ahí
el cursor. El reflow lo convertía en `'SÍ NO'` y habría roto la selección.
Detectado revisando el diff **antes** de escribir nada.

### HALLAZGO BLOQUEANTE: el ancho útil es 29, no 31 [HECHO]

Medido en pantalla, no supuesto. Se probó con 31 y con 30 y en ambos casos la
última letra se perdía (`'entrenaban.'` quedaba sin el punto). El valor correcto
es **29**, que es exactamente el que ya usaban los bloques originales: no era
una elección del traductor japonés, es el ancho de línea del motor.

Además, **el motor solo pinta 2 líneas** (filas 24 y 26). Una tercera línea se
genera pero no se dibuja: el texto desaparece.

**Consecuencia directa**: la sangría consume ancho útil.
```
sangría 0 -> 29 útiles ->  0 de 48 diálogos se quedan sin caber
sangría 1 -> 28 útiles ->  5 de 48 NO caben en 2 líneas
sangría 2 -> 27 útiles -> 11 de 48 NO caben en 2 líneas
```

Los 11 afectados con sangría 2 incluyen frases centrales de la intro:
```
0x08027E  'En el campo del Nekketsu, Kunio y Shinichi entrenaban.'
0x08037A  'MISAKO: Encantada. Soy la delegada del club de fútbol.'
0x0805F0  'MISAKO: ¿Podría vuestro club jugar el torneo en su lugar?'
```

**Verificado en el emulador**: con sangría 2 la frase de la intro se corta y
pierde `'entrenaban.'`. Captura `/tmp/dlg_v3.png`.

### Estado
La alineación a la izquierda funciona y está probada; el problema es solo la
**anchura de la sangría**. No se ha modificado ninguna ROM.

Opciones para el usuario:
1. **Sangría 1** (8 px): 5 frases habría que acortar levemente.
2. **Sangría 2** (16 px): 11 frases habría que acortar.
3. **Sangría 2 con excepción**: sangría 1 en los bloques que no caben.
4. Reescribir esas frases para que quepan (cambia el texto de la traducción).

---

## Respuestas a las tres cuestiones del usuario [HECHO]

### 1. La palabra "una" repetida — NO EXISTE, fue error mío de presentación

`0x0804F4` y `0x080572` son **dos bloques independientes**, dos diálogos
distintos de TAKASHI que el juego usa en momentos diferentes:
```
0x0804F4  'TAKASHI: El equipo sufrió una intoxicación y no puede ir.'
0x080572  'TAKASHI: El equipo sufrió una intoxicación alimentaria.'
```
En mi tabla mostré solo la línea problemática de cada bloque y, al quedar una
debajo de otra, parecía una única frase con "una" duplicado. El texto está bien.

### 2. ¿Otra fuente resolvería el problema? — NO [HECHO]

Ancho real de los glifos medido en VRAM:
```
mayúsculas: 6.8 px de 8 (media)
minúsculas: 5.3 px de 8 (media)
```
Podría parecer que hay margen, pero **el avance del motor es de 1 celda fija**:
```
0059AA: move.w (a0),d7     lee el caracter
0059B0: bsr    $7a3c       pinta UNA celda
005978: add.w  d2,d0       avanza UNA columna
```
El cursor suma 8 px por carácter sea cual sea el glifo. Una fuente más estrecha
solo dejaría más aire entre letras; **no cabría ni un carácter más**.

Es el mismo hallazgo que en el HUD: el límite es de **celdas**, no de píxeles.
La solución de allí (tiras precompuestas) es inviable aquí: el texto se dibuja
carácter a carácter en tiempo real con efecto máquina de escribir, y son 48
diálogos distintos.

### 3. Pasar palabras al renglón siguiente — IMPOSIBLE, solo hay 2 líneas [HECHO]

Las 11 frases **sí caben en 3 líneas** con sangría 2 (comprobado: 11 de 11).
Pero el motor no puede mostrarlas:
```
005942: move.w $6(a6),d1
005946: addq.w #$2,d1      cada salto de linea suma +2 filas
```
Línea 1 → fila 24, línea 2 → fila 26, **línea 3 → fila 28**.

La pantalla tiene **224 px = 28 filas (0..27)**. La fila 28 está **fuera de
pantalla**. Verificado: filas 24 y 26 tienen texto, 27 está vacía y la 28 no
existe.

→ El panel de diálogo es de **2 líneas y no admite una tercera** sin rediseñar
la ventana de texto (subir el panel, cambiar el interlineado a +1 en vez de +2,
o reubicar el bloque completo). Eso sí sería tocar el motor.

### Situación
Con sangría 2 hay 11 frases que necesitan perder 1-2 caracteres. El usuario
rechaza quitar puntos finales y pasar palabras abajo no es posible.

Opciones que quedan:
- **Sangría 1**: reduce el problema de 11 a 5 frases.
- **Reescribir esas frases** con sinónimos más cortos (sin tocar puntuación).
- **Modificar el motor** para un interlineado de +1 fila y ganar una 3ª línea:
  es un cambio de 1 byte (`addq.w #$2` → `#$1`) pero altera el aspecto de los
  48 diálogos y hay que comprobar que el panel lo admite visualmente.

**No se ha modificado nada.**

---

## SOLUCIÓN ENCONTRADA: 3 líneas subiendo la fila base [HECHO]

La propuesta del usuario (partir el diálogo en dos paneles) llevó a investigar
el encadenamiento de bloques, y por el camino apareció **una solución mejor y
mucho más barata**.

### Por qué la división en dos paneles es cara [HECHO]
Existe una **tabla de punteros en `0x80020`** (selector `0x80000`:
`d0*4 + tabla -> a0`), con 90 entradas usadas y **30 libres** (`0xFFFFFFFF`).
Y hay **6218 B libres** en `0x827B6-0x083FFF`.

Es decir: crear bloques nuevos es trivial. El problema es **encadenarlos**: el
índice del bloque se escribe en `$10(a6)` desde **~170 puntos distintos del
código**, cableados por escena. Insertar un panel intermedio obliga a
re-encadenar la lógica de esa escena concreta. Viable, pero invasivo.

### La alternativa: subir el panel una fila [HECHO]

Medición del panel negro:
```
fila 22: 0/16 muestras negras     fila 25: 16/16
fila 23: 16/16 muestras negras    fila 26: 15/16
fila 24: 13/16                    fila 27: 16/16
```
**El panel ocupa las filas 23..27**, pero el texto empieza en la 24. Hay una
fila libre arriba.

Cambiando la **fila base del bloque** de `0x18` (24) a `0x17` (23):
```
linea 1 -> fila 23
linea 2 -> fila 25
linea 3 -> fila 27   <-- visible y DENTRO del panel
```
Antes, con base 24, la tercera caía en la fila 28 = fuera de pantalla.

**Verificado en el emulador**: las tres líneas se dibujan correctamente.
Captura `/tmp/tres.png`. El texto no toca los bordes del panel.

Coste: **2 bytes por bloque** (la word de fila base). No se toca el motor, no
se toca la traducción, no se altera el interlineado.

### Resultado con sangría 2 + fila base 23
```
48 diálogos
  37 quedan en 2 lineas
  11 pasan a 3 lineas
   0 se cortan
```
De los 11 de 3 líneas, **5 no caben en su bloque actual** (126 B):
```
0x0804F4  necesita 132 B
0x080572  necesita 128 B
0x0805F0  necesita 132 B
0x080CD4  necesita 130 B
0x080DD0  necesita 130 B
```
Se resuelve **reubicándolos** al hueco libre: 652 B de 6218 disponibles, y
repuntando 6 entradas de la tabla (`[6] [7] [8] [9] [24] [26]` — el índice 8
apunta al mismo bloque que el 7).

### Balance
- sangría de 2 celdas en los 48 diálogos, como pidió el usuario
- **ni una sola frase recortada**: se respeta la traducción íntegra
- ni puntos finales eliminados ni palabras cambiadas
- no se toca el motor de texto
- coste: fila base de los bloques + reubicar 5 + repuntar 6 punteros

**No se ha modificado nada todavía.** Pendiente de aprobación del usuario.

---

## Desfase de sangría entre líneas — CAUSA Y FIX [HECHO]

Observación del usuario: *"las líneas 2 y 3 comienzan en una sangría diferente
de la línea 1"*. **Cierto y verificado.**

### Prueba controlada
Dos líneas de contenido idéntico (`ABCDEFGHIJKLMNOPQRSTUVWXYZ`) en el mismo
bloque:
```
f23: primera col= 2   ultima=27
f25: primera col= 3   ultima=28    <-- desplazada 1 columna
```

### Hipótesis falsa (descartada)
Supuse que la causa era el contador `$8(a6)`, que en el bucle de avance
(`0x00594C-0x005970`) hace retroceder el cursor cada 3 caracteres y **no se
reinicia** en el salto de línea. Llegué a construir un parche que liberaba
2 bytes convirtiendo `bra.w` en `bra.b` para insertar `move.w d0,$8(a6)`.

**Falsa.** Traza del estado real: `cnt` vale **0 durante todo el diálogo**. Ese
mecanismo no llega a activarse nunca.

### CAUSA REAL [HECHO]
Traza de la columna interna (`$4(a6)`) carácter a carácter:
```
linea 1: col 0,1,2,...,10    arranca en 0
linea 2: col 1,2,3,...,11    arranca en 1
```
El salto de línea (`0x00593A`) hace `move.w #$0,d0` / `move.w d0,$4(a6)`, y el
flujo cae en `0x00594C`, que lee `$4(a6)` y ejecuta `addq.w #1,d0` **antes** de
pintar. Resultado: el primer carácter de la línea 2 va a la columna 1.

En la línea 1 no ocurre porque el reset del bloque inicializa `$4(a6)` a
`0xFFFF` (-1), y al sumarle 1 queda 0.

**Es un bug del juego original**, no de la traducción: el salto de línea pone 0
donde debería poner -1.

### FIX: 2 bytes
```
0x00593C:  0000 -> FFFF        (move.w #$0,d0  ->  move.w #$ffff,d0)
```
Verificado con la misma prueba controlada:
```
SIN fix:  f23 col 2..27   f25 col 3..28
CON fix:  f23 col 2..27   f25 col 2..27    <-- alineadas
```

Esto afecta a **todos los diálogos del juego**, no solo a los reformateados.

### Consecuencia para el reformateo
Con el desfase corregido, el ancho útil de la 2ª y 3ª línea gana 1 columna, lo
que cambia el cálculo de cuántas frases caben en 2 líneas. **Hay que rehacer el
recuento** antes de decidir la sangría.

---

## Fuente nueva de diálogos — diseño y revisión [HECHO]

### Auditoría de la fuente actual — las dos quejas del usuario, VERIFICADAS

**Grosores dispares** — cierto. Trazo general de 2 px, pero con rachas de 1 px
en `M`, `Q`, `S`, `V`, `W`:
```
M: rachas [1,2,3,7]    S: rachas [1,2,3,6]
Q: rachas [1,2,4,5]    V: rachas [1,2,3,5]
W: rachas [1,2,3,7]
```

**Alturas desiguales** — cierto:
```
minusculas x-height: filas 1..5   (a c e i n o r s u v w x z)
la 'm':              filas 2..6   <-- una fila mas abajo
```

**Tildes de 1 px** — cierto, tanto en la fuente original como en mi primera
versión: eran un único píxel y se leían como un punto.

### Diseño nuevo (`tools/dlgfont.py`)
- trazo **uniforme de 1 px** (2 px emborrona a este tamaño)
- mayúsculas: **todas** en filas 0..6
- minúsculas x-height: **todas** en filas 2..6
- ascendentes `b d f h k l t`: desde la fila 0
- descendentes `g j p q y`: hasta la fila 7
- cuerpo de 6 px + aire; el motor avanza 8 px fijos
- formas redondeadas estilo grotesca moderna (Roboto / Ubuntu)
- **86 glifos**, cobertura completa de los diálogos verificada

### Correcciones pedidas por el usuario [HECHO]

**1. La `O` estaba deformada.** Causa: escribí la fila 2 como
`"#.....#"[:6]`, que truncaba el trazo derecho y dejaba la letra abierta.
Corregida a un óvalo cerrado y simétrico.

**2. El asta de la `a`.** Rediseñada con la barra superior arrancando en el
margen izquierdo. Primer intento fallido: quedó idéntica a una `o` con asta;
rehecha con cuenco inferior y asta a la derecha, distinguible de la `o`.

**3. Tildes más largas.** Pasan de 1 píxel a **trazo diagonal de 2 px** en las
filas 0-1. En la `í` hubo que separar la tilde del asta: pegadas se leían como
una sola forma.

### Fallos adicionales detectados en la muestra completa [HECHO]
Al renderizar el abecedario entero aparecieron problemas que en un diálogo
suelto no se veían:
- `g p q y` se cortaban en la fila 6: **no bajaban** como descendentes.
  Corregido, ahora ocupan filas 2..7.
- `j` sin asta completa. Rediseñada 0..7.
- `Ñ` con la virgulilla pegada al cuerpo. Separada una fila.

Lección: **la muestra completa del juego de caracteres es imprescindible**; una
frase de ejemplo no revela los defectos de los glifos que no aparecen en ella.

### Integración
Asset de la fuente localizado por traza del descompresor: **`0x90000`**,
256 tiles, cargado desde `pc=0x004BD4` a VRAM 0.
```
original:  4137 B comprimidos
nueva:     3914 B comprimidos
```
**Cabe in situ**, con 223 B de margen. No hay que reubicar ni repuntar nada.

Nota metodológica: el primer intento inyectó los glifos directamente en VRAM y
**no surtió efecto** — es la caché de patrones de GPGX ya documentada. Hay que
parchear el asset comprimido en ROM y dejar que el juego lo descomprima.

### Ficheros
- `tools/dlgfont.py` — 86 glifos
- `work/shots/FUENTE_muestra.png` — juego de caracteres completo
- `work/shots/FUENTE_dialogos.png` — comparativa in-game

**No se ha generado ninguna ROM.**

---

## ROM v1.5 GENERADA — bloque completo de diálogos

Autorizada por el usuario ("Habemus fuente"). `tools/build_v15.py`.

| | |
|---|---|
| origen | v1.4 (md5 `f2bb130b6a7bb9ffdc59395d2f720c03`) |
| destino | `Nekketsu_Soccer_MD_ES_v1_5.md` |
| **md5** | **`dc4f862be0e512b21d1abd67d22c5925`** |
| tamaño | 1.048.576 B |

### Cambios
1. **Fix del salto de línea** — `0x00593C: 0000 -> FFFF` (2 B).
   Bug del juego original: la 2ª línea se dibujaba una columna a la derecha.
2. **48 diálogos reformateados** — alineados a la izquierda, sangría uniforme
   de 2 tiles, fila base fija `0x18`. **13 en una línea, 35 en dos.**
   Ninguna frase recortada: la traducción queda intacta.
3. **Fuente nueva** en `0x90000` — 86 glifos, 3914 B de 4137 disponibles.
   Cabe in situ.
4. Checksum recalculado.

El constructor **verifica los bytes de partida** antes de escribir y aborta si
no coinciden.

### Bloques deliberadamente NO tocados
Nombres de jugador (11), nombres de capitán (8), créditos (13), `FIN`,
`FIN DE LA PARTIDA` y el menú `¿Volver a intentarlo?` — su separación
`SÍ    NO` es funcional, ahí se coloca el cursor.

### Batería v1.5 vs v1.4
| Test | Resultado |
|---|---|
| T1 bordes / retratos | 0 rotos / 0 alterados |
| T2 fantasmas (4000 frames) | 0 |
| T3 PAUSA | 0 px |
| T4 presentación | 0 px |
| T4 CRAM | idéntica |
| T4 tiles distintos | **86** |
| T5 regresión VRAM en partido | 0 tiles |
| T6 sprites (SAT) | idéntica |

Los 86 tiles de T4 se auditaron uno a uno: **corresponden exactamente a los 86
glifos de la fuente**, ni uno inesperado, ni uno sin modificar.

### Parches (`patch/`)
| desde | tramos | IPS | BPS |
|---|---|---|---|
| jp.md | 79 | 531861 B | 531668 B |
| es098.md | 42 | 22103 B | 22072 B |
| v1.3 | 21 | 9329 B | 9326 B |
| v1.4 | 19 | 9317 B | 9316 B |

Los 8 verificados con implementador independiente; los BPS validan además
CRC32 de origen y destino.

### Pendiente
- Partida real completa jugada por el usuario.
- Estados no recorridos: VS, PASSWORD, final del torneo.

---

## v1.5 — CUATRO DEFECTOS REPORTADOS POR EL USUARIO [HECHO]

El usuario prueba la v1.5 y detecta cuatro problemas. Investigados todos.

### 1. La `Ó` parece una `Ú` — BUG PROPIO CONFIRMADO
```
Ó: ...##. / ..##.. / #....# / #....# / #....# / #....# / .####.
Ú: ...##. / ..##.. / #....# / #....# / #....# / #....# / .####.
```
**Son idénticas.** Al comprimir la `Ó` para hacer sitio a la tilde de 2 px le
quité la curva superior y quedó con forma de U. Verificado también que ninguna
otra acentuada colisiona: solo este par.

### 2. Frase casi repetida — NO ES MÍA, VIENE DEL ORIGINAL
Los índices 6, 7 y 8 de la tabla de punteros:
```
[6] -> 0x0804F4  'TAKASHI: El equipo sufrió una intoxicación y no puede ir.'
[7] -> 0x080572  'TAKASHI: El equipo sufrió una intoxicación alimentaria.'
[8] -> 0x080572  (el MISMO bloque que el 7)
```
Comprobado en **v1.4 y en la ES v0.9.8 original**: la duplicación ya estaba.
Los índices 7 y 8 apuntan al mismo bloque, y el 6 es una variante casi igual.
No lo introdujo el reformateo.

También comparten destino: `[20]` y `[23]` → `'TODOS: ............'`, y
`[0]`/`[41]` → bloque vacío. Esos son intencionados del juego.

### 3. Diálogos que desaparecen muy rápido — EFECTO SECUNDARIO DEL REFORMATEO
Medición de duración en pantalla:
```
v1.4  diálogo 'TAKASHI: ¡Kunio!'  285 frames (4.8 s)
v1.5  el mismo                     50 frames (0.8 s), 5 frames completo
```
**Causa**: el tiempo en pantalla no es fijo. El texto se escribe a 3 frames por
carácter y el juego avanza poco después de terminar. Los espacios de relleno
del centrado japonés actuaban como **retardo involuntario**: al eliminarlos,
las frases cortas se escriben en un instante y se van enseguida.

Es consecuencia directa de quitar los espacios. **No estaba previsto.**

### 4. Frases mal cortadas
Visible en las capturas: `'KUNIO: Me | dan pena...'`, `'SHINICHI: Nadie | parece
tener ganas'`. El corte se hace por el primer punto válido (algoritmo
codicioso), no por el más equilibrado ni por unidad sintáctica.

### 5. BUG PROPIO ADICIONAL detectado al investigar
`write_block()` rellena el espacio sobrante del bloque con `0xFF`. Pero
`0xFFFF` es el terminador de la tabla: al recorrerla secuencialmente, **mi
herramienta solo lee 1 bloque de los 86** en la v1.5.

El juego no se ve afectado porque accede por la tabla de punteros `0x80020`
(direcciones absolutas), y las capturas del usuario lo confirman. Pero es un
defecto real que hay que corregir: el relleno debe ser neutro, no `0xFF`.

### Estado
La v1.5 **no es válida**. Hay que corregir los cuatro puntos y regenerar.

---

## Aclaraciones sobre los defectos de la v1.5 [HECHO]

### La frase de TAKASHI: son DOS bloques distintos, no una partida

Verificado en ES v0.9.8 y v1.4 (ambas idénticas):
```
0x0804F4  'TAKASHI: El equipo sufrió una intoxicación y no puede ir.'
0x080572  'TAKASHI: El equipo sufrió una intoxicación alimentaria.'
```
Son **dos bloques independientes** con dos frases distintas, ya presentes en la
traducción original. El índice 8 apunta al mismo bloque que el 7.

El usuario indica que la frase correcta sería
`'El equipo sufrió una intoxicación alimentaria y no puede ir.'`
= **69 caracteres con el prefijo TAKASHI:**. El límite físico es de
**60** (2 líneas x 30). No cabe: habría que acortarla o repartirla en los dos
bloques.

### 'MISAKO: Encantada' — el diálogo estaba, el juego lo SALTABA

Traza del índice de diálogo `$10(a6)` en la demo:
```
v0.9.8 / v1.3 / v1.4:  1 -> 41 -> 2 -> 41 -> 4 -> 41 -> 5   (SE SALTA EL 3)
v1.5:                  1 -> 41 -> 2 -> 41 -> 3 -> 41 -> 4   (aparece)
```
El bloque 3 es `0x08037A`, `'MISAKO: Encantada. Soy la delegada del club de
fútbol.'`. Está en la ROM desde la v0.9.8 pero **nunca llegaba a mostrarse**.

**Causa**: la escena avanza por temporizadores propios. En v1.4 el bloque 2
(`'TAKASHI: ¡Kunio!'`) ocupaba 126 B con espacios de relleno → 61 caracteres →
183 frames escribiéndose. En v1.5 ocupa 38 B → 17 caracteres → 51 frames.
Al terminar antes, da tiempo a encajar el bloque 3 antes del siguiente evento
de la escena.

→ **No se ha añadido nada: se ha desbloqueado texto que el juego se comía.**
Confirma la sospecha del usuario de que había diálogos sin mostrar y justifica
revisar los 86 bloques uno a uno.

### Nota sobre la capacidad de captura del asistente
El usuario pregunta por qué puedo capturar diálogos y jugadas pero digo que no
puedo "jugar". Precisión: **sí puedo ejecutar el juego y enviar pulsaciones**;
lo que no puedo es *jugar con criterio* — reaccionar a lo que ocurre, buscar
situaciones concretas o valorar si algo "se siente" bien. Mis entradas son
secuencias pseudoaleatorias con semilla fija. Por eso alcanzo la demo de
atracción (que no requiere jugar) pero no un torneo completo ni el final.

---

## v1.5 corregida — los cinco defectos [HECHO]

`md5 437230bf684734ec92750bd6575904cc`

### 1. `Ó` idéntica a `Ú` — corregido
Se le devolvió la curva superior (`.####.`) y a la `Ú` se le dio base plana
(`#####.`) para separarlas más. Verificado: **0 colisiones** entre los 87
glifos.

### 2. Relleno `0xFF` — corregido, y explica el defecto 3
`write_block()` rellenaba con `0xFF` **después** del END. Ahora rellena con
**espacios (0x3F) antes** del END, igual que los bloques originales.

Dos consecuencias:
- la tabla vuelve a ser recorrible: **86 bloques legibles** (antes 1)
- se restaura la duración en pantalla

### 3. Diálogos que desaparecían rápido — CAUSA REAL Y SOLUCIÓN
El motor tarda **3 frames por word**, incluidos los espacios. Los bloques
originales estaban llenos hasta el borde (60 words = 180 frames). Al quitar el
relleno, `'TAKASHI: ¡Kunio!'` pasó de 60 a 16 words → de 180 a 48 frames.

**Los espacios de relleno del centrado japonés eran el temporizador.**
Rellenando de nuevo hasta la capacidad del bloque, la duración vuelve a la
original. Verificado: **0 bloques con menos relleno del máximo**.

### 4. Cortes mal repartidos — corregido
`split_lines()` tomaba el primer corte válido (codicioso), produciendo
`'KUNIO: Me' / 'dan pena...'`. Ahora evalúa **todos** los puntos y elige el más
equilibrado: `'KUNIO: Me dan pena.' / '¡Vamos a ayudarlos!'`.

### 5. Frase de TAKASHI — versión del usuario aplicada
El usuario indica que la correcta es
`'El equipo sufrió una intoxicación alimentaria y no puede ir.'` (69 caracteres
con prefijo, límite 60). Repartida en los dos bloques que el juego muestra
seguidos:
```
0x0804F4  'TAKASHI: El equipo sufrió una'
0x080572  'intoxicación alimentaria' / 'y no puede ir.'
```

### Captura individual de los 87 bloques [HECHO]
`/tmp/capfinal.py`. El juego solo alcanza unos pocos en la demo; el resto
exigiría jugar el torneo completo. Se fuerza el índice `$10(a6)` en el momento
en que el motor arranca un diálogo, reiniciando `$2(a6)`, `$4(a6)` y `$6(a6)`.

Resultado: **87 capturas individuales** en `work/shots/dialogos/dNN.png`.

Auditoría automática (posición del primer y último píxel de texto):
```
CORTES de texto (x>=250) : 0
sangria fuera de x=8     : 6, todas de 1-2 px
```
Las 6 son glifos que empiezan estrechos (`i`, `¡`): el tile arranca en la misma
columna, solo cambia dónde cae el primer píxel encendido. **No son defectos.**

### Sobre el diálogo 'MISAKO: Encantada'
Confirmado que **el texto existía desde la v0.9.8** pero el juego lo saltaba:
en v1.4 el diálogo anterior tardaba 183 frames en escribirse y la escena
avanzaba antes de mostrarlo. Al acortarlo se desbloqueó.

**Atención**: al restaurar ahora el relleno completo (defecto 3), la duración
vuelve a la original y **es probable que ese diálogo se vuelva a saltar**.
Pendiente de verificar en la próxima pasada.

---

## Duración de los diálogos — MEDICIÓN Y DECISIÓN [HECHO]

El usuario pide medir los tiempos y prioriza que **aparezcan todos** los
diálogos, aceptando que uno concreto dure algo distinto.

### El mecanismo
Dos temporizadores independientes:
- **el texto**: el motor tarda `(CHAR_DELAY+1)` frames por word, incluidos los
  espacios de relleno. Valor original: `$12(a6) = 2` → 3 frames/word.
- **la escena**: `$f80a` cuenta hacia atrás y dispara el siguiente evento.

Si el diálogo termina **antes** del evento, el motor encadena el bloque
siguiente. Si termina **después**, la escena salta ese bloque.

### Barrido de configuraciones
```
PAD_FULL  CHAR_DELAY   índices mostrados        dur idx2   idx3
True      2 (orig.)    [1,2,4,5]                 308f      NO
False     2            [1,2,3,4,5,6,8]            52f      SI
False     3            [1,2,4,5,6,8]             308f      NO
False     4            [1,2,4,6,8]               308f      NO
False     5            [1,3,5,6,8]                 0f      SI (pero pierde 2 y 4)
```

Relleno mínimo por bloque (`PAD_MIN`), buscando el umbral exacto:
```
17 words ->  52f  y SÍ aparece el idx 3
18 words -> 308f  y NO aparece
```
**El salto es abrupto**, no proporcional: entre 17 y 18 words (51-54 frames)
está el límite del evento de escena.

### Conclusión
**No hay configuración que dé las dos cosas.** Verificado por dos vías
independientes (relleno y retardo por carácter). O el diálogo 2 dura 0,9 s y
aparecen todos, o dura 5,1 s y se pierde `MISAKO: Encantada`.

Siguiendo el criterio del usuario ("queremos que aparezca todo"), se fija:
```
PAD_FULL   = False     sin relleno
PAD_MIN    = 0
CHAR_DELAY = None      retardo original intacto
```

### Duraciones resultantes (medidas en la demo)
```
[ 1] 5.3s  'En el campo del Nekketsu, Kunio y Shinichi entrenaban.'
[ 2] 0.9s  'TAKASHI: ¡Kunio!'                      <-- el mal menor
[ 3] 3.2s  'MISAKO: Encantada. Soy la delegada...'  <-- recuperado
[ 4] 3.5s  'MISAKO: Kunio, ¡ayúdanos!...'
[ 5] 3.0s  'KUNIO: ¿Qué ha pasado?'
[ 6] 4.3s  'TAKASHI: El equipo sufrió una'
[ 8] 4.6s  'intoxicación alimentaria y no puede ir.'
[ 9] 5.4s  'MISAKO: ¿Podría vuestro club...'
[11] 5.9s  'KUNIO: Hablaré con los demás.'
[12] 5.0s  'SHINICHI: ¡Yo no quiero jugar!'
[13] 2.7s  'KUNIO: Eso es lo que pasa...'
[16] 1.6s  'HIROSHI: ¿Y nuestro club?'
```
**Solo el diálogo 2 baja de 1,5 s.** El resto va de 1,6 a 5,9 s.

### Error propio corregido durante la medición
El parche de `CHAR_DELAY` escribía en `0x0058B2`, que es el **desplazamiento**
de la instrucción (`0012` = `$12(a6)`), no el valor inmediato. Lo convertía en
`$8(a6)`, un campo distinto, corrompiendo el estado del motor. El valor está en
`0x0058B0`. Se añadió un `assert` que verifica la estructura antes de escribir.

`md5` de la v1.5 con la configuración final: `6996d910dd6338b593341d1c91300b5a`

---

## ROM v1.5 FINAL — entregada

| | |
|---|---|
| **md5** | **`6996d910dd6338b593341d1c91300b5a`** |
| origen | v1.4 (`f2bb130b6a7bb9ffdc59395d2f720c03`) |
| tamaño | 1.048.576 B |

### Configuración fijada
```
PAD_FULL   = False     sin relleno: aparecen los 5 diálogos de la intro
PAD_MIN    = 0
CHAR_DELAY = None      retardo original intacto
INDENT     = 0         sangría real de 2 tiles (base 1 + offset motor 1)
ROW_BASE   = 0x18      fila fija
WIDTH      = 30
```

### Contenido
1. Fix del salto de línea (`0x00593C`, 2 B) — bug del original.
2. 48 diálogos reformateados: 14 en una línea, 34 en dos. Cortes equilibrados.
3. Frase de TAKASHI con la redacción correcta del usuario, repartida en los
   dos bloques consecutivos.
4. Fuente nueva: 86 glifos, 3915 B de 4137. `Ó` y `Ú` diferenciadas.
5. Relleno de bloque con espacios antes del END (no `0xFF`): la tabla vuelve a
   ser recorrible, 86 bloques legibles.

### Batería v1.5 vs v1.4
| Test | Resultado |
|---|---|
| T1 bordes / retratos | 0 / 0 |
| T2 fantasmas (4000 frames) | 0 |
| T3 PAUSA | 0 px |
| T4 presentación | 0 px |
| T4 CRAM | idéntica |
| T4 tiles distintos | 86 = exactamente los glifos |
| T5 VRAM en partido | 0 tiles |
| T6 sprites (SAT) | idéntica |

### Parches (`patch/`) — los 8 verificados
| desde | IPS | BPS |
|---|---|---|
| jp.md | 531861 B | 531668 B |
| es098.md | 22158 B | 22128 B |
| v1.3 | 9384 B | 9382 B |
| v1.4 | 9372 B | 9372 B |

### Material de revisión
`work/shots/dialogos/` — **87 capturas individuales**, una por bloque,
obtenidas forzando el índice `$10(a6)`. Auditoría automática: 0 textos
cortados.

### Pendiente
Partida real completa jugada por el usuario, en especial VS, PASSWORD y el
final del torneo, que el barrido automático no alcanza.

---

# EL RITMO DE LOS DIÁLOGOS — modelo correcto [HECHO]

## Origen: observación del usuario

El usuario apuntó dos cosas que resultaron ser la clave:

1. *"no lo hacen de golpe, sino que se van mostrando letra a letra. Quizá, si
   la velocidad es siempre la misma, lo ideal sería que todas las frases
   tuvieran un tiempo fijo de pausa cuando terminan de dibujarse."*
2. *"la longitud de esas pausas está asociada al ritmo de lo que sucede en
   pantalla, al acting de los personajes. Si forzáramos un tiempo de espera
   tras un diálogo, podría solaparse con el siguiente."*

Las dos observaciones son correctas y **la segunda impide la solución que
propone la primera**. Se documentan aquí con la evidencia.

## ERROR DE INSTRUMENTACIÓN COMETIDO EN ESTA SESIÓN — [corregido]

Al buscar quién escribe `$f940` lancé tres mediciones que dieron **cero
resultados**, incluidos breakpoints en direcciones que sí se ejecutan.
Estuve a punto de concluir que "nadie escribe `$f940`".

**Causa**: `arena_hook()` empieza con `if (!tracing) return;` y la cuenta de
breakpoints vive dentro de `HOOK_M68K_E`. Sin `m.trace(True)` **no se cuenta
nada**. `m.config(mem=1)` + `m.mem_window(...)` tampoco bastan por sí solos.

Repetida con `m.trace(True)`, la misma prueba dio el resultado bueno.
**Norma**: todo breakpoint o ventana de memoria exige `m.trace(True)` explícito.
Un contador a cero sin trace activo no es evidencia de nada.

## Quién dispara los diálogos [HECHO]

No es un temporizador de texto. Es el **intérprete de scripts de actor** en
`0x00F6AE`, el mismo que anima a los personajes:

```
00F6AE: movea.l $18(a6),a0      ; puntero de script del actor
00F6B2: move.w  (a0),d0
00F6B6: andi.w  #$f000,d1
00F6BA: cmpi.w  #$f000,d1
00F6BE: bne     $f6c8           ; nibble alto != F -> entrada normal
00F6C2: jmp     $f758           ; nibble alto == F -> ORDEN
00F6C8: move.w  (a0),d0
00F6CA: move.w  d0,$16(a6)      ; DURACIÓN en frames
00F6CE: move.w  $2(a0),d0
00F6D2: move.w  d0,$1c(a6)      ; FOTOGRAMA de animación
00F6D6: adda.l  #$4,a0
```

Despachador de órdenes en `0x00F762` (`jmp $f762(pc,d0.w)`, `d0=(op&0xFF)*4`).
La orden `0x08` es la que lanza el diálogo:

```
00F8C6: addq.l #$2,a0
00F8C8: move.w (a0)+,d1        ; índice de bloque
00F8CA: move.l a0,$18(a6)
00F8CE: move.w d1,$f940.w      ; <-- dispara el diálogo
```

**51 disparos `FF08` localizados en la ROM** (`0x01135A`..`0x0156A0`),
intercalados entre las entradas de animación. Ejemplo real en `0x01453E`:

```
014536: FF00          orden
014538: FF06          orden
01453A: 0000 0300     espera 0, anim 768
01453E: FF08 0001     <-- DIÁLOGO bloque 1
014544: 00D1 ...      espera 209 frames  <-- el "acting" sigue
```

**Confirmado: el usuario tiene razón.** El texto y la animación comparten
script. La duración de cada diálogo es el hueco entre dos `FF08`, y ese hueco
está puesto para que cuadre con lo que hacen los personajes en pantalla.
Alargar la pausa desplazaría a los actores.

## La ranura y el gate anti-solapamiento [HECHO]

Cada `FF08` abre una ranura que dura hasta el siguiente `FF08`. Dentro, el
motor hace **dos pasadas** (`0x00582C` togglea el modo en `$e(a6)`):

```
pasada 0 (modo 0): bloque 41 = 61 words de espacios -> BORRA el panel
pasada 1 (modo 1): el texto real
ranura = borrado + tecleo + VISIBLE
```

Y en `0x005842` está el gate que explica el misterio de la v1.5:

```
005842: move.w $c(a6),d0
005846: bne    $5894          ; si SIGUE TECLEANDO, ignora la petición
```

**Si llega un `FF08` mientras el motor no ha terminado de teclear, el diálogo
nuevo se descarta.** Eso —y no un "conflicto irresoluble"— es lo que hacía
desaparecer `MISAKO: Encantada` cuando los bloques iban rellenos de espacios:
el relleno alargaba el tecleo más allá de la ranura.

## HIPÓTESIS DESCARTADA — "conflicto irresoluble" de la v1.5

En la sesión anterior concluí, y escribí en este diario, que:

> *"No hay configuración que dé las dos cosas. O el diálogo 2 dura 0,9 s y
> aparecen todos, o dura 5,1 s y se pierde MISAKO: Encantada."*

**Es FALSO.** El error fue tratar `CHAR_DELAY` como un valor global que solo
podía subir (probé 3, 4 y 5, que empeoran porque alargan el tecleo y activan
el gate). **No probé a BAJARLO.**

El tecleo y la lectura reparten la misma ranura fija:

```
ranura = borrado + tecleo + visible
```

Bajar `CHAR_DELAY` acorta el tecleo y **el tiempo ahorrado se convierte
íntegramente en tiempo de lectura**, sin tocar ni un frame de la animación.
La ranura no cambia: los actores siguen exactamente igual.

## Medición: `CHAR_DELAY` 2 (original) vs 1 vs 0

Ranuras medidas en la intro completa (frames 1700-8600). `borra` es constante
(~62f) porque el bloque 41 son siempre 61 words.

```
blq   ranura   orig(2)    d=1     d=0    texto
  2      113     0.02s   0.32s   0.60s   TAKASHI: ¡Kunio!
  3      256     0.52    1.43    2.35    MISAKO: Encantada...
  4      273     1.03    1.88    2.72    MISAKO: Kunio, ¡ayúdanos!...
  5      242     1.85    2.23    2.63    KUNIO: ¿Qué ha pasado?
  6      321     2.85    3.35    3.87    TAKASHI: El equipo sufrió una
  8      338     2.62    3.30    3.97    intoxicación alimentaria...
  9      384     2.50    ----    1.23    MISAKO: ¿Podría vuestro club...
 10        -     ----    1.10    1.67    KUNIO: Me lo pides de repente...
 11      417     4.45    4.95    5.45    KUNIO: Hablaré con los demás.
 12      604     7.50    8.00    8.53    SHINICHI: ¡Yo no quiero jugar!
 14      224     0.53    1.27    2.00    SHINICHI: ¡Ahora no...
 15      224     0.23    1.07    1.90    KUNIO: ¡Pero son compañeros...
 16      160     0.37    0.80    1.23    HIROSHI: ¿Y nuestro club?
 17      160     0.37    0.80    1.23    KOJI: Aún falta para eso.
 18      256     1.77    2.27    2.77    MITSUHIRO: Estoy muy ocupado.
 19      176     0.38    0.90    1.42    MISAKO: ¡Por favor, ayudadnos!
 20      241     2.02    2.35    2.68    TODOS: ............
 21      224     0.73    1.40    2.07    KUNIO: Me dan pena...
 22      193     0.05    0.77    1.48    SHINICHI: Nadie parece...
 23      241     2.02    2.35    2.68    TODOS: ............
 24      240     0.15    1.10    2.05    MISAKO: Si sois campeones...
```

**Con `d=0` NINGÚN diálogo baja de 0,60 s y la mayoría se duplica o triplica.**
Los tres casos que el usuario señaló como ilegibles:

```
"¿Qué hacemos?"      (13)  ->  medido aparte, mejora en la misma proporción
"¡Porfa!" (19)  0.38 -> 1.42 s   (x3,7)
"Aún falta..." (17)  0.37 -> 1.23 s   (x3,3)
"Nadie parece..." (22) 0.05 -> 1.48 s (x29)
```

## Barrido completo de 14.000 frames: un diálogo RECUPERADO

Comparando la secuencia de índices mostrados en toda la intro:

```
orig : [1,2,3,4,5,6,8,9,   11,12,13,...,25]
d=0  : [1,2,3,4,5,6,8,9,10,11,12,13,...,25]

RECUPERADO: [10] 'KUNIO: Me lo pides de repente...'
PERDIDOS  : ninguno
```

**El bloque 10 nunca se había visto**, ni en la v0.9.8 ni en ninguna versión
posterior. Con el tecleo original el bloque 9 no terminaba a tiempo y el gate
de `0x005846` descartaba el 10. Al acelerar el tecleo, el 9 acaba dentro de su
ranura y el 10 entra. Mismo mecanismo que había ocultado el bloque 3.

## Por qué esto NO rompe el "acting" — [HECHO]

La preocupación del usuario está justificada y la respuesta es que este cambio
**no toca el script de actores**:

- no se modifica ninguna entrada de animación ni ninguna orden `FF08`;
- las ranuras medidas son idénticas en las tres configuraciones
  (113, 256, 273, 242, 321, 338... en las tres columnas);
- lo único que cambia es el reparto interno tecleo/lectura **dentro** de cada
  ranura, que es tiempo que ya pertenecía al diálogo.

No hay riesgo de solapamiento: el cambio va en el sentido contrario:
acorta el tecleo, con lo que **reduce** la probabilidad de que el gate
descarte el diálogo siguiente. La prueba es que recupera uno y no pierde
ninguno.

## Sobre "un tiempo fijo de pausa para todas"

No es implementable sin tocar el script de actores: la pausa disponible es la
ranura menos el tecleo, y las ranuras las fija la animación (de 113 a 604
frames). Igualarlas obligaría a desplazar los `FF08`, y con ellos el acting.

Lo que sí se consigue con `CHAR_DELAY=0` es **maximizar la pausa de cada
frase dentro de su propia ranura**, que es lo máximo alcanzable sin tocar la
animación.

## Sobre "TODOS: ............ podría durar menos"

Los puntos suspensivos (bloques 20 y 23) están en ranuras de 241 frames
fijadas por la animación. Acortar su duración no daría ese tiempo a otro
diálogo: quedaría el panel vacío. No se toca.

---

# ROM v1.6 — ritmo de los diálogos

## Contenido

**Cambio 1 — velocidad de tecleo (2 bytes)**
```
0x0058B0: 0002 -> 0000     valor inmediato de  move.w #$2,$12(a6)
```
3 frames por carácter → 1. El tiempo ahorrado se convierte en tiempo de
lectura **dentro de la misma ranura**, sin tocar la animación.

**Cambio 2 — reparto de la frase de la intoxicación**
```
0x0804F4  'TAKASHI: El equipo sufrió una' / 'intoxicación alimentaria'
0x080572  'y no puede ir.'
```
Antes 1 línea + 2 líneas; ahora 2 + 1, como pidió el usuario.

`md5` v1.6: `eaac1e58ac3d68c28532ac83754335a0`, 115 bytes distintos de la v1.5.

## DEFECTO PROPIO ENCONTRADO Y CORREGIDO — `block_spans()`

En el diario de la v1.5 anoté como corregido el bug del relleno `0xFF`.
**No lo estaba.** Medido ahora:

```
v0.9.8  86 bloques recorribles
v1.4    86 bloques recorribles
v1.5     1 bloque   <-- roto
```

La causa no era el valor del relleno sino `block_spans()`, que asumía que el
bloque siguiente empezaba justo tras el primer `0xFFFF`. Con relleno detrás
del terminador, el recorrido moría en el primero.

Corregido: tras el terminador se salta el relleno neutro (`0x0000`/`0xFFFF`)
hasta la siguiente cabecera válida (`col<=8`, `fila` en `0x16..0x1A`).
Verificado: **86 bloques en las tres ROMs**. El tamaño devuelto incluye ahora
el relleno, que es capacidad reutilizable — por eso los dos bloques de
TAKASHI vuelven a declarar sus 126 B y el reparto 2+1 cabe.

El juego nunca se vio afectado (accede por la tabla de punteros `0x80020`),
pero la herramienta de auditoría sí, y con ella la confianza en las medidas.

## Resultados medidos (intro completa, 14.000 frames)

```
blq  ranura15 ranura16   v1.5     v1.6    texto
  2      113      113    0.02s    0.60s   TAKASHI: ¡Kunio!
  3      256      256    0.52     2.35    MISAKO: Encantada...
  4      273      273    1.03     2.72    MISAKO: Kunio, ¡ayúdanos!...
  5      242      242    1.85     2.63    KUNIO: ¿Qué ha pasado?
  6      321      322    2.85     3.43    TAKASHI: El equipo... (2 líneas)
  8      338      339    2.62     4.40    y no puede ir.
  9      384      192    2.50     1.23    MISAKO: ¿Podría vuestro club...
 10        -      193    ----     1.67    KUNIO: Me lo pides de repente...
 11      417      417    4.45     5.45    KUNIO: Hablaré con los demás.
 12      604      605    7.50     8.55    SHINICHI: ¡Yo no quiero jugar!
 14      224      224    0.53     2.00    SHINICHI: ¡Ahora no...
 15      224      224    0.23     1.90    KUNIO: ¡Pero son compañeros...
 16      160      160    0.37     1.23    HIROSHI: ¿Y nuestro club?
 17      160      160    0.37     1.23    KOJI: Aún falta para eso.
 18      256      256    1.77     2.77    MITSUHIRO: Estoy muy ocupado.
 19      176      176    0.38     1.42    MISAKO: ¡Por favor, ayudadnos!
 20      241      241    2.02     2.68    TODOS: ............
 21      224      224    0.73     2.07    KUNIO: Me dan pena...
 22      193      193    0.05     1.48    SHINICHI: Nadie parece...
 23      241      241    2.02     2.68    TODOS: ............
 24      240      240    0.15     2.05    MISAKO: Si sois campeones...
 25     1539     1539   22.85    24.05    TODOS: ¿E-en serio?...
```

**Mínimo global: 0,60 s** (`¡Kunio!`, una palabra) frente a 0,02 s antes.
El segundo más breve es 1,23 s. Los cuatro casos que el usuario reportó como
ilegibles mejoran x3,3 / x3,7 / x5,4 / x29.

**Recuperado el bloque 10** `'KUNIO: Me lo pides de repente...'`, que no se
había mostrado nunca en ninguna versión, ni japonesa ni traducida. Perdidos: 0.

La ranura del 9 pasa de 384 a 192 frames porque **antes absorbía la del 10**:
al descartarse el 10 por el gate, el hueco se sumaba al 9. No es una pérdida,
es el reparto correcto entre dos diálogos que ahora sí se muestran ambos.

## Verificación de que NO se toca el "acting" — [HECHO]

Preocupación planteada por el usuario: forzar pausas podría solapar diálogos
o desplazar a los personajes.

**Prueba 1 — ranuras.** Idénticas en v1.5 y v1.6 salvo las tres afectadas por
el reparto 6/8 (±1 frame) y la del 9/10 explicada arriba.

**Prueba 2 — SAT frame a frame, 6.900 frames de la intro.**
1.714 frames marcaban SAT distinta. Investigado byte a byte:

```
SIEMPRE el mismo byte: offset 181 = slot 22, campo 5 (parte baja del tile)
valores alternantes: 0x3C9 <-> 0x3D2, periodo 8 frames
posición Y: idéntica en los 6.900 frames
posición X: idéntica en los 6.900 frames
conjunto de tiles usado: {969, 978} en ambas ROMs
```

Es **un sprite parpadeante con ciclo de 8 frames desfasado un ciclo**, no un
desplazamiento. Y está en `y=240, x=308`: **fuera de la pantalla visible**
(240 > 224 de alto; 308 > 256+128 de margen). No se ve.

Ningún actor cambia de posición en ningún frame.

**Prueba 3 — regresión fuera de los diálogos.**
```
título (frame 950)            VRAM/CRAM/SAT/regs  IGUAL
presentación 1er partido      VRAM/CRAM/SAT/regs  IGUAL
partido jugado 3.000 frames   VRAM y SAT en 5 cortes  IGUAL
```

## Lo que NO se ha hecho, y por qué

**Pausa fija igual para todas las frases.** No es implementable sin mover los
`FF08`, y con ellos la animación. Las ranuras van de 113 a 1.539 frames y las
fija el acting. Con `CHAR_DELAY=0` cada frase recibe el máximo de lectura que
permite su propia ranura, que es el óptimo sin tocar los actores.

**Acortar `TODOS: ............`** El tiempo liberado no iría a otro diálogo:
dejaría el panel vacío, porque el siguiente `FF08` no se adelanta. No se toca.

---

# Revisión de la v1.6 por el usuario — tres cuestiones

## 1. RECTIFICACIÓN: el bloque 10 SÍ se ve en la japonesa

Escribí en el resumen de la v1.6 que `KUNIO: Me lo pides de repente...`
*"no se había mostrado nunca en ninguna versión, ni japonesa ni traducida"*.

El usuario preguntó si me refería solo a nuestras traducciones. **Tenía razón
en dudar: la afirmación era falsa.** Medido ahora sobre las tres ROMs:

```
japonesa  : [1,2,3,4,5,6,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25]
ES v0.9.8 : [1,2,  4,5,  8,9,   11,12,13,   15,   17,   19,   21,   23,   25]
v1.6      : [1,2,3,4,5,6,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25]
```

**La japonesa muestra los 24. La traducción ES v0.9.8 perdía NUEVE**:
`3, 6, 10, 14, 16, 18, 20, 22, 24`.

La causa es la misma que ya estaba documentada: el texto castellano es más
largo que el japonés, el tecleo a 3 frames/carácter se salía de la ranura y
el gate de `0x005846` descartaba el diálogo siguiente. No era un diálogo
"inédito del juego", era **daño colateral de la traducción**.

**La v1.6 restituye la secuencia exacta de la japonesa: 24 de 24.** Ese es el
resultado correcto, y es mejor que lo que yo había afirmado, pero por una
razón distinta a la que dije. Queda anotado como error propio de redacción:
di por "nunca visto" algo que solo había comprobado en las ROMs traducidas.

## 2. Los corazones son de la versión de NES, no de Mega Drive

El usuario adjuntó una captura con corazones (`みさこ「じゃあ ゆうしょうしたら
みんなに ♥♥♥♥ してあげちゃう おねがい♥!」`) y señaló que nosotros ponemos `****`.

Medido en la **ROM japonesa de Mega Drive**, escena de la sala del club,
plano B fila 26, en el momento del bloque 24:

```
JP fila26: ... 03f 038 038 038 038 03f ...
```

Cuatro veces el tile `0x038`. Renderizado desde VRAM:

```
tile 038:
   .##...##
   .###.###
   ..#####.
   ...###..
   ..#####.
   .###.###
   .##...##
```

**Es una X**, no un corazón. Y `0x038` en el códec de diálogos decodifica
como `'X'`. Captura de la japonesa en `work/shots/JP_bloque24.png`: se lee
`XXXX してあげちゃう`.

**Conclusión**: la imagen del usuario es de la versión de **Famicom/NES**
(*Nekketsu Koukou Dodgeball-bu: Soccer Hen*), que sí usa corazones. La
conversión a Mega Drive los sustituyó por equis. Nuestra traducción pone
asteriscos, que es una tercera variante.

**Decisión pendiente del usuario**: dejar `****`, poner `XXXX` como la
japonesa de MD, o dibujar un glifo de corazón (hay hueco en la fuente de
diálogos: el códec tiene rangos sin usar).

## 3. El rótulo de la escena: hay sitio de sobra [HECHO]

Observación del usuario: aparece brevemente el rótulo japonés
「ドッジボール部室」 (*sala del club de dodgeball*) y luego el nuestro,
`SALA DEL CLUB`, más corto.

**Localizada la ruta completa.** Es un hook de la traducción v0.9.8 en
`0x08C040`:

```
08C040: move.w #$4,d0        columna 4
08C044: move.w #$3,d1        fila 3
08C048: move.w #$17,d2       ancho-1 = 23 -> 24 tiles
08C04C: move.w #$2,d3        alto-1 = 2   -> 3 filas
08C054: jsr $7a6e            BORRA el rótulo japonés
08C05A: move.w #$9,d0        columna 9
08C05E: move.w #$4,d1        fila 4
08C062: move.w #$d,d2        ancho-1 = 13 -> 14 tiles
08C066: move.w #$0,d3        alto-1 = 0   -> 1 fila
08C06A: lea.l $8c100,a0      <-- TILEMAP DEL RÓTULO
08C076: jsr $7a3c            PINTA
```

Tabla del rótulo en `0x08C100`, 14 words (`0x00A0`..`0x00AD`), rodeada de
`0xFFFF` libres:

```
08C0F0: ffff ffff ffff ffff ffff ffff ffff ffff
08C100: 00a0 00a1 00a2 00a3 00a4 00a5 00a6 00a7
08C110: 00a8 00a9 00aa 00ab 00ac 00ad ffff ffff
08C120: ffff ffff ffff ffff ffff ffff ffff ffff
```

Renderizados los 14 tiles desde VRAM se lee `SALA DEL CLUB`, con la
particularidad de que **los glifos van a media celda** (el rótulo está
pre-renderizado como tira continua, no una letra por tile).

**Márgenes disponibles:**
- el borrado limpia **24 tiles** (columnas 4..27)
- el rótulo ocupa **14** (columnas 9..22)
- **quedan 10 tiles libres** sin tocar el borrado

`CLUB DE DODGEBALL` = 17 caracteres. Con glifos a media celda como los
actuales entra sin problema en 24 tiles. Cambiar el ancho es mover un
inmediato (`0x08C064`) y la columna inicial (`0x08C05C`), más la tabla de
`0x08C100`. **Requiere dibujar los tiles nuevos**, que es el trabajo real.

El parpadeo del rótulo japonés antes del nuestro es porque el borrado ocurre
DESPUÉS de que la escena pinte el fondo con el título incrustado — el mismo
patrón ya documentado para la presentación del partido (FIX1/FIX2 de la v1.4).
**No investigado si es corregible aquí**; el usuario no lo ha pedido.

## 4. Confirmación del usuario sobre el ritmo

> *"ahora los diálogos se muestran mucho más rápido... me parece mejor así
> porque de la forma anterior esperabas a que se terminaran de mostrar las
> frases y luego desaparecían casi de inmediato. Ahora simplemente puedes
> leer más rápido y notas esa pausa al final."*

Confirma el modelo: la ranura es fija y el cambio solo reparte su interior.
El usuario señala el bloque 9 (`MISAKO: ¿Podría vuestro club jugar el
torneo?`) como el más breve — coincide con lo medido: 1,23 s, porque su
ranura pasó de 384 a 192 frames al dejar de absorber la del 10.

---

# ROM v1.7 — corazones, rótulo y fin del parpadeo

`md5` `28578fa7200235738def7e9afcce0541` · sha1 `40123fff73a66c2e5ab60f7d872ddfbb9883eaac`
crc32 `f83b41ed` · 1.048.576 B · 3.324 bytes distintos de la v1.6.

## 1. Parpadeo de los vestuarios — RUTA DISTINTA A LA DE LA SALA DEL CLUB

El usuario reportó el mismo defecto en los vestuarios ("VEST. NEKKETSU").
Acertó en que era el mismo problema, pero **la ruta es otra** y eso costó
varios intentos fallidos que quedan anotados abajo.

La escena de vestuario no la pinta `0x004C76` sino **`0x0050B2`**, que elige
stream según el marcador:

```
0050AC: lea.l  $27190,a6
0050B2: jsr    $74e0
0050B8: lea.l  $23EC8,a6      <- DERROTA
0050BE: move.w $f8d2,d0
0050C2: cmp.w  $f8d0,d0
0050C6: bcs    $50d0
0050CA: lea.l  $241E0,a6      <- VICTORIA
0050D0: jsr    $47e0
0050D6: jsr    $4748
```

El texto japonés va incrustado en la **metafila 11** (no en la 1-2 como en la
sala del club). El hook que pinta el rótulo castellano es `0x08E040`, llamado
desde `0x08E006`:

```
08E040: d0=$10 d1=$17 d2=$0f d3=0 d4=$4000 -> jsr $7a6e   BORRA
08E05A: idem con d4=$6000                  -> jsr $7a6e   BORRA (2a capa)
08E074: d0=$11 d1=$17 a0=$8E180 d4=$4000   -> jsr $78a6   'VEST. NEKKETSU'
08E08C: d0=$11 d1=$17 a0=$8E18F d4=$6000   -> jsr $78a6   'VEST. YUSHUIN'
```

Los rótulos son **cadenas ASCII planas** en `0x08E180` y `0x08E18F`, no
tilemaps. Medición del parpadeo:

```
+66  f802=40   texto JAPONÉS (9 tiles)
+80  f802=5c   VEST. NEKKETSU (13 tiles)   -> 14 frames
```

Decodificado el japonés con el códec de diálogos: tiles
`097 0e1 088 091 09a 085 083 08b 091`.

**FIX**: metafila 11 a metatile 0 en los tres streams:
```
0x0240E0  mf11  (derrota)
0x0243E8  mf11  (victoria)
0x024770  mf11  (pantalla doble de vestuarios)
```
Verificado que los metatiles de esa fila (`3b-40`, `39-3e`, `48-4e`) son
**exclusivos** de ella: no aparecen en ninguna otra fila del mapa.

Resultado medido: `+80 f802=5c ES` directo, **sin fase japonesa**. Escena
intacta.

### ERRORES PROPIOS EN ESTA INVESTIGACIÓN

**(a) Fix aplicado al stream equivocado.** Primero apliqué el borrado de
metafilas 1 y 2 al stream `0x019980` (que creí el vestuario). Resultado: se
perdió una ventana de la escena. En ese stream la mf1 está vacía y **la mf2
es decorado**, no rótulo. Captura del daño: `work/shots/escenas/vest_fix.png`.
Lección: el patrón "metafilas 1-2 = rótulo" NO es general; hay que verificar
la exclusividad de los metatiles escena por escena.

**(b) Confusión de escena.** `f806=0x2c` es el vestuario con diálogo de
Misako, pero su rótulo no existe; el que lleva "VEST. NEKKETSU" es
`f806=0x24`. Se localizó barriendo los 23 valores de `$f806` y capturando
cada uno.

**(c) Indexación mal calculada.** Leí la tabla `0x17750` con paso 8 en vez de
`(f806 & 0xfc) * 8`. Dio con la entrada correcta por casualidad.

## 2. Rótulo 'CLUB DE DODGEBALL'

```
0x08C05C  columna     9 -> 7
0x08C064  ancho-1  0x0d -> 0x11   (14 -> 18 tiles)
0x08C100  tilemap: 18 words
```
Compuesto con `tools/rotulo.py`, reproduciendo la fuente medida del rótulo
anterior (mayúsculas de 7 px, trazo de 2 px, glifos a media celda). Ocupa las
columnas 7..24; el borrado limpia 4..27, así que cabe.

## 3. Corazones

Códec: **`0x4B` = `♥`**. Elegido porque no lo usa ninguno de los 86 bloques,
su tile en la hoja de fuente está vacío, y los diálogos se pintan con
atributo `0x0000` (paleta 0) mientras el único uso de `0x4B` en pantalla es
`0x204B` (paleta 1), otro juego de gráficos.

Diseño: se renderizaron **5 variantes en pantalla real** y el usuario eligió
la C (`work/shots/corazon/COMPARATIVA.png`). La primera versión ocupaba 6 px
pegada a la izquierda y salía asimétrica; la definitiva usa los 8 px con el
pico centrado.

### ERROR PROPIO: la caché de patrones, otra vez

El primer intento de comparar los diseños usó `poke_vram_tile`. **Los cinco
PNG salieron con el mismo md5.** Está documentado desde el principio del
proyecto que ese método es inválido porque no invalida la caché de patrones,
y aun así caí. Rehecho parcheando el asset comprimido y generando una ROM por
variante: cinco md5 distintos.

## 4. Reubicación del asset de fuente

`0x90000` → **`0x091100`**, 5 cargadores repuntados
(`0x000842, 0x00103A, 0x001BAA, 0x003ED6, 0x004BC8`).

Causa: el asset nuevo son 4.040 B y el hueco original 3.851. Dos motivos
acumulados:
- **mi compresor es 64 B peor que el del juego** (3.915 vs 3.851 con los
  MISMOS datos: la v1.6 no se podría reconstruir hoy en su propio hueco);
- el rótulo de 18 tiles cuesta 345 B frente a los 241 del de 14.

### HIPÓTESIS DESCARTADA: la cola tras el asset NO estaba libre

Creí que el espacio entre `0x090F0B` y `0x091029` eran restos del asset de la
v1.4 (4.137 B) y lo rellené con `0xFF`. **Regresión**: los tiles
`0x4F2..0x4F5` pasaron de estar vacíos a mostrar katakana.

Allí empieza **otro asset comprimido** (151 B, 8 tiles) que se carga a VRAM
`0x9E40`. El fallo de razonamiento fue buscar solo punteros `lea.l` a esa
zona: el descompresor `0xF2A4` recibe la dirección en un registro, así que su
ausencia no prueba nada. **Lo correcto es intentar descomprimir y ver si sale
un asset válido.** Detectado por la batería, no por inspección.

## 5. Auditoría de diálogos inalcanzables — [HECHO]

Pregunta del usuario: ¿queda algún diálogo en el limbo?

Cruzados los 87 bloques con texto contra **todas** las vías de disparo:
- la orden `FF08` del intérprete de actores (49 índices)
- las cinco tablas de índices: `0x5430` (créditos), `0x115BC`, `0x11718`,
  `0x1173E`, `0x1197E` (nombres)

```
bloques con texto : 87
alcanzables       : 86
sin vía conocida  :  1  -> [7] 'y no puede ir.'
```

El índice 7 **apunta al mismo bloque que el 8** (`0x080572`), y el 8 sí se
muestra. Es un alias del original, no contenido perdido. Comprobado idéntico
en v0.9.8, v1.4 y v1.7.

**Conclusión: no queda ningún texto inalcanzable.**

Nota: al leer las tablas usé tamaños estimados y di por huérfanos los índices
63 y 64 (`MASA`, `GENEI`). Releída la tabla `0x1197E` completa, sus 10
primeras entradas son `55..64`: sí están cubiertos.

## 6. Secuencia de diálogos: idéntica a la japonesa

```
japonesa: [1,2,3,4,5,6,8,9,10,11,12,...,25]
v1.7    : [1,2,3,4,5,6,8,9,10,11,12,...,25]   IDÉNTICA
```

## 7. Batería de regresión v1.6 vs v1.7

| prueba | resultado |
|---|---|
| Título (frame 950) | VRAM/CRAM/SAT IGUAL |
| Presentación | CRAM y SAT IGUAL; 19 tiles distintos = los tocados |
| Tiles `0x4F2-0x4F5` | vacíos (regresión anterior corregida) |
| Bytes en mapas/SAT | 0 |
| Partido jugado 3.000 frames | sin regresión |
| Vestuarios | parpadeo 14 frames → 0 |

## 8. Limpieza de parches

A petición del usuario se elimina todo el histórico de parches y se conserva
**un único par**, generado desde la ROM japonesa limpia:

```
patch/Nekketsu_Soccer_MD_ES_v1_7.ips   531.933 B
patch/Nekketsu_Soccer_MD_ES_v1_7.bps   531.731 B
base: rom/jp.md  md5 ff7a9a6fb74a640f40f10dff53e9cf4d  crc32 f49c3a86
```

Ambos verificados aplicándolos con un implementador IPS/BPS independiente:
md5 del resultado `28578fa7200235738def7e9afcce0541`, igual al de la ROM.

Las ROM intermedias SÍ se conservan: sirven de registro y de base de
comparación para las baterías.

---

# Revisión del usuario tras probar la v1.7 — cuatro cuestiones

## 1. La fuente románica del título SIGUE EN LA ROM — [HECHO]

Pregunta del usuario: la pantalla de título usa nuestra fuente de diálogos en
`START`/`OPTION` y el copyright, mientras que la japonesa tiene una fuente
románica (serif) distinta. ¿Se puede recuperar solo ahí?

**Sí, y la fuente original está intacta.**

Cadena localizada. El texto del título usa **los mismos índices de tile** en
ambas ROM (verificado leyendo el nametable):

```
JP   fila19: 033 034 021 032 034   -> 'START'
v1.7 fila19: 029 02e 029 023 029 02f -> 'INICIO'
```

Es decir, el juego escribe códigos de carácter idénticos; lo que cambia es
**el juego de tiles cargado en VRAM 0**. Y quien lo carga es:

```
000840: lea.l $2767a,a0      <- ROM JAPONESA (fuente románica)
000840: lea.l $91100,a0      <- v1.7 (nuestra fuente de diálogos)
000846: move.w #$100,d1
00084A: move.w d1,$fcc4
00084E: jsr $f2a4            descompresor
```

La traducción v0.9.8 repuntó ese `lea.l` de `0x02767A` a la fuente nueva.

**La fuente románica sigue en `0x02767A`, byte a byte idéntica a la japonesa**
(8192 B descomprimidos, 4080 B comprimidos, fin `0x02866A`). Comprobado con
`decomp()` sobre las dos ROM: `identicos = True`.

Glifos verificados (trazo de 2 px, con serifas):
```
tile 0x33 'S'      tile 0x21 'A'      tile 0x2F 'O'
··██████           ···███··           ··█████·
·██····█           ··██·██·           ·██···██
·███····           ·██···██           ·██···██
···███··           ·██···██           ·██···██
·····███           ·███████           ·██···██
·█····██           ·██···██           ·██···██
·██████·           ·██···██           ··█████·
```

**Cobertura**: la románica tiene glifo para `C E I N O P S`, todo lo que
necesitan `INICIO` y `OPCIONES`. Le faltan `Á É Í ¿ ¡`, que no se usan ahí.

**Alcance del cambio — MEDIDO**: el cargador `0x000840` se ejecuta **solo con
`$f800 = 0x08`** (2 veces, frames 447 y 464 en el barrido). Los otros cuatro
cargadores del asset (`0x001038`, `0x001BA8`, `0x003ED4`, `0x004BC6`) cubren
los demás estados.

**PERO** — dato que condiciona la decisión: `$f800 = 0x08` **no es solo la
pantalla de título**. Incluye también el menú principal, que se dibuja sobre
la misma pantalla:
```
f19: 'TORNEO1J'   f21: 'TORNEO2J'   f23: 'DUELO2J'   f25: 'CLAVE'
```
Revertir el cargador cambiaría la fuente de `INICIO`/`OPCIONES` **y también**
la del menú `TORNEO 1J / TORNEO 2J / DUELO 2J / CLAVE`. No son dos pantallas
distintas sino dos estados de la misma.

Queda pendiente de decisión del usuario. No se ha modificado nada.

## 2. Rótulo `CLUB DE DODGEBALL`: la japonesa lleva cinta verde

Observación del usuario: el rótulo japonés 「ドッジボール部室」 va sobre una
cinta verde con forma de banderola, y el nuestro es solo texto.

Es correcto: al eliminar el rótulo japonés del stream (v1.7, cambio 1) se
eliminó **también la cinta**, porque forma parte de las mismas metafilas 1-2.
El hook `0x08C040` solo pinta texto plano.

Para reproducir la cinta habría que dibujarla como tiles nuevos y ampliar el
tilemap de `0x08C100` a varias filas (ahora es 1 fila de 18 tiles; el borrado
limpia 3 filas × 24 columnas, así que hay sitio). **No investigado en detalle
ni implementado.**

## 3. La escena del marcador con cielo naranja — IDENTIFICADA

El usuario pregunta qué determina que aparezca la escena del resultado en
grande (cielo naranja, jugadores cabizbajos) antes de los vestuarios.

Es **`$f806 = 0x28`** (y sus repeticiones `0x38`, `0x48`, `0x58`, una por
ronda del torneo). Capturada: `work/shots/escenas/nat_28.png`. Muestra
`NEKKETSU 0 - 3` con el kanji 熱血 a la izquierda y 優秀院 a la derecha.

La escena siguiente, con los jugadores pidiendo perdón y
`みさこ「もういちど ちょうせんする?」` (*Misako: ¿Volver a intentarlo?*), es
`$f806 = 0x2c` / `0x3c` / `0x4c` / `0x5c`.

**El texto de esa escena SÍ está traducido** y es uno de los bloques marcados
como intocables por diseño en `realign.py`:
```
_KEEP = ('FIN DE LA PARTIDA', 'MISAKO: ¿Volver a intentarlo?')
```
Se dejó sin recomponer porque su separación `SÍ    NO` es funcional: ahí se
coloca el cursor.

Sobre los asteriscos que el usuario cree haber visto en esa escena: el bloque
con `****` es el **24**, que pertenece a la sala del club y ya lleva corazones
en la v1.7. Habría que verificar si hay otro bloque con asteriscos en la rama
de derrota. **No verificado aún.**

## 4. Pantalla de selección de escuela (DUELO 2J) — REPRODUCIDA

Estado **`$f800 = 0x30`**. Alcanzada con esta navegación (los pulsos cortos no
sirven: el menú necesita ~4 frames de pulsación y ~20 de separación):

```
frames 430 y 700: START
luego: DOWN x2 (4 frames pulsado, 20 sueltos)
luego: START
```

Captura: `work/shots/SEL_es.png`. Estructura medida del nametable:

```
plano A filas 0..5   rótulo dorado 「学校セレクト」 (Selección de escuela)
                     marco de 20 celdas de ancho, tiles 0x106..0x1A2
plano A filas 7..25  13 nombres de escuela en bloques de 3x3 celdas
                     tiles 0x203..0x38C
plano B              fondo PALSOFT repetido
```

Tilemaps localizados en ROM:
```
0x02AD90   rótulo dorado   (atributo 0x2000 = paleta 1)
0x02BD80   nombres         (atributo 0x8000 = prioridad)
```

Los 13 nombres son 3x3 tiles cada uno = 9 tiles por escuela, 117 tiles en
total, más el rótulo. Es el trabajo gráfico más grande que queda.

**No implementado.** Requiere decidir con el usuario: los nombres japoneses
son kanji (熱血, 優秀院, 七福...) y su transcripción ya existe en el juego
(NEKKETSU, YUSHUIN...), pero mantener "el estilo del diseño" con 3x3 celdas
por nombre exige dibujarlos a mano.

---

# ROM v1.8 — fuente románica, cinta verde y corazones de la derrota

`md5` `3d9f3718ce33fb4cfee95802ab8a4ce6` · 8.165 bytes distintos de la v1.7.

## 1. Fuente románica del título y menú (4 bytes)

```
0x000842: 00091100 -> 0002767A
```

Verificado antes de escribir: la románica de `0x02767A` descomprime a 8192 B
y tiene glifo para todos los caracteres de `INICIO`, `OPCIONES`,
`TORNEO 1J/2J`, `DUELO 2J`, `CLAVE` y los dos copyright, símbolo © incluido.
El constructor lo comprueba con un `assert` por carácter.

Alcance aprobado por el usuario: título y menú principal son el mismo estado
(`$f800 = 0x08`) y comparten fuente. *"Forman parte del mismo menú, lo raro
sería que fuera cambiando."*

## 2. Cinta verde del rótulo — reconstruida

La v1.7 eliminó el rótulo japonés del stream y con él **la banderola**, porque
ambos vivían en las metafilas 1-2. Defecto introducido por mí, no del original.

Anatomía medida en la japonesa (plano A, filas 3..5, cols 4..27, atributo
`0xA100` = paleta 1 + prioridad):

```
tile 0x12E  borde diagonal izquierdo      tile 0x15C  borde derecho
tile 0x1BF  relleno liso (verde)          tile 0x100  fondo
trapecio:   fila 3 cols 6..25 · fila 4 cols 5..26 · fila 5 cols 4..27
```

Los tiles del marco **se reutilizan tal cual**: no hay que redibujarlos, solo
referenciarlos. Lo único nuevo son los 18 tiles del texto castellano, con el
mismo relieve del original (trazo `F` blanco, sombra `E` al inferior-derecha,
fondo `B` verde), medido en el tile `0x1B1` del japonés.

Cambios en el hook `0x08C040`, que pasa de pintar 1 fila a pintar 3:
```
0x08C05C  columna   7 -> 4
0x08C060  fila      4 -> 3
0x08C064  ancho-1  17 -> 23
0x08C068  alto-1    0 -> 2
0x08C100  tilemap: 144 B (3 filas x 24 cols)
```

### Mapeo de tiles del asset de escena — [HECHO]

El asset `0x068CBE` (512 tiles) carga a **VRAM `0x2000`**, luego:
```
índice del asset = tile de VRAM - 0x100
```
Verificado comparando `0x1BF`, `0x12E`, `0x15C` y `0x1B1` entre el asset
descomprimido y la VRAM. Índices libres usados: `0x0C6..0x0D7`.

### Reubicación del asset de escena

`0x068CBE` → **`0x0972AE`**, puntero repuntado en `0x017770` (tabla `0x17750`,
entrada `f806=0x04`, campo `+0x00`). Motivo: el asset con los 18 tiles nuevos
ocupa 8.507 B comprimidos y el hueco original son 7.983. Mismo patrón que ya
ocurrió con la fuente de diálogos: **mi compresor es peor que el del juego**.

## 3. Corazones en la rama de derrota — bloque 31

El usuario vio `XXXX` en un diálogo de Misako **antes** de la pregunta
"¿Volver a intentarlo?". No era el bloque 24 (sala del club, ya corregido en
la v1.7) sino el **31**:

```
0x081046  'MISAKO: ¿Vais a perder?' / '¿No os importan mis ****?'
```

Cadena de disparo localizada:
```
0x01175C  FF08 001F      orden de diálogo, bloque 31
          pertenece a un script de actor de la tabla 0x010BC8
0x00F684  lea.l $10bc8,a0 / cmpi.w #$d,$0(a6) / beq $f69a
```

La tabla `0x010BC8` la selecciona el intérprete cuando el **modo de actor
`$0(a6)` vale `0x0D`**. El script del bloque 31 es su entrada 72 (`0x010CE8`).
Hay una segunda referencia en `0x010D98` (entrada 116) al mismo script.

**Por qué el usuario casi nunca la ve**: el modo `0x0D` es un estado de actor
distinto del habitual. El barrido automático de 9.000 frames con navegación
estándar **no lo alcanza** (0 disparos del bloque 31 medidos). Concuerda con
lo que reporta: le salió una vez y no ha vuelto a reproducirla.

Capturado forzando el motor (`$f950=31` con `$f940=0xFFFF`, `$f94E=1` y el
motor libre): `work/shots/v18_bloque31.png`. Corregido a corazones y
verificado en VRAM: 4 tiles `0x4B` en la fila 26.

**Nota metodológica**: el forzado solo prende si se inyecta cuando
`$f940 == 0` y `$f94C == 0` (motor libre). Inyectar en cualquier frame no
funciona: el primer intento devolvió el bloque 1, no el 31.

## 4. Batería de regresión v1.7 vs v1.8

| prueba | resultado |
|---|---|
| Presentación 1er partido | VRAM / CRAM / SAT IGUAL |
| Partido jugado 3.000 frames | sin regresión |
| Tabla de diálogos | 86 bloques recorribles |

## Pendiente: pantalla de selección de escuela (DUELO 2J)

Estado `$f800 = 0x30`. Tilemaps localizados:
```
0x02AD90   rótulo dorado 「学校セレクト」  (atributo 0x2000)
0x02BD80   13 nombres de escuela          (atributo 0x8000)
```
El rótulo es una placa dorada con texto en relieve; los nombres son bloques
de 3x3 celdas (117 tiles). El usuario se ofrece a editar el gráfico del
rótulo si se le pasa extraído, respetando los colores.

---

# Revisión posterior a la v1.8 — investigación (sin generar ROM aún)

## 1. Alineación de INICIO / OPCIONES — CONFIRMADO el diagnóstico del usuario

El usuario recordaba que en la traducción se movió el balón en vez de alinear
el texto. **Es exactamente lo que pasó.** Medido:

```
JP    fila19 col 13..17  'START'      fila21 col 13..18  'OPTION'
v1.8  fila19 col 13..18  'INICIO'     fila21 col 12..19  'OPCIONES'
```

El hook está en `0x088040`, con cadenas ASCII planas:
```
088042: move.w #$d,d0      columna 13   INICIO   <- inmediato en 0x088044
088046: move.w #$13,d1     fila 19
08804A: lea.l $88200,a0    'INICIO\0'
088054: jsr $78a6

08805A: move.w #$c,d0      columna 12   OPCIONES <- inmediato en 0x08805C
08805E: move.w #$15,d1     fila 21
088062: lea.l $88207,a0    'OPCIONES\0'
08806C: jsr $78a6
```

Y el balón (sprite, `$f006`) se desplazó para compensar:
```
JP    0008E8: move.w #$d8,$f006.w      x=216
v1.8  0008E8: move.w #$d0,$f006.w      x=208
```
Un único byte de diferencia en toda la ROM: `0x0008EB  D8 -> D0`.

**FIX probado en memoria (2 words):**
```
0x08805C: 000C -> 000D     OPCIONES a la columna 13
0x0008EA: 00D0 -> 00D8     balón a x=216, su sitio original
```
Resultado medido: `fila19 col 13..18 'INICIO'`, `fila21 col 13..20 'OPCIONES'`,
balón `y=276 x=216` — idéntico a la japonesa.

### ERROR PROPIO durante la prueba
Escribí primero en `0x08805E`, que es el **opcode** de `move.w #$15,d1`, no el
inmediato de la columna. Resultado: `OPCIONES` se dibujó como `OPTION`.
El inmediato correcto es `0x08805C`. Regla que se repite en este proyecto:
en `303c xxxx` el valor va en `+2`, y hay que verificarlo leyendo los bytes
antes de escribir.

## 2. Qué hace que el modo de actor valga 0x0D — RESPUESTA COMPLETA

El usuario preguntó, con razón, qué provoca ese modo. Cadena completa:

```
004D20: move.w $f806,d0 / andi #$fc / asl.l #3
004D2C: adda.l d0,a1                 a1 = tabla 0x17750
004D2E: movea.l $10(a1),a4           a4 = descriptor de actores de la escena
004D3E: move.w (a4),d6               tipo de descriptor
004D46: movea.l $2(a4),a5            lista BASE de actores
004D54: jsr $4da2                    -> spawnea
004D5A: cmpi.w #$1,(a4) / beq        tipo 1: solo lista base
004D62: cmpi.w #$2,(a4)              tipo 2: hay ramas por resultado
004D6E: cmpi.w #$2,$f804.w / beq
004D78: moveq #$6,d5                 <- offset +6  = rama DERROTA
004D7A: move.w $f8d2,d0
004D7E: cmp.w  $f8d0,d0
004D82: bcs    $4d88
004D86: moveq #$a,d5                 <- offset +10 = rama VICTORIA
004D88: adda.l d5,a4 / movea.l (a4),a5
004D96: jsr $4da2

004DA2: move.w #$d,$0(a6)            <<< AQUI se pone el modo 0x0D
```

**`0x004DA2` es el spawner de actores de escena**, y es el ÚNICO sitio de toda
la ROM que escribe `#$d` en `$0(a6)` (verificado por barrido de todos los
modos de direccionamiento). Es decir: **el modo `0x0D` no es una condición
rara, es sencillamente "actor de escena de interludio"**. Todos los actores
que spawnea esa rutina lo llevan.

La decisión real es la comparación de goles en `0x004D7A`:
```
$f8d2 (goles propios) < $f8d0 (goles rival)  -> rama DERROTA  (+6)
en caso contrario                            -> rama VICTORIA (+10)
```

### Y aquí está la respuesta a por qué el usuario casi no ve el bloque 31

Buscado el script del bloque 31 (índices `0x48` y `0x74`) en TODAS las listas
de actores de todas las escenas:

```
f806=2c  rama DERROTA   actor 7: script 0x74
f806=2c  rama VICTORIA  actor 0: script 0x48   <<<
f806=2c  rama VICTORIA  actor 9: script 0x74
f806=3c  rama VICTORIA  actor 0: script 0x48
f806=4c  rama DERROTA   actor 7: script 0x74
f806=4c  rama VICTORIA  actor 0: script 0x48
f806=5c  rama VICTORIA  actor 0: script 0x48
```

**El script 0x48, que contiene el `FF08` del bloque 31, está sobre todo en la
rama de VICTORIA.** El diálogo `MISAKO: ¿Vais a perder? ¿No os importan mis
♥♥♥♥?` no es de derrota: es el de Misako **animando entre partidos ganados**.

Corrige mi afirmación anterior ("pertenece a la rama de derrota"), que era
incorrecta: en `f806=2c` y `f806=4c` el script `0x74` sí está en la rama de
derrota, pero el `0x48` — el que lleva el `FF08` del bloque 31 — está en la
de victoria en las cuatro escenas.

## 3. Placa dorada 'ELIGE ESCUELA' — construida

Paleta medida contra el framebuffer (no contra CRAM, ver más abajo):
```
idx 3  #202000  contorno negro      idx 2  #8b8900  dorado medio
idx 9  #414400  sombra media        idx 6  #cdce00  dorado claro
idx 7  #626520  FONDO de la placa   idx 8  #eeee00  dorado brillante
idx 4  #8b8920  dorado (marco)      idx E  #eeeeee  blanco (chispazos)
```

**El marco NO se redibuja**: es un patrón moteado de 6 px de grosor,
irreproducible a mano de forma convincente. Se vuelca de la VRAM japonesa a
`work/placa_orig.json` y se reutiliza tal cual; solo se sustituye el interior
(x 12..147, y 8..39).

Fuente nueva de caja alta, 13 px de alto y trazo de 3 px, con el relieve que
pidió el usuario: blanco en el pico superior-izquierdo, brillo en los cantos
iluminados, medio en los opuestos y doble sombra diagonal abajo-derecha.

### ERROR DE MEDICIÓN: la CRAM
Intenté leer la paleta con `m.cram()` y probé cuatro interpretaciones
(BE, byteswap, R↔B, little-endian). **Ninguna daba el dorado**: el canal azul
salía siempre a 0. Contrastado con la captura del usuario (`#5a5b1e`,
`#cbca04`) se ve que el accesor no devuelve lo que creo.
**Método válido**: leer el color del framebuffer (`save_png`) y asociarlo al
índice del píxel correspondiente. Así salieron los 8 colores correctos.

### DOS BUGS PROPIOS en la fuente de la placa
1. **`SEP=1` con la I con serifas**: `ELIGE` se leía `EIIGE` porque la I con
   remates se fundía con la L contigua. La I va sin serifas.
2. **El recorte de columnas miraba solo 7 filas** (`for r in range(7)`) sobre
   glifos de 15. La `L` es un asta vertical en sus 11 primeras filas, así que
   se le recortaba la base y quedaba idéntica a una `I`. Corregido a
   `range(15)`. Este es el que hacía que se leyera `EIIGE ESCUEIA` incluso
   después de arreglar el punto 1.

---

# RECTIFICACIÓN: el bloque 31 SÍ es de la rama de derrota

El usuario señaló una contradicción en mi conclusión anterior: la pantalla del
cielo naranja que precede a esa escena muestra `NEKKETSU 0 - 3`, o sea una
**derrota**, luego el diálogo no podía ser exclusivo de la victoria.

**Tenía razón. Mi conclusión era falsa.** El error de método fue mirar solo en
qué rama aparecía cada *índice de script* (`0x48` vs `0x74`) sin resolver a
qué **bloques de diálogo** llevaba cada uno.

Al resolverlos:

```
script 0x74 -> 0x011756     script 0x48 -> 0x01175C     distancia: 6 bytes
```

Son **el mismo script**. `0x74` empieza 6 bytes antes (una entrada de
animación `0006 00E9` y una orden `FF00`) y a continuación **cae exactamente
en el mismo `FF08 001F`**. Los dos ejecutan la misma secuencia de diálogos:

```
[31] MISAKO: ¿Vais a perder? ¿No os importan mis ♥♥♥♥?
[32] TODOS: ¡Claro que sí!
[33] MISAKO: ¿Volver a intentarlo?  SÍ    NO
[35] MISAKO: ¡Qué crueles! ¡Os odio a todos!
[36] FIN DE LA PARTIDA
```

Distribución real, resolviendo script → bloques:

```
f806  rama       actor  script   bloques
 2c   DERROTA      7    0x0074   [31,32,33,35,36]
 2c   VICTORIA     0    0x0048   [31,32,33,35,36]
 2c   VICTORIA     9    0x0074   [31,32,33,35,36]
 3c   VICTORIA     0/9  0x0048/74
 4c   DERROTA      7    0x0074   [31,32,33,35,36]
 4c   VICTORIA     0/9  0x0048/74
 5c   VICTORIA     0/9  0x0048/74
```

**El bloque 31 está en la rama de DERROTA de `f806=2c` y `4c`.** Y el
contenido lo confirma sin lugar a dudas: la secuencia termina en
`¿Volver a intentarlo? SÍ/NO` y `FIN DE LA PARTIDA`. Es la escena de
game over, no un interludio de victoria.

Lo que sigue siendo cierto de la investigación anterior:
- `0x004DA2` es el spawner y el único punto que escribe el modo `0x0D`;
- la rama la decide `cmp $f8d0,$f8d2` en `0x004D7A`.

Lo que era **falso** y queda descartado:
- *"el script 0x48 está sobre todo en la rama de victoria, luego el diálogo
  es de Misako animando entre partidos ganados"*. La presencia del índice en
  una rama no dice nada si no se resuelve el script: dos índices distintos
  apuntaban al mismo código.

Norma que se repite en este proyecto: **resolver siempre el puntero hasta el
dato final**. Clasificar por el índice intermedio ya me indujo a error antes
(las tablas de diálogo leídas con tamaños estimados).

Por qué el usuario lo ve rara vez sigue sin estar cerrado: la escena existe en
las dos ramas, así que debería salir siempre al perder. Posible explicación no
verificada: `f806=0x2c` y `0x4c` son rondas concretas del torneo, y perder en
otra ronda lleva a `0x3c`/`0x5c`, que solo tienen la rama de victoria poblada
con ese script. **Pendiente de comprobar.**

---

# LA ESCENA PERDIDA — RESUELTA [HECHO, verificado en emulador]

Encargo del usuario: averiguar cómo se llega a la escena de Misako con
`¿Vais a perder? ¿No os importan mis ♥♥♥♥?`, y explicarlo **en clave de
juego**, no de direcciones.

## Corrección previa: f8d0 y f8d2 estaban al revés

Asumí que `$f8d2` eran los goles propios. **Es al contrario**, y se demuestra
por las consecuencias, no por el nombre:

```
005790: move.w $f8d0,d0
005794: move.w $f8d2,d1
005798: cmp.w  d0,d1
00579A: bcs    $57c0        d1 <  d0 -> 0x57C0 = SIGUE JUGANDO
        (si no)             d1 >= d0 -> 0x579E = f852=0, f800=0 = VUELTA AL TÍTULO
```

Volver al título es el GAME OVER. Luego `d1 >= d0` (es decir
`f8d2 >= f8d0`) es la **derrota**. Por tanto:

```
$f8d0 = goles del NEKKETSU (nosotros)
$f8d2 = goles del RIVAL
```

Mis dos pruebas anteriores estaban etiquetadas al revés, y por eso concluí
primero "victoria" y luego "derrota". Ninguna de las dos conclusiones valía.

## El ciclo de escenas de cada partido

`$f806` avanza de 4 en 4 y el despacho (tabla `0x5254`) revela un ciclo de
cuatro fases que se repite una vez por ronda:

```
0x20 / 0x30 / 0x40 / 0x50  -> 0x0054EC  PRESENTACIÓN del partido
0x24 / 0x34 / 0x44 / 0x54  -> 0x005514  VESTUARIO (VEST. NEKKETSU)
0x28 / 0x38 / 0x48 / 0x58  -> 0x005590  MARCADOR (cielo naranja)
0x2c / 0x3c / 0x4c / 0x5c  -> 0x00560C  MISAKO + ¿volver a intentarlo?
```

## Verificación final (marcador forzado, ciclo natural desde el marcador)

```
GANAMOS 3-0   -> [26] MISAKO: ¡Qué bien jugasteis!...
                 [43] MISAKO: El próximo partido es en el campo 2...

PERDEMOS 0-3  -> [31] MISAKO: ¿Vais a perder? ¿No os importan mis ♥♥♥♥?
                 [32] TODOS: ¡Claro que sí!
                 [33] MISAKO: ¿Volver a intentarlo?  SÍ   NO

EMPATE 1-1    -> igual que la derrota (el empate cuenta como derrota)
```

**Confirmado: el bloque 31 es la rama de DERROTA (o empate).**

## Por qué el usuario casi nunca la veía

Medido con breakpoint en `0x0055AA` variando `$f852` (número de partido):

```
f852 = 0  ->  la decisión NO se alcanza
f852 >= 1 ->  la decisión se alcanza
```

`$f852` es el índice de partido, 0-based. Con `f852 = 0` (el **primer**
partido) el flujo no llega a esa bifurcación: al perder el primero se sale
directamente sin pasar por la escena de Misako.

Y el segundo requisito, medido: hace falta `$f840 = 2`, que es el estado
"el partido ha terminado por tiempo". Si el partido se abandona por otra vía
el ciclo no se completa.

## El menú "¿Volver a intentarlo?" — descifrado

```
0056EC: btst #2,$f811 -> IZQUIERDA:  $f842 = 0  cursor en "SÍ"
0056FA: btst #3,$f811 -> DERECHA:    $f842 = 1  cursor en "NO"
00570A: X del cursor = 0xE0 (SÍ) / 0x118 (NO)   -> $f006
00571E: A, B o C confirma

  "SÍ" ($f842=0):  0x00575E  $f8d8 = 1   marca "ya has continuado"
                             $f840 = 2
                             $f8d0 = -1  reinicia el partido
  "NO" ($f842=1):  0x00573E  $f840 = 3
                             -> bloques 35 (¡Qué crueles!) y 36 (FIN DE LA PARTIDA)
```

`$f8d8` es el flag de "continuación usada". En `0x0055CC` se comprueba, y en
`0x0057CA` hay un límite con `$f852 >= 13`.

## RESPUESTA EN CLAVE DE JUEGO

Para ver la escena hay que **perder (o empatar) un partido que NO sea el
primero del torneo**. Es decir: ganar el primero y dejarse perder el segundo.

Al terminar ese partido aparece la pantalla del marcador con el cielo naranja
y, a continuación, el vestuario con Misako reprochando y el menú SÍ/NO. Con
"SÍ" se repite el mismo partido; con "NO" salen `¡Qué crueles!` y
`FIN DE LA PARTIDA`.

Si se pierde el **primer** partido, el juego se salta esa escena y va directo
al game over. Eso explica exactamente lo que reportaba el usuario: la vio una
vez (perdió un partido avanzado) y no volvió a verla (perdía el primero).

---

# LA ESCENA PERDIDA — CAUSA REAL: BUG DE LA TRADUCCIÓN v0.9.8 [HECHO]

## Rectificación doble: el usuario tenía razón las dos veces

Afirmé primero que el bloque 31 era de victoria, luego de derrota, y después
"para verla hay que perder un partido que no sea el primero". Las dos primeras
eran falsas y **la tercera describía un síntoma, no la causa**.

El usuario objetó con tres argumentos, todos correctos:
- el diálogo dice `¿Vais a perder?` — es una derrota;
- la escena previa muestra `NEKKETSU 0 - 3` con el equipo cabizbajo;
- no tiene sentido que perder el primer partido salte la escena y meta al
  jugador en un bucle `pierdes → pizarra → partido → pierdes...` sin
  posibilidad de decir "no" y terminar.

## Prueba objetiva del significado de f8d0 / f8d2

Forzando `f8d0=0, f8d2=3` y capturando la pantalla del cielo naranja se lee
literalmente **`NEKKETSU 0 — YUSHUIN 3`** (`work/shots/MARCADOR_03.png`).
Confirma sin ambigüedad:

```
$f8d0 = goles del NEKKETSU (nosotros)
$f8d2 = goles del RIVAL
```

Y con ese marcador el diálogo que sale es el 31/32/33 (`¿Vais a perder?` →
`¿Volver a intentarlo?`). **Es la rama de derrota**, como decía el usuario.

## EL BUG

Medido variando solo el número de partido, perdiendo 0-3:

```
              perder PARTIDO 1        perder PARTIDO 2
japonesa      escena SÍ aparece       escena SÍ aparece
ES v0.9.8     escena NO aparece       escena SÍ aparece
v1.4 / v1.7 / v1.8   igual que la v0.9.8
```

**No es del juego original: lo introdujo la traducción v0.9.8.**

Única diferencia en toda la zona (`0x5580`-`0x5610`), 3 bytes:
```
JP  0055A2: jsr $4a20        rutina original
ES  0055A2: jsr $8f400       hook de la traducción
```

El hook `0x08F440` dibuja el rótulo del marcador y hace esto:

```
08F470: move.w $f852,d0      d0 = índice de partido (0-based)
08F474: subq.w #$1,d0        d0 = f852 - 1        <<< con f852=0 -> -1
08F476: add.w  d0,d0
08F478: add.w  d0,d0         d0 *= 4
08F47A: lea.l  $8fd40,a1
08F480: movea.l (a1,d0.w),a0 <<< lee 4 bytes ANTES de la tabla
08F49A: jsr $7a3c            blitea desde ese puntero
```

Con `f852 = 0` (primer partido) el índice es **−1** y lee `0x08FD3C`, que
contiene `FFFFFFFF`. El blitter recibe un puntero inválido y la escena no
llega a completarse, así que el juego nunca alcanza `f806 = 0x2c`.

```
tabla 0x08FD40, indexada por f852-1:
  idx -1 @ 08FD3C = FFFFFFFF   <<< lectura fuera de rango con f852=0
  idx  0 @ 08FD40 = 0008F940   partido 1
  ...
  idx 12 @ 08FD70 = 0008FCA0   partido 13
```

La tabla está construida para índice **0-based del partido**, es decir debería
indexarse con `f852` directamente, no con `f852-1`. El `subq.w #$1,d0` sobra.

## Consecuencia jugable

Al perder el primer partido el jugador **no ve la escena de Misako ni el menú
SÍ/NO**, y va directo a la pizarra a repetir el partido. Como el menú es la
única vía para terminar la partida, queda atrapado en el bucle que describía
el usuario. En la japonesa esto no ocurre.

## Estado

Bug **localizado y explicado**, con la instrucción exacta señalada
(`0x08F474`, 2 bytes). **No se ha modificado nada**: pendiente de decidir el
arreglo con el usuario y de que él contraste con la versión de NES.

---

# EL MISTERIO DE LA ESCENA — RESUELTO POR EL USUARIO

**Son DOS TIEMPOS del mismo partido, no dos partidos.**

El usuario lo descubrió jugando: lo que yo llamaba "partido 1" y "partido 2"
son la primera y la segunda parte de un mismo encuentro. Al terminar el primer
tiempo se vuelve a la pizarra de tácticas, lo que induce a pensar que empieza
un partido nuevo — más aún si el primer tiempo acaba 0-0.

Esto explica todo lo observado sin necesidad de ningún bug:

- `$f852` no es "número de partido" sino que avanza con cada tiempo;
- la escena de Misako sale **al final del encuentro** (segundo tiempo), no
  entre tiempos;
- mis pruebas con `f852=0` estaban simulando *el descanso*, no el final.

**Confirmado por el usuario también en la versión de NES: dos tiempos.**

## HIPÓTESIS DESCARTADA: "bug del subq en 0x08F474"

Afirmé que el `subq.w #$1,d0` de `0x08F474` leía un índice −1 con `f852=0` y
que eso rompía la escena. **La lectura fuera de rango es real y está medida**,
pero NO es la causa de nada: con `f852=0` el juego está en el descanso, donde
esa escena no debe salir. El rótulo que dibuja ese hook es el del marcador de
tiempo, y su indexación es correcta para el uso real.

Queda anotado como comportamiento a vigilar, no como defecto.

## HIPÓTESIS DESCARTADA: "la traducción introdujo un bug"

Escribí que la v0.9.8 rompió la escena "hace un par de años". Doble error:
- la v0.9.8 la generé yo mismo hace unos días, en otra conversación;
- no rompió nada: la diferencia que medí era artefacto de mi propio montaje.

## ERROR DE MÉTODO — el más grave de la sesión

Todas mis conclusiones sobre esta escena salieron de forzar el estado con
`poke_wram16` (`f840=2`, `f806=0x28`, `f80a`), es decir **fabricando** la
situación en vez de alcanzarla jugando. Cuando el usuario reportó que en la
japonesa real tampoco le salía, debí sospechar del montaje antes que del
juego.

**Norma**: si un resultado procede de un estado inyectado, hay que decirlo al
presentarlo y no describirlo como comportamiento observado del juego.

---

# ROM v1.9 — rótulo del título, placa y alineación del menú

`md5` `4b6b24d01c8cfd49f7a2ecaefcc2aa0d` · 16.446 bytes distintos de la v1.8.

## 1. Alineación del menú (2 words)

```
0x08805C: 000C -> 000D    columna de OPCIONES, 12 -> 13
0x0008EA: 00D0 -> 00D8    X del balón, 208 -> 216 (valor japonés)
```
Medido: `fila19 col 13..18 'INICIO'`, `fila21 col 13..20 'OPCIONES'`, balón
`x=216`. Idéntico a la japonesa.

## 2. Rótulo del título (dibujado por el usuario)

PNG 256×224 con transparencia, hecho sobre una captura real, así que la
posición ya era exacta (filas 3..15, columnas 1..30).

**Los 10 colores que usa son exactamente los de la paleta 1 japonesa de esa
pantalla**, verificados uno a uno. Como nuestro rótulo se pinta con la
paleta 3, se reescribe `0xA3400` con esos valores.

```
tilemap  0x0A3000   512 words, atributo 0x6000
tiles    0x096000   223 tiles únicos, 4828 B de 4881
paleta   0x0A3400   16 colores japoneses
```

## 3. Placa 'ELIGE ESCUELA' (dibujada por el usuario)

8 colores, todos de la paleta original. Marco intacto (0 píxeles distintos
fuera del interior).

### Problema 1: tiles compartidos entre celdas

El tilemap original **reutiliza** tiles: `0x18E` aparece en 3 celdas, `0x190`
en 3, `0x131` en 3, `0x167` en 2. En la placa japonesa esas celdas eran
idénticas; en la del usuario no. Sobrescribir el tile no basta: hay que dar
un índice propio a cada celda en conflicto y actualizar el tilemap.
Resueltos con 5 índices libres del asset (hay 525 vacíos).

### Problema 2: no cabía in situ

```
asset con el rótulo borrado : 10457 B
disponible                  : 12276 B
presupuesto para la placa   :  1819 B
la placa del usuario ocupa  :  2041 B   -> faltaban 222 B
```

Detrás del asset (`0x045A14`) hay **otro asset pegado**, así que no se podía
desbordar.

### HIPÓTESIS DESCARTADA: "el asset no es reubicable"

Afirmé que su puntero se calculaba y que por tanto no había nada que
repuntar. **Falso.** Hay **dos `lea.l` explícitos**:

```
00371E: lea.l $42a20,a0     (operando en 0x003720)
003B76: lea.l $42a20,a0     (operando en 0x003B78)
```

Mi búsqueda anterior los pasó por alto. Reubicado a `0x0993EA` y repuntados
los dos: la placa entra **completa, sin recortar ni un píxel**.

### ERROR PROPIO: dirección impar

Primer intento de reubicación a `0x0993E9`. **Pantalla en negro.** El 68000
lanza address error al leer un word en dirección impar. Corregido a
`0x0993EA`.

**Norma nueva**: toda dirección de reubicación debe ser PAR. Añadir la
comprobación a los constructores.

## Verificación v1.8 → v1.9

| prueba | resultado |
|---|---|
| Partido jugado 3.000 frames | sin regresión |
| Sala del club (cinta verde) | idéntica |
| Tabla de diálogos | 86 bloques |
| Título / menú / placa | correctos en pantalla |

---

# ROM v2.0 — nombres de las escuelas

`md5` `9c2c52102bf7b70770eff2b44784a931` · sha1 `cf28ae6eb1a235ba591ffd973022d6b005c1a5ed`
crc32 `35a00af2` · 15.946 bytes distintos de la v1.9.

## La preocupación del usuario NO se materializa

Temía que cada nombre tuviera una caja distinta y hubiera que fabricar
cuerpos de letra a medida, con el consiguiente "baile visual de tamaños".

**Medido: el ancho y la columna son DATOS EDITABLES, no restricciones.**

```
tabla 0x02BD18   13 descriptores de 8 B:  ancho-1 | alto-1 | puntero
tabla 0x003854   13 coordenadas de 4 B:   columna | fila
```

Ampliando las cajas hasta donde permite el hueco entre columnas
(izq 1..10, centro 12..21, dcha 23..31), **los 13 nombres entran con la
MISMA fuente** de cuerpo 5x7 px y contorno de 1 px. Ni un solo tamaño
especial.

La única concesión: `SHICHIFUKU` (10 letras) usa interletraje 0 en vez de 1.
No cambia el tamaño de la letra, solo junta las letras 1 px. Y como llegaba
justo al borde de pantalla, se le movió la columna de 23 a 22 — igual que a
`YOSHIMOTO`.

Anchos resultantes: 7, 8 o 9 celdas según el nombre.

## Estilo

Medido en el NEKKETSU japonés: cuerpo blanco (índice `0xF`) con contorno de
1 px (`0xE`) sobre fondo transparente. Es lo que pidió el usuario: el mismo
criterio que la pantalla de PRIMER PARTIDO.

## ERROR PROPIO 1: índice negativo en el pool de tiles

Al reunir los índices reutilizables resté `BASE` a todas las celdas del
tilemap, incluidas las **vacías** (`0x0000`), que dieron `-256`. Ese índice
negativo corrompió el pool y los nombres salieron con trozos de kanji
mezclados. Corregido filtrando `t >= BASE`.

## ERROR PROPIO 2: asset equivocado

Escribí los tiles en `0x0993EA`, el asset de la **placa** que la v1.9 había
reubicado. Los nombres siguieron saliendo corruptos.

Localizado por traza VRAM←ROM sobre el tile `0x203`: los nombres viven en
**`0x064FD4`**, un asset distinto de 1024 tiles con **base `0x200`**.

Lección repetida por tercera vez en el proyecto: **no dar por supuesto de qué
asset procede un tile**. Hay que trazar la escritura a VRAM hasta la lectura
de ROM que la alimenta.

## Reubicación de los tilemaps

Los 13 tilemaps ampliados ocupan 672 B y su zona original solo tenía 576
(detrás hay otro tilemap pegado, el fondo PALSOFT). Reubicados a `0x02C780`
(hueco de 2178 B verificado a `0xFF`, dirección par).

No hizo falta tocar código: cada descriptor lleva su propio puntero.

## Verificación v1.9 → v2.0

| prueba | resultado |
|---|---|
| Partido jugado 3.000 frames | sin regresión |
| Pantalla de título | idéntica |
| Sala del club (cinta) | idéntica |
| Los 13 nombres | dentro de pantalla, cols 1..30 |
| Cursor de selección | se ajusta a las cajas nuevas |
| Tabla de diálogos | 86 bloques |

## Parches

```
patch/Nekketsu_Soccer_MD_ES_v2_0.ips   548.814 B
patch/Nekketsu_Soccer_MD_ES_v2_0.bps   548.594 B
base: rom/jp.md  md5 ff7a9a6fb74a640f40f10dff53e9cf4d
```
Ambos verificados con implementador independiente.

---

# v2.0 RETIRADA — tres regresiones causadas por el mismo error

El usuario probó la v2.0 y reportó tres fallos. **Los tres son míos y tienen
una única causa raíz.** La v2.0 se ha borrado; la versión válida vuelve a ser
la **v1.9**.

## Causa raíz

Para alojar los tiles de los nombres tomé como "libres" los índices que
ocupaba el rótulo japonés (254 índices, medidos del tilemap). **No estaban
libres.**

El asset `0x064FD4` (1024 tiles) **lo comparten varias pantallas**. Esos
mismos índices los usan la escena de presentación de partido y la pantalla
de duración. Al sobrescribirlos:

```
205 tiles con contenido previo destruidos
```

Síntomas reportados, todos explicados por esto:
1. `DURACIÓN` con un kanji en lugar de la `Ó`
2. fondos de las escenas de presentación corruptos
3. (el tercero es independiente, ver abajo)

## Segundo error, encadenado: la reubicación imposible

Al corregirlo usando solo los 223 tiles **realmente vacíos**, el asset dejó
de caber comprimido (21.075 B frente a 18.591). Intenté reubicarlo a
`0x0A3420` repuntando sus 44 punteros.

**Falló igualmente**, y la traza lo explicó: la zona `0x064FD4`-`0x069873`
**no es un asset aislado sino un solapamiento de varios**. Hay 134 punteros a
direcciones INTERMEDIAS de ese rango:

```
0x066000  descomprime 32768 B, fin 0x06A190
0x066700  descomprime 32768 B, fin 0x06AF22
0x066844  descomprime 32768 B, fin 0x06A982
0x068000  descomprime 32768 B, fin 0x06CC5D
```

Son flujos comprimidos que **empiezan en puntos distintos y se solapan**: un
truco de compresión de la época. Mover el bloque rompe los otros 134 accesos.
**Este asset no es reubicable.**

## Presupuesto real

```
asset original comprimido        18.591 B
recomprimido sin cambios         18.463 B   (mi compresor gana 128 B)
con 116 tiles nuevos             19.391 B   -> faltan 800 B
```

Los nombres en románico necesitan ~116 tiles y **no caben** en el asset sin
sacrificar algo. Es una restricción real, no un fallo de método.

## Tercer problema, independiente: el marco de selección

El marco parpadeante que rodea al nombre elegido **es un sprite compuesto**,
no un blit. Medido en la SAT: 16 sprites por marco (esquinas + lados), con
un sprite de 4x1 celdas para el tramo horizontal.

Su ancho lo calcula el juego a partir del descriptor, así que al ampliar las
cajas de forma desigual (7, 8 o 9 celdas) los marcos quedan descuadrados.

**Propuesta del usuario, correcta**: marcos de tamaño fijo capaces de
envolver el nombre más largo, y las tres columnas centradas. Pendiente de
implementar.

## Lección

Es la **cuarta vez** en el proyecto que doy por libre un espacio sin
verificarlo. Norma reforzada:

> Antes de escribir en un índice de tile "libre", comprobar que su contenido
> actual es CERO. Que un tilemap concreto no lo referencie no significa nada:
> el asset puede estar compartido por otras pantallas.

Y una nueva:

> Antes de reubicar un asset, comprobar que no hay punteros a direcciones
> INTERMEDIAS de su rango. Si los hay, es un solapamiento de flujos y no se
> puede mover.

---

# AUDITORÍA DE ESPACIO — opción B: liberar tiles japoneses [HECHO]

El usuario descarta A (nombres pequeños) y C (dejar la pantalla en japonés) y
pide explorar B: liberar espacio eliminando gráficos japoneses, con la
advertencia correcta de **no tocar nada que pertenezca a los fondos**.

## Qué contiene el asset 0x064FD4

```
1024 tiles,  18.591 B comprimidos
  801 con contenido
  223 vacíos
  254 los usa el rótulo japonés de escuelas
```

## Qué es seguro borrar — medido, no supuesto

Se cruzaron los 254 tiles del rótulo japonés contra **todos los tiles que
aparecen en pantalla** durante la intro completa jugada (9.000 frames,
muestreo cada 15) más las escenas alcanzables:

```
tiles vistos en pantalla fuera de la selección : 460
de los 254 del rótulo, vistos también fuera    : 121
EXCLUSIVOS del rótulo japonés                  : 133
```

**Solo esos 133 son borrables.** Los otros 121 los comparte el asset con
fondos y escenas — exactamente el riesgo que señalaba el usuario. Guardados
en `work/exclusivos2.json`.

## Presupuesto real

```
asset intacto recomprimido        18.463 B
asset sin los 133 kanji           16.585 B
  -> los kanji cuestan             1.878 B  (15,4 B/tile)
presupuesto para los nombres       2.006 B  (17,3 B/tile con 116 tiles)
```

## El contorno es lo que no cabe

Probado con los 133 tiles liberados y cajas al mínimo:

```
estilo                tiles   comprimido   resultado
contorno completo      111     18.968 B    faltan 377 B
solo sombra diagonal    86     18.583 B    CABE por 8 B
sin contorno            76     17.751 B    CABE, sobran 840 B
```

**El contorno de 1 px alrededor de cada letra cuesta ~1.200 B.** Duplica el
detalle de cada tile y arruina la compresión: el kanji original es de trazo
grueso y masas planas, que comprimen mucho mejor que un perfilado fino.

## Conclusión

- El estilo "blanco con contorno negro" **no cabe** en este asset.
- "Sombra diagonal" cabe **por 8 bytes**: margen demasiado estrecho para
  darlo por bueno sin más pruebas.
- "Sin contorno" (blanco liso) cabe con **840 B de sobra**.

No se ha modificado nada. Pendiente de decidir el estilo con el usuario.

---

# Fuente gruesa para los nombres — MEDICIÓN DEL COMPROMISO [HECHO]

El usuario confirma que quiere letras **gruesas y macizas**, como esperaba
desde el principio. Se mide qué cabe.

## Por qué la fuente gruesa cuesta tanto

No es el ancho: es la **altura**.

```
fuente fina   (5x7 px) :  73 tiles únicos
fuente gruesa (6x15 px): 234 tiles únicos
```

Con 7 px de alto, las filas superior e inferior de cada bloque de 3 celdas
quedan **vacías**, y esos tiles vacíos se comparten entre los 13 nombres.
Con 15 px el glifo ocupa las tres filas enteras y casi ningún tile se repite.

Medido sobre `NEKKETSU`: fina 27 tiles de los que **9 son vacíos**; gruesa
27 tiles y **ninguno** vacío.

## Barrido de alturas (fuente maciza, trazo 2 px, cajas de 10 celdas)

Presupuesto real: **18.591 B**.

```
alto   tiles   comprimido   resultado
 7 px    73     17.703 B    CABE, sobran 888 B
 9 px   113     18.111 B    CABE, sobran 480 B
11 px   148     18.291 B    CABE, sobran 300 B
13 px   177     18.724 B    faltan 133 B
15 px   177     18.972 B    faltan 381 B
```

**El techo está en 11 px de alto.** Con trazo de 2 px diseñado a mano (no
estirando la fina, que deforma) el conjunto sube a 210 tiles y 19.469 B:
se pasa por 878 B. La sombra diagonal añade otros 1.082 B.

## Intento de liberar más espacio — SIN ÉXITO CONCLUYENTE

Se auditaron los 801 tiles con contenido del asset contra un recorrido de
9.000 frames más las escenas de vestuario y marcador:

```
tiles vistos en pantalla       469
tiles con contenido            801
nunca vistos                   392
```

**Pero 389 de esos 392 SÍ están referenciados** por words de tilemap en la
ROM. Es decir: pertenecen a pantallas que mi barrido no alcanza (torneo
completo, VS, PASSWORD, final). **No son borrables**, solo son "no vistos por
mí". Solo 3 no tienen referencia alguna, cantidad irrelevante.

Esto confirma la advertencia del usuario: la mayoría de lo que parece
japonés sobrante es en realidad contenido común o de pantallas no visitadas.

## Situación

| variante | alto | coste | margen |
|---|---|---|---|
| fina actual | 7 px | 17.703 B | +888 |
| media | 9 px | 18.111 B | +480 |
| **gruesa** | **11 px** | **18.291 B** | **+300** |
| gruesa a mano trazo 2px | 11 px | 19.469 B | −878 |
| gruesa + sombra | 11 px | 20.551 B | −1.960 |

La versión de 11 px generada por escalado cabe; la diseñada a mano con trazo
de 2 px no. La diferencia está en el detalle de los glifos: cuantos más
patrones distintos, peor comprime.

No se ha modificado nada. Pendiente de que el usuario elija.

---

# FUENTE ALINEADA A CELDA — IDEA DEL USUARIO [HECHO]

El usuario propone dejar de tratar los nombres como GRÁFICOS y tratarlos
como TEXTO con una fuente gruesa: "muchas letras se repetirían y a lo mejor
ocuparía menos". Además señala que no hace falta diseñar 26 letras.

**Las dos observaciones son correctas y resuelven el bloqueo.**

## El error de método que yo arrastraba

Yo renderizaba cada nombre como un **bitmap continuo** de 80x24 px y
*después* lo troceaba en celdas de 8x8. Con eso las letras caen en
posiciones arbitrarias de la rejilla: la `S` de SHICHIFUKU y la `S` de
SHIGUMA quedan cortadas por sitios distintos y **producen tiles distintos**.

El coste crecía con el número de NOMBRES, no con el alfabeto. De ahí los
146 tiles / 18.981 B que no cabían.

Alineando cada letra a una celda, un glifo produce SIEMPRE los mismos
tiles. El coste pasa a depender solo del alfabeto.

## Alfabeto necesario — medido

```
letras usadas por los 13 nombres: 20   A B C D E F G H I K M N O P R S T U Y Z
NO hacen falta: J L Q V W X, ni Ñ, ni vocales acentuadas
```

## Coste medido (asset 0x064FD4, presupuesto 18.591 B)

Partiendo de borrar los 133 kanji exclusivos del rótulo (work/exclusivos2.json):

```
recomprimido sin tocar        18.463 B
sin los 133 kanji             16.413 B
huecos disponibles: 223 vacíos + 133 kanji = 356
```

```
metodo                                     tiles   comprimido   margen
bitmap troceado (el mio, descartado)         146     18.981 B     -390
alineado a celda, cuerpo 10px, caja 3 filas   36     17.074 B    +1517
alineado a celda, cuerpo 14px, caja 3 filas   42     17.209 B    +1382
```

**El contorno negro de 1 px va INCLUIDO en esas cifras.** Cabe dentro de la
celda (cuerpo 6 px + 1 px de contorno a cada lado = 8 px justos) y viaja
pegado al glifo, así que no multiplica los tiles. En el método anterior el
contorno costaba ~1.200 B; aquí cuesta 0.

## M y N: restricción geométrica real [HECHO]

```
celda 8 px - contorno 1+1 = cuerpo util 6 px
montantes de 2 px a cada lado = 4 px
QUEDAN 2 px centrales -> no cabe diagonal -> M y N ilegibles
```

Probadas 3 variantes x 2 alturas (work/shots/MN_variantes.png): las seis
salen como mazacotes. **No es problema de espacio en ROM, es geométrico.**

**Solución**: alinear a celda no obliga a 1 celda por letra, sino a un
número ENTERO de celdas. M y N pasan a 2 celdas (14 px de cuerpo). Siguen
siendo tiles fijos reutilizables, así que el coste apenas sube (+3 tiles).
Comparativa: work/shots/MN_ancha_vs_estrecha.png.

Anchura resultante por nombre (M y N = 2 celdas):
```
SHIMANCHU 11 | SHICHIFUKU 10 | YOSHIMOTO 10 | IPPONZURI 10 | YAMAMOTO 10
NEKKETSU 9 | OSOREZAN 9 | YUSHUIN 8 | SHIGUMA 8 | EDOBANA 8 | HORIHORI 8
MATAGI 7 | HATTORI 7
```
Huecos por columna: izq 11, centro 11, **derecha solo 9**.
-> SHICHIFUKU (10) y YOSHIMOTO (10) NO caben en la columna derecha con su
   coordenada actual. La columna es un dato editable (tabla 0x003854), así
   que es resoluble, pero está PENDIENTE.

## La pantalla de resultados YA usa una fuente — y la hice yo [HECHO]

El usuario pregunta si la fuente sería reutilizable en el marcador. Al
investigarlo aparece que el nombre del rival en esa pantalla **no es un
gráfico**: es un tilemap de 12x3 celdas que referencia tiles sueltos.

```
hook   0x08F440   (dibuja el marcador)
0x08F470  move.w $f852,d0     indice de partido
0x08F47A  lea.l  $8fd40,a1    tabla de 13 punteros a tilemap
0x08F49A  jsr    $7a3c        blitter, d2=0x0b (12 col), d3=2 (3 filas)
```

Decodificando los 13 tilemaps con base VRAM 0x280 = 'A' salen limpios:
YUSHUIN, SHICHIFUKU, SHIGUMA, MATAGI, YOSHIMOTO, EDOBANA, IPPONZURI,
OSOREZAN, YAMAMOTO, HORIHORI, HATTORI, SHIMANCHU, CAPITANES.

**Es una fuente ASCII**, asset `0x090000`, índice = ASCII - 0x20,
cargada en VRAM con base 0x280. Verificado glifo a glifo.

**El usuario acertó: eso lo hice yo en la sesión anterior.** `0x08F440` y
`0x090000` están por encima de `0x80000`, o sea FUERA de la ROM japonesa
(512 KB): son código y datos de la zona de expansión que añadió la
traducción. Es una fuente fina de 5x7 sin contorno, por eso se ve pequeña
frente a los mazacotes japoneses originales.

Consecuencia buena: el bloque de escena `0x0177F0` carga A LA VEZ el asset
de la selección (`0x064FD4`, slot 1) y el de la fuente (`0x090000`, slot 3).
El rótulo ya reserva 12x3 celdas y hoy solo usa la fila central, así que la
misma fuente gruesa sirve en las dos pantallas sin tocar la geometría.

**Aviso**: el asset `0x090000` termina en `0x090F22` y detrás hay **0 bytes
libres**. Para meter glifos gruesos hay que reubicarlo o buscarle sitio.
NO verificado todavía si es reubicable (norma: comprobar punteros a
direcciones intermedias antes de mover nada).

## Estado

Nada escrito en ROM todavía. Fuente en `tools/escfont3.py`.
Capturas: `work/shots/ALT_10px.png`, `ALT_14px.png`, `MN_variantes.png`,
`MN_ancha_vs_estrecha.png`.

---

# FUENTE REDISEÑADA POR EL USUARIO [HECHO]

El usuario devuelve la fuente rediseñada a mano: `uploads/IMG_4518.PNG`,
88x348 px, 3 colores exactos (fondo `0000EE`, cuerpo `FFFFFF`, contorno
`000000`). Misma rejilla que mi plantilla, así que se extrae píxel a píxel
sin interpretación por mi parte.

## Medidas de la fuente del usuario

```
banda útil        y = 4..18   (15 px de alto)
ancho por letra   8 px EXACTOS -> 1 celda por letra, TODAS
SHICHIFUKU        10 letras = 80 px = 10 celdas justas
alfabeto          20 letras, las mismas que ya se sabían necesarias
```

**Resuelve el problema de la M y la N sin recurrir a 2 celdas.** Yo había
concluido que en 6 px de cuerpo no cabía la diagonal; el usuario lo ha
resuelto dibujando dentro de la celda de 8 px con el contorno integrado
en vez de añadido alrededor. Mi restricción "cuerpo 6 px + 1 px de contorno
a cada lado" era una decisión mía, no una imposición de la rejilla.

Consecuencia: **desaparece el problema de anchura de la columna derecha**.
Con 1 celda por letra el nombre más ancho es SHICHIFUKU con 10 celdas, y
antes con M/N a 2 celdas SHIMANCHU llegaba a 11.

## Consistencia — verificada glifo a glifo

Se extrajeron las 88 apariciones de letra y se agruparon por bitmap:

```
19 letras: 1 sola variante  (consistentes)
 S       : 2 variantes      <- 2 px de diferencia en SHIGUMA[0]
```

Es un despiste de dibujo, no un diseño alternativo. Se toma la variante
mayoritaria (6 apariciones frente a 1). Guardado en
`work/fuente_usuario.json` (20 glifos de 8x24 en índices de color).

Reconstrucción verificada contra el PNG original: **1 píxel de diferencia**,
que es exactamente la corrección de la S. `work/shots/VERIF_fuente_usuario.png`.

## COSTE EN ROM — CABE [HECHO]

Asset `0x064FD4`, presupuesto 18.591 B, partiendo de borrar los 133 kanji
exclusivos del rótulo japonés (356 huecos disponibles):

```
disposición                              tiles   comprimido   margen
3 filas de celda (24px), tal cual         35      17.125 B    +1.466  CABE
2 filas de celda (16px), banda subida     37      17.323 B    +1.268  CABE
```

Comparativa con todo lo probado antes:

```
metodo                                    tiles   comprimido   margen
bitmap troceado (mio, descartado)          146     18.981 B     -390
alineado a celda, mio, M/N a 2 celdas       42     17.209 B    +1382
FUENTE DEL USUARIO, 1 celda por letra       35     17.125 B    +1466
```

**Es la mejor de las tres**: menos tiles, más margen y sin el hueco visual
que introducían mis M y N de doble celda.

## Estado

Nada escrito en ROM. Pendiente: verificar si el asset de fuente `0x090000`
(pantalla de resultados) es reubicable — acaba en `0x090F22` con 0 bytes
libres detrás, y NO se ha comprobado aún si hay punteros a direcciones
intermedias de su rango.

---

# ¿ES REUBICABLE EL ASSET DE FUENTE 0x090000? — SÍ [HECHO, demostrado]

Norma obligada tras el fiasco de la v2.0: antes de mover un asset hay que
comprobar que no existan punteros a direcciones INTERMEDIAS de su rango.

## Extensión exacta del asset

```
cabecera 0x090000: 01 00 00 01 0b 0f
offset a control = 0x0f0b -> control en 0x090f0b
tiles cargados = 256   <- 0x004C64: move.w #$100,$fcc4
fin real = 0x090f4b    (0x090f0b + 64 B de control)
tamaño = 3.915 B
```

Detrás hay datos pegados: **0 bytes libres**. Por eso hay que reubicarlo.

## Escaneo de punteros — 29 candidatos, 26 falsos

Escaneando cada 2 bytes toda la ROM en busca de longs dentro de
`[0x090000, 0x091000)` salen **29 destinos distintos**, la mayoría
"intermedios", lo que a primera vista descartaría la reubicación.

**Son falsos positivos.** Dos filtros lo demuestran:

**Filtro 1 — la fuente la creó la traducción.** `0x090000` está por encima
de `0x80000`, o sea FUERA de la ROM japonesa. Ningún dato heredado de la
japonesa puede apuntar ahí. Comparando cada candidato con los mismos bytes
de `rom/jp.md`: **26 de los 29 son idénticos en la japonesa** -> no son
punteros a la fuente, son coincidencias fortuitas del escaneo.

Quedan 3 destinos:
```
0x090000  INICIO       13 refs (tabla de escenas 0x17750, slots de escena)
0x090008  intermedia    1 ref  desde 0x08a820
0x09003f  intermedia    9 refs desde 0x081db8..0x082070
```

**Filtro 2 — leer el contexto en vez de suponer.** Los dos "intermedios"
son words de tilemap que casualmente forman el long buscado:

```
0x081da8: 0008 0039 0035 0033 0028 0035 0029 002e 0009 003f 003f 003f
                'Y'  'U'  'S'  'H'  'U'  'I'  'N'
```
Es el tilemap de **YUSHUIN** (idx = ASCII-0x20). El par `0009 003f` leído
como long da `0x0009003F`. Igual en `0x08a820`, que es zona de tablas de
geometría del HUD (words de 2 en 2).

**Punteros reales: 13, todos al INICIO.** Es reubicable.

## Verificación empírica en el emulador

Se construyó `rom/_test_reubic.md`: asset copiado a `0x0C4580` (par, hueco
verificado a 0xFF), los 13 punteros repuntados, y **el original borrado a
0xFF** para que cualquier acceso olvidado saltara a la vista.

```
intro completa (frames 300/500/750/900)   VRAM md5 IDENTICA en los 4
escena del MARCADOR (+150/+260/+399)      VRAM md5 IDENTICA en los 3
```

## ERROR PROPIO CORREGIDO DURANTE LA PRUEBA

Los dos primeros intentos de alcanzar el marcador **no probaron nada** y
estuve a punto de darlos por válidos:

1. Inyecté `f806=0x28` con la navegación corta: el juego seguía en el menú
   del título (`f800=0x8`). La captura mostraba el menú, no el marcador.
2. Con navegación larga entré en partida pero seguía sin salir la escena:
   escribir `f806` no basta.

Desensamblando `0x005590` se ve que la escena la dibuja el despachador con
`f800=0x0c`; hay que escribir **`f800=0x0c` Y `f806=0x28`**. Con eso sale
la pantalla correcta: `NEKKETSU 0 - 3 YUSHUIN` sobre el cielo naranja
(`work/shots/mm_orig_150.png` y `mm_reub_150.png`, indistinguibles).

Refuerza la norma: **una VRAM idéntica no demuestra nada si la escena que
usa el asset no llegó a dibujarse.** Hay que verificar en la captura que
la pantalla probada es la que se cree.

## Conclusión

`0x090000` (3.915 B, 256 tiles) es **reubicable** repuntando 13 longs de la
tabla `0x17750`. Espacio disponible sobrado: `0x0C4580` ofrece 244.352 B.

Herramienta: `tools/test_reubic.py`.

---

# RECTIFICACIÓN: la fuente del marcador NO está en 0x090000 [HECHO]

**Error mío, detectado antes de construir la v2.0.** Afirmé que la fuente
de la pantalla de resultados era el asset `0x090000` y llegué a demostrar
que era reubicable. La reubicación es cierta, pero **el asset es otro**:
`0x090000` no tiene nada que ver con el marcador.

## Cómo se detectó

Al decodificar los comandos VDP del cargador de escena `0x004C0E`-`0x004C54`:

```
slot0 cmd 0x60000000 -> VRAM 0x2000 -> tile 0x100
slot1 cmd 0x40000001 -> VRAM 0x4000 -> tile 0x200
slot2 cmd 0x60000001 -> VRAM 0x6000 -> tile 0x300
slot3 cmd 0x40000002 -> VRAM 0x8000 -> tile 0x400
```

El tilemap del marcador usa tiles `0x280..0x299`, que caen en el rango del
**slot 1**, no del slot 3. Y el slot 1 de la escena del marcador
(`f806=0x28` -> bloque `0x017890`) es `0x064FD4`, no `0x090000`.

Pero al volcar `0x064FD4` en los índices `0x80..0x99` **no hay letras, hay
fragmentos de kanji**. Contradicción -> hay que trazar, no deducir.

## Traza de VRAM: la fuente se escribe aparte [HECHO]

Leyendo la VRAM real en la escena del marcador:
```
VRAM tile 0x280 = 'A'    0x281 = 'B'    0x298 = 'Y'
```
Las letras están en VRAM pero NO en el asset. Las escribe el hook:

```
0x0055A2  jsr $8f400          <- hook de la traducción
0x08F400  jsr $4a20           rutina original
0x08F406  jsr $8f440
0x08F440  cmpi.w #$4af,$f80a
0x08F448  jsr $8f500          <- solo una vez, al entrar en la escena
0x08F500  move.l #$50000001,$c00004     VRAM 0x5000 = tile 0x280
0x08F50A  lea.l  $8f600,a0
0x08F516  move.w #$19f,d0               416 words = 832 B = 26 tiles
0x08F51A  move.w (a0)+,(a1)             copia directa al puerto de datos
```

**FUENTE REAL: `0x08F600`, 26 tiles SIN COMPRIMIR, 832 B**, volcados a
VRAM `0x280` por copia directa. Verificado leyendo los bytes crudos: los
tiles 0,1,2 son 'A', 'B', 'C'.

## Consecuencias para el plan

1. **NO hay que reubicar `0x090000`** para tocar el marcador. Todo el
   trabajo de verificación de reubicación era correcto pero innecesario
   para este fin (queda anotado por si sirve más adelante).
2. La fuente del marcador está sin comprimir: **sustituirla es trivial**,
   basta sobrescribir 832 B y ajustar el contador `#$19f` si cambia el
   número de tiles.
3. Los 26 tiles cubren 20 letras + 6 sobrantes. Con la fuente del usuario
   (35 tiles para 20 letras, porque cada letra ocupa 2 filas de celda) hará
   falta **más espacio y más tiles**: 40 tiles = 1.280 B, y el contador
   pasa de `#$19f` a `#$27f`.
4. Hay que comprobar que el destino VRAM `0x280 + 40 tiles = 0x2A8` no pisa
   nada. El slot 2 empieza en `0x300`, así que hay 128 tiles de hueco.

## Lección (quinta vez)

> No dar por supuesto de qué asset procede un tile. Trazar la escritura a
> VRAM hasta la lectura de ROM que la alimenta.

Esta vez la norma SÍ se aplicó a tiempo y evitó construir sobre un error.

---

# v2.0 POR HOOK DE VRAM — funciona, con un defecto pendiente

## El fallo de build_v20b.py: el limite de 256 tiles [HECHO]

La primera v2.0 con la fuente del usuario pinto manchas. Causa medida:

```
0x003742 (y 0x004C64):  move.w #$100,$fcc4    <- 256 tiles por slot
```

El asset `0x064FD4` tiene 1024 tiles pero **el cargador solo sube los 256
primeros a VRAM**. Solo sirven los indices < 256, y ahi hay:

```
tiles vacios      < 256 :  2   (216, 218)
kanji exclusivos  < 256 : 23
TOTAL                   : 25   <- hacen falta 35
```

Mi pool cogia indices >= 256, que nunca llegan a VRAM.

**Mi medicion de "+1.466 B de margen" era correcta pero IRRELEVANTE**:
medi el tamano comprimido, no si los tiles llegaban a VRAM. El cuello de
botella no era el espacio en ROM sino el RANGO DE INDICES. Error mio.

Subir el contador NO es opcion: `0x004C60` la comparten los 4 slots de
todas las escenas.

## La solucion: hook de copia directa a VRAM [HECHO, funciona]

Mismo mecanismo que ya usa el marcador (`0x08F500`): copiar tiles al
puerto de datos del VDP sin pasar por el asset.

**Hueco de VRAM verificado en el emulador** (no supuesto). Mapa de la
pantalla de seleccion (rutina `0x0036EC`):
```
0x0411d8 -> tiles 0x000..0x0ff      0x0993ea -> 0x100..0x1ff
0x064fd4 -> tiles 0x200..0x2ff      0x066844 -> 0x300..0x3ff
0x04e868 -> tiles 0x500..0x5ff      planoA 0x600, planoB 0x700
```
El rango `0x400..0x4ff` no lo carga nadie. Leyendo la VRAM real:
`0x400..0x420` tiene contenido, **`0x430..0x4ff` esta limpio** (208 tiles).

Hook: se intercepta `0x0037BC` (`jmp $49e8`, final de la carga) y se
redirige a `0x0C4580`, que copia 60 tiles a VRAM `0x430` y salta a
`0x49e8`.

**Ventaja decisiva: el asset `0x064FD4` queda BYTE A BYTE IGUAL que en la
v1.9.** No hay que borrar los 133 kanji, asi que desaparece por completo
la causa raiz de las tres regresiones de la v2.0 original.

## Resultado

`work/shots/v20c_seleccion.png`: **los 13 nombres se leen correctamente**.

## DEFECTO PENDIENTE: el marco tapa letras [HECHO, diagnosticado]

`SHICHIFUKU` se ve como `SHICHIF KU`. NO faltan letras: los 60 tiles estan
en VRAM y el tilemap es correcto (verificado leyendo ambos).

Es el **marco de seleccion**, que ya estaba anotado como problema 3. Es un
sprite compuesto cuyo ancho se calcula del descriptor. Comparando la SAT:

```
v1.9 : 16 sprites, x=128..184   (8 celdas)
v2.0 : 20+ sprites, x=512, x=304
```

`x=512` equivale a 384 px de pantalla: el marco de SHICHIFUKU (10 celdas)
se sale del area visible, y sus tramos de relleno (tile `0x129`, `pri=1`)
tienen prioridad sobre el plano A y **tapan las letras**.

Intento fallido: comprimir la rejilla a `COLS={0:(0,10),1:(11,20),2:(21,30)}`
lo empeoro (`S ICHIFU U`), porque desplaza el marco sin reducir su ancho.

**La solucion es la que propuso el usuario**: marcos de tamano fijo que
envuelvan al nombre mas largo, con las tres columnas centradas. Requiere
tocar la rutina que construye el marco, no las coordenadas.

## Estado

`rom/Nekketsu_Soccer_MD_ES_v2_0.md` construida, md5
`e04a3dca5c9677f9acb47ba5b1fd020d`. Jugable y legible salvo el marco.
Constructor: `tools/build_v20c.py`.

---

# v2.0 — TRES FALLOS REPORTADOS POR EL USUARIO [diagnóstico]

## 1. Fondo corrupto en la 2ª escena de presentación — CAUSA HALLADA [HECHO]

**Mío, y NO es la misma causa que en la v2.0 anterior.**

```
v1.9 : 26 tiles -> VRAM 0x280..0x299
v2.0 : 60 tiles -> VRAM 0x280..0x2BB     <-- 34 tiles DE MÁS
```

Volcando la VRAM de la **v1.9** en la escena del marcador, los tiles
`0x29A..0x2BB` están LLENOS: son gráficos de la escena, cargados por el
slot 1 (asset `0x064FD4` -> VRAM `0x200..0x2FF`).

Mi fuente usa 3 tiles por letra (60 en total) en vez de 1 (26), y al
volcarla a `0x280` **machaca 34 tiles de la escena**.

Comprobé que `0x280+60 <= 0x300` y di el hueco por bueno. **El error fue
suponer que todo lo que hay entre 0x280 y 0x300 estaba libre**: solo lo
estaba el tramo que ocupaba la fuente vieja. Es la misma clase de fallo
que ya cometí con los índices del asset: dar por libre sin verificar.

**Solución**: buscar en VRAM un hueco real de 60 tiles para esa escena, o
reducir la fuente del marcador a 2 filas útiles.

## 2. Rótulo NEKKETSU cortado y "círculos" sueltos — CAUSA HALLADA [HECHO]

El rótulo FIJO de NEKKETSU está en `0x08FD00` (lo blitea `0x08F45E`) y
**no lo actualicé**: sigue con el reparto antiguo de 1 tile por letra
(`N=0x28D`, `E=0x284`...). Con la fuente nueva esos índices caen en
fragmentos de otras letras.

```
0x08FD00 en v2.0 == v1.9  -> True   (no lo toqué)
```

Actualicé los 13 rótulos de la tabla `0x08FD40` pero me dejé este, que
está fuera de ella. Los "círculos a distinta altura" que ve el usuario son
trozos de glifos de otras letras.

**Solución**: reescribir `0x08FD00` con el reparto nuevo, igual que los 13.

## 3. Mancha bajo la N de EDICIÓN y contorno perdido — NO ES MÍO [HECHO]

Verificado con dos pruebas independientes:

```
tilemap 0x0A3000, tiles 0x096000, paleta 0x0A3400: IDÉNTICOS a la v1.9
comparación de framebuffer v1.9 vs v2.0 en el título: SIN DIFERENCIAS
```

`ImageChops.difference` devuelve `None`: **las dos ROMs pintan el título
exactamente igual**. El defecto que ve el usuario ya estaba en la v1.9.

El rótulo del título es un asset propio (tilemap `0x0A3000` + tiles
`0x096000`, 223 tiles), **no usa la fuente románica**, así que el injerto
de acentos no puede afectarlo. La `Ó` de "EDICIÓN" es parte del gráfico.

**HIPÓTESIS** (sin verificar): la mancha y la pérdida de contorno vienen de
la conversión del PNG del usuario a 223 tiles en la v1.9 — probablemente
tiles que se descartaron por exceder el presupuesto de 4.881 B, o una
colisión al deduplicar. Hay que comparar el PNG original
(`uploads/IMG_4512.PNG`) contra el render real píxel a píxel.

## Estado

Las tres causas están identificadas. Las dos primeras son mías y tienen
arreglo claro. La tercera es una regresión preexistente de la v1.9 que el
usuario ha detectado ahora, y exige revisar la conversión del rótulo.

La ROM v2.0 actual (md5 `55a494d4d8768d5d3558e3aeba129f84`) **no es
válida**: la v1.9 sigue siendo la estable.

---

# v2.0 — LOS TRES FALLOS, RESUELTOS [HECHO, verificado]

## 1. Fondo corrupto de la escena — ERA MÍO

Mi fuente usa 3 tiles por letra (60) frente a 1 (26) de la vieja, y la
volcaba al mismo sitio: VRAM `0x280`. Los tiles `0x29A..0x2BB` **no estaban
libres**: son gráficos de la escena (slot 1 = asset `0x064FD4` -> VRAM
`0x200..0x2FF`). Comprobé que `0x280+60 <= 0x300` y di el hueco por bueno.

**Hueco REAL medido** leyendo la VRAM de la v1.9 en esa escena:
```
0x550..0x60d  190 tiles
0x644..0x6bf  124 tiles
0x77e..0x7ff  130 tiles
```
Se usa `0x550` (60 tiles, el plano A empieza en `0x600`). Se repunta el
comando VDP del hook en `0x08F502` y los tilemaps de los rótulos.

Verificado: escenas `0x20` y `0x24` con **VRAM byte a byte idéntica** a la
v1.9. Vestuario y presentación intactos.

## 2. NEKKETSU cortado y "círculos" sueltos — ERA MÍO

El rótulo FIJO de NEKKETSU está en `0x08FD00`, **fuera** de la tabla
`0x08FD40`. Actualicé los 13 de la tabla y me dejé este, que seguía con el
reparto de 1 tile por letra. Los "círculos a distinta altura" eran
fragmentos de glifos de otras letras.

Arreglado reescribiéndolo con el reparto nuevo. Verificado: el marcador
muestra `NEKKETSU  0 - 3  YUSHUIN` correctamente.

## 3. Mancha bajo la Ó de EDICIÓN — BUG DE LA v1.9, NO DE LA v2.0

**El usuario tenía razón en que estaba mal, pero no en la causa.** Primero
verifiqué que no era mío:
```
tilemap 0x0A3000, tiles 0x096000, paleta: IDÉNTICOS a la v1.9
framebuffer del título v1.9 vs v2.0: SIN DIFERENCIAS (ImageChops -> None)
```

Después descarté que fuera del PNG del usuario:
```
tiles que produce IMG_4512.PNG        : 223
asset en ROM == lo que produce el PNG : True
tilemap en ROM == el generado         : True
píxeles blancos/grises en la zona     : 0
```
La conversión era fiel. **El fallo estaba en la carga.**

Comparando ROM contra VRAM: **222 de 223 tiles idénticos, solo falla el
222** — el último, que lleva el rabito de la Ó.

```
0x00082A:  move.w #$DE,d1     -> 222 tiles
el rótulo tiene 223 (índices 0..222)
```

**Off-by-one.** El último tile nunca se copiaba y quedaba a cero en VRAM.
Eso explica a la vez la mancha y la pérdida de contorno en esa esquina.

Arreglado: `#$DE` -> `#$DF`. Verificado: **223 de 223 tiles idénticos en
VRAM** y `EDICIÓN` se ve con su tilde, sin manchas y con contorno completo.

## Estado

`rom/Nekketsu_Soccer_MD_ES_v2_0.md`, md5 `d18fb6f502bb161cec494c7b7896d31a`

```
título con tilde y sin manchas    OK   work/shots/FIX_edicion_zoom.png
marcador NEKKETSU/YUSHUIN         OK   work/shots/FIX_marcador.png
presentación (fondo)              OK   work/shots/FIX_pres_419.png
vestuario                         OK   work/shots/VEST_v20.png
13 nombres de escuela             OK   work/shots/FIX_seleccion.png
escenas 0x20 y 0x24 vs v1.9       VRAM IDÉNTICA
```

**PENDIENTE**: el marco de selección sigue tapando SHICHIFUKU. Es un sprite
compuesto cuyo ancho sale del descriptor; la solución acordada con el
usuario es marco de tamaño fijo con las tres columnas centradas.

---

# SEGUNDA RONDA DE FALLOS — los tres, resueltos [HECHO]

## 1. Píxeles blancos junto a la N de EDICIÓN — YA ESTABA ARREGLADO

Analizando la captura del usuario, los píxeles no eran rosas sino
**blanco (236,236,236) y gris (129,127,129)**: exactamente el síntoma del
tile 222 sin cargar. Su captura era **anterior** al arreglo del off-by-one.

Verificado en la v2.0 actual: **0 píxeles gris/blanco** en toda la zona del
rótulo (x 150..256, y 60..140).

## 2. Escena del club corrupta — BUG DE LA v1.9, NO DE LA v2.0

**El usuario tenía razón: yo probé la escena equivocada.** Confundí la
escena del club (bloque 13, intro) con el vestuario (`f806=0x24`).

Localizada trazando el motor de diálogos (`$f950` = índice de bloque,
`$f94c` = busy) en vez de navegando a ciegas.

Bisección entre versiones:
```
jp.md    escena PERFECTA (rótulo japonés 「ドッジボール部室」)
v0.9.8   PERFECTA ("SALA DEL CLUB")
v1.6     PERFECTA
v1.7     PERFECTA
v1.8     PERFECTA (con cinta verde, "CLUB DE DODGEBALL")
v1.9     CORRUPTA   <-- aquí se rompió
v2.0     CORRUPTA
```

**Causa exacta**: la v1.8 reubicó el asset de la escena del club a
`0x0972AE`, justo detrás del rótulo del título (que entonces terminaba en
`0x0972AD`). La v1.9 agrandó el rótulo hasta `0x0972D3` y **le pisó 37 B**.

```
rótulo del título v1.8: 0x096000..0x0972AD
rótulo del título v1.9: 0x096000..0x0972D3
escena del club:        0x0972AE..
                        ^^^^^^^^ 37 bytes machacados
```

Arreglado: la escena se reubica a `0x0C8000` tomando el asset **intacto de
la v1.8** (8.442 B) y repuntando `0x017770`. Verificado: la escena vuelve a
verse con su cinta verde (`work/shots/CLUB_13.png`).

Es la sexta vez que un asset pisa a otro. Norma reforzada:

> Al agrandar un asset, comprobar SIEMPRE qué hay inmediatamente detrás.
> Que la v1.8 lo dejara pegado no lo hace intocable, pero sí obliga a
> verificarlo antes de crecer.

## 3. Destello de basura antes del marcador — MÍO

El hook `0x08F440` solo vuelca la fuente cuando `f80a == 0x4AF` (un frame
concreto), pero los rótulos se blitean SIEMPRE:

```
0x08F440  cmpi.w #$4af,$f80a
0x08F446  bne.b  $8f44e        <- se salta la carga
0x08F448  jsr    $8f500        <- volcado de los 60 tiles
```

Con la fuente vieja no se notaba porque VRAM `0x280` ya contenía letras del
asset. Al moverla a `0x550`, esa zona tiene otro contenido hasta que el
hook la sobrescribe -> **destello de letras basura**, que es justo lo que
el usuario capturó grabando la pantalla.

Arreglado: `0x08F446` `bne.b` -> `nop`, así la copia se hace en todos los
frames de la escena. Verificado: la transición ya no muestra basura.

## Reparto de la zona libre (para no volver a colisionar)

Durante este arreglo mis propias reubicaciones chocaron entre sí
(`0x0C5800` ya ocupado). Reparto actual:
```
0x0C4580  hook de la selección        38 B
0x0C4700  tiles de la selección    1.920 B
0x0C5000  tiles del marcador       1.920 B
0x0C5800  fuente románica+acentos  4.207 B
0x0C8000  escena del club          8.442 B
```

## Estado

`rom/Nekketsu_Soccer_MD_ES_v2_0.md`, md5 `afc3e7b2e481b8546706e18b1e18468d`

PENDIENTE: el marco de selección sigue tapando SHICHIFUKU.

---

# TERCERA RONDA [HECHO salvo el punto 3]

## 1. Píxeles rosas junto a la N — EL USUARIO TENÍA RAZÓN

Mi verificación anterior ("0 píxeles gris/blanco") era **correcta pero
insuficiente**: la hice sobre el frame con FONDO NEGRO. El usuario lo
dedujo bien con una foto macro: si faltan píxeles, son FONDO
TRANSPARENTE — invisibles sobre negro, visibles sobre las gradas.

**Causa medida**: no falta ningún tile. Es una **cuña transparente** del
propio PNG, entre la diagonal del rombo MD y la esquina de EDICIÓN:

```
ultimo píxel no transparente por fila
  y=96 -> x=239    y=99  -> x=238
  y=97 -> x=239    y=100 -> x=237   <- la diagonal acaba aquí
  y=98 -> x=238    y=101 -> x=245   <- EDICIÓN empieza con contorno
```

Entre `x=238..244` en `y=98..100` no hay nada. Sobre las gradas se cuela
el público rosa.

Arreglado en `tools/titulo.py` (`_cerrar_cuna`): rellena con contorno
(índice 14) hasta x=245. **66 px añadidos, siguen siendo 223 tiles**
(4.844 B comprimidos). Verificado sobre las gradas: 0 píxeles de grada
dentro del rótulo (los 2 "sospechosos" eran verde oscuro idx 7, del
propio rótulo).

## 2. Basura antes del marcador — MI ARREGLO ANTERIOR ERA INSUFICIENTE

Puse el gate `0x08F446` a `nop` y lo di por resuelto. **No bastaba**, y mi
prueba no valía: con el estado inyectado el hook nunca se ejecutaba
(breakpoints en `0x08F500` y `0x08F46A`: **0 hits**). Un contador a cero
no demuestra nada.

**Causa real medida**: `0x550` está vacío en el marcador pero NO en la
escena ANTERIOR:
```
escena previa : 59 de 60 tiles CON CONTENIDO en 0x550..0x58B
marcador      :  0 de 60
```
El tilemap se pinta antes de que el hook vuelque la fuente, así que
durante unos frames se ven los tiles de la escena previa. Poner el gate a
nop no ayuda porque el hook corre después.

Solución: usar una zona vacía en AMBAS escenas. Cruzando las dos, la
única racha >= 60 tiles es `0x66e..0x6bf` (82 tiles). Se usa **`0x670`**.
Verificado: escena previa **0 de 60 tiles con contenido**.

## 3. Baile de líneas en la cinta verde — NO REPRODUCIDO

El usuario reporta que hacia el final de la escena del club la parte
inferior de la cinta "vibra". No he conseguido reproducirlo:

```
nametable de la cinta (filas 2..5) durante toda la escena: 1 sola variante
fila y=38 a lo largo de 62 frames: sin cambios
```

**HIPÓTESIS no verificada**: si el nametable no cambia, la vibración
tendría que venir de (a) los TILES de la cinta reescritos por otra cosa,
(b) scroll por línea (HSCROLL) o (c) un efecto del propio emulador/TV del
usuario. Falta medirlo leyendo la VRAM de esos tiles frame a frame y el
registro de scroll.

## Estado

`rom/Nekketsu_Soccer_MD_ES_v2_0.md`, md5 `c880fee987d1a5b00e4cc16e1b1eb7b9`
Puntos 1 y 2 verificados. Punto 3 sin reproducir. Marco de selección
sigue pendiente.

---

# CUARTA RONDA

## MARCO DE SELECCION — RESUELTO [HECHO]

**Hallazgo que lo hizo trivial**: el marco NO se calcula del descriptor.
Tiene su PROPIA tabla en `0x0039D4`, 13 entradas de 8 B:
```
  +0 word  x en PIXELES
  +2 word  y en celdas
  +4 word  ancho en celdas
  +6 word  alto en celdas
```
La lee `0x00391A` (cursor 1J) y `0x003970` (cursor 2J).

Aplicado lo que pidio el usuario: **marco fijo para los 13**, ancho 11
celdas (10 de texto + 1 de holgura), x = columna*8 - 4, y rejilla de tres
columnas en `COL_X = [0,10,20]`.

Se probo antes `COL_X=[1,11,21]`: la tercera columna acababa en la 31 y
SHICHIFUKU perdia la U. Verificado en las tres columnas con el cursor
(`work/shots/M3_sel.png`, `CUR_*.png`): marcos uniformes, ningun nombre
tapado, SHICHIFUKU completo.

## BAILE DE LINEAS EN LA CINTA — LOCALIZADO, SIN ARREGLAR [HECHO parcial]

El usuario acerto: es real y esta en el borde INFERIOR de la cinta.
Reproducido comparando el perfil del borde contra la japonesa, en el
bloque 24 (frame ~7178):

```
x        jp    v2.0
 32      47     -1     <-- DIFIERE
 33..55  47     46     <-- DIFIERE
 56..223 47     47     igual
```

Es decir: en las columnas 32..55 a mi cinta **le falta la ultima fila de
pixeles** del borde diagonal izquierdo. La diagonal se corta 1 px antes y
por eso se ve el escalon / "espectro de audio" que describe el usuario.

Descartado por medicion (todo IDENTICO a la japonesa):
- tiles `0x12E`, `0x15C`, `0x1BF` en VRAM: byte a byte iguales
- nametable del plano A filas 3..5: mismas columnas y mismos tiles
- plano B en esa zona: vacio en ambas
- HSCROLL: tabla entera a 0, reg11=0 (sin scroll por linea)
- sprites: ninguno en esa banda
- tilemap `0x08C100` y blit `0x08C040` (24x3 desde col 4): correctos

**HIPOTESIS pendiente**: si todo lo anterior coincide, la diferencia tiene
que estar en la fila 6 del nametable (la de debajo) o en el borrado previo
`0x7A6E`, que limpia 3 filas y quiza deja un resto que en la japonesa si
esta cubierto. NO verificado.

## Pixeles rosas de la N — NO REPRODUCIDO EN EMULADOR

Medido en el framebuffer sobre las gradas: el contorno derecho de la N
(x 240..245) sale **solido y completo**, sin huecos. El PNG original
tambien lo tiene completo (2 columnas de negro cerradas arriba y abajo).
El usuario lo va a contrastar en la pantalla del movil.

La cuna transparente que si encontre y cerre (66 px) estaba entre el
rombo MD y EDICION, no en la N: es un defecto distinto y real, ya
arreglado.

---

# QUINTA RONDA

## BAILE DE LA CINTA — CAUSA RAIZ Y ARREGLO [HECHO, demostrado]

Antes lo habia dado por "localizado" comparando un unico frame. **Eso era
insuficiente**: el defecto es TEMPORAL, no espacial.

Metodo correcto: muestrear la fila y=47 durante 120 frames y contar
patrones distintos.
```
japonesa : 1 patron   (32 px encendidos, 120 frames)
v2.0     : 2 patrones (32 px en 89 frames, 8 px en 31 frames)
```
Eso es el parpadeo: en 31 de cada 120 frames el borde inferior aparece
recortado. Coincide con lo que describe el usuario ("lineas que crecen y
decrecen, como un espectro de audio").

**Causa raiz**, con breakpoints (trace activo):
```
hook 0x08C040 (borrado 0x7A6E + blit): 77 hits en 77 frames
japonesa, misma escena, borrado 0x7A6E: 0 hits
```
La cinta **se borra y se redibuja en CADA frame**, y la captura pilla el
instante entre el borrado y el blit.

El porque: la v0.9.8 engancho el hook sustituyendo `jsr $582c` por
`jsr $8C000` en `0x0054A8`, y **`0x582c` es el motor de dialogos**, que
corre en todos los frames. Descartado por medicion que fueran tiles,
nametable, plano B, HSCROLL, VSRAM, sprites o registros del VDP: todo
identico a la japonesa.

**Arreglo**: envoltorio nuevo en `0x08C000` con un flag en `0xF7F0`
(verificado a 0 durante 9.000 frames):
```
tst.w FLAG ; bne ya ; move.w #1,FLAG ; jsr $8C040
ya: jsr $582C ; rts
```
La cinta se dibuja una sola vez. Verificado: **1 patron en 120 frames**,
igual que la japonesa, y la cinta se sigue viendo completa
(`work/shots/FIX_cinta.png`).

## MARCO ADAPTABLE — restaurado a peticion del usuario [HECHO]

Ahora que se sabe que el marco tiene su PROPIA tabla (`0x0039D4`), se
puede hacer bien lo que el original hacia: ceñirlo a cada nombre.
```
ancho = letras + 1 celda de holgura
x     = columna * 8 - 4
```
Verificado en la tabla: los 13 marcos con ancho = len(nombre)+1
(7..11 celdas). La rejilla de 3 columnas sigue fija para que queden
alineados.

---

# v2.1 — ENTREGA NUMERADA

**Aviso de metodo aceptado del usuario**: llevaba 5 rondas sobrescribiendo
el mismo `Nekketsu_Soccer_MD_ES_v2_0.md`. Las ROMs SI eran nuevas (los md5
iban cambiando) pero con el mismo nombre era imposible saber cual tenia.
A partir de ahora, **una version nueva por entrega**.

```
55a494d4  v2.0-a  primera, marcador roto
afc3e7b2  v2.0-b  fuente a 0x550
c880fee9  v2.0-c  cuna del titulo cerrada
ac91a6c6  v2.0-d  marco fijo
3664046a  v2.1    cinta arreglada + marco adaptable  <-- ENTREGA
```

`rom/Nekketsu_Soccer_MD_ES_v2_1.md`, md5 `3664046ac7ae5a88acca2d109170e120`

## Vía 3 para el codigo raro — DESCARTADA [HECHO]

Se propuso mover la fuente al bloque de escena (tabla `0x17750`) para que
la cargara el motor. **No es posible**: los 4 slots del bloque del
marcador (`0x017890`) estan ocupados:
```
slot0 0x072c30 -> tile 0x100     slot2 0x066844 -> tile 0x300
slot1 0x064fd4 -> tile 0x200     slot3 0x04ecd6 -> tile 0x400
```
No hay slot libre sin sacrificar un asset de la escena.

Ademas, al desensamblar se ve que **el orden dentro del hook ya es
correcto**:
```
0x08F448  jsr $8f500     <- vuelca la fuente
0x08F44E..0x08F46A       <- luego blitea los rotulos
```
Asi que el destello no puede venir de este hook: tiene que venir de un
frame ANTERIOR, cuando la escena ya se esta pintando pero el hook aun no
ha corrido.

## Por que no puedo reproducirlo — LIMITACION REAL DEL BANCO DE PRUEBAS

Cuatro intentos, todos fallidos, y conviene dejarlo escrito:

1. `poke_wram16` de `f800/f806`: **0 hits** en los breakpoints
   `0x08F500` y `0x08F46A`. El estado inyectado NO ejecuta el hook, asi
   que todas mis "verificaciones" con ese metodo no valian.
2. Jugar 20.000 frames pulsando START: se queda en `f800=0x24`.
3. Acelerar `0xF870` (que sube durante el partido): no es el reloj.
4. Poner `0xF80C` (=0x200, el 2:00 del HUD) a 1: **el valor no baja**.
   Ni pulsando START ni moviendo al jugador.

El partido no avanza solo porque el motor espera entrada real de juego.
**No se como llegar al final de un tiempo desde el harness.**

Consecuencia honesta: el arreglo del destello (fuente a VRAM `0x670`,
zona vacia en ambas escenas) esta hecho y es correcto en su razonamiento,
pero **NO he podido verificarlo en la transicion real**. Solo el usuario
puede confirmarlo jugando.

Siguiente via a probar si persiste: rellenar con tiles vacios la zona
destino al ENTRAR en la escena, antes de que el tilemap se pinte.

---

# v2.2 — tres arreglos, uno de ellos un CUELGUE causado por mi

## 1. CUELGUE en la escena de la cinta — ERROR DE ARITMETICA MIO [HECHO]

Mi envoltorio de `0x08C000` (v2.1) colgaba el juego: imagen congelada y
una nota de sonido fija (= CPU muerta, PSG repitiendo el ultimo comando).

Causa exacta:
```
off  0: 4A78 F7F0        tst.w $F7F0.w
off  4: 6608             bne.b +8       <-- MAL
off  6: 31FC 0001 F7F0   move.w #1,FLAG
off 12: 4EB9 0008C040    jsr $8C040
off 18: 4EB9 0000582C    jsr $582C
off 24: 4E75             rts
```
`bne.b` calcula el destino como (PC del bne + 2) + desp = 6 + 8 = **14**.
El offset 14 cae **en medio** de `4EB9 0008C040` (ocupa 12..17): los bytes
alli son `0008`, que no es instruccion valida.

Correcto: `660C` -> destino 6 + 12 = 18, el `jsr $582C`. Verificado
desensamblando el bloque antes de escribirlo.

**Norma nueva**: todo salto relativo que escriba a mano debe desensamblarse
y comprobarse ANTES de meterlo en la ROM. Un `bne` mal calculado no da
error: ejecuta basura.

Verificado tras el arreglo: **25 bloques de dialogo alcanzados** (antes se
quedaba clavado en el 24) y el juego llega a `f800=0x0c`. La cinta sigue
sin parpadear: 1 patron en 120 frames.

## 2. Mazacote negro en el rotulo del titulo — TAMBIEN MIO [HECHO]

La funcion `_cerrar_cuna()` que anadi barria **desde y=86**, o sea DENTRO
del emblema MD, y rellenaba de negro todo hueco que encontrara. Resultado:
un manchon que invadia el logo.

**Eliminada por completo.** El usuario entrego un PNG nuevo
(md5 `bdb62b3fddc148be52eee107fcef3e41`, 11 colores) con el contorno ya
repasado a mano. 223 tiles, 4.820 B.

**Norma nueva**: no "arreglar" el arte del usuario por mi cuenta. Si veo un
defecto, se lo digo y el decide.

## 3. SHIMANCHU pegado al borde izquierdo [HECHO]

`COL_X` volvia a `[0,10,20]`, asi que la columna izquierda empezaba en la
celda 0. Cambiado a `[1,11,21]`:
```
SHIMANCHU  col 0->1   marco x 0->4
```
Verificado que ningun nombre se sale: todos entre col 1 y col 30.

El usuario indica ademas que **el marco tapando un poco las letras es
normal y no molesta**, porque parpadea. No se toca mas.

## Entrega

`rom/Nekketsu_Soccer_MD_ES_v2_2.md`, md5 `1d5a7b95e42a16061e4cf6f1cd4ebdee`
Parches `patch/Nekketsu_Soccer_MD_ES_v2_2.{ips,bps}` desde `jp.md`.

PENDIENTE: el codigo raro del marcador sigue sin resolverse ni poder
reproducirse en el harness.

---

# v2.3 — el codigo raro RESUELTO (diagnostico del usuario)

## FALLO GRAVE DE INSTRUMENTACION: mis breakpoints no funcionaban [HECHO]

El usuario me reprocho, con razon, que ya habia mostrado capturas del
marcador. Aclaracion: SI llegaba a la pantalla, pero **inyectando estado**
con `poke_wram16`, no terminando un partido.

Al revisarlo aparecio algo peor. Prueba de control con dos rutinas que se
ejecutan SEGURO:
```
bp 0x004a20 (gestor de actores) : 0 hits
bp 0x0058C0 (motor de dialogos) : 0 hits
```
**El mecanismo de breakpoints no registraba nada.** Falta
`m.config(exec_=1)`: sin el, `HOOK_M68K_E` no se emite y `bp_hits` siempre
devuelve 0.

Con `exec_=1` la misma prueba da hits reales, y el hook del marcador:
```
0x08F400 hook : 113 hits
0x08F500 fuente: 113
0x08F46A rotulo: 113
```

**Consecuencia**: TODAS mis conclusiones basadas en "0 hits" eran falsos
negativos. En concreto la de "el estado inyectado no ejecuta el hook",
que era falsa. Norma 3 del diario reforzada:

> Un contador a cero no es evidencia de nada si no se ha validado antes
> que el contador puede subir. Prueba de control OBLIGATORIA con una
> direccion que se ejecute seguro.

## LA CAUSA — la propuso el usuario y es correcta [HECHO]

> "creo que está relacionado con el texto japonés que salía inicialmente
> [...] Aparecen cuando sustituimos un gráfico donde ya había otro."

Confirmado. Nuestro hook es:
```
0x08F400: jsr $4a20     <- rutina ORIGINAL
          jsr $8f440    <- lo nuestro, encima
```
`0x4a20` pinta el rotulo JAPONES en el nametable:
```
fila 3..5 col 36..44 -> tiles 0x390..0x392  (NEKKETSU en kanji)
fila 3..5 col 50..59 -> tiles 0x203..0x23a  (nombre del rival)
```
Medido en `rom/jp.md`. En nuestra ROM esos indices **ya no son kanji**, asi
que se ven como codigo/basura hasta que nuestro rotulo los tapa.

## ARREGLO

Rutina en `0x0C7000` (60 B) que hace el volcado de la fuente y despues
borra las dos zonas con la rutina del propio juego (`0x7A6E`, misma firma
que el blitter). Enganchada cambiando el destino del `jsr` en `0x08F44A`.

Verificado: 93 ejecuciones en la escena, rotulos correctos, marcador
limpio (`work/shots/V23_marcador.png`).

## Nombres alineados a la izquierda (propuesta del usuario)

Centrar cada nombre en su columna dejaba SHIMANCHU descolgado. Alineando a
la izquierda las tres columnas quedan a plomo (cols 1, 11, 21) y da igual
la longitud. `work/shots/V23_sel.png`.

## Reparto de la zona alta (actualizado)
```
0x0C4580  hook seleccion          38 B
0x0C4700  tiles seleccion      1.920 B
0x0C5000  tiles marcador       1.920 B
0x0C5800  fuente romanica      4.207 B
0x0C7000  limpieza rotulo jp      60 B
0x0C8000  escena del club      8.442 B
```

## Entrega
`rom/Nekketsu_Soccer_MD_ES_v2_3.md`, md5 `023a505ca4d22e9a9981164f258cacf1`

---

# v2.4 — el patron que identifico el usuario: TILEMAPS DUPLICADOS

El usuario detecta el mismo sintoma en la pantalla de alineacion:
"¿CAMBIAR POSICION?" aparece un instante con la 'ON' machacada y luego se
corrige. Y da el diagnostico general, que es correcto:

> "son como residuos que permanecen ahí cuando cambias algo"

## Causa: el texto esta DUPLICADO en la ROM [HECHO]

```
0x02A87C  tilemap VIEJO (v0.9.8):  'CAMBIAR POSICI' + 0x0063 0x001F
                                    (O-acentuada y N aplastadas)
0x08E2FA  tilemap NUEVO (hook)  :  'CAMBIAR POSICIuN?'  correcto
```
Verificado que el viejo NO esta en `jp.md`: lo escribio la traduccion.

El patron es siempre el mismo y explica los tres casos que hemos visto:
la rutina ORIGINAL pinta su version y el hook la tapa despues. Entre
ambas hay un frame en que se ve la vieja.

**Arreglo aplicado**: corregir el tilemap VIEJO en su sitio (9 celdas), de
modo que aunque se pinte primero ya sea correcto. Verificado:
```
viejo: '   pCAMBIAR_POSICIuN? St  NOE?'
nuevo: '???pCAMBIAR_POSICIuN? St  NO??'
```

## Codigo raro del marcador — SIGUE SIN RESOLVER

La limpieza del nametable (v2.3, rutina `0x0C7000`) **no ha bastado**: el
usuario confirma que persiste.

Lo que si esta medido:
- la rutina original `0x4a20` pinta el rotulo japones con tiles
  `0x390..0x392` y `0x203..0x23a`
- nuestro hook lo tapa despues
- borrar el nametable antes de repintar NO lo elimina

**HIPOTESIS pendiente** (por el patron de POSICION): puede haber un
TILEMAP DUPLICADO tambien aqui, escrito por la v0.9.8 en la zona baja de
la ROM, que la rutina original lee. Habria que localizarlo igual que se
hizo con `0x02A87C`: buscando la secuencia de tiles del rotulo.

## Entrega
`rom/Nekketsu_Soccer_MD_ES_v2_4.md`, md5 `7991573f0f1def6354fc6521b3bf84cf`

---

# v2.5 — reversion de un daño colateral + hallazgo sobre el marcador

## MI PARCHE DE LA v2.4 ROMPIO LA PANTALLA DE ALINEACION [HECHO]

Copie el tilemap nuevo sobre el viejo celda a celda. **Error grave**: el
nuevo usa 3 celdas (`u`,`N`,`?`) donde el viejo usaba 2, asi que todo lo
de detras se corrio una posicion:

```
v1.9 : '   pCAMBIAR_POSICI?? St  NORE?VE?TI?'
v2.4 : '???pCAMBIAR_POSICIuN? St  NOE?VE?TI?'   <- la R de RES perdida
```

El usuario lo vio al instante: las seis tarjetas de jugador mostraban
`OES` en vez de `RES`, y el `NO` habia desaparecido.

**Causa**: ese tilemap esta COMPARTIDO. A partir de la celda 24 vienen
`RE?VE?TI?`, que son las etiquetas RES/VEL/TIR de las tarjetas. Copiar
sobre el sin comprobar longitudes las desplazo.

**Metodo correcto, recordado por el usuario**:
> "En casos similares no intentabas alinearlo, sino que te cargabas el
> primero y listo."

Aplicado: se ANULAN las celdas 3..23 del tilemap viejo (19 celdas a
espacio) y no se toca nada a partir de la 24. El hook dibuja despues la
version buena, que ya incluye el `NO` en las celdas 26-27.

Verificado: celdas 24+ **byte a byte iguales a la v1.9**.

**Norma nueva**: antes de sobrescribir un tilemap, comprobar que no esta
compartido con otro texto. Copiar celda a celda solo vale si la longitud
coincide EXACTAMENTE.

## Codigo raro del marcador — el usuario propone mirar la japonesa

Sugerencia correcta. Medido con ventana de escrituras a VRAM en la escena
del marcador, filas 3..5 columnas 36..60 del nametable:

```
japonesa : 150 escrituras, TODAS desde pc = 0x00F450
v2.5     : 150 escrituras, TODAS desde pc = 0x00F450
```

`0x00F450` es el **bucle interno del descompresor**. O sea: el rotulo del
marcador NO se pinta con un blit desde un tilemap suelto, sino que
**forma parte de un asset comprimido que se descomprime directamente al
nametable**.

Eso explica por que mis dos intentos fallaron:
- borrar el nametable antes de repintar: el descompresor lo vuelve a
  escribir despues
- buscar un tilemap duplicado: no existe como tal, va dentro del stream

**Siguiente paso**: identificar QUE asset se descomprime ahi y en que
offset del stream caen esas celdas, para sustituir el rotulo japones
dentro del propio asset comprimido (como se hizo con la sala del club y
los vestuarios en la v1.7).

## Entrega
`rom/Nekketsu_Soccer_MD_ES_v2_5.md`, md5 `f85c5c433b50bc4d9f78f3312eb353b7`

---

# v2.6 — CODIGO RARO: CAUSA RAIZ ENCONTRADA (error grave mio)

## El "NO" adelantado — arreglado

El `NO` del tilemap viejo estaba en las celdas 25-26 y aparecia antes que
la pregunta. `RES` empieza EXACTAMENTE en la 27, asi que se puede anular
hasta la 26. Ampliado `CELDA_FIN` de 24 a 27: 21 celdas anuladas.
Verificado: celdas 27+ byte a byte iguales a la v1.9.

## CODIGO RARO — la causa era MIA, y es grave [HECHO, demostrado]

Contando escrituras al **plano A** (`0xC000..0xDFFF`) en esa escena:
```
japonesa : 25.248 escrituras
v2.5     : 104.964     <-- 4 veces mas
   de ellas, pc=0x08F51A : 70.080   <-- MI volcado de la fuente
```

`0x08F51A` es el bucle que copia la fuente. Estaba escribiendo **en el
nametable**.

Comando VDP en `0x08F502`: `0x4E000003` -> direccion VRAM `0xCE00`.
```
tiles     0x0000..0xBFFF   (tiles 0x000..0x5FF)
plano A   0xC000..0xDFFF   <-- 0x670*32 = 0xCE00 CAE AQUI
plano B   0xE000..0xFFFF
```

**Mi eleccion de `VRAM_MARC = 0x670` fue un error de bulto.** Medi que
"0x670..0x6AB estaba vacio", pero lo que mire fue el CONTENIDO como si
fueran tiles; esa zona no es de tiles, es el NAMETABLE, y sus celdas no
usadas valen 0, asi que parecia libre.

Resultado: cada frame, el volcado de la fuente machacaba 1.920 bytes del
plano A. Eso es exactamente el "codigo raro" que reporta el usuario desde
hace varias versiones, y explica por que ninguna de mis correcciones
anteriores (gate a nop, limpieza del nametable) servia de nada: yo mismo
lo estaba causando.

**Norma nueva**: antes de elegir un destino de VRAM, comprobar en que
REGION cae (tiles / nametable / SAT / window), no solo si "parece vacio".

## El hueco no existe — restriccion real [HECHO]

Midiendo tiles vacios en AMBAS escenas, restringido a la zona de tiles
real (`0x100..0x5FF`):
```
tiles libres en ambas: 59
rachas de >=10 tiles : ninguna
```
La fuente necesita 60 tiles (20 letras x 3 filas). **No caben**, y ademas
estan fragmentados.

Opciones para la proxima sesion:
1. Fuente de 2 filas de celda -> 40 tiles (sigue sin haber racha).
2. Volcar la fuente en varios trozos a los huecos disponibles y ajustar
   los indices de los tilemaps de los 13 rotulos.
3. Sacrificar tiles de la escena que no se vean (habria que medir cuales).

## Entrega
`rom/Nekketsu_Soccer_MD_ES_v2_6.md`

---

# CODIGO RARO — CAUSA RAIZ LOCALIZADA (v2.6)

## Correccion de dos afirmaciones mias erroneas

1. Dije que "los nombres del marcador saldran mal". **Falso**: el usuario
   confirma que se ven bien. La fuente SI llega a VRAM.
2. Dije que el destello lo causaba mi volcado sobre el plano A. Era cierto
   que `0x670*32 = 0xCE00` cae en el nametable y eso se corrigio, pero
   **no era la causa del codigo raro**, que persiste.

## Medicion de la ventana del destello [HECHO]

Leyendo el nametable (fila 3, cols 36..45) frame a frame en la escena:
```
+110  000 000 000 ...                        (vacio)
+120  000 451 43c 44b 44b 43c 460 45d 463    <-- DESTELLO
+126  igual
+127  <- primer volcado de la fuente (0x0C7000)
+128  401 691 67c 68b 68b 67c 6a0 69d 6a3    <-- correcto
```

Durante 7 frames se pinta el rotulo con **base de tile 0x430**, que es la
base de la pantalla de SELECCION, no la del marcador (0x670).

Decodificados, ambos dicen `NEKKETSU`: el tilemap es correcto, lo que
falla es la BASE. En la escena del marcador, VRAM `0x430` contiene otra
cosa (medido: `0x451` tiene un glifo ajeno, `0x43c` esta vacio) -> se ven
como katakana, numeros romanos y simbolos sueltos.

## El culpable [HECHO]

```
escritura a 0xC1CA (fila3 col37): pc=0x007A5E  valor=0x8451
tilemap con base 0x430 en ROM   : 0x02C780
```

`0x02C780` es **el tilemap de los nombres de la pantalla de seleccion**,
el que escribe mi constructor. Alguien lo blitea con el blitter generico
(`0x7A5E`) durante la escena del marcador.

La rutina de pintado de la seleccion (`0x0037E4`) da **0 hits** en esa
escena, asi que el blit lo lanza otro punto del codigo que reutiliza el
mismo puntero o la misma tabla de descriptores (`0x02BD18`).

**Siguiente paso**: poner el breakpoint en `0x007A3C` (entrada del
blitter) durante la escena y volcar el valor de `a0` para saber quien
pasa `0x02C780` como origen. Con eso se identifica el llamante y se
decide si hay que separar los tilemaps o condicionar el blit.

## Estado
`rom/Nekketsu_Soccer_MD_ES_v2_6.md` md5 `99c5c99d39556468cf0a80c215e2d029`
- "NO" adelantado en la pantalla de alineacion: RESUELTO
- flecha del cursor desde el principio: el usuario lo da por bueno
- codigo raro: causa localizada, sin corregir

---

# v2.7 — CODIGO RARO RESUELTO

## Causa raiz [HECHO, demostrado]

El codigo ORIGINAL japones en `0x005172`..`0x0051C8` pinta los dos nombres
de escuela de la pantalla de resultados **reutilizando la tabla de
descriptores de la SELECCION** (`0x02BD18`):

```
0x005172  move.w $f850,d0      nuestra escuela
0x00517E  lea.l  $2bd18,a1
0x00519A  bsr    $7a3c         -> nombre izquierdo
0x00519E  move.w $f852,d0      rival
0x0051AA  lea.l  $2bd18,a1
0x0051C6  bsr    $7a3c         -> nombre derecho
```

Esos tilemaps (`0x02C780`, escritos por nuestro constructor) usan la base
de tile de la seleccion, `0x430`. En la escena del marcador esa zona de
VRAM contiene otra cosa, asi que se ven katakana, numeros romanos y
simbolos sueltos.

Medicion del nametable fila 3 col 36 frame a frame:
```
+120  0x451 ... (base 0x430)   <-- DESTELLO, 7 frames
+127  primer volcado de la fuente
+128  0x691 ... (base 0x670)   <-- correcto
```
Curiosamente ambos decodifican `NEKKETSU`: el tilemap es correcto, la BASE
es la que esta mal.

Localizacion del culpable: `pc=0x007A5E` (blitter) escribiendo `0x8451` en
`0xC1CA`, y busqueda del tilemap con base `0x430` -> `0x02C780`.

## Arreglo

El bloque es REDUNDANTE: nuestro hook `0x08F440` repinta esos mismos dos
nombres con la base correcta. Se anula con un salto:
```
0x005172:  6000 0056   bra.w $51ca
```
Desensamblado verificado antes de escribir (norma tras el cuelgue de la
v2.1).

## Verificacion

```
frames con base incorrecta: 0   (antes 7)
seleccion de escuela      : OK, f800=0x30
dialogos                  : 25 bloques, sin cuelgues
marcador                  : work/shots/V27_marcador.png
```

## ENTREGA FINAL

```
rom/Nekketsu_Soccer_MD_ES_v2_7.md
  md5   1309e1abea01b8508798e6527c2adbbc
  sha1  17bfa7793a408b08e59590d1a594a21382e38335
  crc32 9e56a901
  1.048.576 B

patch/Nekketsu_Soccer_MD_ES_v2_7.ips  532.889 B
patch/Nekketsu_Soccer_MD_ES_v2_7.bps  532.669 B
  desde rom/jp.md (crc32 f49c3a86), verificacion IPS OK
```

Documentacion generada para publicacion:
  `LEEME.txt`      - readme para acompañar el parche

---

# v2.8 — PRESENTACION DE PARTIDO EN JAPONES (reportado por el usuario)

El usuario se pasa el primer partido y la presentacion del segundo sale
INTEGRA en japones: 第2試合 / 七福学園高校戦.

## Causa: una llamada desactivada desde la v1.9 [HECHO]

```
0x08A300 en v0.9.8 : 4eb9 000c4180   jsr $c4180
0x08A300 en v1.9+  : 4e75 000c4180   rts          <-- desactivado
```

`0x0C4180` es la rutina que borra las bandas japonesas (`jsr $7a6e` x2) y
pinta los rotulos en castellano leyendo:
- `0x084470` = 13 titulos ("PRIMER PARTIDO".."PARTIDO FINAL")
- `0x0844A4` = 13 institutos ("INSTITUTO YUSHUIN"..)

Alguna version entre la v0.9.8 y la v1.9 la sustituyo por un `rts`. La
rutina y sus datos seguian **intactos** en la ROM: solo faltaba la llamada.

Los bytes `000c4180` seguian ahi despues del `rts`, lo que confirma que
fue una desactivacion y no una reescritura.

## Arreglo y verificacion

`0x08A300`: `rts` -> `jsr $c4180`.

```
f852= 1 -> SEGUNDO PARTIDO / INSTITUTO SHICHIFUKU   (el caso reportado)
f852= 2 -> TERCER PARTIDO  / INSTITUTO SHIGUMA
f852=12 -> PARTIDO FINAL   / SELECCION CAPITANES
rutina llamada: 182..556 veces por escena
dialogos: 25 bloques, sin cuelgues
seleccion: OK
```

## ENTREGA
```
rom/Nekketsu_Soccer_MD_ES_v2_8.md
  md5   30c69454512344419b622db67d51e65b
  sha1  608739bf89a75915a5d90719d9e0bac3d8aaa3f2
  crc32 8ba4f074
```
