## Slide 6 — Percepción: HuskyLens

Como dijimos en los objetivos, lo primero que tuvimos que resolver fue qué cámara usar. Probamos varias opciones y nos quedamos con la HuskyLens.

¿Por qué está y no otra?

Primero, porque el procesamiento de imagen ya viene integrado: no nos manda una imagen cruda, sino datos ya procesados. Eso era clave, porque implementar eso nos hubiera llevado un montón de tiempo, y no era el foco del proyecto.

Segundo, trae varios algoritmos ya implementados: reconoce caras, colores, objetos, tags, y puede seguir líneas u objetos.

Tercero, es fácil de entrenar. Desde de la cámara le enseñas los tags que necesitás o la línea que tiene que seguir, sin la necesidad de entrenar un modelo.

Cuarto, se comunica por I2C, igual que el sensor ya integrado al robot que se ubica debajo, que usamos como sensor de proximidad. Así que reutilizamos el mismo módulo I2C ya implementado en lugar de armar uno nuevo.

Por último, hay una librería oficial de Arduino que usamos como punto de partida. La migramos a C y la conectamos con el módulo I2C que ya teníamos. Este nuevo módulo C vive en LuaRTOS, al igual que los módulos que se encargan de manejar el resto de los periféricos.

Para nuestra actividad tuvimos que enseñarle dos cosas a la cámara: la línea negra del tablero, y los tags que se usan durante el recorrido.

De todos los algoritmos que trae la HuskyLens, nosotros usamos solo dos: seguimiento de línea y reconocimiento de tags.

## Slide 7 — Seguimiento de línea

Este algoritmo es lo que utilizamos para hacer que Robotito avance por el tablero.

En este modo, la camara intenta dibujar una flecha recta que matche lo mejor posible con la liena que tiene delante. Esta linea tiene que estar previamente aprendida por la camara

De esta flecha, recibimos 3 valores:

- Las coordenadas del punto de inicio 
- Las coordenadas del punto fin de la flecha
- Su identificador

En la imagen de la izquierda, cuando el robot está bien alineado con un tramo recto, el punto de inicio y el de fin de la felcha quedan alineados. Ahora, en la imagen de la derecha aparece una curva, pero la cámara tambien retorna los datos de una flecha recta. Es importante entender que la cámara siempre nos da una recta, aunque la imagen que procesa la cámara contenga una curva.

Sabiendo esto, para que el robot pueda seguir el recorrido sin perderse, lo que tenemos que lograr es intentar siempre estar en el caso 1, que la flecha este centrada en la imagen y que el punto de origen y de desitno esten alineados verticalmente 


## Slide 8 — Seguimiento de línea (fórmula de error y tabla de giro)

Para lograr esto, utilizamos la informacion que nos retorna la camara de cada flecha y calculamos dos errores.

El error de posición, que lo definimos como distancia entre la punta de la flecha (xT) y el centro de la imagen. Nos dice para qué lado y qué tan lejos del camino está el centro.

El error de dirección es la diferencia entre xT y xO. Nos dice hacia dónde está apuntando la línea, así anticipamos una curva antes de que el robot se desvíe del todo.

Estos dos errores los combinamos en uno solo, ponderado: 60% posición y 40% dirección. 

Con ese error total usamos una tabla de decisión: si el error es chico —entre 0 y 30— el robot sigue derecho a velocidad normal. A medida que crece el error, el giro se hace más fuerte. Si pasa de 120, además de girar más fuerte, el robot frena un poco, porque una curva tan cerrada necesita más tiempo para poder procesar más líneas y evitar perderse. Cuando un error es positivo corregimos la direccion del robot a la derecha, y cuando el error es negativo, a la izquierda. Hay una tabla analoga a esta para errores negativos 

Tanto los pesos de los errores, como los rangos que se ven en la tabla los decidimos en base a varias prubeas y muchos errores. Con estos valores econtramos el mejor funcionamiento del robot 

## Slide 9 — Detección de tags

El otro algoritmo que usamos es detección de tags. Esto es lo que le permite a Robotito reconocer las estaciones de parada, los tags en las tarjetas de estacion, y otros tag de control que utiliazmos en la actividad.

En vez de una flecha, la camara nos devuelve un "bloque". Los valores que obtenemos son:
- Las coordenadas del centro del bloque
- su ancho 
- su alto
- un identificador único. 

En base a este id unico Robotito sabe qué está viendo: si es una parada, el tag de inicio, el de fin, el de reset, o una tarjeta de la secuencia y ejecuta una accion determinada

Como ven en la imagen, cada señal —por ejemplo el cartel de "PARE"— tiene en realidad dos copias o mas del mismo tag. Esto lo decidimos después de varias pruebas: si el robot no llega a leer uno, tiene una segunda chance con el otro, así evitamos errores de lectura por el ángulo o la distancia.

Iterando entre estos dos modos durante todo el recorrido, logramos que el robot avance por el tablero siguiendo la línea, y pueda detectar los diferentes tags que aparecen durante el recorrido.
