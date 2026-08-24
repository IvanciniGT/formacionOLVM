# Día 1 · Fundamentos, arquitectura y ecosistema OLVM

---

# Qué vamos a conseguir hoy

Hoy **no vamos a entrar en el portal** ni vamos a utilizar la instalación del curso.

No es un accidente. Es deliberado.

Si empezamos pulsando botones, aprenderéis dónde está cada menú. Si empezamos entendiendo la arquitectura, sabréis qué botón buscar, qué efecto tendrá y dónde mirar cuando falle.

Al terminar el día quiero que podáis explicar:

1. Qué es OLVM y qué parte de la virtualización realiza.
2. Cómo se relacionan Engine, VDSM, libvirt y QEMU/KVM.
3. Qué diferencia hay entre Data Center, Cluster y Host.
4. Para qué sirven Storage Domains y SPM.
5. Cómo se relacionan NIC, bond, VLAN, bridge, Logical Network y vNIC.
6. Qué portales y herramientas existen y cuándo usar cada uno.
7. Qué conceptos se parecen a vSphere y dónde deja de funcionar la analogía.

La frase del día es:

> **OLVM administra la virtualización; KVM la ejecuta.**

---

---

# Reparto de la jornada

| Bloque | Duración | Tema |
|---|---:|---|
| 1 | 35 min | Ecosistema OLVM y fundamentos mínimos de virtualización |
| 2 | 45 min | Arquitectura OLVM: Engine, hosts, flujos y bases de datos |
| Pausa | 15 min | |
| 3 | 45 min | Data Center, Cluster, Host, VM y SPM |
| 4 | 45 min | Conceptos y herramientas de almacenamiento |
| Pausa | 15 min | |
| 5 | 50 min | Conceptos y herramientas de networking |
| 6 | 50 min | Portales, API, herramientas, casos y consolidación |

Total: **5 horas**, incluidas dos pausas de 15 minutos.

---

---

# Bloque 1 · Ecosistema OLVM y fundamentos de virtualización

## La pregunta que ordena el curso

¿Qué es OLVM?

> **OLVM es la plataforma central que configura, monitoriza y administra un conjunto de hosts Oracle Linux KVM.**

No es el hipervisor. No ejecuta directamente las instrucciones de las VMs. Tampoco es una simple página web.

Si venís de vSphere, la primera aproximación es:

```text
OLVM Engine  ≈ vCenter Server
Host KVM     ≈ host ESXi
```

El reparto general se parece: un plano central administra y los hosts ejecutan.

## Las piezas, sin convertir esto en un curso de KVM

Necesitamos conocer cuatro nombres porque aparecerán en servicios, procesos y logs:

| Pieza | Qué hace | Qué no hace |
|---|---|---|
| **KVM** | Proporciona virtualización acelerada desde el kernel Linux | No administra Clusters ni usuarios |
| **QEMU** | Ejecuta el proceso de la VM y presenta su hardware virtual | No decide en qué host debe arrancar |
| **libvirt** | Gestiona localmente el ciclo de vida de VMs y dispositivos | No coordina toda la infraestructura |
| **VDSM** | Representa al host ante el Engine y ejecuta sus operaciones de gestión | No sustituye a KVM |

La cadena conceptual es:

```text
Engine → VDSM → libvirt → QEMU/KVM → VM
```

- El **Engine coordina**.
- **VDSM habla por el host**.
- **libvirt gestiona localmente**.
- **QEMU/KVM ejecuta la VM**.

No necesitamos hoy estudiar módulos del kernel, parámetros de QEMU ni XML de libvirt. Eso sería interesante en un curso de KVM, pero nuestro producto es OLVM.

## Plano de control frente a plano de datos

El Engine envía decisiones y recibe estado. El tráfico normal de una VM no viaja a través del Engine:

```text
CONTROL
Portal/API → Engine → VDSM → host

DATOS
VM → red del host
VM → almacenamiento accesible por el host
```

En vSphere ocurre algo parecido: vCenter no está en medio de cada paquete o de cada operación de disco.

## Por qué hace falta OLVM si ya existe KVM

Un host KVM aislado puede ejecutar VMs. OLVM añade gestión del conjunto:

- inventario común;
- selección de host y scheduling;
- migración;
- políticas de Cluster;
- permisos centralizados;
- almacenamiento y redes compartidos;
- templates y pools;
- eventos y observabilidad;
- alta disponibilidad;
- backup y recuperación del gestor.

## oVirt, RHV y OLVM

- **oVirt** es el proyecto abierto del que procede gran parte de la arquitectura.
- **Red Hat Virtualization** fue un producto basado en oVirt.
- **OLVM** es la plataforma de Oracle basada en oVirt e integrada con Oracle Linux.

Por eso veremos nombres como `ovirt-engine`, `ovirt-engine-dwh` u `ovirt-log-collector`.

Pero no digamos “OLVM es exactamente oVirt”. La frase correcta es:

> **OLVM está basado en oVirt.**

Oracle publica sus versiones, requisitos, correcciones y condiciones de soporte. La documentación de oVirt puede ampliar conceptos; la documentación de Oracle manda para nuestro entorno.

## OLVM no es Oracle VM clásico

**Oracle VM** utilizaba Xen. **Oracle Linux Virtualization Manager** administra KVM.

Si una búsqueda habla de Oracle VM Server, Xen o el antiguo Oracle VM Manager, estamos ante otro producto. No reutilizaremos sus comandos ni procedimientos.

## Comprobación oral

1. ¿Qué ejecuta la VM? **QEMU utilizando KVM en un host.**
2. ¿Qué administra el conjunto? **El Engine.**
3. ¿Qué comunica el Engine con el host? **VDSM.**
4. ¿Pasa todo el I/O por el Engine? **No.**
5. ¿OLVM y Oracle VM son el mismo producto? **No.**

Si estas cinco respuestas están claras, ya tenemos suficiente KVM para empezar a estudiar OLVM.


---

# Bloque 2 · Arquitectura OLVM y flujo de gestión

## El Engine

El **Engine** es el plano central de control. Es una aplicación Java basada en WildFly que mantiene inventario y estado, valida permisos, aplica políticas y coordina los hosts.

Administra hosts, VMs, redes, almacenamiento, usuarios, migración, HA, eventos y métricas.

Lo que no hace es ejecutar las instrucciones de las VMs.

```text
OLVM Engine ≈ vCenter Server
```

Es la comparación más útil del curso: ambos son gestores centrales, no hipervisores.

## Qué ocurre cuando arrancamos una VM

Imaginad que alguien solicita arrancar `vm-app-01`.

1. La petición llega al Engine desde un portal, la API o una automatización.
2. El Engine valida usuario, permisos, estado, redes, almacenamiento y políticas.
3. Selecciona un host compatible con capacidad suficiente.
4. Ordena la operación a VDSM en ese host.
5. VDSM prepara recursos y utiliza libvirt.
6. libvirt inicia QEMU y KVM acelera la ejecución.
7. La VM utiliza red y almacenamiento desde el host.
8. VDSM informa y el Engine actualiza el estado.

```text
Usuario/API
    │
    v
Engine: autoriza, valida, selecciona y coordina
    │
    v
VDSM → libvirt → QEMU/KVM → VM
                            ├── red del host
                            └── almacenamiento
```

El botón parece sencillo porque el Engine oculta toda esta lógica.

## ¿Qué ocurre si cae el Engine?

Las VMs que ya funcionan **normalmente continúan ejecutándose en sus hosts**, porque sus procesos no viven dentro del Engine.

Pero perdemos o degradamos:

- administración central;
- nuevas decisiones de scheduling;
- creación, edición y operaciones coordinadas;
- visibilidad consolidada;
- respuestas automáticas que dependan del Engine.

> La caída del Engine no apaga automáticamente las VMs, pero deja la plataforma sin su plano central de gestión.

Es la misma prudencia que aplicaríais a vCenter: las VMs siguen en ESXi, pero vCenter no es prescindible.

## ¿Qué ocurre si VDSM no responde?

Los procesos QEMU existentes pueden seguir ejecutándose, pero el Engine pierde una vía fiable de control y monitorización.

“No puedo hablar con el host” no significa “el host está apagado”. Arrancar sus VMs en otro nodo sin confirmar el fallo podría provocar dos instancias escribiendo en los mismos discos.

> **Falta de comunicación no equivale a fallo confirmado.**

Más adelante veremos fencing, leases y HA.

## Engine independiente y Self-Hosted Engine

En un **Engine independiente**, el gestor vive fuera de la infraestructura que administra.

En **Self-Hosted Engine**, el Engine vive como una VM dentro del entorno. Agentes específicos de hosted-engine en los hosts pueden vigilar y recuperar esa VM. Para ofrecer HA, Oracle documenta un mínimo de dos hosts KVM.

```text
Cluster OLVM
    ├── Host A
    ├── Host B
    └── Engine VM protegida mediante hosted-engine
```

Se parece a ejecutar VCSA dentro de un clúster vSphere, pero no es simplemente una VM normal: OLVM incorpora mecanismos hosted-engine específicos.

## Engine DB y Data Warehouse

OLVM utiliza PostgreSQL con dos finalidades:

| Base | Finalidad |
|---|---|
| `engine` | Inventario, configuración, relaciones, estado e información operativa persistente |
| `ovirt_engine_history` | Histórico de configuración y métricas del Data Warehouse |

```text
engine DB ── extracción y transformación ──> ovirt_engine_history
```

El estado actual pertenece al ámbito operativo del Engine. La evolución histórica es el terreno del DWH y de Grafana.

## Casos para localizar la capa

- El portal falla pero las aplicaciones siguen respondiendo: pensad primero en **gestión**, no en KVM de todos los hosts.
- El Engine no ve un host pero una VM de ese host responde: **control y ejecución son capas distintas**.
- Una VM concreta no encuentra su disco: investigad su camino de **storage**, no el DWH.
- Grafana no tiene histórico pero el inventario funciona: puede fallar **DWH/monitorización**, no el hipervisor.

---

---

# Bloque 3 · Data Center, Cluster, Host, VM y SPM

## La pregunta más útil

Cuando no sepáis dónde configurar algo, preguntad:

> **¿A qué nivel pertenece?**

```text
OLVM Engine
 └── Data Center
      ├── Storage Domains
      ├── Logical Networks
      └── Cluster
           ├── Host A
           └── Host B
```

## Data Center

Es una entidad lógica de alto nivel que agrupa recursos físicos y lógicos. Relaciona Clusters, hosts a través de esos Clusters, Storage Domains y Logical Networks.

Para inicializarse necesita al menos un Cluster, un host activo y un Data Storage Domain activo.

Decir “Data Center es el nivel del almacenamiento” ayuda a recordar una relación, pero es incompleto. No agrupa sólo almacenamiento.

```text
OLVM Data Center ≈ vSphere Datacenter
```

## Cluster

Agrupa hosts KVM compatibles bajo políticas comunes:

- arquitectura y tipo de CPU;
- versión de compatibilidad;
- scheduling y balanceo;
- migración;
- HA y afinidad;
- funciones de las redes utilizadas.

Oracle exige arquitectura de CPU compatible: no mezclamos hosts Intel y AMD en el mismo Cluster.

```text
OLVM Cluster ≈ vSphere Cluster
```

La compatibilidad de CPU recuerda a EVC como problema funcional, pero no es el mismo mecanismo.

## Host

Es el servidor Oracle Linux KVM que aporta CPU, memoria, red y acceso al almacenamiento. Ejecuta QEMU/KVM, libvirt y VDSM.

Cada host pertenece a un Cluster.

```text
Host KVM ≈ host ESXi
```

## Máquina virtual

Una VM es un objeto administrado por el Engine y una carga ejecutada en un host.

Se relaciona con:

- un Cluster que delimita dónde puede ejecutarse;
- discos ubicados en Storage Domains;
- vNICs conectadas a Logical Networks;
- permisos y políticas;
- opciones de migración y HA.

La VM no vive en el Engine: su definición se gestiona centralmente, sus discos viven en storage y su ejecución ocurre en un host.

## Storage Pool Manager

El **SPM** es un rol que el Engine asigna a uno de los hosts del Data Center.

Coordina metadatos y operaciones que deben serializarse: creación y manipulación de discos, snapshots y templates, además de asignaciones sobre almacenamiento de bloques.

No es el camino normal de I/O:

```text
Control: Engine → host SPM → metadatos
Datos:   VM → su host → almacenamiento
```

El host SPM puede ejecutar VMs. Si deja de estar disponible, el Engine puede asignar el rol a otro host.

No hay un equivalente directo y limpio en vSphere. No todos los conceptos necesitan pareja.

## Migración en vivo como prueba del modelo

Para migrar una VM encendida hacen falta, entre otros requisitos:

- origen y destino operativos;
- mismo Cluster;
- CPU compatible;
- capacidad en destino;
- red de migración;
- acceso a los discos y redes;
- ausencia de reglas o dispositivos que lo impidan.

Se parece a vMotion con shared storage: misma finalidad general, implementación diferente.

## Actividad de pizarra

Dibujad un Engine, un Data Center, dos Clusters, dos hosts por Cluster, un Storage Domain y dos Logical Networks.

Preguntas:

1. ¿Puede un host pertenecer a dos Clusters? **No.**
2. ¿Puede una VM migrar normalmente de un Cluster Intel a uno AMD? **No.**
3. ¿Cuántos hosts ejercen el rol SPM en el Data Center en un momento dado? **Uno.**
4. ¿Dónde se define una Logical Network? **En el Data Center; después se aplica al Cluster y se implementa en hosts.**

---

# Bloque 4 · Conceptos y herramientas de almacenamiento

## La regla de las capas

Cuando alguien dice “el disco de la VM”, puede estar hablando de cosas diferentes:

```text
Almacenamiento físico
        │
Protocolo o acceso
        │
Storage Domain de OLVM
        │
Virtual Disk
        │
Particiones y filesystem dentro del invitado
```

Si no separamos estas capas, el troubleshooting se vuelve caótico.

## Backend y forma de acceso

El backend puede ser una cabina, un servidor NFS, un target iSCSI, Fibre Channel, GlusterFS o discos locales.

Después aparece la forma de acceso:

- **NFS y GlusterFS:** almacenamiento basado en ficheros.
- **iSCSI y FCP:** almacenamiento de bloques mediante LUNs.
- **Local storage:** almacenamiento ligado directamente a un host.

NFS no es “un disco remoto”: proporciona un filesystem compartido. iSCSI no proporciona un filesystem terminado: presenta bloques sobre los que la plataforma construye sus estructuras.

## Storage Domain

El **Storage Domain** es la abstracción administrada por OLVM. Puede contener discos virtuales, snapshots, templates, ISO y metadatos.

```text
Storage Domain ≈ Datastore
```

Es una analogía útil, no una identidad técnica.

En almacenamiento de ficheros, las imágenes se materializan como ficheros. En almacenamiento de bloques, OLVM utiliza LUNs, grupos de volúmenes y volúmenes lógicos.

## Virtual Disk y filesystem invitado

La VM recibe un Virtual Disk. Dentro, el sistema operativo crea particiones, LVM y filesystems.

Ampliar un disco en OLVM no amplía automáticamente XFS, ext4 o NTFS dentro del invitado. Son capas y operaciones distintas.

## Shared storage frente a local storage

Si varios hosts ven de forma segura el mismo almacenamiento, podemos plantear migración, mantenimiento con evacuación y reinicio de VMs en otros hosts.

El almacenamiento compartido no garantiza por sí solo esas funciones, pero suele ser un requisito esencial.

Local storage puede ser válido, pero Oracle documenta restricciones de live migration, scheduling y fencing.

## SPM dentro del mapa

El SPM coordina metadatos y determinadas operaciones administrativas. No sirve todo el tráfico de disco:

```text
Operación administrativa:
Engine → host SPM → metadatos

I/O normal:
VM → host que la ejecuta → almacenamiento
```

## Herramientas de almacenamiento que utilizaremos

Hoy sólo situamos para qué sirve cada una:

- **Administration Portal:** crear, asociar, activar y revisar Storage Domains.
- **Eventos:** conocer por qué un dominio no se activa o una operación falla.
- **Logs de Engine y VDSM:** reconstruir la operación desde ambos lados.
- **Herramientas NFS:** revisar export, montaje, permisos y conectividad.
- **Herramientas iSCSI:** descubrir targets, iniciar sesiones y comprobar LUNs.
- **Multipath:** validar caminos redundantes en storage de bloques.
- **LVM:** entender las estructuras usadas sobre almacenamiento de bloques.
- **SPM:** observar qué host ejerce el rol; no es un programa que “abrimos”.

## Comparación con vSphere

| OLVM | vSphere aproximado | Precaución |
|---|---|---|
| Storage Domain | Datastore | Metadatos y operaciones diferentes |
| NFS backend | NFS Datastore | La integración de cada plataforma es propia |
| iSCSI/FCP LUN | LUN presentada a ESXi | OLVM construye sus estructuras sobre ella |
| Virtual Disk | VMDK como concepto | No implica mismo formato |
| SPM | Sin equivalente directo | No forzar una correspondencia |

## Comprobación oral

1. ¿NFS entrega ficheros o bloques? **Ficheros.**
2. ¿iSCSI entrega un filesystem terminado? **No, entrega bloques/LUNs.**
3. ¿Storage Domain y filesystem invitado son lo mismo? **No.**
4. ¿El SPM sirve todo el I/O? **No.**
5. ¿Local storage ofrece las mismas funciones que shared storage? **No.**

---

# Bloque 5 · Conceptos y herramientas de networking

## El mapa de capas

```text
NIC física
    │
  Bond                    opcional
    │
  VLAN                    opcional
    │
Bridge/configuración del host
    │
Logical Network + vNIC Profile
    │
vNIC de la VM
```

No todas las redes tienen todas las capas. El dibujo sirve para distinguir objetos.

## NIC física

Es el puerto real del host: enlace, velocidad, driver y conexión al switch.

En vSphere, la referencia mental es una `vmnic`.

## Bond

Agrupa NICs Linux para aportar redundancia y, según el modo, distribución de tráfico.

Se parece al teaming de uplinks. El objetivo general coincide; los modos y la configuración no son intercambiables.

## VLAN

Una VLAN es segmentación de capa 2 mediante 802.1Q.

> **Una Logical Network no es una VLAN.**

Una Logical Network puede asociarse a un VLAN ID, pero también existir sin etiquetado. La red lógica expresa conectividad dentro de OLVM; la VLAN es un mecanismo de red.

## Bridge

Para una red de VMs convencional, el host crea un bridge que conecta interfaces virtuales con la red del host.

La idea recuerda al switching de un vSwitch, pero no son componentes idénticos.

## Logical Network

Representa una necesidad de conectividad. Se define en el Data Center, se aplica a Clusters y se implementa sobre interfaces de hosts.

Puede cumplir funciones como:

- management;
- VM network;
- migration;
- display;
- tráfico de infraestructura.

```text
Logical Network ≈ Port Group
```

Es una analogía útil para entender dónde se conecta una VM, pero el modelo de objetos es distinto.

## vNIC Profile y vNIC

La **vNIC** es la tarjeta que ve la VM. El **vNIC Profile** define cómo puede conectarse a una Logical Network y participa en políticas y delegación.

```text
vNIC → vNIC Profile → Logical Network → red del host
```

## Separación de tráfico

No conviene transportar todo por la misma red sólo porque sea posible. Separar gestión, migración, display, VMs y almacenamiento ayuda a:

- reducir contención;
- mejorar seguridad;
- aislar fallos;
- asignar capacidad adecuada;
- facilitar diagnóstico.

Igual que en vSphere, el diseño lógico sólo funciona si switches, VLANs, MTU, routing y enlaces físicos están alineados.

## Herramientas de networking que utilizaremos

- **Administration Portal:** definir Logical Networks y revisar su estado.
- **Host Network Setup:** asociar redes con NICs, bonds y VLANs.
- **vNIC Profiles:** controlar cómo se conectan las VMs.
- **Eventos:** detectar redes obligatorias ausentes o no operativas.
- **Herramientas Linux:** `ip`, NetworkManager, bridges, bonds, rutas y MTU.
- **Switch físico:** trunking, VLANs, LACP y conectividad real. OLVM no configura mágicamente el switch.

## Tabla de traducción

| OLVM/Linux | vSphere aproximado |
|---|---|
| NIC física | vmnic |
| Bond | Uplink teaming |
| Bridge | Función de switching del vSwitch |
| Logical Network | Port Group, aproximadamente |
| vNIC Profile | Política/configuración de conexión, aproximadamente |
| vNIC | vNIC de la VM |
| VLAN ID | VLAN ID |

## Frases que debemos corregir

- “He creado una VLAN en OLVM.” → Has creado una Logical Network y quizá le has asignado un VLAN ID.
- “La VM se conecta al Engine para salir.” → Utiliza la red implementada en su host.
- “Si existe la Logical Network, el switch ya está configurado.” → Falso.
- “Bond suma siempre el ancho de banda de todas las NICs.” → Depende del modo, el flujo y el switch.
- “Port Group y Logical Network son exactamente lo mismo.” → Son una buena analogía, no el mismo objeto.

---

# Bloque 6 · Portales, API y herramientas disponibles

Ahora que sabemos qué hay debajo, podemos decidir qué herramienta usar.

## Administration Portal

Es la interfaz web administrativa completa para Data Centers, Clusters, hosts, VMs, storage, redes, permisos, eventos y políticas.

```text
Administration Portal ≈ vSphere Client conectado a vCenter
```

## VM Portal

Está orientado al usuario final. Lo que ve y puede hacer depende de sus permisos: operar sus VMs, acceder a consola o realizar autoservicio autorizado.

No es un portal “simplificado para administradores”. Resuelve otro caso: delegar sin entregar toda la infraestructura.

## Monitoring Portal

Abre los dashboards integrados de Grafana para consultar inventario, tendencias y métricas históricas.

```text
Administration Portal → administrar
VM Portal             → autoservicio
Monitoring Portal     → observar
```

## Consolas

La consola permite interactuar con una VM aunque su red invitada no funcione. No es lo mismo que conectarse por SSH o RDP a la IP del invitado.

Pensad en VMRC o web console de vSphere: acceso de consola y conectividad de aplicación son rutas diferentes.

## API REST y automatización

OLVM ofrece una API REST para inventario, operaciones masivas, integración con CMDB o portales, pipelines y comprobaciones repetibles.

```text
Administrador → Portal ─┐
                        ├──> Engine
Automatización → REST ───┘
```

Alrededor de la API existen SDKs y herramientas como colecciones Ansible del ecosistema oVirt. Antes de usarlas verificaremos compatibilidad con nuestra versión y soporte de Oracle.

## Herramientas que aparecerán durante el curso

| Herramienta | Propósito | Día |
|---|---|---:|
| `engine-setup` | Configurar o reconfigurar el Engine | 4 |
| `hosted-engine` | Desplegar y operar Self-Hosted Engine | 4 |
| `engine-backup` | Backup y restore del Manager | 5 |
| `engine-config` | Consultar/cambiar parámetros del Engine | Según necesidad |
| `ovirt-log-collector` | Recopilar información diagnóstica | 5 |
| PostgreSQL tools | Revisar actividad y estado de bases | 5 |

En los hosts utilizaremos `systemctl`, `journalctl`, logs de VDSM/libvirt/QEMU, herramientas de red y storage y, para observación local controlada, `virsh`.

Que `virsh` pueda cambiar una VM administrada por OLVM no significa que debamos hacerlo. Modificar por debajo del Engine puede crear discrepancias.

## Evento, log, métrica y dashboard

- **Evento:** algo significativo ocurrido en la plataforma.
- **Log:** detalle técnico generado por un componente.
- **Métrica:** valor medido a lo largo del tiempo.
- **Dashboard:** representación visual de datos.

La secuencia de diagnóstico será:

```text
Síntoma → objeto/capa → evento → log → métricas
```

No empezaremos reiniciando servicios al azar. Reiniciar cambia el escenario y puede borrar la evidencia que necesitamos.

## Mapa OLVM frente a vSphere

| OLVM | vSphere aproximado | Límite |
|---|---|---|
| Engine | vCenter Server | Arquitectura diferente |
| Administration Portal | vSphere Client | Modelo y menús distintos |
| Host KVM | ESXi | OLVM expone el stack Linux/KVM |
| Data Center | vSphere Datacenter | Ámbitos diferentes |
| Cluster | vSphere Cluster | Políticas propias |
| Storage Domain | Datastore | Formatos y metadatos distintos |
| Logical Network | Port Group | No son el mismo objeto |
| Bond | NIC teaming | Modos diferentes |
| Live migration | vMotion | Implementación propia |
| VM Portal | Portal de autoservicio | Sin equivalente único |
| VDSM | Agente de host, aproximadamente | Sin pareja uno a uno |
| SPM | Sin equivalente directo | Concepto propio de OLVM/oVirt |
| Self-Hosted Engine | vCenter como VM, aproximadamente | Hosted-engine es específico |

Usaremos la columna central para aprender rápido y la última para no aprender mal.

## Caso final: arrancar una VM

1. El usuario solicita el arranque.
2. El Engine valida permisos, estado y recursos.
3. Selecciona un host del Cluster.
4. VDSM ejecuta la operación mediante libvirt.
5. QEMU/KVM inicia la VM.
6. El host conecta discos y red.
7. VDSM devuelve estado.
8. El Engine registra eventos y actualiza los portales.
9. DWH conserva histórico para observabilidad.

Preguntas:

- ¿El I/O normal atraviesa el Engine? **No.**
- ¿El SPM ejecuta todas las VMs? **No.**
- ¿Cerrar el navegador detiene la operación? **No; el portal es un cliente.**
- ¿Grafana arranca VMs? **No; visualiza información.**

---

# Resumen del día

Si mañana sólo recordáis estas ideas, que sean estas:

1. **OLVM administra; KVM ejecuta.**
2. Engine ≈ vCenter y Host KVM ≈ ESXi, como analogías.
3. Engine → VDSM → libvirt → QEMU/KVM es la cadena de control.
4. El Engine no transporta el I/O normal de las VMs.
5. Data Center, Cluster y Host son niveles diferentes.
6. SPM coordina operaciones de storage; no sirve todo el tráfico de disco.
7. Storage Domain ≈ Datastore, sin identidad técnica.
8. Logical Network no es VLAN.
9. Logical Network ≈ Port Group, con diferencias.
10. NIC, bond, VLAN, bridge, perfil y vNIC son capas distintas.
11. Administration, VM y Monitoring Portal tienen usuarios y propósitos distintos.
12. Antes de buscar un botón o un log, identificamos el objeto y la capa.

Mañana llevaremos storage y networking a la plataforma. Cada objeto del portal ya tendrá un sitio en el mapa.

---

# Chuleta del día 1

```text
CONTROL
Portal/API → Engine → VDSM → libvirt

EJECUCIÓN
QEMU + KVM → VM

JERARQUÍA
Engine → Data Center → Cluster → Host → VM

STORAGE
Backend → protocolo → Storage Domain → Virtual Disk → filesystem invitado

NETWORKING
NIC → bond → VLAN → bridge → Logical Network/perfil → vNIC

HISTÓRICO
Engine DB → DWH → ovirt_engine_history → Grafana

ANALOGÍAS
Engine ≈ vCenter
Host KVM ≈ ESXi
Data Center ≈ vSphere Datacenter
Cluster ≈ vSphere Cluster
Storage Domain ≈ Datastore
Logical Network ≈ Port Group
SPM = sin equivalente directo
```
