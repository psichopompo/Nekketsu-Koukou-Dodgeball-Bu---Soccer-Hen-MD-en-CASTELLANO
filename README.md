================================================================================
  NEKKETSU KOUKOU DODGEBALL-BU  -  SOCCER HEN MD
  Traduccion al castellano  -  v2.8
================================================================================

  Mega Drive / Genesis  ·  Technos Japan, 1992  ·  Japon (exclusivo)


--------------------------------------------------------------------------------
  1. QUE ES ESTO
--------------------------------------------------------------------------------

Parche de traduccion al castellano de "Nekketsu Koukou Dodgeball-bu -
Soccer Hen MD", el futbol arcade de Kunio-kun que Technos publico en 1992
solo en Japon.

El juego nunca salio de Japon y nunca tuvo traduccion oficial ni de
aficionados a ningun idioma occidental. Esta es, hasta donde sabemos, la
primera version jugable en un idioma que no sea japones.

Se ha traducido TODO el texto visible: menus, dialogos de la historia,
nombres de las trece escuelas, rotulos de escena, marcador, pantalla de
alineacion y creditos.


--------------------------------------------------------------------------------
  2. ROM NECESARIA
--------------------------------------------------------------------------------

El parche se aplica sobre la ROM japonesa original, SIN CABECERA y SIN
modificar:

  Nombre habitual : Nekketsu Koukou Dodgeball Bu - Soccer Hen MD (Japan).md
  Tamano          : 524.288 bytes (512 KB)
  CRC32           : f49c3a86
  MD5             : ff7a9a6fb74a640f40f10dff53e9cf4d
  SHA-1           : d865b01e58a269400de369fc1fbb3b3e84e1add0

Si tu ROM no coincide con estas sumas, el parche no funcionara. No se
distribuye la ROM: buscala por tu cuenta.


--------------------------------------------------------------------------------
  3. ROM RESULTANTE
--------------------------------------------------------------------------------

  Tamano : 1.048.576 bytes (1 MB)
  CRC32  : 8ba4f074
  MD5    : 30c69454512344419b622db67d51e65b
  SHA-1  : 608739bf89a75915a5d90719d9e0bac3d8aaa3f2

La ROM crece de 512 KB a 1 MB. Es normal: la traduccion necesita espacio
para las fuentes nuevas, los graficos redibujados y el codigo anadido.


--------------------------------------------------------------------------------
  4. COMO APLICAR EL PARCHE
--------------------------------------------------------------------------------

Se incluyen dos formatos. Usa el que prefiera tu herramienta:

  Nekketsu_Soccer_MD_ES_v2_8.bps   (recomendado, verifica la ROM base)
  Nekketsu_Soccer_MD_ES_v2_8.ips

  BPS  ->  Flips (Floating IPS), beat, o cualquier parcheador compatible
  IPS  ->  Lunar IPS, Flips, ips.js y similares

Pasos con Flips:
  1. Abre Flips y pulsa "Apply Patch"
  2. Elige el archivo .bps
  3. Elige tu ROM japonesa
  4. Guarda la ROM parcheada con el nombre que quieras

IMPORTANTE: aplica el parche sobre una COPIA de tu ROM original.


--------------------------------------------------------------------------------
  5. COMPATIBILIDAD
--------------------------------------------------------------------------------

Probado en:
  - Genesis Plus GX (nucleo usado durante todo el desarrollo)

Deberia funcionar en cualquier emulador razonablemente preciso y en
hardware real vía flashcart (Mega EverDrive y similares), ya que solo se
han usado tecnicas estandar del 68000 y del VDP. No se ha probado en
hardware real: si alguien lo hace, agradeceriamos el informe.


--------------------------------------------------------------------------------
  6. QUE INCLUYE LA TRADUCCION
--------------------------------------------------------------------------------

TEXTO
  · Los 86 bloques de dialogo de la historia, con la secuencia completa
    de 24 escenas restituida (una version anterior perdia 9 dialogos).
  · Menus, opciones y pantalla de clave.
  · Pantalla de alineacion ("¿CAMBIAR POSICION?", RES / VEL / TIR).
  · Nombres de los trece equipos en la seleccion de DUELO 2J.
  · Nombres de los equipos en la pantalla de resultados.
  · Rotulos de escena: "CLUB DE DODGEBALL", "VEST. NEKKETSU", etc.
  · Titulos de partido: "PRIMER PARTIDO" ... "PARTIDO FINAL".
  · Creditos.

GRAFICOS
  · Rotulo del titulo redibujado ("SOCCER MD EDICION").
  · Placa "ELIGE ESCUELA" de la pantalla de seleccion.
  · Cinta verde del rotulo de la sala del club, reconstruida.
  · Fuente gruesa nueva de 20 letras para los nombres de equipo.
  · Corazones (♥) en los dialogos de Misako, como en el original.
  · Vocales acentuadas y signos ¿ ¡ injertados en la fuente del titulo.

AJUSTES
  · Ritmo de los dialogos acelerado a 1 fotograma por caracter.
  · Marcos de seleccion ajustados a cada nombre.
  · Correccion de varios fallos graficos heredados de versiones previas.


--------------------------------------------------------------------------------
  7. LIMITACIONES CONOCIDAS
--------------------------------------------------------------------------------

  · En la pantalla de alineacion, la flecha del cursor SI/NO aparece un
    instante antes que la pregunta. Es cosmetico y apenas perceptible.

  · Las partes del juego que no se alcanzan facilmente (torneo completo
    hasta el final, modo VS, pantalla de clave con codigos validos) han
    recibido menos pruebas. Si encuentras algo raro, avisa.

  · El juego no tiene sistema de guardado; funciona con claves, como el
    original.


--------------------------------------------------------------------------------
  8. AGRADECIMIENTOS Y AVISOS
--------------------------------------------------------------------------------

Nekketsu Koukou Dodgeball-bu - Soccer Hen MD es propiedad de sus
respectivos titulares. Esta traduccion es un trabajo de aficionados sin
animo de lucro y no esta autorizada ni respaldada por Technos Japan ni
por sus sucesores.

Se distribuye UNICAMENTE el parche. No se incluye ni se enlaza la ROM.

Puedes redistribuir el parche libremente siempre que se mantenga este
archivo y no se cobre por el.


--------------------------------------------------------------------------------
  9. HISTORIAL
--------------------------------------------------------------------------------

  v2.8  Restaurada la presentacion de partido en castellano a partir del
        segundo encuentro: una version anterior habia desactivado la
        llamada a la rutina que la dibuja y salia integra en japones.
  v2.7  Corregido el destello de caracteres japoneses en la pantalla de
        resultados. Anulado el pintado original que reutilizaba la tabla
        de la pantalla de seleccion.
  v2.6  Corregido el volcado de fuente que escribia sobre el nametable.
        Eliminado el "NO" residual de la pantalla de alineacion.
  v2.5  Anulado el texto viejo de "¿CAMBIAR POSICION?" sin dañar las
        etiquetas RES / VEL / TIR que comparten tilemap.
  v2.4  (retirada) Intento fallido: desplazo las etiquetas de las fichas.
  v2.3  Nombres alineados a la izquierda. Limpieza del rotulo japones.
  v2.2  Corregido un cuelgue en la escena del club. Rotulo del titulo
        restaurado. SHIMANCHU despegado del borde.
  v2.1  Cinta verde sin parpadeo. Marcos de seleccion adaptables.
  v2.0  Nombres de las trece escuelas traducidos con fuente gruesa.
  v1.9  Rotulo del titulo y placa "ELIGE ESCUELA".
  v1.8  Fuente romanica restaurada. Cinta verde reconstruida.
  v1.7  Corazones de Misako. Rotulo "CLUB DE DODGEBALL".
  v1.6  Secuencia completa de 24 dialogos restituida. Ritmo acelerado.
  v0.9.8  Version de partida.

================================================================================
