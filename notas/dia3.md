
# Pods

En kubernetes la ud. mínima de trabajo es el pod.
Un pod es un grupo de contenedores (puede ser sólo 1) que:
- Comparten configuración de red, y por ende ip. También pueden hablar entre si usando la palabra localhost.
- Escalan juntos (hacía arriba y hacia abajo).
- Se despliegan en el mismo host.
  -> Pueden compartir volúmenes de almacenamiento locales.

---

# Kubernetes

Más potente que docker: está preparado para entornos de producción.
- Escalabilidad
- Alta disponibilidad
Es un estandar del que hay muchas implementaciones.
La mínima ud de configuración es el POD.

Se instala en varias máquinas:
- Al menos 3 máquinas para el propio Kubernetes
- N nodos (al menos 2) para las apps.

La configuración de kubernetes se hace mediante con ficheros de configuración en formato yaml: Archivos de manifiesto.

# Arquitectura de Kubernetes

Al comenzar a instalar kubernetes, lo que hacemos es montar una herramienta llamada kubeadm. Esa herramienta necesitamos instalarla en cada host que vaya a formar parte del cluster. Además, vamos a montar una segunda herramienta (que queda corriendo como servicio también en cada host: kubelet).
También hace falta montar en cada host un gestor de contenedores (crio, containerd).
- Kubeadm (COMANDO): Nos ayuda con tareas de gestión del cluster:
  - Instalación del cluster
  - Añadir/eliminar máquinas al cluster
  - Generación de certificados SSL
- Kubelet(SERVICIO que queda corriendo en cada host): Se encarga de la comunicación entre los nodos, los programas de kubernetes, y el gestor de contenedores de ese host.

Adicionalmente hemos de instalar lo que se denomina:

## Plano de control de Kubernetes

El plano de control de Kubernetes es un conjunto de programas (internos de kubernetes) que instala el kubeadm. Estos programas se ejecutan COMO CONTENEDORES dentro de los nodos MAESTROS.
Encontramos entre ellos:
- kube-apiserver: Servicio HTTP que permite recibir peticiones de clientes a Kubernetes.
                  Es quién recibe TODAS LAS PETICIONES DE TRABAJO
    (2 copias)
- kube-controller-manager: Se come todo el trabajo:
                - Controla el estado del cluster
                - Controla el estado de los nodos
                - Controla el estado de los pods
                - Solicitar la creación de nuevos pods a los kubelets en cada nodo.
    (2 copias)
- kube-scheduler: Se encarga de decidir en qué nodo se va a colocar un pod.
    (2 copias)
- etcd: Base de datos de Kubernetes. Guarda el estado actual y deseado del cluster.
    (3 copias)
- core-dns: Servicio de DNS para el cluster.
    (2 copias)
- kube-proxy: Servicio que se encarga de enrutar el tráfico de red entre los pods.
              De proxy vamos a tener 1 copia (1 programa en ejecución)  en cada nodo del cluster (maestro y trabajador).
- Algunos adicionales... 

  +--------------------------------------------------------------------- red de mi empresa (192.168.0.0/16)
  |  Cluster
  += 192.168.0.10- Nodo Maestro 1
  ||                   |    LINUX
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.6- apiserver
  ||                   +-10.10.10.2- controller-manager
  ||                   +-10.10.10.3- etcd
  ||                   +-10.10.10.4- core-dns
  ||                   +-10.10.10.5- kube-proxy
  += 192.168.0.11- Nodo Maestro 2
  ||                   |    LINUX
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.7- apiserver
  ||                   +-10.10.10.8- etcd
  ||                   +-10.10.10.9- core-dns
  ||                   +-10.10.10.10- kube-proxy
  += 192.168.0.12- Nodo Maestro 3
  ||                   |    LINUX
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.11- etcd
  ||                   +-10.10.10.12- kube-proxy
  ||
  += 192.168.0.101- Nodo Trabajador 1
  ||                   |    LINUX
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.13- kube-proxy
  += 192.168.0.101- Nodo Trabajador 2
  ||                   |    LINUX
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.14- kube-proxy

    Red virtual interna del cluster: 10.10.0.0/16

## Otros programas importantes

Adicionalmente en el primer nodo maestro que instalo, instalo un programa llamado kubectl.

Es el cliente de kubernetes. Un comando que ejecuto desde una terminal.. y que habla por detrás con el api-server.
Este comando también lo instalaré en máquinas fuera del cluster. En aquellas máquinas desde las que yo quiera dar instrucciones al cluster.

No es el único cliente de kubernetes.
- Kubernetes tiene un proyecto OFICIAL (que no se instala por defecto en un cluster, hay que hacerlo a posteriori) llamado DASHBOARD. Es un cliente WEB que también (por detrás) habla con el API-SERVER
- Openshift tiene su propio dashboard gráfico, mucho más potente que el que trae kubernetes por defecto.
- Además, openshift también tiene su propio cliente de linea de comandos (compatible con kubectl, de hecho tiene exactamente la misma sintaxis que kubectl... aunque agregando alguna funcionalidad adicional). Ese cliente se llama oc.

Puedo usar kubectl también para comunicarme con un cluster Openshift. Y puedo usar oc para comunicarme con un cluster kubernetes. Y esto es porque Openshift es una implementación de kubernetes... vitaminado (con algunos añadidos)

Sea como fuere... al menos en el nodo maestro primero, hemos de instalar también ese cliente.
Y tiene una razón. Hemos de configurar a nivel del cluster una RED VIRTUAL.

Kubernetes requiere una RED VIRTUAL común entre los hosts (es una red virtual) que usará la red FISICA que conecta los nodos. Sobre ella, montamos una red virtual. Y a esa red es a la que se pinchan los PODs. Y por ello pueden comunicarse.

Kubernetes no ofrece una implementación de red privada virtual. Pero hay muchas implementaciones de redes privadas virtuales que se pueden usar con kubernetes. 
Openshift trae una implementación de red privada virtual propia. Y es la que se usa por defecto.

---

En general esto aplica a prácticamente todos los sistemas de almacenamiento de datos (BBDD, Sistemas de mensajería, ElasticSearch, etc).

    Nodo1
        BBDD 1 - DATO2 (id=2), DATO1(id=1)
     v^
    Nodo2                   <<< BALANCEADOR DE CARGA        <<< PETICIONES
        BBDD 2 - DATO1 (id=1), DATO2(id=2)
     v^
    Nodo3 (* MAESTRO)
        BBDD 3 - DATO1 (id=1), DATO2(id=2)

Hago una petición para guardar 1 dato. Le llega al balanceador y la enruta a una instancia del programa de bbdd.        DATO1
En paralelo podría recibir otra petición que se mandase al nodo1... DATO2

Los ids los decide solo 1 de los nodos. En un cluster de almacenamiento, 1 de los nodos actúa como MAESTRO.

    Nodo1
        BBDD 1 - DATO2 (id=2), DATO1(id=1), DATO4(id=3)
     v^
    Nodo2 (* MAESTRO)       <<< BALANCEADOR DE CARGA        <<< PETICIONES
        BBDD 2 - DATO1 (id=1), DATO2(id=2), DATO4(id=3)
     xx
    Nodo3
        BBDD 3 - DATO1 (id=1), DATO2(id=2), DATO3(id=3)

Normalmente los maestros se eligen entren ellos. Hablan y llegan a un acuerdo.. Pues TU Nodo 3 ... te ha tocao!
Pero si nodo3 pierde comunicación con nodo 1 y nodo 2. Nodo 1 y nodo 2, se pondrían a hablar entre ellos y elegirían un nuevo maestro: NODO 2

Ese fenómeno (muy difícil de solucionar) se llama SPLIT BRAIN. Si esto ocurre, estamos perdidos. Puedo tirar la BBDD a la basura. Los datos son irreconciliables.

Hay que evitar esto. Qué hacemos normalmente. Exigir un número IMPAR de nodos. Y para que un nodo pueda actuar como maestro debe tener al menos la mitad de votos más 1 del resto de nodos.

---

# Qué cosas puedo pedirle a Kubernetes que quiero tener en mi entorno de producción.
# DESCRIBIENDO EL ENTORNO DE PRODUCCION

Toda configuración que hacemos contra un cluster de kubernetes va a ser mediante OBJETOS DE KUBERNETES.
Kubernetes define unos 30 objetos, que podemos configurar. 
Las distribuciones de kubernetes pueden traer más objetos.
Programas adicionales que puedo montar en un cluster de kubernetes también pueden traer sus propios objetos.
Esos objetos se definen en archivos YAML.

Cuales son los que kubernetes trae por defecto:
- Namespace
- Pod
- Deployment
- StatefulSet
- DaemonSet
- Job
- CronJob
- Secret
- ConfigMap
- PersistentVolume
- PersistentVolumeClaim
- Service
- Ingress
- NetworkPolicy
- ResourceQuota
- LimitRange
- HorizontalPodAutoscaler
- ServiceAccount
- PodDisruptionBudget
- ...
Nuestro principal objetivo en los próximos 2 días es aprender muchos de esos objetos. Qué son y para qué sirven.

La forma de cargar esos objetos en el cluster es mediante el comando kubectl:
 $ kubectl create -f fichero.yaml       Crea en el cluster los recursos (objetos) que se describen en el fichero.
 $ kubectl apply  -f fichero.yaml       Crea o modifica en el cluster los recursos (objetos) que se describen en el fichero.
    (Éste sería equivalente a un docker compose up)
 $ kubectl delete -f fichero.yaml       Borra del cluster los recursos (objetos) que se describen en el fichero.
    (Éste sería equivalente a un docker compose down)

El comando de kubernetes, además nos permite interactuar con los objetos que tenemos creados en el cluster... con una sintaxis muy similar a la del comando docker:
    $ kubectl <VERBO> <TIPO DE OBJETO> <args opcional>

    TIPOS DE OBJETO: Los de arriba
    VERBOS: Depende del objeto (igual que en docker)... aunque hay muchos comunes a todos los objetos:
        get, delete, describe
    
    Si quiero ver los pods que tengo en el cluster:
    $ kubectl get pods
    
    Si quiero ver los servicios que tengo en el cluster:
    $ kubectl get services

    Si quiero borrar un configmap:
    $ kubectl delete configmap mi-configmap

    Todos los objetos, puedo escribirlos en singular, plural o algunos con un alias:
        pod     pods
        service services    svc
        persistemvolumeclaim persistentvolumeclaims pvc
        presistentvolume persistentvolumes pv

    Si quiero ver todos los persistemvolumeclaims que tengo en el cluster:
    $ kubectl get persistentvolumeclaim
    $ kubectl get persistentvolumeclaims
    $ kubectl get pvc

Esa misma estructura / sintaxis la hereda el cliente de Openshift: oc

    Si en un cluster de Openshift quiero ver los persistentvolumeclaims:
    $ oc get persistentvolumeclaim
    $ oc get persistentvolumeclaims
    $ oc get pvc


Hay un argumento que siempre usamos con los comandos de Kubectl: -n NAMESPACE ( --namespace NAMESPACE)

    $ kubectl get pods -n wordpress
    $ kubectl get pods -n wordpress-desa
    $ kubectl get pods -n wordpress-prod
    $ kubectl delete pod mi-wordpress -n wordpress-prod
    $ kubectl apply -f fichero.yaml   -n wordpress-prod
    $ kubectl apply -f fichero.yaml   -n wordpress-desa

----

# Namespace (está en la librería básica de kubernetes, versión 1.0 - apiVersion:    v1)

Un namespace es una segmentación lógica (un entorno aislado) dentro de un cluster de kubernetes.
Dentro un namespace (asociado a un namespace) configuro objetos de kubernetes.
La mayor parte de los objetos de kubernetes (salvo contadas excepciones) deben ir dentro de un namespace.

El namespace me permite agrupar objetos de alguna forma relacionados:
- Todos los objetos que necesito configurar para el despliegue de una determinada app.
- Todos los objetos que necesito configurar para el despliegue de una determinada app para un cliente.
- Todos los objetos que necesito configurar para el despliegue de una determinada app para un tipo de entorno (desarrollo, producción, etc).

Por ejemplo, si quiero desplegar un Wordpress, podría crear un namespace llamado wordpress, dentro del que pongo:
- La BBDD
- El Apache
- .... y el resto de cosas que necesite para el wordpress

Incluso podría crear un namespace llamado: wordpress-desa, que tenga dentro la BBDD y el Apache de desarrollo.
Y otro namespace llamado: wordpress-prod, que tenga dentro la BBDD y el Apache de producción.

Los namespace también me ayudan con la seguridad.
Posteriormente veremos que puedo limitar el acceso a los objetos de un namespace a determinados usuarios.
- FELIPE solo puede toquetear los objetos del namespace wordpress-prod
- JUAN solo puede toquetear los objetos del namespace wordpress-desa

Adicionalmente vamos a poder limitar también el consumo global de recursos que pueden consumir los objetos creados dentro de un namespace. Por ejemplo, 
- En el namespace wordpress-desa, en total, entre todo lo que haya ahí dentro, no pueden hacer uso de más de 4 cores y 16Gbs de RAM.
- Mientras que en el namespace wordpress-prod, en total, entre todo lo que haya ahí dentro, no pueden hacer uso de más de 8 cores y 32Gbs de RAM.

Cómo configuro un Namespace en Kubernetes... Exactamente igual que cualquier otro objeto.
# Pod                                                       (apiVersion: v1)

Conjunto de contenedores que...
- Comparten configuración de red, y por ende ip. También pueden hablar entre si usando la palabra localhost.
- Escalan juntos (hacía arriba y hacia abajo).
- Se despliegan en el mismo host.
  -> Pueden compartir volúmenes de almacenamiento locales.

Cómo creamos objetos de estos? con un fichero de configuración en formato yaml.

## Asociación/Limitacion de recursos (RAM/CPU) a los pods

Realmente esa limitación se establece a nivel de contenedor:
```yaml
      resources:      # Limitaciones de recursos de cpu y ram para el contenedor
        requests:     # Lo que quiero que me sea garantizado
          cpu:        100m # Milicores... 500m = 0.5 cores... 1000m = 1 core
          memory:     512Mi
        limits:       # Lo máximo que se me permite llegar a usar
          cpu:        3 # el equivalente a 3 cores al 100% . NO SE ASIGNAN 3 CORES FISICAS.
          memory:     512Mi
```

Pregunta... a qué pensaís que va a afectar esto?
- A la estrategia de ubicación de un POD: Scheduler
    Además de la cantidad de CPU y RAM que tiene un nodo, el scheduler también tiene en cuenta:
    - El uso actual de recursos
    - Tintes y tolerancias
    - Afinidades y anti-afinidades
- Rendimiento cuando el programa esté corriendo.
- Consumo a nivel de host
                                          LO REAL        RESERVA       SIN RESERVAR     USO ACTUAL      SIN USAR
                                        CPU    RAM     CPU     RAM     CPU     RAM      CPU     RAM     CPU     RAM
                            NODO 1       8      20                      2       10       6      10      2        10
                                mariadb1                6      10                        6      10
                                
                            NODO 2       8      20                      2       10       1       4      7       16
                                mariadb2                6      10                        1       4

    Quiero desplegar un MariaBD... y a su contenedor le pongo:
        Request: 6 core, 10Gb

    Quiero desplegar otro MariaBD... igual.
        El scheduler solo tiene en cuenta lo reservado... no lo que se está usando a la hora de ubicar un POD.

    Quiero desplegar otro MariaBD... igual. El tercer MariaDB quedaría sin ubicar. PENDING!

    Quiero montar ahora un apache... y a su contenedor le pongo:
        Request: 2 core, 10Gb
        Limits:  3 core, 12Gb

    Si un contenedor está usando más CPU de la garantizada (siempre inferior al límite.. no se le deja más), si en un momento
    dado hace falta CPU para alguien que SI la tenga garantizada... le empezamos a encolar peticiones (KERNEL: LINUX es un SO de tiempo compartido).

    Si un contenedor está usando más RAM de la garantizada (siempre inferior al límite.. no se le deja más), si en un momento
    dado hace falta RAM para alguien que SI la tenga garantizada... Kubernetes se cruje el contenedor. UOPS !!!!!
        RECOMENDACION: En el caso de la ram siempre poner el límite igual a la reserva.
        De hecho esa estrategia la llevamos usando décadas en otro virtualizado que tenemos ppor ahí... la JVM.
        En la jvm hay dos parametros  -Xms y -Xmx y la recomendación de JAVA es poner esos valores iguales.
    Puede el SO quitarle RAM ya asignada a un programa? Ni de broma. No pasa nada... KILL -9... y es lo que hace Kubernetes!

    Kubernetes en cualquier caso EXIGE desactivar la SWAP.

# Deployment                                                (apiVersion: apps/v1)
# StatefulSet                                               (apiVersion: apps/v1)          
# DaemonSet                                                 (apiVersion: apps/v1)
# Job                                                       (apiVersion: batch/v1)
# CronJob                                                   (apiVersion: batch/v1)
# Secret                                                    (apiVersion: v1)                
# ConfigMap                                                 (apiVersion: v1)
# PersistentVolume (No va asociado a un namespace)          (apiVersion: v1)
# PersistentVolumeClaim                                     (apiVersion: v1)
# Service                                                   (apiVersion: v1)
# Ingress                                                   (apiVersion: networking.k8s.io/v1)
# NetworkPolicy                                             (apiVersion: networking.k8s.io/v1)
# ResourceQuota                                             (apiVersion: v1)
# LimitRange                                                (apiVersion: v1)            
# HorizontalPodAutoscaler                                   (apiVersion: autoscaling/v1)
# ServiceAccount (No va asociado a un namespace)            (apiVersion: v1)
# PodDisruptionBudget                                       (apiVersion: v1)

---

memory:     512Mi
            ^^^^^
Qué estoy poniendo ahí? Eso es medio giga? NO! Eso es 512 MEBIBYTES.

Kubernetes habla en Mebibytes... y en kibi, tebibytes, gibibytes, etc.
Igual que AMAZON.

1 GB = 1000 MB
1 MB = 1000 KB
1 KB = 1000 bytes

No estoy redondeando... Es así... No son 1024. Antiguamente si lo eran... pero eso cambió hace solo 25 años!
Hoy en día son 1000. Lo hicieron para estandardizar los prefijos con SI (Sistema Internacional de Unidades).

Se inventaron una nueva ud de medida el bibyte. que trabaja en base 2.
1 GiB = 1024 MiB
1 MiB = 1024 KiB
1 KiB = 1024 bytes
