## Slide 6 — Percepción: HuskyLens

Como dijimos en los objetivos, lo primero que tuvimos que resolver fue qué cámara usar. Probamos varias opciones y nos quedamos con la HuskyLens.

¿Por qué está y no otra?

Primero, porque el procesamiento de imagen ya viene integrado: no nos manda una imagen cruda, sino datos ya procesados. Eso era clave, porque implementar eso nos hubiera llevado un montón de tiempo, y no era el foco del proyecto.

Segundo, trae varios algoritmos ya implementados: reconoce caras, colores, objetos, tags, y puede seguir líneas u objetos.

Tercero, es fácil de entrenar: desde la propia interfaz de la cámara le enseñas los tags que necesitás o la línea que tiene que seguir, sin la necesidad de entrenar un modelo. Otro ahorro grande de tiempo.

Cuarto, se comunica por I2C, igual que el sensor ya integrado al robot que se ubica debajo, que usamos como sensor de proximidad. Así que reutilizamos el mismo módulo I2C ya implementado en lugar de armar uno nuevo.

Por último, hay una librería oficial de Arduino que usamos como punto de partida: la migramos a C y la conectamos con el módulo I2C que ya teníamos. Este nuevo módulo C vive en LuaRTOS, al igual que los módulos que se encargan de manejar el resto de los periféricos.

Para nuestra actividad tuvimos que enseñarle dos cosas a la cámara: la línea negra del tablero, y los tags que se usan durante el recorrido. Estos tags se dividen en dos grupos: los de control —inicio, fin, parada, reset, e inicio y fin de configuración— y los de las tarjetas de estación. Los de control tienen los primeros IDs, y todo el resto queda para las tarjetas de estación.

De todos los algoritmos que trae la HuskyLens, nosotros usamos solo dos: seguimiento de línea y reconocimiento de tags.

## Slide 7 — Seguimiento de línea

Seguimiento de línea: esto es lo que hace que Robotito avance por el tablero.

Cuando la HuskyLens está en modo de seguimiento de línea, nos devuelve algo que nosotros llamamos "flecha". Los datos que recibimos de esta flecha son:

- Las coordenadas de inicio y las coordenadas de fin de la flecha
- Su identificador

Básicamente es un vector que marca hacia dónde va la línea dentro de lo que ve la cámara.

En la imagen de la izquierda, cuando el robot está bien alineado con un tramo recto, el punto de origen y el de destino quedan alineados, formando una flecha vertical. Ahora, en la imagen de la derecha aparece una curva, y la cámara nos devuelve exactamente lo mismo: un origen, un destino y su id. Es importante entender que la cámara siempre nos da una recta, aunque la imagen que procesa la cámara contenga una curva.

## Slide 8 — Seguimiento de línea (fórmula de error y tabla de giro)

¿Y qué hacemos con estos datos? El objetivo es que la línea esté siempre en el centro de la imagen, y que el origen y el destino estén alineados. Cuando eso pasa, sabemos que el robot va bien encaminado. Para esto calculamos errores que nos permiten decidir hacia dónde mover al robot, para que se mantenga lo más centrado posible sobre la línea.

Con esos dos puntos —origen y destino de la flecha— calculamos dos errores.

El error de posición, que es la distancia entre la punta de la flecha (xT) y el centro de la imagen. Nos dice para qué lado y qué tan lejos del camino está el centro del robot.

El error de dirección es la diferencia entre xT y xO. Nos dice hacia dónde está apuntando la línea, así anticipamos una curva antes de que el robot se desvíe del todo.

Estos dos errores los combinamos en uno solo, ponderado: 60% posición y 40% dirección. ¿Por qué esos pesos? Prueba y error, no más. Fue la combinación que mejor nos funcionó.

Con ese error total usamos una tabla de decisión: si el error es chico —entre 0 y 30— el robot va derecho a velocidad normal. A medida que crece el error, el giro se hace más fuerte. Si pasa de 120, además de girar más fuerte, el robot frena un poco, porque una curva tan cerrada necesita más tiempo para poder procesar más líneas y evitar perderse.

## Slide 9 — Detección de tags

El otro algoritmo que usamos es detección de tags. Esto es lo que le permite a Robotito reconocer las estaciones del recorrido y las tarjetas que acción.

Acá el modo de la cámara cambia: en vez de una flecha, la HuskyLens nos devuelve un "bloque". Es un rectángulo con la posición del centro del tag, su ancho, su alto, y lo más importante: un identificador único. Cada tag físico tiene un ID distinto, y con eso Robotito sabe qué está viendo: si es una parada, el tag de inicio, el de fin, el de reset, o una tarjeta de la secuencia.

Como ven en la imagen, cada señal —por ejemplo el cartel de "PARE"— tiene en realidad dos copias del mismo tag. Esto lo decidimos después de varias pruebas: si el robot no llega a leer uno, tiene una segunda chance con el otro, así evitamos errores de lectura por el ángulo o la distancia.

Combinando estas dos capacidades —seguir la línea y leer tags— logramos que el robot avance por el tablero siguiendo la línea negra, y pueda detectar los diferentes tags de estación que aparecen durante el recorrido.
