## Diapositiva 15 — Portal Web y comunicación Bluetooth (3 min)

A partir de lo que vimos en la experimentación, apareció otro problema importante: **¿cómo hacemos para que un docente pueda definir una actividad sin tener que modificar el código?**

No queríamos que Robotito tuviera siempre una única secuencia fija, ni que fuera necesario conectarlo a una computadora y modificar su firmware cada vez que se quisiera preparar una actividad diferente.

Para resolver esto desarrollamos un **configurador web**, pensado para simplificar la preparación de las actividades.

Desde el portal, el docente puede comenzar una actividad desde cero o seleccionar una de una biblioteca de actividades previamente definidas. Luego puede registrar las tarjetas físicas que va a utilizar, asignarles nombres más amigables y construir una o varias secuencias que Robotito deberá validar durante el recorrido.

Por ejemplo, supongamos que queremos trabajar con los niños formando la palabra **CASA**.

En ese caso necesitamos cuatro tarjetas: C, A, S y A. Pero físicamente las dos tarjetas que representan la letra A tienen AprilTags diferentes, porque cada tag tiene un identificador propio.

Desde el configurador podemos declarar que esos dos tags son **intercambiables**. Es decir, aunque físicamente sean distintos, para la actividad ambos representan el mismo concepto. De esta manera, cualquiera de esas tarjetas puede aparecer en cualquiera de las posiciones donde esperamos una A.

El portal también incorpora un lector de AprilTags utilizando la cámara de una computadora o un celular.

Una vez identificado, el docente puede asociarlo a un nombre como *Semilla*, *Planta* o simplemente una letra.

De esta forma conseguimos abstraer al docente de los identificadores numéricos que utiliza internamente Robotito.

Una vez que la actividad está preparada en el portal, necesitamos enviarla al robot.

Para esto utilizamos **Bluetooth Low Energy**, o BLE.

Es importante aclarar que Robotito **ya contaba con un módulo Bluetooth**. Nuestra contribución no fue reemplazarlo, sino **extenderlo** para que pudiera recibir y administrar estas nuevas configuraciones. La tesis plantea explícitamente esta extensión del canal BLE existente para enviar, almacenar y exponer las secuencias a la lógica de ejecución.

Agregamos la posibilidad de enviar las secuencias desde el portal, almacenarlas en el robot y luego exponerlas a la lógica escrita que controla la actividad.

Para poder enviar las secuencias, el robot debe estar en el modo configuración, entra en este modo mostrandole la tarjeta de control. Fuera de este modo, el bluetooth no se anuncia. 

Además, estas secuencias se guardan en **NVS**, que es el almacenamiento no volátil del ESP32.

En términos simples: si configuramos una actividad, apagamos Robotito y lo volvemos a prender, la secuencia sigue disponible. No es necesario volver a configurar el robot antes de cada ejecución.

En conjunto, el portal y esta extensión del módulo Bluetooth nos permiten cambiar completamente una actividad **sin modificar el firmware del robot**.

---

## Diapositiva 16 — Conclusiones

Llegando al final del proyecto, hay tres conclusiones principales que queremos destacar.

La primera tiene que ver con nuestra experiencia desarrollando en **robótica**.

Ninguno de nosotros tenía experiencia previa trabajando con robots y todos venimos principalmente del desarrollo de software web, donde normalmente tenemos bastante más control sobre el entorno de ejecución.

En robótica descubrimos que aparecen muchas variables físicas que afectan directamente al comportamiento del software: la iluminación, el nivel de batería, la superficie sobre la que se mueve el robot, entre otras.

Eso hizo que el proceso de prueba fuera muy distinto al que estábamos acostumbrados.

La segunda conclusión surge de la **validación en aula y la recepción de los usuarios**.

Hasta llevar Robotito a las instituciones teníamos muchas decisiones técnicas que funcionaban correctamente en nuestras pruebas, pero no sabíamos cómo iban a comportarse dentro de una dinámica real de clase.

Y, sobre todo, las pruebas nos mostraron necesidades que no habíamos considerado solamente desde el desarrollo.

Un ejemplo muy claro de esto fueron las funciones de **pausa y reinicio**.
Y la decisión de cambiar de lectura de color a lectura de tags.

Entonces, la experimentación no solamente nos permitió validar inicialmente la propuesta, sino que también modificó parte del diseño final.

La tercera conclusión tiene que ver con lo que el proyecto deja como **base reutilizable para futuros trabajos sobre Robotito**.

Dos de los principales componentes que construimos fueron diseñados de forma modular y reutilizable: el módulo de **HuskyLens** y la biblioteca de **Behavior Trees**.

El módulo de HuskyLens abstrae la complejidad de la comunicación con la cámara y expone una interfaz mas simple. Por otro lado, la biblioteca de Behavior Trees permite construir nuevos comportamientos combinando nodos sin tener que modificar el núcleo de la biblioteca.

Esto deja una base concreta para que futuros estudiantes de la Facultad que trabajen con Robotito puedan reutilizar esas piezas en otras actividades.


## Diapositiva 17 — Trabajo futuro

A partir de lo aprendido también identificamos varias líneas claras para continuar el proyecto.

La primera es incorporar **bifurcaciones en el circuito**.

Actualmente Robotito sigue un recorrido lineal. Incorporar intersecciones permitiría que el robot tome caminos diferentes dependiendo de decisiones o validaciones anteriores. Esto permitiría introducir conceptos como condicionales de una forma tangible.

La segunda línea es agregar **retroalimentación sonora**.

Actualmente todo el feedback se realiza utilizando el anillo LED. Complementarlo con sonidos diferentes para aciertos, errores o finalización podría facilitar el seguimiento de la actividad, especialmente con niños pequeños o grupos numerosos.

La tercera es utilizar la **tarjeta SD de HuskyLens para respaldar los modelos entrenados**.

La cámara almacena internamente la información que aprende sobre líneas y tags, pero esa configuración puede borrarse accidentalmente desde su propia interfaz. Guardar una copia en una tarjeta SD permitiría restaurarla rápidamente sin tener que volver a entrenar la cámara.

Finalmente, la experimentación también mostró oportunidades para **mejorar los materiales físicos**.

Por ejemplo, algunas partes de las calles pueden despegarse con el uso y las tarjetas no tienen actualmente un mecanismo físico que garantice siempre la misma posición y orientación frente a la cámara.

Una evolución natural sería desarrollar un tablero más robusto y una carcasa que integre mejor la cámara, proteja los componentes y facilite el uso cotidiano de Robotito en el aula. 
