# Kit 05 · Editor de Vídeo Vertical

Eres el editor de vídeo del usuario. Él graba hablando a cámara y te da el archivo
en bruto; tú devuelves un vídeo vertical de 1080x1920 con los silencios cortados,
subtítulos quemados que se encienden palabra a palabra, y los rótulos y efectos que
hagan falta. Habla SIEMPRE en español, cercano y sin jerga — el usuario puede no
saber programar, y desde luego no sabe qué es un códec. Cada respuesta termina con
la siguiente acción concreta.

Tú ejecutas los comandos. **El usuario no abre nunca una terminal** y no tiene por
qué saber que existen los scripts: para él, esto es una conversación en la que pide
un vídeo y le llega un vídeo.

## Lo que hace el kit, en una frase

Un vídeo horizontal de 4 minutos con silencios, muletillas y tomas repetidas entra;
un vertical de 2:40 sin pausas muertas, con subtítulos y con ritmo, sale. Todo se
hace **en este ordenador**: nada se sube a ningún servidor, no hay claves y funciona
sin internet una vez instalado.

## Primer arranque y reapertura

Si estás respondiendo, la conexión con el modelo YA funciona. Lo que no se puede dar
por hecho es que estén ffmpeg y el modelo de voz.

- Si NO existe `.claude/setup-completado.json`: es la primera vez en este ordenador.
  Da la bienvenida en 3 líneas (qué es el kit, qué va a conseguir) y sugiérele
  escribir `/setup` — el wizard instala lo que falta y lo comprueba editando un
  vídeo de prueba. No intentes editar antes: fallará a mitad y con un error feo.
- Si existe: saluda con el menú corto. "¿Qué hacemos hoy?
  1. Editar un vídeo — arrastra el archivo a `entrada/` y dime *edita este vídeo*
  2. Continuar un vídeo a medias" (lista lo que haya en `workspace/`)
  "3. Retocar uno ya terminado (más cortes, otro estilo de subtítulos, un rótulo)
  4. Ver qué puede hacer el kit"
- El campo `python` de ese archivo dice qué comando de Python funciona en este
  ordenador (`python3`, `python` o `py`). Úsalo siempre; no vuelvas a averiguarlo.

## La cadena de montaje

Nueve pasos, en este orden. Cada uno deja su resultado en disco, así que si algo se
corta se retoma por donde iba sin repetir nada.

| # | Comando | Qué hace | Deja |
|---|---|---|---|
| 1 | `python scripts/nuevo.py "<vídeo>"` | Crea el proyecto y avisa de lo que pueda dar guerra (sin audio, ya vertical, rotado) | `workspace/<nombre>/proyecto.json` |
| 2 | `python scripts/transcribir.py <proyecto>` | Transcribe con marca de tiempo en **cada palabra** | `trabajo/transcripcion.json` |
| 3 | `python scripts/plan_cortes.py <proyecto>` | Decide qué se quita: silencios, muletillas, tomas repetidas | `trabajo/plan-cortes.json` |
| 4 | `python scripts/encuadre.py <proyecto>` | Saca 3 fotogramas con una regla para elegir la franja vertical | 3 PNG que **hay que mirar** |
| 5 | `python scripts/cortar.py <proyecto>` | Aplica los cortes y pasa a vertical, en una sola recompresión | `trabajo/master.mp4` |
| 6 | `python scripts/subtitulos.py <proyecto>` | Dibuja los subtítulos karaoke como imágenes | `trabajo/subs/*.png` |
| 7 | `python scripts/capturar.py <proyecto>` | Rótulos y mockups animados (opcional) | `trabajo/visuales/`, `trabajo/timeline.json` |
| 8 | `python scripts/componer.py <proyecto>` | Monta zooms, capas y subtítulos en una pasada | `trabajo/montaje.mp4` |
| 9 | `python scripts/sonido.py <proyecto>` | Iguala el volumen a -14 LUFS y mezcla efectos | **`<proyecto>/<nombre>-vertical.mp4`** |

El detalle de cada paso, con las opciones y los criterios, está en la skill
`editor-vertical`. Lo que no se puede saltar:

- **El paso 4 es el único donde decide el usuario.** Un vídeo horizontal no cabe en
  vertical: hay que quitar la mitad de la imagen. Enséñale los tres fotogramas y
  pregúntale dónde está su cara. Adivinarlo sale mal y se nota en el vídeo entero.
- **El paso 3 se le cuenta antes de cortar**: cuántos silencios, cuántas muletillas
  y cuánto va a durar el resultado. Es su vídeo; que sepa qué se va a quitar.

## Tabla de decisión

| Lo que dice el usuario | Lo que haces |
|---|---|
| "hola", "empieza", "qué hago" | Bienvenida + `/setup`, o menú de reapertura (ver arriba) |
| "edita este vídeo", "hazme esto vertical", "ponle subtítulos" | Skill `editor-vertical` completa, narrando cada fase en una línea |
| Suelta un vídeo sin decir nada | Míralo con `nuevo.py`, resume qué has recibido (duración, tamaño, si tiene audio) y pregunta si lo edita entero o solo un trozo |
| "continúa", "sigue con el de ayer" | Lee `workspace/<nombre>/proyecto.json`, di por qué paso iba y sigue desde ahí. Nada de empezar de cero |
| "corta más", "quedan pausas", "va muy lento" | Solo el paso 3 con `--umbral 0.40` y luego 5→9. No hace falta volver a transcribir |
| "ha cortado de más", "se come palabras" | `--umbral 0.75`, y si sigue, `--sin-micropausas`. Explica el compromiso: menos agresivo = más pausas naturales |
| "no me quites las muletillas" / "deja las repeticiones" | `plan_cortes.py --sin-muletillas` / `--sin-tomas` |
| "los subtítulos no me gustan" | Estilo de subtítulos en la skill (`referencias/estilo.md`): tamaño, colores, palabras por grupo, posición |
| "ponme un rótulo aquí", "mete un chat de Claude", "un titular al principio" | Visuales: escribe `trabajo/visuales.json` y corre el paso 7. Plantillas disponibles en `referencias/visuales.md` |
| "métele zooms", "dale más ritmo" | Zooms a mano en `trabajo/timeline.json` (`referencias/visuales.md`) y vuelve a componer |
| "cambia los efectos de sonido" | `trabajo/sfx.json`, que el paso 9 ya deja escrito con una propuesta sobria |
| "sale muy bajito" / "suena mal" | El paso 9 ya iguala a -14 LUFS. Si el original venía muy bajo o saturado, dilo: se puede mejorar, no arreglar |
| "cuánto va a tardar" | Alrededor de **1 minuto de proceso por minuto de vídeo** en un portátil normal; la transcripción es la mitad de ese tiempo |
| "algo no funciona", "me da un error" | Protocolo de diagnóstico (abajo) |
| "¿cómo funciona por dentro?" | Resume el README en cristiano, sin un solo término técnico sin traducir |
| "hazme el vídeo entero desde un guion", "genera tú la voz" | Este kit **edita** lo que se ha grabado; no genera vídeo ni voz. Dilo claro y ofrécele lo que sí hace |
| "quítale la cara a alguien", "que no se me reconozca la voz" | No es lo que hace el kit. Si quiere tapar una zona concreta de la imagen, eso sí se puede: una capa encima |

## Si algo falla (protocolo de diagnóstico)

1. NO repitas el comando que falló. Lee el error completo: estos scripts explican
   qué pasó y qué hacer.
2. Consulta la tabla:

| Error | Causa y solución |
|---|---|
| `No encuentro 'ffmpeg'` | No está instalado en este ordenador. `python scripts/instalar_ffmpeg.py` lo baja dentro del kit sin pedir contraseña |
| `Tu ffmpeg viene recortado` / falta `libx264` | Hay un ffmpeg del sistema sin codificadores. `python scripts/instalar_ffmpeg.py --forzar` pone la copia buena en `bin/`, que tiene prioridad |
| `No module named 'faster_whisper'` (o `PIL`, o `playwright`) | Falta un paquete de Python. `python scripts/doctor.py --instalar` |
| La transcripción se queda parada mucho rato la primera vez | Está descargando el modelo de voz, unos 500 MB. Es una sola vez. Avísale y espera |
| La transcripción devuelve pocas palabras o texto raro | Modelo pequeño o audio malo. Prueba `transcribir.py --modelo medium --refrescar`, y comprueba que el idioma es el correcto con `--idioma es` |
| `El vídeo no tiene pista de audio` | Sin audio no hay transcripción ni cortes por silencio. Se puede pasar a vertical y ponerle rótulos, nada más. Pregunta si grabó el sonido aparte |
| El vídeo sale con la cara cortada | El encuadre está mal elegido. `encuadre.py --centro <valor>` y `cortar.py --refrescar`. Si la acción se mueve por todo el plano, `--modo marco` |
| El vídeo final sale del revés o tumbado | Vídeo de móvil con la rotación en los metadatos. El kit ya lo tiene en cuenta; si aun así pasa, dime la marca del móvil y el resultado de `nuevo.py` |
| Los subtítulos se salen o tapan la cara | Estilo: menos palabras por grupo o subirlos. Está en `referencias/estilo.md` |
| Los rótulos no se generan (`playwright`, `chromium`, `Executable doesn't exist`) | Falta el Chrome sin ventana: `python -m playwright install chromium`. Si no se puede, el kit edita igual **sin** rótulos: dilo y sigue |
| `No space left on device` / el disco se llena | Mientras monta, un vídeo de un minuto puede ocupar 2 GB. Borra las carpetas `trabajo/` de proyectos ya terminados |
| El proceso se corta a mitad (se cierra el portátil, se va la luz) | No se pierde nada: cada paso deja su resultado. Lee `proyecto.json` y retoma por el primero que falte |
| Todo verde en el doctor pero falla al editar | El doctor comprueba que los programas están, no que funcionen. Lanza `python scripts/prueba.py`: si pasa, el problema es ese vídeo concreto |

3. Si el error no está en la tabla: investiga, arréglalo y **añade la fila** para el
   siguiente.
4. Si tras 2 intentos sigue atascado: sugiérele preguntar en la comunidad donde
   consiguió el kit, pegando el error literal.

## Reglas

- **Nunca inventes lo que dice el vídeo.** Los subtítulos salen de la transcripción,
  palabra por palabra. Si algo no se entiende, se marca y se pregunta; no se
  completa "con lo que tendría sentido". Un subtítulo inventado es poner en boca del
  usuario algo que no dijo.
- **No toques el vídeo original.** Se copia o se referencia, jamás se sobrescribe.
  Todo lo que se genera va a `workspace/<nombre>/`.
- **Una sola recompresión por paso.** No encadenes conversiones "por si acaso": cada
  recompresión pierde calidad. Los scripts ya lo hacen todo en una pasada.
- **El encuadre lo decide el usuario** (paso 4). Es el único paso que no se puede
  automatizar sin arriesgar el vídeo entero.
- Antes de cortar, **cuéntale qué vas a quitar** y cuánto va a durar el resultado.
- Los vídeos terminados van SIEMPRE a `workspace/`. Nunca sueltos por la raíz ni
  encima del original.
- Nunca pidas al usuario que abra una terminal: los comandos los ejecutas tú.
- Secretos (claves de API) nunca por el chat. Este kit no necesita ninguna clave: la
  transcripción es local. Si alguien quiere usar AssemblyAI, la clave va a
  `.env.local`, que está en el `.gitignore`.
- **Nada sale de este ordenador.** No subas el vídeo a ningún servicio ni sugieras
  hacerlo. Si el usuario pide el motor AssemblyAI, avísale de que eso sí sube el
  audio a un servidor externo antes de usarlo.
- Si el vídeo trae a terceros (una llamada, un cliente, un menor), no lo comentes en
  el chat más de lo necesario y no propongas publicarlo: eso es decisión suya.
- ≤2 intentos por error. Después, comunidad, con el error literal.
# Graba esto con el móvil (20 segundos)

Un guion pensado para que veas funcionando **todo** lo que hace el kit en un solo
clip. No es para publicarlo: es para que veas el antes y el después con tu propia voz
y tu propia cara.

Si no quieres grabar nada, salta al final: el kit fabrica su propio vídeo de prueba.

## Antes de darle a grabar

| | |
|---|---|
| **Cómo** | El móvil **en horizontal** (tumbado). Sí, en horizontal: así ves cómo lo pasa a vertical, que es la mitad del truco |
| **Dónde** | Cualquier sitio con algo de luz en la cara. No hace falta nada más |
| **Sonido** | Los auriculares del móvil ya son mejor que nada. Si no tienes, habla cerca |
| **Sitúate** | **No en el centro: ponte a un lado del encuadre.** Ahí se ve para qué sirve el paso del recorte |
| **Duración** | Unos 20 segundos con las pausas. No pasa nada si son 30 |

## El guion

Lee esto tal cual, **con las pausas de verdad**. Las pausas son la mitad del guion: son
lo que el kit va a quitar.

> **Hola.**
>
> *(cuenta dos segundos en silencio, mirando a cámara)*
>
> **Eeeh… esto es una prueba del kit de edición.**
>
> *(dos segundos de silencio otra vez)*
>
> **Lo que hace es… lo que hace es quitarme los silencios.**
>
> *(un segundo)*
>
> **Y me pone los subtítulos solo.**
>
> *(dos segundos de silencio, sin cortar la grabación)*
>
> **Si ves esto en vertical y con letras, ya funciona.**

Y ya. Para de grabar.

## Qué está probando cada trozo

| El trozo | Lo que se comprueba |
|---|---|
| Grabar en horizontal | El paso del recorte: de 1920x1080 a 1080x1920 |
| Ponerte a un lado | Que el kit te pregunta **dónde estás** antes de recortar, en vez de adivinar |
| Los silencios de dos segundos | El corte de silencios. Son ~7 s de los 20: se van |
| El "Eeeh…" | La detección de muletillas |
| **"Lo que hace es… lo que hace es"** | La detección de tomas repetidas: se queda con la segunda. Esto es lo que más sorprende |
| Hablar seguido | Los subtítulos karaoke, palabra por palabra |

## Después de grabar

1. Pasa el vídeo al ordenador y déjalo en la carpeta **`entrada/`** del kit.
2. En Claude Code, escribe:

```
edita este vídeo
```

3. Te va a preguntar **una sola cosa**: en qué parte del encuadre estás. Te enseñará
   tres fotogramas con una regla encima para que lo veas.

De unos 20 segundos vas a salir con unos 12, en vertical y con subtítulos.

## Lo que vas a ver en el resultado

- **Vertical de 1080x1920**, el formato de TikTok, Reels y Shorts.
- **Sin las pausas** — pero fíjate en que las pausas cortas siguen ahí: se acortan, no
  se quitan. Si se quitaran todas, sonaría atropellado.
- **Sin el "eeeh"**.
- **La frase repetida, una sola vez.** Busca el momento: dijiste dos veces "lo que hace
  es" y en el vídeo final solo hay una.
- **Subtítulos que se encienden** palabra a palabra mientras hablas.

## Si no quieres grabar nada

El asistente de instalación (`/setup`) fabrica su propio vídeo de prueba, con la voz
del ordenador, y lo edita de punta a punta. Termina abriéndotelo para que lo veas.
Tarda unos 15 segundos y comprueba exactamente lo mismo — salvo que la voz no es tuya
y la cara no sale.

Es más rápido, pero con tu clip se entiende mejor lo que estás mirando.

## Un aviso, ya que estás

Cuando grabes de verdad, **no edites nada a mano antes de pasárselo al kit**. Déjalo
todo: las pausas para pensar, los "eeeh", y la frase que has repetido cuatro veces
hasta que salió. Cortar tú antes solo le quita información al kit y te quita tiempo a
ti.
# Kit 05 · Empieza aquí — 15 minutos y a editar

Este kit convierte Claude Code en tu editor de vídeo. Le das el vídeo tal como
salió de la cámara —horizontal, con silencios, con tus "eeeh" y con la frase que
repetiste tres veces— y te devuelve un **vertical de 1080x1920 con subtítulos
quemados**, las pausas muertas fuera y ritmo. Todo en tu ordenador: nada se sube a
ningún sitio y no hay ninguna clave que pagar. 4 pasos.

## Paso 1 · Lo que necesitas instalado

1. **VS Code** — gratis, en code.visualstudio.com (instalar dándole a "siguiente").
2. La extensión **Claude Code** — en VS Code: Extensiones → busca "Claude Code" →
   Instalar. Necesitarás tu cuenta de Claude: Claude Code funciona con suscripción
   (Pro o superior) o con cuenta API de Anthropic.
3. **Python** — en python.org/downloads, el botón grande. Es el idioma en el que
   está escrito el motor del kit.
   - **En Windows, en la primera pantalla del instalador marca la casilla
     "Add python.exe to PATH"**. Se olvida todo el mundo y sin ella el kit no lo
     encuentra. Si te has pasado, vuelve a lanzar el instalador y marca la casilla.
   - En Mac, el instalador normal, dos clics.
4. Solo en Windows: **Git para Windows** (git-scm.com/download/win, todo
   "siguiente") — Claude Code lo necesita para funcionar.

No hace falta instalar ffmpeg, ni Homebrew, ni ningún programa de vídeo: de eso se
encarga el paso 4 y no te va a pedir la contraseña del ordenador.

## Paso 2 · Abre la carpeta

- Antes de nada: si aún ves el kit dentro del ZIP, descomprímelo (en Windows: clic
  derecho sobre el ZIP → "Extraer todo") y trabaja siempre con la carpeta extraída
  — nunca desde dentro del ZIP.
- En VS Code: `Archivo → Abrir carpeta...` y elige esta carpeta del kit.
- Abre Claude Code desde el panel lateral. La primera vez te preguntará en inglés
  *"Do you trust the files in this folder?"* — elige **Yes, proceed** (es un aviso
  estándar de seguridad). Mientras trabaja también te pedirá algún permiso en
  inglés (botones "Allow" o "Yes"): es normal, acéptalos.

## Paso 3 · Deja el vídeo a mano

Arrastra tu vídeo a la carpeta **`entrada/`** del kit. Vale cualquier cosa que
salga de un móvil o de una cámara: `.mp4`, `.mov`, `.mkv`, `.webm`.

¿No tienes nada grabado? En **`ejemplos/GUION-DEMO.md`** tienes un guion de 20
segundos para grabar con el móvil ahora mismo, pensado para que se vea funcionando
todo. Y si quieres verlo antes de grabar nada, el paso 4 fabrica su propio vídeo de
prueba.

## Paso 4 · Escribe /setup

El asistente instala lo que falta (unos 700 MB de descargas, una sola vez y sin
contraseña de administrador), y después **se comprueba a sí mismo editando un vídeo
de prueba que fabrica él**: te enseña el resultado en vertical, con subtítulos y con
un rótulo. Si eso sale bien, el kit funciona en tu ordenador. Sin más.

Luego, editar es una frase:

```
edita este vídeo
```

## Lo único que tienes que saber antes de grabar

**No edites nada a mano.** Habla seguido y déjalo todo: las pausas para pensar, los
"eeeh", y la frase que has repetido cuatro veces hasta que salió bien. El kit quita
los silencios, quita las muletillas y, cuando repites una toma, **se queda con la
última**. Cortarlo tú antes solo le quita información y te quita tiempo.

Dos cosas más que ayudan mucho:

- **Habla mirando a un lado fijo del encuadre.** Si grabas en horizontal, la mitad
  de la imagen se va a ir (en vertical no cabe): el kit te preguntará qué franja se
  queda, y es más fácil si no te mueves de lado a lado.
- **Graba el sonido lo mejor que puedas.** Los subtítulos salen de entender lo que
  dices: con ruido de fondo o con la mano tapando el micro, salen peor. Los
  auriculares del móvil ya son mejores que nada.

## Cuánto tarda

| | |
|---|---|
| Instalación (una vez) | 5-15 minutos, casi todo descargas |
| Editar un vídeo | Alrededor de **1 minuto por cada minuto de vídeo** |
| La primera transcripción | Suma unos minutos: baja el modelo de voz (500 MB, una vez) |

Va todo en tu ordenador, así que en un portátil antiguo tarda más. No consume nada
de tu plan de Claude: el vídeo lo procesa tu máquina, no el modelo.

## Si algo falla al arrancar

| Lo que ves | Qué significa | Qué hacer |
|---|---|---|
| "Python was not found" o se abre la tienda de Microsoft | Python no está instalado (eso es un atajo vacío de Windows) | Instálalo de python.org marcando "Add python.exe to PATH", cierra VS Code y vuelve a abrirlo |
| Claude Code no arranca (Windows) | Falta Git para Windows | Instálalo (git-scm.com/download/win) y reinicia VS Code |
| La extensión pide iniciar sesión | Normal la primera vez | Inicia sesión con tu cuenta de Claude (suscripción o API) |
| Se queda parado varios minutos al transcribir por primera vez | Está bajando el modelo de voz (500 MB) | Espera. Es una sola vez |
| "No encuentro ffmpeg" | El programa de vídeo no se descargó | Dile a Claude: *"instala ffmpeg"*. Lo baja dentro del kit, sin contraseña |
| Los rótulos no salen pero el vídeo sí | Falta el Chrome interno de los rótulos | Es opcional: el vídeo se hace igual. Si los quieres, dile *"instala los visuales"* |
| El vídeo sale con tu cara cortada | Se eligió mal la franja vertical | Dile *"la cara está más a la izquierda"* y lo vuelve a montar |
| "Has alcanzado tu límite de uso" | Límite temporal de tu plan de Claude | Espera a que se restablezca. Lo ya hecho no se pierde: al volver di *"continúa el vídeo"* |
| Nada de esto funciona | — | Pregunta en la comunidad donde conseguiste el kit, pegando el error tal cual |

¿Más detalle? `README.md` cuenta qué hace cada paso, qué se puede cambiar y qué no
hace el kit.
# Deja aquí tus vídeos

Arrastra a esta carpeta el vídeo que quieras editar, tal como salió del móvil o de la
cámara. Luego, en Claude Code:

```
edita este vídeo
```

Nada de lo que dejes aquí se modifica: **el original no se toca nunca.** El resultado
y todos los pasos intermedios van a `workspace/<nombre-del-vídeo>/`.

## Qué formatos valen

`.mp4`, `.mov`, `.mkv`, `.webm`, `.avi` — cualquier cosa que salga de un móvil o de una
cámara. Horizontal o vertical, da igual: si es horizontal, el kit te preguntará qué
franja se queda.

## Qué tamaño es razonable

Está pensado para clips de **hasta 3 o 4 minutos**. Con una grabación de una hora
funcionaría, pero tardaría una hora y ocuparía muchos gigas de disco: para eso, corta
antes el trozo que te interesa y trae solo ese.

## Antes de grabar, dos cosas

1. **No edites nada a mano.** Deja las pausas, los "eeeh" y la frase que repetiste
   cuatro veces. El kit quita los silencios, quita las muletillas y, cuando repites una
   toma, se queda con la última. Cortarlo tú antes le quita información.
2. **Sitúate a un lado fijo del encuadre** si grabas en horizontal. La mitad de la
   imagen se va a ir (en vertical no cabe), y es más fácil elegir bien si no te mueves
   de lado a lado.

¿No tienes nada grabado? En `ejemplos/GUION-DEMO.md` hay un guion de 20 segundos para
grabar ahora mismo con el móvil, pensado para ver funcionando todo el kit.

## También aquí

Esta carpeta es buen sitio para las capturas de pantalla o las fotos que quieras que
salgan en el vídeo a pantalla completa. Dile *"mete esta captura cuando digo tal cosa"*
y se coloca.

## Privacidad

Todo se procesa en tu ordenador. Ni el vídeo ni el audio se suben a ningún servidor,
ni siquiera para hacer los subtítulos. Los vídeos que dejes aquí no viajan si compartes
el kit: esta carpeta está excluida.
/* Motor de animación de las plantillas.
 *
 * La regla de oro: aquí no hay animaciones de CSS ni relojes. Cada plantilla
 * expone una función que, dado un instante en segundos, coloca la pantalla
 * exactamente en ese estado. Así capturar.py puede pedir el fotograma 37 y
 * salir siempre igual, en cualquier ordenador y sin depender de la velocidad
 * del navegador. Con animaciones de CSS o requestAnimationFrame las capturas
 * salen descuadradas y el resultado cambia de un día para otro.
 *
 * Prohibido dentro de una plantilla: transition, animation, Date.now(),
 * Math.random(), setTimeout. Todo se calcula a partir de t.
 */

const Motor = (() => {
  const tope = (v, a, b) => Math.min(Math.max(v, a), b);

  /* Suavizados. Reciben 0..1 y devuelven 0..1 (el resorte se pasa un poco). */
  const suavizados = {
    lineal: u => u,
    suave: u => u * u * (3 - 2 * u),
    entra: u => u * u,
    frena: u => 1 - Math.pow(1 - u, 3),
    // Se pasa de largo y vuelve: da sensación de peso.
    rebote: u => {
      const s = 1.9;
      const v = u - 1;
      return 1 + (s + 1) * v * v * v + s * v * v;
    },
    // Oscila y se asienta, como un resorte.
    resorte: u => (u >= 1 ? 1 : 1 - Math.exp(-6.5 * u) * Math.cos(8.2 * u)),
  };

  /* Trozo de tiempo normalizado: 0 antes de empezar, 1 al acabar. */
  function tramo(t, inicio, duracion, suavizado) {
    const u = tope((t - inicio) / Math.max(duracion, 0.0001), 0, 1);
    return (suavizados[suavizado] || suavizados.suave)(u);
  }

  /* Entra, se queda y sale. Devuelve 0..1..0 */
  function presencia(t, inicio, entrada, visible, salida) {
    if (t < inicio) return 0;
    const fin = inicio + entrada + visible;
    if (t <= inicio + entrada) return tramo(t, inicio, entrada, "frena");
    if (t <= fin) return 1;
    return 1 - tramo(t, fin, Math.max(salida, 0.0001), "suave");
  }

  /* Retraso en cascada para listas: el elemento i entra un poco después. */
  const cascada = (i, paso) => i * paso;

  const numero = (u, desde, hasta) => desde + (hasta - desde) * u;

  /* Cuántas letras se han escrito ya. Se redondea hacia abajo para que la
     letra aparezca de golpe: a medias parpadea. */
  function escrito(texto, u) {
    return texto.slice(0, Math.floor(u * texto.length + 0.0001));
  }

  /* Coloca un elemento. Solo toca transform y opacity: son las dos cosas
     que el navegador dibuja sin recalcular la página, así la captura es
     rápida incluso con muchos elementos. */
  function poner(el, p) {
    if (!el) return;
    const partes = [];
    if (p.x || p.y) {
      partes.push(`translate3d(${p.x || 0}px, ${p.y || 0}px, 0)`);
    }
    if (p.escala !== undefined && p.escala !== 1) {
      partes.push(`scale(${p.escala})`);
    }
    if (p.giro) partes.push(`rotate(${p.giro}deg)`);
    el.style.transform = partes.length ? partes.join(" ") : "none";
    el.style.opacity = p.opacidad === undefined ? 1 : p.opacidad;
    if (p.difuminado !== undefined) {
      el.style.filter = p.difuminado > 0.05 ? `blur(${p.difuminado}px)` : "none";
    }
  }

  /* Entrada estándar del kit: sube, aparece y se asienta con un rebote. */
  function entrada(el, t, inicio, duracion, opciones) {
    const o = opciones || {};
    const u = tramo(t, inicio, duracion || 0.42, o.suavizado || "resorte");
    const uo = tramo(t, inicio, (duracion || 0.42) * 0.5, "frena");
    poner(el, {
      y: numero(u, o.desdeY === undefined ? 46 : o.desdeY, 0),
      x: numero(u, o.desdeX || 0, 0),
      escala: numero(u, o.desdeEscala === undefined ? 0.9 : o.desdeEscala, 1),
      opacidad: uo * (o.opacidadFinal === undefined ? 1 : o.opacidadFinal),
      difuminado: numero(1 - uo, 0, o.difuminado || 0),
    });
  }

  /* Lo que cada plantilla llama al final para darse de alta. */
  function registrar(definicion) {
    window.PLANTILLA = definicion;
    window.LIENZO = definicion.lienzo || { ancho: 1080, alto: 1920 };
    // capturar.py llama a estas tres desde fuera.
    window.cargar = datos => definicion.cargar(datos || {});
    window.duracion = () => definicion.duracion();
    window.fotograma = t => definicion.fotograma(t);
  }

  /* Atajos para escribir las plantillas sin repetir document.querySelector. */
  const uno = sel => document.querySelector(sel);
  const todos = sel => Array.from(document.querySelectorAll(sel));
  const crear = (etiqueta, clase, html) => {
    const el = document.createElement(etiqueta);
    if (clase) el.className = clase;
    if (html !== undefined) el.innerHTML = html;
    return el;
  };
  /* Texto del usuario: se escapa siempre, nunca se inyecta tal cual. */
  const texto = s => String(s === undefined || s === null ? "" : s)
    .replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");

  /* Marca en negrita naranja lo que venga entre *asteriscos*. */
  const marcado = s => texto(s).replace(/\*([^*]+)\*/g,
    '<span class="resalte">$1</span>');

  return { tope, suavizados, tramo, presencia, cascada, numero, escrito,
           poner, entrada, registrar, uno, todos, crear, texto, marcado };
})();
# Kit 05 · Editor de Vídeo Vertical — para Claude Code

Le das el vídeo tal como salió de la cámara. Te devuelve el vídeo publicable.

| | |
|---|---|
| **Entra** | Un vídeo en bruto: horizontal, con silencios, con muletillas, sin subtítulos |
| **Sale** | Vertical 1080x1920 a 60 fps, subtítulos quemados palabra a palabra, silencios fuera, volumen igualado y efectos |
| **Dónde** | En tu ordenador. Nada se sube a ningún servidor |
| **Coste** | Cero. Sin claves, sin suscripciones de vídeo, sin marcas de agua |
| **Tiempo** | Alrededor de 1 minuto de proceso por minuto de vídeo |

Funciona igual en Mac y en Windows. No consume tu plan de Claude editando: el vídeo
lo trabaja tu máquina; el modelo solo dirige.

## Empezar

Lee `EMPIEZA-AQUI.md` (4 pasos, 15 minutos) y escribe `/setup`. El wizard instala lo
que falta y **se comprueba a sí mismo editando un vídeo de prueba**, así que sabes
que funciona antes de gastar tu primer vídeo de verdad.

Después, todo se pide hablando:

```
edita este vídeo
corta más, quedan pausas
la cara está más a la izquierda
ponme un rótulo que diga "Paso 1" en el segundo 12
```

## Los nueve pasos

Cada paso deja su resultado en disco. Si algo se corta —se cierra el portátil, se va
la luz, se acaba tu límite de Claude— se retoma por donde iba sin repetir nada.

**1 · Se abre el proyecto.** Se lee el vídeo y se avisa de lo que puede dar guerra:
que no tenga audio, que ya sea vertical, que venga rotado desde el móvil.

**2 · Se transcribe con marca de tiempo en cada palabra.** No basta saber qué se
dice: hay que saber en qué milisegundo empieza cada palabra, porque de ahí salen los
subtítulos que se encienden solos y la detección de silencios. Lo hace un modelo de
voz que corre en tu ordenador (faster-whisper), gratis y sin internet.

**3 · Se decide qué se quita.** Cuatro señales, combinadas:

- **Silencios de verdad**, medidos en el audio (no solo huecos entre palabras).
- **Pausas largas** entre palabras, que se acortan a un tiempo natural en vez de
  eliminarse: quitarlas del todo suena atropellado.
- **Muletillas** ("eeeh", "mmm", "o sea" repetido) cuando están sueltas.
- **Tomas repetidas**: si dices "el primero es… el primero es que…", se queda con la
  segunda. Esto es lo que más tiempo ahorra y es lo que nadie hace a mano.

Antes de cortar, se te cuenta qué se va a quitar y cuánto va a durar el resultado.

**4 · Eliges el encuadre.** Aquí decides tú, y es el único paso donde hace falta.
Un vídeo horizontal no cabe en vertical: hay que tirar la mitad de la imagen. El kit
te saca tres fotogramas con una regla encima y le dices dónde estás: *"un poco a la
izquierda"*. También puede meter el vídeo entero dentro de un marco, con su propio
fondo desenfocado detrás, si la acción se mueve por todo el plano.

**5 · Se corta y se pasa a vertical**, todo en una sola recompresión. Una sola
importa: cada vez que se recomprime un vídeo se pierde calidad, y las cadenas de
herramientas mal hechas recomprimen cinco veces.

**6 · Se dibujan los subtítulos.** Estilo karaoke: la palabra que se está diciendo se
enciende. Se dibujan como imágenes con transparencia, no con el filtro de texto de
ffmpeg ni con archivos `.ass`. Es una decisión importante: así el resultado es
idéntico en Mac y en Windows y no depende de cómo esté compilado el ffmpeg de cada
uno, que es de donde vienen la mitad de los problemas con los subtítulos quemados.

**7 · Rótulos y mockups** (opcional). Seis plantillas animadas —rótulo, titular,
móvil con un chat, terminal, recapitulación, llamada a la acción— que se generan
fotograma a fotograma en un Chrome sin ventana. No se graba la pantalla: se le pide
el fotograma 12 y se guarda el fotograma 12, así que el resultado es idéntico
siempre, aunque el ordenador vaya lento. Si esta parte no se puede instalar, el kit
edita igual y solo te quedas sin rótulos.

**8 · Se monta todo en una pasada**: los zooms sobre las frases importantes, las
capas de rótulos, las imágenes que quieras meter a pantalla completa y, encima de
todo, los subtítulos.

**9 · Se cierra el sonido.** El volumen se iguala a **-14 LUFS**, que es el nivel al
que suenan TikTok, Instagram y YouTube. Si no se hace, tu vídeo se oye más bajo que
el resto del feed y la gente pasa de largo. Y se mezclan los efectos: un barrido
corto en los saltos, un golpe en los rótulos.

## Qué hay en el kit

| Carpeta | Qué es |
|---|---|
| `entrada/` | Donde sueltas los vídeos en bruto |
| `workspace/` | Un proyecto por vídeo, con el resultado y todos los intermedios |
| `scripts/` | El motor: 13 programas de Python — uno por paso, más los de instalación y prueba |
| `plantillas/` | Las seis plantillas de rótulos y mockups, en HTML |
| `assets/fuentes/` | Tres tipografías variables con licencia libre (OFL) |
| `assets/sfx/` | 19 efectos de sonido con licencia de uso libre (Pixabay) |
| `ejemplos/` | El guion para grabar tu clip de prueba de 20 segundos |
| `bin/` | Aparece al instalar: aquí vive el ffmpeg del kit |

## Lo que se puede cambiar (sin tocar nada, hablando)

| Quieres | Dices algo como |
|---|---|
| Más ritmo | "corta más" — se baja el umbral de silencio |
| Menos agresivo | "ha cortado de más, se come palabras" |
| Dejar las muletillas | "no me quites los 'eeeh'" |
| Otro encuadre | "la cara está más a la derecha" / "que se vea el vídeo entero" |
| Otros subtítulos | "más grandes", "más abajo", "de tres en tres palabras", "en amarillo" |
| Un rótulo | "un rótulo que diga 'Paso 1' cuando empiezo a explicar" |
| Zooms | "métele zoom donde digo lo importante" |
| Sin efectos | "quítale los sonidos" |

## Lo que este kit NO hace

Se dice antes para que no haya sorpresas:

- **No genera vídeo ni voz.** Edita lo que tú has grabado. No hay avatares, no hay
  voz sintética, no hay imágenes creadas.
- **No escribe tu guion.** Eso es otro kit.
- **No arregla un audio malo.** Iguala el volumen y quita silencios, pero si grabaste
  con viento y el micro tapado, se notará. Mejora, no resucita.
- **No pone música.** Efectos cortos sí; una canción de fondo no, porque lo que
  puedes publicar depende de la licencia de esa canción y eso es decisión tuya.
- **No sube nada a ninguna red.** El vídeo terminado se queda en tu carpeta; lo
  publicas tú.
- **No hace vídeos largos cómodamente.** Está pensado para clips de hasta 3-4
  minutos. Con una hora de grabación funcionaría, pero tardaría una hora y ocuparía
  mucho disco: para eso, primero corta el trozo que te interesa.

## Privacidad

Todo pasa en tu ordenador. La transcripción usa un modelo local: **tu audio no sale
de la máquina**, ni siquiera para transcribir, y una vez instalado funciona en avión.

Hay un motor de transcripción opcional (AssemblyAI) que es algo más preciso con
audio malo. Ese **sí sube el audio a un servidor externo** y necesita una clave de
pago. Está documentado, no está activado, y si lo pides se te avisa antes. Su clave,
si la usas, va a `.env.local`, que nunca viaja con el kit.

## Si usas Windows

Funciona igual, con dos avisos:

- Al instalar Python, **marca "Add python.exe to PATH"** en la primera pantalla del
  instalador. Es el fallo más común de todos.
- ffmpeg se instala dentro del kit, no en el sistema: no hace falta winget, ni tocar
  variables de entorno, ni permisos de administrador.

## Cuánto ocupa

| | |
|---|---|
| El kit descargado | Unos 3 MB |
| ffmpeg (se descarga al instalar) | Unos 90 MB, dentro de `bin/` |
| Modelo de voz | Unos 500 MB, una vez, en la carpeta de tu usuario |
| Chrome de los rótulos | Unos 150 MB, una vez (opcional) |
| Mientras monta un vídeo de 1 minuto | Hasta 2 GB temporales en `workspace/` |

Para desinstalarlo del todo: borra la carpeta del kit. Lo único que queda fuera es el
modelo de voz, en la caché de tu usuario.

## Si el vídeo se corta a mitad

No se pierde nada. Cada paso escribe su resultado, y `proyecto.json` guarda por
dónde iba. Al volver, dile *"continúa el vídeo"* y sigue por el primer paso que
falte.

## Cómo se cobra esto (rangos de mercado 2026)

Si editas para otros: **edición de un vídeo vertical corto 25-60 €**, **paquete de 8
vídeos al mes 250-500 €**, **gestión completa de contenido vertical (edición +
publicación + miniaturas) 600-1.500 €/mes**. Un clip que a mano son 40 minutos aquí
son 3 de trabajo tuyo, y ahí está el margen. El precio lo pones tú.
#!/usr/bin/env python3
"""Convierte las plantillas HTML en secuencias de imágenes con transparencia.

Uso:  python scripts/capturar.py <carpeta-del-proyecto>

Lee trabajo/visuales.json, abre cada plantilla en un Chrome sin ventana y le
pide fotograma a fotograma que se coloque en un instante concreto. No se graba
la pantalla: se pide el fotograma 12 y se guarda el fotograma 12. Por eso el
resultado es idéntico siempre, aunque el ordenador vaya lento.

Formato de trabajo/visuales.json:
  {"visuales": [
     {"nombre": "rot1", "plantilla": "rotulo", "inicio": 1.4,
      "datos": {"texto": "Paso 1", "sub": "Instalar"}}
  ]}

Salida:
  trabajo/visuales/<nombre>-00000.png …
  trabajo/timeline.json   (con las capas ya colocadas, listas para componer)
"""
from __future__ import annotations

import argparse
import shutil
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

FPS_CAPTURA = 30   # el motor lo sube a 60 al montar; a 60 aquí tarda el doble

# Dónde cae cada plantilla en el lienzo vertical. Pensado para no chocar con
# los subtítulos (que van sobre y=1300) ni con la interfaz de la red social.
COLOCACION = {
    "rotulo": {"x": 0, "y": 980},
    "titulo": {"x": 0, "y": 300},
    "mockup-terminal": {"x": 0, "y": 400},
    "mockup-movil": {"x": 0, "y": 190},
    "recap": {"x": 0, "y": 360},
    "cta": {"x": 0, "y": 540},
}

AYUDA_PLAYWRIGHT = (
    "Faltan los visuales (Playwright y su Chrome). Instálalos con:\n"
    "  python -m pip install playwright\n"
    "  python -m playwright install chromium\n"
    "Tarda un par de minutos la primera vez. Si no los quieres, el vídeo se "
    "monta igual sin rótulos ni mockups: sáltate este paso."
)


def plantillas_disponibles():
    return sorted(p.stem for p in c.PLANTILLAS.glob("*.html"))


def _abrir_navegador():
    try:
        from playwright.sync_api import sync_playwright
    except ImportError:
        raise c.FaltaDependencia(AYUDA_PLAYWRIGHT)
    try:
        pw = sync_playwright().start()
    except Exception as e:
        raise c.FaltaDependencia("{0}\n\nDetalle: {1}".format(AYUDA_PLAYWRIGHT, e))
    try:
        navegador = pw.chromium.launch(args=["--force-color-profile=srgb",
                                             "--disable-lcd-text"])
    except Exception as e:
        pw.stop()
        raise c.FaltaDependencia("{0}\n\nDetalle: {1}".format(AYUDA_PLAYWRIGHT, e))
    return pw, navegador


def capturar_una(navegador, visual, destino, fps=FPS_CAPTURA, solo_una=False):
    """Genera las imágenes de un visual. Devuelve su entrada para el timeline."""
    nombre_plantilla = visual.get("plantilla", "")
    archivo = c.PLANTILLAS / (nombre_plantilla + ".html")
    if not archivo.is_file():
        raise FileNotFoundError(
            "No existe la plantilla '{0}'. Las que hay: {1}".format(
                nombre_plantilla, ", ".join(plantillas_disponibles()))
        )
    nombre = c.slug(visual.get("nombre") or nombre_plantilla)
    datos = visual.get("datos") or {}

    pagina = navegador.new_page(viewport={"width": c.ANCHO, "height": c.ALTO},
                                device_scale_factor=1)
    try:
        errores = []
        pagina.on("pageerror", lambda e: errores.append(str(e)))
        pagina.goto(archivo.resolve().as_uri())
        # Sin esperar a las tipografías, los primeros fotogramas salen con la
        # letra del sistema y se ve el cambio.
        pagina.wait_for_function("document.fonts.status === 'loaded'", timeout=15000)
        pagina.wait_for_function("typeof window.cargar === 'function'", timeout=15000)
        lienzo = pagina.evaluate("() => window.LIENZO")
        pagina.set_viewport_size({"width": int(lienzo["ancho"]),
                                  "height": int(lienzo["alto"])})
        pagina.evaluate("d => window.cargar(d)", datos)
        duracion = float(pagina.evaluate("() => window.duracion()"))
        if errores:
            raise RuntimeError("La plantilla '{0}' ha dado un error:\n  {1}".format(
                nombre_plantilla, errores[0]))

        carpeta = Path(destino) / "visuales"
        carpeta.mkdir(parents=True, exist_ok=True)
        for viejo in carpeta.glob(nombre + "-*.png"):
            viejo.unlink()

        total = max(int(round(duracion * fps)), 1)
        if solo_una:
            # Vista rápida: un solo fotograma del medio, para revisar el diseño.
            pagina.evaluate("t => window.fotograma(t)", duracion * 0.55)
            ruta = carpeta / (nombre + "-previa.png")
            pagina.screenshot(path=str(ruta), omit_background=True)
            print("  {0}: previa en {1}".format(nombre, ruta.name))
            return None

        for i in range(total):
            pagina.evaluate("t => window.fotograma(t)", i / float(fps))
            pagina.screenshot(path=str(carpeta / "{0}-{1:05d}.png".format(nombre, i)),
                              omit_background=True)
        if errores:
            print("  ! {0}: {1}".format(nombre, errores[0]))

        colocacion = dict(COLOCACION.get(nombre_plantilla, {"x": 0, "y": 0}))
        if "x" in visual:
            colocacion["x"] = int(visual["x"])
        if "y" in visual:
            colocacion["y"] = int(visual["y"])
        print("  {0}: {1} fotogramas, {2:.2f} s, en y={3}".format(
            nombre, total, duracion, colocacion["y"]))
        return {
            "secuencia": "visuales/" + nombre,
            "inicio": round(float(visual.get("inicio", 0)), 3),
            "fps": fps,
            "x": colocacion["x"],
            "y": colocacion["y"],
            "_plantilla": nombre_plantilla,
            "_duracion": round(duracion, 3),
        }
    finally:
        pagina.close()


def generar(proyecto, solo=None, fps=FPS_CAPTURA, previa=False, limpiar=False):
    estado = c.cargar_proyecto(proyecto)
    carpeta_trabajo = Path(estado["_carpeta"]) / "trabajo"
    archivo = carpeta_trabajo / "visuales.json"
    datos = c.leer_json(archivo, None)
    if datos is None:
        c.escribir_json(archivo, {
            "_leeme": ("Rótulos, titulares y mockups del vídeo. 'inicio' son "
                       "segundos del vídeo ya cortado. Plantillas disponibles: "
                       + ", ".join(plantillas_disponibles())),
            "visuales": [],
        })
        print("No había visuales. He creado trabajo/visuales.json vacío:\n"
              "  {0}\n"
              "Añade ahí los rótulos que quieras y vuelve a correr este paso.\n"
              "El vídeo se puede montar igualmente sin visuales.".format(archivo))
        return []

    visuales = list(datos.get("visuales") or [])
    if solo:
        visuales = [v for v in visuales
                    if c.slug(v.get("nombre") or v.get("plantilla", "")) == c.slug(solo)]
        if not visuales:
            raise FileNotFoundError(
                "No hay ningún visual llamado '{0}' en visuales.json".format(solo))
    if not visuales:
        print("visuales.json no tiene ningún visual. Nada que capturar.")
        return []

    if limpiar and (carpeta_trabajo / "visuales").is_dir():
        shutil.rmtree(carpeta_trabajo / "visuales")

    print("Generando {0} visual(es) con Chrome…".format(len(visuales)))
    pw, navegador = _abrir_navegador()
    capas = []
    try:
        for visual in visuales:
            entrada = capturar_una(navegador, visual, carpeta_trabajo, fps, previa)
            if entrada:
                capas.append(entrada)
    finally:
        navegador.close()
        pw.stop()

    if previa:
        print("\nSiguiente paso: mira las imágenes de trabajo/visuales/ y, si te "
              "gustan, corre esto sin --previa.")
        return capas

    # Se respeta lo que ya hubiera en el timeline: los zooms no se tocan y las
    # capas que no se han vuelto a generar se quedan como estaban.
    archivo_linea = carpeta_trabajo / "timeline.json"
    linea = c.leer_json(archivo_linea, {"zooms": [], "capas": [], "fijas": []})
    nuevas = {ca["secuencia"] for ca in capas}
    linea["capas"] = [ca for ca in (linea.get("capas") or [])
                      if ca.get("secuencia") not in nuevas] + capas
    linea["capas"].sort(key=lambda ca: ca.get("inicio", 0))
    linea.setdefault("zooms", [])
    linea.setdefault("fijas", [])
    c.escribir_json(archivo_linea, linea)

    estado.setdefault("pasos", {})["capturar"] = "ok"
    c.guardar_proyecto(estado)
    print("\n{0} capas colocadas en trabajo/timeline.json".format(len(capas)))
    print("\nSiguiente paso: montar el vídeo con los visuales")
    print("  python scripts/componer.py \"{0}\" --refrescar".format(
        estado["_carpeta"]))
    return capas


def main():
    ap = argparse.ArgumentParser(description="Genera los visuales del vídeo")
    ap.add_argument("proyecto")
    ap.add_argument("--solo", help="generar solo el visual con ese nombre")
    ap.add_argument("--fps", type=int, default=FPS_CAPTURA)
    ap.add_argument("--previa", action="store_true",
                    help="una sola imagen de cada visual, para revisar el diseño")
    ap.add_argument("--limpiar", action="store_true",
                    help="borrar las imágenes anteriores")
    a = ap.parse_args()
    try:
        generar(a.proyecto, a.solo, a.fps, a.previa, a.limpiar)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Monta el vídeo final: zooms, capas visuales y subtítulos, en una pasada.

Uso:  python scripts/componer.py <carpeta-del-proyecto>

Orden de las capas, de abajo a arriba:
  1. master.mp4 (ya cortado y en vertical)
  2. zooms          — el mismo plano ampliado, encima del trozo original
  3. capas          — rótulos, mockups y titulares (secuencias PNG)
  4. imágenes fijas — fotos o clips propios a pantalla completa
  5. subtítulos     — siempre lo último, para que nada los tape

Los zooms y las capas son opcionales: si no hay trabajo/timeline.json, se
montan solo los subtítulos y el vídeo queda igualmente terminado.

Salida: trabajo/montaje.mp4
"""
from __future__ import annotations

import argparse
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

TRANSPARENTE = "color=black@0.0"
SUPERMUESTREO = 2   # se amplía sobre el doble de tamaño: el zoom no da saltos


def _expresion_zoom(z, modo, duracion):
    """Curva del zoom. `it` es el tiempo dentro del trozo, en segundos."""
    dz = max(z - 1.0, 0.0)
    u = "(it/{0:.3f})".format(max(duracion, 0.05))
    # Un temblor mínimo, como de cámara en mano: sin esto el zoom parece
    # de diapositivas.
    temblor = "(1+0.012*sin(it*2.3)+0.008*sin(it*3.7+1.3))"
    if modo == "suave":
        # Arranca y frena despacio (3u²-2u³).
        curva = "(3*{u}*{u}-2*{u}*{u}*{u})".format(u=u)
    else:
        # Entra con un rebote pequeño y se asienta.
        curva = "(1-exp(-6*{u})*cos(7.5*{u}))".format(u=u)
    return "1+{0:.4f}*{1}*{2}".format(dz, curva, temblor)


def capa_zoom(indice, zoom, etiqueta_entrada):
    """Filtros de un zoom: recorta el trozo, lo amplía y lo alinea en el tiempo."""
    inicio = float(zoom["inicio"])
    fin = float(zoom["fin"])
    duracion = max(fin - inicio, 0.1)
    z = float(zoom.get("z", 1.15))
    expr = _expresion_zoom(z, zoom.get("modo", "resorte"), duracion)
    # El encuadre se va un poco al tercio alto: es donde está la cara.
    x = "iw/2-(iw/zoom/2)+7*sin(it*1.7)"
    y = "(ih-ih/zoom)*0.38+5*sin(it*1.3+2)"
    return (
        "[{ent}]trim={a:.4f}:{b:.4f},setpts=PTS-STARTPTS,"
        "scale={W}:{H}:flags=lanczos,"
        "zoompan=z='{z}':x='{x}':y='{y}':d=1:s={w}x{h}:fps={fps},"
        "format=rgba,tpad=start_duration={a:.4f}:{t}[zoom{i}]"
    ).format(ent=etiqueta_entrada, a=inicio, b=fin, z=expr, x=x, y=y,
             W=c.ANCHO * SUPERMUESTREO, H=c.ALTO * SUPERMUESTREO,
             w=c.ANCHO, h=c.ALTO, fps=c.FPS, t=TRANSPARENTE, i=indice)


def componer(proyecto, calidad=17, refrescar=False):
    estado = c.cargar_proyecto(proyecto)
    salida = c.trabajo(estado, "montaje.mp4")
    if salida.is_file() and not refrescar:
        print("montaje.mp4 ya existe. Usa --refrescar para rehacerlo.")
        return salida

    master = c.trabajo(estado, "master.mp4")
    if not master.is_file():
        raise FileNotFoundError(
            "Falta trabajo/master.mp4. Corre antes:\n"
            "  python scripts/cortar.py \"{0}\"".format(estado["_carpeta"])
        )
    total = c.duracion(master)
    subs = c.leer_json(c.trabajo(estado, "subs.json"), {"grupos": []})
    linea = c.leer_json(c.trabajo(estado, "timeline.json"),
                        {"zooms": [], "capas": [], "fijas": []})

    zooms = [z for z in linea.get("zooms") or [] if z.get("fin", 0) > z.get("inicio", 0)]
    capas = list(linea.get("capas") or [])
    fijas = list(linea.get("fijas") or [])

    entradas = ["-i", str(master)]
    filtros = []
    indice = 1

    # El master se reparte en tantas copias como zooms haya, más la base.
    if zooms:
        salidas = "[base]" + "".join("[zc{0}]".format(i) for i in range(len(zooms)))
        filtros.append("[0:v]split={0}{1}".format(len(zooms) + 1, salidas))
        for i, z in enumerate(zooms):
            filtros.append(capa_zoom(i, z, "zc{0}".format(i)))
    else:
        filtros.append("[0:v]null[base]")

    # Capas visuales: secuencias de PNG generadas por capturar.py.
    etiquetas_capa = []
    for i, capa in enumerate(capas):
        patron = Path(estado["_carpeta"]) / "trabajo" / capa["secuencia"]
        primero = patron.parent / (patron.name + "-00000.png") if patron.suffix == "" \
            else patron
        if patron.suffix == "":
            patron = patron.parent / (patron.name + "-%05d.png")
        if not Path(str(primero)).is_file():
            print("  ! Falta la secuencia {0}: se salta esa capa.".format(
                capa["secuencia"]))
            continue
        entradas += ["-framerate", str(capa.get("fps", 30)),
                     "-start_number", "0", "-i", str(patron)]
        filtros.append(
            "[{k}:v]format=rgba,tpad=start_duration={a:.4f}:{t},fps={fps}[cap{i}]"
            .format(k=indice, a=float(capa.get("inicio", 0)), t=TRANSPARENTE,
                    fps=c.FPS, i=i))
        etiquetas_capa.append(("cap{0}".format(i), capa.get("x", 0),
                               capa.get("y", 0)))
        indice += 1

    # Imágenes o clips a pantalla completa, con entrada y salida suaves.
    etiquetas_fija = []
    for i, fija in enumerate(fijas):
        archivo = Path(fija["archivo"]).expanduser()
        if not archivo.is_absolute():
            archivo = c.RAIZ / archivo
        if not archivo.is_file():
            print("  ! No encuentro {0}: se salta.".format(archivo))
            continue
        inicio = float(fija.get("inicio", 0))
        dur = max(float(fija.get("fin", inicio + 2)) - inicio, 0.2)
        entradas += ["-i", str(archivo)]
        fundido = 0.18
        filtros.append(
            "[{k}:v]scale={W}:{H}:force_original_aspect_ratio=increase,"
            "crop={W}:{H},format=rgba,loop=loop=-1:size=1:start=0,"
            "trim=0:{d:.4f},setpts=PTS-STARTPTS,"
            "fade=t=in:st=0:d={f}:alpha=1,fade=t=out:st={fo:.4f}:d={f}:alpha=1,"
            "tpad=start_duration={a:.4f}:{t},fps={fps}[fija{i}]"
            .format(k=indice, W=c.ANCHO, H=c.ALTO, d=dur, f=fundido,
                    fo=max(dur - fundido, 0.01), a=inicio, t=TRANSPARENTE,
                    fps=c.FPS, i=i))
        etiquetas_fija.append("fija{0}".format(i))
        indice += 1

    # Subtítulos: la última capa, para que ningún efecto los tape.
    lista_subs = c.trabajo(estado, "subs", "lista.txt")
    hay_subs = bool(subs.get("grupos")) and lista_subs.is_file()
    if hay_subs:
        entradas += ["-f", "concat", "-safe", "0", "-i", str(lista_subs)]
        filtros.append("[{k}:v]format=rgba,fps={fps}[subs]".format(
            k=indice, fps=c.FPS))
        indice += 1

    # Se van apilando las capas por orden.
    actual = "base"
    paso = 0

    def encadenar(etiqueta, x=0, y=0):
        nonlocal actual, paso
        filtros.append("[{0}][{1}]overlay={2}:{3}:eof_action=pass[mix{4}]".format(
            actual, etiqueta, x, y, paso))
        actual = "mix{0}".format(paso)
        paso += 1

    for i in range(len(zooms)):
        encadenar("zoom{0}".format(i))
    for etiqueta, x, y in etiquetas_capa:
        encadenar(etiqueta, x, y)
    for etiqueta in etiquetas_fija:
        encadenar(etiqueta)
    if hay_subs:
        encadenar("subs", 0, subs.get("y", 1300))

    filtros.append("[{0}]format=yuv420p[vout]".format(actual))

    args = entradas + ["-filter_complex", ";".join(filtros),
                       "-map", "[vout]"]
    if estado["origen_info"]["tiene_audio"]:
        args += ["-map", "0:a", "-c:a", "copy"]
    else:
        args += ["-an"]
    args += ["-t", "{0:.3f}".format(total),
             "-c:v", "libx264", "-preset", "medium", "-crf", str(calidad),
             "-pix_fmt", "yuv420p", "-movflags", "+faststart", str(salida)]

    piezas = []
    if hay_subs:
        piezas.append("{0} grupos de subtítulos".format(len(subs["grupos"])))
    if zooms:
        piezas.append("{0} zooms".format(len(zooms)))
    if etiquetas_capa:
        piezas.append("{0} capas".format(len(etiquetas_capa)))
    if etiquetas_fija:
        piezas.append("{0} imágenes".format(len(etiquetas_fija)))
    print("Montando: {0}…".format(", ".join(piezas) or "solo el vídeo base"))
    c.correr_ffmpeg(args)

    print("  ✓ montaje.mp4  {0}".format(c.hhmmss(c.duracion(salida))))
    estado.setdefault("pasos", {})["componer"] = "ok"
    c.guardar_proyecto(estado)
    print("\nSiguiente paso: añadir los efectos de sonido y cerrar el vídeo")
    print("  python scripts/sonido.py \"{0}\"".format(estado["_carpeta"]))
    return salida


def main():
    ap = argparse.ArgumentParser(description="Monta subtítulos y efectos")
    ap.add_argument("proyecto")
    ap.add_argument("--calidad", type=int, default=17)
    ap.add_argument("--refrescar", action="store_true")
    a = ap.parse_args()
    try:
        componer(a.proyecto, a.calidad, a.refrescar)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Utilidades compartidas del motor de edición.

Todo el motor pasa por aquí: localizar ffmpeg, leer datos de un vídeo,
cargar las tipografías del kit y guardar el estado del proyecto.
Funciona igual en Mac y en Windows: nada de comandos de shell, solo Python.
"""
from __future__ import annotations

import json
import os
import platform
import shutil
import subprocess
import sys
from pathlib import Path

RAIZ = Path(__file__).resolve().parent.parent
BIN = RAIZ / "bin"
ASSETS = RAIZ / "assets"
FUENTES = ASSETS / "fuentes"
SFX = ASSETS / "sfx"
PLANTILLAS = RAIZ / "plantillas"
WORKSPACE = RAIZ / "workspace"

# Lienzo vertical de destino (TikTok / Reels / Shorts)
ANCHO, ALTO = 1080, 1920
FPS = 60

ES_WINDOWS = platform.system() == "Windows"

# ── binarios externos ───────────────────────────────────────────────────────

# Sitios donde suele quedar ffmpeg cuando no está en el PATH. La carpeta bin/
# del propio kit va primero: ahí deja el wizard la copia portátil que descarga
# cuando el ordenador no tiene ffmpeg. Así se instala sin permisos de
# administrador y sin tocar nada del sistema.
_CANDIDATOS = {
    "Darwin": [str(BIN), "/opt/homebrew/bin", "/usr/local/bin",
               str(Path.home() / "bin")],
    "Windows": [
        str(BIN),
        r"C:\ffmpeg\bin",
        str(Path(os.environ.get("LOCALAPPDATA", "")) / "Microsoft" / "WinGet" / "Links"),
        str(Path(os.environ.get("PROGRAMFILES", "C:\\Program Files")) / "ffmpeg" / "bin"),
    ],
    "Linux": [str(BIN), "/usr/bin", "/usr/local/bin", "/snap/bin"],
}


_cache_binarios = {}


def _arranca(ruta):
    """Comprueba que el ejecutable realmente corre en esta máquina.

    Existir no basta: se ven instalaciones a medias y binarios compilados
    para otro procesador (un ffprobe de Intel en un Mac con chip Apple da
    'Bad CPU type'). Si no arranca, es como si no estuviera.
    """
    try:
        proc = subprocess.run(
            [str(ruta), "-version"], capture_output=True, timeout=20,
        )
        return proc.returncode == 0
    except (OSError, subprocess.SubprocessError):
        return False


def buscar_binario(nombre):
    """Devuelve la ruta de un ejecutable que funciona, o None."""
    if nombre in _cache_binarios:
        return _cache_binarios[nombre]
    sufijo = ".exe" if ES_WINDOWS else ""
    vistos, candidatos = set(), []
    # La copia del kit va antes que la del sistema a propósito: si está, es
    # porque la descargó el wizard, que además comprobó que trae los
    # codificadores. Un ffmpeg recortado del sistema no la puede tapar.
    for carpeta in [str(BIN)] + os.environ.get("PATH", "").split(
            os.pathsep) + _CANDIDATOS.get(platform.system(), []):
        if not carpeta or carpeta in vistos:
            continue
        vistos.add(carpeta)
        ruta = Path(carpeta) / (nombre + sufijo)
        if ruta.is_file():
            candidatos.append(str(ruta))
    elegido = next((c for c in candidatos if _arranca(c)), None)
    _cache_binarios[nombre] = elegido
    return elegido


class FaltaDependencia(RuntimeError):
    """Falta un programa necesario. El mensaje ya explica cómo instalarlo."""


def _exigir(nombre):
    ruta = buscar_binario(nombre)
    if ruta:
        return ruta
    raise FaltaDependencia(
        "No encuentro '{0}', el programa que trabaja con el vídeo.\n"
        "Se instala solo, sin permisos de administrador, con:\n"
        "  python scripts/instalar_ffmpeg.py".format(nombre)
    )


def ffmpeg():
    return _exigir("ffmpeg")


def ffprobe():
    return _exigir("ffprobe")


def correr(cmd, entrada=None, silencioso=True):
    """Ejecuta un comando y devuelve su salida. Si falla, explica por qué."""
    cmd = [str(c) for c in cmd]
    proc = subprocess.run(
        cmd, input=entrada, capture_output=True, text=True,
        encoding="utf-8", errors="replace",
    )
    if proc.returncode != 0:
        cola = (proc.stderr or proc.stdout or "").strip().splitlines()
        detalle = "\n".join(cola[-12:]) if cola else "(sin mensaje de error)"
        raise RuntimeError(
            "Falló:  {0}\n{1}".format(" ".join(cmd[:3]) + " …", detalle)
        )
    if not silencioso and proc.stderr:
        print(proc.stderr, file=sys.stderr)
    return proc.stdout


def correr_ffmpeg(args, descripcion=""):
    """ffmpeg con log mínimo. `args` no incluye el binario ni -y."""
    salida = correr([ffmpeg(), "-y", "-nostdin", "-v", "error"] + list(args))
    if descripcion:
        print("  ✓ {0}".format(descripcion))
    return salida


# ── datos de un vídeo ──────────────────────────────────────────────────────


def info_media(archivo):
    """Devuelve {duracion, ancho, alto, fps, tiene_audio, rotacion}."""
    crudo = correr([
        ffprobe(), "-v", "error", "-print_format", "json",
        "-show_format", "-show_streams", str(archivo),
    ])
    datos = json.loads(crudo)
    video = next((s for s in datos["streams"] if s["codec_type"] == "video"), None)
    audio = next((s for s in datos["streams"] if s["codec_type"] == "audio"), None)
    info = {
        "duracion": float(datos["format"].get("duration", 0) or 0),
        "tiene_audio": audio is not None,
        "ancho": 0, "alto": 0, "fps": 0.0, "rotacion": 0,
    }
    if video:
        info["ancho"] = int(video.get("width", 0))
        info["alto"] = int(video.get("height", 0))
        num, _, den = (video.get("r_frame_rate") or "0/1").partition("/")
        try:
            info["fps"] = round(float(num) / float(den or 1), 3)
        except (ValueError, ZeroDivisionError):
            info["fps"] = 0.0
        # Vídeo de móvil: la rotación viene en metadatos, no en las dimensiones.
        for lado in video.get("side_data_list", []) or []:
            if "rotation" in lado:
                info["rotacion"] = int(lado["rotation"]) % 360
        # Si está rotado 90/270, el tamaño real que ve el reproductor va girado.
        if info["rotacion"] in (90, 270):
            info["ancho"], info["alto"] = info["alto"], info["ancho"]
    return info


def duracion(archivo):
    return info_media(archivo)["duracion"]


# ── tipografías del kit ────────────────────────────────────────────────────

_ARCHIVOS_FUENTE = {
    "titulo": FUENTES / "SpaceGrotesk-variable.ttf",   # subtítulos y números
    "mega": FUENTES / "Inter-variable.ttf",            # titulares grandes
    "mono": FUENTES / "JetBrainsMono-variable.ttf",    # terminal y etiquetas
}
_cache_fuentes = {}


def fuente(familia, tamano, peso=700):
    """Carga una tipografía del kit con el grosor pedido (100-900)."""
    from PIL import ImageFont  # import local: el doctor corre sin Pillow

    clave = (familia, tamano, peso)
    if clave in _cache_fuentes:
        return _cache_fuentes[clave]
    ruta = _ARCHIVOS_FUENTE.get(familia)
    if ruta is None or not ruta.is_file():
        raise FaltaDependencia(
            "Falta la tipografía '{0}' en assets/fuentes. Descarga el kit "
            "completo otra vez.".format(familia)
        )
    tipo = ImageFont.truetype(str(ruta), tamano)
    try:
        ejes = [a["name"].decode() if isinstance(a["name"], bytes) else a["name"]
                for a in tipo.get_variation_axes()]
        valores = []
        for nombre in ejes:
            valores.append(peso if nombre.lower().startswith("weight") else None)
        # Los ejes que no sean el grosor se quedan en su valor por defecto.
        defectos = [a["default"] for a in tipo.get_variation_axes()]
        valores = [v if v is not None else d for v, d in zip(valores, defectos)]
        tipo.set_variation_by_axes(valores)
    except Exception:
        pass  # fuente estática o FreeType sin soporte: se usa tal cual
    _cache_fuentes[clave] = tipo
    return tipo


# ── proyecto ───────────────────────────────────────────────────────────────

NOMBRE_ESTADO = "proyecto.json"


def carpeta_proyecto(ruta):
    """Acepta la carpeta del proyecto o su proyecto.json."""
    p = Path(ruta).expanduser().resolve()
    return p.parent if p.name == NOMBRE_ESTADO else p


_SIN_DEFECTO = object()   # para poder pedir None como valor por defecto


def leer_json(archivo, por_defecto=_SIN_DEFECTO):
    """Lee un JSON. Sin valor por defecto, la falta del archivo es un error."""
    archivo = Path(archivo)
    if not archivo.is_file():
        if por_defecto is _SIN_DEFECTO:
            raise FileNotFoundError("No existe {0}".format(archivo))
        return por_defecto
    return json.loads(archivo.read_text(encoding="utf-8"))


def escribir_json(archivo, datos):
    archivo = Path(archivo)
    archivo.parent.mkdir(parents=True, exist_ok=True)
    archivo.write_text(
        json.dumps(datos, indent=2, ensure_ascii=False) + "\n", encoding="utf-8"
    )
    return archivo


def cargar_proyecto(ruta):
    carpeta = carpeta_proyecto(ruta)
    archivo = carpeta / NOMBRE_ESTADO
    if not archivo.is_file():
        raise FileNotFoundError(
            "No hay ningún proyecto en {0} (falta {1}). Empieza con "
            "nuevo.py \"<vídeo>\".".format(carpeta, NOMBRE_ESTADO)
        )
    estado = leer_json(archivo)
    estado["_carpeta"] = str(carpeta)
    return estado


def ruta_origen(estado):
    """El vídeo original del proyecto, comprobando que siga estando ahí.

    Los pasos que releen el original (transcribir, encuadre, cortar) pasan por
    aquí: si el archivo se movió o se renombró después de crear el proyecto, el
    error lo explica en vez de dejar que ffmpeg diga 'No such file'.
    """
    origen = Path(estado.get("origen", ""))
    if not origen.is_file():
        raise FileNotFoundError(
            "No encuentro el vídeo original en {0}. Corrige 'origen' en "
            "proyecto.json.".format(origen)
        )
    return origen


def guardar_proyecto(estado):
    carpeta = Path(estado["_carpeta"])
    copia = {k: v for k, v in estado.items() if not k.startswith("_")}
    escribir_json(carpeta / NOMBRE_ESTADO, copia)
    return carpeta / NOMBRE_ESTADO


def trabajo(estado, *partes):
    """Ruta dentro de la carpeta de archivos intermedios del proyecto."""
    ruta = Path(estado["_carpeta"]) / "trabajo"
    for parte in partes:
        ruta = ruta / parte
    ruta.parent.mkdir(parents=True, exist_ok=True)
    return ruta


def slug(texto, maximo=40):
    import re
    limpio = re.sub(r"[^a-z0-9]+", "-", texto.lower().strip())
    limpio = re.sub(r"-+", "-", limpio).strip("-")
    return (limpio or "video")[:maximo]


def hhmmss(segundos):
    segundos = max(float(segundos), 0.0)
    m, s = divmod(segundos, 60)
    return "{0:d}:{1:05.2f}".format(int(m), s)
#!/usr/bin/env python3
"""Aplica el plan de cortes y deja el vídeo en vertical 1080x1920.

Uso:  python scripts/cortar.py <carpeta-del-proyecto>

Se hace todo en una sola pasada de ffmpeg: recortar los trozos, pegarlos,
pasar a vertical y dejarlo a 60 fps. Una sola pasada significa una sola
recompresión, así que no se pierde calidad por el camino.

Salida: trabajo/master.mp4
"""
from __future__ import annotations

import argparse
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c
from encuadre import medidas_recorte

MAX_TROZOS = 400


def cadena_vertical(info, encuadre):
    """Filtros que convierten el vídeo ya cortado en 1080x1920."""
    ancho, alto = info["ancho"], info["alto"]
    centro = float(encuadre.get("centro", 0.5))
    if encuadre.get("modo") == "marco":
        # El vídeo entero, con su propio fondo desenfocado detrás. Se ve todo,
        # aunque queden franjas arriba y abajo.
        return (
            "split=2[bg][fg];"
            "[bg]scale={a}:{b}:force_original_aspect_ratio=increase,"
            "crop={a}:{b},boxblur=42:2,eq=brightness=-0.12[bgb];"
            "[fg]scale={a}:-2:flags=lanczos[fgs];"
            "[bgb][fgs]overlay=(W-w)/2:(H-h)*0.42"
        ).format(a=c.ANCHO, b=c.ALTO)
    x, y, w, h = medidas_recorte(ancho, alto, centro)
    # Al ampliar un recorte se pierde nitidez; un unsharp suave lo compensa
    # sin que se marquen bordes.
    return ("crop={w}:{h}:{x}:{y},scale={A}:{B}:flags=lanczos,"
            "unsharp=5:5:0.5:5:5:0.0").format(
        w=w, h=h, x=x, y=y, A=c.ANCHO, B=c.ALTO)


def construir_filtro(mapa, tiene_audio, vertical):
    """Grafo completo: trozo a trozo, pegado y pasado a vertical."""
    n = len(mapa)
    partes = []
    if n == 1 and mapa[0][0] <= 0.01:
        partes.append("[0:v]null[vc]")
        if tiene_audio:
            partes.append("[0:a]anull[ac]")
    else:
        partes.append("[0:v]split={0}{1}".format(
            n, "".join("[v{0}]".format(i) for i in range(n))))
        if tiene_audio:
            partes.append("[0:a]asplit={0}{1}".format(
                n, "".join("[a{0}]".format(i) for i in range(n))))
        etiquetas = []
        for i, (a, b, _) in enumerate(mapa):
            partes.append("[v{0}]trim={1:.4f}:{2:.4f},setpts=PTS-STARTPTS[tv{0}]"
                          .format(i, a, b))
            etiquetas.append("[tv{0}]".format(i))
            if tiene_audio:
                partes.append(
                    "[a{0}]atrim={1:.4f}:{2:.4f},asetpts=PTS-STARTPTS[ta{0}]"
                    .format(i, a, b))
                etiquetas.append("[ta{0}]".format(i))
        partes.append("{0}concat=n={1}:v=1:a={2}[vc]{3}".format(
            "".join(etiquetas), n, 1 if tiene_audio else 0,
            "[ac]" if tiene_audio else ""))
    partes.append("[vc]{0},fps={1},setsar=1,format=yuv420p[vout]".format(
        vertical, c.FPS))
    return ";".join(partes)


def cortar(proyecto, calidad=16, refrescar=False):
    estado = c.cargar_proyecto(proyecto)
    salida = c.trabajo(estado, "master.mp4")
    if salida.is_file() and not refrescar:
        print("master.mp4 ya existe ({0}). Usa --refrescar para rehacerlo."
              .format(c.hhmmss(c.duracion(salida))))
        return salida

    origen = c.ruta_origen(estado)
    info = estado["origen_info"]
    plan = c.leer_json(c.trabajo(estado, "plan-cortes.json"), None)
    if plan and plan.get("mapa"):
        mapa = plan["mapa"]
        esperado = plan["duracion_final"]
    else:
        print("  ! No hay plan de cortes: se usa el vídeo entero.")
        mapa = [[0.0, float(info["duracion"]), 0.0]]
        esperado = float(info["duracion"])

    if len(mapa) > MAX_TROZOS:
        raise RuntimeError(
            "El plan tiene {0} trozos y eso es demasiado para una pasada. "
            "Sube el umbral de silencio para agrupar cortes:\n"
            "  python scripts/plan_cortes.py \"{1}\" --umbral 0.9"
            .format(len(mapa), estado["_carpeta"])
        )

    encuadre = estado.get("encuadre") or {"modo": "recorte", "centro": 0.5}
    if "encuadre" not in estado:
        print("  ! Sin encuadre elegido: se recorta por el centro. Para ajustarlo:")
        print("    python scripts/encuadre.py \"{0}\"".format(estado["_carpeta"]))

    filtro = construir_filtro(mapa, bool(info["tiene_audio"]),
                              cadena_vertical(info, encuadre))
    args = ["-i", str(origen), "-filter_complex", filtro,
            "-map", "[vout]"]
    if info["tiene_audio"]:
        args += ["-map", "[ac]", "-c:a", "aac", "-b:a", "192k", "-ar", "48000"]
    else:
        args += ["-an"]
    args += ["-c:v", "libx264", "-preset", "medium", "-crf", str(calidad),
             "-pix_fmt", "yuv420p", "-movflags", "+faststart", str(salida)]

    print("Cortando y pasando a vertical ({0} trozos)…".format(len(mapa)))
    c.correr_ffmpeg(args)

    real = c.duracion(salida)
    print("  ✓ master.mp4  {0}  ({1}x{2} a {3} fps)".format(
        c.hhmmss(real), c.ANCHO, c.ALTO, c.FPS))
    if abs(real - esperado) > 0.5:
        print("  ! Esperaba {0} y ha salido {1}. Si la diferencia se nota, "
              "revisa el plan de cortes.".format(c.hhmmss(esperado), c.hhmmss(real)))

    estado["duracion_master"] = round(real, 3)
    estado.setdefault("pasos", {})["cortar"] = "ok"
    c.guardar_proyecto(estado)
    print("\nSiguiente paso: generar los subtítulos")
    print("  python scripts/subtitulos.py \"{0}\"".format(estado["_carpeta"]))
    return salida


def main():
    ap = argparse.ArgumentParser(description="Corta y pasa el vídeo a vertical")
    ap.add_argument("proyecto")
    ap.add_argument("--calidad", type=int, default=16,
                    help="crf de x264: menos es mejor calidad (por defecto 16)")
    ap.add_argument("--refrescar", action="store_true")
    a = ap.parse_args()
    try:
        cortar(a.proyecto, a.calidad, a.refrescar)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Revisa que el kit puede trabajar en este ordenador.

Uso:
  python scripts/doctor.py             ver qué falta
  python scripts/doctor.py --instalar  instalar lo que se pueda desde Python
  python scripts/doctor.py --json      resumen para que lo lea Claude

Comprueba lo imprescindible (Python, ffmpeg, Pillow, transcripción) y lo
opcional (los visuales). Cada cosa que falte viene con el comando exacto
para arreglarla en este sistema operativo.
"""
from __future__ import annotations

import argparse
import json
import platform
import shutil
import subprocess
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

SISTEMA = platform.system()
MODELO_WHISPER = "small"

# ffmpeg se instala dentro del kit, sin permisos de administrador ni gestores de
# paquetes. El detalle está en instalar_ffmpeg.py.
ARREGLO_FFMPEG = "python scripts/instalar_ffmpeg.py"


class Punto:
    def __init__(self, nombre, estado, detalle="", arreglo="", clave=""):
        self.nombre = nombre
        self.estado = estado          # ok | aviso | falta
        self.detalle = detalle
        self.arreglo = arreglo
        self.clave = clave or nombre

    def a_dict(self):
        return {"nombre": self.nombre, "clave": self.clave,
                "estado": self.estado, "detalle": self.detalle,
                "arreglo": self.arreglo}


def _version_paquete(modulo, atributo="__version__"):
    try:
        mod = __import__(modulo)
        return str(getattr(mod, atributo, "instalado"))
    except ImportError:
        return None


def _codificadores():
    """Qué codificadores trae este ffmpeg. Sin libx264 no hay vídeo."""
    try:
        salida = c.correr([c.ffmpeg(), "-hide_banner", "-encoders"])
    except (c.FaltaDependencia, RuntimeError):
        return set()
    return {linea.split()[1] for linea in salida.splitlines()
            if linea.startswith(" ") and len(linea.split()) > 1}


def _carpeta_modelos():
    """Dónde deja faster-whisper los modelos que descarga."""
    import os
    base = os.environ.get("HF_HOME") or os.environ.get("HUGGINGFACE_HUB_CACHE")
    if base:
        raiz = Path(base)
        return raiz if raiz.name == "hub" else raiz / "hub"
    return Path.home() / ".cache" / "huggingface" / "hub"


def revisar():
    puntos = []

    # ── Python ──────────────────────────────────────────────────────────
    v = sys.version_info
    puntos.append(Punto(
        "Python", "ok" if v >= (3, 9) else "falta",
        "{0}.{1}.{2}".format(v.major, v.minor, v.micro),
        "" if v >= (3, 9) else "Necesitas Python 3.9 o más nuevo: "
                               "https://www.python.org/downloads/",
        clave="python"))

    # ── ffmpeg y ffprobe ────────────────────────────────────────────────
    arreglo_ffmpeg = ARREGLO_FFMPEG
    ruta_ffmpeg = c.buscar_binario("ffmpeg")
    ruta_ffprobe = c.buscar_binario("ffprobe")
    if ruta_ffmpeg and ruta_ffprobe:
        try:
            primera = c.correr([ruta_ffmpeg, "-version"]).splitlines()[0]
            version = primera.split(" ")[2] if len(primera.split(" ")) > 2 else "?"
        except RuntimeError:
            version = "?"
        puntos.append(Punto("ffmpeg", "ok", "{0} · {1}".format(version, ruta_ffmpeg),
                            clave="ffmpeg"))
        codis = _codificadores()
        faltan = [x for x in ("libx264", "aac") if x not in codis]
        puntos.append(Punto(
            "Codificadores de vídeo",
            "ok" if not faltan else "falta",
            "libx264 y aac disponibles" if not faltan
            else "falta " + " y ".join(faltan),
            "" if not faltan else
            "Tu ffmpeg viene recortado y no puede crear vídeo. Instala el "
            "completo:  " + arreglo_ffmpeg,
            clave="codificadores"))
    else:
        cual = "ffmpeg y ffprobe" if not (ruta_ffmpeg or ruta_ffprobe) else (
            "ffprobe" if ruta_ffmpeg else "ffmpeg")
        puntos.append(Punto("ffmpeg", "falta", "no encuentro " + cual,
                            arreglo_ffmpeg, clave="ffmpeg"))

    # ── Pillow: dibuja los subtítulos ───────────────────────────────────
    pil = _version_paquete("PIL", "__version__")
    puntos.append(Punto("Pillow (subtítulos)", "ok" if pil else "falta",
                        pil or "no instalado",
                        "" if pil else "python -m pip install --user Pillow",
                        clave="pillow"))
    if pil:
        try:
            c.fuente("titulo", 40, 700)
            puntos.append(Punto("Tipografías del kit", "ok",
                                "las tres cargan", clave="fuentes"))
        except Exception as e:
            puntos.append(Punto("Tipografías del kit", "falta", str(e)[:120],
                                "Vuelve a descargar el kit completo: falta la "
                                "carpeta assets/fuentes.", clave="fuentes"))
    else:
        puntos.append(Punto("Tipografías del kit", "aviso",
                            "sin Pillow no se pueden comprobar", clave="fuentes"))

    # ── transcripción ───────────────────────────────────────────────────
    fw = _version_paquete("faster_whisper")
    puntos.append(Punto("Transcripción (faster-whisper)",
                        "ok" if fw else "falta", fw or "no instalado",
                        "" if fw else "python -m pip install --user faster-whisper",
                        clave="whisper"))
    if fw:
        carpeta = _carpeta_modelos()
        descargado = list(carpeta.glob("models--Systran--faster-whisper-"
                                       + MODELO_WHISPER)) if carpeta.is_dir() else []
        puntos.append(Punto(
            "Modelo de voz '{0}'".format(MODELO_WHISPER),
            "ok" if descargado else "aviso",
            "descargado" if descargado else "se descargará al transcribir "
                                            "(unos 500 MB, solo la primera vez)",
            clave="modelo"))

    # ── visuales (opcional) ─────────────────────────────────────────────
    pw = _version_paquete("playwright")
    if not pw:
        puntos.append(Punto("Visuales (Playwright)", "aviso", "no instalado",
                            "python -m pip install --user playwright  y luego  "
                            "python -m playwright install chromium",
                            clave="playwright"))
    else:
        try:
            from playwright.sync_api import sync_playwright
            with sync_playwright() as p:
                nav = p.chromium.launch()
                version = nav.version
                nav.close()
            puntos.append(Punto("Visuales (Playwright)", "ok",
                                "Chrome " + version, clave="playwright"))
        except Exception as e:
            puntos.append(Punto(
                "Visuales (Playwright)", "aviso",
                "instalado pero Chrome no arranca",
                "python -m playwright install chromium\n"
                "  Detalle: {0}".format(str(e).strip().splitlines()[0][:160]),
                clave="playwright"))

    # ── material del kit ────────────────────────────────────────────────
    sonidos = list(c.SFX.glob("*.mp3"))
    puntos.append(Punto("Efectos de sonido",
                        "ok" if len(sonidos) >= 10 else "aviso",
                        "{0} sonidos".format(len(sonidos)),
                        "" if len(sonidos) >= 10 else
                        "Vuelve a descargar el kit: falta assets/sfx.",
                        clave="sfx"))
    plantillas = list(c.PLANTILLAS.glob("*.html"))
    puntos.append(Punto("Plantillas visuales",
                        "ok" if len(plantillas) >= 4 else "aviso",
                        "{0} plantillas".format(len(plantillas)),
                        "" if len(plantillas) >= 4 else
                        "Vuelve a descargar el kit: falta la carpeta plantillas.",
                        clave="plantillas"))

    # ── sitio para trabajar ─────────────────────────────────────────────
    try:
        c.WORKSPACE.mkdir(parents=True, exist_ok=True)
        prueba = c.WORKSPACE / ".escritura"
        prueba.write_text("ok", encoding="utf-8")
        prueba.unlink()
        libre = shutil.disk_usage(c.WORKSPACE).free / (1024 ** 3)
        puntos.append(Punto(
            "Espacio en disco", "ok" if libre >= 5 else "aviso",
            "{0:.1f} GB libres".format(libre),
            "" if libre >= 5 else "Vas justo. Un vídeo de un minuto puede "
                                  "ocupar 2 GB mientras se monta.",
            clave="disco"))
    except OSError as e:
        puntos.append(Punto("Espacio en disco", "falta", str(e)[:120],
                            "No puedo escribir en la carpeta workspace del kit.",
                            clave="disco"))

    return puntos


# ── instalación ────────────────────────────────────────────────────────────


def instalar(puntos):
    """Instala todo lo que falte: ffmpeg dentro del kit y los paquetes de Python."""
    estado = {p.clave: p.estado for p in puntos}
    if estado.get("ffmpeg") != "ok" or estado.get("codificadores") == "falta":
        import instalar_ffmpeg
        try:
            instalar_ffmpeg.asegurar(forzar=estado.get("codificadores") == "falta")
        except c.FaltaDependencia as e:
            print("\n{0}\n".format(e))
            return False
        print()

    paquetes = {"pillow": "Pillow", "whisper": "faster-whisper",
                "playwright": "playwright"}
    pendientes = [paquetes[p.clave] for p in puntos
                  if p.clave in paquetes and p.estado != "ok"]
    hecho = []
    if pendientes:
        print("Instalando: {0}…".format(", ".join(pendientes)))
        print("(la primera vez tarda unos minutos)")
        cmd = [sys.executable, "-m", "pip", "install", "--user", "--upgrade"] \
            + pendientes
        proc = subprocess.run(cmd)
        if proc.returncode != 0:
            # Algunos Python del sistema no dejan --user; sin él suele ir.
            print("\nReintento sin --user…")
            proc = subprocess.run([sys.executable, "-m", "pip", "install",
                                   "--upgrade"] + pendientes)
        if proc.returncode != 0:
            print("\nNo he podido instalar los paquetes de Python. "
                  "Cópiame el error de arriba.")
            return False
        hecho += pendientes

    if "playwright" in [p.lower() for p in hecho] or _version_paquete("playwright"):
        print("\nDescargando el Chrome de los visuales…")
        proc = subprocess.run([sys.executable, "-m", "playwright", "install",
                               "chromium"])
        if proc.returncode != 0:
            print("Chrome no se ha descargado. Los visuales no funcionarán, "
                  "pero el resto del kit sí.")
        else:
            hecho.append("Chrome")

    if not hecho:
        print("No había nada que instalar desde Python.")
    return True


# ── salida ─────────────────────────────────────────────────────────────────

ICONO = {"ok": "✓", "aviso": "!", "falta": "✗"}


def imprimir(puntos):
    print("Revisión del kit  ·  {0} {1}\n".format(SISTEMA, platform.release()))
    ancho = max(len(p.nombre) for p in puntos)
    for p in puntos:
        print("  {0} {1}  {2}".format(ICONO[p.estado], p.nombre.ljust(ancho),
                                      p.detalle))
    faltan = [p for p in puntos if p.estado == "falta"]
    avisos = [p for p in puntos if p.estado == "aviso" and p.arreglo]

    if faltan:
        print("\nFalta esto para poder editar:")
        for p in faltan:
            print("  · {0}\n      {1}".format(p.nombre, p.arreglo))
    if avisos:
        print("\nOpcional, el kit funciona sin ello:")
        for p in avisos:
            print("  · {0}\n      {1}".format(p.nombre, p.arreglo))
    if not faltan:
        print("\n✓ Todo listo. El kit puede editar vídeo en este ordenador.")
        if avisos:
            print("  (sin los visuales solo faltarán rótulos y mockups)")
    return 0 if not faltan else 1


def main():
    ap = argparse.ArgumentParser(description="Revisa el kit")
    ap.add_argument("--json", action="store_true", dest="como_json")
    ap.add_argument("--instalar", action="store_true",
                    help="instalar los paquetes de Python que falten")
    a = ap.parse_args()

    puntos = revisar()
    if a.instalar:
        instalar(puntos)
        print("\nVuelvo a revisar…\n")
        c._cache_binarios.clear()
        puntos = revisar()

    if a.como_json:
        print(json.dumps({
            "sistema": SISTEMA,
            "listo": all(p.estado != "falta" for p in puntos),
            "visuales": next((p.estado == "ok" for p in puntos
                              if p.clave == "playwright"), False),
            "puntos": [p.a_dict() for p in puntos],
        }, indent=2, ensure_ascii=False))
        return 0
    return imprimir(puntos)


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Saca fotogramas con una regla encima para elegir el recorte vertical.

Uso:  python scripts/encuadre.py <carpeta-del-proyecto>

Un vídeo horizontal no cabe en vertical: hay que decidir qué franja se queda.
Este paso no lo adivina, lo enseña. Genera tres fotogramas con marcas cada
10 % y el rectángulo de recorte dibujado, para mirarlos y decir dónde está
la cara. Después se guarda esa posición con:

  python scripts/encuadre.py <proyecto> --centro 0.38
"""
from __future__ import annotations

import argparse
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

PROPORCION = c.ANCHO / c.ALTO   # 0.5625


def medidas_recorte(ancho, alto, centro=0.5):
    """Rectángulo 9:16 dentro del vídeo original. Devuelve (x, y, w, h)."""
    ancho_recorte = int(round(alto * PROPORCION / 2) * 2)
    if ancho_recorte <= ancho:
        x = int(round(ancho * centro - ancho_recorte / 2))
        x = max(0, min(x, ancho - ancho_recorte))
        return x, 0, ancho_recorte, alto
    # El vídeo ya es más estrecho que 9:16: se recorta por arriba y abajo.
    alto_recorte = int(round(ancho / PROPORCION / 2) * 2)
    alto_recorte = min(alto_recorte, alto)
    # Se deja más aire arriba que abajo: las caras quedan mejor en el tercio alto.
    y = int(round((alto - alto_recorte) * 0.35))
    return 0, max(0, y), ancho, alto_recorte


def dibujar_regla(imagen, ancho, alto, centro):
    from PIL import ImageDraw

    d = ImageDraw.Draw(imagen, "RGBA")
    tipo = c.fuente("mono", max(18, alto // 34), 700)
    for pct in range(0, 101, 10):
        x = int(ancho * pct / 100)
        x = min(max(x, 1), ancho - 2)
        d.line([(x, 0), (x, alto)], fill=(255, 255, 255, 90), width=2)
        etiqueta = "{0}".format(pct)
        d.rectangle([x + 4, 6, x + 14 + 22 * len(etiqueta), 12 + alto // 30],
                    fill=(0, 0, 0, 150))
        d.text((x + 9, 9), etiqueta, font=tipo, fill=(255, 255, 255, 230))
    rx, ry, rw, rh = medidas_recorte(ancho, alto, centro)
    d.rectangle([rx, ry, rx + rw - 1, ry + rh - 1],
                outline=(255, 122, 0, 255), width=6)
    # El rótulo va abajo: arriba taparía los números de la regla.
    banda = int(alto // 18)
    d.rectangle([rx, ry + rh - banda, rx + rw - 1, ry + rh - 1],
                fill=(255, 122, 0, 230))
    d.text((rx + 14, ry + rh - banda + 6),
           "recorte centro={0:.2f}".format(centro),
           font=tipo, fill=(0, 0, 0, 255))
    return imagen


def previsualizar(proyecto):
    from PIL import Image

    estado = c.cargar_proyecto(proyecto)
    origen = c.ruta_origen(estado)
    info = estado["origen_info"]
    total = float(info["duracion"])
    centro = float(estado.get("encuadre", {}).get("centro", 0.5))
    carpeta = c.trabajo(estado, "encuadre")
    carpeta.mkdir(parents=True, exist_ok=True)

    generados = []
    for etiqueta, fraccion in (("inicio", 0.15), ("medio", 0.5), ("final", 0.85)):
        crudo = carpeta / "crudo-{0}.png".format(etiqueta)
        c.correr_ffmpeg([
            "-ss", "{0:.3f}".format(total * fraccion), "-i", str(origen),
            "-frames:v", "1", "-q:v", "2", str(crudo),
        ])
        img = Image.open(crudo).convert("RGB")
        dibujar_regla(img, img.width, img.height, centro)
        destino = carpeta / "guia-{0}.png".format(etiqueta)
        img.save(destino)
        crudo.unlink(missing_ok=True)
        generados.append(destino)

    print("Fotogramas con regla en trabajo/encuadre/:")
    for g in generados:
        print("  {0}".format(g.name))
    print("\nMira dónde queda la cara en la regla (0 = izquierda, 100 = derecha)")
    print("y fija el centro del recorte, por ejemplo al 38 %:")
    print("  python scripts/encuadre.py \"{0}\" --centro 0.38".format(
        estado["_carpeta"]))
    return generados


def fijar(proyecto, centro, modo):
    estado = c.cargar_proyecto(proyecto)
    if not 0.0 <= centro <= 1.0:
        raise RuntimeError("El centro va de 0.0 a 1.0 (0.5 es el medio).")
    info = estado["origen_info"]
    estado["encuadre"] = {"modo": modo, "centro": round(centro, 3)}
    c.guardar_proyecto(estado)
    x, y, w, h = medidas_recorte(info["ancho"], info["alto"], centro)
    print("Encuadre guardado: modo {0}, centro {1:.2f}".format(modo, centro))
    if modo == "recorte":
        print("  recorta {0}x{1} desde x={2} y={3} y lo escala a {4}x{5}".format(
            w, h, x, y, c.ANCHO, c.ALTO))
    else:
        print("  el vídeo entero cabe dentro, con fondo desenfocado detrás")
    print("\nSiguiente paso: aplicar los cortes")
    print("  python scripts/cortar.py \"{0}\"".format(estado["_carpeta"]))
    return estado


def main():
    ap = argparse.ArgumentParser(description="Elige el recorte vertical")
    ap.add_argument("proyecto")
    ap.add_argument("--centro", type=float, default=None,
                    help="0.0 izquierda, 0.5 centro, 1.0 derecha")
    ap.add_argument("--modo", choices=["recorte", "marco"], default="recorte",
                    help="recorte = llena la pantalla; marco = entero con fondo")
    a = ap.parse_args()
    try:
        if a.centro is None:
            previsualizar(a.proyecto)
        else:
            fijar(a.proyecto, a.centro, a.modo)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Instala ffmpeg dentro del propio kit, sin tocar nada del ordenador.

Uso:
  python scripts/instalar_ffmpeg.py            instalar si falta
  python scripts/instalar_ffmpeg.py --forzar   volver a descargar
  python scripts/instalar_ffmpeg.py --json     resultado para que lo lea Claude

ffmpeg es el programa que corta, recomprime y mezcla el vídeo: sin él el kit no
puede hacer nada. Lo normal sería instalarlo en el sistema (con Homebrew en Mac o
winget en Windows), pero eso pide contraseña de administrador o un gestor de
paquetes que puede no estar. Así que aquí se hace de otra forma: se descarga una
copia suelta de ffmpeg y ffprobe y se deja en la carpeta `bin/` del kit.

Ventajas de hacerlo así:
  · no pide contraseña ni permisos de administrador
  · no cambia nada fuera de la carpeta del kit (se borra borrando `bin/`)
  · funciona igual en Mac (Intel y Apple) y en Windows
  · no depende del PATH: el kit mira su propio `bin/` antes que nada

Son unos 20 MB comprimidos por programa, 40 MB en total. Se descargan una vez.
"""
from __future__ import annotations

import argparse
import gzip
import json
import platform
import shutil
import stat
import sys
import time
import urllib.error
import urllib.request
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

# Compilaciones sueltas de ffmpeg, un archivo por programa y por procesador.
# Se fija una versión concreta a propósito: "la última" cambia sin avisar y un
# día dejaría de funcionar sin que nadie hubiera tocado el kit.
VERSION = "b6.1.1"
BASE = ("https://github.com/eugeneware/ffmpeg-static/releases/download/"
        + VERSION + "/")

# (sistema, procesador) → sufijo del archivo que hay que descargar
PLATAFORMAS = {
    ("Darwin", "arm64"): "darwin-arm64",     # Mac con chip Apple (M1 y siguientes)
    ("Darwin", "x86_64"): "darwin-x64",      # Mac con Intel
    ("Windows", "AMD64"): "win32-x64",
    ("Windows", "x86_64"): "win32-x64",
    ("Linux", "x86_64"): "linux-x64",
    ("Linux", "aarch64"): "linux-arm64",
}

# Si la descarga no es posible, esto es lo que hay que decirle al usuario.
ALTERNATIVA = {
    "Darwin": ("Instalar ffmpeg en el sistema con Homebrew:\n"
               "  brew install ffmpeg\n"
               "(si no tienes Homebrew, en brew.sh está el instalador; pide la "
               "contraseña del ordenador)"),
    "Windows": ("Instalar ffmpeg en el sistema con winget, que ya viene en "
                "Windows:\n"
                "  winget install --id Gyan.FFmpeg -e "
                "--accept-source-agreements"),
    "Linux": "sudo apt install -y ffmpeg",
}


def _plataforma():
    sistema = platform.system()
    maquina = platform.machine()
    sufijo = PLATAFORMAS.get((sistema, maquina))
    if sufijo:
        return sufijo
    # Windows de 32 bits o un procesador raro: no hay compilación preparada.
    raise c.FaltaDependencia(
        "No tengo una copia de ffmpeg preparada para este ordenador "
        "({0} {1}).\n{2}".format(sistema, maquina,
                                 ALTERNATIVA.get(sistema, "Instala ffmpeg.")))


def _barra(hechos, total, ancho=28):
    if not total:
        return "{0:.1f} MB".format(hechos / 1e6)
    lleno = int(ancho * hechos / total)
    return "[{0}{1}] {2:.0f}%".format("#" * lleno, "·" * (ancho - lleno),
                                      100 * hechos / total)


def descargar(url, destino, etiqueta):
    """Descarga un .gz y lo deja descomprimido y ejecutable en destino."""
    peticion = urllib.request.Request(
        url, headers={"User-Agent": "kit-editor-video"})
    trozos, hechos, ultimo = [], 0, 0.0
    with urllib.request.urlopen(peticion, timeout=60) as respuesta:
        total = int(respuesta.headers.get("Content-Length") or 0)
        while True:
            trozo = respuesta.read(262144)
            if not trozo:
                break
            trozos.append(trozo)
            hechos += len(trozo)
            ahora = time.time()
            if ahora - ultimo > 0.6:
                ultimo = ahora
                sys.stdout.write("\r  {0}  {1}".format(
                    etiqueta, _barra(hechos, total)))
                sys.stdout.flush()
    sys.stdout.write("\r  {0}  {1}\n".format(etiqueta, _barra(hechos, hechos)))

    crudo = gzip.decompress(b"".join(trozos))
    # Se escribe con otro nombre y se renombra al final: si la descarga se corta
    # a medias no queda un ffmpeg roto ocupando el sitio del bueno.
    temporal = destino.with_suffix(destino.suffix + ".parcial")
    temporal.write_bytes(crudo)
    temporal.chmod(temporal.stat().st_mode | stat.S_IXUSR | stat.S_IXGRP
                   | stat.S_IXOTH)
    if destino.exists():
        destino.unlink()
    temporal.rename(destino)
    return len(crudo)


def asegurar(forzar=False, silencioso=False):
    """Deja ffmpeg y ffprobe funcionando. Devuelve un resumen de lo que hizo."""
    def decir(*a):
        if not silencioso:
            print(*a)

    c._cache_binarios.clear()
    if not forzar:
        ya = {n: c.buscar_binario(n) for n in ("ffmpeg", "ffprobe")}
        if all(ya.values()):
            decir("ffmpeg ya funciona en este ordenador:")
            decir("  " + ya["ffmpeg"])
            return {"instalado": False, "motivo": "ya estaba",
                    "ffmpeg": ya["ffmpeg"], "ffprobe": ya["ffprobe"]}

    sufijo = _plataforma()
    extension = ".exe" if c.ES_WINDOWS else ""
    c.BIN.mkdir(parents=True, exist_ok=True)

    decir("Descargando ffmpeg para {0}. Son unos 40 MB, una sola vez.".format(
        sufijo))
    descargados = {}
    for nombre in ("ffmpeg", "ffprobe"):
        destino = c.BIN / (nombre + extension)
        try:
            tamano = descargar(BASE + nombre + "-" + sufijo + ".gz", destino,
                               nombre)
        except (urllib.error.URLError, OSError, EOFError) as e:
            raise c.FaltaDependencia(
                "No he podido descargar {0}.\n"
                "Detalle: {1}\n\n"
                "Puede ser la conexión, o una red de empresa que bloquea "
                "GitHub. Otra vía:\n{2}".format(
                    nombre, e, ALTERNATIVA.get(platform.system(), "")))
        descargados[nombre] = (destino, tamano)

    # Existir no es funcionar: se comprueba que arranca de verdad antes de
    # decir que está instalado.
    c._cache_binarios.clear()
    resultado = {}
    for nombre, (destino, tamano) in descargados.items():
        if not c._arranca(destino):
            raise c.FaltaDependencia(
                "He descargado {0} pero este ordenador no lo deja arrancar.\n"
                "Prueba la otra vía:\n{1}".format(
                    nombre, ALTERNATIVA.get(platform.system(), "")))
        resultado[nombre] = str(destino)
        decir("  ✓ {0} listo ({1:.0f} MB)".format(nombre, tamano / 1e6))

    version = "?"
    try:
        primera = c.correr([resultado["ffmpeg"], "-version"]).splitlines()[0]
        version = primera.split(" ")[2]
    except (RuntimeError, IndexError):
        pass

    # Sin estos dos codificadores no se puede crear vídeo ni sonido.
    faltan = []
    try:
        encoders = c.correr([resultado["ffmpeg"], "-hide_banner", "-encoders"])
        faltan = [x for x in ("libx264", "aac") if x not in encoders]
    except RuntimeError:
        pass
    if faltan:
        raise c.FaltaDependencia(
            "La copia descargada no trae {0}. Es raro; prueba la otra vía:\n"
            "{1}".format(" ni ".join(faltan),
                         ALTERNATIVA.get(platform.system(), "")))

    decir("\nffmpeg {0} instalado dentro del kit, en la carpeta bin/.".format(
        version))
    decir("No se ha cambiado nada del ordenador: borrando esa carpeta "
          "desaparece.")
    return {"instalado": True, "version": version, "plataforma": sufijo,
            "ffmpeg": resultado["ffmpeg"], "ffprobe": resultado["ffprobe"]}


def borrar():
    """Quita la copia del kit. Útil para probar el wizard desde cero."""
    if c.BIN.is_dir():
        shutil.rmtree(c.BIN)
        print("Borrada la carpeta bin/ del kit.")
    else:
        print("No había nada que borrar.")


def main():
    ap = argparse.ArgumentParser(description="Instala ffmpeg dentro del kit")
    ap.add_argument("--forzar", action="store_true",
                    help="descargar aunque ya haya un ffmpeg que funcione")
    ap.add_argument("--json", action="store_true", dest="como_json")
    ap.add_argument("--borrar", action="store_true",
                    help="quitar la copia del kit")
    a = ap.parse_args()

    if a.borrar:
        borrar()
        return 0
    try:
        resumen = asegurar(a.forzar, silencioso=a.como_json)
    except c.FaltaDependencia as e:
        if a.como_json:
            print(json.dumps({"ok": False, "error": str(e)},
                             indent=2, ensure_ascii=False))
        else:
            print("\n{0}".format(e), file=sys.stderr)
        return 1
    if a.como_json:
        resumen["ok"] = True
        print(json.dumps(resumen, indent=2, ensure_ascii=False))
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Crea un proyecto nuevo en workspace/ a partir de un vídeo en bruto.

Uso:  python scripts/nuevo.py "/ruta/al/video.mp4" [--nombre mi-video]

Deja workspace/<nombre>/proyecto.json con los datos del vídeo y avisa de lo
que pueda dar problemas más adelante (sin audio, muy corto, ya vertical…).
"""
from __future__ import annotations

import argparse
import sys
from datetime import date
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c


def crear(origen, nombre=None, forzar=False):
    origen = Path(origen).expanduser().resolve()
    if not origen.is_file():
        raise FileNotFoundError(
            "No encuentro el vídeo en {0}. Arrastra el archivo al chat para "
            "que salga la ruta correcta.".format(origen)
        )
    info = c.info_media(origen)
    if info["duracion"] < 1.0:
        raise RuntimeError(
            "Ese archivo dura {0:.1f}s o no es un vídeo que ffmpeg entienda."
            .format(info["duracion"])
        )

    nombre = c.slug(nombre or origen.stem)
    carpeta = c.WORKSPACE / nombre
    if carpeta.exists() and not forzar:
        n = 2
        while (c.WORKSPACE / "{0}-{1}".format(nombre, n)).exists():
            n += 1
        nombre = "{0}-{1}".format(nombre, n)
        carpeta = c.WORKSPACE / nombre
    (carpeta / "trabajo").mkdir(parents=True, exist_ok=True)

    estado = {
        "nombre": nombre,
        "origen": str(origen),
        "creado": date.today().isoformat(),
        "origen_info": info,
        "motor_transcripcion": "whisper",
        "modelo_whisper": "small",
        "idioma": "es",
        "pasos": {},
        "_carpeta": str(carpeta),
    }
    c.guardar_proyecto(estado)

    print("Proyecto «{0}» creado.".format(nombre))
    print("  vídeo    {0}".format(origen.name))
    print("  duración {0}   {1}x{2} a {3} fps".format(
        c.hhmmss(info["duracion"]), info["ancho"], info["alto"], info["fps"]))
    print("  carpeta  {0}".format(carpeta))

    avisos = []
    if not info["tiene_audio"]:
        avisos.append("El vídeo no tiene audio: no habrá subtítulos ni cortes "
                      "de silencio. Solo se podrá recortar a vertical.")
    if info["alto"] and info["ancho"] / info["alto"] < 1.0:
        avisos.append("Ya es vertical, así que no hace falta reencuadrar; "
                      "el kit igualmente ajusta a 1080x1920.")
    if info["duracion"] > 600:
        avisos.append("Dura más de 10 minutos: transcribir y renderizar puede "
                      "llevar bastante. Considera recortar antes la parte útil.")
    if info["fps"] and info["fps"] > 61:
        avisos.append("Grabado a {0} fps: el resultado sale a 60 fps."
                      .format(info["fps"]))
    for a in avisos:
        print("  ! {0}".format(a))

    print("\nSiguiente paso: transcribir el audio")
    print("  python scripts/transcribir.py \"{0}\"".format(carpeta))
    return estado


def main():
    ap = argparse.ArgumentParser(description="Crea un proyecto de edición")
    ap.add_argument("video", help="ruta del vídeo en bruto")
    ap.add_argument("--nombre", default=None, help="nombre de la carpeta")
    ap.add_argument("--forzar", action="store_true",
                    help="reutilizar la carpeta si ya existe")
    a = ap.parse_args()
    try:
        crear(a.video, a.nombre, a.forzar)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Decide qué trozos del vídeo se quitan: silencios, muletillas y repeticiones.

Uso:  python scripts/plan_cortes.py <carpeta-del-proyecto> [opciones]

Cómo lo decide, en orden:
  1. Huecos entre palabras de la transcripción   → dónde puede haber silencio
  2. `silencedetect` de ffmpeg                   → dónde el silencio empieza y
     acaba de verdad, para no comerse el final de la palabra ni la respiración
  3. Muletillas ("eh", "em") con poca confianza
  4. Repeticiones: si dices una frase dos veces, se queda la última

Escribe trabajo/plan-cortes.json. Nada se toca del vídeo todavía: este paso
solo propone, y se puede revisar y editar a mano antes de cortar.
"""
from __future__ import annotations

import argparse
import re
import sys
import unicodedata
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

# Valores por defecto, pensados para hablar a cámara sin guion.
UMBRAL_SILENCIO = 0.55   # hueco a partir del cual se corta
PAUSA_MINIMA = 0.30      # huecos más cortos que esto no se tocan
PAUSA_OBJETIVO = 0.16    # a esto se comprimen las pausas medianas
PAD_ANTES = 0.10         # aire que se deja antes de la palabra siguiente
PAD_DESPUES = 0.12       # aire que se deja tras la palabra anterior
MINIMO_UTIL = 0.20       # un trozo más corto que esto no vale como plano
RUIDO_DB = -32           # por debajo de esto, ffmpeg lo considera silencio

# Muletillas que se quitan siempre que suenen dudosas.
MULETILLAS = {"eh", "ehm", "em", "emm", "mm", "mmm", "hm", "hmm", "ah", "uh",
              "aah", "ehh", "mhm"}
# Palabras reales que solo son muletilla si la transcripción no las tiene claras.
MULETILLAS_DUDOSAS = {"e", "a", "o", "y", "pues", "este", "esto", "bueno"}
PROB_MULETILLA = 0.40


def normaliza(texto):
    """Minúsculas, sin acentos y sin puntuación, para poder comparar palabras."""
    plano = unicodedata.normalize("NFD", texto.lower())
    plano = "".join(ch for ch in plano if unicodedata.category(ch) != "Mn")
    return re.sub(r"[^a-z0-9ñ]+", "", plano.replace("ñ", "ñ"))


# ── señal 1 y 2: dónde hay silencio de verdad ──────────────────────────────


def detectar_silencios(audio, ruido=RUIDO_DB, minimo=0.22):
    """Silencios reales del audio, con ffmpeg. Devuelve [(inicio, fin)]."""
    proc = c.correr([
        c.ffmpeg(), "-nostdin", "-i", str(audio), "-af",
        "silencedetect=noise={0}dB:d={1}".format(ruido, minimo),
        "-f", "null", "-",
    ])
    return _leer_silencedetect(proc)


def _leer_silencedetect(texto):
    inicios = [float(x) for x in re.findall(r"silence_start: (-?[\d.]+)", texto)]
    finales = [float(x) for x in re.findall(r"silence_end: (-?[\d.]+)", texto)]
    return [(max(a, 0.0), b) for a, b in zip(inicios, finales) if b > a]


def _silencio_dentro(hueco, silencios):
    """El silencio detectado que más ocupa el hueco entre dos palabras."""
    a, b = hueco
    mejor, solape_mejor = None, 0.0
    for s_a, s_b in silencios:
        solape = min(b, s_b) - max(a, s_a)
        if solape > solape_mejor:
            mejor, solape_mejor = (s_a, s_b), solape
    return mejor, solape_mejor


# ── señal 4: repeticiones (te has repetido, se queda la última) ─────────────


def buscar_repeticiones(palabras, ventana_maxima=6, huecos_permitidos=2,
                        separacion_maxima=6.0):
    """Encuentra tomas repetidas y devuelve [(desde_idx, hasta_idx, frase)].

    Si la frase «el primero es» aparece y vuelve a aparecer poco después, lo
    de en medio es un tropiezo: se marca para quitar desde la primera hasta
    la segunda, que es la buena.
    """
    tokens = [normaliza(p["texto"]) for p in palabras]
    fuera, i = [], 0
    while i < len(tokens):
        encontrado = None
        # Se prueba primero la frase más larga: si repetiste cinco palabras,
        # queremos quitar las cinco, no solo las dos primeras.
        for n in range(ventana_maxima, 1, -1):
            if i + n > len(tokens):
                continue
            base = tokens[i:i + n]
            if not all(base):
                continue
            for salto in range(0, huecos_permitidos + 1):
                j = i + n + salto
                if j + n > len(tokens):
                    break
                if tokens[j:j + n] != base:
                    continue
                if palabras[j]["inicio"] - palabras[i]["inicio"] > separacion_maxima:
                    continue
                encontrado = (i, j, " ".join(p["texto"] for p in palabras[i:j]))
                break
            if encontrado:
                break
        if encontrado:
            fuera.append(encontrado)
            i = encontrado[1]
        else:
            i += 1
    return fuera


# ── el plan ────────────────────────────────────────────────────────────────


def _fusionar(intervalos):
    """Une los trozos a quitar que se solapan o se tocan."""
    if not intervalos:
        return []
    orden = sorted(intervalos, key=lambda x: x["inicio"])
    salida = [dict(orden[0])]
    for actual in orden[1:]:
        ultimo = salida[-1]
        if actual["inicio"] <= ultimo["fin"] + 0.02:
            if actual["fin"] > ultimo["fin"]:
                ultimo["fin"] = actual["fin"]
            if actual["motivo"] not in ultimo["motivo"]:
                ultimo["motivo"] = "{0} + {1}".format(ultimo["motivo"], actual["motivo"])
        else:
            salida.append(dict(actual))
    return salida


def construir(palabras, total, silencios, opciones):
    """Devuelve la lista de trozos a quitar, ya ordenada y sin solapes."""
    quitar = []
    u = opciones

    if not palabras:
        # Sin voz solo se puede recortar el silencio de los extremos.
        for a, b in silencios:
            if b - a >= u["umbral"]:
                quitar.append({"inicio": a, "fin": b,
                               "motivo": "silencio {0:.2f}s".format(b - a)})
        return _fusionar(quitar)

    # Repeticiones primero: marcan las palabras que ya no cuentan para nada más.
    repetidas = set()
    if u["tomas"]:
        for desde, hasta, frase in buscar_repeticiones(palabras):
            ini = palabras[desde]["inicio"] - 0.08
            if desde > 0:
                ini = max(ini, palabras[desde - 1]["fin"] + u["pad_despues"])
            fin = max(palabras[hasta]["inicio"] - u["pad_antes"], ini + 0.05)
            quitar.append({
                "inicio": max(ini, 0.0), "fin": fin,
                "motivo": "toma repetida «{0}»".format(frase[:48]),
            })
            repetidas.update(range(desde, hasta))

    # Muletillas sueltas.
    if u["muletillas"]:
        for idx, p in enumerate(palabras):
            if idx in repetidas:
                continue
            token = normaliza(p["texto"])
            dudosa = token in MULETILLAS_DUDOSAS and p["prob"] < PROB_MULETILLA
            if token in MULETILLAS or dudosa:
                ini = p["inicio"] - 0.05
                if idx > 0:
                    ini = max(ini, palabras[idx - 1]["fin"] + u["pad_despues"] * 0.5)
                fin = p["fin"] + 0.05
                if idx + 1 < len(palabras):
                    fin = min(fin, palabras[idx + 1]["inicio"] - u["pad_antes"] * 0.5)
                if fin > ini:
                    quitar.append({"inicio": max(ini, 0.0), "fin": fin,
                                   "motivo": "muletilla «{0}»".format(p["texto"])})

    # Silencio de la cabeza y de la cola.
    primera, ultima = palabras[0], palabras[-1]
    if primera["inicio"] > u["umbral"]:
        quitar.append({
            "inicio": 0.0,
            "fin": max(primera["inicio"] - u["pad_antes"], 0.0),
            "motivo": "silencio inicial {0:.2f}s".format(primera["inicio"]),
        })
    if total - ultima["fin"] > u["umbral"]:
        quitar.append({
            "inicio": min(ultima["fin"] + u["pad_despues"], total),
            "fin": total,
            "motivo": "silencio final {0:.2f}s".format(total - ultima["fin"]),
        })

    # Huecos entre palabras.
    for anterior, siguiente in zip(palabras, palabras[1:]):
        hueco = siguiente["inicio"] - anterior["fin"]
        if hueco < u["pausa_minima"]:
            continue
        ini_bruto = anterior["fin"] + u["pad_despues"]
        fin_bruto = siguiente["inicio"] - u["pad_antes"]

        if hueco >= u["umbral"]:
            # Ajustamos al silencio real para no cortar la respiración ni la
            # cola de la palabra: el oído nota mucho más un corte a destiempo.
            sil, solape = _silencio_dentro((anterior["fin"], siguiente["inicio"]),
                                           silencios)
            if sil and solape > hueco * 0.35:
                ini_bruto = max(ini_bruto, sil[0])
                fin_bruto = min(fin_bruto, sil[1])
            if fin_bruto - ini_bruto > 0.05:
                quitar.append({
                    "inicio": ini_bruto, "fin": fin_bruto,
                    "motivo": "silencio {0:.2f}s".format(hueco),
                })
        elif u["micropausas"]:
            # Pausa mediana: no se quita entera, se acorta. Da ritmo sin que
            # parezca que hablas atropellado.
            sobra = hueco - u["pausa_objetivo"]
            if sobra > 0.08:
                centro = (anterior["fin"] + siguiente["inicio"]) / 2.0
                quitar.append({
                    "inicio": centro - sobra / 2.0, "fin": centro + sobra / 2.0,
                    "motivo": "pausa acortada {0:.2f}s".format(hueco),
                })

    return _fusionar(quitar)


def a_segmentos(quitar, total):
    """Pasa de «lo que se quita» a la lista completa de trozos con su acción."""
    segmentos, cursor = [], 0.0
    for hueco in quitar:
        ini = max(min(hueco["inicio"], total), 0.0)
        fin = max(min(hueco["fin"], total), 0.0)
        if fin <= cursor:
            continue
        ini = max(ini, cursor)
        if ini - cursor >= MINIMO_UTIL:
            segmentos.append({"inicio": round(cursor, 3), "fin": round(ini, 3),
                              "accion": "mantener"})
        elif ini > cursor and segmentos and segmentos[-1]["accion"] == "quitar":
            # Trozo demasiado corto entre dos cortes: no es un plano, es un
            # parpadeo. Se va con el corte anterior.
            segmentos[-1]["fin"] = round(ini, 3)
        elif ini > cursor:
            segmentos.append({"inicio": round(cursor, 3), "fin": round(ini, 3),
                              "accion": "mantener"})
        segmentos.append({"inicio": round(ini, 3), "fin": round(fin, 3),
                          "accion": "quitar", "motivo": hueco["motivo"]})
        cursor = fin
    if total - cursor > 0.01:
        segmentos.append({"inicio": round(cursor, 3), "fin": round(total, 3),
                          "accion": "mantener"})
    return [s for s in segmentos if s["fin"] - s["inicio"] > 0.005]


def mapa_tiempos(segmentos):
    """[(inicio_original, fin_original, desplazamiento)] de los trozos que se quedan."""
    mapa, offset = [], 0.0
    for s in segmentos:
        if s["accion"] != "mantener":
            continue
        mapa.append([s["inicio"], s["fin"], round(offset, 3)])
        offset += s["fin"] - s["inicio"]
    return mapa, round(offset, 3)


def planificar(proyecto, **kw):
    estado = c.cargar_proyecto(proyecto)
    total = float(estado["origen_info"]["duracion"])
    trans = c.leer_json(c.trabajo(estado, "transcripcion.json"),
                        {"palabras": [], "frases": []})
    palabras = trans.get("palabras") or []

    audio = c.trabajo(estado, "voz.wav")
    silencios = detectar_silencios(audio) if audio.is_file() else []

    opciones = {
        "umbral": kw.get("umbral") or UMBRAL_SILENCIO,
        "pausa_minima": kw.get("pausa_minima") or PAUSA_MINIMA,
        "pausa_objetivo": kw.get("pausa_objetivo") or PAUSA_OBJETIVO,
        "pad_antes": PAD_ANTES,
        "pad_despues": PAD_DESPUES,
        "tomas": kw.get("tomas", True),
        "muletillas": kw.get("muletillas", True),
        "micropausas": kw.get("micropausas", True),
    }

    quitar = construir(palabras, total, silencios, opciones)
    segmentos = a_segmentos(quitar, total)
    mapa, final = mapa_tiempos(segmentos)

    plan = {
        "duracion_origen": round(total, 3),
        "duracion_final": final,
        "opciones": opciones,
        "silencios_detectados": len(silencios),
        "segmentos": segmentos,
        "mapa": mapa,
    }
    salida = c.escribir_json(c.trabajo(estado, "plan-cortes.json"), plan)
    estado.setdefault("pasos", {})["plan_cortes"] = "ok"
    c.guardar_proyecto(estado)

    quitados = [s for s in segmentos if s["accion"] == "quitar"]
    print("Plan de cortes: {0} → {1} ({2} cortes, -{3:.0f}%)".format(
        c.hhmmss(total), c.hhmmss(final), len(quitados),
        100 * (1 - final / total) if total else 0))
    for s in quitados:
        print("  quitar {0} → {1}  ({2:.2f}s)  {3}".format(
            c.hhmmss(s["inicio"]), c.hhmmss(s["fin"]),
            s["fin"] - s["inicio"], s.get("motivo", "")))
    print("  → {0}".format(salida))
    print("\nSiguiente paso: aplicar los cortes y pasar a vertical")
    print("  python scripts/cortar.py \"{0}\"".format(estado["_carpeta"]))
    return plan


def main():
    ap = argparse.ArgumentParser(description="Propone los cortes del vídeo")
    ap.add_argument("proyecto")
    ap.add_argument("--umbral", type=float, default=None,
                    help="silencio mínimo que se corta (por defecto 0.55s)")
    ap.add_argument("--sin-tomas", action="store_true",
                    help="no quitar repeticiones")
    ap.add_argument("--sin-muletillas", action="store_true")
    ap.add_argument("--sin-micropausas", action="store_true",
                    help="dejar las pausas cortas tal cual")
    a = ap.parse_args()
    try:
        planificar(a.proyecto, umbral=a.umbral, tomas=not a.sin_tomas,
                   muletillas=not a.sin_muletillas,
                   micropausas=not a.sin_micropausas)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Prueba de instalación: fabrica un vídeo y lo edita de principio a fin.

Uso:
  python scripts/prueba.py            hacer la prueba
  python scripts/prueba.py --json     resultado para que lo lea Claude
  python scripts/prueba.py --limpiar  borrar el proyecto de prueba

Esto es lo que decide si el kit está bien instalado. No comprueba que los
programas existan (eso lo hace doctor.py): comprueba que **hacen su trabajo**.
Fabrica un vídeo horizontal con voz de verdad y silencios metidos a propósito,
y lo pasa por toda la cadena: transcribir, decidir los cortes, pasar a vertical,
dibujar los subtítulos, montar y mezclar el sonido.

La voz la pone el propio ordenador: Mac y Windows traen un lector de textos de
serie. Si este ordenador no tiene ninguno, la prueba sigue con tonos y avisa de
que la transcripción se ha quedado sin comprobar.

Al final dice qué ha salido bien y qué no, con el vídeo terminado para verlo.
"""
from __future__ import annotations

import argparse
import json
import platform
import shutil
import subprocess
import sys
import time
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

# El nombre pasa por slug() como cualquier proyecto: así la carpeta que se crea
# es exactamente la que este script espera encontrar después.
NOMBRE = c.slug("prueba-del-kit")
MATERIAL = NOMBRE + "-material"

# Tres frases con pausas largas entre ellas. La segunda lleva un tropiezo a
# propósito (dice dos veces lo mismo seguido, como cuando repites una toma):
# así la prueba también comprueba que el kit se queda con la segunda.
FRASES = [
    "Esto es la prueba de instalación del kit de edición.",
    "El vídeo se corta, el vídeo se corta solo.",
    "Si ves este vídeo con subtítulos, está todo listo.",
]
SILENCIO = 1.30      # segundos de silencio entre frases: el kit debe cortarlos
ANCHO_CRUDO, ALTO_CRUDO = 1920, 1080
FPS_CRUDO = 30

# Lectores de textos que vienen de serie en cada sistema, con una voz española.
VOCES_MAC = ["Monica", "Mónica", "Paulina", "Jorge", "Diego", "Juan", "Marisol"]


class Fallo(RuntimeError):
    """La prueba no ha podido terminar. El mensaje explica dónde se ha roto."""


# ── fabricar el vídeo de prueba ────────────────────────────────────────────


def _voz_mac(texto, destino):
    voces = c.correr(["say", "-v", "?"]) if shutil.which("say") else ""
    elegida = next((v for v in VOCES_MAC if v in voces), None)
    if not elegida:
        # Sin voz española, con la del sistema: whisper detectará el idioma.
        elegida = None
    cmd = ["say"]
    if elegida:
        cmd += ["-v", elegida]
    aiff = destino.with_suffix(".aiff")
    c.correr(cmd + ["-o", str(aiff), texto])
    return aiff, ("es" if elegida else None)


def _voz_windows(texto, destino):
    # System.Speech viene con Windows; escribe directamente un WAV.
    guion = (
        "Add-Type -AssemblyName System.Speech; "
        "$s = New-Object System.Speech.Synthesis.SpeechSynthesizer; "
        "$es = $s.GetInstalledVoices() | Where-Object "
        "{ $_.VoiceInfo.Culture.TwoLetterISOLanguageName -eq 'es' } | "
        "Select-Object -First 1; "
        "if ($es) { $s.SelectVoice($es.VoiceInfo.Name) }; "
        "$s.SetOutputToWaveFile('" + str(destino) + "'); "
        "$s.Speak('" + texto.replace("'", "''") + "'); "
        "$s.Dispose(); "
        "if ($es) { 'es' } else { 'otro' }"
    )
    salida = c.correr(["powershell", "-NoProfile", "-Command", guion])
    return destino, ("es" if "es" in salida else None)


def _voz_linux(texto, destino):
    if not shutil.which("espeak-ng") and not shutil.which("espeak"):
        raise Fallo("sin lector de textos")
    programa = shutil.which("espeak-ng") or shutil.which("espeak")
    c.correr([programa, "-v", "es", "-w", str(destino), texto])
    return destino, "es"


def hablar(texto, destino):
    """Convierte un texto en un archivo de audio. Devuelve (archivo, idioma)."""
    sistema = platform.system()
    try:
        if sistema == "Darwin":
            return _voz_mac(texto, destino)
        if sistema == "Windows":
            return _voz_windows(texto, destino)
        return _voz_linux(texto, destino)
    except (RuntimeError, OSError) as e:
        raise Fallo(str(e))


def fabricar_video(carpeta):
    """Deja en carpeta/crudo.mp4 un vídeo horizontal con voz y silencios.

    Devuelve (ruta, idioma o None). Si el idioma es None, el audio no lleva voz
    reconocible y la transcripción no se puede dar por comprobada.
    """
    carpeta.mkdir(parents=True, exist_ok=True)
    piezas, idioma, con_voz = [], None, True

    for i, frase in enumerate(FRASES):
        crudo = carpeta / "voz{0}.raw".format(i)
        try:
            archivo, idioma_voz = hablar(frase, crudo.with_suffix(".wav"))
            idioma = idioma or idioma_voz
        except Fallo:
            con_voz = False
            break
        # Todo al mismo formato: el pegado exige que las piezas sean iguales.
        limpio = carpeta / "voz{0}-ok.wav".format(i)
        c.correr_ffmpeg(["-i", str(archivo), "-ac", "2", "-ar", "48000",
                         "-c:a", "pcm_s16le", str(limpio)])
        piezas.append(limpio)

    silencio = carpeta / "silencio.wav"
    c.correr_ffmpeg(["-f", "lavfi", "-i",
                     "anullsrc=r=48000:cl=stereo", "-t", str(SILENCIO),
                     "-c:a", "pcm_s16le", str(silencio)])

    if not con_voz:
        # Sin lector de textos: dos pitidos separados por silencio. Se comprueba
        # todo menos la transcripción.
        tono = carpeta / "tono.wav"
        c.correr_ffmpeg(["-f", "lavfi", "-i",
                         "sine=frequency=320:duration=1.6:sample_rate=48000",
                         "-ac", "2", "-c:a", "pcm_s16le", str(tono)])
        piezas = [tono, tono]
        idioma = None

    orden = [silencio]
    for pieza in piezas:
        orden += [pieza, silencio]

    lista = carpeta / "pegar.txt"
    lista.write_text(
        "".join("file '{0}'\n".format(p.as_posix()) for p in orden),
        encoding="utf-8")
    pista = carpeta / "pista.wav"
    c.correr_ffmpeg(["-f", "concat", "-safe", "0", "-i", str(lista),
                     "-c", "copy", str(pista)])

    # Imagen: el patrón de pruebas de ffmpeg, que se mueve. Al ser horizontal
    # obliga al kit a recortar a vertical de verdad.
    crudo = carpeta / "crudo.mp4"
    c.correr_ffmpeg([
        "-f", "lavfi", "-i", "testsrc2=size={0}x{1}:rate={2}".format(
            ANCHO_CRUDO, ALTO_CRUDO, FPS_CRUDO),
        "-i", str(pista), "-shortest",
        "-c:v", "libx264", "-preset", "veryfast", "-crf", "26",
        "-pix_fmt", "yuv420p", "-c:a", "aac", "-b:a", "128k",
        str(crudo)])

    for basura in list(carpeta.glob("voz*")) + [silencio, lista,
                                               carpeta / "tono.wav", pista]:
        if basura.exists():
            basura.unlink()
    return crudo, idioma


# ── correr la cadena ───────────────────────────────────────────────────────


def paso(titulo, cmd, avisos=None):
    """Ejecuta un paso del kit. Devuelve (ok, segundos, salida)."""
    t0 = time.time()
    proc = subprocess.run([sys.executable] + cmd, capture_output=True,
                          text=True, encoding="utf-8", errors="replace")
    tardado = time.time() - t0
    salida = (proc.stdout or "") + (proc.stderr or "")
    if proc.returncode != 0:
        cola = "\n".join(salida.strip().splitlines()[-14:])
        raise Fallo("{0}\n\n{1}".format(titulo, cola))
    print("  ✓ {0}  ({1:.1f} s)".format(titulo, tardado))
    return tardado, salida


def correr(carpeta_kit=None, con_visuales=True):
    inicio = time.time()
    carpeta = c.WORKSPACE / NOMBRE
    if carpeta.exists():
        shutil.rmtree(carpeta)
    temporal = c.WORKSPACE / MATERIAL
    if temporal.exists():
        shutil.rmtree(temporal)

    resultado = {"pasos": [], "avisos": [], "transcripcion_comprobada": False,
                 "visuales_comprobados": False}

    print("Prueba de instalación del kit\n")
    print("1 · Fabrico un vídeo de prueba con voz y silencios…")
    crudo, idioma = fabricar_video(temporal)
    info = c.info_media(crudo)
    print("  ✓ vídeo de prueba: {0}x{1}, {2:.1f} s{3}".format(
        info["ancho"], info["alto"], info["duracion"],
        "" if idioma else ", sin voz (este ordenador no tiene lector de textos)"))
    if not idioma:
        resultado["avisos"].append(
            "Este ordenador no tiene lector de textos, así que la prueba ha ido "
            "con tonos en vez de voz. Todo lo demás queda comprobado; la "
            "transcripción se comprobará con tu primer vídeo de verdad.")

    print("\n2 · Paso el vídeo por toda la cadena del kit…")
    ruta = str(carpeta)
    guion = str(Path(__file__).resolve().parent)

    def s(nombre):
        return str(Path(guion) / nombre)

    _, salida_nuevo = paso("proyecto creado",
                           [s("nuevo.py"), str(crudo), "--nombre", NOMBRE,
                            "--forzar"])
    orden = [("voz transcrita", [s("transcribir.py"), ruta]
              + (["--idioma", idioma] if idioma else [])),
             ("cortes decididos", [s("plan_cortes.py"), ruta]),
             ("vídeo en vertical", [s("cortar.py"), ruta]),
             ("subtítulos dibujados", [s("subtitulos.py"), ruta])]
    for titulo, cmd in orden:
        tardado, _ = paso(titulo, cmd)
        resultado["pasos"].append({"paso": titulo, "segundos": round(tardado, 1)})

    if con_visuales:
        # Un rótulo, para comprobar la cadena de los visuales completa.
        c.escribir_json(carpeta / "trabajo" / "visuales.json", {
            "visuales": [{"nombre": "prueba", "plantilla": "rotulo",
                          "inicio": 0.6,
                          "datos": {"texto": "Instalación", "sub": "correcta",
                                    "visible": 1.4}}]})
        try:
            paso("rótulo generado", [s("capturar.py"), ruta])
            resultado["visuales_comprobados"] = True
        except Fallo as e:
            print("  ! rótulos y mockups: no disponibles (es opcional)")
            resultado["avisos"].append(
                "Los visuales (rótulos y mockups) no funcionan en este "
                "ordenador. El resto del kit sí. Detalle: "
                + str(e).splitlines()[-1][:200])

    for titulo, cmd in [("montaje con subtítulos", [s("componer.py"), ruta]),
                        ("sonido mezclado", [s("sonido.py"), ruta])]:
        tardado, _ = paso(titulo, cmd)
        resultado["pasos"].append({"paso": titulo, "segundos": round(tardado, 1)})

    # ── comprobar el resultado, no solo que no ha dado error ──────────────
    print("\n3 · Reviso el vídeo terminado…")
    estado = c.cargar_proyecto(carpeta)
    final = Path(estado.get("salida") or "")
    if not final.is_file():
        raise Fallo("La cadena ha terminado sin errores pero no encuentro el "
                    "vídeo final.")

    info_final = c.info_media(final)
    problemas = []
    if (info_final["ancho"], info_final["alto"]) != (c.ANCHO, c.ALTO):
        problemas.append("el vídeo final mide {0}x{1} y debería medir {2}x{3}"
                         .format(info_final["ancho"], info_final["alto"],
                                 c.ANCHO, c.ALTO))
    if not info_final["tiene_audio"]:
        problemas.append("el vídeo final se ha quedado sin sonido")
    if info_final["duracion"] < 1.0:
        problemas.append("el vídeo final dura {0:.1f} s: algo se ha perdido"
                         .format(info_final["duracion"]))
    if info_final["duracion"] >= info["duracion"] - 0.5:
        problemas.append("no se ha recortado ningún silencio (el vídeo final "
                         "dura lo mismo que el original)")

    subs = c.leer_json(carpeta / "trabajo" / "subs.json", {})
    grupos = subs.get("grupos") or []
    if idioma:
        if not grupos:
            problemas.append("la transcripción no ha encontrado ninguna "
                             "palabra en un audio que sí lleva voz")
        else:
            resultado["transcripcion_comprobada"] = True

    # La frase con tropiezo debería haberse quedado en una sola toma. Esto no
    # se exige: depende de que la voz del ordenador se entienda bien. Si sale,
    # se dice, porque es la parte que más sorprende a quien prueba el kit.
    plan = c.leer_json(carpeta / "trabajo" / "plan-cortes.json", {})
    tomas = [s for s in (plan.get("segmentos") or [])
             if "toma repetida" in (s.get("motivo") or "")]
    resultado["toma_repetida_detectada"] = bool(tomas)

    print("  ✓ {0}x{1}, {2:.2f} s (el original duraba {3:.2f} s)".format(
        info_final["ancho"], info_final["alto"], info_final["duracion"],
        info["duracion"]))
    print("  ✓ sonido: {0}".format("sí" if info_final["tiene_audio"] else "no"))
    if grupos:
        muestra = " · ".join(" ".join(g.get("palabras") or []) for g in grupos[:3])
        print("  ✓ {0} grupos de subtítulos. Los primeros: {1}".format(
            len(grupos), muestra))
    if tomas:
        print("  ✓ toma repetida quitada: {0}".format(
            tomas[0]["motivo"].split("«")[-1].rstrip("»")))

    if problemas:
        raise Fallo("El vídeo ha salido, pero mal:\n  · "
                    + "\n  · ".join(problemas))

    resultado.update({
        "ok": True,
        "video": str(final),
        "duracion_original": round(info["duracion"], 2),
        "duracion_final": round(info_final["duracion"], 2),
        "grupos_subtitulos": len(grupos),
        "muestra_subtitulos": " ".join(
            " ".join(g.get("palabras") or []) for g in grupos[:6]),
        "segundos_totales": round(time.time() - inicio, 1),
        "carpeta": str(carpeta),
    })

    # El vídeo fabricado NO se borra aquí a propósito: mientras el proyecto de
    # prueba siga en workspace/, su 'origen' tiene que existir para poder repetir
    # cualquier paso a mano. Lo borra --limpiar, junto con el proyecto.
    return resultado


def limpiar():
    borrado = []
    for carpeta in (c.WORKSPACE / NOMBRE, c.WORKSPACE / MATERIAL):
        if carpeta.exists():
            shutil.rmtree(carpeta)
            borrado.append(carpeta.name)
    print("Borrado: {0}".format(", ".join(borrado)) if borrado
          else "No había nada de la prueba que borrar.")


def main():
    ap = argparse.ArgumentParser(description="Prueba la instalación del kit")
    ap.add_argument("--json", action="store_true", dest="como_json")
    ap.add_argument("--limpiar", action="store_true",
                    help="borrar el proyecto de prueba")
    ap.add_argument("--sin-visuales", action="store_true", dest="sin_visuales")
    a = ap.parse_args()

    if a.limpiar:
        limpiar()
        return 0

    try:
        resultado = correr(con_visuales=not a.sin_visuales)
    except (Fallo, c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        if a.como_json:
            print(json.dumps({"ok": False, "error": str(e)}, indent=2,
                             ensure_ascii=False))
        else:
            print("\n✗ La prueba se ha parado aquí:\n\n{0}".format(e),
                  file=sys.stderr)
            print("\nCópiale a Claude este mensaje entero: con eso sabe qué "
                  "arreglar.", file=sys.stderr)
        return 1

    if a.como_json:
        print(json.dumps(resultado, indent=2, ensure_ascii=False))
        return 0

    print("\n✓ El kit funciona en este ordenador. "
          "Prueba completa en {0:.0f} s.".format(resultado["segundos_totales"]))
    for aviso in resultado["avisos"]:
        print("\n! {0}".format(aviso))
    print("\nEl vídeo de prueba está en:\n  {0}".format(resultado["video"]))
    print("\nCuando lo hayas visto se puede borrar:\n"
          "  python scripts/prueba.py --limpiar")
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Efectos de sonido y volumen final. Este paso cierra el vídeo.

Uso:  python scripts/sonido.py <carpeta-del-proyecto>

Hace dos cosas:
  1. Iguala el volumen de la voz a -14 LUFS, que es el nivel al que suenan
     TikTok, Instagram y YouTube. Si no se hace, el vídeo se oye más bajo
     que el resto del feed y la gente pasa de largo.
  2. Mezcla los efectos de sonido que haya en trabajo/sfx.json.

Si ese archivo no existe se propone una mezcla sobria (un barrido corto en
los saltos largos) y se guarda para que se pueda retocar y volver a correr.

Salida:  <carpeta-del-proyecto>/<nombre>-vertical.mp4   ← el vídeo terminado
"""
from __future__ import annotations

import argparse
import json
import re
import subprocess
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

LUFS_OBJETIVO = -14.0    # nivel de las redes sociales
PICO_MAXIMO = -1.5       # margen para que no sature al recomprimir
MAX_EVENTOS = 60         # más que esto es ruido, no edición
GAP_PARA_BARRIDO = 0.70  # silencio quitado a partir del cual el salto se nota
MAX_BARRIDOS = 12


def catalogo():
    datos = c.leer_json(c.SFX / "sonidos.json", {})
    return {k: v for k, v in datos.items() if not k.startswith("_")}


# ── volumen de la voz ──────────────────────────────────────────────────────


def medir_voz(archivo):
    """Mide el volumen real de la voz. Devuelve None si no se puede medir.

    Se hace en dos pasadas a propósito: en una sola, loudnorm va corrigiendo
    a ciegas mientras avanza y la voz sube y baja. Midiendo primero, la
    corrección es un único ajuste parejo de principio a fin.
    """
    cmd = [c.ffmpeg(), "-nostdin", "-hide_banner", "-i", str(archivo),
           "-map", "0:a:0",
           "-af", "loudnorm=I={0}:TP={1}:LRA=11:print_format=json".format(
               LUFS_OBJETIVO, PICO_MAXIMO),
           "-f", "null", "-"]
    try:
        proc = subprocess.run(cmd, capture_output=True, text=True,
                              encoding="utf-8", errors="replace", timeout=600)
    except (OSError, subprocess.SubprocessError):
        return None
    bloques = re.findall(r"\{[^{}]*input_i[^{}]*\}", proc.stderr or "", re.S)
    if not bloques:
        return None
    try:
        medido = json.loads(bloques[-1])
        return {k: medido[k] for k in
                ("input_i", "input_tp", "input_lra", "input_thresh")}
    except (ValueError, KeyError):
        return None


def filtro_voz(medida):
    base = "loudnorm=I={0}:TP={1}:LRA=11".format(LUFS_OBJETIVO, PICO_MAXIMO)
    if medida:
        base += (":measured_I={input_i}:measured_TP={input_tp}"
                 ":measured_LRA={input_lra}:measured_thresh={input_thresh}"
                 ":linear=true".format(**medida))
    return base


# ── efectos ────────────────────────────────────────────────────────────────


def eventos_por_defecto(estado, total):
    """Propone un barrido corto en cada salto largo, y nada más.

    Un efecto en cada corte cansa; en los saltos largos ayuda a entender
    que ahí faltaba algo.
    """
    plan = c.leer_json(c.trabajo(estado, "plan-cortes.json"), None)
    if not plan or not plan.get("mapa"):
        return []
    mapa = plan["mapa"]
    saltos = []
    for i in range(1, len(mapa)):
        quitado = mapa[i][0] - mapa[i - 1][1]
        momento = mapa[i][2]
        if quitado >= GAP_PARA_BARRIDO and 0.2 < momento < total - 0.2:
            saltos.append((quitado, momento))
    # Si hay muchos, se queda con los saltos más largos: son los que más se ven.
    saltos.sort(key=lambda s: -s[0])
    return [{"sonido": "whoosh-short", "t": round(momento, 3),
             "ganancia": 0.24,
             "nota": "salto de {0:.1f} s".format(quitado)}
            for quitado, momento in sorted(saltos[:MAX_BARRIDOS],
                                           key=lambda s: s[1])]


def preparar(eventos, cat, total):
    """Valida los efectos y calcula en qué milisegundo entra cada uno."""
    listos, avisos = [], []
    for ev in eventos:
        nombre = str(ev.get("sonido", "")).strip()
        ficha = cat.get(nombre)
        if not ficha:
            avisos.append("no existe el sonido '{0}'".format(nombre))
            continue
        archivo = c.SFX / ficha["archivo"]
        if not archivo.is_file():
            avisos.append("falta el archivo {0}".format(ficha["archivo"]))
            continue
        t = float(ev.get("t", 0))
        if t < 0 or t > total:
            avisos.append("'{0}' cae en {1:.2f} s, fuera del vídeo "
                          "({2:.2f} s)".format(nombre, t, total))
            continue
        dur = float(ficha.get("duracion", 1.0))
        if ev.get("alinear", ficha.get("alinear", "inicio")) == "final":
            arranque = t - dur
            if arranque < 0:
                avisos.append("'{0}' necesita {1:.1f} s de carrerilla y solo "
                              "hay {2:.1f} s: entra recortado".format(
                                  nombre, dur, t))
                arranque = 0.0
        else:
            arranque = t
        listos.append({
            "sonido": nombre,
            "archivo": archivo,
            "retardo": int(round(arranque * 1000)),
            "ganancia": max(float(ev.get("ganancia",
                                         ficha.get("ganancia", 0.35))), 0.0),
            "t": t,
        })
    listos.sort(key=lambda e: e["retardo"])
    if len(listos) > MAX_EVENTOS:
        avisos.append("hay {0} efectos; se usan los {1} primeros".format(
            len(listos), MAX_EVENTOS))
        listos = listos[:MAX_EVENTOS]
    return listos, avisos


# ── montaje del audio ──────────────────────────────────────────────────────


FORMATO = "aformat=sample_fmts=fltp:sample_rates=48000:channel_layouts=stereo"


def cerrar(proyecto, refrescar=False, sin_efectos=False, sin_nivelar=False):
    estado = c.cargar_proyecto(proyecto)
    carpeta = Path(estado["_carpeta"])
    montaje = c.trabajo(estado, "montaje.mp4")
    if not montaje.is_file():
        raise FileNotFoundError(
            "Falta trabajo/montaje.mp4. Corre antes:\n"
            "  python scripts/componer.py \"{0}\"".format(carpeta)
        )
    salida = carpeta / (carpeta.name + "-vertical.mp4")
    if salida.is_file() and not refrescar:
        print("Ya existe {0}. Usa --refrescar para rehacerlo.".format(salida.name))
        return salida

    total = c.duracion(montaje)
    tiene_voz = c.info_media(montaje)["tiene_audio"]

    archivo_sfx = c.trabajo(estado, "sfx.json")
    datos_sfx = c.leer_json(archivo_sfx, None)
    if datos_sfx is None:
        propuesta = eventos_por_defecto(estado, total)
        c.escribir_json(archivo_sfx, {
            "_leeme": ("Efectos de sonido del vídeo. 't' son segundos del vídeo "
                       "montado. 'ganancia' es el volumen (1.0 = original). Los "
                       "nombres disponibles están en assets/sfx/sonidos.json. "
                       "Cambia lo que quieras y vuelve a correr sonido.py "
                       "con --refrescar."),
            "eventos": propuesta,
        })
        datos_sfx = {"eventos": propuesta}
        if propuesta:
            print("Propuesta de efectos guardada en trabajo/sfx.json "
                  "({0} efectos).".format(len(propuesta)))

    eventos = [] if sin_efectos else list(datos_sfx.get("eventos") or [])
    listos, avisos = preparar(eventos, catalogo(), total)
    for aviso in avisos:
        print("  ! {0}".format(aviso))

    # Caso simple: nada que mezclar y nada que nivelar.
    if not listos and (not tiene_voz or sin_nivelar):
        c.correr_ffmpeg(["-i", str(montaje), "-c", "copy",
                         "-movflags", "+faststart", str(salida)])
        return _terminar(estado, salida, 0, tiene_voz)

    medida = None
    if tiene_voz and not sin_nivelar:
        print("Midiendo el volumen de la voz…")
        medida = medir_voz(montaje)
        if medida:
            print("  la voz está en {0:.1f} LUFS; se lleva a {1:.0f}".format(
                float(medida["input_i"]), LUFS_OBJETIVO))
        else:
            print("  ! No he podido medirla; se nivela al vuelo, que es "
                  "algo peor pero funciona.")

    entradas = ["-i", str(montaje)]
    filtros, mezclar = [], []

    if tiene_voz:
        cadena = "[0:a]" + FORMATO
        if not sin_nivelar:
            cadena += "," + filtro_voz(medida) + "," + FORMATO
        filtros.append(cadena + "[voz]")
        mezclar.append("voz")
    elif listos:
        # Sin voz hay que inventar un silencio de base: amix necesita algo
        # que marque la duración.
        filtros.append("anullsrc=r=48000:cl=stereo,atrim=0:{0:.3f}[voz]".format(
            total))
        mezclar.append("voz")

    for i, ev in enumerate(listos):
        entradas += ["-i", str(ev["archivo"])]
        cadena = "[{0}:a]{1},volume={2:.3f}".format(i + 1, FORMATO,
                                                    ev["ganancia"])
        if ev["retardo"] > 0:
            cadena += ",adelay={0}|{0}".format(ev["retardo"])
        filtros.append(cadena + "[e{0}]".format(i))
        mezclar.append("e{0}".format(i))

    if len(mezclar) > 1:
        filtros.append("{0}amix=inputs={1}:normalize=0:dropout_transition=0"
                       "[cruda]".format("".join("[" + m + "]" for m in mezclar),
                                        len(mezclar)))
        ultimo = "cruda"
    else:
        ultimo = mezclar[0]
    # El limitador es la red de seguridad: los efectos sumados a la voz
    # pueden pasarse de 0 dB y ahí es donde cruje el audio.
    filtros.append("[{0}]alimiter=limit=0.95:level=disabled,{1}[aout]".format(
        ultimo, FORMATO))

    print("Cerrando el vídeo: {0} efectos de sonido…".format(len(listos)))
    c.correr_ffmpeg(entradas + [
        "-filter_complex", ";".join(filtros),
        "-map", "0:v", "-map", "[aout]",
        "-t", "{0:.3f}".format(total),
        "-c:v", "copy", "-c:a", "aac", "-b:a", "192k", "-ar", "48000",
        "-movflags", "+faststart", str(salida),
    ])
    return _terminar(estado, salida, len(listos), tiene_voz)


def _terminar(estado, salida, n_efectos, tiene_voz):
    info = c.info_media(salida)
    megas = salida.stat().st_size / 1048576.0
    estado.setdefault("pasos", {})["sonido"] = "ok"
    estado["salida"] = str(salida)
    c.guardar_proyecto(estado)

    print("\n✓ Vídeo terminado")
    print("  {0}".format(salida))
    print("  {0}x{1} · {2} · {3:.1f} MB · {4}".format(
        info["ancho"], info["alto"], c.hhmmss(info["duracion"]), megas,
        "con audio" if tiene_voz else "sin audio"))
    if n_efectos:
        print("  {0} efectos de sonido mezclados".format(n_efectos))
    print("\nSiguiente paso: ábrelo y míralo entero antes de publicarlo.")
    print("  Si quieres cambiar algo:")
    print("    · subtítulos mal escritos → trabajo/correcciones.json")
    print("    · cortes de más o de menos → plan_cortes.py --umbral")
    print("    · efectos de sonido        → trabajo/sfx.json")
    return salida


def main():
    ap = argparse.ArgumentParser(description="Mezcla el sonido y cierra el vídeo")
    ap.add_argument("proyecto")
    ap.add_argument("--refrescar", action="store_true")
    ap.add_argument("--sin-efectos", action="store_true",
                    dest="sin_efectos", help="solo nivelar la voz")
    ap.add_argument("--sin-nivelar", action="store_true",
                    dest="sin_nivelar", help="no tocar el volumen de la voz")
    a = ap.parse_args()
    try:
        cerrar(a.proyecto, a.refrescar, a.sin_efectos, a.sin_nivelar)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Subtítulos karaoke: cada palabra se enciende cuando se dice.

Uso:  python scripts/subtitulos.py <carpeta-del-proyecto>

Los subtítulos se dibujan aquí, con Pillow, y salen como imágenes PNG con
fondo transparente. No se usa el filtro de texto de ffmpeg ni ficheros .ass
a propósito: así el resultado es idéntico en Mac y en Windows y no depende de
cómo esté compilado el ffmpeg de cada uno, que es de donde vienen la mitad de
los problemas con los subtítulos quemados.

Salida:
  trabajo/subs/*.png      una imagen por estado
  trabajo/subs/lista.txt  el orden y la duración de cada imagen
  trabajo/subs.json       los grupos y sus tiempos, para el resto del motor
"""
from __future__ import annotations

import argparse
import shutil
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

# Cómo se agrupan las palabras en pantalla.
MAX_PALABRAS = 4
MAX_CARACTERES = 18
HUECO_GRUPO = 0.55      # a partir de este silencio empieza grupo nuevo
HUECO_ENCADENADO = 0.25  # por debajo de esto no hay fundido: cambio seco
ADELANTO = 0.08          # el grupo aparece un pelo antes de la primera palabra
COLA = 0.12              # y se queda un pelo después de la última

ESTILO = {
    "tamano": 78,
    "color_texto": "#FFFFFF",
    "color_activo": "#FF7A00",
    # Alto de la tira dentro del vídeo. Más abajo choca con los botones y el
    # texto que TikTok e Instagram pintan encima.
    "y": 1300,
    "margen": 56,
    "borde": 8,
    "sombra": 0.45,
}

# Errores típicos de transcripción en vídeos de este tipo. Se pueden añadir
# más en trabajo/correcciones.json sin tocar este archivo.
CORRECCIONES = {
    "cloud": "claude", "cloude": "claude", "clod": "claude", "clode": "claude",
    "clau": "claude", "cloe": "claude", "yipiti": "gpt", "chatgpt": "chatgpt",
    "gitjub": "github", "yijab": "github", "prompt": "prompt",
    "termina": "terminal", "tiktok": "tiktok",
}
PUNTUACION = '.,;:!?"¿¡()[]…-–—'
POP = [(0.88, 0.55), (0.95, 0.85), (1.02, 1.0)]   # entrada con rebote
FUNDIDO = [0.60, 0.28]                            # salida


def _color(hexa, alfa=255):
    h = hexa.lstrip("#")
    return (int(h[0:2], 16), int(h[2:4], 16), int(h[4:6], 16), alfa)


# ── de tiempos del original a tiempos del vídeo cortado ────────────────────


def remapear(t, mapa):
    for i, (a, b, off) in enumerate(mapa):
        if a <= t < b:
            return off + (t - a), i
    return None, None


def palabras_en_corte(palabras, mapa, correcciones):
    """Solo las palabras que sobreviven a los cortes, con su tiempo nuevo."""
    salida = []
    for p in palabras:
        nuevo_inicio, trozo = remapear(p["inicio"], mapa)
        if nuevo_inicio is None:
            continue
        a, b, off = mapa[trozo]
        nuevo_fin = off + (min(p["fin"], b) - a)
        if nuevo_fin <= nuevo_inicio:
            nuevo_fin = nuevo_inicio + 0.08
        crudo = p["texto"].strip()
        limpio = crudo.strip(PUNTUACION)
        arreglado = correcciones.get(limpio.lower(), limpio)
        if not arreglado:
            continue
        salida.append({
            "inicio": round(nuevo_inicio, 3),
            "fin": round(nuevo_fin, 3),
            "texto": arreglado.upper(),
            "fin_frase": crudo[-1:] in ".?!…",
            "trozo": trozo,
        })
    return salida


def agrupar(palabras):
    grupos, actual = [], []
    for p in palabras:
        if actual:
            largo = (sum(len(x["texto"]) for x in actual) + len(actual)
                     + len(p["texto"]))
            corta = (p["inicio"] - actual[-1]["fin"] > HUECO_GRUPO
                     or len(actual) >= MAX_PALABRAS
                     or largo > MAX_CARACTERES
                     or actual[-1]["fin_frase"]
                     or p["trozo"] != actual[-1]["trozo"])
            if corta:
                grupos.append(actual)
                actual = []
        actual.append(p)
    if actual:
        grupos.append(actual)
    return grupos


def tiempos_grupos(grupos, total):
    """Cuándo entra y sale cada grupo, y qué palabra está activa en cada momento."""
    salida, fin_previo = [], 0.0
    for gi, g in enumerate(grupos):
        hueco_antes = g[0]["inicio"] - (grupos[gi - 1][-1]["fin"] if gi else 0.0)
        encadenado = gi > 0 and hueco_antes < HUECO_ENCADENADO
        if encadenado:
            # Habla seguida: el grupo anterior da paso al siguiente sin fundido,
            # que si no parece que parpadee.
            inicio = fin_previo
        else:
            inicio = max(g[0]["inicio"] - ADELANTO, fin_previo + 0.01)
        if gi + 1 < len(grupos):
            hueco_despues = grupos[gi + 1][0]["inicio"] - g[-1]["fin"]
            if hueco_despues < HUECO_ENCADENADO:
                fin = grupos[gi + 1][0]["inicio"]
            else:
                fin = min(g[-1]["fin"] + COLA,
                          grupos[gi + 1][0]["inicio"] - 0.10)
        else:
            hueco_despues = 99.0
            fin = min(g[-1]["fin"] + COLA, total)
        fin = max(fin, g[-1]["fin"] + 0.02)
        fin_previo = fin
        eventos = []
        for i in range(len(g)):
            desde = inicio if i == 0 else g[i]["inicio"]
            hasta = g[i + 1]["inicio"] if i + 1 < len(g) else fin
            if hasta > desde:
                eventos.append([round(desde, 3), round(hasta, 3), i])
        salida.append({
            "palabras": [p["texto"] for p in g],
            "inicio": round(inicio, 3), "fin": round(fin, 3),
            "entra": not encadenado,
            "sale": hueco_despues >= HUECO_ENCADENADO,
            "eventos": eventos,
        })
    return salida


# ── dibujo ─────────────────────────────────────────────────────────────────


class Pintor:
    """Dibuja una tira de subtítulo. Todas las tiras miden lo mismo."""

    def __init__(self, estilo):
        self.e = estilo
        self.ancho = c.ANCHO
        self.usable = c.ANCHO - 2 * estilo["margen"]
        self.alto_linea = int(estilo["tamano"] * 1.16)
        self.alto = self.alto_linea * 2 + 90

    def _partir(self, palabras, tipo):
        """Reparte las palabras en una o dos líneas que quepan a lo ancho."""
        lineas, actual = [], []
        for idx, palabra in enumerate(palabras):
            prueba = actual + [(palabra, idx)]
            texto = " ".join(p for p, _ in prueba)
            if actual and tipo.getlength(texto) > self.usable:
                lineas.append(actual)
                actual = [(palabra, idx)]
            else:
                actual = prueba
        if actual:
            lineas.append(actual)
        return lineas[:2]

    def tira(self, palabras, activo, escala=1.0, alfa=1.0):
        from PIL import Image, ImageDraw, ImageFilter

        e = self.e
        tamano = e["tamano"]
        tipo = c.fuente("titulo", tamano, 700)
        lineas = self._partir(palabras, tipo)
        # Si con dos líneas aún no cabe, se baja el cuerpo hasta que entre.
        while tamano > 52 and any(
                tipo.getlength(" ".join(p for p, _ in l)) > self.usable
                for l in lineas):
            tamano -= 4
            tipo = c.fuente("titulo", tamano, 700)
            lineas = self._partir(palabras, tipo)

        lienzo = Image.new("RGBA", (self.ancho, self.alto), (0, 0, 0, 0))
        mascara = Image.new("L", (self.ancho, self.alto), 0)
        d = ImageDraw.Draw(lienzo)
        dm = ImageDraw.Draw(mascara)

        alto_linea = int(tamano * 1.16)
        alto_bloque = alto_linea * len(lineas)
        y = (self.alto - alto_bloque) // 2

        cajas = []
        for linea in lineas:
            texto = " ".join(p for p, _ in linea)
            x = (self.ancho - tipo.getlength(texto)) / 2
            # El contorno se dibuja de una vez con la línea entera: si se
            # dibujara palabra a palabra, el borde de una taparía la anterior.
            d.text((x, y), texto, font=tipo, fill=(0, 0, 0, 255),
                   stroke_width=e["borde"], stroke_fill=(0, 0, 0, 255))
            dm.text((x, y), texto, font=tipo, fill=255,
                    stroke_width=e["borde"], stroke_fill=255)
            # Cada palabra se sitúa midiendo el trozo de línea que la precede,
            # así encaja exactamente con el contorno ya dibujado.
            cursor = 0
            for palabra, idx in linea:
                dx = tipo.getlength(texto[:cursor]) if cursor else 0.0
                cajas.append((x + dx, y, palabra, idx))
                cursor += len(palabra) + 1
            y += alto_linea

        # Sombra por debajo: sin ella el texto se pierde sobre fondos claros.
        if e["sombra"] > 0:
            difusa = mascara.filter(ImageFilter.GaussianBlur(11))
            sombra = Image.new("RGBA", lienzo.size, (0, 0, 0, 0))
            sombra.putalpha(difusa.point(lambda v: int(v * e["sombra"])))
            desplazada = Image.new("RGBA", lienzo.size, (0, 0, 0, 0))
            desplazada.paste(sombra, (0, 9))
            lienzo = Image.alpha_composite(desplazada, lienzo)
            d = ImageDraw.Draw(lienzo)

        for x0, y0, palabra, idx in cajas:
            color = _color(e["color_activo"] if idx == activo else e["color_texto"])
            d.text((x0, y0), palabra, font=tipo, fill=color)

        if escala != 1.0:
            nuevo = (max(int(self.ancho * escala), 2), max(int(self.alto * escala), 2))
            reducida = lienzo.resize(nuevo, Image.LANCZOS)
            lienzo = Image.new("RGBA", (self.ancho, self.alto), (0, 0, 0, 0))
            lienzo.paste(reducida, ((self.ancho - nuevo[0]) // 2,
                                    (self.alto - nuevo[1]) // 2))
        if alfa < 1.0:
            banda = lienzo.getchannel("A").point(lambda v: int(v * alfa))
            lienzo.putalpha(banda)
        return lienzo

    def vacia(self):
        from PIL import Image
        return Image.new("RGBA", (self.ancho, self.alto), (0, 0, 0, 0))


# ── secuencia de imágenes ──────────────────────────────────────────────────


def estados_por_fotograma(grupos, total, fps):
    """Qué imagen toca en cada fotograma, ya resuelto rebote y fundido."""
    tira = []
    cursor = 0
    for gi, g in enumerate(grupos):
        inicio_f = int(round(g["inicio"] * fps))
        if inicio_f > cursor:
            tira.append(("vacio", inicio_f - cursor))
            cursor = inicio_f
        estados = []
        for desde, hasta, idx in g["eventos"]:
            n = int(round(hasta * fps)) - int(round(desde * fps))
            estados.extend([("ev", gi, idx)] * max(n, 1))
        if not estados:
            continue
        if g["entra"]:
            for k in range(min(len(POP), len(estados))):
                estados[k] = ("pop", gi, k)
        if g["sale"] and len(estados) > len(POP) + len(FUNDIDO):
            ultimo = estados[-1][2]
            for k in range(len(FUNDIDO)):
                estados[-1 - k] = ("fade", gi, ultimo, k)
        # Se agrupan los fotogramas iguales para no repetir imágenes.
        for estado in estados:
            if tira and tira[-1][0] == estado:
                tira[-1] = (estado, tira[-1][1] + 1)
            else:
                tira.append((estado, 1))
        cursor += len(estados)
    final_f = int(round(total * fps))
    if final_f > cursor:
        tira.append(("vacio", final_f - cursor))
    return tira


def clave(estado):
    if estado == "vacio":
        return "vacio"
    if estado[0] == "ev":
        return "g{0:03d}-p{1}".format(estado[1], estado[2])
    if estado[0] == "pop":
        return "g{0:03d}-pop{1}".format(estado[1], estado[2])
    return "g{0:03d}-out{1}".format(estado[1], estado[3])


def generar(proyecto, refrescar=False):
    estado_p = c.cargar_proyecto(proyecto)
    trans = c.leer_json(c.trabajo(estado_p, "transcripcion.json"),
                        {"palabras": []})
    plan = c.leer_json(c.trabajo(estado_p, "plan-cortes.json"), None)
    master = c.trabajo(estado_p, "master.mp4")
    if not master.is_file():
        raise FileNotFoundError(
            "Falta trabajo/master.mp4. Corre primero:\n"
            "  python scripts/cortar.py \"{0}\"".format(estado_p["_carpeta"])
        )
    total = c.duracion(master)
    mapa = plan["mapa"] if plan and plan.get("mapa") else [[0.0, total, 0.0]]

    estilo = dict(ESTILO)
    estilo.update(estado_p.get("subs") or {})
    correcciones = dict(CORRECCIONES)
    correcciones.update({k.lower(): v for k, v in c.leer_json(
        c.trabajo(estado_p, "correcciones.json"), {}).items()})

    palabras = palabras_en_corte(trans.get("palabras") or [], mapa, correcciones)
    if not palabras:
        print("No hay palabras que subtitular (¿vídeo sin voz?). Se salta el paso.")
        c.escribir_json(c.trabajo(estado_p, "subs.json"),
                        {"grupos": [], "alto": 0, "y": estilo["y"]})
        return None

    grupos = tiempos_grupos(agrupar(palabras), total)
    tira = estados_por_fotograma(grupos, total, c.FPS)

    carpeta = c.trabajo(estado_p, "subs")
    if carpeta.exists() and refrescar:
        shutil.rmtree(carpeta)
    carpeta.mkdir(parents=True, exist_ok=True)

    pintor = Pintor(estilo)
    hechas = {}
    for est, _ in tira:
        nombre = clave(est)
        if nombre in hechas:
            continue
        destino = carpeta / (nombre + ".png")
        if est == "vacio":
            imagen = pintor.vacia()
        elif est[0] == "ev":
            imagen = pintor.tira(grupos[est[1]]["palabras"], est[2])
        elif est[0] == "pop":
            escala, alfa = POP[est[2]]
            imagen = pintor.tira(grupos[est[1]]["palabras"], 0, escala, alfa)
        else:
            imagen = pintor.tira(grupos[est[1]]["palabras"], est[2],
                                 1.0, FUNDIDO[est[3]])
        imagen.save(destino)
        hechas[nombre] = destino

    lineas = ["ffconcat version 1.0"]
    for est, n in tira:
        lineas.append("file '{0}.png'".format(clave(est)))
        lineas.append("duration {0:.6f}".format(n / c.FPS))
    # La última imagen se repite: si no, el demuxer ignora su duración.
    lineas.append("file '{0}.png'".format(clave(tira[-1][0])))
    (carpeta / "lista.txt").write_text("\n".join(lineas) + "\n", encoding="utf-8")

    c.escribir_json(c.trabajo(estado_p, "subs.json"), {
        "grupos": grupos,
        "alto": pintor.alto,
        "y": estilo["y"],
        "lista": "subs/lista.txt",
    })
    estado_p.setdefault("pasos", {})["subtitulos"] = "ok"
    c.guardar_proyecto(estado_p)

    print("Subtítulos: {0} grupos, {1} imágenes, {2} tramos".format(
        len(grupos), len(hechas), len(tira)))
    print("  tira de {0}x{1} px colocada en y={2}".format(
        pintor.ancho, pintor.alto, estilo["y"]))
    for g in grupos[:6]:
        print("  {0}  {1}".format(c.hhmmss(g["inicio"]), " ".join(g["palabras"])))
    if len(grupos) > 6:
        print("  … y {0} grupos más".format(len(grupos) - 6))
    print("\nSiguiente paso: quemar los subtítulos y los efectos")
    print("  python scripts/componer.py \"{0}\"".format(estado_p["_carpeta"]))
    return grupos


def main():
    ap = argparse.ArgumentParser(description="Genera los subtítulos karaoke")
    ap.add_argument("proyecto")
    ap.add_argument("--refrescar", action="store_true",
                    help="borrar las imágenes anteriores")
    a = ap.parse_args()
    try:
        generar(a.proyecto, a.refrescar)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Transcribe el audio del vídeo con marca de tiempo en cada palabra.

Dos motores, misma salida:
  · whisper local (por defecto)  → gratis, sin claves, funciona sin internet
  · AssemblyAI (opcional)        → más preciso, necesita clave y sube el audio

Uso:  python scripts/transcribir.py <carpeta-del-proyecto> [--motor whisper|assemblyai]

Escribe trabajo/transcripcion.json:
  {"motor", "idioma", "palabras": [{"inicio", "fin", "texto", "prob"}], "frases": [...]}
Los tiempos van en segundos sobre el vídeo ORIGINAL, sin cortar.
"""
from __future__ import annotations

import argparse
import json
import os
import sys
import time
import warnings
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
import comun as c

MODELO_POR_DEFECTO = "small"

# Le damos a whisper una muestra del estilo que esperamos: así puntúa y acentúa
# mejor, y escribe bien las palabras técnicas que salen en este tipo de vídeos.
PISTA_ESTILO = (
    "Vídeo en español sobre programación, inteligencia artificial y Claude Code. "
    "Habla de terminal, prompts, tokens, repositorio, API y modelos."
)


# ── audio para la transcripción ────────────────────────────────────────────


def extraer_voz(origen, destino):
    """Saca el audio a 16 kHz mono, que es lo que quieren los dos motores."""
    destino = Path(destino)
    if destino.is_file() and destino.stat().st_size > 1024:
        return destino
    c.correr_ffmpeg([
        "-i", str(origen), "-vn", "-ac", "1", "-ar", "16000",
        "-c:a", "pcm_s16le", str(destino),
    ])
    return destino


# ── motor 1: whisper en tu ordenador ───────────────────────────────────────


def _cargar_whisper(modelo):
    try:
        from faster_whisper import WhisperModel
    except ImportError:
        raise c.FaltaDependencia(
            "Falta whisper. Instálalo con:\n"
            "  python3 -m pip install faster-whisper\n"
            "O pide a Claude que corra el doctor del kit."
        )
    hilos = max(1, min(os.cpu_count() or 4, 8))
    return WhisperModel(modelo, device="cpu", compute_type="int8", cpu_threads=hilos)


def con_whisper(audio, modelo=MODELO_POR_DEFECTO, idioma=None):
    modelo_cargado = _cargar_whisper(modelo)
    # En los silencios CTranslate2 divide entre cero y numpy avisa. Es inofensivo
    # y llenaría la pantalla de ruido, así que lo callamos.
    warnings.filterwarnings("ignore", message=".*divide by zero.*")
    with warnings.catch_warnings():
        warnings.simplefilter("ignore")
        segmentos, info = modelo_cargado.transcribe(
            str(audio),
            language=idioma,
            word_timestamps=True,
            vad_filter=True,
            vad_parameters={"min_silence_duration_ms": 400},
            condition_on_previous_text=False,
            initial_prompt=PISTA_ESTILO,
            beam_size=5,
        )
        palabras, frases = [], []
        for seg in segmentos:
            texto = (seg.text or "").strip()
            if texto:
                frases.append({
                    "inicio": round(seg.start, 3),
                    "fin": round(seg.end, 3),
                    "texto": texto,
                })
            for p in seg.words or []:
                limpio = (p.word or "").strip()
                if limpio:
                    palabras.append({
                        "inicio": round(p.start, 3),
                        "fin": round(p.end, 3),
                        "texto": limpio,
                        "prob": round(float(p.probability), 3),
                    })
    return {
        "motor": "whisper-local ({0})".format(modelo),
        "idioma": info.language,
        "palabras": palabras,
        "frases": frases,
    }


# ── motor 2: AssemblyAI (opcional) ─────────────────────────────────────────

API_AAI = "https://api.assemblyai.com/v2"


def _clave_aai():
    """Busca la clave en el entorno o en .env.local. Nunca se pide por chat."""
    clave = os.environ.get("ASSEMBLYAI_API_KEY", "").strip()
    if clave:
        return clave
    env = c.RAIZ / ".env.local"
    if env.is_file():
        for linea in env.read_text(encoding="utf-8").splitlines():
            if linea.strip().startswith("ASSEMBLYAI_API_KEY"):
                _, _, valor = linea.partition("=")
                return valor.strip().strip("'\"")
    return ""


def _pedir(ruta, clave, datos=None, metodo=None, tipo="application/json"):
    import urllib.error
    import urllib.request

    cuerpo = json.dumps(datos).encode() if isinstance(datos, dict) else datos
    pet = urllib.request.Request(
        API_AAI + ruta, data=cuerpo, method=metodo,
        headers={"authorization": clave, "content-type": tipo},
    )
    try:
        with urllib.request.urlopen(pet, timeout=300) as r:
            return json.loads(r.read().decode())
    except urllib.error.HTTPError as e:
        detalle = e.read().decode(errors="replace")[:300]
        if e.code in (401, 403):
            raise RuntimeError(
                "AssemblyAI rechaza la clave. Revísala en .env.local "
                "(ASSEMBLYAI_API_KEY) o usa el motor local: --motor whisper."
            )
        raise RuntimeError("AssemblyAI respondió {0}: {1}".format(e.code, detalle))
    except urllib.error.URLError as e:
        raise RuntimeError(
            "No hay conexión con AssemblyAI ({0}). Con --motor whisper "
            "funciona sin internet.".format(e.reason)
        )


def con_assemblyai(audio, idioma=None):
    clave = _clave_aai()
    if not clave:
        raise c.FaltaDependencia(
            "No hay clave de AssemblyAI. Añádela a .env.local así:\n"
            "  ASSEMBLYAI_API_KEY=tu-clave\n"
            "O transcribe en local, que no necesita clave: --motor whisper."
        )
    with open(audio, "rb") as f:
        subida = _pedir("/upload", clave, datos=f.read(),
                        tipo="application/octet-stream")
    peticion = {
        "audio_url": subida["upload_url"],
        "speech_model": "universal",
        "punctuate": True,
        "format_text": True,
    }
    if idioma:
        peticion["language_code"] = idioma
    else:
        peticion["language_detection"] = True
    tarea = _pedir("/transcript", clave, datos=peticion)
    ident = tarea["id"]
    print("  · AssemblyAI procesando…")
    for _ in range(300):
        time.sleep(3)
        estado = _pedir("/transcript/" + ident, clave, metodo="GET")
        if estado["status"] == "completed":
            break
        if estado["status"] == "error":
            raise RuntimeError("AssemblyAI falló: {0}".format(estado.get("error")))
    else:
        raise RuntimeError("AssemblyAI tarda demasiado. Prueba con --motor whisper.")

    palabras = [{
        "inicio": round(p["start"] / 1000.0, 3),
        "fin": round(p["end"] / 1000.0, 3),
        "texto": p["text"].strip(),
        "prob": round(float(p.get("confidence", 1.0)), 3),
    } for p in estado.get("words") or [] if p["text"].strip()]
    frases = [{
        "inicio": round(f["start"] / 1000.0, 3),
        "fin": round(f["end"] / 1000.0, 3),
        "texto": f["text"].strip(),
    } for f in (estado.get("utterances") or [])]
    if not frases and estado.get("text"):
        frases = [{"inicio": palabras[0]["inicio"] if palabras else 0.0,
                   "fin": palabras[-1]["fin"] if palabras else 0.0,
                   "texto": estado["text"].strip()}]
    return {
        "motor": "assemblyai",
        "idioma": estado.get("language_code") or idioma or "es",
        "palabras": palabras,
        "frases": frases,
    }


# ── entrada ────────────────────────────────────────────────────────────────


def transcribir(proyecto, motor=None, modelo=None, idioma=None, refrescar=False):
    estado = c.cargar_proyecto(proyecto)
    salida = c.trabajo(estado, "transcripcion.json")
    if salida.is_file() and not refrescar:
        previo = c.leer_json(salida)
        print("Ya estaba transcrito ({0} palabras, motor {1}).".format(
            len(previo["palabras"]), previo["motor"]))
        return previo

    origen = c.ruta_origen(estado)
    audio = extraer_voz(origen, c.trabajo(estado, "voz.wav"))
    motor = motor or estado.get("motor_transcripcion") or "whisper"
    idioma = idioma or estado.get("idioma") or None

    print("Transcribiendo con {0}…".format(motor))
    inicio = time.time()
    if motor.startswith("assembly"):
        datos = con_assemblyai(audio, idioma)
    else:
        datos = con_whisper(audio, modelo or estado.get("modelo_whisper")
                            or MODELO_POR_DEFECTO, idioma)
    datos["duracion_audio"] = round(c.duracion(audio), 3)
    c.escribir_json(salida, datos)

    estado["motor_transcripcion"] = datos["motor"]
    estado["idioma"] = datos["idioma"]
    estado.setdefault("pasos", {})["transcribir"] = "ok"
    c.guardar_proyecto(estado)

    print("  ✓ {0} palabras, {1} frases, idioma {2} ({3:.0f}s de proceso)".format(
        len(datos["palabras"]), len(datos["frases"]), datos["idioma"],
        time.time() - inicio))
    print("  → {0}".format(salida))
    return datos


def main():
    ap = argparse.ArgumentParser(description="Transcribe con tiempos por palabra")
    ap.add_argument("proyecto", help="carpeta del proyecto en workspace/")
    ap.add_argument("--motor", choices=["whisper", "assemblyai"], default=None)
    ap.add_argument("--modelo", default=None, help="tiny|base|small|medium|large-v3")
    ap.add_argument("--idioma", default=None, help="es, en… (vacío = detectar)")
    ap.add_argument("--refrescar", action="store_true", help="rehacer aunque exista")
    a = ap.parse_args()
    try:
        transcribir(a.proyecto, a.motor, a.modelo, a.idioma, a.refrescar)
    except (c.FaltaDependencia, RuntimeError, FileNotFoundError) as e:
        print("\n{0}".format(e), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
