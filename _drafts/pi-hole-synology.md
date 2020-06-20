---
layout: post
title: pihole en la NAS.
subtitle: internet sin anuncios.
cover-img: /assets/img/path.jpg
tags: [Synology]
---

¿Qué es pi-hole?, básicamente es un servicio de red para la resolución de nombres de Dominio (DNS), que bloquea los anuncios y publicidad; opcionalmente, también puede dar servicio de asignación de IPs (DHCP) a todos los dispositivos que están conectados a una red doméstica. 
Y ¿cómo lo hace?: funciona como un DNS local que intercepta las peticiones a DNS que hacen todos los dispositivos conectados a tu red, y comprueba si estás llamando a servidores incluídos en unas listas de sitios que se dedican a servir anuncios. 

Muchos utilizamos este tipo de bloqueadores desde hace tiempo en nuestros navegadores; aunque esos plug-ins emplean otras técnicas. Utilizando pi-hole, puedes conseguir el mismo efecto, pero en todos los dispositivos que hoy en día tenemos en casa conectados a internet: smartTVs, tablets, teléfonos móviles, el ordenador del niño y del abuelo...

La instalación de pi-hole como un docker sobre una NAS Synology tiene varios pasos, y los vamos a comentar en este post. Ya hay multitud de videos en youtube que cubren cómo configurar una pi-hole una vez instalada, así que nos vamos a centrar en las particularidades que tiene la configuración de una pi-hole para que arranque como un docker en una Synology. En gran medida, tenemos que lidiar con el hecho de que una Synology tiene muchos servicios arrancados "de serie", y por tanto la probabilidad de que haya un conflicto de puertos es alta; esto hace que parezca más dificil instalar una pi-hole en una Synology, que en una Raspberry. Pero si tomais las precauciones que os explico a continuación, vereis que no hay ningún problema.

El objetivo que queremos conseguir es hacer que la pi-hole corra en un docker con una IP propia, que sea accesible desde cualquier equipo pinchado a mi red (incluído el Host Docker, la propia Synology); es decir, que la pi-hole tiene que tener su propia IP dentro del rango de direcciones locales... Lo que vulgarmente conocemos como el rango: 192.168.1.XXX y además su dirección debe ser distinta a la de la Synology que hace de Host.

Para que no te pierdas durante el post: vamos a suponer que la IP de la Synology es la 192.168.1.14, y que hemos decidido que la pi-hole va a tener la 192.168.1.30 Esto, lógicamente, son dos ejemplos. Tú puedes decidir las IPs que en tu red. Sólo asegurate de que no estén ya asignadas, o que no están el el rango asignable por DHCP.

Lo primero que hacemos es iniciar sesión en nuestra NAS, y vamos a la carpeta donde se guardan los archivos persistentes de los dockers. Normalmente, en una Synology, la ruta es /volume1/docker/
Creamos en ella la carpeta persistente de la pi-hole: 
`mkdir pi-hole`
y dentro de ella dos carpetas más: 

`cd pi-hole` 

`mkdir etc-pihole` 

`mkdir etc-dnsmasq.d` 


A continuación, crearemos un controlador de red virtual (llamado macvlan en docker), gracias a este controlador podremos conseguir que la interfaz de red que usa el docker de pi-hole sea diferente de la que usa la Synology, pudiendo tener una dirección IP diferente. Si no hiciésemos esto, la forma estandard en la que los dockers se conectan con su Host, es a través del bridge que se crea por defecto. El efecto que esto produce es que, para acceder a un determinado docker desde mi LAN, voy a tener que usar la IP del Host. Si para acceder al docker pi-hole, uso la IP de la Synology, no podré llegar sin redirigir puertos... Como la Synology tiene muchos puertos ya ocupados, lo ideal es evitar la posibilidad de conflicto asignando su propia IP al docker pi-hole y olvidarnos del riesgo de problemas. Esto puede sonar a "matar moscas a cañonazos", pero no es así, el servicio DNS es crítico para la estabilidad de nuestra conexión.

Lo primero es averiguar cuál es el nombre de la interfaz de red de la synology, escribo:
`ip addr` 
obtendré una lista de interfaces, y anotaré el nombre de la que está asociada con la IP de mi Synology (en mi ejemplo: la IP es 192.168.1.30, y el nombre de la interfaz es: eth0). El nombre de la interfaz lo vamos a necesitar para crear la interfaz macvlan. 
Usamos ahora el siguiente comando: 
`sudo docker network create -d macvlan --subnet=192.168.1.0/24 --ip-range=192.168.1.30/32 --gateway=192.168.1.1 -o parent=eth0 pihole_network` 

Este comando docker nos permite:
- crear una red docker con network create.
- usando un driver del tipo macvlan.
- la asignamos al rango 192.168.1.X, con máscara 255.255.255.0, 
- y con el rango fijado de forma que asignamos una única dirección: 192.168.1.30 a esta interfaz. 
- definimos qué gateway debe utilizar, que es la dirección de router que estás usando para tu salida a internet, el estandard suele ser: 192.168.1.1
- Lo siguiente que tenemos que definir es el adaptador de red principal, el que hemos obtenido al principio: eth0. Esto es, el adaptador que está usando nuestro host Doker -la NAS Synology-. Haciendolo de esta manera, conseguiremos tener conectividad por LAN entre la Synology y la pi-hole.
- por último, le damos un nombre a esta red: pihole_network.

Si ahora accedemos a la aplicación de gestión de Dockers de Synology, veremos que la red se ha creado, pero no tiene ningún docker conectado a ella.

En el siguiente paso, vamos a crear un bridge al que conectaremos la pi-hole, empleando la interfaz de red que acabamos de definir. Usamos este comando:

`sudo docker network create -d bridge --subnet=192.168.1.0/24 --ip-range=192.168.1.30/32 --gateway=192.168.2.1 pihole_bridge` 

Este comando docker nos permite:
- crear una red docker con network create.
- usando un driver del tipo bridge.
- le asignamos al rango 192.168.2.X, con máscara: 255.255.255.0, 
- y con el rango fijado, de forma que asignamos una única dirección: 192.168.2.2 a esta interfaz. 
- definimos qué gateway debe utilizar, que en este caso es: 192.168.2.1
- por último, le damos un nombre a esta red: pihole_network.


Una vez hemos creado estas dos redes, el procedimiento a seguir para instalar el docker de pi-hole en Synology es: 
- bajamos la imagen de pi-hole del registro.
- lanzamos el docker, y marcamos que se ejecute con privilegios si pensamos usarlo como DHCP (además de como DNS).
- en la pestaña Volume, asociamos los dos puntos de montaje que pide el docker con las dos carpetas que hemos creado al inicio.
- en la pestana Networks, conectamos el docker a las dos redes que hemos creado (bridge y macvlan). No olvidar desconectar el docker a la red bridge a la que se conectan por defecto todos los nuevos dockers instanciados.
- la pestaña port settings la podemos dejar como viene por defecto.


El resto de la configuración serán las variables de entorno del Docker de pi-hole. Todos estos son detalles no específicos, y puedes encontrar las variable de entorno necesarias, y sus valores recomendados más actualizados en el docker hub para [pi-hole](https://hub.docker.com/r/pihole/pihole/)

Para finalizar, ejecutaremos el docker con pi-hole y haremos las siguientes verificaciones:
- Desde una terminal del docker pi-hole, haremos ping a la IP de la NAS. En nuestro ejemplo: 192.168.14 Si todo ha salido bien, debemos NO de tener respuesta. Si hacemos ping a 192.168.2.1 si deberíamos tenerla.
- Desde una terminal de nuestra Synology, haremos ping a la IP de la pi-hole. En nuestro ejemplo: 192.168.1.30 Este ping debería NO funcionar. A continuación probaremos hacer ping a la dirección IP que me da acceso desde Synology a pi-hole (a través de la red pihole_bridge que hemos creado) En nuesto ejemplo: 192.168.2.2 y si deberíamos tener respuesta.
- Si tenemos más dockers corriendo en el Host Synology, podremos conseguir este mismo efecto.

Por último, si estas pruebas se han superado, ya podemos completar la configuración. El último paso consiste en ajustar cuál es el DNS primario de la Synology, dentro de los parámetros de red indicaremos que su DNS primario es: 192.168.2.2
De este modo, cuando la NAS tenga que resolver un nombre empleará como DNS el docker pi-hole que reside en ella.

Sólo un comentario para finalizar este post. Tambien es posible lograr que pi-hole funcione símplemente redirigiendo puertos, como hacemos con muchos otros dockers. Sin embargo, en mi experiencia, esta configuración es más estable por evitar el riesgo de conflictos originados al activar un servicio Synology que entre en conflicto con las comunicaciones de pi-hole durante su normal operación.
