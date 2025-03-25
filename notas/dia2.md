
# Concepto de contenedor

Es un entorno aislado dentro de un sistema operativo (Linux) donde puedo ejecutar procesoS.
AISLADO? En cuánto a qué?
√ Tiene su propia configuración de red -> Su propia IP
- Tiene su propio sistema de archivos (HDD)
√ Tiene sus propias variables de entorno
- Puede tener limitaciones de acceso al hardware

        +---------------------------------- red amazon
        |
        IvanPC
        |
        +---+------------------------------ red docker
            |
            172.17.0.2
            |
            Contenedor con Nginx 
            |
            proceso Nginx <- Abre el puerto 80

Cómo puedo ver los procesos que tengo corriendo en un host linux?

$ ps -eaf

Si ejecuto este comando, a nivel del host, veré los procesos que están corriendo dentro del contenedor o no?        SI

Si tengo un Windows con Virtualbox... Y dentro del virtualbox tengo una VM Linux Ubuntu.,. en la que está corriendo un proceso, Desde el windows (administrador de tareas) puedo ver los procesos que están corriendo en la VM ubuntu? NI DE COÑA

La vida de un contenedor va asociada a la vida del proceso inicial que se abre en ese contenedor cuando ejecutamos el docker container start (CMD).

## Variables de entorno

Un contenedor tiene sus propias variables de entorno.
Vienen muchas de ellas preconfiguradas en la IMAGEN DEL CONTENEDOR.
Cuando creo (y solo cuando creo) y contenedor puedo cambiar esas variables de entorno... 

$ docker container create --name mi_contenedor -e VAR1=VALOR1 -e VAR2=VALOR2 mi_imagen

También puedo cambiarlas en caliente... pero con un reinicio del contenedor pierdo los valores... ESTO NO LO HACEMOS NUNCA!
---

Podemos abrir nuevos procesos dentro de un contenedor. Con docker:

docker container exec <nombre> <comando>

---

# Qué ocurre cuando creamos un contenedor

1. Me descargo la imagen del contenedor y la descomprimo en el disco.
   La imagen del contenedor descomprimida es INMUTABLE

/
    bin/
    etc/
    lib/
    usr/
        bin/
            ls
    var/
        lib/
            docker/
                containers/
                    mi-nginx/   ESTA CARPETA CONTIENE LOS CAMBIOS que se van haciendo sobre la estructura de archivos que venía en la imagen
                        
                        tmp/
                            miArchivo.txt
                    mi-nginx2/
                images/
                    nginx/                      
                                El contenido de esta carpeta NO PUEDE MODIFICARSE. NADIE PUEDE MODIFICARLA
                        bin/
                        etc/
                            nginx/
                                nginx.conf
                        lib/
                        usr/
                            bin/
                                ls
                        var/
                        home/
                        root/
                        tmp/
                        dev/
                        opt/
                             nginx/
                                    nginx
    home/
    root/
    tmp/
    dev/

2. Creo un contenedor
3. Docker crea una "carpeta" para guardar los cambios que se hagan sobre el sistema de archivos que venía en la imagen del contenedor


4. Lo arranco. Al arrancarlo se crea a nivel de SO un proceso para el contenedor... éste proceso a su vez que es el que se encarga de ejecutar el proceso inicial del contenedor (CMD: nginx -g daemon off;)
5. Lo que hace un gestor de contenedores es engañar a ese proceso CONTENEDOR para que piense que el root (la raíz) del sistema de archivos es la superposición de la carpeta del contenedor sobre la carpeta de la IMAGEN del contenedor
    De hecho este truco rastrero lo llevamos haciendo más de 40 años.
        $ chroot


---

# Persistencia de datos en contenedores

Los cambios que voy haciendo sobre el sistema de archivos de un contenedor se persisten a disco FISICO... y tienen la misma persistencia que los datos que tengo en mi disco duro.
Y pasa lo mismo que con los hosts o las vms. Mientras no borre el contenedor, los datos que he creado en él se mantienen. Igual que mientras no borre mi host o una vm, los datos que he creado en ellos se mantienen.

Dónde está el tema aquí?
- Cuántas veces borro un host? CUANDO YA VOY A DESMANTELAR UN PROYECTO
- Cuántas veces borro una VM?  CUANDO YA VOY A DESMANTELAR UN PROYECTO
- Cuántas veces borro un contenedor? de 1 a 200.000 al día . NPI!
  De continuo.

# Escenarios:

## Escenario 1
    CLUSTER VMWare:
        EsXi1 - Se jode
            VM1 -> La migro al EsXi2
        EsXi2

            Los archivos de disco de la VM se guardan en un almacenamiento (datastore) fuera del host EsXi1 o EsXi2
            Lo único que se hace es montar en el host EsXi2 el disco de la VM que estaba en el EsXi1 y arrancar la VM.
    CLUSTER MAQUINAS
        Host Linux 1 < Se jode
            Contenedor 1 -> Lo migro al Host Linux2 ? Cómo se hace?
                Realmente lo que hacemos es MATAR / BORRAR el contenedor 1 (si tengo acceso al host)
        Host Linux 2
            Creo un contenedor NUEVO
## Escenario 2
    Tengo una VM con oracle 19.2 -> Oracle 19.3
    Qué hago?
        - Opción 1: Entro en la VM y actualizo el oracle
        - Opción 2: Borro esa VM y creo una nueva con la nueva versión
          Pero para poder hacer esto necesito tener los datos de la BBDD en un almacenamiento externo a la VM.

    Tengo un contenedor con oracle 19.2 -> Oracle 19.3
    Qué hago?
        - Borro el contenedor y creo uno nuevo con la nueva versión
          Pero para poder hacer esto necesito tener los datos de la BBDD en un almacenamiento externo al contenedor.

Al trabajar con contenedor, hacemos lo mismito que con VMs o hierros.
Solo que NUNCA NOS PLANTEAMOS el escenario de entrar dentro de un contenedor a tocar los programas que ahí vienen.
Siempre borramos y creamos contenedores desde 0:
- Bien sea para actualizar la versión del programa que viene dentro
- Bien sea para migrar un contenedor a otro host por perdida de un host
- Bien sea porque quiero escalar un servicio y posteriormente volver a escalarlo a la baja
- Bien sea porque migrar un contenedor a otro host por saturación de recursos
El concepto de migrar/mover un contenedor no existe tal cual. Lo que hacemos realmente SIEMPRE es borrar el contenedor y crear uno nuevo.


Esto implica que los datos que realmente quiero tener persistidos incluso tras la muerte (eliminación) del contenedor, los tengo que tener en un almacenamiento externo al contenedor.

Esto lo puedo hacer tranquilamente con Linux pelao! mount en el sistema de archivos.
En los contenedores hago lo mismo... solo que el mount me lo regala el gestor de contenedores: VOLUMEN.

Un VOLUMEN en contenedores no es sino un almacenamiento externo al contenedor que se monta en el sistema de archivos del contenedor.

Donde está ese almacenamiento externo?
- Lo puedo tener en el host (Esto es lo más habitual si trabajo con docker)
  Con persistencia en disco o en RAM. 
- Lo puedo tener en un almacenamiento en la nube
- Lo puedo tener en una cabina de discos
- Lo puedo tener en un NAS
- Lo puedo tener en un cluster de almacenamiento

Docker, que no es una herramienta pensada para HA (hay un engendro llamado docker SWARM que intenta hacer algo en este sentido... y que pasamos de ello en favor de kubernetes, que ofrece 500 veces más funcionalidad que la ofrece docker SWARM) me permite usar tipos de volúmenes de almacenamiento muy limitados.

Kubernetes me ofrece una cantidad enorme de tipos de volúmenes de almacenamiento.
Y además, cualquier fabricante de cabinas de discos o de NAS o de almacenamiento en la nube, tiene un plugin para kubernetes que me permite usar sus volúmenes de almacenamiento en kubernetes.

## Para qué sirven los volúmenes?

- Persistir datos tras la muerte de un contenedor
- Para compartir datos entre varios contenedores
- Para poder inyectar archivos/carpetas dentro de un contenedor para que los programas que ahí corren tengan acceso a ellos (configuración).

Los volúmenes pueden ser DIRECTORIOS o ARCHIVOS.

# Monitorización de contenedores

## Cómo sabe un gestor de contenedores si un contenedor está vivo o muerto?

Se mira el proceso inicial que se arranca desde el proceso del contenedor.
Ese proceso es el que se monitoriza.
Si está corriendo, se entiende que el contenedor está vivo.
Si no está corriendo se entiende que el contenedor está parado/muerto.
    En este caso, se captura el código de salida de ese proceso y se guarda.

## Qué son los logs de un contenedor?

Se entiende como el log de un contenedor la SALIDA ESTANDAR y DE ERROR del proceso inicial que se arranca en el contenedor.

En un contenedor el proceso principal arranca en PRIMER PLANO... y va escribiendo en la salida estándar (terminal) y de error.

Esa información (stdout/stderr) es lo que el gestor de contenedores captura y considera el LOG del contenedor.

## Es suficiente con monitorizar si un proceso está corriendo para saber si realmente está realizando su trabajo de forma efectiva?

NO...

Y cómo puedo asegurarme de ello? DEPENDE del proceso que esté corriendo en el contenedor.

Si el proceso abre un puerto, asegurándome que el puerto está abierto... y que contesta a las peticiones que se le hacen... puedo asegurarme de que el proceso está funcionando correctamente.

Quizás estoy ejecutando un proceso que NO ABRE PUERTOS (un backup de bbdd).
Ese proceso abre un puerto?  NO
Quizás ese programa me ofrezca un mecanismo para saber si está funcionando correctamente... 
- va escribiendo en un archivo el estado y puedo monitorizar ese archivo.
- quizás hay un comando adicional para saber si la copia sigue en marcha o no.

En DOCKER existe un mecanismo llamado HEALTHCHECK que me permite definir un comando que se ejecuta de forma periódica dentro del contenedor para saber si el proceso está funcionando correctamente o no. ESTO NO ES ALGO ESTANDAR en el mundo de los contenedores. Es algo que ofrece docker... bastante CUTRE POR CIERTO.

KUBERNETES nos da algo similar... pero infinitamente más potente. Es que Kubernetes está pensado para entornos de producción reales, con HA de verdad de la buena!

Evidentemente, más potencia suele implicar más complejidad. Pero es que el concepto en sí, ya es complejo.... harto complejo!

En Kubernetes existe el concepto de PROBE. De hecho, hay 3 tipos de PROBES:
- StartupProbe      Sirve para determinar si un proceso ha arrancado correctamente o está en ello
- LivenessProbe     Sirve para determinar si un proceso está vivo y saludable o no
- ReadinessProbe    Sirve para determinar si un proceso está listo para atender peticiones o no

Arranco un contenedor con una BBDD. Cuánto puede tardar eso en arrancar?
Me puede llegar incluso a minutos.
Pero puede ser más cruento. Imaginad que arranco un contenedor de Oracle... que acabo de crear desde una imagen Oracle 19.3... enchufándole unos datos de un contenedor que tenía de antes que usaba una imagen de Oracle 19.2.
No es sólo que arranque... es que además se va a meter en un proceso de actualización de la BBDD... que puede tardar bastantes minutos...

Definiré una prueba de arranque (startupProbe) que me permita saber si el proceso de arranque ha terminado o no.
    Y configuraré un timeout GRANDE para esta prueba.

Una vez ha arrancado, lo que miraré es si está vivo y saludable (livenessProbe)... y esa prueba la haré con una periodicidad de 5 segundos.
    Y si en 2 segundo No contesta... y así 3 veces... ALARMAS !!!!!

Ahora, que un proceso esté vivo y saludable, implica que esté listo para recibir peticiones?    NO

Imaginad que tengo una BBDD que de repente entra en modo mnto... para hacer un backup en frío. Y me pongo a hacer el backup.
Está vivo y saludable... haciendo el backup... pero no está lista para atender peticiones.
-> Mato el proceso? NO
-> Qué hago entonces? Lo saco de BALANCEO!

---

En docker hub tenemos unas imágenes llamadas:
- Ubuntu 20Mb
- Debian
- Fedora
- Alpine 3Mbs

cuando creamos una imagen de contenedor propia (por ejemplo soy NGINX y quiero fabricar una imagen de contenedor con el NGINX instalado), lo que hago es partir de lo que se llama una IMAGEN BASE.
Esas imágenes de arriba son IMÁGENES BASE de contenedor, que usamos para crear otras imágenes de contenedor.
Qué viene en ellas?
- Una estructura de carpetas POSIX estándar
- Los programas comunes de LINUX (Según POSIX):
  - ls
  - cat
  - cp
  - mv
  - rm
  - mkdir
  - sh
  - wget
- Y otros programas que puedan ser de interés... propios de cada distro:
  - Ubuntu: apt + bash
  - Fedora: dnf + bash
  - Alpine

---

docker container create \
    -e VARIABLE1=VALOR1 \
    --name mi-nginx \
    -p 0.0.0.0:8083:80 \
    -v /home/ubuntu/environment:/tmp2 \
    nginx

En ocasiones tendremos contenedores que:
- Necesiten 20-200 variables de entorno (elastic search)
- Necesiten 5-10 volumenes
- Abran 4, 5 puertos
- ...
Y me saldrá un comando de narices!

Además... qué tipo de lenguaje (paradigma de programación) uso al escribir uno de esos comandos?

Docker ofrece una forma de definir contenedores usando un lenguaje DECLARATIVO: DOCKER COMPOSE

Adoramos docker compose. Habitualmente no tiramos ahí comanditos para crear contenedores.

Antiguamente era un programa independiente: $ docker-compose
Ahora es un subcomando de docker:           $ docker compose

Cuando trabajo con compose usamos un fichero YAML para definir (declaro) los contenedores que queremos manejar.

Posteriormente aplico ese comando docker compose sobre ese fichero YAML, y le pido que ejecute una acción:
- docker compose up
  - Crea los contenedores (si es necesario)
  - O los borra y los recrea (si han sufrido cambios)
  - Los arranca (si es necesario)
- docker compose down
  - Para los contenedores (si es necesario)
  - Los borra (si es necesario)
- docker compose stop
  - Para los contenedores (si es necesario)
- docker compose start
  - Arranca los contenedores (si es necesario)
- docker compose restart

Ese comando busca por defecto un fichero llamado docker-compose.yml en el directorio donde se ejecuta.
Pero puedo especificar otro fichero con la opción -f

    $ docker compose -f mi-fichero.yml up

Cuando trabajo con docker compose HABLO DE ESTADOS FINALES... como quiero el sistema.
Ya docker se encarga de llevarme al estado final que le he pedido.

Además es mucho más cómodo usar un fichero de texto plano (que puedo subir a un sistema de control de versiones) que no tener que andar con comanditos de narices sueltos por ahí en una terminal.

Hay varios gestores de contenedores que han adoptado el formato de docker compose:
- Podman

Kubernetes hace algo parecido. También nos permite usar un lenguaje declarativo para definir el estado final que queremos en nuestro sistema.
Y también usa ficheros YAML para ello.
Solo que los ficheros YAML de kubernetes son harto más complejos que los de docker compose.
El mismo despliegue puede ocupar:
- 30 lineas en docker compose
- 300-1000 lineas en kubernetes

Eso si... no va a ser realmente el mismo despliegue. Kubernetes me ofrecerá 50 veces más funcionalidad.

--- 

# YAML?

Yaml es un lenguaje de marcado de información de propósito general.
como por ejemplo XML, JSON, CSV, INI.

YAML comparte letras (en sus siglas ) con? XML, HTML... 
    ML -> Markup Language

Yet another markup language... pero va a ser que no. Habría sido una posible explicación de su nombre... pero no es la real.

YAML es un acrónimo recursivo:
    YAML Ain't Markup Language

Además YAML tiene mucha más funcionalidad que otros lenguajes como por ejemplo JSON. Es más YAML se ha comido a JSON (literalmente). Hoy en día JSON es parte de la especificación de YAML (que está en la versión 1.2).

---

# Kubernetes

En kubernetes no existe el concepto de CONTENEDOR.
En su lugar existe el concepto de POD.

No es la única herramienta en la que existe ese concepto: podman
(pod manager)

Docker no tiene el concepto de POD. Docker tiene el concepto de CONTENEDOR.

## Qué es un pod?

Un pod es un CONJUNTO de CONTENEDORES (puedo tener un conjunto con solo 1 contenedor), que:
- Comparten configuración de red (y por ende IP... y por ende pueden hablarse entre ellos por localhost)
- Se garantiza que en un cluster de kubernetes, los pods de un conjunto se ejecutan en el mismo host físico
- Pueden compartir almacenamiento local (a nivel de host)
- Y escalan juntos
    Si de un contenedor de un pod quiero hacer una replica... no puedo.
    Lo que hago la replica es del pod... del paquete entero.

Imaginad que:
Quiero instalar un WORDPRESS:
- Servidor web httpd
- Servidor de BBDD mysql

Preguntas:

1. Quiero 1 contenedor o 2?
    - Pongo bbdd y servidor web en el mismo contenedor?
    - Pongo bbdd y servidor web en contenedores separados?
    Separados... por qué?
    - Actualizaciones
    - Escalabilidad
    
    !!!!! SIEMPRE: Cada programa en su contenedor: SIEMPRE !!!!

2. 1 pod o 2 pods?
   - Pongo 1 pod con 1 contenedor de httpd y 1 contenedor de mysql?
   - Pongo 2 pods: 
     - 1 con 1 contenedor de httpd 
     - otro con 1 contenedor de mysql?
    En 2 pods... 
    quiero que escalen por separado
    Y no hay necesidad de que compartan ip, ni volumenes locales... ni nada de eso.


----

# Escenario 2.

ElasticSearch / Kibana Indexador.
Se usa mucho para monitorización de logs.

                Nodo1
                    Httpd1
                        /var/log/httpd/error.log
                    Filebeat        >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
                Nodo2
BBDD                Httpd2
                        /var/log/httpd/error.log                                ESearch < Kibana
                    Filebeat        >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
                Nodo3
                    HttpdN
                        /var/log/httpd/error.log
                    Filebeat        >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

Filebeat + Httpd

1 contenedor o 2?
    !!!!! SIEMPRE: Cada programa en su contenedor: SIEMPRE !!!!

1 pod o 2 pods? 1 pod
    x Comparten configuración de red?
    √ Comparten almacenamiento local
        Montaría un almacenamiento en el host para los logs que monte en ambos contenedores
        Este volumen incluso le podría poner con persistencia en RAM <<< BOOST RENDIMIENTO 
            Le monto al apache 2 logs rotados de 100kbs.
            Ya tengo limitado el uso de ram a 200kb para logs.
    √ Escalan juntos
    √ Despliegue conjunto

En estos escenarios, al contenedor filebeat le llamamos contenedor SIDE-CAR.
    Es un contenedor que acompaña a otro contenedor.
    No es un contenedor que se despliegue por separado.
    No es un contenedor que se despliegue en otro pod.
    Es un contenedor que acompaña a otro contenedor en el mismo pod.