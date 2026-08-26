# Speech — Slide 6: "Actividad"

Si bien la actividad fue el ultimo paso, creemos que es util mostrarle la actividad que construimos antes de comenzar a hablar del resto de los objetivos, para que entiendan como funciona y que partes tiene. Asi, cuando mas adelante se mencionen las diferentes partes de la activdad, ya sepan de que se trata  

**Las partes.**
La actividad tiene 3 elementos imporantes:
1- Robotito, que ahora cuenta con la cámara integrada. 
2- Un tablero modular con un recorrido marcado por una línea continua, que tiene distintas estaciones de parada señalizadas con tags, como si fueran carteles de PARE. 
3 - Un conjunto de tarjetas que representan una secuancia, que por ejemplo, puede ser una historia, y que como toda secuencia, tiene un numero ordenado de pasos. Cada una de estas tarjetas representa un paso de la secuencia, tiene un tag unico que la cámara puede leer, y un espacio libre para agregar algo que representa ese paso de la secuencia, por ejemplo, un dibujo.

El objetivo de la actividad es colocar las tarjetas en las estaciones de parada, y que el robot valide el orden de la secuencia mientras va avanzando por el tablero

Robotito arranca desde el punto de inicio y empieza a recorrer la línea. Al llegar a una estación, frena, gira sobre sí mismo buscando la tarjeta, la lee y verifica si corresponde a la posición correcta dentro de la secuencia. Si corresponde, se prende el anillo led en verde, y sino, en rojo. Después retoma el camino y repite este proceso en cada estación hasta llegar al final del recorrido, donde da un feedback general de cómo le fue en toda la secuencia utilizando tambien las luces led

