
# RESUMEN

OLVM <- OVIRT --proyecto upstream-> RedHatVirtualization

Oracle ha tenido muchas herramientas de virtualización:
- Oracle Virtualization (OVM) - Se basaba en el hipervisor Xen y estaba orientado a servidores Oracle.
- Oracle Virtual Server (OVS) 

OLVM usa el hipervisor KVM/QEMU y opera sobre el kernel de linux.
Se monta sobre un Oracle Linux, que es una distro de GNU/Linux basada a su vez en la distro de RedHat Enterprise Linux (RHEL).

Su corazón (cerebro) es el Engine, que es una app web desarrollada en JAVA y que se instala/corre sobre un servidor de aplicaciones WILDFLY (proyecto upstream de JBOSS).

# CONCEPTOS/VOCABULARIO propios:

    Centro de datos (Data Center): Contenedor lógico superior de recursos físicos y lógicos.
        Clusters: Conjunto de hosts que comparten recursos y están bajo la misma política de seguridad.
            Hosts: Servidores físicos que ejecutan VMs.
                VDSM: Agente del host para el Engine:
                    Se comunica con libvirt y KVM/QEMU.
                    Crea y gestiona VMs, snapshots, discos, etc.
                    Configura redes
                    Actúa de SPM (Storage Pool Manager) si el host es elegido para ello.
                SPM: Storage Pool Manager. Rol de host que coordina metadatos y operaciones de storage.
            Storage Domains: Contenedores lógicos de almacenamiento (NFS, iSCSI, GlusterFS, etc.)
            Logical Networks: Expresan conectividad y pueden asociarse o no a un VLAN ID.

# VIRTUALIZACION CAPA COMPUTO

    ENGINE -> VSDM -> libvirt -> KVM/QEMU

# Intro a almacenamiento y a redes

    No voy a hablar ahora de esto, ya que es a lo que hoy vamos a dedicar la mañana!

---

# Nuestra instalación

Yo tengo un cluster de 5 máquinas. Estas máquinas tienen un cluster de kubernetes.
Son 5 mac-mins 2012 (16Gbs y i7 (8 cores)). Almacenamiento local: 2 ssd de 256/512Gb en espejo.
Además, tengo un DAS conectado al maestro (2 discos en espejo de 4Tbs) que se ofrece al resto por nfs... desde el primer nodo: maestro.

     Ubuntu 24

    +-Maestro
    |    NFS                        <- Almacenamiento (3-4Tbs)
    |       Tengo exportadas, entre otras, 2 carpetas de almacenamiento de OLVM
    |       Cada carpeta de almacenamiento de OLVM es un Storage Domain
    |         Esto nos permite jugar a mover discos entre Storage Domains y ver cómo se comporta OLVM.
    |         Realmente ambos están sobre los mismos disco. Esto no tiene sentido en un entorno de producción, pero nos sirve para aprender.
    +-Worker1
    +-Worker2                       <- Engine                   Oracle Linux
    +-Worker3-+                     <- Host1                    Oracle Linux
    +-Worker4-+                     <- Host2                    Oracle Linux
    |         |
    |         |
    LAN (192.168.2.0/24)

    Lo que pasa es que en el entorno, por facilidad de limpieza y aislamiento:
        Tanto el engine como los hosts son Máquinas virtuales gestionadas por KVM/QEMU.
    
    Las VMs que vamos a crear dentro de los HOSTS van con anidamiento en la virtualización, es decir, que dentro de las VMs de los hosts, vamos a crear otras VMs.
        Esta capado a nivel de los HOSTS el nivel de anidamiento de virtualización. En qué sentido: está limitada la generación de las CPUs.
        El modelo de CPU presentado dentro de las VMs esta capado a una generación más antigua -> Las VMs corren más despacio de lo normal, pero es suficiente para hacer pruebas y aprender.

        Worker 3 y Worker 4 llevan doble tarjeta de red: INTERNA + USB

        Las comunicaciones del VDSM con el ENGINE y las comunicaciones con ALMACENAMIENTO se hacen por la INTERNA.
        Las VMs que creemos dentro de los HOSTS van a usar la USB para conectarse a la LAN y a Internet.
            Está bien como ejemplo más real.
            En mi caso además ha sido para no joder mucho el otro interfaz (interno) que es el que usa también kubernetes... que monta su propia VLAN en esa NIC interna (CALICO)

    Toda la instalación la tengo vía ANSIBLE. Tengo 2 playbooks que son los que instalan TODO. -> ES UN REGISTRO COMPLETO DE LA INSTALACION

    El engine (app web) tiene un certificado autofirmado.
                                       ---> WIFI
                                       ---> RED PUBLICA DENTRO DE LA CASA 
    INTERNET A ---> ROUTER PROVEEDOR A ---> ROUTER INTERNO -> SWITCH (LAN) (Servidores, mis ordenadores de trabajo)
    INTERNET B ---> ROUTER PROVEEDOR B --->  NAT
                    NAT de la IP publica de cada proeedor
                        80             -->  80              -> Una VIPA que apunta a uno de los nodos de Kubernetes (FAILOVER)
                        443            -->  443

Dentro del cluster de kubernetes tengo mi PROXY REVERSO (NGINX-INGRESS CONTROLLER)     
        Dentro del proxy reverso he configurado :
            olvm.ivanosuna.com ----> IP en la LAN del host del engine -> WILDFLY (443)
            LETS ENCRIPT                                                 Certificado autogenerado por un CA interna mia.
            CERTIFICADO firmado por una CA PUBLICA DE CONFIANZA
                REENCRYPT
            olvm-console.ivanosuna.com ----> IP en la LAN del host del engine -> WILDFLY (5900)

Lo normal es que las consolas (terminales) a las VMs dentro de OLVM van por otro puerto (5900).
Yo lo he cambiado a nivel del proxy reverso para que vaya por el 443 y así no tener que abrir puertos en el router de la casa.


En Linux, una cosa es la NIC (tarjeta de red) y otra cosa es la INTERFAZ DE RED (eth0, eth1, enp0s3, etc). 
La interfaz es lo que usan los programas para conectarse a una red.
Esa interfaz estará asociada a 1 o varias NICs físicas:
    INTERFAZ1 -> NIC1
    INTERFAZ2 -> NIC1

    INTERFAZ3 -> NIC2 + NIC3 (bonding)



    lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP> 
enp1s0f0         UP             0c:4d:e9:9c:00:0b <BROADCAST,MULTICAST,UP,LOWER_UP> 
    NIC INTERNA
enxf8e43b59dd23  UP             f8:e4:3b:59:dd:23 <BROADCAST,MULTICAST,UP,LOWER_UP> 
    NIC USB

macvtap3@enp1s0f0 UP             52:54:00:b3:ed:e9 <BROADCAST,MULTICAST,ALLMULTI,PROMISC,UP,LOWER_UP> 
br-olvm          UP             7a:0e:47:a0:19:99 <BROADCAST,MULTICAST,UP,LOWER_UP> 
vnet0            UNKNOWN        fe:54:00:31:11:21 <BROADCAST,MULTICAST,UP,LOWER_UP> 


---

SELINUX:
Security-Enhanced Linux (SELinux)

El kernel de linux trae la posibilidad de aplicar politicas restrictivas de seguridad a nivel de kernel.
Antes de cualquier operación, el kernel puedo configurarlo para que hable con un programa externo, para que le autorice a realizar esa operación.

En RHEL se incluye un programa para esto ya preconfigurado en el kernel: SELINUX.
No es el único que hay.- En la familia Debian/Ubuntu hay otro llamado AppArmor.

SELINUX tiene 2 formas de trabajo (realmente 3):
- Desactivar
- Permisivo
- Enforcing

La operativa normal es instalar el entorno completo y lo pongo a funcionar: unas horas, semanas, dias...
Y lo coloco en modo permisivo. Esto hace que SELINUX no bloquee nada, pero si que registre en un log todas las operaciones que se solitan y se bloquearían.
Por defecto viene sin REGLAS DE SEGURIDAD PRECONFIGURADAS.
Una vez registradas todas esas operaciones, un humano las revisa, las aprueba (o rechaza) y pone el SELINUX en modo enforcing. A partir de ese momento, SELINUX va a bloquear todas las operaciones que no estén aprobadas.

---

# Gestión de energía:

OLVM quiere poder reiniciar los hosts de forma remota, para poder hacer mantenimiento, si no hay respuesta del host, etc.
Necesita tener habilitado en el host un mecanismo de gestión de energía remota. Esto se hace a través de IPMI (Intelligent Platform Management Interface) o Redfish (nuevo estándar), o protocolos propietarios de cada fabricante de servidores (Dell, HP, etc).

---

# Funciones de almacenamiento

Hay muchos tipos de almacenamiento (FUNCIONES QUE PUEDEN REALIZAR):
- Ficheros 
    Weblogic con una app web que gestiona expedientes de algo.
    A esos expedientes se les asocian adjuntos, que los usuarios cargan.
    Tendré 4 instancias de la app corriendo en 4 weblogics, corriendo en 4 vms distintas, corriendo en 4 hosts distintos.
    Y quiero que cuando uno de ellos suba un archivo, los otros 3 lo vean inmediatamente.
    Lo que interesa aquí es un volumen de almacenamiento compartido, que pueda ser accedido por varios hosts a la vez.
    Además, lo que voy a estar guardando es archivos... que posiblemente tenga organizados en carpetas.
    AQUI QUIERO UN ALMACENAMIENTO DE FICHEROS (NFS, CIFS, etc)
- Bloques (DISCOS)
    Tengo una BBDD o el propio SO de una VM... me interesa más un almacenamiento de bloques, que me permita crear discos virtuales y asociarlos a las VMs.
    Y la VM toma control del disco, lo formatea, le pone un sistema de ficheros, etc.
    Nadie más que ella puede acceder a ese disco virtual.
- Objetos (ISO)


    Disco Físico / LUN / Carpeta compartida                 VOLUMEN DE ALMACENAMIENTO FISICO
            |
    Protocolo de acceso (NFS, iSCSI, GlusterFS, etc)
            |
    Storage Domain (SD)                                     -> Contenedor lógico de almacenamiento (DEFINE TIPOS DE ALMACENAMIENTO: QUIERO VARIOS)
            |
    Virtual Disk (VD)                                       -> Disco virtual que se asocia a una VM
            |
    Lo monto en una VM -> File System (FS)                  -> Ficheros y carpetas dentro de la VM

Si hay un problema, a qé nivel afecta ese problema.

STORAGE DOMAIN:
Lo podemos ver como un tipo de almacenamiento que feino en el cluster de OLVM.
Lo puedo ver también como la colección de discos virtuales que hay en el cluster de OLVM.

Realmente, no sólo discos:
- Discos de VMs
- ISOs (imágenes de instalación de sistemas operativos)
- Snapshots (copias de seguridad de VMs)
- Templates (plantillas de VMs)

En la interfaz vemos una clasificación de FUNCIONES:
- DATA
- ISOs
- Export

Hoy en día, lo que creamos son DATA DOMAINS. Ahí puedo guardar de todo!
Los otros están deprecados.

# Tipos de almacenamiento

- NFS
    Ventaja: Sencillo de entender y de acceder a él para ver lo que tiene
    Problemas: export, permisos, propietario, latencia, servidor es un punto de fallo, etc.
- ISCSI
    Ventaja: Es almacenamiento de bloque y es un protocolo mucho más ligero que NFS.
    Entrega el volumen al SO de la VM y la VM lo monta como un disco físico y toma control de él.
         Iniciator (cliente)
         Target (servidor)
         IQN (identificador único de cada volumen)
         LUN (Logical Unit Number) (identificador único de cada volumen dentro del target)
- GlusterFS
- Fibre channel
    La diferencia con iSCSI es que el protocolo de transporte no es TCP/IP, sino que es un protocolo propietario de la fibra óptica.
    Conceptualmente es similar a iSCSI, pero más rápido y más caro.
- FS compatible POSIX
    Es almacenamiento local a nivel del host.
    Puede ser conveniente en algunos casos (máquinas vituales efímeras)
    Problemas/Limitaciones: No es compartido, no es tolerante a fallos, no hay posibilidad de live migration, ni scheduling, etc.

# Perfiles de disco

Son políticas de almacenamiento que se aplican a los discos virtuales de las VMs.
- Limitar el ancho de banda de lectura/escritura
- Limitar el número de IOPS (Input/Output Operations Per Second)
- Proteger el acceso a ciertos usuarios

---

Prácticas:

- Qué host está actuando de SPM?
- Qué Domain tiene el role master?
- Qué carpeta nfs se está usando en el primer storage domain (curso-nfs)
- Hay una máquina debian. Cuanto disco tiene? 3gbs
  Cuánto ocupa? 1gbs                                    thin provisioning: El disco puede crecer hasta 3gbs (y al SO se le presenta como tal), pero de momento sólo ocupa 1gbs.


---

2 GiB                   Gibibytes

    1 GB = 1000 MB
    1 MB = 1000 KB
        Antiguamente eran 1024. Hace casi 30 años que cambió!

1 GiB
    1GiB = 1024 MiB       Los GB y MB de toda la vida.


---

# Networking

Queremos tener VMs conectadas entre si, a una LAN y a Internet.

    Nuestras VMs ya tienen conexión ainternet, por qué? quien aporta eso?

                        192.168.1.0/24              
    INTERNET A ---> ROUTER PROVEEDOR A ---> ROUTER INTERNO -> SWITCH (LAN) <- VM1, VM2
    INTERNET B ---> ROUTER PROVEEDOR B --->                                    192.168.2.???


    Quién habilita que las máquinas tengan conexión a internet? EL ROUTER INTERNO EL PRIMERO

    El switch permite que las máquinas hablen entre si. Me da una red.
    El router es el que está enrutando esa RED 192.168.2.0/24 hacia otra red: 192.168.1.0/24
    El router externo (proveedores) a su vez enruta hacia internet. Por eso las máquinas tienen conexión a internet.

    No es algo que gestione desde OLVM.
    OJO!
    Otra cosa es que tuviera OVN.. y me defina ahí mis redes... y mis switches y mis routers virtuales. Ahí si que podría gestionar la red desde OLVM.
    SDN = Software Defined Networking. Esto es lo que hace OVN. Pero no lo vamos a ver hoy.

Aquí hay 2 partes a tener en cuenta:
- La infraestructura de "red física": Cables, Switches, routers, firewalls, etc.
- La configuración lógica de la red: IPs, VLANs, subredes, enrutamiento, etc.

Y luego otra adicional: Que esa información esté disponible dentro de OLVM, para que OLVM configure las VMs y sus interfaces de red de forma automática.



    HOST
        NICs    \
        NICs    / Bond - interfaz SO Anfitrion!     ESTO ME LO COMO YO!

        En nuestro caso, este host está virtualizado

    Por su parte, el agente del host (VDSM) creará en la máquina (en nuestro caso virtual) un "switch" virtual al que conectar las VMs.
    Esos switches virtuales son los que llamamos "LINUX BRIDGES" y son los que se ven en la interfaz de OLVM... más o menos.

    El switch lo monta OLVM. Pero en un switch yo puedo configurar un huevo de redes lógica (VLANS).
    Esas configuraciones de RED son las que hacemos y vemos en la interfaz de OLVM: Son los "Logical Networks".




        HOST 1

             VM1 VM2  VM3
              v   v    v     
            Bridge1 (br-vms) -> eth -> Bond NICs    ----+
                (Esto es un switch virtual)             |
                                                        +---- ROUTER EXTERNO --- INTERNET o a otras LANs
                                                        |
                                                        SWITCH EXTERNO
        HOST 2                                          |
             VM4 VM5  VM6                               |
              v   v    v                                |
            Bridge1 (br-vms) -> eth -> Bond NICs   -----+     
                (Esto es un switch virtual)


No todo switch admite la misma confiuración ni ofrece la misma funcionalidad...
No tiene que ver un switch TPLINK con un switch CISCO... ni el que monto con LINUX BRIDGES (este es bastante parquito en funcionalidad)

Hago una configuración (192.168.5.0/24; VLAN 17) a nivel de los switches virtuales del OLVM (2 linux bridges) y luego hago que las VMs se conecten a esos switches virtuales.
    Con esto , hasta aquí, las máquinas VM1, VM2, VM3 pueden hablar entre si, pero no pueden hablar con las máquinas VM4, VM5, VM6 a priori!
    Las máquinas VM4, VM5, VM6 pueden hablar entre si, pero no pueden hablar con las máquinas VM1, VM2, VM3 a priori!

SWITCH EXTERNO: Necesito configurarle una VLAN 17 y hacer que el trafico de esa VLAN 17 llegue a los 2 hosts.
Si lo configuró, entonces las máquinas VM1, VM2, VM3 pueden hablar entre si y con las máquinas VM4, VM5, VM6 a priori!

Tienen salida a internet?
A ese Switch le tengo conectado un router que vaya a una red en internet?
Lo tendré que conectar.
Y tendré que configurar el router para que enrute el tráfico de la VLAN 17 hacia internet.

OLVM por defecto trabaja así.
El gestiona la red interna!
Pero no gestiona la red externa. No gestiona el router ni el switch externo.

Otra cosa es que en OLVM monte un PROVEEDOR EXTERNO DE RED (External Network Provider) y que ese proveedor externo de red gestione la red externa.
Puede ser un proveedor que trabaje contra equipos hardware (cisco...) o puedo montar una herramienta que me permita definir redes virtuales (por software), con sus switches, sus routers... Esto es lo que hace OVN (Open Virtual Network > Open Virtual Switch).
Puedo montar ese OVN en un host dentro del cluster de OLVM y éste monta tuneles entre los hosts creando sus propias red lógicas lógicas / virtuales. 

En el mundo VMWare, las herramientas / Soluciones están más integradas/paquetizadas. 
Cómo llamamos a este tinglao en el mundo VMWare?
Se llama NSX (VMware NSX).

En VMWare pelao, puedo montar en los host esXI un Virtual Switch y defino portgroups y VLANs. PAra local.
Puedo hacer uso de un NSX Manager y montar redes virtuales entre hosts, con sus routers, firewalls, etc. Para eso necesito un NSX Manager y licencias de NSX.


El proveedor externo es un programa que permite a OLVM comunicarse con los programas que gestionan la red externa.
En nuestro caso podríamos optar por tener a OVN gestionando la red externa... y montaríamos el proveedor de OLVM para OVN.
Una cosa es montar el OVN y otra cosa es montar el proveedor de OLVM para OVN. Tendré que hacer las 2.

Y la instalación / configuración del OVN es cosa mía! El preparar OVN (INFRAESTRUCTURA DE RED) y luego preparar el proveedor de OLVM para OVN para que OLVM (vía OVN) vaya configurando redes lógicas en la infra virtual creada por OVN.

Si tuvieramos que hacer bond de interfaces en los hosts, ese trabajo lo hace network manager.
En las máquinas RHEL, el network manager es un demonio que corre en segundo plano y que se encarga de gestionar las interfaces de red y sus configuraciones.

El Agente de los hosts (VDSM) puede hablar con el network manager...
Esos bonds de los que hablábamos se configuran a nivel de SO (con el network manager), aunque ese trabajo podemos automatizarlo desde OLVM (vía VDSM).
Se hiría haciendo esa configuración a nivel de cada host. El engine ya habla con el VDSM y el VDSM habla con el network manager del host y le dice: "oye, monta un bond con estas 2 interfaces y asígnale esta IP y esta máscara de red y este gateway".

---

Para un buen desempeño de la red:
- Varias interfaces de red con salidas físicas diferentes a nivel de cada host para usos distintos:
  - VMs (al menos 1 interfaz de red dedicada a las VMs)
  - Migración en vivo (al menos 1 interfaz de red dedicada a la migración en vivo)
        Migrar una VM en vivo, puede implicar una cantidad de datos por red enormes. 
         - Discos (puede ser que estén en un almacenamiento compartido, pero puede ser que no)
         - RAM    (la memoria de la VM). En un servidor puedo tener fácil 32 Gbs, 64 Gbs, 128Gbs.
         - CPU    (el estado de la CPU de la VM) NADA
  - Almacenamiento (al menos 1 interfaz de red dedicada al almacenamiento)
    - En un almacenamiento iscsi o nfs... el consumo de red es enorme
  - Administración

Solo con eso, estaríamos hablando de 4 interfaces de red. Y cada una debería tener al menos 2 NICs.
Y no solo eso.. sino 4 lans diferentes. Si al final todas son gestionadas por el mismo switch, el ancho de banda se va a ver afectado. Lo ideal es que cada LAN tenga su propio switch físico. Habitualmente tenemos Switches de alta capacidad (10Gbs, 20Gbs, 40Gbs, etc). Muchas veces creamos VLANS con limitaciones (QoS)

Como buena práctica, aunque tenga pocas tarjetas de red (NICs) vamos a crear todas esas interfaces de red por separado. Y hacemos que todas operen por la misma NIC física (o bond). Esto me da flexibilidad al trabajar. Si el día de mañana meto tarjetas de red adicionales, no tendré problema.. ya tengo los huecos donde asignarlas (INTERFACES)

Esto va a ir muy condicionado por la INFRA.
- La infra de red
- El tipo de almacenamiento que tengamos (FIBER CHANNEL, ISCSI, NFS)

Un servidor Oracle T8 puede llevar hasta 16 PCIs. Y cada PCI puedo meterle 2 tarjetas de red.
Puedo montar una máquina con 32 interfaces de red físicas. Es un sin sentido, porque necesitaré PICs para otras cosas (almacenamiento, etc). Pero es un ejemplo de que puedo tener muchas interfaces de red físicas. Y puedo montar muchas LANs diferentes.

# BSD

Berkeley Software Distribution
La universidad de Berkeley en Calfornía creo un SO llamado 386BSD, basado en los estándares de UNIX.
cuando UNIX dejó de fabricarse y se convirtió en estándares, los de la universidad de berkley basado en los estándares montaron su propio SO "supuestamente compatible con POSIX y SUS". Lo llamaron 386BSD.
Se les ocurrió decir que tenía un SO UNIX... Llego AT&T y demanda al canto! Casí una década de litigios. Al final ganó la Universidad de Berkeley pero ya ni usábamos esa arquitectura de microprocesador.
Como el proyecto quedó congelado, otra gente intentop hacer algo parecido: GNU.
GNU no iba a usar Linux (ni existía)... pensaban montar su propio kernel. Y montaron de todo lo que hacía falta para un SO.
No valieron para hacer un kernel.
    GNU. GNU is Not Unix

El código de BSD se usó posteriormente para otros SOs:
- NetBSD
- FreeBSD
- OpenBSD
- MacOS (Apple) UNIX®
