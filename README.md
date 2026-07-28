# NEKKETSU KOUKOU DODGEBALL-BU — SOCCER HEN MD

### Traducción al castellano · v2.8

**Mega Drive / Genesis · Technos Japan, 1992 · Japón (exclusivo)**

---

## 1. Qué es esto

Parche de traducción al castellano de *Nekketsu Koukou Dodgeball-bu -
Soccer Hen MD*, el fútbol arcade de Kunio-kun que Technos publicó en 1992
solo en Japón.

El juego nunca salió de Japón y nunca tuvo traducción oficial ni de
aficionados a ningún idioma occidental. Esta es, hasta donde sabemos, la
primera versión jugable en un idioma que no sea japonés.

Se ha traducido **todo** el texto visible: menús, diálogos de la historia,
nombres de las trece escuelas, rótulos de escena, marcador, pantalla de
alineación y créditos.

---

## 2. ROM necesaria

El parche se aplica sobre la ROM japonesa original, **sin cabecera** y sin
modificar:

| | |
|---|---|
| Nombre habitual | `Nekketsu Koukou Dodgeball Bu - Soccer Hen MD (Japan).md` |
| Tamaño | 524.288 bytes (512 KB) |
| CRC32 | `F49C3A86` |
| MD5 | `FF7A9A6FB74A640F40F10DFF53E9CF4D` |
| SHA-1 | `D865B01E58A269400DE369FC1FBB3B3E84E1ADD0` |

Si tu ROM no coincide con estas sumas, el parche no funcionará.

> **No se distribuye la ROM.** Búscala por tu cuenta.

---

## 3. ROM resultante

| | |
|---|---|
| Tamaño | 1.048.576 bytes (1 MB) |
| CRC32 | `8BA4F074` |
| MD5 | `30C69454512344419B622DB67D51E65B` |
| SHA-1 | `608739BF89A75915A5D90719D9E0BAC3D8AAA3F2` |

La ROM crece de 512 KB a 1 MB. Es normal: la traducción necesita espacio
para las fuentes nuevas, los gráficos redibujados y el código añadido.

---

## 4. Cómo aplicar el parche

Se incluyen dos formatos:

| Archivo | Notas |
|---|---|
| `Nekketsu_Soccer_MD_ES_v2_8.bps` | **Recomendado.** Verifica la ROM base |
| `Nekketsu_Soccer_MD_ES_v2_8.ips` | Alternativa |

- **BPS** → Flips (Floating IPS), beat o cualquier parcheador compatible
- **IPS** → Lunar IPS, Flips, ips.js y similares

**Pasos con Flips:**

1. Abre Flips y pulsa *Apply Patch*
2. Elige el archivo `.bps`
3. Elige tu ROM japonesa
4. Guarda la ROM parcheada con el nombre que quieras

> ⚠️ Aplica el parche sobre una **copia** de tu ROM original.

---

## 5. Compatibilidad

Probado en **Genesis Plus GX** (núcleo usado durante todo el desarrollo).

Debería funcionar en cualquier emulador razonablemente preciso y en
hardware real vía flashcart (Mega EverDrive y similares), ya que solo se
han usado técnicas estándar del 68000 y del VDP.

No se ha probado en hardware real: si alguien lo hace, agradeceríamos el
informe.

---

## 6. Qué incluye la traducción

### Texto

- Los **86 bloques de diálogo** de la historia, con la secuencia completa
  de 24 escenas restituida *(una versión anterior perdía 9 diálogos)*
- Menús, opciones y pantalla de clave
- Pantalla de alineación (`¿CAMBIAR POSICIÓN?`, `RES` / `VEL` / `TIR`)
- Nombres de los trece equipos en la selección de **DUELO 2J**
- Nombres de los equipos en la pantalla de resultados
- Rótulos de escena: `CLUB DE DODGEBALL`, `VEST. NEKKETSU`, etc.
- Títulos de partido: `PRIMER PARTIDO` … `PARTIDO FINAL`
- Créditos

### Gráficos

- Rótulo del título redibujado (`SOCCER MD EDICIÓN`)
- Placa `ELIGE ESCUELA` de la pantalla de selección
- Cinta verde del rótulo de la sala del club, reconstruida
- Fuente gruesa nueva de 20 letras para los nombres de equipo
- Corazones (♥) en los diálogos de Misako, como en el original
- Vocales acentuadas y signos `¿` `¡` injertados en la fuente del título

### Ajustes

- Ritmo de los diálogos acelerado a 1 fotograma por carácter
- Marcos de selección ajustados a cada nombre
- Corrección de varios fallos gráficos heredados de versiones previas

---

## 7. Limitaciones conocidas

- En la pantalla de alineación, la flecha del cursor SÍ/NO aparece un
  instante antes que la pregunta. Es cosmético y apenas perceptible.
- Las partes del juego que no se alcanzan fácilmente (torneo completo
  hasta el final, modo VS, pantalla de clave con códigos válidos) han
  recibido menos pruebas. Si encuentras algo raro, avisa.
- El juego no tiene sistema de guardado; funciona con claves, como el
  original.

---

## 8. Agradecimientos y avisos

*Nekketsu Koukou Dodgeball-bu - Soccer Hen MD* es propiedad de sus
respectivos titulares. Esta traducción es un trabajo de aficionados sin
ánimo de lucro y no está autorizada ni respaldada por Technos Japan ni por
sus sucesores.

Se distribuye **únicamente el parche**. No se incluye ni se enlaza la ROM.

Puedes redistribuir el parche libremente siempre que se mantenga este
archivo y no se cobre por él.

---

## 9. Historial

| Versión | Cambios |
|---|---|
| **v2.8** | Restaurada la presentación de partido en castellano a partir del segundo encuentro: una versión anterior había desactivado la llamada a la rutina que la dibuja y salía íntegra en japonés. |
| v2.7 | Corregido el destello de caracteres japoneses en la pantalla de resultados. Anulado el pintado original que reutilizaba la tabla de la pantalla de selección. |
| v2.6 | Corregido el volcado de fuente que escribía sobre el *nametable*. Eliminado el `NO` residual de la pantalla de alineación. |
| v2.5 | Anulado el texto viejo de `¿CAMBIAR POSICIÓN?` sin dañar las etiquetas `RES` / `VEL` / `TIR` que comparten tilemap. |
| v2.4 | *(retirada)* Intento fallido: desplazó las etiquetas de las fichas. |
| v2.3 | Nombres alineados a la izquierda. Limpieza del rótulo japonés. |
| v2.2 | Corregido un cuelgue en la escena del club. Rótulo del título restaurado. `SHIMANCHU` despegado del borde. |
| v2.1 | Cinta verde sin parpadeo. Marcos de selección adaptables. |
| v2.0 | Nombres de las trece escuelas traducidos con fuente gruesa. |
| v1.9 | Rótulo del título y placa `ELIGE ESCUELA`. |
| v1.8 | Fuente románica restaurada. Cinta verde reconstruida. |
| v1.7 | Corazones de Misako. Rótulo `CLUB DE DODGEBALL`. |
| v1.6 | Secuencia completa de 24 diálogos restituida. Ritmo acelerado. |
| v0.9.8 | Versión de partida. |
