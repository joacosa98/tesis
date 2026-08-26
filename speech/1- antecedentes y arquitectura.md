DIAPO ANTECEDENTES
--------------------
Antes de explicar nuestro trabajo, es importante entender qué es Robotito y con qué capacidades ya contaba.

**Robotito es una plataforma educativa de software libre y hardware abierto**, pensada para acercar a niños de nivel inicial y primaria a la robótica y al pensamiento computacional.

Cuando comenzamos el proyecto, Robotito ya tenía un **sensor de color** en la parte inferior, **sensores de distancia** alrededor del robot, **tres ruedas omnidireccionales** para desplazarse en distintas direcciones y un **anillo de LEDs** para dar feedback visual. 

El sensor de color se utilizaba para reconocer tarjetas en el piso, y los sensores de distancia se usaban en otra actividad para detectar objetos cercanos.

Estas capacidades permitían realizar distintas actividades, pero también limitaban cuánto podía percibir Robotito de su entorno. 

Nuestro trabajo parte justamente de esa limitación: **extender la capacidad de percepción del robot para poder plantear actividades más complejas**.

--------------------

DIAPO ARQUITECTURA


Esta es la arquitectura existente de Robotito sobre la que partimos para desarrollar nuestro trabajo.

En la capa más baja está el ESP32 junto con su SDK oficial ESP-IDF, que permite a las capas superiores comunicarse con el hardware. Por encima se encuentra Lua-RTOS donde viven los módulos escritos en C que se encargan por ejemplo de manejar los motores o el anillo led. Finalmente, en la capa superior están los scripts en Lua, donde se define la lógica de las actividades y el comportamiento del robot usando los modulos expuestos por la capa anterior.

