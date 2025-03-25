# Contenedores

La contenedorización es una alternativa para el despliegue / instalación de software.
De cara a instalar software tenemos varias opciones:

## Forma más tradicionales

       App1  +  App 2  +  App 3         PROBLEMAS GRAVES: 
    ------------------------------          - Incompatibilidad con las dependencias
           Sistema Operativo                - Qué ocurre si App1 tiene un bug (CPU: 100%) --> Offline
    ------------------------------                                            App2 y App3 --> Offline
                HIERRO                      - Seguridad

## Máquinas virtuales

       App1   |  App 2  +  App 3        PROBLEMAS:
    ------------------------------          - Configuración/Instalación es mucho más compleja
       SO1    |       SO2                   - Mantenimiento se complica
    ------------------------------          - Merma de recursos
       MV1    |       MV2                   - Menor rendimiento en las aplicaciones
    ------------------------------
              Hypervisor        
     VMWare, Citrix, VBox, Hyper-V
    ------------------------------
           Sistema Operativo
    ------------------------------
                HIERRO

## Contenedores

        App1  |   App2 + App3
    ------------------------------
        C1    |        C2
    ------------------------------
       Gestor de contenedores:
      Docker, Containerd, CRIO, 
      Podman
    ------------------------------
       Sistema Operativo "Linux"
    ------------------------------
                HIERRO

Cuando trabajamos con contenedores, no instalamos software, desplegamos software.

# Concepto de contenedor

Es un entorno aislado dentro de un sistema operativo (Linux) donde puedo ejecutar procesoS.
AISLADO? En cuánto a qué?
- Tiene su propia configuración de red -> Su propia IP
- Tiene su propio sistema de archivos (HDD)
- Tiene sus propias variables de entorno
- Puede tener limitaciones de acceso al hardware

Creamos contenedores desde IMÁGENES DE CONTENEDOR.

## Imágenes de contenedor

Es un triste archivo comprimido (tar) que contiene:
- Una estructura de carpetas compatible con posix (la realidad es que no es necesario... aunque es lo habitual)
    bin/
    etc/
    opt/
    var/
    ...
- Programas ya instalados de antemano en esas carpetas
- Configuraciones de esos programas
- Paquetes (Dependencias) necesarios para que esos programas funcionen
- Metadatos... (como los comandos que se deben ejecutar para arrancar el programa)

Esas imágenes de contenedor las encontramos en REGISTROS DE REPOSITORIOS DE IMAGENES DE CONTENEDOR.
El más famoso: 
- Docker Hub
- Quay.io          <<< Es el registro de Red Hat
- Oracle container registry
- Microsoft artifact registry

Las empresas suelen tener sus propios registros de imágenes de contenedor, para software privado que ellas mismas desarrollan y no quieren compartir con el mundo:
- gitlab registry
- nexus 
- artifactory
- Azure container registry
- AWS container registry

Una imagen de contenedor se identifica por una URL:
    https://docker.io/library/ nginx :1.21.0
    <-----registry----------><-repo-><-tag->

Muchos gestores de contenedores (docker, podman, container, crio), me permiten especificar la URL sin incluir el registry:
    nginx:1.21.0
En esos casos, el gestor de contenedores buscará en el registro por defecto.
- docker: docker.io
- podman: quay.io + docker.io

## Qué imagen de contenedor me interesa usar?

Existen 2 tipos de imágenes de contenedor:
- TAGs fijos (apunta siempre a una versión concreta)
                    10-Abril        nginx: 2.0.0
                    10-Marzo        nginx: 1.22.0
                    10-Febrero      nginx: 1.21.2
                    10-Enero        nginx: 1.21.1
- TAGs variables (apunta en cada momento a una diferente)
                    nginx: 1.21
                        15-enero:       1.21 ---> 1.21.1
                        15-febrero:     1.21 ---> 1.21.2
                        15-marzo:       1.21 ---> 1.21.2
                        15-abril:       1.21 ---> 1.21.2
                    nginx: 1
                        15-enero:       1 ---> 1.21.1
                        15-febrero:     1 ---> 1.21.2
                        15-marzo:       1 ---> 1.22.0
                        15-abril:       1 ---> 1.22.0
                    nginx: latest
                        15-enero:       latest ---> 1.21.1
                        15-febrero:     latest ---> 1.21.2
                        15-marzo:       latest ---> 1.22.0
                        15-abril:       latest ---> 2.0.0

** Nota: algunos repositorios/Fabricantes deciden libremente si añadir una versión (tag) llamado "latest" (por convenio) que apunta a la última versión disponible. No es una palabra mágica en el mundo de la contenedorización. Simplemente es un tag que `por convenio` algunos fabricantes deciden crear apuntando a la última versión disponible.

Por defecto, los gestores de contenedores, si no especificamos un tag, tratan de buscar la imagen de contenedor con el tag "latest"... que ya hemos dicho que pueden no existir.
    https://docker.io/library/ nginx :1.21.0
                               nginx
   Y en una URL como la de la linea de arriba: "nginx":
   - El gestor busca ese repositorio en los registros de imágenes de contenedor por defecto
   - Y en ese repo busca por el tag "latest"

### En un entorno de producción... que tag elegir?

    **nginx: 1.21.0**   si quiero ser ULTRA CONSERVADOR 
    **nginx: 1.21**     Esta es una buena elección en general!
                        Quiero la funcionalidad que ofrece la versión 1.21... y quiero la mayor cantidad de bugs corregidos:
                        1.21.1 -> 1.21.2 (esta tiene menos bugs)
    nginx: 1            ESTO TAMPOCO! El día de mañana, me puede pasar de un nginx 1.21 a un 1.22
                        Qye traerá funcionalidad nueva que NO SE USA!
                        Y puede venir con nuevos bugs.
    nginx: latest       ESTO NUNCA! El día de mañana, me puede pasar a un major diferente... 
                        y que todo deje de funcionar

## Creación de un contenedor

Implica elegir una imagen de contenedor, descargarla, descoprimirla.
Ese proceso lo hacemos mediante el gestor de contenedores.
En muchos momentos querremos modificar la configuración de serie que trae la imagen de contenedor:
- Variables de entorno (Todo programa preparado para contenedores, tiene variables de entorno para configurar)
- Inyectar archivos de configuración específicos dentro del contenedor

Posteriormente puedo ejecutar un programa dentro de ese contenedor.

Qué comando uso para arrancar un nginx? NPI
Y para pararlo? NPI
Qué comando uso para arrancar un sonarqube? NPI
Y para pararlo? NPI
Qué comando uso para arrancar un mongo? NPI
Y para pararlo? NPI
En qué carpeta se guardan los logs de un mongo? NPI
En qué carpeta se guardan los logs de un sonarqube? NPI

Esos comandos vienen configurados en la imagen de contenedor (metadatos).
Esos implica:
Yo lo que haré será pedir al gestor de contenedores que arranque el programa que viene configurado de serie en la imagen de contenedor (con el comando de arranque que viene configurado de serie en la imagen de contenedor).
Y ya se encargará el gestor de contenedores de arrancar el programa en un proceso aislado dentro del sistema operativo.

La OPERACIÓN DE software instalado en un contenedor, está ESTANDARIZADA!
Con independencia del programa que esté instalado en el contenedor:
- La forma de arrancar ese programa será siempre la misma si trabajo con contenedores:
  - Gestor de contenedores: Ejecuta el programa que viene configurado de serie en la imagen de contenedor
- La forma de parar ese programa será siempre la misma si trabajo con contenedores:
  - Gestor de contenedores: Para el programa que viene configurado de serie en la imagen de contenedor
- La forma de reiniciar ese programa será siempre la misma si trabajo con contenedores:
  - Gestor de contenedores: Reinicia el programa que viene configurado de serie en la imagen de contenedor
- La forma de revisar los logs de ese programa será siempre la misma si trabajo con contenedores:
  - Gestor de contenedores: Muestra los logs del programa que viene configurado de serie en la imagen de contenedor

Y esto tiene una implicación  DIRECTA: 
Si la forma de operar el software es siempre la misma, con independencia del software que esté instalado en el contenedor, entonces puedo AUTOMATIZAR la operación del software.

## Docker: Un gestor de contenedores

No fué el primero.. ni el último... pero lo hicieron muy bien.
En el año 2013 apareció docker y revolucionó el mundo de la contenedorización:
Por crear estándares:
- Definir una imagen de contenedor
- Definir un registro de imágenes de contenedor
- Definir un despliegue contenedorizado de una app.
Rápidamente firmaron acuerdos con:
- Red Hat
- Microsoft
- AWS

Originalmente era una startup... y todos sus programas Open Source.
Parte de docker (del producto) fue donado a una fundación: Open Container Initiative (OCI).
Esa parte del producto es la responsable de:
- Crear contenedores, arrancarlos, monitorizarlas: Containerd

## Comandos de docker

$ docker <TIPO DE OBJETO> <VERBO> <ARGUMENTOS>

TIPO DE OBJETO:     VERBOS

image               list    rm  inspect     pull
container           list    rm  inspect     start stop restart logs create
volume              list    rm  inspect
network             list    rm  inspect

## Alias de docker
                                            ALIAS
docker container list                       docker ps
docker image list                           docker images
docker container start                      docker start
docker image rm                             docker rmi

## Comunicaciones con contenedores

        +---------------------------------------------------------+ red de mi empresa (192.168.10.0/24)
        |                                                         |
        192.168.10.100                                          MenchuPC
        |           :8083 ---> 172.17.0.2:80 (NAT)
        |
        IvanPC                  mi-nginx (nginx:latest)
        |   |                   |
        |   172.17.0.1         172.17.0.2:80
        |   |                   |
        |   +-------------------+--------------- (docker)
        |
        127.0.0.1 (localhost)
        |
        | (loopback)

$ docker container create --name mi-nginx -p IP:8083:80 nginx:latest
$ docker container create --name mi-nginx -p 8083:80 nginx:latest
    ---> El valor por defecto de la IP es 0.0.0.0 (una máscara)
    Que abre el puerto 8083 en todas las IPs del host

Las redirecciones de puertos las usamos con 2 finalidades en docker:
- Que externamente a un contenedor, se pueda acceder a un puerto de un contenedor
- Facilitarme el acceso local a un puerto de un contenedor: al poder usar la palabra "localhost" en lugar de la IP del contenedor (que a priori desconozco.. y tendría que consultar con un docker container inspect)

---

# Kubernetes

Kubernetes es un ESTANDAR ! para montar/diseñar un sistema en un entorno de producción, con software desplegado en contenedores.
Hay muchas implementaciones:
- K8s: Kubernetes más conocido
- K3s: Implementación de kubernetes para entornos de edge computing
- Minikube: Implementación de kubernetes para entornos de desarrollo
- Openshift: Implementación de kubernetes de Red Hat
- Tanzu: Implementación de kubernetes de VMware
- Rancher: Implementación de kubernetes de Suse
- Karbon: Implementación de kubernetes de Nutanix

Cluster:
    Nodo maestro 1          KUBERNETES
    Nodo maestro 2          KUBERNETES
    Nodo maestro 3          KUBERNETES
    Nodo1                   Aplicaciones
    Nodo2                   Aplicaciones
    Nodo3                   Aplicaciones
    ...
    Nodo n                  Aplicaciones
    

Aprovechándonos de la estandarización de la operación de software en contenedores, Kubernetes nos permite:
AUTOMATIZAR LA OPERACIÓN DE SOFTWARE EN UN ENTORNO DE PRODUCCIÓN

Antiguamente esta operación la hacían seres humanos... ADMINISTRADORES DE SISTEMAS.
Hoy en día ese trabajo lo hace kubernetes... ya no lo hacen los administradores de sistemas.

Nosotros le pediremos a kubernetes:
- Oye Kubernetes! quiero tener un mariadb en mi entorno de producción
- Y lo quiero con 3 réplicas
- Y quiero que cada una guarde sus datos en un volumen persistente diferente de una cabina de almacenamiento
- Y quiero un balanceador de carga que me reparta el tráfico entre las 3 réplicas
- Y quiero que si una réplica falla, me la levantes automáticamente en otro nodo
- Y quiero que si la carga de trabajo aumenta, me levantes más réplicas automáticamente
- Que yo me voy de baretas !!!! A partir de ahora, que ya te he explicado LO QUE QUIERO, es tu marrón. NO EL MIO !

Los entornos de producción, hoy en día están operados por Kubernetes y no por seres humanos.

Por cierto... una curiosidad... que nos vuelve locos: Con kubernetes hablamos usando lenguaje DECLARATIVO.
Lo que vamos a indicar a Kubernetes es COMO QUEREMOS EL ENTORNO DE PRODUCCION.
Kubernetes es quién montará y mantendrá ese entorno de producción de acuerdo a nuestras indicaciones.

Una de las cosas que hemos de aprender en la formación es a cómo pedirle a kubernetes que monte un entorno de producción según una especificación que le daremos.

Para configurar/crear un programa, usualmente (desde hace más de 50 años) hemos creado scripts.
Indicando las tareas que DEBIAN REALIZARSE sin intervención manual.

Esos scripts (programas) los escribimos usando un paradigma de programación principalmente IMPERATIVO.
Hoy en día a los administradores de sistemas, les hemos cambiado el contrato (por detrás y sin previo aviso).
Ya no pone en su contrato Administrador de sistemas... ahora pone PROGRAMADORES.
Porque hoy en día los administradores de sistemas, no administran sistemas... sino que escriben programas que administran sistemas.
Esos programas antiguamente los escribían en bash, python, perl, ruby, powershell... EN LENGUAJES IMPERATIVOS.
Básicamente dando una tras otra en ORDEN las instrucciones que debían ejecutarse por la computadora.

Poco a poco, en el mundo de la programación nos hemos dado cuenta de que el lenguaje imperativo es un MIERDOLO ! Y CADA DIA LO ODIAMOS MAS.

Esto de los paradigmas de programación NO ES SINO UN NOMBRE HORTERA QUE LOS DESARROLLADORES DE SOFTWARE LE HAN PUESTO A la forma en la que usamos un lenguaje para expresar ideas...
Pero eso existe también en los lenguajes humanos.

> FELIPE, IF (SI) hay algo debajo de la ventana que no sea una silla
    > FELIPE, QUITALO DE AHÍ                            IMPERATIVO
> FELIOPE (IF) no hay una silla debajo de la ventana
    > FELIPE, IF NO HAY SILLAS (Silla = false)
    > VETE AL IKEA y compra SILLA                       IMPERATIVO
    > FELIPE, pon una silla debajo de la ventana.       IMPERATIVO

El problema del paradigma imperativo es que me descentro de mi objetivo... centrándome en lo que se debe hacer para llegar a él.

> FELIPE, quiero que haya una silla debajo de la ventana (DECLARATIVO)
> Y a partir de ahora... es tu problema ! Tú sabrás qué debes hacer para conseguir ese estado final. PERO NO ME CUENTES TU VIDA !
---

# Qué es AUTOMATIZAR?

Diseñar una máquina (o cambiar el comportamiento de una que exista, por ejemplo mediante un PROGRAMA) para que realice tareas que antes realizaban seres humanos.

Ejemplo: Puedo automatizar el lavado de la ropa: LAVADORA

---
## Entornos de producción

Qué diferencia un entorno de producción del resto de entornos (desa, test, preprod)?
con qué características debe cumplir un entorno de producción?
- Alta disponibilidad
  Tratar de garantizar un determinado tiempo de servicio (pactado previamente).
    90%         36,5 días al año el sistema offline                         | €
    99%         3,65 días al año el sistema offline (PELUQUERIA DE BARRIO)  | €€
    99,9%       8,76 horas al año el sistema offline (TIENDA ONLINE)        | €€€€€
    99,99%      52,56 minutos al año el sistema offline (BANCO)             | €€€€€€€€€€€
    99,999%     5,26 minutos al año el sistema offline (HOSPITAL)           | €€€€€€€€€€€€€€€€€€€€€€€€€€€€€€€
- Escalabilidad                                                             v REDUNDANCIA
    Capacidad de ajustar la infra a las necesidades de cada momento

        App1: No necesito escalabilidad
            Día 1:      100 usuarios
            Día 100:    100 usuarios
            Día 1000:   100 usuarios
        App2: 
            Día 1:      100 usuarios        ESCALABILIDAD VERTICAL: Más máquina
            Día 100:    1000 usuarios
            Día 1000:   10000 usuarios
        App3: ESTO ES LO MAS HABITUAL HOY EN DÍA (Internet)
            Día n:      100 usuarios
            Día n+1:    1000000 usuarios        Black Friday
            Día n+2:    1000 usuarios
            Día n+3:    10000000 usuarios       Ciber Monday

            Web telepi:
                00:00        0 usuarios
                 6:00        0 usuarios 
                11:00        5 usuarios
                13:00      500 usuarios         ESCALABILIDAD HORIZONTAL: Más máquinas
                14:00     5000 usuarios
                16:30       10 usuarios
                20:30     100000000 usuarios
                23:00        0 usuarios

        Quien me ofrece hoy en día la posibilidad de hacer esto? CLOUDS (AWS, Azure, GCP...)

        En cualquier caso, aquí aplicamos también el concepto de REDUNDANCIA

    CLUSTER: Grupo de máquinas /programas que corren en esas máquinas ofreciendo el mismo servicio.
        Activo/pasivo: 1 máquina activa, el resto están a la espera de que la activa falle
        Activo/activo: Todas las máquinas están activas, y se reparten el trabajo

    Servidor 1: (App1)
        CPU: 55%
    Servidor 2: (App1)
        CPU: 55%
    Estoy en riesgo absoluto.
    La alta disponibilidad es MUY CARA !
- Estabilidad

## ARQUITECTURA DE UN ENTORNO DE PRODUCCION

    BBDD (IP: 192.168.100.20 / fqdn: mibbdd.app1)
        A ésta solo pueden atacarla desde 192.168.100.101 y 192.168.100.102
    Servidor1 (IP: 192.168.100.101) <<<                                             192.168.10.10
        Tomcat                          BALANCEADOR DE CARGA  <<< PROXY REVERSO  <<< PROXY  <<<<    Usuario
    Servidor2 (IP: 192.168.100.102) <<<  (IP: 192.168.100.100)    (IP: 192.168...90)            92.168.50.101
                                                                                                www.app1.com
        Tomcat

    DNS público (resolverá app1.com a la IP que le toque)
    DNS privado (resolverá mibbdd.app1 a la IP que le toque)
    Firewall... para limitar el acceso a los puertos/IPs de ciertas máquinas

    Y los datos de los tomcat??? Dónde los guardo?
    - OPCION 1: En unos HDD que tengo en los servidores
    - OPCION 2: En una cabina de almacenamiento <<< ésta tiene ventajas.

    Proxy: Proteger al solicitante (cliente) en una comunicación
    Proxy reverso/inverso (nginx, haproxy, traefik, envoy, apache): Proteger al servidor en una comunicación

De estos conceptos es de los que hablamos en Kubernetes
                        KUBERNETES
- Servidores:           Pods
- Balanceador de carga: Service
- DNS:                  Service
- Proxy reverso:        Ingress Controller
- Almacenamiento:       Persistent Volume
---

El almacenamiento es caro o barato hoy en día? EL ALMACENAMIENTO ES LO MAS CARO CON DIFERENCIA EN UN ENTORNO DE PRODUCCIÓN:
- Los HDD de un entorno de producción son varias veces más caros que los domésticos.

Cuántas copias de un dato hago en un entorno de producción? Al menos 3.

Para casa, 1 tb me cuesta 50€
En un entorno de producción, 1 tb (en un disco caro) me cuesta 300€
    Pero necesito 3 copias: 900€
    Y ahora backups: 1800€
Se acepta... al fin y al cabo, el DATO Es lo UNICO QUE APORTA VALOR A NEGOCIO !

---

# Kubernetes/Openshift

- Sistemas
- Orquestación de servidores
- Microservicios
- Escalabilidad
- Aislamiento
- Seguridad
- Replicación
- Alta disponibilidad
- Portabilidad

----

# Linux?

Es un kernel de Sistema Operativo.
Los sistemas operativos hoy en día todos se construyen basándose en un kernel.

De hecho es el kernel de SO más usado del mundo.

Sistema operativo GNU/Linux. Se ofrece en diferentes distribuciones:
- RHEL/Fedora/CentOS/Oracle Linux
- Debian/Ubuntu
- Suse
- Arch

---

# Qué era Unix?

Un sistema operativo creado por la gente de los lab Bell de AT&T en los 70s.
Se dejó de hacer a principios de los 2000.

# Qué es Unix?

Hoy en día son 2 estándares que definen una forma (puede haber otras) de hacer un sistema operativo:
- Posix
- Single Unix Specification

Hoy en día un SO Unix, es un sistema que cumple con estos estándares.
- HP: HP-UX (UNIX®)
- IBM: AIX (UNIX®)
- Oracle: Solaris (UNIX®)
- Apple: MacOS (UNIX®)

Hay gente que creó sistemas operativos basados en esos estándares, pero que no se certificaron... De hecho, incluso empezaron a evolucionar de forma diferente.
- BSD
- GNU/Linux

# Posix???

Dentro de posix se definen muchas cosas:
- Usuario/Grupo/Permisos a nivel de archivos
- Variables de entorno
- Comandos de shell (cp, mv, cat...)
- Estructura del sistema de archivos
   /
     boot/
     bin/               Comandos de sistema (cp, mv, ls, cat, ...)
     etc/               Configuraciones del sistema operativo y de las aplicaciones
     var/               Archivos de datos de los programas: logs, bases de datos, documentos html, ...
     home/              Directorios de los usuarios (equivalente a C:\Users\ en Windows)
     root/              Directorio del usuario root
     opt/               Aplicaciones de terceros
     usr/
     lib/

---

# Nginx: Proxy reverso con capacidades de servidor web

Imágenes de nginx:
nginx:1.19.10
nginx:1.19.10-perl
nginx:1.20.0
nginx:1.21.0
nginx:1.21.0-perl
nginx:1.21.0-php
nginx:1.21.0-php-debian
nginx:1.21.0-php-alpine

---

# Esquema de versionado SEMANTICO (Semantic Versioning: semver)

Versión de software: 1.2.3

              Cuándo se incrementan?
1 = MAJOR     Breaking changes
              Cambios que hacen que la nueva versión no sea compatible con la anterior
              (básicamente es quitar cosas del api del programa)
2 = MINOR     Nuevas funcionalidades
              O/Y funcionalidades marcadas como obsoletas (deprecated)
               (adicionalmente, en un nuevo minor, pueden venir bugs corregidos)
3 = PATCH     Corrección de bugs

---

En esta formación:
- Debemos entender en profundidad el concepto de contenedor
- Cómo trabajar con contenedores en local (docker)
- Cómo trabajar con contenedores en un entorno de producción (kubernetes/openshift)

---

# Qué entendemos por DEVOPS?

Es una cultura, es una filosofía, es un movimiento en pro de la automatización. Automatización de qué?
DE TODO LO QUE HAY entre el DEV ---> OPS

- Grupo de tareas que implican a los desarrolladores y sysadmins
- No tener que repetir tareas que siempre son iguales de manera manual: AUTOMATIZAR

PLAN
CODE
BUILD
--------> Desarrollo ágil:
            Si consigo automatizar el empaquetado (BUILD) de un producto de software
TEST
--------> Integración continua:
            Si consigo tener CONTINUAMENTE la última versión desarrollada del producto en un entorno DE INTEGRACION sometido a pruebas automatizadas.
RELEASE
--------> Entrega continua: (CONTINUOUS DELIVERY)
            Si consigo tener CONTINUAMENTE la última versión desarrollada y probada del producto en manos de mi cliente
DEPLOY
--------> Despliegue continuo: (CONTINUOUS DEPLOYMENT)
            Si consigo tener CONTINUAMENTE la última versión desarrollada y probada del producto en producción
OPERATE
MONITOR
--------> Devops:
            Si consigo tener CONTINUAMENTE la última versión desarrollada y probada del producto en producción y monitorizado

Kubernetes es una pieza clave en la automatización del flujo de vida de un producto de software.
Entra en: DEPLOY, OPERATE, MONITOR

NOTA IMPORTANTE:
Quién sabe cómo hacer un despliegue de una app en un entorno de producción? (YA QUE ESTO ES LO QUE HAY QUE CONTARLE A KUBERNETES)
- Desarrolladores? Pero ... quién sabe si una app necesita una determina versión de Oracle, con una configuración activada particular... y unas variables de entorno configuradas con unos valores concretos?
- Administradores de sistemas? Proxies reversos, balanceadores de carga, firewalls, almacenamiento, DNS

La adopción de una herramienta como kubernetes (o sus variantes: Openshift, tanzu...) tarda AÑOS en una empresa.
Implica:
- Crear departamentos nuevos a nivel organizativo
- Formar a un huevo de personas en kubernetes (en distintos aspectos de kubernetes)
- Implica migrar todas las aplicaciones a contenedores
- Implica cambiar los procedimientos de trabajo de los desarrolladores y sysadmins

Esto lleva años ! (no menos de 2-3)

