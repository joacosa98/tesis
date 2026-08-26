# Guión de defensa — Behavior Trees (slides 11 a 23)

Duración objetivo: **10 a 11 minutos** en total.
Texto en *cursiva* = lo que decís. `[entre corchetes]` = acotación / cuándo avanzar.

---

## Slide 11 — Comportamiento: Behavior Trees (~2 min)

**1. Presentar el concepto**

*"Con la percepción resuelta, ahora voy a hablar del control del comportamiento del robot. Para esto implementamos un módulo propio de Árboles de Comportamiento, o Behavior Trees, en Lua.*

*Entonces, qué es un Behavior Tree? Es básicamente una estructura jerárquica de nodos que se ejecuta desde la raíz hacia abajo mediante ticks periódicos. En cada tick el árbol se recorre desde la raíz y cada nodo decide qué hacer y qué devolverle a su padre. O sea: no es un grafo de estados con transiciones entre sí, sino un árbol que se vuelve a evaluar completo, una y otra vez."*

**2. Los tipos de nodos (mencionar por arriba, sin detallar)**

*"Los nodos se agrupan en dos familias: los de **ejecución** y los de **control**, que son los que se muestran ahi. Y en las próximas dos slides vamos a explicar la ejecución de los nodos usados en el árbol que definimos"*

**3. Resultados posibles**

*"Algo importante a entender de los nodos es que estos, en cada tick, devuelven uno de tres valores. SUCCESS, que representa "la tarea se completó correctamente". FAILURE, si no se pudo completar o se completó con error. Y RUNNING, si la tarea sigue en progreso y hay que retomarla en los ticks siguientes.*

*Este último estado permite que el robot nunca se quede trabado adentro de una acción, sino que cede el control y en el próximo tick retoma."*

**4. La estructura toma las decisiones**

*"Uno de los principales beneficios de Behavior Trees es que debido a la estructura jerárquica del árbol, un nodo solo sabe hacer su tarea y reportar cómo le fue, pero es la estructura del árbol la encargada de decidir qué pasa a continuación. Esto permite modularización y reutilización de nodos, y facilidad de a la hora de desarrollar y mantener el sistema*


> `[Avanzar]`

---

## Slide 12 — Nodos de ejecución (~1 min 15 s)

*"Empecemos por los nodos de ejecución"*

**Action** `[rectángulo]`

*"El nodo action encapsula una operación concreta, por ejemplo, seguir la línea, rotar buscando una tarjeta."*

**Condition** `[elipse]`

*"El nodo condition evalúa un predicado (¿el tag leido corresponde a un tag de parada?) y devuelve SUCCESS o FAILURE. A diferencia de los demás, nunca devuelve RUNNING"*

**Decorator** `[rombo, un único hijo]`

*"El decorator es un nodo con un único hijo que modifica su comportamiento o su resultado, sin modificar al hijo. Se lo puede pensar como un envoltorio*

*En nuestro módulo implementamos un ejemplo del modelo clásico*

- ***repeater**, que repite la ejecución de su hijo n veces -o indefinidamente- por ejemplo, seguir linea por 5 ticks*

> `[Avanzar]`

---

## Slide 13 — Nodos de control (~1 min 45 s)

**Sequence** `[la flecha]`

*"El sequence funciona como un AND. Ejecuta a sus hijos de izquierda a derecha y avanza al siguiente solo si el anterior devolvió SUCCESS. Si un hijo devuelve FAILURE o RUNNING, se detiene ahí y propaga ese estado a su padre. Devuelve SUCCESS solo si todos tuvieron éxito. Sirve para encadenar pasos: seguir la línea, detectar la parada, escanear la tarjeta, validarla, mostrar el resultado, retomar el recorrido."*

**Fallback / Selector** `[el signo de pregunta]`

*"El fallback —o selector— funciona como un OR. Intenta a sus hijos en orden (de izquierda a derecha) hasta que uno devuelve SUCCESS o RUNNING, y devuelve FAILURE solo si fallaron todos. Sirve para alternativas: en nuestro árbol es el que decide, en cada ciclo, si lo que hay adelante es un tag de parada, un tag de fin, o ninguno de los dos y hay que seguir avanzando."*

**Con memoria vs. sin memoria**

*"Un punto importante para aclarar, es qué pasa en el tick siguiente cuando un hijo quedó en RUNNING. Para eso se definen nodos con memoria o sin memoria*

*Un nodo **con memoria** recuerda en qué hijo se quedó: en el próximo tick retoma directamente desde ese hijo, sin volver a evaluar a los anteriores.*

*Un nodo **sin memoria** —reactivo— vuelve a empezar desde el primer hijo en cada tick y re-evalúa todas las condiciones previas. Cuesta más, pero garantiza que si una condición dejó de cumplirse, la rama se abandona inmediatamente.*

*En nuestra implementación los compositores son **con memoria**"*

*"Y un último detalle sobre nuestra implementación. Los nodos no se pasan datos entre sí: todos trabajan sobre un mismo objeto compartido, al que llamamos contexto, donde vive el estado del sistema. Por ejemplo, si un docente levanta el robot en el medio del recorrido no se pierde nada: se lo reubica y retoma donde estaba."*

> `[Avanzar]`

---

## Slide 14 — El subárbol de la estación (~2 min)

> `[Primera imagen de la serie: el árbol completo, sin ningún nodo resaltado.]`

**Dónde estamos parados**

*"Con estos nodos armamos el árbol que controla toda la actividad, y queriamos mostrar un ejemplo visual de ejecución del mismo. Como el árbol es extenso, vamos a mostrar el subárbol de la parte central de la actividad: lo que hace el robot cuando llega a una estación.*

*Hay una parte del recorrido en la que el robot realiza algunas iteraciones avanzando siguiendo la línea, y después cambia la cámara a modo detección de tags, realiza una lectura, y le pregunta a un fallback qué hacer con lo que ve. Este es el primer nodo hijo del fallback y corresponde a la secuencia correspondiende a "leer un tag de estación"*

**El nodo raíz**

*"Arriba de todo se ve la **sequence** (con memoria según el asterisco)"*

**Los cinco hijos, uno por uno**

*"El primero es una **condition**, que se pone al inicio de la sequence como "guarda" que evita que se ejecute la secuencia cuando no se quiere, y corresponde a 'stop tag seen'. Solo verifica si entre la lectura que la cámara acaba de realizar incluye el tag de parada.*

*El segundo es **scan checkpoint**: Detiene el robot y lo pone a rotar en sentido horario buscando el tag de la tarjeta que el grupo dejó en la estación, y una vez leído el tag de tarjeta, valida la secuencia y guarda el resultado*

*El tercero es **checkpoint led**: el robot muestra retroalimentación de la estación leída y de la secuencia leída hasta el momento*

*El cuarto es **align stop**: el robot vuelve a rotar en sentido antihorario hasta volver a encuadrar el tag de parada*

*Y el quinto es **resume line**: devuelve la cámara a modo seguimiento de línea y deja correr unos ticks de estabilización antes de seguir el recorrido*

*Vamos a recorrerlo paso a paso."*

> `[Avanzar]`

---

## Slide 15 — La guarda pasa (~25 s)

> `[Condition en verde, sequence y scan checkpoint en naranja.]`

*"Acá la cámara ya vio el tag de parada. La condition devuelve SUCCESS, y por primera vez la secuencia habilita al segundo hijo. En ese mismo tick, como el primer hijo ya retornó SUCCESS, scan checkpoint se ejecuta y retorna en RUNNING. El robot frena y empieza a rotar."*

## Slide 16 — Escaneando (~25 s)

> `[Solo scan checkpoint en naranja; la condition ya no está marcada.]`

*"En los siguientes ticks, se ve la memoria de la que hablábamos. Ahora la condition ya no está pintada: la secuencia no la vuelve a evaluar, retoma directo en el hijo que había quedado corriendo. El robot sigue rotando, buscando el tag de la tarjeta, y esto se repite durante varios ticks."*

## Slide 17 — Tarjeta encontrada (~20 s)

> `[Scan checkpoint en verde, checkpoint led en naranja.]`

*"En este paso el robot encontró el tag de la tarjeta. Lo compara con la posición actual de la secuencia configurada, guarda el resultado y devuelve SUCCESS. La sequence avanza al tercer hijo: el feedback."*

## Slide 18 — El robot piensa (~20 s)

> `[Solo checkpoint led en naranja; anillo blanco girando.]`

*"Este es el estado 'pensativo': la luz blanca gira alrededor del anillo. Técnicamente no hace falta —la validación ya está hecha—, existe puramente por una razón pedagógica: darle al grupo un segundo para entender que el robot está evaluando lo que acaba de leer, antes de mostrar el resultado de la validación."*

## Slide 19 — El veredicto (~20 s)

> `[Checkpoint led en verde, align stop en naranja.]`

*"Se mostró el resultado y la acción devuelve SUCCESS. Ahora arranca align stop: el robot quedó mirando la tarjeta, así que tiene que volver a rotar hasta encontrar el tag de parada."*

## Slide 20 — Realineando (~15 s)

> `[Solo align stop en naranja.]`

*"Sigue rotando y retornando RUNNING"*

## Slide 21 — De vuelta a la línea (~20 s)

> `[Align stop en verde, resume line en naranja.]`

*"Hasta que encuentra el tag de parada y retorna SUCCESS. Entra el último hijo, resume line, que devuelve la cámara a modo seguimiento de línea y deja unos ticks de estabilización para que el robot no arranque con una lectura todavía inestable."*

## Slide 22 — Estación completa (~40 s)

> `[Resume line en verde y, sobre todo, la sequence raíz en verde.]`

*"Con los cinco hijos en SUCCESS, la sequence devuelve SUCCESS a su padre. La estación está cerrada: el ciclo de recorrido se completa y vuelve a empezar —seguir la línea, mirar tags— hasta la próxima estación. Y como la rama terminó, su memoria se reinicia: en la próxima vuelta la guarda se vuelve a evaluar desde cero.*

> `[Avanzar al video]`

---

## Slide 23 — El recorrido real (~40 s)

> `[Reproducir el video. Acá hablar poco: ya está todo explicado, esto es la confirmación en velocidad real.]`

*"Y así se ve todo eso a velocidad real: el robot llega a la estación, se detiene, busca la tarjeta, la valida y retoma el recorrido.*

**Cómo quedó implementado** `[cierre de la sección]`

*"Como conclusión sobre la implementación. Todo esto está repartido en dos piezas bien separadas.*

*El **motor de Behavior Trees** es un módulo en Lua, independiente y sin dependencias: implementa los nodos que vimos —secuencias, selectores, acciones, condiciones y los decoradores— y no sabe nada de Robotito, de la cámara ni de esta actividad. Es la biblioteca reutilizable que planteamos como objetivo.*

*El **comportamiento concreto** es otro archivo Lua, el de la actividad, que importa esa biblioteca y arma el árbol combinando sus nodos: ahí viven las funciones que efectivamente mueven el robot, leen la cámara, encienden el anillo y validan la secuencia.*

*Esa división es la que hace que cambiar el comportamiento del robot no requiera recompilar ni reflashear nada: se edita el script de la actividad y se sube al robot. El código de los dos está publicado en los repositorios del proyecto."*


---

## Notas de apoyo

### Estructura de la sección en el deck

El deck ya tiene las slides: 11 concepto, 12 nodos de ejecución, 13 nodos de control, **14 a 22 el paso a paso** (una por imagen de `images/bt-recorrido/`) y **23 el video**. El deck pasó de 22 a 31 slides y los contadores están actualizados.

Ojo con el orden respecto de lo que veníamos hablando: como las imágenes van antes del video, el paso a paso quedó **antes** y el video **después**, como cierre en velocidad real. Funciona bien —cuando se ve el video ya está todo explicado— pero implica que la slide 14 hace doble tarea: presenta la estructura del subárbol y además es el primer cuadro de la serie.

Las nueve slides no llevan título ni subtítulo: son solo la imagen, para que el diagrama se lea lo más grande posible.

### Preguntas probables del tribunal

- **¿Por qué BT y no una máquina de estados?** → Implementar el módulo de Behavior Trees era uno de los objetivos del proyecto, justamente como alternativa al módulo de máquinas de estado que ya usaban los scripts de Robotito. Sobre el fondo de la pregunta: en una FSM la lógica de transición se reparte entre los estados y crece de forma cuadrática; en un BT está concentrada en la estructura y agregar comportamiento es colgar un subárbol.
- **¿Por qué no usaron una biblioteca de BT existente?** → Era un objetivo específico implementarla como biblioteca reutilizable en Lua para Lua RTOS; las implementaciones disponibles no apuntan a ese entorno y son más pesadas de lo que tolera un ESP32.
- **¿Cuál es el costo en un ESP32?** → El módulo implementa solo lo esencial, evita referencias circulares y opera sobre un único contexto compartido. Por eso no implementamos parallel: la reactividad se resolvió con decoradores de interrupción, más baratos en memoria.
- **¿Cada cuánto se ejecuta el tick?** → [completar con el período real del script].
- **¿Qué pasa si una acción nunca termina?** → Queda en RUNNING; los decoradores de interrupción permiten abortarla ante un evento prioritario, y las acciones de escaneo tienen además un límite de tiempo que las hace fallar y disparar el modo de recuperación.
- **¿Cómo se recupera de una falla?** → El árbol entra en la rama de recuperación: detiene el robot, parpadea en amarillo y espera a que se lo reubique y se le muestre el tag de inicio (retoma con el progreso intacto) o el de reinicio (descarta la secuencia validada y arranca de cero).

### Antes de la defensa

- Completar el período del tick.
- Ensayar la slide 14 con el video corriendo: la narración tiene que entrar en la duración del clip (33 s).
- Verificar que la carpeta `videos/` viaje junto al HTML en la máquina de la defensa.
