
# Formación OLVM:

2 objetivos!
- Aprender realmente el OLVM
- Capacitarnos para el examen de certificación

# Dia 1: Fundamentos, Arquitectura de OLVM y Vocabulario.

- Qué es OLVM y qué parte de la virtualización realiza.
- Engine, VDSM, libvirt, QEMU/KVM
- Data center, Cluster, Host
- Storage domains, SPM
- NIC, bond, VLAN, bridge, logical networks, network profiles, vnics
- Herramientas

En paralelo iré haciendo analogías con el ecosistema de VMware, para que puedan relacionar conceptos y entender mejor la arquitectura de OLVM.

# OLVM

Plataforma de virtualización de código abierto que nos ofrece Oracle, basada en KVM y QEMU, que permite la gestión de máquinas virtuales y recursos de hardware de manera eficiente. OLVM se centra en la administración centralizada de entornos virtualizados, ofreciendo una interfaz web intuitiva y herramientas avanzadas para la monitorización y optimización del rendimiento.

# Virtualización

    App1  |   App2 + App3
    --------------------------
    SO1   |    SO2
    --------------------------
    VM1   |    VM2
    --------------------------
        Hipervisor
    --------------------------
        Sistema Operativo
    --------------------------
            Hierro

# Hipervisor

Hablamos de 2 tipos de hipervisores:
- Tipo 1: Se instala directamente sobre el hardware, sin necesidad de un sistema operativo subyacente.
          Básicamente tenemos Hipervisor + SO como u unidad
          ESXi, KVM 
- Tipo 2: Se instala sobre un sistema operativo existente, lo que significa que depende del sistema operativo anfitrión para funcionar.
          VirtualBox, VMware Workstation 

# KVM

Lo que vamos a tener por debajo de todo esto es LINUX.
Es lo que hay por debajo de ORacle Linux, que es una distro de GNU/Linux.

Lo que pasa es que KVM trabaja a nivel del kernel de Linux, convirtiendo el kernel en un hipervisor de tipo 1. Esto significa que KVM aprovecha las capacidades de virtualización del hardware y permite ejecutar múltiples máquinas virtuales de manera eficiente.

Pero... realmente KVM no es quien crea/gestiona/opera Máquinas virtuales. Es la labor la hace? QEMU

Trabajan en equipo, KVM se encarga de la virtualización a nivel de hardware y QEMU se encarga de propporcionar el proceso a nivel de SO para la ejecución de las VMS.

Si ya tengo una herramienta como KVM + QEMU, para qué necesito OLVM?
Una pregunta similar sería, si ya tengo ESXi, para qué necesito vCenter?

# Entornos de producción:

- Alta disponibilidad
  - "Tratar de garantizar" el acceso a los servicios / datos en caso de fallos de hardware o software.
   
- Escalabilidad
  - Capacidad de ajustar la infra a las necesidades cambiantes de la empresa, ya sea aumentando o disminuyendo recursos según la demanda.

- Recuperación ante desastres

Tanto HA / Escalabilidad la estrategia es REDUNDANCIA.
Tener répicas de:
- Proveedores de internet
- Proveedores de energía eléctrica
- Hardware
- Datos
- Programas / Configuraciones

Los hipervisores: KVM, ESXi trabajan a nivel de un hierro/host.
Y si se jode el host... estamos jodidos. 

Necesitamos herramientas adicionales que nos ayuden a orquestar las VMs, almacenamiento y redes, para poder tener HA y Escalabilidad, en caso de que un host se joda. Y eso es lo que hace OLVM o el vCenter de VMware.


    OLVM                -> Host 1
      Ovirt-Engine          VDSM -> libvirt -> KVM/QEMU
                        -> Host 2
                            VDSM -> libvirt -> KVM/QEMU
                        -> Host 3
                            VDSM -> libvirt -> KVM/QEMU


Hay más herramientas aquí en medio:

KVM         . Proporciona virtualización a nivel de hardware en el kernel de Linux.
QEMU        . Emula hardware y proporciona virtualización a nivel de software. Quien ejecuta las VMs.
LIBVIRT     . Programa que opera/gestiona las VMs (QEMU/KVM ) que tenemos en un host, ofreciendo un API (comunicación) para que otras herramientas puedan interactuar con las VMs.

Cuando trabajamos en el mundo VMWare, tenemos una serie de conceptos/utilidades muy integradas entre sí, que nos permiten gestionar y orquestar la virtualización de manera eficiente. 

En el mundo Linux, las cosas son similares, pero los conceptos y utilidades no están tan integrados. Realmente lo están.. pero se tratan como proyectos independiente. Hay muchos paquetes/herramientas, que unas se apoyan en otras, pero no están integradas/ensambladas como en VMware. Por eso es importante conocer el ecosistema de Linux y cómo se relacionan entre sí.

En un host tendremos 1 proceso de libvirt, que se encarga de gestionar tooodas las VMs que tengamos en ese host.
Por cada VM que creemos, libvirt creará un proceso de QEMU/KVM, que será el que ejecute la VM.

Todo esto son herramientas / proyectos Opensource que puedo instalar/hacer uso de ellos en el mundo Linux.

OLVM es un proyecto Opensource, pero esto ya es de Oracle.
OLVM tiene muchos componentes. Tampoco es solo un programa. Son muchos programas:
- Engine: Se instala a nivel de un servidor y es el gran orquestador de todo el ecosistema. Es el que se encarga de gestionar los hosts, las VMs, el almacenamiento y las redes. Es el que nos ofrece la interfaz web y la API para interactuar con el sistema.
  Está desarrollado en JAVA y corre sobre un servidore de aplicaciones: Wildfly
- VDSM: Agente que se instala en cada host y es el que se encarga de la comunicación entre el Engine y el host (libvirt y otras cosas...). Es el que ejecuta las órdenes que le envía el Engine, como crear, eliminar o migrar VMs.

# Wildfly (JBOSS)

WildFly es un proyecto Opensource y gratuito.
Quién está detrás de WildFly? RedHat

Y hay varias cositas que nos salen por aquí con el appelido RedHat.

Redhat es una empresa, que crea muchos productos de software, todos ellos Opensource... Algunos gratuitos, otros bajo un modelo de subscripción.

Tiene de todo proyecto/producto 2 variantes:
- La versión de pago
- La versión gratuita

La versión de pago suele ser una versión "VIEJA" de la versión gratuita, pero con soporte y actualizaciones garantizadas por RedHat. Y garantía de que tiene "POCOS" bugs.

    Sistema operativo:          RHEL                         <-- Upstream -- FEDORA
    Servidor de aplicaciones:   JBoss EAP                    <-- Upstream -- WildFly
    Ansible:                    Ansible Automation Platform  <-- Upstream -- AWX
    Virtualización:             RedHat Virtualization        <-- Upstream -- Ovirt

Oracle Linux es una distro de GNU/Linux, basada en RHEL, que Oracle ofrece de manera gratuita.
Qué significa eso de que está basada en RHEL?
La gente de Redhat por la licencia de Linux y de GNU (GPL) tiene obligación de proporcionar todo el código fuente de sus productos Opensource.
Lo suele publicar 2 veces al año (RHEL)

Lo que hace Oracle es coger ese código fuente, le cambia los logos, le cambia 4 programas adicionales por los que Redhat cobra (canal de actualizaciones, soporte, etc) por herramienta suyas y lo publica como Oracle Linux.

Esto... pasa no solo con Oracle Linux.
Muchas utilidades/productos de Oracle, están a su vez basadas en productos Opensource de RedHat, como por ejemplo:

# OVIRT

OVirt es un proyecto Opensource, patrocinado por RedHat para crear una plataforma de virtualización basada en KVM y QEMU, que permita la gestión centralizada de entornos virtualizados (ha, escalabilidad, monitorización, etc).

Redhat tenía su propia herramienta de virtualización, basada en Ovirt... Dejaron de hacerla.
Pero Oracle cogió el proyecto Ovirt y lo adaptó a sus necesidades, creando OLVM.

Veremos muchas herramientas, paquetes que comienzan por el nombre ovirt. Son herramientas que forman parte del proyecto Ovirt y que Oracle ha adaptado a su plataforma OLVM.

    ovirt-engine: Es el Engine de Ovirt, que Oracle ha adaptado a su plataforma OLVM.

# NOTA:

Pero... Oracle no tiene su propio Servidor de Aplicaciones Web JAVA? Si.. WebLogic.
Pero :
- Weblogic es un proyecto muerto! Está en fase legacy!
- El engine -> ovirt-engine, era un proyecto originalmente de REDHAT... y el servidor de aplicaciones de REDHAT gratuito es el Wildfly. Por eso Oracle ha decidido usar Wildfly para el Engine de OLVM.

Wildfly es un servidor de aplicaciones de clase EMPRESARIAL... no es un tomcat, aunque es gratuito!

# Qué más nos ofrece el OLVM:

- Inventario común: imágenes, plantillas, vms
- Seguridad: usuarios, roles, permisos
  - El OLVM tiene su propio "gestor de usuarios"... aunque podemos darle el cambiazo por un proveedor externo. De hecho esto es algo recomendado!
  - La herramienta recomendada aquí es KEYCLOAK, que es un IAM (opensource) de RedHat, que nos permite gestionar usuarios y grupos de usuarios, y que se integra con el OLVM.
- Utilidades para migrar VMs entre hosts, sin que se note que la VM se ha movido de un host a otro.
- Políticas de cluster para HA, escalabilidad, etc.
- Templates, Pools, Snapshots, Clones, etc.
- Eventos y observabilidad
    -> Integrado con Prometheus y Grafana
- Backups y recuperación ante desastres
- Gestión de redes y almacenamiento

# Notas adicionales sobre Virtualización y Oracle.

> Oracle ha tenido distintas tecnologías para virtualización.

- OLVM: Basada en KVM/QEMU y Ovirt. Es la que estamos viendo ahora.
- Oracle Virtualization: Basada en el hipervisor XEN... Deprecado!
- Oracle VM Server: Para los procesadores Spark y operando principalmente con SO Solaris.

Oracle es el propietario de JAVA.
Pero el proyecto inicialmente era de otra empresa llamada Sun Microsystems. Sun Microsystems fue adquirida por Oracle en 2010.
A Oracle le importaba el JAVA nada y menos.
A Oracle lo que le interesaba era la INFRA / HARDWARE de Sun. Sun era fabricante de hardware... Servidores BRUTALES!

> OLVM está basado en ovirt

Era un proyecto auspiciado por RedHat, que Oracle ha adaptado a sus necesidades.
La documentación de ovirt está mucho más desarrollada que la de OLVM. Me puede interesar buscar información de ovirt en algunos casos (troubleshooting, problemas, etc) y adaptarla a OLVM.

# El engine de OLVM (Proyecto de OVIRT)

Es el plano de control del OLVM. App JAVA que corre en Wildfly (usa una BBDD PostgreSQL) y que se encarga de gestionar los hosts, las VMs, el almacenamiento y las redes. Es el que nos ofrece la interfaz web y la API para interactuar con el sistema.

Viene a ser el equivalente en el mundo VMware del vCenter.

Ninguno de ellos son hipervidores. Son herramientas de orquestación y gestión de la virtualización.

> Qué ocurre cuando arranco una VM

1. Se lanza petición al Engine de OLVM (UI, API)
2. El engige valida (usuarios, permisos, estado, recursos, redes, almacenamiento...)
3. Selecciona un host compatible para ejecutar la VM. Aquí podremos fijar la VM a un host, definir políticas de afinitidad, etc.
4. Se procede a comunicar al VDSM de ese host que ejecute la VM.
5. VDSM prepara recursos (disco, red, memoria, etc) y llama a libvirt para que ejecute la VM.
6. Libvirt crea un proceso de QEMU/KVM para ejecutar la VM.
7. La VM usa los recursos, la red, el almacenamiento.. a nivel del host.
8. VDSM se encarga de ir informando al Engine del estado de la VM, para que podamos monitorizarla desde la UI o la API.


    USUARIO/AUTOMATIZACION(API)
                V
            Engine (autoriza, valida, coordina)
                V
            VDSM (prepara recursos, monitoriza, informa)
                V                                V
                Almacenamiento, Red             Libvirt (crea proceso QEMU/KVM)
                                                    v
                                                    VM

Al final, en la UI del engine, tenemos un botón. Pero hay mucho detrás.

> Qué pasa si se cae el engine?

Las VMs siguen operando con normalidad.
Los procesos de las VMs no corren dentro del engine. Corren dentro de los hosts, a nivel de KVM/QEMU.

Qué pierdo?
- Administración centralizadas de las VMs
- Ciertas operaciones: Migración de VMs
- Visibilidad a nivel central
- ...

La caída del engine no apaga la máquina, lo que tenemos es que la plataforma pierde el plano de CONTROL.

En VMWare sería igual. Si se cae el vCenter, las VMs siguen operando con normalidad. Lo que perdemos es la administración centralizada de las VMs.
El ESXi sigue operando con normalidad, pero no podemos hacer ciertas operaciones, como migrar VMs entre hosts, etc.

Más adelante, que hablemos de la instalación, veremos que hay 2 formas de instalar el engine:
- En un host independiente.
- Lo podemos montar como una VM dentro del propio ecosistema.

# Los datos... dónde se guarda.

Ya hemos dicho que en postgreSQL. Con 2 finalidades:

`engine`              -> Inventario, configuración, estado de las VMs, hosts, redes, almacenamiento, etc.
    Esto es obligatorio
`ovirt-engine-history`      -> Historial de cambios, eventos y métricas de las VMs, hosts, redes, almacenamiento, etc.  
    Si no instalo esto, pierdo ciertas utilidades en la UI del engine, como por ejemplo la monitorización de métricas de las VMs, hosts, redes, almacenamiento, etc.

El control del estado actual y de las configuraciones son responsabilidad del `ovirt-engine`. 
El historial de cambios y eventos es responsabilidad del `ovirt-engine-history`.

---

# Datacenter

En una entidad LOGICA (no representa nada físico: Un host o cosas así) que agrupa recursos FISICOS y LOGICOS!
Dentro de un datacenter, tenemos: Clusters, HOSTS, Storage Domains, Logical networks, etc.

Cuidado con pensar que DataCenter tiene SOLO que ver con ALMACENAMIENTO (por tener la palabra DATA)

# Cluster

Agrupa hosts con KVM/QEMU y VDSM, donde crear mis VMs.
Habitualmente en un cluster agrupamos hosts con características similares, para poder crear VMs y que estas puedan migrar entre hosts sin problemas.
- Misma arquitectura de CPU
- Programación de vms, migración, balanceo de carga, HA, etc.
  
OLVM exige arquitectura de CPU compatible dentro de un cluster.

# Host

Máquina Oracle Linux con KVM/QEMU, libvirt y VDSM, que forma parte de un cluster y donde se ejecutan las VMs.
Opcionalmente puede llevar CEPH, openvswitch, etc.

# VM

Es administrador por el engine, pero se ejecuta a nivel de un host, dentro de un proceso de QEMU/KVM.
Esta vinculado a un cluster y a un host (cuando se está ejecutando). Puede migrar entre hosts del mismo cluster.
Una VM va a estar relacionada también:
- Con un Storage Domain, donde se almacenan sus discos virtuales.
- Con vNICs, que son las tarjetas de red virtuales que se le asignan a la VM y que están vinculadas a Logical Networks.

Dentro del Data center, encontramos:

    OLVM Engine:
        - DATACENTER A
          - Storage Domain 1
          - Storage Domain 2
          - Logical Network 1
          - Logical Network 2
          - Cluster A
            - Host 1
            - Host 2
          - Cluster B
            - Host 3
            - Host 4
        - DATACENTER B

# SPM (Storage Pool Manager) 

Realmente es un rol que toma (el engine se lo asigna) uno de los hosts de DataCenter.
Qué hace?
- Coordina las operaciones de almacenamiento dentro del DataCenter.
- Se encarga de crear y manipular discos virtuales, plantillas y snapshots en los Storage Domains.
- También de las asignaciones de almacenamiento de bloques y de la gestión de los Storage Domains.

NO ES EL CAMINO HABITUAL DE I/O

    CONTROL: Engine -> host SPM -> Metadatos
                                    vvv  
                DATOS:   VM -> Host -> almacenamiento

No hay un equivalente directo en el mundo VMWare.

El host que tenga vinculado el SPM puede en paralelo ejecutar VMs.
Con las mismas, si el host deja de estar disponible, el engine asignará el rol de SPM a otro host disponible dentro del DataCenter.

# Migración de VMs de un HOST a OTRO

Necesitamos para hacer una migración:
- MISMO CLUSTER
- Necesito que ambos estén operativo
- Capacidad en destino
- Acceso a redes y discos desde destino

---

# Almacenamiento

        Almacenamiento físico
                   ^
        Protocolo o forma de acceso
                   ^
        Storage Domain (OLVM)
                   ^
        Discos virtuales (OLVM)
                   ^
        VM: Particiones, filesystem  

# Almacenamiento físico

Podemos usar:
- Cabina
- Un servidor NFS
- Conectar con un almacenamiento GlusterFS o FibreChannel
- Discos locales de los hosts

# Storage Domain

Es una abstracción (concepto lógico) que nos ofrece OLVM para gestionar el almacenamiento físico.
En el mundo VMWare el equivalente es un Datastore.

- Las imágenes se van a guardar como ficheros.
- Discos virtuales de las VMs se van a guardar como de bloques.

OLVM usa LUN

# Esas LUNS -> Virtual Disks (Discos virtuales de las VMs)

- YA es el SO de la VM el que se encarga de crear particiones, formatear, crear filesystem, etc.
- LVM

Podemos por ejemplo ampliar un disco dentro del OLVM
    10Gbs -> 20Gbs

Pero eso no cambia ni las particiones ni el filesystem del que se tiene acceso en la VM. Para eso hay que entrar en la VM y hacer los cambios a nivel de particiones y filesystem.

Hay 2 formas en las que trabajar:
- Local Storage:   Los hosts usan privativamente su almacenamiento local para los discos virtuales.
- Shared Storage:  Los hosts usan un almacenamiento compartido para los discos virtuales, que no puede estar por definición asociado privativamente a un host concreto. 

Las 2 opciones están soportadas en OLVM. 
Shared me da más funcionalidad y facilidad para ciertas funcionalidades (por ejemplo live migración)
Si estoy con un almacenamiento local, SI PUEDO MIGRAR VMs de un host a otro.. pero no en modo live.


    HOST 1
    HOST 2
    HOST 3

Quiero tener un cluster Activo-Activo de MySQL/MariaDB: GALERA!
Va a tener 3 nodos.
Es decir, tendremos 3 procesos de SO de tipo mariadb, corriendo en 3 vms distintas, en 3 hosts distintos.
Qué hacemos con el almacenamiento? Me interesa shared o local?
Me podría venir bien aquí un almacenamiento local? 
Un almacenamiento shared también ofrecería HA.

Al final... cada VM necesita su propio HDD.
MariaDB/MySQL si tengo 7 nodos, cada nodo tiene una copia de los datos (quizás no de todos)

    MariaDB1 - dato1    dato2 -> Su Volumen de almacenamiento1
    MariaDB2 - dato1    dato3 -> Su Volumen de almacenamiento2
    MariaDB3 - dato2    dato3 -> Su Volumen de almacenamiento3

INSERT dato1

Esto se guardaría en uno, en 2 o en los 3? En 2

Un cluster Activo-Activo en BBDD es para REPARTIR CARGA/ Mejorar rendimiento (Ingesta y consulta) -> ESCALABILIDAD
Para HA, montaría un cluster Activo/Activo? No necesariamente, podría ser Activo-Pasivo.

En una máquina puedo hacer 1 operación de escritura por unidad la tiempo.
En 3 máquinas puedo hacer 3 operaciones de escritura por 2 unidades de tiempo.

Es decir, la mejora máxima teórica que puedo tener en un cluster Activo-Activo de BBDD es de un triste 50%... que al final con las comunicacions y orquestación interna mno pasa del 30% de mejora. FRUSTRANTE:
    x3 en la infra: 300%
    x1.5 en el rendimiento. Mejora del 50%.


Esos Volúmenes de almacenamiento pueden estar en un almacenamiento local o en un almacenamiento compartido.

Compartido no significa que las 3instancias de BBDD (los 3 procesos, VMs) estén usando el mismo volumen de almacenamiento.
Significa que el almacenamiento es accesible por red.. desde los 3 hosts.
Por ejemplo, en una cabina de almacenamiento.

    Host1                           Cabina
        VM1 Mariadd              <-     Lun1
    Host2                               
        VM2 Mariadd              <-     Lun2
    Host3
        VM3 Mariadd              <-     Lun3

Eso me permitiría que si una VM del mariaDB se jode... o si un Host se jode... pueda mover esa VM a otro host.

    Host1  --> OFFLINE                 Cabina
    Host2                               
        VM1 Mariadd              <-     Lun1
        VM2 Mariadd              <-     Lun2
    Host3
        VM3 Mariadd              <-     Lun3
    

Si por contra trabajo con almacenamiento local:

    Host1                           
        VM1 Mariadd    -> HDD interno del host1
    Host2                               
        VM2 Mariadd    -> HDD interno del host2
    Host3
        VM3 Mariadd    -> HDD interno del host3

Si se cae el host 1, no hay manera de levantar la VM1 en otro host.

Pero en este caso, la HA la ofrece el MariaDB (con el cluster Galera) y no quiero o necesito que la ofrezca el OLVM.
Si opto por esto, posiblemente me interesa más un almacenamiento local:
- Más rápido (dependiendo del almacenamiento compartido)
- No saturo la red con tráfico de almacenamiento

Estas decisiones DEPENDEN DEL SOFTWARE CONCRETO QUE MONTO ARRIBA.
Necesito entender QUE PROGRAMAS VOY A PONER A FUNCIONAR Y COMO FUNCIONAN ESOS PROGRAMAS PARA PODER TOMAR BUENAS DECISIONES.
Y si mi programa me gestiona la HA y la escalabilidad, no necesito que el OLVM me lo haga.
DECISIONES!
Y puede ser que me interese vincular las VMs a HOSTS concretos.. sin migración, sin posibilidad de planificación por el OLVM... Ya lo planifico yo de antemano:
    VM MariaDB1 -> HOST1
    VM MariaDB2 -> HOST2
    VM MariaDB3 -> HOST3


Esto no va a afectar solo a ALMACENAMIENTO, también a REDES.
En las redes me puede pasar igual.
A veces asocio una VM a un host concreto... y hago uso de las funciones de virtualizacion nativas de las NICs. Toda tarjeta de red ofrece virtualización a nivel de hardware. Las exponen mediante SR-IOV. 
Y puedo hacer que la misma tarjeta de red física se exponga como varias tarjetas de red virtuales, o almenos algunas de sus funciones, que pueden ser usadas por distintas VMs directamente... sin necesidad de OLVM, Configuraciones a nivel de kernel de Linux, OpenVSwitch.
Pero eso solo lo puedo hacer si la VM está asociada a un host concreto.
Esto me va a dar mejor rendimiento, pero pierdo H a nivel de OLVM. 
Si tengo hosts con varias tarjetas de red, y les configuro bonds... quizás me interese.
En el mundo de las BBDD es especialmente interesante, para minimizar la latencia de las comunicaciones a la BBDD.

Y Aquí si puede haber desparrame... puedo tener en el mismo cluster un host con x tarjetas físicas de red y otro con otras.

# Redes:

Los HOSTS tienen tarjetas físicas de red NICs.
Y van a estar conectados entre si por una red Puede ser una subnet, una vlan, etc.
Pero eso es para comunicación entre ellos.

Nosotros vamos a crear VM, que querremos comunicar entre si, y comunicar con el exterior.

Aquí entran / se utilizan conceptos propios/ utilidades propias del kernel de Linux.


    NIC Física < Bond

## Bond de NICs?
Lo que hacemos es agrupar tarjetas de red... y configurarlas:
- Failover: Si una NIC se cae, que el tráfico salga por la otra NIC.
- Balanceo de carga: Que el tráfico se reparta entre las NICs.
      No es que si quiero mandar mediante una conexión 200Gbs, que los 200Gbs se repartan entre las NICs...
      Es que mientras tenga consumido el ancho de banda de una NIC, hay otra que puede estar disponible para mandar/recibir tráfico de otra conexión.
        ESTE ESCENARIO NO ES ALGO QUE GESTIONE EXCLUSIVAMENTE EL KERNEL DE LINUX. Implica configuración adicional en el switch de red, para que el tráfico se reparta entre las NICs.

Dentro del host, tendré VMs. Y esas máquinas se deben conectar a RED.
Para ello, necesitan una tarjeta de red. 
Aquí hay 2 opciones:
- Tarjeta de red virtual (vNIC).
- Darle acceso directo a la tarjeta de red física (SR-IOV).

La segunda ofrece mejor rendimiento... pero a costa de perder flexibilidad. Lo que usamos normalmente (90%) es la primera opción. 

Esas VNICs son gestionadas por OLVM.
A dónde se conectan esas VNICs? 
    A una Logical network, que es un concepto lógico que nos ofrece OLVM para gestionar las redes de las VMs.
    OJO, QUE NO ES LO MISMO QUE UNA RED VIRTUAL!

---

     VM1                                    VM2
        VNIC1                              VNIC2   
         |                                  |
         |                                  |           Puedo hacer un Linux Bridge
        NIC1  >---------SWITCH--------<    NIC2
    HOST 1                               HOST 2

                Entre esas máquinas podemos (gestionado por el switch) puedo tener una LAN, una VLAN, etc.

# Linux Bridge

Es una utilidad que encontramos en el kernel de Linux, que nos permite crear un switch virtual dentro del host, al que podemos conectar las VNICs de las VMs y las NICs físicas del host.

    VM1                                    VM2
        VNIC1                              VNIC2   
         |                                  |
         |                                  |           Puedo hacer un Linux Bridge
        NIC1  >---------SWITCH--------<    NIC2
    HOST 1                               HOST 2

Los bridge dentro de un host pueden ser de distintos tipo:
- Puedo tener un bridge que conecte las VNICs de las VMs entre si, pero no con la NIC física del host. Es decir, que las VMs puedan comunicarse entre si, pero no con el exterior.


            HOST 1

                VM1 - VNIC1  --- Bridge (switch interno) 
                VM2 - VNIC2  ---

- Pero... en la mayor parte de los casos, lo que queremos es que las VMs puedan comunicarse entre si incluso estando en distintos hosts, y con el exterior. Para ello, necesitamos que las VNICs de las VMs se conecten a un bridge que a su vez esté conectado a la NIC física del host ( o a un bond).

            HOST 1

                VM1 - VNIC1  --- Bridge (switch interno) --- NIC1 (física)
                VM2 - VNIC2  ---

Pero claro... cómo sale el tráfico vía la NIC del host? Varias opciones:
1. Tener unas NICs/LAN dedicadas para las VMs = POCO PRACTICO
2. Vía VLAN. que opera sobre la red física del host. Esto es lo más habitual (en este caso, el tag de la VLAN lo pone el switch de red, no el host)

Esa VLAN puede ser gestionada directamente por el switch en colaboración con los BRIDGES de los host.
Podemos hacer que el bridge etiquete el tráfico de las VMs con un tag de VLAN concreto, y que el switch de red lo reenvíe a la VLAN correspondiente.

El tema es cómo se hace este trabajo.
OVLM puede ponerse en comunicación directa con el kernel de Linux, para hacer este trabajo... vía el agente VDSM en los host.

Pero hay otras opciones más avanzadas.
Hay una herramienta que usamos mucho en el mundo LINUX que nos permite configurar de forma sencilla esos brigdes: OpenVSwitch
PERO... OLVM no trabaja directamete con OpenVSwitch.

Hay otra herramienta llamada OVN (Open Virtual Network) que es un proyecto Opensource patrocinado por RedHat, que nos permite crear redes virtuales de forma avanzada apoyándose en OpenVSwitch, y que OLVM si soporta vía un proveedor de red externo, que podemos instalar o no en el engine/hosts.
Esta es la mejor opción... la recomendada.

Lo normal es usar OVN -> OpenVSwitch -> Linux Bridge  para crear redes virtuales para las VMs, que se integren con la red física del host y con el switch de red.

Podemos o no usar esas herramientas.


    OLVM -> VDSM -> Linux Bridge 
    OLVM -> VDSM  -> OVN -> OpenVSwitch -> Linux Bridge

OVN no es una herramienta que se instale en cada host. Lo que se instala en cada host es OpenVSwitch y un controlador OVN, que se comunica con un OVN centralizado, que puede estar en el engine o en un host independiente.

De hecho, cuando instalamos el OLVM, es algo que podemos decidir, si queremos usar o no OVN (y por ende OVS) para gestionar las redes virtuales de las VMs.

Este trabajo es algo que se hace cuando instalamos OLVM y va en paralelo con la infraestructura de red física del host y con el switch de red.
No es algo con lo que a posteri estamos enredando. Es parte de la "instalación" y puesta en marcha del OLVM.

Puede ser que en el futuro, cambie o amplie esta configuración... pero no es algo que harán los usuarios normales del OLVM. Es algo que hace el administrador de la plataforma, y que tiene implicaciones importantes en la infraestructura y equipos de red de la empresa / hosts.


Otra cosa será el uso... El uso en general es bastante simple:
Nos definiremos una Logical Network, que es un concepto lógico que nos ofrece OLVM para gestionar las redes de las VMs.

Entiendo una Logical Network como una necesidad de conectividad de las VMs.

Y lo que haremos será definir Profiles de VNICs, que son plantillas de configuración de las VNICs de las VMs, que se vinculan a una Logical Network concreta.

    VM X -> vNIC1 -> vNIC Profile -> Logical Network -> Red física del host -> Red empresa...
    ------------------------------------------------
        Hasta aquí es lo que habitualmente 
        configuramos en el día a día

Como mucho, me puede pasar que queramos una VLAN nueva... que habrá que registrar en el switch de red y en el OLVM (Logical Network) y crear un vNIC Profile para esa Logical Network.

Pero otra cosa es la instalación de todo este tinglado. Y esto si es más complejo.

---

Si podemos hacer la comunicación/conexión de las VMs entre si, directamente con Linux Brigde, porqué querríamos usar OpenVSwitch y OVN?
- OpenVSwitch y OVN nos ofrecen más funcionalidades
- Es más robusto.

La gestión de todo esto es compleja.
Y requiere de mucha configuración.

Openvswitch al final lo que hace es montar linux Bridges.
OpenVnetwork lo que hace es configurar varios OpenVSwitch.

Básicamente la diferencia está en:

    - Crear 50 bridges de linux, con sus configuraciones particulares. (BAJO NIVEL DURO)
    - Configurar 1 openvswitch, sobre cada host, al que le pido que genere las redes que necesito (5 redes = 5 peticiones -> 50 linux bridges) (MAS ALTO NIVEL )
    - Configurr 1 openvnetwork, que cuando pido que quiero una determinada red la genera (configura los openvswitch) en todos los openwswitch de los hosts. (1 operación)


---

                                Red física (cableado) + Red lógica (switches)
                +------------------------------SWITCH-------------------------------------------+   
                |                                                                               |   
    HOST 1     NIC1                                                                 HOST 2    NIC 2
                |                                                                               |
        BRIDGE EXT (Switch virtual)                                                     BRIDGE EXT
            |                                                                               |
            |                                                                               |
        BRIDGE INT (Switch virtual)                                                 BRIDGE INT (Switch virtual) <--- INFRA!
            |           |                                                           |       |
            |           |                                                           |       |
            VNIC1     VNIC2                                                      VNIC3     VNIC4
             |          |                                                           |       |
             VM1        VM2                                                         VM3     VM4

OPCION 1: Puedo tirar 4 comandos en Linux, para montar los 4 bridges                                <<< BAJO NIVEL. DURO
OPCION 2: Puedo ejecutar 2 comandos de Openwsitch, para que me genere los 4 bridges                 <<< MEDIO NIVEL. MAS FACIL
          Ejecuto 1 comando en host1, que me va a generar los 2 bridges en host1
          Ejecuto 1 comando en host2, que me va a generar los 2 bridges en host2
OPCION 3: Puedo ejecutar 1 comando en OVN, que me va a generar los 2 comandos de Openvswitch en host1 y host2, que a su vez me van a generar los 4 bridges.                                                                                             <<< ALTO NIVEL. MAS FACIL. MAS ROBUSTO. MAS COMPLETO. MAS ESCALABLE.

Puedo hacer que OLVM hable con Linux
Pero puedo hacer que OLVM hable con OVN... OVN es una herramienta muy estable, usada en un huevo de proyectos (más allá de OLVM) y que nos ofrece muchas funcionalidades adicionales, que no nos ofrece Linux Bridge para montar redes virtuales para las VMs.

Esas funcionalidades extras puedo implementarlas yo, mediante configuraciones y programas adicionales, pero OVN ya me las ofrece de serie y de forma robusta.

Para comunicaciones muy simples, puedo usar Linux Bridge desde OLVM. Pero para comunicaciones más complejas, robustas y escalables, es mejor usar OVN.


Tengo un coche. Puedo cambiarle yo el aceite...y los filtros... 
    Si solo necesito eso.. ok.. voy tirando.
Si quiero trucarle el motor y meterle un turbo, nitro, etc.. mejor que me lo haga un taller especializado.


---

El almacenamiento...
Puedo tirar con un almacenamiento montado vía NFS? SI
Me interesa más una cabina? SEGURO!
Hay funcionalidades que NFS no te da.

Quiero backups. NFS curratelos!
                    Backup Incremental? SI... Montate un proceso que corra como demonio, con sync para que vaya ... FOLLON.
                CABINA: SEGURO QUE VIENE CON UN HUEVO DE UTILIDADES PARA HACER BACKUPS: Internos (mucho más eficientes)
                                                                                        Incrementales.


OLVM no gestiona nada! Orquesta! Pero no gestiona!

    Cómputo -> KVM/QEMU
    Almacenamiento -> CEPH, GlusterFS, NFS, Cabina, Linux(local).
    Redes -> Linux Bridge, OVN.

---

# Cómo hablamos con el OLVM

    - Administration Portal (Webapp - engine)
      - Administración completa de la plataforma: DataCenters, Clusters, Hosts, VMs, Storage Domains, Logical Networks, etc.
    - VM Portal (Webapp - engine)
      - Orientado a usuario final: Crear VMs, arrancarlas... 
        - Aqui solo se eligen: plantilalas de VMs, redes a las que quiero contectarla...
    - Instalación/configuración del OLVM...
      - AQUI HAY QUE TOMAR MUCHAS DECISIONES... En base a la INFRAESTRUCTURA que tengamos.
        - comandos
        - tiene impacto en la infra.
                Herramientas que se ofecen:
                - engine-setup
                - hosted-engine     Desplegar el engine como una VM dentro del propio OLVM.
                - engine-backup
                - engine-config
    - API REST (automatización)
      - Para automatizar tareas, integraciones con otras herramientas, etc.
  


---

# Linux

Linux no es un sistema operativo.
Es un núcleo (kernel).
Un sistema operativo no es un programa. Son miles de ellos.
Una parte de esos programas es lo que llamamos "kernel". El kernel son un huevo de programas.
Todo SO tiene un kernel. El kernel ofrece las funcionalidad básicas y más importantes para el control del hierro y la ejecución de programas. El kernel es el que se encarga de gestionar los recursos del sistema, como la memoria, el procesador y los dispositivos de entrada/salida. También :
- Planifica discos
- Seguridad
- ...
Windows tiene kernel?
Microsoft ha tenido 2 kernels:
- DOS
- NT -> windows NT, 2000, XP, Vista, 7, 8, 10, 11, Servers.

Linux se usa para crear muchos sistema operativos. De hecho es el Kernel de SO más usado del mundo, gracias a un SO llamado: ANDROID.
El sistema operativo que usamos habitualmente en los servidores de las empresas como se llama? GNU/Linux
Lo que pasa que ese SO es:
- Duro de instalar
- Muy flexible en cuanto a los componentes adicionales que se le pueden agregar

Hay distribuciones de ese SO (GNU/Linux), que son opinionadas, es decir, que ya vienen con una serie de decisiones predefinidas:
- Qué shells se utilizan
- Qué gestores de paquetes se utilizan
- Qué GUIs
- Qué servicios se instalan por defecto
- Instalador

Dentro de las distros:
- Debian: Ubuntu, Mint, Kali, Raspbian
- Red Hat Enterprise Linux: Fedora, Oracle Linux, AlmaLinux, Rocky Linux
- SUSE Linux Enterprise Server: openSUSE, SLES
- ...


# Unix

Unix ERA un SO de la americana de telco AT&T. Se dejo de hacer hace más de 20 años.

Hoy en día UNIX son 2 estándares: SUS + POSIX.
Cualquier SO que cumple con esos estándares decimos que es un sistema operativo UNIX.

- Oracle Solaris: UNIX®
- IBM AIX: UNIX®
- HP-UX: UNIX®
- Apple macOS: UNIX®

---


# CEPH

Es un programa/protocolo para almacenamiento distribuido. Está soportado directamente por el kernel de Linux. Es un proyecto Opensource, patrocinado por RedHat.


Básicamente nos permite:
- Usar los HDDs de una seríe de computadoras para ofrecer un alamcenamiento escalable, distribuido y tolerante a fallos en red.

            CephFS  Bloques   Objetos  
             \        |       /
                    CEPH
   (internamente lo que ofrece es almaceniamiento de objetos)
        /             |               \
    Host1           Host2           Host3   
    HDD1            HDD1            HDD1
    HDD2            HDD2            HDD2
    HDD3            HDD3            HDD3

# Tipos de almacenamiento:

- Orientado a ficheros (NFS, CIFS, SMB)
- Orientado a bloques (iSCSI, FC)
- Orientado a objetos (S3)
    Lo que guardamos es un objeto con un ID (clave)
        Objeto:
        - Imagen de SO
        - Disco de una VM
        - Fichero 

CEPH lo que hace es cuando le llega un archivo/objeto lo divide en trozos.
De cada trozo hace al menos 3 copias y las distribuye entre los distintos hosts que forman el cluster de CEPH (HA + Escalabilidad)
Cuando tengo que escribir un archivo, lo estoy escribiendo en varios HDD simultáneamente, y cuando tengo que leer un archivo, lo estoy leyendo de varios HDD simultáneamente. Además en varias máquinas (usando distintos tubos de red) para que sea más rápido. El rendimiento de lectura/escritura es brutal. Y además es tolerante a fallos, porque si se cae un host, los otros hosts siguen teniendo copias de los trozos de los archivos.

Es opensource... pero el que está detrás es RedHat. Y RedHat tiene su propia distribución de CEPH de pago: RedHat Ceph Storage.

Openshift <- Kubernetes. Openshift viene de serie con CEPH.
OpenStack <- Para clouds privados (para yo montar mi AWS, Azure)
    Redhat tiene su distro de pago.. que viene preparada para CEPH: RedHat OpenStack Platform.


En el mundo VMWare que sería lo equivalente?
    VMWare vSAN