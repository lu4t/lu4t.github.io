---
layout: post
title: La autenticación de microservicios.
subtitle: Seguridad en un service mesh.
gh-repo: lu4t/lu4t.github.io
gh-badge: [star, follow]
tags: [Cloud,Security]
comments: true
---

La Arquitectura de Microservicios es la Arquitectura de Software más famosa y utilizada hoy en día, en este post hablamos sobre algunos de los beneficios e inconvenientes que tiene este tipo de arquitectura. Y es que uno de los aspectos más problemáticos, es el relacionado con aplicar los principios de seguridad de la información en estos escenarios.

El principal problema de aplicar los principios y políticas de seguridad a la capa de software, cualesquiera que éstas sean, es que puede derivar en problemas que no aparecían cuando las políticas se diseñaron para ser aplicadas en la infraestructura. Algunos de estos problemas podrían ser de este estilo:

* Los Desarrolladores puede que no sean capaces de implementar las políticas que definamos correctamente.
* Los Desarrolladores de microservicios no son por lo general expertos en estándares de seguridad, y pueden obviar ciertos aspectos considerados básicos para el que define la política.
* Por lo general, los lenguajes y stacks de desarrollo incorporan ciertos criterios y buenas prácticas de seguridad; que no tienen porqué ser perfectamente compatibles con las nuestras.
* El uso de certificados en la capa de aplicación incrementa la complejidad en la administración de la aplicación; y el origen de esta complejidad está precisamente en la gestión de los certificados que emplean las aplicaciones.

La seguridad es un requisito no funcional de todas las aplicaciones; pero sucede que en la mayoría de las empresas, el área que gestiona la seguridad y define los estándares a seguir no tiene un vínculo natural con los equipos de desarrollo. Tradicionalmente su función se concentra en la infraestructura, lo que se traducía en definir reglas en firewall, establecer los controles y gestionar los privilegios de acceso a los servicios de red, etc.

Consecuentemente, la industria se encuentra actualmente en una búsqueda desesperada por encontrar cuál es el estándar de seguridad más efectivo para aplicar a los microservicios. Y cada vez es más frecuente encontrarnos con que los profesionales de la seguridad llegan a la conclusión de que implementar un Service Mesh es la solución a muchos de los problemas que hemos adelantado como asociados a los microservicios. Los service meshes proporcionan una serie de funcionalidades y capacidades básicas, que los hacen especialmente atractivos: Descubrimiento, Balanceo de Carga y Autenticación de microservicios.

De la misma manera que Kubernetes se ha convertido en los últimos años en el estándar “de facto” para la orquestación de carga desplegada en nubes híbridas, Istio está empezando a ser reconocido por la industria como el service mesh estándar “de facto”. De todos modos, no conviene olvidar que igual que Kubernetes no es la única herramienta para desplegar microservicios, Istio tampoco es el único service mesh disponible.

De una manera resumida: ¿qué es Istio?; es un service mesh desarrollado como una colaboración entre Google, IBM y Lyft, de código abierto (open-source), que permite conectar, monitorizar y securizar microservicios. Como cualquier otro service mesh, Istio incorpora las funcionalidades de descubrimiento, balanceo y autenticación, para microservicios que hayan sido desplegados on-prem, en la nube, o en una plataforma de orquestación (como Kubernetes, por ejemplo).

No es objeto de este post hablar de Istio en detalle, ni de todas las funcionalidades que ofrece que no están directamente relacionadas con la mencionada en el título de este post: La Autenticación de los Microservicios gestionada por el Service Mesh.

# ¿Qué tipos de autenticación hay en un Service Mesh?

Básicamente hay dos tipos de autenticación. El primero involucra a los usuarios finales del microservicio, y el segundo tiene que ver con la autenticación entre servicios -de servicio a servicio- en la que típicamente se emplearán certificados.

Vamos a presentar con un poco más de detalle las diferencias entre ambos modelos.

## Autenticación del usuario final.

Esta función tiene como objetivo autenticar al usuario final de un microservicio; aquí, cuando decimos usuario final, en realidad hablamos de una entidad. Ya que podemos referirnos a una persona que está intentando acceder a nuestro sistema, pero también a un dispositivo (IoT), o a una aplicación que está intentando acceder a nuestra solución.

El service mesh permitirá una integración sencilla con algún proveedor de identidad, mediante la especificación OIDC (OpenID-Connect) o un estándar equivalente, y la autenticación de cada solicitud será manejada con el intercambio de JWT (Java Web Tokens), que es la especificación de seguridad más utilizada para aplicaciones Cloud nativas.

Una vez autenticada la entidad que pretende hacer uso del microservicio, la autorización de cada llamada puede gestionarse de manera simple con alguno de los flujos de autorización sobre el protocolo OAuth 2.0.

A esta categoría de autenticación/autorización en ocasiones se la denomina “Norte-a-Sur”.

## Autenticación entre servicios

A este tipo de autenticación a veces se le denomina “autenticación de transporte”, o “autenticación Este-Oeste”; tiene por objetivo verificar la conexión entre clientes, y así permitir la comunicación entre ellos.

Los Service Meshes habitualmente emplean MTLS (Mutual TLS) como mecanismo para realizar la autenticación de transporte. El uso de MTLS implica la descarga y uso de unos certificados digitales, gestionados por el Service Mesh, y que son cargados en los proxies sidecar (habitualmente proxies envoy) que están asociados a cada microservicio; estos proxies se encargan de establecer túneles seguros de comunicación entre microservicios.

Este mecanismo, que suena como muy enrevesado, es en realidad de uso muy habitual en entornos con transporte https; con él, cada microservicio hablará con un microservicio vecino a través de un proxy empleando para ello un túnel de comunicación, encriptado por TLS.

En este tipo de autenticación, hay dos flujos muy relevantes de los que se ocupará el service mesh en tiempo de ejecución: el flujo de configuración de los proxies, y el flujo de autenticación entre proxies.

### FLUJO DE CONFIGURACIÓN

Este flujo, gobernado por completo por el Service Mesh, se encargará de la distribución de los certificados digitales en cada pod (en realidad, en cada proxy de cada pod); al conjunto de proxies de todos los pods de un mismo microservicio, le llamamos plano de datos (Data Plane).

Estos son los pasos de alto nivel que se implementan en el flujo de configuración:

* El Service Mesh (SM) escucha el API del servidor de Kubernetes, y crea un certificado y un par de claves para cada cuenta de servicio nueva o existente. El SM almacena el certificado y los pares de claves dentro del vault que tenga definido.
* Cuando se crea un nuevo pod, el SM proporciona al gestor de pods (por ejemplo Kubernetes) el certificado y el par de claves necesarias para que arranque el pod, empleando para acceder al vault la cuenta de servicio asociada al pod.
* El SM vigila también la vida útil de cada certificado que ha proporcionado al orquestador, y automáticamente rota los certificados a punto de caducar, para que sea posible reescribir los nuevos secretos en cada pod que los necesite en el momento adecuado.
* El SM también genera y proporciona información de nombres, estableciendo la relación de qué cuenta o cuentas de servicio pueden ejecutar un determinado microservicio. Esta información es además compartida con el proxy Envoy.

Todos estos pasos son llevados a cabo por el Service Mesh de forma automática. Algo que es muy importante, ya que hace que los certificados digitales sean fáciles de administrar, además de convertirse en el punto centralizado de actuación en el caso de que suceda algo incorrecto con los certificados.

### FLUJO DE AUTENTICACIÓN ENTRE PROXIES

Una de las funciones indispensables de un Service Mesh (SM), es ser capaz de enrutar tráfico entre microservicios. El tráfico permitido, o no permitido, se define en el SM a nivel de servicio. A diferencia de una arquitectura tradicional, el microservicio A no tiene porqué conocer el rango de direcciones IP en el que se encuentra el servicio B con el que se quiere comunicar, el SM se encarga de informar al sidecar proxy anexo a cada microservicio cómo y dónde debe enrutar el tráfico permitido. Para ello, se emplea el flujo de autorización entre servicios, sigue estos pasos descritos a continuación:

* Si el Servicio A quiere hablar con el B, el SM redirige el tráfico saliente desde el proxy del cliente origen, al proxy del cliente destino.
* El proxy Envoy del cliente origen inicia un handshake sobre MTLS con el proxy Envoy del servicio destino. Durante el handshake, el proxy del lado del cliente también verificará que la cuenta de servicio asociada al certificado utilizado, tiene autorización para ejecutar el servicio en destino.
* Cuando se completa el handshake entre proxies, se establece una conexión MTLS, y el SM reenvía el tráfico desde el proxy origen al destino.
* Una vez completada la autorización, el proxy origen reenvía el tráfico al destino empleando conexiones TCP locales.

Como se ver, este flujo está íntimamente ligado en las características nativas de los proxies Envoy, de ahí que sean los proxys más frecuentemente utilizados como sidecars.

# Conclusión.

Como se puede ver, el Service Mesh es una pieza clave en la configuración de la seguridad de una arquitectura de microservicios. De hecho, es el punto centralizado de gestión y gobierno de los controles de seguridad de este entorno. Pese a ello, la autenticación sucede de manera distribuida en el plano de datos.
