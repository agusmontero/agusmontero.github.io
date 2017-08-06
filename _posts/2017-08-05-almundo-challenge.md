---
layout: post
title: "AlMundo Challenge - ECI 2017"
date: 2017-08-05
categories: challenge
tags: [tsp, integer-programming, challenge]
image:
---

La semana pasada participé en la competencia [AlMundo Challenge](https://almundo.com.ar/eci/contest)
organizada por [almundo.com](https://almundo.com.ar/) en el contexto de la [ECI2017](https://www.dc.uba.ar/events/eci/2017).

El problema me pareció interesante, y si bien el modelo con el que trabajé quedó en 4to puesto [(ranking)](https://almundo.com.ar/eci/ranking), me gustaría compartirlo como un ejemplo de aplicación de programación lineal entera.

<!--
One of the rewards of switching my website to [Jekyll](http://jekyllrb.com/) is the
ability to support **MathJax**, which means I can write LaTeX-like equations that get
nicely displayed in a web browser, like this one \\( \sqrt{\frac{n!}{k!(n-k)!}} \\) or
this one \\( x^2 + y^2 = r^2 \\). -->

# El Problema

La descripción completa del problema está [acá](https://almundo.com.ar/eci/contest).

En resumen, se tienen las coordenadas de 41011 ciudades, y un turista que debe volar a **todas** las ciudades, visitando cada una de ellas **exactamente una vez**. Además, en cada vuelo debe decidir qué agencia de viajes usar. Existen 4 agencias de viaje, y cada una ofrece descuentos si se cumplen determinadas condiciones. Por ejemplo, una de las agencias otorga un 35% de descuento en caso de volar con ella 3 veces consecutivas.

El objetivo del problema es decidir *en qué orden* se deben visitar las ciudades y *con qué agencia* en cada vuelo, buscando minimizar el costo total, teniendo en cuenta que todas las agencias tienen la misma tarifa: $0.01 por Km.

Una parte de este problema (*en qué orden*) está relacionada con otro problema muy famoso: [El problema del viajante de comercio](https://es.wikipedia.org/wiki/Problema_del_viajante), muy estudiado y para el cual existe [mucha literatura](http://www.math.uwaterloo.ca/tsp/index.html).

# Solución propuesta

Una de mis herramientas favoritas para atacar problemas de optimización es la [**programación lineal entera**](https://en.wikipedia.org/wiki/Integer_programming). Sin entrar en muchos detalles, la idea es construir un modelo matemático que describa las condiciones impuestas por el problema y la función objetivo que se quiere minimizar. Como su nombre lo indica, es importante que las relaciones entre las variables del modelo sean **lineales**.

Decidí partir el problema original en dos partes: 1) obtener un recorrido inicial sin tener en cuenta las agencias, y en consecuencia sin descuentos (el recorrido debe visitar todas las ciudades exactamente una vez y sea lo más barato posible); 2) buscar **la mejor** forma posible de asignar agencias a un recorrido dado.

Observar que esto es una *heurística*, dado que la solución óptima al problema original no necesariamente se obtiene asignando agencias al recorrido de menor costo, y muchas veces utilizar un recorrido de mayor costo podría inducir mayores descuentos (dadas las condiciones de las agencias), y en consecuencia un menor costo total.

## Etapa I: el recorrido

Para conseguir un recorrido inicial (dejando de lado las agencias), en primer lugar pensé en usar [Concorde](https://en.wikipedia.org/wiki/Concorde_TSP_Solver). Este programa permite obtener el recorrido óptimo siempre y cuando la cantidad de ciudades sea *razonable*. No logré que Concorde funcione para las 41011 ciudades, y opté por utilizar la heurística [Lin–Kernighan](https://en.wikipedia.org/wiki/Lin%E2%80%93Kernighan_heuristic), en particular la implementación que está [acá](http://www.math.uwaterloo.ca/tsp/concorde/downloads/downloads.htm).

El mejor recorrido que obtuve en esta etapa fue de **7754930.68** km.

## Etapa II: asignación
### Modelado

Si consideramos el recorrido de la etapa anterior y asignamos en todos los viajes la agencia D, tenemos un costo total de **$65924.31** que nos sirve como punto de partida.

Ahora queremos obtener la **mejor** forma de asignar las agencias. Para esto construí un modelo con las variables $$X_{a,e}$$ donde $$a \in \{A,B,C,D\}$$ y $$e$$ es un tramo del recorrido.

Si definimos:

$$X_{a,e} = \begin{cases} 1 & \mbox{si } a\mbox{ está asignada al tramo } e \\ 0 & \mbox{caso contrario} \end{cases}$$

es posible modelar la restricción ```todo tramo debe tener asignada una agencia``` como:

$$X_{A,e} + X_{B,e} + X_{C,e} + X_{D,e} = 1$$

para cada tramo $$e$$ en el recorrido.

Es necesario además modelar la función de costo que queremos minimizar. ```La agencia B aplica un descuento del 15% siempre que el tramo asignado tenga más de 200km```. Teniendo en cuenta esto, podemos modelar el descuento de la agencia B como:

$$ \sum_{e\in T} X_{B,e} * w_e * 0.15 * (w_e > 200) * 0.01 $$

donde $$T$$ es el recorrido, y $$w_e$$ es la distancia en kilómetros del tramo $$e \in T$$.

Por otro lado, ```la agencia C aplica un descuento del 20% siempre que el tramo anterior tenga asignada la agencia B```. Para modelar este descuento, una posibilidad es incorporar las siguientes variables:

$$Y_{e_i,e_{i+1}} = \begin{cases} 1 & \mbox{si } X_{B,e_i} = 1 \wedge X_{C,e_{i+1}} = 1 \\ 0 & \mbox{caso contrario} \end{cases}$$

para cada par de tramos $$(e_i,e_{i+1})$$ consecutivos en el recorrido.

La motivación de esta variable es poder saber cuándo hay una agencia B asignada antes que la agencia C, y así poder aplicar el descuento correspondiente. El descuento de la agencia C se modela entonces como:

$$ \sum_{1\leq i \leq 41010} Y_{e_i,e_{i+1}} * w_{e_{i+1}} * 0.20 * 0.01 $$

Además es necesario establecer linealmente la relación entre las variables $$Y_{e_i,e_{i+1}}$$, $$X_{B,e_i}$$ y $$X_{C,e_{i+1}}$$ para que cumpla con la definición antes propuesta:

$$Y_{e_i,e_{i+1}} \leq X_{B,e_i}$$

$$Y_{e_i,e_{i+1}} \leq X_{C,e_{i+1}}$$

$$Y_{e_i,e_{i+1}} + 1 \geq X_{B,e_i} + X_{C,e_{i+1}}$$

para cada $$i$$ tal que $$1\leq i \leq 41010$$.

Para modelar el descuento de viajero frecuente, donde ```la agencia A aplica un descuento del 35% cada vez que la agencia se utiliza por tercera vez consecutiva```, usamos una idea parecida. Incorporamos la variable:

$$Y_{e_i,e_{i+1},_{i+2}} = \begin{cases} 1 & \mbox{si } X_{A,e_i} = 1 \wedge X_{A,e_{i+1}} = 1 \wedge X_{A,e_{i+2}} = 1 \\ 0 & \mbox{caso contrario} \end{cases}$$

para cada $$i$$ tal que $$1\leq i \leq 41009$$.

La motivación es saber que hay 3 asignaciones consecutivas de la agencia A y que entonces vale aplicar el descuento. La sutileza en este caso está en que si se tiene por ejemplo, 8 veces consecutivas la agencia A, el descuento aplica solo en el costo de la tercera y sexta vez, como aclara el [FAQ](https://almundo.com.ar/eci/faq). Por esta razón, vamos a pedir lo siguiente:

$$Y_{e_i,e_{i+1},e_{i+2}} + Y_{e_{i+1},e_{i+2},e_{i+3}} + Y_{e_{i+2},e_{i+3},e_{i+4}} \leq 1$$

para cada $$i$$ tal que $$1\leq i \leq 41007$$.

Y también una versión adaptada de las restricciones anteriores:

$$Y_{e_i,e_{i+1},e_{i+2}} \leq X_{A,e_i}$$

$$Y_{e_i,e_{i+1},e_{i+2}} \leq X_{A,e_{i+1}}$$

$$Y_{e_i,e_{i+1},e_{i+2}} \leq X_{A,e_{i+2}}$$

$$Y_{e_{i-2},e_{i-1},e_i} + Y_{e_{i-1},e_i,e_{i+1}} + Y_{e_i,e_{i+1},e_{i+2}} + 2 \geq X_{A,e_i} + X_{A,e_{i+1}} + X_{A,e_{i+2}}$$

con el ajuste correspondiente en los primeros 2 tramos del recorrido, según si hace falta $$Y_{e_{i-2},e_{i-1},e_i}$$ y/o $$Y_{e_{i-1},e_i,e_{i+1}}$$.

Podemos entonces modelar el descuento como:

$$ \sum_{1\leq i \leq 41009} Y_{e_i,e_{i+1},e_{i+2}} * w_{e_{i+2}} * 0.35 * 0.01 $$

El último detalle es sobre el descuento por acumular kilómetros, donde ```la agencia D reintegra un monto fijo de $15 cada 10000km recorridos con esta agencia```. Definimos:

$$Y_D = \left\lfloor \frac{1}{10000} * \sum_{e \in T} X_{D,e} * w_e \right\rfloor$$

como la cantidad de veces que descontamos $15. Para linearizar $$Y_D$$ agregamos las siguientes restricciones:

$$\frac{1}{10000} * \sum_{e \in T} X_{D,e} * w_e \leq Y_D + (1-\epsilon)$$

$$Y_D \leq \frac{1}{10000} * \sum_{e \in T} X_{D,e} * w_e$$

y modelamos el descuento simplemente como:

$$Y_D * 15$$

### Cómputo

Para resolver el modelo existen muchas alternativas, cada una con diferentes ventajas/desventajas. Durante la competencia consideré:

  * [PuLP](https://pypi.python.org/pypi/PuLP), una librería de Python que permite escribir el modelo y llamar a solvers de propósito general.
  * [CPLEX](https://en.wikipedia.org/wiki/CPLEX), de lo mejor (si no es el mejor) de los solvers de propósito general.

Utilicé PuLP para programar rápido el modelo y CPLEX para resolverlo.

La asignación óptima sobre el recorrido mencionado anteriormente tiene un costo de **$63828.5** (es decir que mejoramos la solución en $2095.81).

# Algunas observaciones

La mejor solución final la conseguí incorporando un poco de búsqueda local sobre la solución anterior para tratar de refinarla. Logré bajar a **$63815.82** únicamente intercambiando el orden de ciudades consecutivas.

Una observación interesante es que es posible modelar el problema original (**completo**) utilizando programación lineal entera. Faltando unas horas para que termine la competencia, un amigo que estaba jugando (*fedepousa*) me pasó el modelo que había armado dado que él se estaba yendo de vacaciones y no podía dedicarle más tiempo. Su modelo es exacto para el problema completo, pero es impensable correrlo sobre toda la instancia. Tomé su modelo y lo incorporé como un operador de búsqueda local más, aplicándolo en *chunks* de mi mejor recorrido. Es un operador costoso en términos de tiempo, y aplicándolo en sub-recorridos de 20 ciudades, cada 20 ciudades, tardaba un poco menos de 2hs. Si bien se está resolviendo el **óptimo** en cada *chunk*, se corre el riesgo de empeorar la solución total dado que el operador no tiene visibilidad global del recorrido. Una combinación final de este operador, los intercambios, y nuevamente la asignación óptima que presenté antes, fue la que me dejó en el cuarto puesto con un recorrido de **$63801.03**. Gracias *fedepousa*!

Si bien el tiempo que pude dedicar durante los días de la competencia fue acotado, creo que sería valioso experimentar con más operadores de búsqueda local (y en particular con el modelo exacto para todo el problema).

# Bibliografía
Si te gustó la forma de pensar el problema, o tenés curiosidad sobre programación lineal entera o el problema del viajante de comercio, te recomiendo:

* [Discrete-optimization](https://www.coursera.org/learn/discrete-optimization), un gran curso sobre Optimización Discreta en general (y programación entera en particular).
* Cook, W. (2012). In pursuit of the traveling salesman: mathematics at the limits of computation. Princeton University Press.
* Applegate, D. L., Bixby, R. E., Chvatal, V., & Cook, W. J. (2011). The traveling salesman problem: a computational study. Princeton university press.
* Wolsey, L. A. (1998). Integer programming. Wiley.
* Williams, H. P. (2013). Model building in mathematical programming. John Wiley & Sons.
