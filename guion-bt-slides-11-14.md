# Guión de defensa — Behavior Trees (slides 11 a 23)

Duración objetivo: **10 a 11 minutos** en total.
Texto en *cursiva* = lo que decís. `[entre corchetes]` = acotación / cuándo avanzar.

---

## Slide 11 — Comportamiento: Behavior Trees (~2 min)

**1. Presentar el concepto**

*"Ahora voy a hablar del control del comportamiento del robot. Para esto implementamos un módulo propio de Árboles de Comportamiento, o Behavior Trees, en Lua.*

*Un Behavior Tree es una estructura jerárquica de nodos que se ejecuta desde la raíz hacia abajo mediante ticks periódicos. En cada tick el árbol se recorre desde la raíz y cada nodo decide qué hacer y qué devolverle a su padre. Es decir: no es un grafo de estados con transiciones entre sí, sino un árbol que se vuelve a evaluar completo, una y otra vez."*

**2. Los tipos de nodos (mencionar por arriba, sin detallar)**

*"Los nodos se agrupan en dos familias. Por un lado los nodos de ejecución —action, condition y decorator— que son los que efectivamente hacen algo o miran el estado del sistema. Por otro los nodos de control —sequence, fallback y parallel— que no hacen nada por sí mismos, sino que deciden en qué orden y bajo qué criterio se ejecutan sus hijos. En las próximas dos slides los vemos uno por uno."*

**3. Resultados posibles**

*"Lo que hace que todo esto funcione es que todos los nodos hablan el mismo idioma: cada nodo ejecutado, en cada tick, devuelve uno de tres valores. SUCCESS, si la tarea se completó correctamente. FAILURE, si no se pudo completar. Y RUNNING, si la tarea sigue en progreso y hay que retomarla en los ticks siguientes.*

*Ese tercer estado, RUNNING, es la clave para un sistema embebido como este: permite que una acción larga —por ejemplo, seguir la línea hasta la próxima estación, o rotar buscando una tarjeta— se ejecute a lo largo de muchos ticks sin bloquear el resto del sistema. El robot nunca se queda trabado adentro de una acción: cede el control y en el próximo tick retoma."*

**4. La estructura toma las decisiones**

*"Uno de los principales beneficios de Behavior Trees es dónde vive la lógica de decisión. Debido a la estructura jerárquica del árbol, un nodo no necesita saber quién es su padre, ni quiénes son sus hermanos, ni qué pasa después de él. Un nodo action solo sabe hacer su tarea y reportar cómo le fue. Quien decide qué pasa a continuación es la estructura del árbol, no el nodo, evitando lógica complicada que dificulta el desarrollo y modularización del sistema."*

**5. Los demás beneficios (encadenar con lo anterior: son la consecuencia)**

*"De esa propiedad se desprende todo lo demás. Los nodos son independientes entre sí, así que el mismo nodo se puede reutilizar en distintas partes del árbol o en un árbol completamente distinto, y agregar un comportamiento nuevo es colgar un subárbol sin revisar lo que ya funcionaba. El árbol se lee como se lee el comportamiento, así que el diagrama es la documentación. Como se re-evalúa en cada tick, el robot reacciona a lo que pasa en el entorno —perder la línea, que lo levanten de la mesa, leer un tag de parada— sin haber previsto esa transición desde cada punto posible. Y como nuestro motor opera sobre un contexto con el estado del sistema, en vez de llamar directo al hardware, el módulo quedó como una biblioteca reutilizable e independiente del hardware de Robotito, que otros proyectos pueden usar tal cual."*

> `[Avanzar]`

---

## Slide 12 — Nodos de ejecución (~1 min 15 s)

*"Empecemos por los nodos de ejecución, que son las hojas del árbol: los que efectivamente interactúan con el robot."*

**Action** `[rectángulo]`

*"El nodo action encapsula una operación concreta y potencialmente prolongada: seguir la línea, rotar buscando una tarjeta, mostrar el feedback en el anillo de LEDs. Devuelve SUCCESS si la tarea se completó, FAILURE si es imposible completarla, y se mantiene en RUNNING mientras está en curso a lo largo de varios ticks. Es el único nodo que actúa sobre el mundo."*

**Condition** `[elipse]`

*"El nodo condition evalúa un predicado instantáneo sobre el contexto —'¿estoy viendo un tag de parada?', '¿este tag es el que esperaba en esta posición?'— y devuelve SUCCESS o FAILURE. Nunca devuelve RUNNING y no guarda estado: se resuelve en el mismo tick. Es lo que le da al árbol la información para decidir por dónde seguir."*

**Decorator** `[rombo, un único hijo]`

*"El decorator es un nodo con un único hijo que modifica su comportamiento o su resultado, sin modificar al hijo. Se lo puede pensar como un envoltorio con una política.*

*En nuestro módulo implementamos tres:*

- ***repeater**, que repite la ejecución de su hijo n veces —o indefinidamente— reiniciándolo entre repeticiones;*
- ***interrupt**, que representa un evento prioritario: evalúa una condición externa y, si se cumple, dispara el nodo asociado a esa interrupción;*
- ***interruptible**, que envuelve un nodo principal con una lista de interrupciones y permite abortar su ejecución de forma preventiva si alguna se activa.*

*Estos dos últimos son un agregado nuestro sobre el modelo clásico, pensados para un entorno reactivo. Son los que le permiten al robot cortar lo que está haciendo cuando pasa algo más importante: en nuestro caso, que lo levanten del tablero, que pierda la línea, o que se venza el tiempo buscando una tarjeta."*

> `[Avanzar]`

---

## Slide 13 — Nodos de control (~1 min 45 s)

*"Los nodos de control no ejecutan nada: coordinan. Tienen N hijos y lo único que definen es en qué orden se ejecutan y con qué criterio se combina el resultado."*

**Sequence** `[la flecha]`

*"El sequence es un AND. Ejecuta a sus hijos de izquierda a derecha y avanza al siguiente solo si el anterior devolvió SUCCESS. Si un hijo devuelve FAILURE o RUNNING, se detiene ahí y propaga ese estado a su padre. Devuelve SUCCESS solo si todos tuvieron éxito. Sirve para encadenar pasos: seguir la línea, detectar la parada, escanear la tarjeta, validarla, mostrar el resultado, retomar el recorrido."*

**Fallback / Selector** `[el signo de pregunta]`

*"El fallback —o selector— es el dual: un OR. Intenta a sus hijos en orden hasta que uno devuelve SUCCESS o RUNNING, y devuelve FAILURE solo si fallaron todos. Sirve para alternativas y contingencias: en nuestro árbol es el que decide, en cada ciclo, si lo que hay adelante es un tag de parada, un tag de fin, o ninguno de los dos y hay que seguir avanzando."*

**Parallel** `[la doble flecha]`

*"El parallel ejecuta a todos sus hijos en el mismo tick, en lugar de uno por vez, y combina los resultados según una política de umbral: por ejemplo, devuelve SUCCESS cuando M de sus N hijos tuvieron éxito. Se usa típicamente para correr una acción y su monitoreo al mismo tiempo. Lo mencionamos porque forma parte del modelo clásico; en nuestro módulo la reactividad la resolvimos con los decoradores de interrupción, que resultaron más simples y más baratos en memoria para un ESP32."*

**Con memoria vs. sin memoria**

*"Un punto que vale la pena aclarar, porque cambia por completo el comportamiento del árbol, es qué pasa en el tick siguiente cuando un hijo quedó en RUNNING.*

*Un nodo **con memoria** recuerda en qué hijo se quedó: en el próximo tick retoma directamente desde ese hijo, sin volver a evaluar a los anteriores. Es más eficiente y es lo natural cuando los pasos ya validados no van a cambiar, pero el árbol pierde reactividad frente a condiciones que quedaron atrás.*

*Un nodo **sin memoria** —reactivo— vuelve a empezar desde el primer hijo en cada tick y re-evalúa todas las condiciones previas. Cuesta más, pero garantiza que si una condición dejó de cumplirse, la rama se abandona inmediatamente.*

*En nuestra implementación los compositores son **con memoria**: cada nodo guarda el índice del hijo activo, y por eso una estación ya validada no se vuelve a validar en el tick siguiente. La reactividad la conseguimos por otra vía: los decoradores interrupt e interruptible, que se evalúan antes que la rama principal y pueden abortarla. Así pagamos re-evaluación solo donde hace falta —las condiciones de falla— y no en todo el árbol, que en un ESP32 no es un detalle menor."*

> `[Avanzar]`

---

## Slide 14 — El subárbol de la estación (~2 min)

> `[Primera imagen de la serie: el árbol completo, sin ningún nodo resaltado.]`

**Dónde estamos parados**

*"Con estos nodos armamos el árbol que gobierna toda la actividad: el modo de configuración por Bluetooth, la espera del tag de inicio, el ciclo de recorrido y la recuperación ante fallas. En lugar de mostrarlo entero, quiero mostrarles en detalle una rama, que es la parte central de la actividad: lo que hace el robot cuando llega a una estación. Arriba van a ver al robot, y abajo el subárbol correspondiente, con el nodo activo resaltado.*

*Este subárbol cuelga del ciclo de recorrido. Ese ciclo, en cada vuelta, hace dos cosas: avanza siguiendo la línea unos ticks, y después cambia la cámara a modo tags y le pregunta a un fallback qué hacer con lo que ve. La primera alternativa de ese fallback es esta rama."*

**El nodo raíz**

*"Arriba de todo hay una **sequence** —la flecha— y esa estrellita indica que es una sequence **con memoria**: recuerda en qué hijo se quedó. Sus cinco hijos se ejecutan de izquierda a derecha, y solo se avanza al siguiente si el anterior devolvió SUCCESS. En las próximas slides van a ver los nodos pintados de naranja cuando están en RUNNING y de verde cuando devuelven SUCCESS."*

**Los cinco hijos, uno por uno**

*"El primero es una **condition**: 'stop tag seen'. Es la única azul, porque es la única que no actúa: solo mira si entre los tags que la cámara acaba de detectar está el tag de parada. Y acá aparece un patrón muy típico de los Behavior Trees: poner una condición como primer hijo de una secuencia la convierte en la **guarda** de toda la rama. Si no hay tag de parada, devuelve FAILURE, la secuencia entera falla en el primer paso y el fallback de arriba pasa a probar la siguiente alternativa. O sea: el robot no 'decide' entrar acá; entra si y solo si se cumple la guarda.*

*El segundo es **scan checkpoint**: la acción que hace el trabajo. Detiene el robot y lo pone a rotar en sentido horario buscando el tag de la tarjeta que el grupo dejó en la estación. Mientras busca devuelve RUNNING, tick tras tick. Cuando lo encuentra, ignora los tags de control y toma solo el de la tarjeta, y lo compara contra la posición actual de la secuencia que el docente configuró desde la web. Si pasa demasiado tiempo sin encontrar nada, devuelve FAILURE.*

*El tercero es **checkpoint led**: la retroalimentación. Primero hace girar una luz blanca alrededor del anillo —es el estado 'pensativo', que agregamos por recomendación de las maestras para darle tiempo al grupo a entender qué está pasando— y recién después enciende el anillo entero en verde o en rojo.*

*El cuarto es **align stop**: después del feedback el robot quedó mirando hacia la tarjeta, no hacia la línea. Esta acción lo hace rotar en sentido antihorario hasta volver a encuadrar el tag de parada, que es su referencia para saber que está de nuevo alineado con el recorrido.*

*Y el quinto es **resume line**: devuelve la cámara a modo seguimiento de línea y deja correr unos ticks de estabilización antes de retomar la marcha.*

*Fíjense que los cinco son piezas chicas, con una sola responsabilidad. Ninguna sabe de las otras: la que escanea no sabe que después vienen las luces, y la que alinea no sabe que después se retoma la línea. Ese orden lo impone la sequence, no los nodos.*

*Vamos a recorrerlo paso a paso."*

> `[Avanzar]`

---

## Slide 15 — La guarda pasa (~25 s)

> `[Condition en verde, sequence y scan checkpoint en naranja.]`

*"Acá la cámara ya vio el tag de parada —se alcanza a ver en la pantallita del robot—. La condition devuelve SUCCESS, y por primera vez la secuencia habilita al segundo hijo: scan checkpoint se pone en RUNNING. El robot frena y empieza a rotar."*

## Slide 16 — Escaneando (~25 s)

> `[Solo scan checkpoint en naranja; la condition ya no está marcada.]`

*"Este es el tick siguiente, y acá se ve la memoria de la que hablábamos. Miren que la condition ya no está pintada: la secuencia no la vuelve a evaluar, retoma directo en el hijo que había quedado corriendo. El robot sigue rotando, buscando el tag de la tarjeta, y esto se repite durante muchos ticks."*

## Slide 17 — Tarjeta encontrada (~20 s)

> `[Scan checkpoint en verde, checkpoint led en naranja.]`

*"Encontró el tag de la tarjeta. Lo compara con la posición actual de la secuencia configurada, guarda el resultado y devuelve SUCCESS. La sequence avanza al tercer hijo: el feedback."*

## Slide 18 — El robot piensa (~20 s)

> `[Solo checkpoint led en naranja; anillo blanco girando.]`

*"Este es el estado 'pensativo': la luz blanca gira alrededor del anillo. Técnicamente no hace falta —la validación ya está hecha—, existe puramente por una razón pedagógica: darle al grupo un segundo para entender que el robot está evaluando lo que acaba de leer, antes de dar el veredicto."*

## Slide 19 — El veredicto (~20 s)

> `[Checkpoint led en verde, align stop en naranja.]`

*"Se mostró el resultado y la acción devuelve SUCCESS. Ahora arranca align stop: el robot quedó mirando la tarjeta, así que tiene que darse vuelta."*

## Slide 20 — Realineando (~15 s)

> `[Solo align stop en naranja.]`

*"Rota en sentido antihorario, buscando de nuevo el tag de parada, que es su referencia para saber que volvió a quedar alineado con el recorrido."*

## Slide 21 — De vuelta a la línea (~20 s)

> `[Align stop en verde, resume line en naranja.]`

*"Reencontró el tag de parada: SUCCESS. Entra el último hijo, resume line, que devuelve la cámara a modo seguimiento de línea y deja unos ticks de estabilización para que el robot no arranque con una lectura todavía inestable."*

## Slide 22 — Estación completa (~40 s)

> `[Resume line en verde y, sobre todo, la sequence raíz en verde.]`

*"Con los cinco hijos en SUCCESS, la sequence devuelve SUCCESS a su padre. La estación está cerrada: el ciclo de recorrido se completa y vuelve a empezar —seguir la línea, mirar tags— hasta la próxima estación. Y como la rama terminó, su memoria se reinicia: en la próxima vuelta la guarda se vuelve a evaluar desde cero.*

***Y si algo sale mal:** si scan checkpoint no encuentra la tarjeta dentro del tiempo límite, devuelve FAILURE. La sequence se corta ahí —no ejecuta las luces, ni la alineación, ni el retomar— y como el árbol entero está envuelto en un interruptible, se dispara la rama de recuperación: el robot se detiene, parpadea en amarillo y espera a que se lo reubique y se le muestre el tag de inicio, para retomar con el progreso intacto, o el de reinicio, para empezar de cero."*

> `[Avanzar al video]`

---

## Slide 23 — El recorrido real (~40 s)

> `[Reproducir el video. Acá hablar poco: ya está todo explicado, esto es la confirmación en velocidad real.]`

*"Y así se ve todo eso a velocidad real: el robot llega a la estación, se detiene, busca la tarjeta, la valida y retoma el recorrido.*

*Lo importante es que ninguno de esos nodos sabe qué viene después. Cambiar la actividad —agregar un paso, cambiar el orden, sumar una bifurcación en el recorrido— es reacomodar el árbol, no reescribir la lógica."*

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
