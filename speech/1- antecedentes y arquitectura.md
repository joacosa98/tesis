DIAPO ANTECEDENTES
--------------------
Antes de explicar nuestro proyecto, vamos a comentarles qué es Robotito y qué elementos tenía de base.

Robotito es un robot educativo de hardware y software libre, pensado para acercar a niños de nivel inicial y primaria a la robótica y al pensamiento computacional.

Robotito estaba equipado con un sensor de color y distancia en la parte inferior, sensores de distancia alrededor del robot, tres ruedas omnidireccionales para desplazarse en distintas direcciones y un anillo de LEDs para dar feedback visual. 

El sensor de color se utilizaba para reconocer tarjetas en el piso, y los sensores de distancia se usaban en otra actividad para detectar objetos cercanos.

--------------------

DIAPO ARQUITECTURA


Esta es la arquitectura existente de Robotito sobre la que partimos para desarrollar nuestro trabajo.

En la capa más baja está el ESP32 junto con su SDK oficial ESP-IDF, que permite a las capas superiores comunicarse con el hardware. Por encima se encuentra Lua-RTOS donde viven los módulos escritos en C que se encargan por ejemplo de manejar los motores o el anillo led. Finalmente, en la capa superior están los scripts en Lua, donde se define la lógica del comportamiento del robot usando los modulos expuestos por la capa anterior.