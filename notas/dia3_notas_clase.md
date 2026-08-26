
Entendimiento de la instalacfión que existe en el lab.
Conceptos más avanzados de networking 
Conceptos de almacenamiento
    vvv
Trabajar con VMs:
    Parametrización que podemos hacer en las VMs
    Flujos de trabajo con VMs



SPM = ROLE QUE SE ASIGNA A HOST
Dominio maestro = ROLE QUE SE ASIGNA A UN STORAGE DOMAIN

Los 2 tienen que ver con almacenamiento:

    El HOST con role SPM, lo que hace es que el agente del host (VDSM) se encarga de coordinar las operaciones de almacenamiento, como la creación de discos virtuales, la migración de discos entre hosts y la gestión de snapshots. El SPM también es responsable de mantener la consistencia de los metadatos del almacenamiento y garantizar que las operaciones se realicen de manera ordenada.

    Esos metadatos que va preparando/gestionando/almacenando el SPM, se guardan en un Storage Domain que tiene el rol de "Dominio Maestro". Este dominio maestro es esencial para la operación del almacenamiento en el entorno de virtualización, ya que contiene información crítica sobre los discos virtuales y las operaciones de almacenamiento.

---

Servidor NFS
    Disco Físico <- Formato XFS
      Crea una carpeta /datos
      Exporta esa carpeta vía NFS
        nfs://servidor_nfs/
        
Host 1
    MOUNT de esa carpeta /datos del servidor NFS -> /datos_del_servidor_nfs
    A esa carpeta yo no le puedo elegir el sistema de archivos. La gestiona el servidor NFS. El host 1 solo monta el export del servidor NFS.

    VM1
    
        /datos_del_servidor_nfs/disco1_vm1

    Lo que hace el host es crear un archivo dentro de esa carpeta montada, que es el disco virtual de la VM. 
    Ese archivo se trata como un VOLUMEN, que vincula a la VM1.

    A esa VM se le presenta ese archivo como si fuera un volumen físico.
    Y la VM SI TIENE la capacidad de asignar a ese volumen su sistema de archivos: EXT4.

SR-IOV es una capacidad que tienen algunas tarjetas de red para permitir que una sola tarjeta física exporte funciones de red virtualizadas.
Me permite que una sola tarjeta de red pueda presentarse como múltiples interfaces de red . Esto permite que, en nuestro caso:
- El host tenga una interfaz lógica de red, conectada a la tarjeta física, que se puede usar para la comunicación del host con la red.
- Las VMs puedan tener sus propias interfaces de red virtuales, vinculadas a la tarjeta física a través de SR-IOV.

La tarjeta se presenta a la red externa como si fueran múltiples tarjetas físicas, con distintas direcciones MAC y capacidades de red independientes. 

En este caso, no se usa un bridge Linux, sino que las VMs se conectan directamente a la tarjeta física a través de SR-IOV, lo que puede mejorar el rendimiento de la red y reducir la sobrecarga de virtualización.

    NORMAL / MAS FLEXIBLE

        VM1 -> tap -> LinuxBridge (~Switch) -> NIC física
        VM2 -> tap ->
        VM3 -> tap ->

    ALTERNATIVA

        VM1 -> tap -> LinuxBridge (~Switch) -> NIC física
        VM2 -> tap ->
        VM3 --------SR-IOV------------------->

    Mejor rendimiento, pero menos flexible. Porque si la VM3 está usando SR-IOV, no puede migrar a otro host, porque la tarjeta física está en el host original. Si la tarjeta se cae, la VM3 pierde conectividad incluso dentro del mismo host. Podría hacer un bod.. presentando varias tarjetas a la VM3.. y ella tendría que manejar la agregación de enlaces.
                                                
                                                -> NIC2    
    VM1 -> tap -> LinuxBridge (~Switch) -> Bond -> NIC1 
    VM2 -> tap ->                          
    VM3 -> bond --------SR-IOV-------------------> NIC1
                --------SR-IOV-------------------> NIC3

---

    Rocky Linux     <- CentOS
    Alma Linux 9
RHEL
    ^
    centos(Rama de git)
    ^
    Fedora

---

En esas máquinas alma-linux tenemos el agente (o varios) de OLVM instalado.
Realmente en OLVM no hay un agente como en VMware (tools).
Lo que tenemos son muchos agentes:
- Agente cliente / Guest Agent:   El que se encarga de la comunicación entre el OLVM y la VM
    Como paquete instalable:    qemu-guest-agent

        dnf install qemu-guest-agent
- watchdog:   Detecta un bloqueo de la VM y avisa al OLVM  mientras no hay ese bloqueo.
              Va mandando un heartbeat al OLVM para avisar que la VM está viva.
              En OLVM puedo configurar el comportamiento que deseo para la VM cuando no se recibe el heartbeat. Por ejemplo, reiniciar la VM.
- spice-vdagent:  Escritorio: Resolución de pantalla, portapapeles
- kernel virtio:  Drivers de red y disco para la VM
                  Balloon


    Nuestras máquinas Alma tienen de RAM:
        1GB Definida
        1GB Garantizado
        4GB de Máxima

    Lo que va a ver la máquina cuando arranque es lo DEFINIDO: 1GB
    Si en un caso, veo (YO) que la máquina necesita más RAM, podría YO aumentarla mientras está arrancada hasta el máximo de 4GB. 
    Más vale que el SO de la VM tenga soporte para hotplug de RAM, porque si no, no va a poder usar esa RAM extra que le estoy dando.
    Pero no es algo que se haga en automático. Yo tengo que hacer un cambio en la configuración de la VM para aumentar la RAM mientras está arrancada.

    Lo más follonero es lo de garantizado.

    Acaso puedo quitar a una VM RAM? NO


    De hecho, en un estado saludable, una máquina qué % de memoria RAM debería estar usando? 100% Coño, para algo la pago!
    Para que se usa la RAM? Alamacenar información volatil que estamos usando. Qué tipo de información?
        USO:
    - Código de los programas
    - Datos que manejan los programas
    - Estado de ejecución de un proceso/hilos
        CACHE/BUFFERS: 
    - Buffers (de red, de disco, de pantalla, etc)
    - Cache (guardar datos para un acceso más rápido)

    El SO es bastante listo.
        Si lee un archivo de disco -> RAM (cache)
        Si es necesario volver a leer ese archivo -> Lo lee de RAM (cache) (ESTO ES LO QUE HACEN LAS BBDD van colocando páginas/bloques-trozos de archivo- en RAM para que la próxima vez que se necesite, no tenga que ir a disco)

    Con el tiempo el SO debe llenar la cache hasta las trancas! Hasta que no quede no un triste BYTE libre. 100% de uso en la RAM.
    Y ESO ES SANISIMO!

    Otra cosa es que si el SO en un momento dado necesita RAM para colocar un programa o sus datos, y está toda ocupada, que hace? 2 opciones:
    - Bajar páginas de la RAM a HDD (paginación o swap)
    - Liberar espacio de cache/buffers. 
      - La cache hace que todo vaya más rápido.. pero si no hay ram.. pues que vaya más lento.. pero que todo siga funcionando.

    Balloon:
        Es un mecanismo / Agente que corre dentro de la VM.
        Si el Gestor (OLVM)  necesita más RAM (hemos hecho un overcommit de RAM), el Gestor le dice al Balloon que necesita más RAM. 
        El ballon puede influir el comportamiento del SO de la VM para que libere RAM:
        - conseguir que baje páginas de la RAM a HDD (paginación o swap)
        - liberar espacio de cache/buffers (inflándolo artifialmente)

    El SO sigue viendo la misma cantidad de RAM, pero hay una parte de la RAM que está siendo aparentemente "consumida" por el Balloon, y el SO no puede usarla. El Balloon (GLOBO) se infla y ocupa RAM, que el SO ya no puede usar. Si lo desinflo libera RAM y el SO puede usarla para otros procesos.
    Esa memoria RAM realmente no es USADA por el Balloon, El balloon solo hace como que la está usando.
    Esa RAM que supuestamente está usando el balloon, queda libre FISICAMENTE para us uso por otras VMs. 

        1GB Definida
        1GB Garantizado

        En este caso, el mecanismo de balloon no entraría en funcionamiento.

        1GB Definida
        512MB Garantizado

        En este caso, el mecanismo de balloon podría entrar en funcionamiento caso que el Gestor (OLVM) necesite más RAM para otras VMs. El balloon se inflaría y ocuparía parte de la RAM de la VM, haciendo que el SO de la VM no pueda usar esa RAM, liberándola para otras VMs.


    Y puedo ver en un host con 16Gbs de RAM (olvidemonos de lo que usa el SO) host:
        VM1: 10Gbs  (esto es lo que ven las VMs)
        VM2: 10Gbs  (esto es lo que ven las VMs)
    Eso sería posible a priori? NO... solo tengo 16Gbs de RAM. O los usa una o los usa la otra.
        Lo puedo conseguir con truco(balloon).
            Pongo un balloon en la VM1 y otro en la VM2.
            El balloon de la VM1 se infla 1Gbs
            En La VM2 se infla 3Gbs

        Las máquinas siguen viendo 10Gbs
            Realmente la VM1 está usando 9Gbs (1Gb ocupados por un proceso corriendo dentro, que realmente no los usa.. solo los libera)
            Realmente la VM2 está usando 7Gbs (3Gbs ocupados por un proceso corriendo dentro, que realmente no los usa.. solo los libera)

        Aunque vean 10, solo tienen disponibles realmente 9 y 7 respectivamente. Y el host tiene 16Gbs de RAM, que es lo que realmente hay.

---

Formato RAW vs QCOW2

RAW no trabaja con capas. -> Coger lo que hay en un disco físico y volcarlo tal cual en un archivo. Es un volcado bit a bit del disco físico. No hay compresión, no hay capas. Es simple y directo.

QCOW2 es un formato de disco virtual que ofrece varias ventajas sobre el formato RAW, especialmente en entornos de virtualización. Trabaja con capas. Qué permiten las capas.

Para que nos hagamos una idea, lo que tendríamos es una capa base, que queda "congelada" y no se puede modificar.
Es donde viene SO y paquetería instalada.

Cuando empiezo a hacer modificaciones en la VM, se van guardando en una capa nueva, que es la capa de cambios. Esta capa de cambios puede crecer y ocupar más espacio en disco a medida que se realizan más modificaciones en la VM.

- Si hago un snapshot, lo único que se hace es crear una nueva capa de cambios, que se superpone a la capa base y a las capas de cambios anteriores. Los snapshots se hacen de forma inmediata. No se copia nada...

Al operar, lo que ve el SO es la superposición de todas las capas. El SO no sabe que hay capas, solo ve un disco virtual que refleja el estado actual de la VM.

Puedo crear por ejemplo máquinas sin estado (STATELESS), que entre arranque y arranque borran la capa de cambios y vuelven a la capa base. Esto es útil para entornos de laboratorio o pruebas, donde quiero que la VM siempre arranque en un estado limpio.

---

# Cloud-init?

Esto se usa mucho hoy en dia... de hecho lo que más usamos.
Es un paquete que monto a nivel de SO (Linux). Qué permite...

Si tengo plantilla, tiro de plantilla!
Cuando creo una VM, si no tengo plantilla, por ejemplo, porque voy a crear la plantilla, como se configura el SO?

Qué prefiero, trabajar con una ISO o trabajar con DISCO?
En la ISO viene un INSTALADOR. Me tengo que comer el proceso de INSTALACION DEL SO... tarda 3 minutos? NO... tarda un huevo! Y tengo que tomar muchas decisiones.
En el DISCO viene una instalación ya realizada por alguien. No tengo que correr instalador y no tengo que tomar decisiones durante esa instalación. ESTO VA MUCHO MAS RÁPIDO, y además puedo AUTOMATIZAR la creación de una VM.

Ahora... lo necesito es PERSONALIZAR LA INSTALACION YA HECHA.
La instalacion esta guay que no me la coma.. pero durante la instalación se da cierta condfiguración del entorno:
- Nombre de host
- Usuario y contraseña
- Configuración de red
- Configuración de SSH
Eso son parámetros que admiten los instaladores de las distintas distros de GNU/Linux.

Cloud-init es un paquete que puede ejecutarse sobre una instalación ya realizada... y configurarla basándose en unos datitos que escribiremos un archivo YAML... de 8 lineas.. cutre cutre.

Paso ese archivo a la VM y cuando arranca, cloud-init lo lee y configura el SO de la VM según los parámetros que le hemos pasado.

Esto lo usamos un huevo hoy en día


HOY EN DIA QUIERO AUTOMATIZAR TODO LO QUE PUEDA AUTOMATIZAR.
Qué ventajas tenemos por ejemplo al automatizar la creación de una VM?
- Libera horas al humano. Trabajos que antes hacía un humano ahora los hace un programa.
- Automatizar la creación de una VM es automatizar una tarea.
  Pero cuando automatizo muchas tareas, puedo automatizar un PROCESO!
  Puedo montar un programa que revise el estado de CPU/RAM de mis 3 VMs. Si ve que voy apretado, quiero que genere una cuarta máuina virtual en AUTOMATICO, y la configure como parte de un cluster de balanceo de carga. Y si ve que no hace falta, que la apague y la borre (no quiero que se quede ocupando espacio)... si el día de mañana hace falta de nuevo: La creo en automático. ESCALABILIDADA AUTOMATIZADA.

  El objetivo final es llegar a AUTOMATIZAR LA CREACION/GESTION DE TODA LA INFRAESTRUCTURA.

  Hago un despliegue de INFRA para un cliente.
  Pero otro cliente necesita el mismo despliegue (o similar). Lanzo el mismo proceso de automatización y en minutos tengo la infraestructura para el segundo cliente. Y si el cliente 1 necesita más recursos, puedo escalar su infraestructura en automático.

  Aquí hablamos de 2 conceptos reyes hoy en día en el mundo de sistemas:
    - IaC: Infrastructure as Code. Infraestructura como código. La infraestructura se define en un archivo de texto, que puede ser versionado, revisado, compartido y reproducido. Esto permite que la infraestructura sea tratada como software, con todas las ventajas que eso conlleva.
      TERRAFORM / CLOUD FORMATION
      Quiero tratar la infra como si fuera código... y esto no es solo definir la infra en ficheritos de texto.
      Es someterla a control de versiones! Igual que se hace con el código fuente de un programa. 
      
      Esto permite que cualquier cambio en la infraestructura sea rastreable, reversible y auditable.

      Tengo una infra v1. Y me piden cambios en ella, para ir a v2 (porque tengo instalado un sistema v7, que tengo que subir a v8)... en alghunas instalaciones, quizás no en todas.
      Y puede ir mal el despliegue.. y tener que volver a la v7 del programa.. y quiero poder volver a la v1 de la infraestructura. Y quiero poder hacerlo de forma rápida y segura.

      Si no llevo control de cambios/versiones en la infra estoy jodido...
      Oye que hay que dar marcha atrás, que hemos hecho?
      - NOO.. ayer entro Menchu allí al OLVM.. y edito la VM.. que le cambió 3 cositas.. y luego Federico porque no funcionaba... y cambio otra cosita.
      Lo quiero todo en ficheros. Y que programas (TERRAFORM...) lean esos ficheros y hagan/editen la infra en automático. Y si hay que volver atrás, que lean los ficheros de la versión anterior y vuelvan a montar la infra como estaba. 
      NO QUIERO RELLENAR FORMULARIOS. OBSOLETO! Eso es de hace 15 años. Ya no trabajamos así. O no queremos trabajar así.

      La configuración de las máquinas/SO/Programas, también quiero automatizarlo y someterlo a control de versiones: ANSIBLE, PUPPET, CHEF, SALTSTACK, etc.

    - DEVOPS:
      Devops es una cultura, movimiento una filosofía.
      Toda cultura, movimiento defiende algo.
      Devops es un movimiento en pro de la AUTOMATIZACION de TODOS LOS TRABAJOS QUE HAY entre el DEV / OPS de un sistema.

---
Cloud-init es un paquete que puede ejecutarse sobre una instalación de SO ya realizada... y configurarla basándose en unos datitos que escribiremos un archivo YAML... de 8 lineas.. cutre cutre.

OLVM me permite inyectar archivos YAML a las VMs durante su creación, para que cloud-init los lea y configure el SO de la VM según los parámetros que le hemos pasado.

De hecho OLVM me permite que la mayor parte de los datos que pondría en el YAML los ponga en campos de un formulario, y él me genera el YAML y se lo pasa a la VM. Pero si quiero, puedo generar yo mismo el YAML y pasárselo a la VM.

---
# Qué pinta tiene un archivo cloud-init

```yaml
# Datos identificativos de la máquina
hostname: mi-vm

users:
  - name: usuario
    groups: [wheel, adm]
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    lock_passwd: false
  - name: root

chpasswd:
    expire: false
    list: |
      usuario:mi-contraseña
      root:mi-contraseña

packages:
  - vim
  - git
  - curl

runcmd:
  - # Arrancar el servicio de mysql
    - systemctl 
    - start
    - mysql
  
```

---

QEMU es quien ejecuta la VM... es el proceso que corre a nivel de SO.

VSDM es el agente que corre en el host y solicita la ejecución de un proceso QEMU... a libvirt. Y libvirt es la librería que se encarga de gestionar la virtualización a nivel de SO.

VSDM puede monitorizar ese proceso a nivel de SO, para ver si la máquina virtual está viva o no.
Si no está viva (SI EL PROCESO QEMU HA MUERTO A NIVEL DEL HOST), VSDM notifica a ENGINE.
Por otro lado, puede ser que el HOST MUERA (VSDM incluido). En ese caso, ENGINE no recibe notificación de VSDM, y puede asumir que el HOST ha muerto. Y si el HOST ha muerto, las VMs que estaban corriendo en ese HOST también han muerto.

Qué hace OLVM en este caso?
Esto es lo que definimos cuando marcamos la VM como HA (High Availability).

Problemas que podemos tener?
La VM está corriendo en Host1
Y Host1(VSDM) pierde comunicación con el ENGINE.
Qué piensa el engine? HOST 1 ha muerto... y que la VM ha muerto.
Pero la VM sigue corriendo en Host1. 
Y Está escribiendo sobre los HDD que están guardados en el Backend de almacenamiento, en el Data Domain.
Si El ENGINE decide "mover" (arrancar) la VM en otro host, por ejemplo Host2, y Host2 empieza a escribir sobre los mismos HDD que la VM estaba escribiendo en Host1, se puede corromper el sistema de archivos de esos HDD. Y eso es un desastre.

Ahi entra el concepto de "lease". ALQUILER DE LA VM
Es un mecanismo por el cuál, cuando una MV va a tomar discos de un Data Domain, pide un "lease" al ENGINE. El ENGINE le da el lease a la MV, y mientras tenga ese lease, nadie más puede tocar esos discos. Si el ENGINE detecta que la MV ha muerto, libera el lease y otra MV puede tomar esos discos.

Nos aseguramos que mientras una MV está corriendo, nadie más puede tocar los discos que está usando. Y si la MV muere, el ENGINE libera el lease y otra MV puede tomar esos discos.

---

# Watchdog.
Es monitorización INTERNA de la MV.

VSDM puede mirar si el proceso de QEMU está vivo o no. Pero si el proceso de QEMU está vivo, no significa que la MV esté en un estado SANO (Pantallazo azul de windows)... el proceso QEMU sigue corriendo, pero la MV no está en un estado sano.

Necesito un espia dentro de la VM que informe a VSDM si la MV está en un estado sano o no. Ese espía es el WATCHDOG.

Eso es un paquete que se instala dentro de la MV, y que manda un heartbeat a VSDM. Si VSDM deja de recibir el heartbeat, asume que la MV está en un estado no sano, y notifica al ENGINE. Y el ENGINE puede tomar decisiones sobre qué hacer con esa MV.