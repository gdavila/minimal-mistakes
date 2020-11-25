---
title: Wall clock usando ffmpeg
excerpt: Wall clock usando ffmpeg
---


Dentro del mundo de Video Broadcast, se conoce como *Wall Clock* a un stream de video que tiene impreso un cronómetro. Típicamente se usa para medir la latencia entre dos puntos de la red de video, por ejemplo, tomando una captura del *time stamp* del cronomertro en los dos puntos de red al mismo tiempo, se puede calcular la diferencia entre ambos valores y el resultado es la latencia del punto A al punto B.

En este breve tutorial, se muestra como construir un video con *Wall Clock* usando la herramienta open source [ffmeg](https://ffmpeg.org/). 

La mayor parte de este contenido fue originalmente expuesto por [Gyan](https://stackoverflow.com/users/5726027/gyan) en [stackoverflow](https://stackoverflow.com/questions/47543426/ffmpeg-embed-current-time-in-milliseconds-into-video)



# Definiciones

* El siguiente ejemplo de ffmpeg genera un *Wall Clock* con pasos en escala de milisegundos y precisión de $\mu$s.  
* Bitrate video= 5 Mbps
* Bitrate mux= 8 Mbps @ CBR
* Codec = H.264/AVC
* Frame rate = 30 fps (ntsc)
* Resolución = 640x480
* Output = multicast udp
* GOP length = 6



# Pasos

1\. Instalar [ffmeg](https://ffmpeg.org/)

2\.  Ejecutar el siguiente codigo en el host que hará el stream: 


```
ffmpeg -re -f lavfi -i color=black:640x480:rate=ntsc,format=yuv420p -g 6 -r ntsc \
-vf "settb=AVTB,setpts='trunc(PTS/1K)*1K+st(1,trunc(RTCTIME/1K))-1K*trunc(ld(1)/1K)'\
,drawbox=y=ih/PHI:color=black@0.4:width=iw:height=48:t=fill,\
drawtext=text='%{localtime}.%{eif\:1M*t-1K*trunc(t*1K)\:d}:fontfile='/usr/share/fonts/truetype/ \
dejavu/DejaVuSans.ttf':fontsize=40:fontcolor=white:x=(w-tw)/2:y=240'" \
-b:v 5M -minrate:v 5M -maxrate:v 5M -bufsize:v 1M -preset ultrafast \
-vcodec libx264  -r ntsc -tune zerolatency  -muxrate 8M  \
-pcr_period 40 -f mpegts udp://@239.123.123.1:50000?pkt_size=1316 
```

	
- La primera linea de código llama a `ffmpeg` tomando como formato un video sintético usando la librería lavfi (`-f lavfi`) y manteniendo el frame rate del input (`re`). Este video usa como background un lienzo negro de 640x480 a 30 fps en formato de color YUV 4:2;0 (`color=black:640x480:rate=ntsc,format=yuv420p`). La salida del video codificado tendrá un GOP de 6 y un frame rate de 30 fps (`-g 6 -r ntsc ` ).
	
- Las lineas 2-5, imprimen el wallclock en el video. El reloj saldrá en el formato  `yyyy-MM-dd h:m:s.ms`. Desafortunadamente las librerías para impresion de fecha de ffmpeg no incluyen de forma nativa la información en milisegundos. Para añadir esta información, primero se modifican los 3 dígitos menos significativos del PTS para sincronizarlos con el momento exacto donde se añade el timestamp del wallclock . Esto no impacta a la velocidad de reproducción, ya que los trés digitos menos significativos del PTS corresponden a los microsegundos y el frame rate de un video está tipicamente en el orden de los milisegundos.
		
- En la segunda linea se usa el filtro `-vf` que añade el wallclock como un texto sobre el stream de video. Para esto se elige el timebase por defecto `settb=AVTB`, que significa una precisión de `10e-6` ($\mu$s). Adicionalmente  se añade los milisegundos del tiempo RTC en el PTS. Para esto se modifica la variable `setpts`. Primero se redondea el PTS para eliminar la información de los tres digitos menos significativos y se lo vuelve a las unidades originales ($\mu$) `trunc(PTS/1K)*1K`. En segundo lugar, se extrae los milisegundos del timer RTC `st(1,trunc(RTCTIME/1K))-1K*trunc(ld(1)/1K)` y se los suma a los dígitos menos significativos del PTS. `st()` y `lt()` sirven como métodos para almacenar y leer variables (1-10). En el código, primero se almacena la variable 	`1` y después se lee la variable `1`.
	
- En la tercera linea, se añade un cuadro de texto a la  altura `ih/PHI` o 640/*Golden Ratio*, con relleno `t=fill` negro a una transparencia del 40% `color=black@0.4`, ancho igual al del video `iw`, altura de 48 px. En la cuarta linea se añade el texto final,  que consiste en la variable `{localtime}` seguido de un punto y los milisegundos que previamente se añadieron en el PTS. Para esto se accede a la variable [`t`](https://ffmpeg.org/doxygen/2.4/aeval_8c_source.html), que representa el PTS en segundos y se extrae los milisegundos (digitos menos significativos) previamente agregados `\:1M*t-1K*trunc(t*1K)\`. El resto obedece a parametros para la selección de font y tamaño de letra.
	
- Las lineas 6-8, setean los parametros de salida del stream, como la codificación (*H264*), el bitrate del video (*5 Mbps*), el bitrate del mux (*8 Mbps*) y la dirección multicast del stream. 
	
3\. Reproducir el sream. Para hacer una prueba del stream previamente definido, se puede reproducir el video en el mismo host que hace el stream usando fflpay

```
ffplay -i  udp://239.123.123.1:50000 -fflags nobuffer
```
# Resultados
- El resultado del video con el wallclock debería ser:

![ejemplo ffplay](/video/ffmpegClock/ffplay1.png)
	

- Se puede modificar el código anterior para añadir video dinámico en lugar de un fondo negro. Por ejemplo, usando los *sources test* incluidos en ffmpeg `-i testsrc2=640x480:rate=ntsc,format=yuv420p`, el stream luciría así:
	
![ejemplo ffplay](/video/ffmpegClock/ffplay2.png)
