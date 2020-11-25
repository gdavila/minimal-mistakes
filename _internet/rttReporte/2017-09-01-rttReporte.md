---
title: Sobre el Comportamiento del RTT
excerpt: Reporte técnico de algunas experiencias monitoreando el RTT sobre distintos escenarios de Internet
---


# 1. Resumen

El presente documento es un breve reporte de las **investigaciones preliminares** realizadas para entender el comportamiento del *round-trip-time (rtt)* entre origen y destino, cuando la ruta entre estos dos puntos atraviesa: Túneles MPLS, Middleboxes, y tecnologías de acceso distintas desde los puntos de medición. A su vez, se pone a prueba una metodología de medición basada en *scamper* y *tracebox* que se implementa en la herramienta *rttExplorer*.

Lastimosamente esta investigación preliminar tiene varias limitaciones, por ejemplo, pocos puntos de medición y complejidades  para interpretar los resultados, ya que cada resultado requiere un analisis puntual y exaustivo dificil de automatizar.

Estos resultados parciales y tempranos, indican que no existe relación entre la presencia de tecnologías como MPLS o middleboxes con la distribución que muestra el *rtt*. Por otro lado, se encuentra  que la distribución de *rtt* varía entre un ISP y otro. No queda claro si esta diferencia se debe a la tecnología de acceso de cada uno o por otros factores. Adicionalmente se confirma trabajos preliminares que encuentran que la distribución de *rtt* parece seguir  una distribución *estable* de una sola componente, y el hecho de que eventualmente aparezca una segunda componente (distribución bimodal) se relaciona con cambios o problemas de red en la ruta entre el punto de medición y el hop analizado.


# 2. Introducción

La idea inicial de este trabajo surge al intentar analizar la distribución del *rtt* sobre distintos hops de internet. Dado que aparentemente el *rtt* tiene una distribución del tipo *estable*, se intentó encontrar porqué en algunas ocaciones esta distribución posee más de una componente modal. Basandose en trabajos anteriores, la sospecha  *a priori* fue que esta segunda componente modal podría originarse en la existencia de alguna tecnología invisible para la topología IP, como MPLS o middleboxes. Lastimosamente no se encontró evidencia suficiente para sustentar esta hipotesis.

En este sentido, en el resto de este documento se describen algunas observaciones y resultados producto de esta investigación preliminar. Inicialmente se describe brevemente la herramienta y la metodología usada para medir el *rtt* de un hop determinado. Se describe adicionalmente las características del experimento y finalmente se muestran los ejemplos que se consideran más relevantes de los resultados observados.

# 3. Dataset

La metodología para obtener el dataset consiste en realizar mediciones del *rtt* a cada uno de los hops descubiertos mediante una sonda basada en *paris-traceroute*. En la siguientes secciones se detallan las características de estas mediciones.

## 3.1. Herramientas utilizadas

Los datos expuestos en el presente reporte técnico fueron obtenidos mediante [*rttExplorer*](https://github.com/gdavila/rttExplorer), que no es más que una herramienta basada en Python para realizar mediciones del *rtt (round-trip-time)*. Estas mediciones se realizan sobre los distintos saltos que atraviesa una sonda tipo *traceroute* entre origen y destino al viajar sobre el Internet. Especificamente *rttExplorer* usa [*scamper*](https://www.caida.org/tools/measurement/scamper/) y [*tracebox*](http://www.tracebox.org/) para garantizar que los paquetes enviados en cada sonda eviten en lo posible el balanceo de carga de los enrutadores del Internet.

La metodología que implementa *rttExplorer* es simple:

* Inicialmente se envía una sonda de *descubrimiento* hacia el destino deseado. Como resultado esta sonda revela todos los hops encontrados. Cada cierto intervalo de tiempo (típicamente en el orden de decenas de minutos), una sonda de similares características vuelve a enviarse para confirmar que la ruta previamente desubierta  se mantiene estable o revelar un nuevo camino. Este proceso se repite periodicamente mientras dura la exploración.

* El siguiente paso, una vez que se descubren los hops entre origen y destino, es enviar una sonda de *mediciones*. Esta segunda sonda tiene por objeto medir el *rtt* a cada hop previamente descubierto por la última sonda de *descubrimiento*. La herramienta *rttExplorer* trata en lo posible de medir el *rtt* para todos los hops de forma casi simultanea. Estas mediciones se repiten para todos los hops de cada ruta de forma periodica cada cierto intervalo de tiempo (típicamente en el orden de segundos), mientras dura la exploración.

* Finalmente los resultados son almacenados localmente en formato json y son expuestos en una mongoDB en donde se almacena de forma definitiva los resultados de las mediciones.


## 3.2. Selección de puntos de prueba y destinos

Debido a limitaciones para ubicar puntos de prueba, el presente dataset solamente utiliza  ubicaciones que se encuentran en puntos domiciliarios de internet pertenecientes a los siguientes operadores:

* Telecom: Servicio sobre tecnología Docsis (Fibertel)
* Telefónica: Servicio sobre tecnología DSL (Speedy)
* Personal: Servicio sobre tecnología LTE

Los destinos se eligen al azar, sin ningún criterio específico e intentando que se ubiquen en distintas zonas geográficas.

## 3.3. Detalles de la sonda

Se realizan exploraciones y mediciones usando indistintamente sondas TCP o UDP. En cada uno de los resultados mostrados en las siguientes secciones se describirá el protocolo utizado.


<a id="table1"></a>

| Protocolo                   	| tcp/udp              	|
|-----------------------------	|----------------------	|
| Puerto origen               	| random               	|
| Puerto destino              	| 443(tcp)/4444(udp)   	|
| Método                      	| tcp-paris, udp-paris 	|
| Intervalo de Descubrimiento 	| 10 min               	|
| Intervalo de Mediciones     	| 1 sec                	|


## 3.4. Limitaciones del experimento

* Limitados puntos de medición disponibles.
* Resultados basados en pocos experimentos (Tres origenes a aproximadamente una decena de destinos).


# 4. Resultados

La idea inicial del experimento fue entender con el mayor detalle posible el comportamiento del *rtt* sobre *links* de internet que enfrentan distintos escenarios, por ejemplo, que atraviesan *middle boxes*, túneles MPLS, o tecnologías subyacentes distintas (como distintas redes de acceso o redes de transporte). Desafortunadamente, dadas las limitaciones en esta fase para tener más puntos de medición, no todos los escenarios pudieron ser evaluados ni se hizo suficientes pruebas para obtener resultados concluyentes.

Sin embargo, preliminarmete se encuentró que el comportamiento del *rtt* no se ve afectado por la presencia de *middle boxes (MB)* o túneles MPLS. Asimismo se confirma la distribución del tipo *estable* del *rtt* como ya se había planteado en discusiones [anteriores](). Esta distribución típicamente presenta una sola componente modal. Aunque eventualmente pueden aparecer distribuciones *estables* bimodales.

La aparición de componentes modales adicionales parece no tener relación alguna con la presencia de MPLS o MB, sino más bien se relaciona con:

* Cambios en el *path* que resultan imperceptibles en terminos de *hops*: Es decir, se observa que el *rtt* cambia aún cuando se mantienen tanto los *hops* hasta el destino (misma ruta) y la cantidad de *hops* desde el destino al punto de medición (```reply_ttl```).

* Una segunda componente modal en la distribución parece estar relacionada a la carga de la red. Esto es principalmente notorio cuando se hace analiza mediciones que tienen una duración prolongada (en el orden de varias horas).

En las siguientes secciones se muestran los resultados mas representativos de los distintos casos analizados.

## 4.1. Distribución *estable* con componente modal única.

### 4.1.1. Variación de *rtt* en función de horas pico de consumo del ISP.
<a id="table1"></a>

| parametro                   	| valor              	|
|-----------------------------	|----------------------	|
| ISP punto de medicion    		| Telecom		      	|
| Tecn. Acceso punto de medición| Docsis		      	|
| IP origen               		| 192.168.0.126       	|
| IP destino              		| 187.102.77.237	   	|
| hop IP                     	| 200.89.165.222	 	|
| ttl inicial			 		| 5		              	|
| hop AS			     		| AS10318 (Telecom)    	|

![](/internet/rttReporte/unnamed-chunk-1-1.png)

La distribución de *rtt*  presenta una sola componente modal.
**No se observan túneles MPLS** en el path **ni middleboxes**.

Adicionalmente se puede observar que el *rtt* varía en el tiempo en función de las horas de mayor consumo de la red de acceso.


### 4.1.2. Variación de *rtt* por *outages* en la red.

<a id="table2"></a>

| parametro                   	| valor              	|
|-----------------------------	|----------------------	|
| ISP punto de medicion    		| Telefonica	      	|
| Tecn. Acceso punto de medición| DSL		      		|
| IP origen               		| 192.168.1.35       	|
| IP destino              		| 185.45.165.14		   	|
| hop IP                     	| 200.51.208.166	 	|
| ttl inicial			 		| 4		              	|
| hop AS			     		| AS22927 (Telefonica) 	|


![](/internet/rttReporte/unnamed-chunk-2-1.png)

La distribución del *rtt*  presenta una sola componente modal.
**El *hop* analizado es el *ingress* LSR de un túnel MPLS**. Finalmente, en el path **No se observan middleboxes**.

Adicionalmente se puede observar que la componente modal del *rtt* no cambia  con el tiempo a pesar de que se detectan cortos intervalos de *outage* durante el monitoreo.


## 4.2. Distribución *estable* con componente bimodal.

### 4.2.1. Variación de *rtt* por cambios leves y de larga duración.

<a id="table3"></a>

| parametro                   	| valor              	|
|-----------------------------	|----------------------	|
| ISP punto de medicion    		| Telefonica	      	|
| Tecn. Acceso punto de medición| DSL		      		|
| IP origen               		| 192.168.1.35       	|
| IP destino              		| 185.45.165.14		   	|
| hop IP                     	| 201.179.128.1		 	|
| ttl inicial			 		| 2		              	|
| hop AS			     		| AS22927 (Telefonica) 	|

![](/internet/rttReporte/unnamed-chunk-3-1.png)
La distribución del *rtt* presenta dos componentes modales que ocurren por cambios leves en el *rtt* durante intervalos prolongados. Este cambio puede notarse principalmente las *12:15* y *15:00*. Sin embargo, si se grafica la distribución resultante en intervalos más cortos de tiempo, solo se observaría una componente estable.

Durante la ruta hasta el *hop* analizado, **no** se encuentran ni **túneles mpls** ni **middleboxes**.




### 4.2.2. Variación de *rtt* por cambios bruscos y de larga duración.



<a id="table3"></a>

| parametro                   	| valor              	|
|-----------------------------	|----------------------	|
| ISP punto de medicion    		| Telefonica	      	|
| Tecn. Acceso punto de medición| DSL		      		|
| IP origen               		| 192.168.1.35       	|
| IP destino              		| 187.49.218.114	   	|
| hop IP                     	| 187.49.218.114	 	|
| ttl inicial			 		| 19		           	|
| hop AS			     		| AS28154 (Telecom)		|

![](/internet/rttReporte/unnamed-chunk-4-1.png)

La distribución del *rtt* presenta dos componentes modales que ocurren por cambios bruscos en el *rtt* durante intervalos prolongados. Este cambio puede notarse principalmente las *04:30*. Sin embargo, si se grafica la distribución resultante haciendo un corte a las 04:30, solo se observaría una componente estable en la distribución.

Durante la ruta hasta el *hop*, se descubren **túneles mpls** antes de alcanzar el hop analizado y **no** se registran **middleboxes**. Sin embargo, los LSR (routers MPLS) en hops previos, no influyen en el cambio de comportameinto del *rtt*.

El cambio brusco del *rtt* en el dominio temporal podría significar que las sondas cambiaron de ruta, sin embargo no se detecta ningun indicio de esto analizando la ruta del traceroute (hops, ```probe_ttl``` y  ```reply_ttl```). Este comportamiento también se observa en el ejemplo siguiente (hop 190.216.88.34).


<a id="table3"></a>

| parametro                   	| valor              	|
|-----------------------------	|----------------------	|
| ISP punto de medicion    		| Telefonica	      	|
| Tecn. Acceso punto de medición| DSL		      		|
| IP origen               		| 192.168.1.35       	|
| IP destino              		| 181.30.134.68		   	|
| hop IP                     	| 190.216.88.34		 	|
| ttl inicial			 		| 11		           	|
| hop AS			     		| AS4323 (Level 3 AR) 	|

![](/internet/rttReporte/unnamed-chunk-5-1.png)

### 4.2.3. Variación de *rtt* por cambios bruscos y cortos.

<a id="table3"></a>

| parametro                   	| valor              	|
|-----------------------------	|----------------------	|
| ISP punto de medicion    		| Telecom	    	  	|
| Tecn. Acceso punto de medición| Docsis	      		|
| IP origen               		| 192.168.0.126       	|
| IP destino              		| 198.45.49.161		   	|
| hop IP                     	| 200.89.165.222	 	|
| ttl inicial			 		| 6		           		|
| hop AS			     		| AS10318 (Telecom) 	|


![](/internet/rttReporte/unnamed-chunk-6-1.png)
La distribución del *rtt* presenta dos componentes modales que ocurren por cambios eleves en el *rtt* durante un intervalo corto de tiempo. Este cambio puede notarse principalmente antes de las *22:30*.

**El *hop* analizado es el *ingress* LSR de un túnel MPLS**. Finalmente, en el path **No se observan middleboxes**.

En este caso, el cambio leve del *rtt* en el dominio temporal podría coincidir con saturación de la red del ISP.

### 4.3. variación de *rtt* por ISP

Lastimosamente no se tienen suficientes puntos de medición para entender porqué el comportamiento del *rtt* varía de ISP en ISP. Pero resulta importante remarcar la diferencia en las características de la distribición estable que surge al analizar los resultados desde diferentes ISPs. A continuación se muestran tres gráficas representativas de cada ISP.

Es caracerístico que las mediciones desde *Telecom* presentan mayores variaciones en el tiempo, mientras que las mediciones desde *Telefónica* parecen ser más planas. Esto se observa en las siguientes figuras.

<p style="text-align: center;"> Personal (LTE) </p>
![](/internet/rttReporte/unnamed-chunk-7-1.png)
<p style="text-align: center;"> Telecom (Docsis) </p>
![](/internet/rttReporte/unnamed-chunk-8-1.png)
<p style="text-align: center;"> Telefonica (DLS) </p>
![](/internet/rttReporte/unnamed-chunk-9-1.png)

# 5. Conclusiones

* De forma preliminar, la presencia de túneles MPLS o Middleboxes no caracterizan ningún comportamiento especial en la distribución del *rtt*.
* La doble componente modal de la distribución estable parece estar más relacionada con cambios en la red, que con una característica propia de la distribución del *rtt*. Esto es, la distribución bimodal solamente aparecería si se mide el *rtt* durante un tiempo suficientemente grande (orde de varias horas) como para aumentar la probabilidad de que haya algún cambio en el comportamiento de la red.

# 6. Posibles investigaciones futuras

* Replicar las experiencias presentadas en este reporte preliminar con más puntos de medición.
* Resultaría interesante si las características de la distribución *estable* permiten conocer la tecnología de subyacente de la red, por ejemplo, qué tipo de redes de acceso se usan (LTE, Docsis, DSL, etc).
* No queda claro cuales son los fenómenos que causan que el *rtt* cambie bruscamente aún cuando no hay otros indicios de cambio de path. Probablemente esto se deba que hay tecnologías invisibles para la topología IP (MPLS, Ethernet, Tecnologías de Transporte, etc). Intentar explicar a qué se deben estas variaciones de *rtt* resultaría interesante, así mismo se podría analizar si los cambios bruscos de *rtt* podrían permitir inferir con precisión los cambios de ruta en un path.
