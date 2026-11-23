# **Práctica 3**

## Controlador Máquina Expendedora 

## **Objetivos:**

En esta práctica se aborda el diseño e implementación del controlador software para una máquina expendedora automática basada en la plataforma Arduino UNO y en los distintos sensores y actuadores disponibles en el kit de laboratorio de la asignatura.

El sistema deberá responder a eventos físicos en tiempo real, interactuar con el usuario a través de una pantalla LCD, pulsadores, indicadores LED y un joystick analógico, a la vez que monitoriza magnitudes ambientales (temperatura y humedad) y de proximidad (distancia mediante ultrasonidos) de manera no bloqueante.

## **Componentes Hardware:**

He implementado todos los componentes obligatorios de la práctica:

• Arduino UNO  
• Pantalla LCD  
• Módulo Joystick  
• Sensor de Temperatura y Humedad DHT11.  
• Sensor de Distancia por Ultrasonidos  
• Pulsador / Botón externo.  
• 2 LEDs estándar.

Sumados a las resistencias para los LEDs y un potenciómetro. Este último lo he implementado para poder ajustar el brillo de las letras de la pantalla LCD, evitando así que el brillo fuera demasiado bajo o, por otro lado, que se viesen los rectángulos del display sobre las letras.

## **Esquemático del Circuito:**

Este es el esquemático que he hecho en Fritzing del circuito.

<img width="2422" height="1595" alt="Esquematico" src="https://github.com/user-attachments/assets/e01b8871-bc26-4b44-9337-d74f7f2a447a" />


## **Fotos del Circuito:**

Estas son las imágenes del circuito. A simple vista es difícil distinguir todas las conexiones, pero sumado al esquemático puede interpretarse (He hecho los colores de los cables del esquema iguales a los del circuito real).

## **Descripción de la solución:**

Como se sugiere/requiere en el enunciado, he utilizado ArduinoThreads, Interrupciones (Hardware), Watchdog y diferentes estados.  
Empezando por lo más sencillo de implementar, he aplicado Watchdog, aplicando un timeout de 2 segundos y reiniciándolo al principio de cada iteración del bucle principal. Lo inicializo al final del setup(), pero por si acaso, al principio lo desactivo. Esto lo hago debido a que podría entrar en un bucle de reinicios si al reiniciarse conserva el contador anterior. Es improbable pero es una prevención.

En cuanto al funcionamiento, he creado 5 threads diferentes. Cada uno es un callback() para los diferentes componentes del circuito. Tenemos:

* Un callback para el LED inicial \-\> callback\_led() cada 1000 ms  
* Un callback para la pantalla LCD \-\> callback\_lcd() cada 200 ms  
* Un callback para el ultrasonidos \-\> callback\_distance() cada 100 ms  
* Un callback para el sensor DHT \-\> callback\_dht() cada 2000 ms  
* Un callback para el joystick \-\> callback\_joystick() cada 50 ms


Estos periodos son los siguientes debido a que necesitamos que el led parpadee en tramos de 1 segundo. En condiciones normales la temperatura y humedad casi no varían de un instante a otro, por tanto actualizarlo cada 2 segundos es suficiente. Para que no se perciba retraso el joystick y la pantalla tienen periodos más bajos, al igual que la distancia, para que no tarde en detectar si se ha acercado alguien o si se ha ido.

En cuanto a los estados del sistema he creado los 3 estados propuestos, además de un estado de inicialización, por tanto tenemos STARTUP, WAITING, SERVICE\_MODE, ADMIN. Aunque los nombres ya lo dan a entender, STARTUP es el estado de inicio, en el que el led parpadea y la pantalla imprime “Cargando…”. WAITING es la espera hasta que el sensor detecte algo a menos de un metro, SERVICE\_MODE es el modo de servicio para el cliente y ADMIN el modo administrador. Algunos estados cuentan con flags (que funcionan como estados dentro de los estados principales). Por ejemplo, SERVICE\_MODE tiene un flag showingDHT que se activa al pasar de WAITING a SERVICE\_MODE, ya que debe enseñar los valores DHT durante 5 segundos.

Lo que hace mi código es lo siguiente. Cada callback cuenta con condiciones que comprueban el estado. Dependiendo del estado actual, el mismo callback puede hacer cosas distintas. Pero no todos los callbacks dependen del estado ni todos los threads están activos en todo momento. Los callbacks de distancia y DHT siempre hacen lo mismo, calcular la distancia y medir la temperatura y humedad, pero sólo se activan cuando son necesarios. La distancia la necesitamos en el modo de espera (para saber si llega alguien), el modo servicio (para saber si la persona se ha ido), y en el modo administrador, pero sólo cuando se selecciona la opción de ver la distancia. 

Por tanto con threadDistance.enabled al principio del bucle, seguido de las condiciones que acabo de enumerar, cuando llamamos al controlador, sólo se activa si es necesario. Así con el resto de threads, aunque otros, como la pantalla LCD, cambian lo que hacen dependiendo del estado. Si está en STARTUP callback\_lcd escribe “Cargando…”, pero si está en SERVICE\_MODE, con el flag preparingProduct, imprime “Preparando” y el nombre de la bebida (Que se obtiene de una variable global productName\[productIndex\], la cual es un array cuyo índice incrementa o decrementa con el callback del joystick)

Para las operaciones que requieren un tiempo concreto, como el preparado del producto (cuyo tiempo es pseudoaleatorio como dice el enunciado, ya que lo calculo en cada iteración usando random()), utilizo variables unsigned long donde guardo con millis() el momento de inicio y con una condición millis() \- momento\_inicio \<  t, hago que terminen tras t milisegundos. 

Esto último lo utilizo también para darle un segundo de gracia al sensor ultrasonidos, ya que hay veces que a pesar de tener algo cerca, devuelve valores erróneos o 0\. Por tanto, para que si la persona está, evitar que se resetee el modo servicio al pensar que se ha alejado, guardo en una variable el momento con la última medida de distancia en el rango deseado, y si un segundo después no tiene una medida válida, ahí si que da por hecho que la persona se ha ido.

En cuanto a las interrupciones, las he utilizado para el botón externo. En vez de comprobar si ha sido pulsado una y otra vez en el bucle principal, lo hacemos con interrupciones.  Conectamos el botón al pin 2, que tiene las interrupciones por hardware INT0, y en el setup() lo asociamos a una función isr\_button, que se activa aunque no lo controlemos con software, gracias a la interrupción hardware. Cuando se detecta esta interrupción, la función guarda el instante y al detectarse de nuevo (al soltar) vuelve a hacerlo. El bucle principal comprueba la diferencia de los dos instantes. Si está en el rango 2-3 y se está en modo servicio se reinicia ese modo. En caso de que la diferencia sea de 5 segundos o más, se entra o se sale del modo Admin.

## **Otras decisiones:**

Para hacer que el LED2, que he puesto de color verde, se ilumine poco a poco primero probé con un crecimiento lineal, pero me parecía poco fluido para la vista y tras investigar vi que el más adecuado para el ojo humano es un crecimiento cuadrático: 

ledIntensity \= 50 \* (progress \* progress);

Puse como 50 el tope ya que más intensidad deslumbraba, además de que a partir de cierta intensidad, no se aprecia tanto el aumento. Además utilicé para este LED el pin 9, ya que cuenta con PWM, que es necesario para poder regular la intensidad de esta forma.

En cuanto a la pantalla LCD, he preferido no usar la función clear(), y he optado por sobrescribir con print() y los espacios necesarios, además de escribir por líneas en vez de seguido, para evitar mensajes partidos a la mitad. He decidido hacerlo así ya que si cada 200 ms llamamos a print, se genera un poco de parpadeo, sobre todo en datos reactivos como la distancia. 
