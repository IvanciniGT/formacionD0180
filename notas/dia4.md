Una de las gracias de kubernetes (ya lo habíamos comentado) es que tiene en cuenta tanto a desarrolladores como a sistemas.

---

Volumenes persistentes:

- No existe referencia biunívoca entre pvc y pv en su definición.

Se crean totalmente independientes.

Un desarrollador habrá pedido un volumen.

Un administrador habrá creado un volumen (o 50).

Y kubernetes hace MATCH... es el TINDER de los volúmenes.

En realidad... 
- Los desarrolladores de apps crean PVCs...
- Pero los administradores de sistemas NO CREAN PVs.

La misión hoy en día (no siempre ha sido así) de un administrador de kubernetes es instalar un PROVISIONADOR DINAMICO DE VOLUMENES en el cluster.

Un provisionador dinámico de volumenes es un programa que está monitorizando todas las PVC que se crean en el cluster.
En cuantito se crea una PVC, el provisionador:
  1. Crea un volumen en el backend real en automático. (se conecta con la cabina/nfs/aws y crea un volumen)
  2. Crea un PV en kubernetes que apunta a ese volumen (lo registra en el cluster)
  3. Fuerza a kubernetes a que vincule la PVC con el PV que se acaba de crear

De forma que:
- El proceso queda automatizado
- El volumen se crea bajo demanda, ajustado a las necesidades de la aplicación


Kubernetes ofrece algunos provisionadores de volumenes dinámicos . Aunque ninguno viene instalado por defecto. Me toca instalarlo a mi.
Los clouds, cuando les contrato un cluster de kubernetes, me instalan provisionadores de volumenes dinámicos trabajando contra sus volumenes.

Tanzu (el kubernetes de vmware):
- Cuando se pide un volumen (PVC) en tanzu, ya viene instalado un provisionador de volumenes dinámicos que trabaja contra los volumenes de vmware.
Los provisionadores de volumenes dinámicos se asocian a un StorageClass.

De forma que en el cluster hay 2,3,10 storageclass. Cada storageclass tiene asociado un provisionador de volumenes dinámicos (o no... aunque en la realidad es un si)

---

# Cuántos pods vamos a crear en un cluster de kubernetes? 

NINGUNO! Nosotros no vamos a crear NINGUN POD en kubernetes.
Quién va a crear los pods? KUBERNETES....
Y yo le daré a kubernetes es una PLANTILLA de pod.

Dicho de otra forma. NO VAMOS A ESCRIBIR NINGÚN FICHERO que contenga un objeto de tipo POD. NI USO SOLO.

Lo que le diremos a kubernetes es:
- Quiero tener 5 pods basados en esta plantilla.
- Quiero tener 3 pods basados en esta otra plantilla.
- Quiero tener en el entorno de producción entre 3-10 pods basados en esta otra plantilla.

Y para ello, kubernetes tiene otros 3 tipos de objetos, que SI SON LOS QUE NOSOTROS VAMOS A DEFINIR:
- Deployment    Una plantilla de pod, junto con el número de replicas inicial que quiero tener.
- StatefulSet   Una plantilla de pod, junto con al menos 1 plantilla de pvc, junto con el número de replicas inicial que quiero tener
- DaemonSet     Una plantilla de pod, de la que en automático, kubernetes monta 1 replica en cada nodo del cluster. (esto es raro... más para cosas internas del cluster)

---

Wordpress


  Mariadb-galera-1
  Mariadb-galera-2      <<<         apache1: wordpress   <<<< BC     <<< Cliente
  Mariadb-galera-3                  apache2: wordpress


Pregunta:
- Los mariadb... comparten volumen de almacenamiento? 
- Cada mariadb guarda sus datos o todos tiran de un almacenamiento común?
   CADA MARIADB TIENE SU PROPIO ALMACENAMIENTO


  Mariadb-galera-1  DATO1 DATO2
  Mariadb-galera-2  DATO1 DATO3
  Mariadb-galera-3  DATO2 DATO3

  Con una máquina tengo un rendimiento de 1 datos /unidad de tiempo
  Con 3 máquinas tengo un rendimiento máximo teórico de 3 datos /2 unidad de tiempo
    Mejora potencial de rendimiento del 50% lo cual es decepcionante... Pero claro, la HA es cara

  Un cliente sube un pdf.... que guarda el apache1.
  Otro cliente entra al apache2, debe de ver el pdf? si
  Necesito tener el pdf replicado para ello? NO.
    No voy a permitir que 2 apaches toquen el mismo archivo... pero no hace falta en el uso normal de un wp.

Para los apaches, querría un almacenamiento común.          DEPLOYMENT
En cambio, para los mariadb no.                             STATEFULSET
Y eso viene determinado por la forma en la que trabaja cada uno de esos programas.

Al trabajar con un statefulset, para cada instancia, kubernetes crea una pvc, desde la plantilla suministrada.

---

# Comunicaciones en un cluster de kubernetes
                                                
          192.168.0.50:80 -> 192.168.0.102:30080
             |               192.168.0.101:30080
             |               192.168.0.12:30080
             |               192.168.0.11:30080
             |               192.168.0.10:30080
             |  
          BALANCEADOR           app.miempresa.com:192.168.0.50     chrome: http://app.miempresa.com 
             |                              |                        |
          192.168.0.50                     DNS                    MenchuPC
             |                              |                        |
  +----------+------------------------------+------------------------+-- red de mi empresa (192.168.0.0/16)
  |  Cluster
  += 192.168.0.10- Nodo Maestro 1
  ||                   |    LINUX
  ||                   |       netfilter. Os imagináis quien da las reglas a netfilter? kube-proxy 
  ||                   |          10.10.30.100:3307 -> 10.10.20.100:3306
  ||                   |          10.10.30.101:87   -> 10.10.20.101:80 | 10.10.20.102:80
  ||                   |          192.168.0.10:30080 -> proxy-inverso:87
  ||                   |          10.10.30.102:81 -> 10.10.20.103:80
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.6- apiserver
  ||                   +-10.10.10.2- controller-manager
  ||                   +-10.10.10.3- etcd
  ||                   +-10.10.10.4- core-dns
  ||                   |                  proxy-inverso:        10.10.30.102
  ||                   |                  wp-desarrollo:        10.10.30.101
  ||                   |                  basedatos-desarrollo: 10.10.30.100 # esta ip no cambia
  ||                   |                                                       no importa si los host
  ||                   |                                                       la tienen en cache
  ||                   +-10.10.10.5- kube-proxy
  += 192.168.0.11- Nodo Maestro 2
  ||                   |    LINUX
  ||                   |       netfilter
  ||                   |          10.10.30.100:3307 -> 10.10.20.100:3306
  ||                   |          10.10.30.101:87   -> 10.10.20.101:80 | 10.10.20.102:80
  ||                   |          192.168.0.11:30080 -> proxy-inverso:87
  ||                   |          10.10.30.102:81 -> 10.10.20.103:80
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.7- apiserver
  ||                   +-10.10.10.8- etcd
  ||                   +-10.10.10.9- core-dns
  ||                   |                  basedatos-desarrollo: 10.10.30.100
  ||                   |                  wp-desarrollo:        10.10.30.101
  ||                   |                  proxy-inverso:        10.10.30.102
  ||                   +-10.10.10.10- kube-proxy
  += 192.168.0.12- Nodo Maestro 3
  ||                   |    LINUX
  ||                   |       netfilter
  ||                   |          10.10.30.100:3307 -> 10.10.20.100:3306
  ||                   |          10.10.30.101:87   -> 10.10.20.101:80 | 10.10.20.102:80
  ||                   |          192.168.0.12:30080 -> proxy-inverso:87
  ||                   |          10.10.30.102:81 -> 10.10.20.103:80
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.11- etcd
  ||                   +-10.10.10.12- kube-proxy
  ||
  += 192.168.0.101- Nodo Trabajador 1
  ||                   |    LINUX
  ||                   |       netfilter
  ||                   |          10.10.30.100:3307 -> 10.10.20.100:3306
  ||                   |          10.10.30.101:87   -> 10.10.20.101:80 | 10.10.20.102:80
  ||                   |          192.168.0.101:30080 -> proxy-inverso:87
  ||                   |          10.10.30.102:81 -> 10.10.20.103:80
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.13- kube-proxy
  ||                   +-10.10.20.103- nginx: 80 < INGRESS CONTROLLER
  ||                   |                                                  v INGRESS
  ||                   |                  Si te llaman usando el nombre app.miempresa.com
  ||                   |                  lo mandas al servicio wp-desarrollo:87
  ||                   +-10.10.20.100- mariadb : 3306
  ||                   +-10.10.20.102- apache2 : 80
  ||                                      WORDPRESS_DB_HOST: basedatos-desarrollo:3307
  += 192.168.0.101- Nodo Trabajador 2
  ||                   |    LINUX
  ||                   |       netfilter
  ||                   |          10.10.30.100:3307 -> 10.10.20.100:3306
  ||                   |          10.10.30.101:87   -> 10.10.20.101:80 | 10.10.20.102:80
  ||                   |          192.168.0.102:30080 -> proxy-inverso:87
  ||                   |          10.10.30.102:81 -> 10.10.20.103:80
  ||                   |       kubelet -> Servicio
  ||                   |       kubeadm
  ||                   |       containerd
  ||                   +-10.10.10.14- kube-proxy
  ||                   +-10.10.20.101- apache1 : 80
                                        tengo que configurar la BBDD con la que trabajo
                                            WORDPRESS_DB_HOST: basedatos-desarrollo:3307
                                              Funcionaría eso?  SI
                                              Lo quiero?        NO
                                                Ni conozco a priori la IP del mariadb
                                                Y además esa IP puede cambiar

    Red virtual interna del cluster: 10.10.0.0/16

Este es mi cluster... Instalamos un WP?
- Para la BBDD. Vamos a usar mariadb. Cuántos pods al menos tengo que montar para tener HA? 1
  HA?   Tratar de garantizar un determinado tiempo de servicio (pactado previamente).
        Y para ello tomo medidas.
        Eso significa que si un sistema se cae, todo debe seguir funcionando?
        - Activo/Activo
        - Activo/Pasivo En Kubernetes, un activo/pasivo lo monto con solo 1 replica.
          Es decir, yo tengo solo 1 pod. Si se cae, kubernetes levanta otro, al instante! 
- Montemos un apache. 1

SERVICE CLUSTERIP: Un service es una entrada en el DNS interno de Kubernetes que apunta a una IP de balanceo,
que a su vez balancea sobre uno o varios pods.
  basedatos-desarrollo: 10.10.30.100:3307

SERVICE NODEPORT = SERVICE CLUSTERIP + NAT en cada host apuntando al puerto del service

SERVICE LOADBALANCER = SERVICE NODEPORT + configuración automática de un balanceador externo de carga compartible con kubernetes
    Cuanto contrato a un Cloud (AWS, Google CP, Azure) un cluster de kubernetes, me "regalan" (€€€€) un balanceador de carga
    externo, compatible con kubernetes.

  Si monto un kubernetes on premise, tengo que montar un balanceador de carga externo compatible con kubernetes YO MISMO.
  Y tenemos 1 opción: METALLB
  Metal lb se instala también dentro de kubernetes, como contenedores.

En kubernetes NO HAY NADA por defecto para configurar automáticamente un DNS Externo.
Openshift SI lo tiene... tenemos un oobjeto en OS llamado ROUTE ! Y éste configura automáticamente un DNS externo (compatible).
Lo que si hay en kubernetes es un addon que puedo instalar.


---
```yaml

kind:         Service
apiVersion:   v1
metadata:
  name:       basedatos-desarrollo # corresponde con el nombre de la entrada
                                   #  en el DNS interno de kubernetes

spec:
  type:           ClusterIP # por defecto. No hace falta ponerlo
  ports:
    - port:       3307        # Puerto en la IP de balanceo interna
      targetPort: 3306        # Puerto en el/los pod(s) entre los que hay que balancear
  selector:
    app:          mariadb     # Selecciona los pods que tienen el label app=mariadb
---
kind:         Service
apiVersion:   v1
metadata:
  name:       wp-desarrollo # corresponde con el nombre de la entrada
                                   #  en el DNS interno de kubernetes

spec:
  type:           LoadBalancer
  ports:
    - port:       87        # Puerto en la IP de balanceo interna
      targetPort: 80        # Puerto en el/los pod(s) entre los que hay que balancear
      nodePort:   30080
  selector:
    app:          wordpress
```


Por lo que os he contado, que % de servicios de cada tipo voy a tener en un cluster de Kubernetes ordinario:

                          %           Valor absoluto
  CLusterIP                               Todos menos 1
  NodePort                                0
  LoadBalancer                            1

Así es cualquier cluster de kubernetes.

Lo que voy a instalar es un PROXY REVERSO!
Y ese es el úinico servicio que expongo a pecho descubierto.
Cualquier otro servicio que quiera exponer al exterior, lo haré a través del proxy reverso.

En Kubernetes un PROXY REVERSO recibe el nombre de INGRESS CONTROLLER

Cuando monto un cluster de kubernetes on premise, tengo que instalar un INGRESS CONTROLLER YO MISMO. Hay decenas para elegir
NGINX, APACHE HTTPD, ENVOY, TRAEFFIC, HAPROXY

Cuando contrato un cluster a un cloud, o monto tanzu o el openshift de redhat, ya viene instalado un INGRESS CONTROLLER.

Oficial de kubernetes hay 1. Que tengo que instalar por separado. Se llama NGINX INGRESS CONTROLLER.

Cuando instalo (en kubernetes o no) un proxy reverso... luego tengo que configurarle reglas.
  Oye, proxy reverso, cuando recibas una peticion que viene a nombre del dominio: app.miempresa.com redirige al backend 10.10.30.101:87
  Esas reglas he de configurarlas en el proxy reverso.
  Pregunta. En apache, se configuran igual las reglas que en nginx? En el mismo formato? NO
            Y en haproxy? Tampoco
            Y en envoy? Tampoco
  Y entonces, en Kubernetes se han inventado un objeto nuevo, con su propia sintaxis.
  Y cuando instalamos uno de estos programas, además de venirme el proxy reverso, me viene un otro programa que es capaz de generar la configuración específica para ese proxy reverso (la regla en la sintaxis que ese proxy reverso entiende) a partir de esas intaxis unificada que kubernetes ha definido ... en el objeto INGRESS

Un ingress es una REGLA proxy reverso que puedo configurar en kubernetes... y que un controllador de ingress traducirá al lenguaje concreto que hable el proxy reverso que yo haya instalado.
Si el día de mañana cambio de propxy reverso, no tengo que cambiar las reglas... solo el controlador de ingress, y ese nuevo controlador, ya configurará las reglas para el nuevo proxy reverso, en su lenguaje específico.

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "app.empresa.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: wp-desarrollo
            port:
              number: 87

```

---

# Que era un pod?

Un conjunto de contenedores:
- Que comparten conf de red (IP)
- Que escalan juntos
- Que se despliegan el mismo host
- Que pueden compartir volumenes locales de almacenamiento

Nos falta un matiz aquí!

Sobre los contenedores... loca contenedores de un pod deben ejecutar servicios (procesos que no acaban nunca... que se quedan corriendo de perpetua)

Puedo en un contenedor ejecutar un comando/script? SI
Pero si ese contenedor lo tengo en un POD De kubernetes, NO DEBEREIA HACER ESO JAMAS EN LA VIDA!

Kubernetes espera que los contenedores de un pod se queden en ejecución. Si el proceso principal de un contenedor acaba (aunque sea con código de salida 0), KUBERNETES ENTRA EN PANICO Y REINICIA EL POD.... y no hay nada que pueda hacer para evitarlo.

En docker no pasa. Puedo tener contenedores que ejecuten una tarea y acaben... y ya.

Entonces... kubernetes no me permite ejecutar scripts en contenedores? SI
Pero no en los contenedores de un pod.

Algún ejemplo que se os ocurra de escenario donde esto nos haría falta?
- Backup de una base de datos
- Limpieza de logs
- Descarga de unos certificados
- ...

Cómo hacvemos eso en kubernetes.
Tenemos 2 opciones... orientadas a casos MUY DIFERENTES:
- Hay veces que cuando creamos un pod, queremos ejecutar un script.... por ejemplo:

      POD de mi web
        InitContainer: Descarga de los certificados -> VOLUMEN COMPARTIDO A NIVEL DEL POD
                                                             ^
        Contenedor 1: Apache --------------------------------+

      Los pods, además de permitir contenedores, permiten ejecutar  InitContainers... que son otro tipo de contenedores.
      Cuando en un pod/plantilla tenemos definidos initContainers, kubernetes ejecuta esos contenedores ANTES de ejecutar los contenedores normales. Si hay varios, los va ejecutando en orden... uno detrás de otro... hasta que no acaba uno, no empieza el siguiente. Una vez todos los initContainers han acabado, kubernetes ejecuta los contenedores normales... esos sí, en paralelo.

      Si el proceso de un initContainer no acaba, o acaba con un código de salida distinto de 0, kubernetes reinicia el pod.
- En otras ocasiones lo que necesito es ejecutar procesos batch/scripts en el cluster.
  - Backup
  - Notificaciones
  - Comprobación
  - Actualización
  - ...

  Para eso kubernetes tiene otro tipo de objeto: Job

  Un Job es como un Pod, pero donde se espera que los contenedores acaben su trabajo. Si no acaban, kubernetes se desquicia y reinicia el job.

  Lo normal es que creemos NINGUN JOB en un cluster de kubernetes.
  Los jobs queremos (en la mayor parte de los casos) que se ejecuten muchas veces... con una periodicidad en el tiempo. Lo que va definir no es un JOB... sino una PLANTILLA DE JOB!
  Igual que con los PODs nosotros no creamos ninguno... sino que definimos plantillas de PODs.
  
  Puede ocurrir un caso muy puntual que requiera ejecutar una tarea una única vez... RARO! pero puede ocurrir. En ese caso crearía directamente un JOB.
  Pero lo normal es lo otro.
  - Hazme un backup los jueves por la noche!

 Para eso, en kubernetes existe el concepto de CRONJOB = PLANTILLA DE JOB + CRON