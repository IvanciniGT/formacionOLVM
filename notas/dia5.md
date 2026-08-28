
# Overcommit de CPU

En ocasiones asignamos más recursos físicos a las VMs de la capacidad real del host. Overcommit.
Puede ser de RAM, puede ser de CPU.

RAM: Es bastante complejo el tema. No es posible quitar RAM a un programa o VM(QEMU) en ejecución.
Para poder hacer esto, aplicamos técnicas bastante avanzadas y complejas. BALLOONING
Metiamos un globo dentro de la VM que si lo necesito hace como que usa memoria RAM (se infla) pero en realidad no la usa.. sino que la reserva, dejándola disponible/accesible para otra vm que la necesite. Cuando la VM necesita más RAM, el globo se desinfla y libera RAM para la VM.

CON LA CPU es más simple. Estamos por abajo usando LINUX, que es un kernel de SO de tiempo compartido.

Según llegan peticiones de ejecución de instrucciones de CPU, el kernel las va encolando y las va ejecutando según el orden de llegada. Pero puede ir alternando entre procesos, de manera que tenemos la sensación deque se están ejecutando cosas en paralelo, cuando en realidad se están ejecutando de manera secuencial.


    PROCESO 1: T1, T2, T3 (HILO 1)
    PROCESO 2: T1, T2, T3 (HILO 2)

    Pero quizás ambos hilos están siendo ejecutados por el mismo núcleo de CPU (y el core no tiene capacidad multihilo.. o si la tiene, pero tengo 4 hilos de SO). El hecho es que hay más hilos en ejecución que capacidad de ejecución tienen los cores del host.

    El sistema operativo lo que hace es ir encolando todas las peticiones y las va mandando a los cores de CPU según le interese.
        CORE1, T1(HILO1), T1(HILO2), T2(HILO1), T3(HILO1), T1(HILO2), T3(HILO2)

Aprovechando esto, podemos resolver fácil el overcommit de CPU.

Puedo tener 8 cores efectivos a nivel del host.
VM1: 6 cores
VM2: 6 cores
VM3: 4 cores

He solicitado/asignado 20 cores virtuales.

La idea de esto es que quizás no todas las VMs van a estar ejecutando procesos que necesiten CPU al mismo tiempo. Por lo tanto, puedo asignar más cores virtuales de los que realmente tengo en el host.
Mientras no se ejecuten simultáneamente, no hay problema. 
El problema viene cuando varias de ellas VMs empiezan a ejecutar procesos que necesitan CPU al mismo tiempo. 
Si en un momento dado hay más trabajos que cores efectivos a nivel del host, el SO empieza a encolar los trabajos y a ir alternando entre ellos, de manera que todos van a ir ejecutándose, pero más lentamente.

Qué criterio usa para decidir a qué MV le da más prioridad? Eso es lo que configuramos con los CPU Shares.
Es un valor relativo. No significa que la VM tiene garantizados esos cpus, Velocidad o Tiempo de ejecución. Significa que si hay competencia por los cores, la VM con más CPU Shares va a tener más prioridad para ejecutar sus procesos.

---

CPU Pinning es vincular vcpus a cores físicos de un host. Esto es útil para VMs que necesitan un rendimiento muy alto y no pueden permitirse esperar a que el SO les asigne tiempo de CPU.

> Ejemplo:
 
Tengo un host con un procesador con 8 cores de 2 hilos.

A nivel de la VM defino 4 vcpus.
Puedo coger y vincular esas 4 vcps a 2 cores físicos del host, haciendo uso del multihilo de esos cores. Y digo a cuales:
    
    vcpu0 -> core0 - hilo0
    vcpu1 -> core0 - hilo1
    vcpu2 -> core1 - hilo0
    vcpu3 -> core1 - hilo1

Si lo dejo libre... habría asignaciones random que podrían cambiar con el tiempo.

    Podría ser que en un instante del tiempo:
        vcpu0 -> core0 - hilo0
        vcpu1 -> core1 - hilo0
        vcpu2 -> core1 - hilo1
        vcpu3 -> core4 - hilo0

Esto es especialmente relevante cuando tengo cpus con varios sockets. Porque si no hago pinning, podría ser que una vcpu esté ejecutándose en un core de un socket y otra vcpu de la misma VM esté ejecutándose en un core de otro socket. Esto es malo porque la comunicación entre sockets (CACHE/MEMORIA) es más lenta que la comunicación dentro del mismo socket.

Son cosas pequeñas... pero en sistemas muy especiales pueden marcar la diferencia entre un rendimiento aceptable y un rendimiento excelente.

---

NUMA: Non-Uniform Memory Access. 

No todas las CPUs tienen la misma latencia de acceso a la memoria.
Porque no todas las CPUS están a la misma distancia FISICA de la memoria... especialmente cuando tengo varios sockets de CPU.

    Hay cores a los que les pilla más cerca unos determinados bancos de memoria que otros. Por contra, a otros cores les pilla más cerca otros bancos de memoria. Por lo tanto, el tiempo de acceso a la memoria no es uniforme.
    Esto lo puede tener en cuenta un hipervisor para ver qué cores de CPU le asigna a cada vCPU de una VM y qué bancos de memoria le asigna a cada vCPU de una VM.

Es una arquitectura de memoria en la que el tiempo de acceso a la memoria depende de la ubicación de la memoria con respecto al procesador. En sistemas NUMA, cada CPU tiene su propia memoria local y puede acceder a la memoria de otras CPUs, pero a un costo mayor.

En entornos virtualizados, es importante tener en cuenta la topología NUMA al asignar vCPUs y memoria a las VMs para evitar penalizaciones de rendimiento debido a accesos remotos a la memoria.

---

En servidores gordos (marcados como NUMA) CPU y MEMORIA se agrupan en nodos (interno). Acceder a la memoria local de un nodo desde los cores de ese nodo es más rápido que acceder a la memoria de otro nodo. 
Me puede interesar que esto se tenga en cuenta a la hora de asignar los vcpus y la memoria a las VMs. O puedo hacer que no se tenga en cuenta.

---

# Agentes en las vms:

- qemu-guest-agent: Es un agente que se instala dentro de la VM y permite al Engine comunicarse con la VM para obtener información y realizar ciertas acciones.
  - Operaciones: Apagar, reiniciar, congelar, descongelar, ejecutar comandos.
  - Información: Nombre del host, direcciones IP...
- watchdog: Es un mecanismo de supervisión que permite al Engine detectar si una VM está respondiendo o no. Si la VM deja de responder, el watchdog puede reiniciarla automáticamente. Manda latidos vía un dispositivo virtual. Si el engine deja de recibir latidos, asume que la VM está colgada y la reinicia. Va con temporizador.
- Los agentes de virtio: No es uno. En realidad son controladores/drivers que permiten hacer un uso más eficiente de los dispositivos virtuales. Por ejemplo, el driver virtio para la red permite una comunicación más rápida entre la VM y el host.
  - virtio-iscsi: Permite a la VM conectarse a un almacenamiento iSCSI de manera más eficiente.
  - virtio-balloon: Permite a la VM ajustar dinámicamente la cantidad de memoria que utiliza, lo que es útil para el overcommit de RAM.

---
# KSM
KSM (Kernel Same-page Merging) es una técnica que permite al kernel de Linux compartir páginas de memoria idénticas entre procesos o máquinas virtuales para reducir el consumo de memoria física. Esto es especialmente útil en entornos virtualizados donde muchas VMs pueden tener contenido de memoria similar.

La memoria sirve para muchas cosas. Una de las cosas por ejemplo que ponemos en memoria es EL CODIGO DE LOS PROGRAMAS.

Imaginad que tengo un host... con 10 VMs. Y todas ellas están ejecutando el mismo sistema operativo. Van a tener cada una (proceso QEMU) una copia del mismo código del sistema operativo en memoria. Esto es un desperdicio de memoria. KSM permite al kernel detectar estas páginas de memoria idénticas y fusionarlas en una sola página compartida entre todas las VMs, liberando así memoria física.

Puede ser que un proceso trate de modificar una página de memoria que está siendo compartida por KSM. En ese caso, el kernel crea una copia de esa página para ese proceso específico, permitiendo que el proceso modifique su propia copia sin afectar a los demás procesos que comparten la página original. Esto se conoce como "copy-on-write".

Eso lo hace un demonio que puedo tener corriendo en el host llamado ksmd. Este demonio se encarga de escanear la memoria en busca de páginas idénticas y fusionarlas. También se encarga de gestionar las páginas compartidas y de crear copias cuando un proceso intenta modificarlas.

Hay muchas estrategias para otimizar el uso de ram:
- KSM: Fusionar páginas idénticas entre procesos o VMs.
- Ballooning: Permitir a las VMs ajustar dinámicamente la cantidad de memoria que utilizan
- Swapping: Mover páginas de memoria inactivas a disco para liberar RAM.

En VMWare está este mismo concepto con otro nombre: Transparent Page Sharing (TPS). La idea es la misma: compartir páginas de memoria idénticas entre VMs para reducir el consumo de memoria física.

Si tengo vms con mismos sistemas operativos, similar paquetería, etc... es muy probable que haya muchas páginas de memoria idénticas que se puedan fusionar y compartir entre las VMs, reduciendo así el consumo de memoria física en el host.

ESTA ESTA BONITO. Como concepto.
Cual es la tendencia hoy en la industria?
Pasar un poco de todo esto. Por qué? Me sale más caro el ajo que la gallina!

Muchas veces gasto más recursos (dinero, tiempo, esfuerzo) en gestionar estas técnicas de optimización de memoria que lo que me costaría enchufar más memoria física al host. Además, estas técnicas pueden introducir complejidad y problemas de rendimiento si no se configuran correctamente. Por lo tanto, la tendencia actual es invertir en hardware con suficiente memoria física para evitar la necesidad de estas optimizaciones.

Una cosa es que YO SEA AMAZON AWS... que tengo máquinas físicas aberrantes: 96, 192 cores, con 4 Tbs de RAM... y vaya a ejecutar dentro VMs con 1-2-4 cores... compartidos (overcommit) y 2-4 gbs de ram (alguna 8)... muchos so iguales... doy a elegir entre 2/3 para muchos servicios.

CLOUD9: Un servicio de AWS para entornos de desarrollo virtuales. 2 cores y 2 gbs.. y sobraba. Y daban a alegir entre ubuntu 22.4 o Amazon linux 9. Y NO HAY MAS TOMAS!

En casos como este, me puedo beneficiar mucho de cosas como KSM, porque muchas de las VMs van a tener el mismo sistema operativo y van a estar ejecutando procesos similares, lo que significa que habrá muchas páginas de memoria idénticas que se pueden fusionar y compartir entre las VMs, reduciendo así el consumo de memoria física en el host.
Voy a tener ciento y la madre de VMs dentro de un host gigante con el mismo sistema operativo y paquetería. Y me interesa mucho optimizar el uso de memoria.

En una empresa para uso interno, esto es menos habitual.. aunque hay escenarios: ESCRITORIOS VIRTUALES. Muchas VMs con el mismo sistema operativo y aplicaciones, pero para diferentes usuarios. En este caso, KSM puede ser útil para reducir el consumo de memoria física en el host.
Windows cargarlo en RAM: 600Mbs-1Gb y tengo 50 VMs igualitas... Esto es mucho!
Y prefiero incluso sacrificar rendimiento (uso de cpu) a cambio de ahorrar RAM. Porque la realidad es que no va a haber tanto uso de CPU y el factor limitante en esos HOSTS va a ser RAM.Y puedo liberar un huevo aplicando este tipo de técnicas.

Si tengo VMs para BBDD y servidores de apps y sistemas de mensajería, proxies... Pues .. será cada una de su padre y de su madre... Se parecerán... pero poco más. NO SON CLONES!

Distintos tipos de software se comportan de manera diferente y es algo que tenemos en cuenta.
ETL = EXTRACT, TRANSFORM, LOAD. Software que extrae datos de una fuente, los transforma y los carga en otra fuente. 

Los tipicos programas de carga/migración de datos que se ejecutan a las 4 de la mañana y que no se ejecutan en paralelo con otros procesos. Backups...
Esos programas normalmente no entran en conflicto con otros tipos de programas que tenga. WEBLOGIC (esto se usa cuando tengo a mis empleados operativos por la mañana.. no por la noche)
En un escenario de este estilo KSM no aporta nada... pero por ejemplo BALLOONING sí. Porque puedo tener 2-3 VMs que se ejecutan a la vez y que compiten por RAM. Y si una de ellas necesita más RAM, el globo se infla y libera RAM para esa VM, mientras que las otras VMs pueden seguir funcionando con la RAM que tienen asignada.
En un host dedicado a escritorios virtuales, ballonning no aporta nada. Porque todas las VMs van a estar ejecutándose a la vez y van a necesitar su RAM asignada. No hay margen para liberar RAM de una VM para otra.

Puedo hacer un reparto muy fino de las VMs a los hosts... LO QUE PASA ES QUE ESTO ES COMPLEJO DE NARICES EL HACERLO BIEN! Y muchas veces no compensa! Me sale más caro que más hosts o más ram...

---

# Planificación de vms en hosts -> Reglas de Afinidad en OLVM

Ya hemos hablado de asignación de recursos, arquiectura, quien ejecuta qué...
Pero el punto importante es en este caso: Cuando una VM tiene que arrancar, migrar, recuperarse despues de un fallo, ¿En qué host del cluster se puede ejecutar? ¿En qué host conviene que se ejecute?

Y esa respuesta la da un módulo/componente del engine que se llama Scheduler. Este módulo tiene en cuenta las reglas de afinidad que hayamos definido para decidir en qué host ejecutar la VM.

Las reglas, aunque puedo darlas a nivel de vm, también se dan a otros niveles.
A nivel de cluster se definen algunas políticas generales, que el scheduler tiene en cuenta durante: 
- Planificación de arranque de VMs
- Planificación de migración de VMs
- Planificación de recuperación de VMs después de un fallo

Pero luego están las reglas propias de cada VM, que pueden ser más específicas y que el scheduler también tiene en cuenta.

# Cómo el scheduler decide en qué host ejecutar una VM?
Hay 2 pasos, que responden a las preguntas que plantábamos antes:
- Hosts candidatos. Los que potencialmente podrían albergar la VM. Esto lo decide el scheduler en base a las reglas de afinidad de VMs y a la disponibilidad de recursos en los hosts.
- Una vez identificado un conjunto de hosts candidatos, el scheduler determina cúal es el ideal en base a las políticas de más alto nivel definidas en el cluster.

Como el scheduler no encuentre ningún host candidato, la VM no se podrá ejecutar. Esto puede ocurrir si no hay hosts disponibles con suficientes recursos para ejecutar la VM, o si las reglas de afinidad son demasiado restrictivas y no permiten que la VM se ejecute en ningún host del cluster.

Hay que tener mucho cuidado con las reglas de afinidad. Están bien.. pero si abuso de ellas, llega un momento que no me entero de nada... y no sé porque leches una vm no se está arrancado.

## Condiciones obligatorias para que un host sea candidato a ejecutar una VM

- Que el host esté activo y en estado operativo.
- Que su CPU sea compatible con la CPU de la VM.
- Que tenga configuradas las redes lógicas a las que la VM necesita conectarse.
- Que pueda acceder al storage domain.
- Caso que tenga definido reglas de NUMA o de pinning que las cumpla / compatibilidad con la topología de la VM.
- Si la VM necesita de un tipo de dispositivo especial (GPU, USB, etc.) que el host tenga ese dispositivo disponible y compatible.

Hay otras condiciones que ya dependen de reglas de afinidad que hayamos definido para la VM o para el cluster. 

# Grupo de afinidad

Un objeto que definimos en el engine, dentro de un cluster.
Vamos a meter dentro de un grupo de afinidad:
- Una o varias VMs
- Opcionalmente uno o varios hosts
- Reglas de relación entre las VMs
- Reglas de relación entre las VMs y los hosts
- Defino el caracter preferente u obligatorio de cada una de esas reglas.

Es un poco follón. porque tenemos una pantalla para hacerlo todo... y son reglas de afinidad muy diferentes. 

## ¿Cuales son conceptualmente las 4 reglas de afinidad que necesitamos conocer?

OJO porque todas se rellenan en la misma pantalla.

### Relación positiva de afinidad entre 2 VMs: Preferente u Obligatoria.

En scheduller trata/obliga a que esas 2 VMs se ejecuten en el mismo host.
Esto es útil cuando tengo 2 VMs que se comunican mucho entre ellas y quiero minimizar la latencia de comunicación entre ellas o no quiero saturar la red del cluster con tráfico entre ellas.

> Opcion 1

    VM Tomcat (app web)     ------+
                                  | MISMO HOST
    VM MySQL  (base de datos) ----+

> Opcion 2

    VM Tomcat (app web)     ------+
                                  | MISMO HOST
    Redis (cache)  ---------------+

    VM MySQL  (base de datos) ----+ OTRO HOST   
    
> Opcion 3
    
    VM Tomcat (app web)     ------+
                                  | MISMO HOST POR NARICES
    Redis (cache)  ---------------+
                                                                ESTO SON 2 relaciones (2 grupos de afinidad) diferentes.
    VM Tomcat (app web)     ------+
                                  | MISMO HOST EN LO POSIBLE
    VM MySQL  (base de datos) ----+ 
    

### Relación negativa de afinidad entre 2 VMs: Preferente u Obligatoria.

En scheduller trata/obliga a que esas 2 VMs se ejecuten en hosts diferentes.

Lo usamos un huevo esto!
- Nodos redundantes que forman un cluster: 3 weblogic en cluster activo-activo. 3 VMs con mariaDB activo-activo.
    Si están en el mismo host... como se caiga el host me quedo sin servicio.. y he montado un cluster activ-activo para evitar precisamente eso.. y montar un cluster activo-activo es complejo.
            FAULT TOLERANCE
    Si solo quiero HA (poder recuperar el servicio rápido despues de un fallo... posiblemente activo / pasivo ) sea suficiente.. o directamente solo un activo... sin pasivo. Y que si jode host, se levante en otro host. Pero si quiero cluster activo-activo, necesito que las VMs estén en hosts diferentes.

Habitualmente este tipo de reglas las definimos como PREFERIDAS. Si tengo un cluster de 3 nodos... en lo posible los quiero separados, pero si no hay narices, que no se quede un nodo sin funcionar.

### Relación positiva entre vm-host

Las máquinas virtuales deben / deseo en lo posible ejecutarse en un host  de un grupo de hosts. 

Esto es útil cuando:
- tengo un grupo de hosts con características similares y quiero que ciertas VMs se ejecuten en ese grupo de hosts para aprovechar esas características.
    > Escritorios virtuales y tener habilitado a nivel de host el KSM. Si no tengo habilitado KSM en un host, las VMs que se ejecuten en ese host no se van a beneficiar de la optimización de memoria que ofrece KSM. Por lo tanto, quiero que todas las VMs que se beneficien de KSM se ejecuten en los hosts que tienen KSM habilitado. 
- tengo vms con similares características y quiero que todas ellas se ejecuten en nodos que reservo para esas vms. Aunque sea por tener las ocsas organizadas.
  > Voy a crear escritorios remotos (pool de vms) y quiero que todas esas vms se ejecuten en un grupo de hosts que reservo para escritorios remotos. Aunque no sea necesario, me interesa tenerlo organizado.

    MV-Escritorio1      Host1
    MV-Escritorio2      Host2
    MV-Escritorio3

    No es quiera que MV-Escritorio1 y MV-Escritorio2 estén en el mismo host. No me importa que estén en hosts diferentes. Pero quiero que estén en un grupo de hosts que reservo para escritorios remotos.
    > Potencial decisión del scheduler:
        MV-Escritorio1      Host1
        MV-Escritorio2      Host2
        MV-Escritorio3      Host1
    > Otra decisión del scheduler:
        MV-Escritorio1      Host1
        MV-Escritorio2      Host1
        MV-Escritorio3      Host1

Otro ejemplo:

Tengo un grupo de hosts que tienen gpus... y tengo una MV que necesita GPU
    GPU => Cualquier requisisto de dispositivo hardware

### Relación negativa entre vm-host

Quiero que unas determinadas VMs no se ejecuten o tiendan a no ejecutase en un determinado host o grupo de hosts. 

Por qué? Ejemplos:
Tengo un grupo de hosts que tienen gpus... y tengo una MV que no necesita GPU
Quiero que vaya a un host que tiene gpu? NO.                                    ME AGOTAN EL HOST!

Si tuvieramos un escenario como este:

    HOST 1: GPU         VM1 , VM2 /lleno (hasta las trancas)
    HOST 2: NO GPU      VACIO

        VM1 y VM2 no necesitan GPU.

    Quiero soltar VM3(que necesita GPU) ... que pasaría?
        OLVM migraría las VM1 y/o VM2 para hacer hueco a VM3? NO!
        Eso sería bonito! Pero eso no va a ocurrir. El scheduller entra en juego cuando una VM tiene que arrancar, migrar o recuperarse de un fallo. No entra en juego para mover VMs de un host a otro para hacer hueco a otra VM.
    En este escenario, VM3 se queda sin ejecutar! Y ESTOY JODIDO.
    Tendré que entrar yo a migrar VM1 o VM2 a HOST2 para hacer hueco a VM3. 

    Y esto es así en la mayor parte de sistemas de virtualización (incluso con contenedores y kubernetes).

## Combinaciones de las anteriores

Tengo host1
Tengo host2

VM1
VM2

Y necesito que esas vms vayan a uno de esos hosts, pero que no vayan al mismo.
    > Relación positiva entre vm-host: VM1 y VM2 deben ejecutarse en host1 o host2
    > Relación negativa entre vm-vm: VM1 y VM2 no deben ejecutarse en el mismo host

En general, mi recomendación es definir las reglas por separado. FACILITA UN HUEVO EL MENTENIMIENTO DE LAS REGLAS.

---

# Etiquetas de afinidad

## Esto era así
    Es una alternativa a los grupos de afinidad. Son más simples, pero menos potentes.

    Asigno a VMs etiquetas
    Y a los hosts también les asigno etiquetas.

    Esto se comporta como una relación vm-host positiva y obligatoria a la vez.

    - GPUs
    - Tipo de almacenamiento especial

## En la última versión de OLVM ha cambiado.

    Las etiquetas quedan solo como grupos.
    Creo que cambió en OLVM 4.7
    Si me voy a certificar, revisar la versión de OLVM que tengo en el examen. Porque si me preguntan por etiquetas, no es lo mismo que en la versión 4.6.


Eso es así cuando en la etiqueta defino tanto MVs como HOSTS.
Si solo defino MVs o HOSTs, lo que tengo es GRUPOS de VMs o grupos de HOSTS. 
    Y luego puedo usarlas en grupos de afinidad para definir relaciones entre grupos de VMs (las que comparten etiqueta) y grupos de HOST (los que comparten etiqueta).

Por ejemplo... caso GPUs.
Al final necesito una regla positiva de VMs-> HOSTS
Pero también una negativa de VMs-> HOSTS. 
Y no quiero tener que manetner actualizado el listado de HOSTS con GPU en ambos grupos de afinidad.

Si mañana meto una máquina copn GPU nueva al cluster no quiero tener que darlo de alta en 2 grupos de afinidad diferentes.

Si tengo una etiqueta "GPU" que es la que uso en ambos grupos de afinidad, solo tengo que dar de alta la nueva máquina en la etiqueta "GPU" y automáticamente se va a tener en cuenta en ambos grupos de afinidad.
    FACILITA EL MANTENIMIENTO Y REDUCE POTENCIALES ERRORES DE CONFIGURACIÓN.


Hay que intentar evitar demasiadas reglas. Pero casi siempre son necesarias algunas de ellas, para hacer un buen uso de los recursos del cluster.


---

# Instalación y mantenimiento de OLVM

Hay 2 formas grandes de instalar un cluster de OLVM:
- Montar el engine primero como una máquina independiente y luego ir añadiendo hosts al cluster.
    Esto es un stand alone engine. Esto lo malo es que me deja sin HA en el engine.
- Montar un self-hosted engine. Esto es un engine que se ejecuta dentro de una VM que está dentro del cluster. Esto me permite tener HA en el engine, pero es más complejo de montar.

En cualquier de los casos, una vez hecha la instalación, el mantenimiento del cluster se hace de la misma forma.

# Instalación de Self hosted engine 

En una instalación donde el engine está dentro de una VM gestionada por el engine... EIN???
esto es el problema de huevo y la gallina. 
Porque para instalar el engine necesito un cluster de hosts, pero para tener un cluster de hosts necesito un engine que lo gestione.

Cómo se hace esto? Se hace en 2 fases:
- Previo: Instalar un host y montarle la paquetería minima y el programa de instalacion de olvm
         En lugar de solicitar una instalación a hierro del engine (comando engine-setup), 
         se solicita una instalacion de tipo hosted-engine. 
         Esto hace que se instale un engine previo dentro del host.. en una VM.
         Ese engine previo es el que ayuda a registrar el host y crear un cluster.
         Posteriormente en ese host se instala el engine definitivo como una vm (otra), y se pega el cambiazo de engine previo a engine definitivo. Y el engine definitivo es el que se queda gestionando el cluster.
    
    Ahora bien, si lo tengo dentro es para conseguir la funcionalidad de HA del engine.
    Y por ende, querré más de un host con el engine dentro de una VM para que si se cae un host, el engine siga funcionando en otro host. Cuando insalo así, además se montan unos agentes a nivel de los host que ejecutan el engine, de forma que esos monitorizan el engine y si se cae, dan el cambiazo de engine a otro host.
    
    Lo que pasa es que no quiero habitualmente que todos los host puedan ejecutar el engine.
    Lo que hacemos es marcar explicitamente qué hosts pueden ejecutar el engine.

# Mínimos a nivel de cada host:
- Oracle Linux Minumal
- FQDN y IP (en red de gestión)
- DNS/Fecha 
- repos

# Instalacion mínima:
- ovirt-engine
- dependencias requeridas para nuestro caso concreto (almacenamiento, dispositivos especiales, etc.)

# En un cluster standalone engine:

- Instalar el paquete de olvm
- Ejecutar el engine-setup para configurar el engine y crear la base de datos.
    - Decisiones que tenemos que tener pretomadas, y que el engine setup nos preguntará:
      - Mecanismo/herramientas para gestión de usuarios y autenticación (Interna, LDAP, Keycloak, etc.)
        Keycloak es una IAM (Identity and Access Management) que permite gestionar usuarios, roles y permisos de manera centralizada. Además admite autenticación federada, lo que permite a los usuarios iniciar sesión con sus credenciales de otros sistemas (como Google, GitHub, Active Directory, etc.). Redhat impulsa mucho el uso de Keycloak.. de hecho tiene su propia versión de pago. Y ya dijimos que el OLVM viene del ovirt que es de Redhat. Por lo tanto, es normal que Keycloak sea la opción recomendada para la gestión de usuarios y autenticación en OLVM.
      
      - DataWareHouse: Es un almacen donde ir guardando información de monitorización y métricas del cluster.
        Puedo no montarlo.. Lo habitual es montarlo... aunque la monitorización la vamos a llevar normalmente por otro lado. (En realidad es una BBDD que se crea en postgreSQL)
            Tenerlo interesa para tener una vista actual más amplia cuando accedo a la UI del engine. Y tener cierta visión de métricas.

        PERO! OLVM no es una herramienta de monitorización. Lo normal ( Y viene preparado para ello ) es usar una herramienta tipo Prometheus/Grafana para monitorizar el cluster. Y OLVM tiene un plugin que permite exportar métricas a Prometheus. Y luego con Grafana puedo hacer dashboards y alertas.
      
      - OVN:
        No es obligatorio, aunque facilita mucho el mantenimiento sobre todo en clusters grandes. Nos gestiona redes virtuales. Acabaremos con muchas redes lógicas dentro del OLVM. Y si solo hago uso de mecanismos nativos de OLVM (Linux bridges) para la gestión de redes virtuales, el mantenimiento se complica mucho (entra el equipo EXTERNO DE REDES). OVN es un proyecto de código abierto que proporciona una solución de red virtual para entornos de virtualización. Permite crear y gestionar redes virtuales de manera centralizada, facilitando la configuración y el mantenimiento de las redes lógicas dentro del cluster OLVM. 

        

Si tengo una instalación Standalone:

    HOST1
        engine
    HOST2
    HOST3

    El HOST1 no puede albergar VMs.

Si tengo una instalación Self-hosted engine:

    HOST1
        vm1
            engine
        vm4
    HOST2
        vm2
            engine
        vm5
    HOST3
        vm3

    Todos los hosts pueden albergar VMs.

Para este tipo de herramientas solemos centralizar la gestión del logging. Vamos a tener N servidores (HOSTS) con un montón de VMs e irán generando logs en las carpetas /var/log de los hosts. Tiramos de herramientas tipo ELK (Elasticsearch, Logstash, Kibana) o Loki/Grafana para centralizar la gestión de logs. Esto nos permite tener una visión global de lo que está ocurriendo en el cluster y facilita la detección y resolución de problemas.