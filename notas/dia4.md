
        OLVM

        HOST1                       HOST2                       HOST3
        VDSM                        VDSM                        VDSM
        VM(engine)                  VM1                         VM2
            qemu-guest-agent        qemu-guest-agent            qemu-guest-agent
                                    VM3                         VM3
                                    qemu-guest-agent            qemu-guest-agent

        VDSM =  Agente a nivel de host
                VDSM (Virtual Desktop and Server Manager) se encarga de:
                - Recibe peticiones del Engine para operar MVs en el host donde reside:
                  - Crear
                  - Power on/off
                  - Pausar / Reanudar
                    (muchas de esas operaciones se realizan a través de la API de libvirt para finalmente gestionar procesos QEMU) 
                - Comunica al Engine información sobre la MV (BASICAMENTE LO UNICO QUE PUEDE HACER EL VDSM es monitorizar el proceso de QEMU)
                  - Estado de la MV
                    - Encendida/apagada... 
                  - Estadísticas
                      - Ver si ese proceso está corriendo o no
                      - El consumo de cpu y memoria de ese proceso
                      - El uso de red por parte de ese proceso
                      - El uso de i/o por parte de ese proceso
                - Monta los discos de la MV en el host donde reside (si es necesario) o los desmonta cuando no se necesitan ya.
                - Uno de ellos (SPM) se encarga de gestionar el almacenamiento compartido y de coordinar la creación de snapshots y clones de discos.
                - Monta los LinuxBridge en su máquina.. y los va configurando (SWITCHES DE RED INTERNOS) donde conectamos las VMS
                - Sellando una MV, el VDSM ejecuta el comando virt-sysprep en el host donde reside la MV.
    La IP de la VM es accesible por VDSM?
    O eso es trabajo del agente interno de la VM?
    El SO que estyá instalado el la VM es accesible por VSDM?
    VDSM puede hacer un apagado ordenado de una MV?
    El hostname de la MV es accesible por VDSM?
    Todas las configuraciones y operaciones INTERNAS de la MV no son accesibles por VDSM.
        - IP
        - Hostname
        - Reinicio ordenado
        - Apagado ordenado

        qemu-guest-agent: Agente a nivel de VM
            - Se comunica con el VDSM a través de un canal de comunicación (virtio-serial)
            - Permite al VDSM acceder a información interna de la MV y realizar operaciones internas de la MV.
                - IP
                - Hostname
                - Reinicio ordenado
                - Apagado ordenado
        watchdog: Proceso que se encarga de monitorizar el estado interno de la MV y avisar al VDSM si detecta que la MV está colgada.
                  Pantallazo azul de windows
        virtIO  : virtio-net, virtio-scsi, virtio-balloon, virtio-serial, virtio-console

Si a una MV le doy 8Gbs de RAM pero internamente está usando 4Gbs, a qué datos puede acceder VDSM?
Sabe el VSDM que está usando los 4Gbs? O ve los 8 Gbs? Ve los 4Gbs... QEMU va cogiendo solo lo que la VM va usando



---

kill en posix, heredado en linux.
Lo que hace es mandar señales a un proceso. Señales posix.
Es un mecanismo de comunicación entre procesos definido en el estándar posix.

Hay muchas formas de comunicar procesos:
- Socket
- Pipes
- Shared memory
- Portapapeles de windows
- Signals




> La IP es siempre la del bridge del host. 

Los bridge no tienen IP, son un switch virtual que se gestionan no vía RED, sino API (kernel de Linux).

    HOST1
        NIC - ETH0 - BRIDGE0 - tap - VM1
                             - tap - VM2
    HOST2
        NIC - ETH0 - BRIDGE0 - tap - VM1
                             - tap - VM2

---


    Programa -> Necesita ejecutar trabajos ---> Ese trabajo son INSTRUCCIONES QUE MANDO A LA CPU PARA QUE EJECUTE
                                                Para ese trabajo necesito DATOS (Esos datos los tengo en RAM*)
                                                Las CPUs tiene memoria donde poner datos? Tiene su cache interna
                                                Y vamos moviendo datos de la RAM a la cache de la CPU y de la cache de la CPU a la RAM.
                                                Hay distintos niveles:
                                                L1, L2, L3
                                                L1 y L2 son por core
                                                L3 Entre cores
                                                Quién lleva el trabajo a la CPU? THREAD/HILO de ejecución de un programa.


                                                Hay CPUs con CORES capaces de ejecutar el trabajo de 2 hilos de ejecución a la vez (Hyperthreading). Otros cores de otras cpus no tienen hyperthreading y solo pueden ejecutar un hilo de ejecución a la vez.

        En la práctica "me da igual" ejecutar 2 hilos en un core que ejecutar 2 hilos uno en un core y otro en otro core. Lo importante es que se ejecute el trabajo de los 2 hilos.
        "me da igual" . Lo pongo entre comillas.. porque hay cositas a tener en cuenta.
            Si esos 2 hilos comparten datos! me interesa que esos 2 hilos estén en el mismo core para que compartan la cache L1, L2.
            O al menos que sean ejecutados en el mismo socket de CPU para que compartan la cache L3.

            Eso si mejoraría algo el rendimiento de ejecución de los 2 trabajos.
            
                                                        +-------------+
                --- Hilo1 va generando unos datos------>|   CORE1     |
                                                        |             |
                --- Hilo2 va consumiendo esos datos --->|      CACHE  |
                                                        +-------------+

                                                        +-----------------------------+  Socket1
                                                        | +----------------+          |
                --- Hilo1 va generando unos datos------>| | CORE1   CACHE -|---+      |  
                                                        | +----------------+   V      |
                                                        |                  Cache L3   |
                                                        | +----------------+   v      |
                --- Hilo2 va consumiendo esos datos --->| | CORE2   CACHE <|---+      |
                                                        | +----------------+          | 
                                                        +-----------------------------+      

                                                        +-----------------------------+  Socket1
                                                        | +----------------+          |
                --- Hilo1 va generando unos datos------>| | CORE1   CACHE -|--------------------------------+
                                                        | +----------------+          |                     v
                                                        +-----------------------------+                    RAM
                                                                                                            |
                                                        +-----------------------------+  Socket2            |
                                                        | +----------------+          |                     |
                --- Hilo2 va consumiendo esos datos --->| | CORE2   CACHE <|--------------------------------+
                                                        | +----------------+          | 
                                                        +-----------------------------+      

        En general esas diferencias son despreciables.. solo para programas MUY ESPECIALES las tendríamos en cuenta.

        Pero se nos complica más. Lo que configuro en el OLVM en: Número de sockets, Número de cores por socket, Número de hilos por core..
        tiene impacto en esto? NINGUNO. Son sockets virtuales, cores virtuales y hilos virtuales. No tienen nada que ver con la arquitectura física de la CPU.
            Los 2 sockets virtuales con 2 cores virtuales y 2 hilos virtuales por core.. 
            pueden estar ejecutándose en 1 socket con 8 cores físicos de 1 hilo.
        Entonces para que ostias valen esas configuraciones? HAY PROGRAMAS CUYA LICENCIA SE PAGA POR SOCKET, CORE...
        Entonces, hay algo que podamos hacer para estos programas tan especiales, que pueden beneficiarse de ejecutar sus hilos en el mismo core o en el mismo socket?  SI
        Pero no esos parámetros.

    * Los tengo en RAM.. pero allí han sido generados o un programa los ha puesto al leerlos de HDD, de red.

---

# Sellado de máquinas (LINUX)

Lo que hacemos es despersonalziar la máquina virtual para que cuando genera instancias de ella (normalmente lo hacemos en plantillas) no tenga datos identificativos de la máquina original:

    sudo rm -f /etc/hostname
    sudo rm -f /etc/machine-id
    sudo rm -f /etc/ssh/ssh_host_*
    history -c
    ....

En realidad, aunque yo podría entrar dentro y ejecutar todos esos comandos, hay un programa (paquete disponible en linux) que hace todo eso por mi. 
Se llama "virt-sysprep" y lo que hace es ejecutar todos esos comandos y otros más para dejar la máquina virtual despersonalizada.

Cómo está la máquina ahora mismo (mientras estamos haciendo la plantilla) ? Esta parada
Es posible estando parada entrar en la máquina a ejecutar esos comandos?  NO

Ese comando virt-sysprep no se ejecuta dentro de la máquina virtual, sino que se ejecuta en el host donde está la máquina virtual.
Ese host monta los discos a su nivel (de hecho los tiene montados, para presentarlos a la VM) y como conooce la estructura de carpetas y archivos (la define posix) puede entrar en esos discos y borrar lo que deba ser borrado.

No necesitamos instalar este paquete virt-sysprep en la máquina virtual, sino en el host donde está la máquina virtual.
El engine, cuando marcamos SELLAR la máquina virtual, lo que hace es ejecutar ese comando virt-sysprep en el host donde está la máquina virtual?
No puede.. está en otra máquina.
Quién lo ejecuta ese comando? El agente que está en el host donde está la máquina virtual. El VDSM. El engine le dice al VDSM que ejecute ese comando virt-sysprep en el host donde está la máquina virtual.


DataCenter
    Plantillas
    Cluster
        Host1
                VM1
                VM2
        Host2
                VM3
                VM4

---

VM1

    BASE
            < Añadí un paquete a linux: net-tools
    snapshot1
            < Añado un paquete a linux: htop
    snapshot2
            < Añado un paquete a linux: mc
    snapshot3
            Estoy trabajando en mis cositas
        

            Quiero borrar el snapshot 1.. qué implica?
            - Voy a perder el net-tools? NO
            - Pero... net-tools está en el snapshot2? NO

           Entonces, a qué conclusión llegamos? Qué implica borrar el snapshot1? Hay que mete todo lo que está en el snapshot1 en el snapshot2, dando por bueno lo que hay en snahshot2, en caso de conflicto, prevalece lo que hay en snapshot2.

           Crear un snapshot es abrir capa nueva => INMEDIATO
           Borrar un snapshot es cerrar capa     => HAY QUE MOVER DATOS DE UNA CAPA A OTRA => LENTO

            El snapshot3 no tendía que ser actualizado.


---

base
snapshot2            sin archivo /home/alumno/fichero.txt                                         
activa                                                              <<<< OJO!

---

# Alta disponibilidad

Crítico en un entorno de producción.

De qué quiero la HA? VMs
Pero ello puede ser que necesite REDUNDANCIA a nivel de HOSTs. No siempre... puede ser!

## Situaciones:


| SITUACION                                         | HOST ACTIVO/RESPONDE?   | VM en ejecución?     | Respuesta/Trabajo habitual                                             |
|---------------------------------------------------|-------------------------|----------------------|------------------------------------------------------------------------|
| Mantenimiento planificado a nivel de host         | Si                      | Sí                   | Apagar vm de forma ordenada o migrarlas                                |
| Proceso QEMU muerto de una VM                     | Si                      | No                   | VDSM informa al engine y el engine toma decisiones - política definida |
| Proceso QEMU vivo, pero VM muerta                 | Si                      | Si, no responde      | VDSM informa? NO. WatchDog deja de informar a VDSM? Si lo tengo!       |
|                                                   |                         |                      | Si no tengo Watchdog... estoy jodido!                                  |
| Host no operativo. Se me joden las NICs de ls VMs | Si, pero con problemas  | Puede.. lo miro      | Analizo causa y tomo decisiones: Migrar, Apagar, cambiar configuración |
| Host no responde... pierde conexión a red con engine | Puede ser.. No lo se | Puede ser.. No lo se | Aislar/fence del host                                                  |
| Host caído (a efectos prácticos igual que arriba) | No                      | Puede ser.. No lo se | Aislar/fence del host                                                  |
| Engine caído                                      | A quién?                | Yo que sé... si      | Reiniciar engine/Migrar. No hay efectos colaterales en cuanto a VMs    |
| Problema backend Storage                          | Si                      | Seguramente no       | Como sea recuperar storage                                             |

Si host no responde o host está caido... una cosa que si podría intentar hacer el engine sería? REINICIARLO (muy a malas.. apagarlo) pero para eso que hace falta?
GESTION DE ENERGIA (IPMI, Redfish, iLO, DRAC, etc) y que el host tenga esa gestión de energía habilitada y configurada en el engine.

Nuestros hosts del lab no tienen gestión de energía... pero es que no son hosts.. son tristes VMs dentro de un HOST. El botonazo sería reiniciar la VM.
Esto no hay nada para ello.. porque no es algo ni remotamente habitual en un entorno de producción. En un entorno de producción, los hosts son físicos y tienen gestión de energía.
En un entorno real no quiero DOBLE CAPA DE VIRTUALIZACION. Virtualización anidada.

Es importante que el almacenamiento me ofrezca garantías. Tiene que ofrecer HA por si mismo -> 
- Buenas cabinas de almacenamiento con tropecientos HDD, RAIDs, Fuentes de alimentación redundantes, controladoras de red redundantes y de almacenamiento.
Si no consigo recuperarlo.. más vale que tenga backups exportados a otros Dominios de almacenamiento. Porque si no, me quedo sin nada.

Una VM altamente disponible, es cuando tengo capacidad para reiniciarse en otro host después de un fallo:
- Interrupción de servicio
- Pérdida de contenido en la RAM
- Se podrá ver agravado en base al tiempo de detección del problemna, aislamiento del host y reinicio de la VM en otro host.


HA: No significa que el servicio debe estar funcionando 24x7. 
    Esto lo puedo garantizar? NO
    Tratar de garantizar un determinado tiempo de servicio pactado previamente con el cliente. SLA (Service Level Agreement)
    99% -> 3,65 días de indisponibilidad al año (fallos, mantenimiento, backups, etc)
    99.9% -> 8,76 horas de indisponibilidad al año (fallos, mantenimiento, backups, etc)
    99.99% -> 52,56 minutos de indisponibilidad al año (fallos, mantenimiento, backups, etc)
    99.999% -> 5,26 minutos de indisponibilidad al año (fallos, mantenimiento, backups, etc)

Hay otro concepto que usamos mucho que es el de FAULT TOLERANCE. 
Que significa que si se produce un fallo, el servicio sigue funcionando sin interrupción. Esto es mucho más caro de conseguir y no siempre es necesario.
Y esto implica siempre clusters ACTIVOS-ACTIVOS. No siempre es necesario.

# Migración frente a recuperación.

Migración es cuando los hosts coopera y la vm está viva y se mueve de un host a otro.
Hay una microparada... muchos sistemas lo que hacen es encolar peticiones a red durante ese espacio.

# Recuperación ante fallo:

La VM está jodida... y lo que es peor.. puede que incluso el host donde estaba la VM esté jodido.
Y digo que es peor porque en este escenario, si el host está jodido, ni siquiera sé si la VM está jodida o no.

Y aquí está la diferencia entre NO OPERATIVO y NO RESPONDE.

En el primer caso, puedo hacer fácil un fencing del host (aislamiento) y reiniciar la VM en otro host.
En el segundo caso, el problema es más grave.. como ostias aislo al host si no responde? No puedo. 
    Y si no puedo aislarlo, cómo me aseguro que la VM no se está ejecutando en ese host? No puedo por las vías tradicionales... Aquí entra el concepto de Alquiler (LEASE) de recursos de almacenamiento compartido. Si el almacenamiento es compartido, puedo hacer un fencing del host y reiniciar la VM en otro host. Si el almacenamiento no es compartido, no puedo hacer nada.

# FENCING

Apartar un host dudoso / AISLARLO... Cómo: Actuando sobre un dispositivo de gestión de energía del host (IPMI, Redfish, iLO, DRAC, etc): LO CRUJO!

Fencing no intenta arreglar las VMs solo garantiza que el host donde estaban NO PUEDE ESTAR EJECUTANDOLAS. Y si el host no puede ejecutar las VMs, entonces puedo reiniciarlas en otro host.

Cuando aislo (CRUJO) un host, puedo remover/quitar los leases de almacenamiento que tenía ese host y que impedían que otro host pudiera ejecutar las VMs que estaban en ese almacenamiento compartido.

Y entonces, arranco las VMs en otro host y ya está. La VM se reinicia en otro host y el servicio vuelve a estar disponible. (CON PERDIDAS DE DATOS, TIEMPOS, MEMORIA)
La gracia es que si luego trato de iniciar el otro host, y lo consigo, que no pueda ejecutar las VMs que esta ejecutándo previamente... No se le permitirá porque no tendrá los leases de almacenamiento que le permitan ejecutar esas VMs, y el engine no se lo dará.