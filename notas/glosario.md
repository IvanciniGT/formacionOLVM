# Glosario OLVM

Glosario de apoyo para la formación. Las definiciones están escritas para entender el ecosistema y reconocer los objetos en el portal, no para sustituir la documentación del producto.

---

# Arquitectura y gestión

## OLVM

**Oracle Linux Virtualization Manager.** Plataforma de Oracle para administrar de forma centralizada hosts KVM, VMs, almacenamiento, redes, usuarios y políticas. Se basa en oVirt.

## oVirt

Proyecto upstream de virtualización sobre KVM del que proceden muchos componentes y nombres utilizados por OLVM, como `ovirt-engine`.

## Engine

Servicio central de gestión de OLVM. Mantiene inventario y estado, presenta el portal y la API y coordina operaciones sobre hosts, VMs, storage y redes. Comparación aproximada: **vCenter**.

## VDSM

Agente de gestión instalado en cada host. Recibe instrucciones del Engine y opera sobre libvirt, storage y red del host. Significa **Virtual Desktop and Server Manager**.

## KVM

Subsistema del kernel Linux que utiliza las extensiones de virtualización del procesador y permite ejecutar código invitado de forma acelerada.

## QEMU

Proceso que representa y ejecuta una VM, presenta sus dispositivos virtuales y utiliza KVM para acelerar la virtualización cuando está disponible.

## libvirt

Capa de gestión y API para hipervisores. En el host, VDSM se apoya en libvirt para administrar procesos QEMU/KVM y sus dispositivos.

## Plano de control

Camino por el que se decide, configura y coordina una operación. Ejemplo: `Engine → VDSM → configuración del host`.

## Plano de datos

Camino real que siguen los bloques de disco o las tramas de red. El Engine no está en el recorrido de cada bloque ni de cada paquete.

## Data Center

Contenedor lógico superior de OLVM que agrupa Clusters, Storage Domains y Logical Networks bajo un nivel de compatibilidad. No es una VM ni un host.

## Cluster

Conjunto de hosts con características y políticas compatibles para ejecutar y migrar VMs. Define, entre otros aspectos, redes requeridas, compatibilidad y políticas de scheduling.

## Host

Servidor Oracle Linux con KVM, libvirt y VDSM que aporta CPU, memoria, red y acceso al almacenamiento para ejecutar VMs. Comparación aproximada: **host ESXi**.

## Versión de compatibilidad

Nivel funcional establecido para Data Center o Cluster. Limita las características disponibles y permite que hosts/componentes operen con un conjunto común de capacidades.

## HA

**High Availability.** Mecanismos orientados a recuperar el servicio tras fallos, por ejemplo reiniciando una VM altamente disponible en otro host cuando se cumplen los requisitos.

## Live migration

Movimiento de una VM encendida entre hosts compatibles minimizando la interrupción. Requiere coherencia de CPU, red, almacenamiento y configuración.

## Scheduling

Proceso de selección del host adecuado para arrancar o migrar una VM conforme a capacidad, afinidad, políticas, redes y otras restricciones.

## Self-Hosted Engine

Topología en la que el Engine se ejecuta como una VM con mecanismos específicos para su monitorización y recuperación. No significa que cada host tenga un Engine independiente.

## Data Warehouse

Componente que conserva información histórica y métricas para informes y análisis. No controla hosts ni sustituye al Engine.

---

# Almacenamiento

## Backend de almacenamiento

Sistema real que proporciona capacidad: servidor NFS, cabina iSCSI/FCP, almacenamiento local u otra plataforma compatible.

## Storage Domain

Objeto administrado por OLVM que agrupa imágenes y metadatos sobre un backend compartido o local. Comparación aproximada: **datastore**.

## Data Domain

Storage Domain utilizado para Virtual Disks, snapshots, templates y otros datos de VMs. Es el tipo principal de dominio de datos en OLVM actual.

## Data Domain maestro

Data Domain que ejerce el rol maestro dentro del pool/Data Center. Participa en la coordinación y metadatos del almacenamiento. No debe confundirse con el host SPM.

## SPM

**Storage Pool Manager.** Rol que desempeña uno de los hosts para coordinar determinadas operaciones y metadatos de los Storage Domains. No transporta toda la E/S de todas las VMs.

## NFS

**Network File System.** Protocolo de almacenamiento de ficheros por red. Cada host monta el export compartido y OLVM organiza sobre él el Storage Domain.

## Export NFS

Ruta que un servidor NFS publica a clientes autorizados. Deben coincidir conectividad, permisos, identidad, versión y políticas de seguridad.

## NFSv3 / NFSv4

Versiones del protocolo NFS con diferencias de descubrimiento, puertos, seguridad y modelo de namespace. No se diagnostican suponiendo que todos los comandos se comportan igual.

## iSCSI

Transporte de comandos SCSI sobre IP. Presenta LUNs como dispositivos de bloques a los hosts. Se diferencia de NFS, que presenta un filesystem compartido.

## FCP

**Fibre Channel Protocol.** Acceso a almacenamiento de bloques sobre una infraestructura Fibre Channel.

## LUN

Unidad lógica de almacenamiento de bloques presentada por una cabina. No equivale directamente a un filesystem dentro de la VM.

## Local storage

Almacenamiento ligado a un único host. Simplifica algunos despliegues, pero limita capacidades como migración y HA basadas en acceso compartido.

## Virtual Disk

Disco creado o importado en OLVM y presentado a una VM mediante una interfaz virtual. Reside en un Storage Domain o puede mapear un recurso compatible.

## RAW

Formato o representación de disco con estructura simple y baja sobrecarga. Las funciones disponibles dependen del backend y de la política de asignación.

## QCOW2

Formato de imagen de QEMU que admite funciones como copy-on-write y snapshots, con costes y ventajas diferentes a RAW.

## Thin provisioned

El disco presenta un tamaño virtual mayor que los bloques consumidos inicialmente. Ahorra capacidad al principio, pero exige vigilar crecimiento y sobreasignación.

## Preallocated

La capacidad se reserva/asigna de forma anticipada según el backend y formato. Puede beneficiar determinados patrones de rendimiento a costa de mayor consumo inicial.

## Snapshot

Punto lógico del estado de discos y, según la operación, memoria/configuración de una VM. No es por sí solo una estrategia completa de backup.

## Template

Modelo reutilizable a partir del cual se crean VMs con una configuración e imágenes base controladas.

---

# Networking

## NIC

Interfaz de red del host. Puede ser física o, en un laboratorio anidado, una interfaz virtual presentada al host OLVM.

## Bond

Interfaz lógica que agrupa varias NICs para redundancia y, según el modo, distribución de tráfico. Debe coordinarse con el switch físico cuando el modo lo requiera.

## VLAN

Segmentación de capa 2 basada en etiquetas IEEE 802.1Q. Una Logical Network puede tener o no un VLAN ID.

## Bridge Linux

Switch de capa 2 implementado por el kernel Linux. Conecta puertos TAP/vnet de VMs con interfaces físicas, bonds o interfaces VLAN.

## Logical Network

Objeto de OLVM que expresa una red y su función dentro del Data Center/Cluster: VM Network, VLAN, MTU, red requerida, management, migration u otras propiedades.

## VM Network

Logical Network habilitada para conectar vNICs de VMs. Una red usada solo por el host para una función interna puede no marcarse como VM Network.

## vNIC

Adaptador de red virtual concreto conectado a una VM. Tiene modelo, MAC, estado y un vNIC Profile.

## vNIC Profile

Objeto que define cómo se conecta una vNIC a una Logical Network y qué políticas aplica: QoS, filtro, mirroring, passthrough, permisos u otras opciones.

## TAP / vnet

Interfaz del host que conecta el proceso de la VM con el bridge o mecanismo de red. Es el puerto del lado host correspondiente a la vNIC.

## Uplink

Interfaz física o bond por el que un bridge alcanza la red exterior del host.

## FDB

**Forwarding Database.** Tabla de direcciones MAC aprendidas por un bridge para decidir por qué puerto reenviar una trama. No es la tabla ARP ni una tabla de rutas.

## MAC address

Identificador de capa 2 de una interfaz Ethernet. Debe ser único dentro del dominio donde pueda producirse una colisión.

## MAC Pool

Rango administrado por OLVM para asignar MAC a vNICs de manera controlada y reducir duplicidades.

## MAC spoofing

Uso de una MAC de origen distinta de la autorizada. Puede ser malicioso o necesario en diseños concretos; debe habilitarse solo con una justificación y alcance controlados.

## Filtro `vdsm-no-mac-spoofing`

Filtro que ayuda a impedir que una vNIC emita con MAC de origen no autorizadas. Puede afectar a bridging interno, virtualización anidada o servicios que utilicen MAC virtuales.

## MTU

**Maximum Transmission Unit.** Tamaño máximo de paquete/trama admitido sin fragmentación en el punto correspondiente. Debe ser coherente a lo largo del camino.

## QoS

**Quality of Service.** Política que controla o prioriza el consumo de recursos, por ejemplo límites de ancho de banda asociados a perfiles.

## Port mirroring

Copia de tráfico de una interfaz hacia otra para análisis o monitorización. Tiene impacto de seguridad, rendimiento y privacidad.

## SR-IOV

Tecnología que permite exponer funciones virtuales de una NIC física y asignarlas de forma más directa a VMs. Aporta rendimiento, pero puede reducir flexibilidad.

## PF

**Physical Function.** Función principal de una NIC compatible con SR-IOV desde la que se administran Virtual Functions.

## VF

**Virtual Function.** Función ligera de una NIC SR-IOV que puede asignarse a una VM.

## Management Network

Red utilizada para la gestión de los hosts por OLVM. En la instalación suele llamarse `ovirtmgmt`, aunque el nombre no sustituye a la comprobación de sus roles reales.

## Migration Network

Red seleccionada para el tráfico de migración de VMs entre hosts. Puede compartir infraestructura o separarse según el diseño.

## Red requerida

Logical Network que debe estar correctamente configurada en los hosts de un Cluster. Su ausencia o falta de sincronía puede impedir que un host sea operativo.

---

# Diagnóstico

## Evento

Registro visible en el portal que describe una acción, advertencia o error relacionado con un objeto. Es el primer contexto, no siempre la causa raíz.

## Estado `Active`

Indica que un objeto como un Storage Domain está disponible administrativamente en su contexto. No garantiza por sí solo latencia, capacidad futura ni ausencia de errores.

## Estado `Uninitialized`

Indica que un Data Center todavía no reúne o no ha activado los componentes necesarios para completar su inicialización.

## Fuera de sincronía

Situación en la que la configuración real del host no coincide con la esperada por OLVM, frecuente al analizar redes administradas.

## Alcance del fallo

Conjunto de objetos afectados: una VM, un host, una red, un Storage Domain o todo el Data Center. Determinarlo es una de las formas más rápidas de reducir causas posibles.

## Prueba de división

Comprobación elegida para separar el camino en dos y localizar en qué mitad aparece el fallo: mismo host frente a otro host, IP frente a DNS o un host frente a todos.
