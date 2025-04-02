# Docker / Kubernetes

## POD

Grupo de contenedores que:
- Comparten conf. de red (IP)
- Escalan juntos
- Se despliegan en el mismo nodo
- Pueden compartir volumenes locales

Los PODs tiene contenedores... pero tienen una peculiaridad: No pueden terminar su ejecución.
Caso que necesitemos hacer algún trabajo antes de que esos contenedores inicien: initContainers

Nosotros jugamos con Plantillas de pods:
- Deployment    Plantilla de pod + número inicial de replicas
- StatefulSet   Plantilla de pod + Plantilla(s) de petición de volumen + número inicial de replicas
- DaemonSet     Plantilla de pod de la que kubernetes monta un pod (una réplica/instancia) en cada nodo del cluster

En paralelo, si queremos ejecutar trabajos puntuales (que acaben) dentro del cluster hay otro tipo de objeto: JOB

Nosotros nos solemos crear Jobs... solemos dar una plantilla:
- CronJob = Plantilla de Job + CRON (programación)

## Volumenes

- Compartir datos (ficheros/carpetas) entre contenedores
    EmptyDir   
- Persistir la información tras la eliminación de un pod/contenedor
    Hay muchos tipos de volumenes... y son muy específicos (del backend que use para el almacenamiento: nfs, iscsi, fc...)
    Además, para cnfigurarlos necesitamos datos muy sensibles y técnicos (LUN, ID en AWS....)
        Datos de los que los desarrolladores adolecen (no los tienen):
            PV      Sistemas:   Aquí se define el detalle del volumen real + Información del volumen en el backend
            PVC     Desarrollo: Aquí se define el detalle del volumen solicitado (en lenguaje del dueño de la app)
        Kubernetes hace match entre PV y un PVC.
        En la práctica hablamos que los sysadmins de un cluster hacen otra cosa: Montar un Provisionador DINAMICO de volumenes
            que atiende a las peticiones de volumen (PVC) que se van haciendo.
- Inyectar ficheros / Carpetas al filesystem de un contenedor (configuración):
    ConfigMap / Secret   

## Comunicaciones

Para garantizar la HA / Escalabilidad, Kubernetes ofrece varios tipos de objetos:

### Service.

Hay 3 tipos de service:

- ClusterIP         Entrada en DNS de kubernetes + IP de balanceo (que balancea entre unos pods que ofrecen un SERVICIO)
- NodePort          ClusterIP + NAT de un puerto (> 30000) que apunta a la IP de balanceo en cada host del cluster
                        Esto exige tener adicionalmente OTRO BALANCEADOR DE CARGA (IP Externa al cluster)
- LoadBalancer      NodePort + Configuración automatizada del BALANCEADOR DE CARGA EXTERNO (debe ser compatible con Kubernetes)
                        Si instalo Openshift, tanzu, si contrato un cluster a un cloud... no problem!!!!
                        Pero si me monto yo un cluster on premise... Tengo que montar ese balanceador: MetalLB

En cualquier cluster (como en cualqueior entorno de producción) para mejorar la seguridad y 
facilitar las configuraciones, solemos montar un proxy inverso. 
Ese proxy inverso en kubernetes recibe el nombre de INGRESS CONTROLLER.
Cada regla que configuro en ese proxy reverso se llama INGRESS.

---

1. Instalar cluster de verdad de Kubernetes
2. Comenzar a instalar cosas allí
        Dashboard
        Programas (WordPress)
3. Openshift
4. Algunos comentarios adicionales
 
---

SELINUX / APPARMOR

SeLinux: Security Enhanced for Linux

En entornos Linux donde tenemos REQUISITOS FUERTES DE SEGURIDAD utilizamos apps tipo SeLinux
Eso es gracias a cómo está diseñado el KERNEL DE LINUX.
El Kernel de Linux permite instalar un programa al que antes de ejecutar cualquier operación, 
se le pregunte si esa operación está permitida.

Por defecto esos programas cuando los instalo prohiben todo.
SELIONUX funciona de una forma muy pecualiar (GENIAL).
Podemos tenerlo: 
- Deshabilitado (entonces pa' que????)
- Modo permisivo
- Modo forzado      <<< Es el que activamos en un entorno ya configurado... para que se apliquen reglas de seguridad ESTRICTAS

Cómo configuro las reglas? ESO ES UN FOLLON son cientos: Aquñi entra el modo permisivo.
PASO 1. Activo el modo permisivo
PASO 2. Empiezo a instalar y ejecutar cosas.
            SeLinux me permite ir haciendo esas cosas... 
            Pero en el journal de Linux (LOG) se van generando entradas de aquellas cosas que si tuviera el Selinux 
                en modo forzado NO SE PERMITIRIAN
PASO 3. Acabo de instalar mis aventuras... y pongo el SELINUX en modo forzado.
PASO 4. Utilizo esas entradas que SELINUX ha ido generando en el log para AUTOMATICAMENTE configurar todas las reglas que
            son necesarias para que el entorno funcione tal y como lo he ido creando.
            

---

kubectl (cliente de kubernetes) ----> apiserver
        v                        ??? Tu quien eres? AUTENTICACION
        v                        
Archivo configuración                                
    ~/.kube/config <<<< IP del apiserver de kubernetes
                        Usuario
                        Token de seguridad (contraseña)
                                
---

Para configurar cosas dentro de un cluster de kubernetes es necesario cargarle ficheros de manifiesto.
TODA LA COMUNICACION QUE HACEMOS CON KUBERNETES ES ASI!

Pero eso nognifica:
1. Que esos ficheros los vaya a crear YO
2. Que yo sea quién va a ejecutar el comando kubectl apply -f FICHERO


---

HELM! 

Pregunta... es fácil crear esos ficheros de manifiesto? NO... eso prácticamente SOLO lo puede hacer el creador de la app.

Y entonces, cómo instalo yo una app que haya hecho otro equipo de desarrollo ... que puede no ser ni de mi empresa.

Quiero instalar GITLAB!
Yo no he desarrollado gitlab. NPI del funcionamiento INTERNO de gitlab

Pero es más complejo. Porque no sólo se trata de instalar gitlab... Se trata de instalar gitlab adecuado / configurado para mi 
ENTORNO DE PRODUCCION!

Parte de la información la aporta el desarrollador... parte la aporta sistemas (de mi empresa).
Es más complicado aún. Muchos programas admiten formas de despliegue bastante avanzadas...

Por ejemplo. Podría instalar un Wordpress junto con una BBDD y una herramienta tipo REDIS para tener una cache de peticiones a BBDD
O no!
Pero si quiero tener cache...
1. Necesito adicionalmente montar un REDIS (con un determinado número de réplicas) + un balanceador (service) ... Sus volumenes...
2. Configurar el WP para que trabaje contra el REDIS.

Quién toma esta decisión? Los desarrolladores de Wordpress!

No puede existir un único fichero de manifiesto para un tipo de despliegue.
Ese fichero contendrá todo lo necesartio para MI INSTALACION... en base a MIS NECESIDADES.

Cómo funcionamos en la realidad:
- Los desarrolladores de un sistema, suelen entregar una plantilla de despliegue de app en kubernetes.
  Esa plantilla es ALTISIMAMENTE PARAMETRIZABLE
  Y recibe el nombre de CHART

    CHART: Plantilla de despliegue de app en kubernetes
    Nosotros lo que hacemos cuando queremos desplegar algo en un cluster es APORTAR configuración para el procesamiento de esa plantilla.
    Desde la plantilla + nuestra configuración -> se generan archivos de manifiesto (YAML)
    Eso es lo que hace helm
    
---

# Scheduler... Cosas que le afectan al tomar una decisión:

- Recursos: Request/Limit
- Uso real del hardware
- Afinidades/Antiafinidades a nivel de pod y afinidades a nivel de nodo.

# Afinidades

En cualquier pod, podemos configurar afinidades... de 3 tipos:

# Afinidad a nivel de nodo
    
    Nos permite que el scheduler prefiera montar un pod (o que por narices tenga que hacerlo) en nodos que:
    
    - Tengan una determinada etiqueta (label)... podemos jugar aquí (que no tengan un label... que tenga un label cuyo valor sea mayor que otro valor.)
    
    Tengo una app que necesita GPU para correr (IA, MachineLearning, Datamining)
    Puedo tener un cluster con 50 nodos... de los cuales sólo 10 tienen GPU.
    Les pongo una etiqueta (label). [A cualquier elemento de kubernetes le puedo poner un label]
        kubectl label node NOMBRE_NODO gpu=true
    En los pods que quiero que el scheduller ponga en esos nodos que tiene gpu, configuro una regla de afinidad a nivel de nodo.
        Con aquellos nodos que tengan esa etiqueta.
    
    También existe la posibilidad de asignar un pod a un nodo concreto... pero es raro... como se caiga el nodo me quedo sin pod.
    

# Afinidad a nivel de pod. ESTO SE USA MUY POCO !
    Aquí solo tenemos la sintaxis esa ENORME
    
    En este caso, lo que le indicamos a kubernetes (en concreto al scheduler) es que queremos que nos ponga el pod en un nodo
    que ya tenga un pod con una etiqueta... Estamos generando afinidad con PODs entre si.
        
    Ponme este pod en un nodo que ya tenga un pod de bbdd
    ```yaml
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - bbdd
        topologyKey: kubernetes.io/hostname
    ```
    
    Podría crear una etiqueta en los nodos llamada por ejemplo zona-de-disponibilidad
    Y tener 10 nodos con zona-de-disponibilidad=1
    Y otros 10 con zona-de-disponibilidad=2
    
    ```yaml
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - bbdd
        topologyKey: zona-de-disponibilidad
    ```
    
    En este caso, ya no estaría diciendo con esta regla a Kubernetes: Ponme este pod en un nodo que ya tenga un pod de bbdd
    Lo que le estaría diciendo es: Ponme este pod en cualquier nodo que comparta zona-de-disponibilidad con otro nodo que ya 
    tenga un pod con la etiqueta app:bbdd

    
    
# Antiafinidad a nivel de pod. MAS USADO DE TODOS... LO USAMOS SIEMPRE !

    Si estoy definiendo un pod con etiqueta app: nginx

    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx
        topologyKey: kubernetes.io/hostname

    Trata de evitar (preferred) poner este pod en un nodo (topologyKey) que ya tenga un pod (app:nginx) como éste.