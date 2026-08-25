# Día 2 · Storage Domains y networking nativo en OLVM

---

# Qué vamos a conseguir hoy

Ayer construimos el mapa. Hoy vamos a buscar cada pieza en una instalación real.

No vamos a instalar el Engine ni nuevos hosts. Tampoco vamos a entrar todavía en redes SDN. Trabajaremos con los dos caminos fundamentales que debemos dominar antes de añadir capas:

```text
STORAGE
Backend → protocolo → Storage Domain → Virtual Disk → VM

NETWORKING NATIVO
Logical Network → VDSM/nmstate → bridge/VLAN/bond → NIC física
```

Al terminar el día quiero que podáis:

1. Seguir un disco desde la VM hasta su backend.
2. Distinguir almacenamiento de ficheros, bloques y local.
3. Explicar qué coordina el SPM y qué no pasa por él.
4. Reconocer los estados principales de un Storage Domain.
5. Seguir una vNIC hasta el bridge y la NIC física del host.
6. Diferenciar Logical Network, vNIC Profile, VLAN, bridge y bond.
7. Utilizar el portal y herramientas Linux para comprobar la configuración.
8. Localizar la capa probable de un problema antes de modificar nada.

La regla del día será:

> **Primero dibujamos el camino; después buscamos el punto donde se rompe.**

---

# Reparto de la jornada

| Bloque | Duración | Contenido |
|---|---:|---|
| 1 | 30 min | Reconocer la instalación y traducir el mapa del día 1 al portal |
| 2 | 70 min | Storage Domains en profundidad |
| Pausa | 15 min | |
| 3 | 50 min | Laboratorio de almacenamiento |
| 4 | 55 min | Networking nativo: NIC, bond, VLAN, bridge y Logical Network |
| Pausa | 15 min | |
| 5 | 45 min | vNIC Profiles, MAC, SR-IOV y laboratorio de red |
| 6 | 20 min | Caso integrado y troubleshooting |

Total: **5 horas**, incluidas dos pausas de 15 minutos.

---

# Punto de partida: qué sabemos de nuestros hosts

La pantalla de resumen muestra, entre otros datos:

- Oracle Linux Server 9.8.
- Kernel UEK 6.12.
- KVM/QEMU 7.2.
- libvirt 9.0.
- VDSM 4.50.5.
- `librbd` de Ceph disponible como componente.
- Open vSwitch 2.17 instalado como componente.
- nmstate 2.2 instalado.
- Un proveedor SDN no configurado.

La pantalla nos dice qué componentes están instalados o detectados, no necesariamente qué componente está siendo utilizado. Hoy no abriremos esa rama. Comprobaremos únicamente el camino de red nativo:

```bash
ip link show type bridge
bridge link
ip -br link
ip -br address
```

Estos comandos existen tanto en los nodos exteriores como en los hosts OLVM, pero la salida representa capas distintas. Antes de interpretar una interfaz comprobaremos siempre en qué máquina estamos.

Hoy nos quedamos con:

```text
Engine → VDSM → nmstate/NetworkManager → red Linux del host
```

Las alternativas SDN se estudiarán otro día.

---

# Bloque 1 · Reconocer la instalación

## Antes de tocar nada

Entramos al Administration Portal y recorremos el entorno en modo observación.

No vamos a crear todavía una red, poner un host en mantenimiento ni desactivar un dominio. Primero necesitamos saber qué sostiene el laboratorio.

## Recorrido por la jerarquía

### Data Center

Localizamos:

- nombre y estado;
- versión de compatibilidad;
- Clusters asociados;
- Storage Domains;
- Logical Networks.

Pregunta: ¿qué le faltaría para permanecer `Uninitialized`? Un Data Center necesita un Cluster, al menos un host activo y un Data Storage Domain activo.

### Cluster

Revisamos:

- hosts que pertenecen al Cluster;
- arquitectura y tipo de CPU;
- redes asignadas;
- política de scheduling;
- proveedor de red predeterminado, sin desarrollarlo todavía.

### Hosts

Para cada host anotamos:

- estado;
- SPM, si ejerce el rol;
- interfaces de red;
- VMs en ejecución;
- versión de VDSM, libvirt y KVM;
- acceso a Storage Domains.

### Storage Domains

Anotamos:

- tipo y estado;
- Data Center asociado;
- backend y dirección;
- capacidad total, usada y disponible;
- contenido visible.

## Nuestra instalación real

En la pantalla actual aparecen tres objetos:

| Nombre | Tipo de dominio | Backend | Formato | Estado | Capacidad mostrada |
|---|---|---|---|---|---:|
| `curso-nfs` | Datos (Maestro) | NFS | V5 | Activo | 3666 GiB totales / 966 GiB usados / 2700 GiB libres |
| `curso-nfs-2` | Datos | NFS | V5 | Activo | 3666 GiB totales / 966 GiB usados / 2700 GiB libres |
| `ovirt-image-repository` | Imagen | OpenStack Glance | V1 | Separado | No disponible |

Esta pantalla nos proporciona varios ejercicios reales.

### ¿Qué significa “Datos (Maestro)”?

`curso-nfs` es un Data Domain y, además, ejerce el rol de dominio maestro dentro del Data Center. Es una función de coordinación y metadatos del pool de almacenamiento.

No confundimos tres conceptos:

```text
Data Domain maestro  → rol de un dominio de almacenamiento
SPM                  → rol ejercido por un host
Host de una VM       → host que realiza la E/S de esa VM
```

Que `curso-nfs` sea Maestro no significa que todos los discos tengan que residir en él ni que toda la E/S de `curso-nfs-2` pase a través suyo.

### Las capacidades idénticas son una pista, no una conclusión

Los dos dominios NFS muestran exactamente:

```text
Total: 3666 GiB
Usado:  966 GiB
Libre: 2700 GiB
```

Podrían ser exports respaldados por el mismo filesystem del servidor NFS. También podría existir otra explicación. Antes de sumar y afirmar que tenemos 7332 GiB, abriremos los detalles y compararemos:

- servidor;
- ruta del export;
- filesystem real que respalda cada ruta;
- cabina/volumen subyacente;
- dependencia de red y servidor.

> **Dos Storage Domains no implican automáticamente dos sistemas de almacenamiento independientes.**

Si ambos exports dependen del mismo filesystem, el espacio libre mostrado en los dos puede representar el mismo conjunto de bloques. En ese caso sumar las columnas duplicaría la capacidad aparente.

### ¿Y `ovirt-image-repository`?

Es un dominio/proveedor de imágenes basado en OpenStack Glance y aparece separado. No es uno de nuestros Data Domains NFS activos y no sostiene, por ese solo hecho, los discos de las VMs que estamos estudiando.

Nos sirve para preguntar:

- ¿es Data Domain o Image Domain? **Image Domain.**
- ¿está activo en el Data Center? **No, aparece separado.**
- ¿su estado separado vuelve inactivos a `curso-nfs` y `curso-nfs-2`? **No.**

Hoy lo identificaremos y seguiremos con los dos dominios de datos NFS.

### Logical Networks

Anotamos:

- si son VM Network;
- si tienen VLAN ID;
- Clusters donde se usan;
- función: management, migration, display u otra;
- perfiles vNIC asociados.

## Resultado del bloque

Al terminar tendremos un dibujo de **la instalación real**, no de una instalación imaginaria:

```text
Data Center
 ├── Cluster
 │    ├── Host 1
 │    └── Host 2
 ├── Storage Domain A
 ├── Storage Domain B
 ├── Logical Network management
 └── Logical Network de VMs
```

---

# Bloque 2 · Storage Domains en profundidad

## Empecemos por eliminar una confusión

Estas cinco cosas no son lo mismo:

```text
Discos físicos/LUNs
        │
Protocolo de acceso
        │
Storage Domain
        │
Virtual Disk
        │
Filesystem de la VM
```

Un fallo en XFS dentro de la VM no convierte automáticamente al Storage Domain en defectuoso. Y un Storage Domain inactivo no se arregla haciendo `fsck` dentro del invitado.

## Qué es un Storage Domain

Es una colección de imágenes que comparte una interfaz de almacenamiento. Puede contener:

- Virtual Disks;
- snapshots;
- templates;
- imágenes ISO;
- representaciones OVF y metadatos.

En OLVM moderno hablamos principalmente de **Data Domains**. Las opciones históricas de ISO Domain y Export Domain aparecen todavía en algunos lugares, pero están deprecadas. Las ISO pueden subirse a un Data Domain.

```text
Storage Domain ≈ Datastore
```

La analogía con vSphere sirve para ubicarlo, pero no para deducir formatos o procedimientos.

## Almacenamiento de ficheros

### NFS

El host monta un filesystem exportado por un servidor NFS.

```text
Servidor NFS
   └── export
        └── host KVM
             └── Storage Domain
                  └── imágenes de disco como ficheros
```

Ventajas conceptuales:

- sencillo de entender y desplegar;
- varios hosts pueden montar el mismo export;
- las imágenes se observan como ficheros.

Problemas habituales:

- DNS o conectividad;
- export incorrecto;
- permisos y ownership;
- versión/opciones NFS;
- firewall;
- latencia o saturación;
- servidor NFS como punto de fallo si no está protegido.

### GlusterFS

También presenta almacenamiento basado en ficheros, pero distribuido entre nodos. No asumiremos que “NFS distribuido” y GlusterFS son equivalentes. Tiene arquitectura, operación y restricciones propias.

Su compatibilidad depende mucho de versión y sistema operativo del host. Lo desarrollaremos únicamente si forma parte de la instalación o del objetivo práctico.

## Almacenamiento de bloques

### iSCSI

iSCSI transporta comandos SCSI sobre una red IP.

```text
iSCSI Target
    └── LUN
         └── sesión iSCSI en los hosts
              └── dispositivo de bloques
                   └── estructuras de OLVM
                        └── Virtual Disks
```

Elementos que debemos distinguir:

- **Initiator:** cliente iSCSI, en cada host.
- **Target:** servicio que publica almacenamiento.
- **Portal:** IP y puerto por el que descubrimos el target.
- **IQN:** identificador iSCSI.
- **LUN:** unidad de bloques presentada.
- **Multipath:** varios caminos hacia la misma LUN.

OLVM crea sus estructuras sobre las LUNs seleccionadas. No debemos formatearlas manualmente como si fueran discos normales del host.

### Fibre Channel

También presenta LUNs de bloques, pero utiliza una fabric FC y HBAs en lugar de transportar SCSI sobre TCP/IP.

Desde el punto de vista de OLVM, iSCSI y FCP comparten la idea de almacenamiento de bloques, aunque el descubrimiento y la infraestructura sean distintos.

## Local storage

Local storage está ligado a un host. OLVM coloca ese host en un Cluster de un único host porque no existe almacenamiento compartido accesible por el resto.

Oracle documenta que con local storage no están disponibles funciones como:

- live migration;
- scheduling entre hosts;
- fencing dentro de ese modelo de Cluster compartido.

No significa que local storage sea inútil. Significa que priorizamos sencillez, coste o rendimiento local frente a movilidad y HA proporcionadas por OLVM.

## Virtual Disks

El Storage Domain no se conecta directamente al sistema operativo invitado. OLVM crea un Virtual Disk y lo presenta mediante un controlador virtual.

Podemos encontrar, según backend y configuración:

- discos RAW;
- discos QCOW2;
- preallocated;
- thin provisioned;
- direct LUN;
- discos shareable bajo restricciones.

No vamos a memorizar combinaciones sin contexto. La decisión se basa en rendimiento, espacio, snapshots y funcionalidad requerida.

## SPM: coordinación, no camino de datos

El Engine asigna a un host el rol SPM.

El SPM coordina operaciones de metadatos y participa en la creación, eliminación o manipulación de discos, snapshots y templates.

```text
CONTROL
Engine → SPM → metadatos/operación administrativa

DATOS
VM → host donde corre → Storage Domain
```

Si el SPM cambia, las VMs no cambian automáticamente de host y su tráfico normal no empieza a atravesar el nuevo SPM.

## Estados y transiciones

Los nombres exactos visibles pueden variar ligeramente por pantalla, pero debemos comprender el ciclo:

```text
Storage descubierto/creado
        │
asociado al Data Center
        │
activado
        │
disponible para operaciones
```

Para desasociar o realizar determinadas tareas administrativas suele ser necesario:

```text
Active → Maintenance/Inactive → Detach/Remove
```

No se fuerza una transición sin comprobar dependencias. Un dominio puede contener discos de VMs, templates, leases o tareas activas.

## Comparación con vSphere

| OLVM | vSphere aproximado | Diferencia importante |
|---|---|---|
| Storage Domain | Datastore | Gestión interna y metadatos distintos |
| NFS Domain | NFS Datastore | Integración propia de cada plataforma |
| iSCSI/FCP LUN | LUN presentada a ESXi | OLVM construye sus estructuras sobre ella |
| Virtual Disk | VMDK como concepto | Formatos y operaciones diferentes |
| SPM | Sin equivalente directo | Es un rol propio de OLVM/oVirt |

---

# Bloque 3 · Laboratorio de almacenamiento NFS

## Objetivo del laboratorio

Nuestra instalación utiliza NFS. Eso nos permite seguir el camino completo sin cambiar de tecnología a mitad de explicación:

```text
VM
 └── Virtual Disk
      └── imagen almacenada en un Data Domain
           └── montaje NFS del host
                └── export del servidor NFS
```

No quiero que al terminar sepáis solamente que “hay un NFS”. Quiero que, ante un problema, podáis responder cuatro preguntas:

1. ¿Qué Storage Domain contiene el disco?
2. ¿Qué export y qué servidor lo respaldan?
3. ¿Qué hosts pueden montar ese export?
4. ¿Falla OLVM, el montaje NFS, la red o el propio servidor?

## Normas de seguridad

Durante la primera parte del laboratorio todo es de solo lectura.

No haremos ninguna de estas operaciones sobre dominios que sostengan VMs importantes:

- desactivar o separar un Storage Domain;
- desmontar manualmente el NFS;
- borrar ficheros dentro del export;
- cambiar permisos “a ver si funciona”;
- poner un host en mantenimiento;
- reiniciar VDSM, NetworkManager o el servidor NFS;
- editar directamente los metadatos del dominio.

> **Dentro del export, los nombres y directorios pertenecen a OLVM. No administramos un Storage Domain moviendo sus ficheros a mano.**

## Práctica 1 · Inventario desde el portal

Abrimos **Storage → Domains** y elegimos el Data Domain NFS.

Cada alumno debe localizar y anotar:

- nombre del dominio;
- estado;
- tipo de dominio;
- formato o versión, si aparece;
- Data Center al que está asociado;
- servidor NFS;
- ruta exportada;
- espacio total, usado y disponible;
- hosts con acceso;
- discos almacenados.

### Preguntas durante la demostración

- ¿El nombre del Storage Domain tiene por qué coincidir con el nombre del export?
- ¿Puede el mismo export utilizarse como directorio general para guardar cualquier fichero?
- ¿Qué se perdería si solo uno de los hosts pudiera alcanzar el servidor NFS?
- ¿Tener el dominio en estado `Active` demuestra que todas las operaciones son rápidas?
- ¿Espacio libre y rendimiento son la misma métrica?

La última pregunta es importante. Un NFS puede tener mucho espacio libre y sufrir latencia, timeouts o saturación.

## Práctica 2 · Seguir el disco de una VM

Elegimos una VM de laboratorio que podamos inspeccionar sin riesgo.

En el portal seguimos:

```text
VM → Disks → disco → Storage Domain
```

Anotamos:

- alias del disco;
- tamaño virtual;
- tamaño real, si la interfaz lo muestra;
- formato;
- política de asignación;
- interfaz virtual del disco;
- dominio donde reside;
- existencia de snapshots.

### Lo que estamos demostrando

El tamaño que ve la VM, el consumo real del backend y el espacio disponible del dominio son medidas diferentes.

Ejemplo:

```text
Disco virtual presentado a la VM: 100 GiB
Bloques realmente consumidos:      18 GiB
Espacio libre del Storage Domain:  420 GiB
```

No debemos afirmar que todos los discos thin ocuparán siempre poco. Crecen con las escrituras, y los snapshots también consumen capacidad.

## Práctica 3 · Ver el NFS desde un host

Entramos por SSH en uno de los hosts y observamos. No modificamos.

### Montajes NFS

```bash
findmnt -t nfs,nfs4
df -hT
nfsstat -m
```

Qué buscamos:

- servidor y export;
- punto de montaje;
- NFSv3 o NFSv4;
- opciones de montaje;
- capacidad visible;
- coherencia con lo mostrado en el portal.

Los puntos de montaje usados por OLVM pueden estar dentro de directorios gestionados por VDSM. No hay que “corregir” su nombre ni remontarlos manualmente.

### Resolución y conectividad

Si el servidor está configurado por nombre:

```bash
getent hosts nombre-del-servidor-nfs
```

Para comprobar el servicio NFS sin iniciar una escritura:

```bash
rpcinfo -p nombre-del-servidor-nfs
showmount -e nombre-del-servidor-nfs
```

`showmount` es útil en muchos entornos NFSv3, pero puede no mostrar lo esperado con determinadas configuraciones NFSv4. Que un comando de descubrimiento no devuelva exports no prueba por sí solo que el montaje NFSv4 esté roto.

### Procesos y registros relacionados con OLVM

```bash
systemctl status vdsmd
journalctl -u vdsmd --since "30 minutes ago"
```

Si necesitamos el detalle específico de VDSM, consultaremos sus logs, habitualmente bajo `/var/log/vdsm/`, sin editar ni truncar nada.

## El asunto de los permisos NFS

El host necesita acceder al export con la identidad y permisos esperados por OLVM/VDSM. En despliegues OLVM es habitual encontrar la identidad de servicio `vdsm:kvm`; los identificadores numéricos pueden aparecer como `36:36`, pero no debemos aplicar números a ciegas.

Primero comprobamos en el host:

```bash
getent passwd vdsm
getent group kvm
```

Después se valida en el servidor NFS:

- propietario y grupo del directorio exportado;
- permisos;
- exportación a las redes o hosts correctos;
- efecto de `root_squash`;
- versión y seguridad NFS;
- firewall.

> **Un `chmod 777` puede ocultar el diagnóstico y crear un problema de seguridad. No es un procedimiento de reparación.**

## Práctica 4 · Operación controlada con un disco

Esta parte solo se realiza sobre una VM de laboratorio y con un Storage Domain que tenga capacidad suficiente.

### Fase A · Crear y adjuntar

1. Abrimos la VM de laboratorio.
2. Añadimos un disco pequeño, por ejemplo 2 GiB.
3. Elegimos explícitamente el Data Domain NFS.
4. Revisamos formato, asignación e interfaz antes de confirmar.
5. Comprobamos en la pestaña de discos que queda asociado a la VM.
6. Si el hot-plug está permitido, lo conectamos; de lo contrario, seguimos el procedimiento de apagado previsto.

Antes de pulsar aceptar preguntamos:

> ¿En qué Storage Domain se va a crear? ¿Lo hemos elegido o estamos confiando en un valor predeterminado?

### Fase B · Reconocerlo dentro de Linux invitado

Dentro de la VM:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Identificamos el nuevo dispositivo por su tamaño. No suponemos que siempre será `/dev/vdb`.

Si el laboratorio permite escribir, se puede crear partición, filesystem y montaje. Ese trabajo sucede **dentro de la VM** y no convierte el filesystem invitado en un Storage Domain.

### Fase C · Relacionar las capas

Completamos esta tabla:

| Capa | Elemento observado |
|---|---|
| Aplicación/VM | Punto de montaje dentro del invitado |
| Dispositivo invitado | Disco detectado por Linux |
| OLVM | Alias e identificador del Virtual Disk |
| Storage | Data Domain NFS |
| Host | Montaje NFS gestionado por VDSM |
| Backend | Servidor y export NFS |

Si el alumno puede rellenarla sin saltos, ha comprendido el camino.

## Práctica 5 · Localizar el SPM

En **Compute → Hosts** identificamos qué host tiene el rol SPM.

Preguntas:

- ¿La VM que estamos observando tiene que ejecutarse en el SPM? **No.**
- ¿Todos los bloques de datos de todas las VMs atraviesan el SPM? **No.**
- ¿Puede cambiar el host que ejerce de SPM? **Sí.**
- ¿Por qué existe? Para coordinar operaciones y metadatos de almacenamiento que no deben ejecutarse de forma concurrente sin control.

### Frase para recordar

> **El SPM es el coordinador del dominio, no un proxy de E/S para todas las máquinas.**

## Diagnóstico guiado de NFS

Supongamos que el Storage Domain aparece `Inactive` o `Unresponsive`.

No empezamos reiniciando servicios. Seguimos capas:

```text
1. Portal: estado, eventos y alcance del fallo
2. Host: resolución DNS y ruta hacia el NFS
3. NFS: servicio y export accesibles
4. Montaje: versión, opciones y errores
5. Permisos: identidad efectiva de VDSM
6. Backend: capacidad, inodos, latencia y salud
7. VDSM/Engine: logs y tareas pendientes
```

### Pregunta 1 · ¿Falla un host o todos?

- Un solo host apunta a configuración, conectividad o montaje de ese host.
- Todos los hosts apuntan con más fuerza al servidor, export, red común o configuración del dominio.

No es una prueba absoluta, pero reduce rápidamente el espacio de búsqueda.

### Pregunta 2 · ¿Falla acceso o rendimiento?

- `Permission denied`: export, identidad, permisos o política de seguridad.
- `No route to host`: ruta, interfaz, VLAN o firewall.
- `Connection timed out`: servicio, red, firewall o servidor saturado.
- Monta pero va lento: latencia, pérdida, saturación, backend del NFS o carga.
- `No space left on device`: revisar capacidad e inodos; no siempre significa que el disco físico esté completamente lleno.

### Pregunta 3 · ¿Qué cambió?

- dirección o nombre del servidor;
- reglas de export;
- credenciales o identidad;
- firewall;
- red de storage;
- actualización del host;
- capacidad del filesystem;
- mantenimiento del servidor NFS.

Buscar el cambio reciente suele ser más productivo que probar soluciones aleatorias.

## Cierre del bloque de almacenamiento

Antes de la pausa cada alumno debe poder explicar en voz alta:

```text
“El disco X de la VM Y está en el Storage Domain Z.
Z usa el export NFS servidor:/ruta.
Los hosts lo montan y acceden directamente.
El SPM coordina operaciones del dominio, pero no transporta toda la E/S.”
```

---

# Bloque 4 · Networking nativo en OLVM

## La pregunta que organiza todo el bloque

Una VM envía una trama Ethernet. ¿Por dónde sale?

No necesitamos empezar por herramientas avanzadas. Necesitamos seguir puertos y enlaces:

```text
Aplicación de la VM
  ↓
TCP/IP del sistema invitado
  ↓
vNIC de la VM
  ↓
interfaz TAP/vnet en el host
  ↓
bridge Linux
  ↓
interfaz física, bond o subinterfaz VLAN
  ↓
switch físico
```

OLVM define y administra la intención. Linux transporta las tramas en el host.

## Nuestra red real: `alumnos`

En el Data Center `Curso` tenemos una Logical Network creada para las VMs de los alumnos.

| Propiedad | Valor observado |
|---|---|
| Nombre | `alumnos` |
| Data Center | `Curso` |
| Descripción | Red de las VMs de los alumnos - salida a la LAN |
| VM Network | Sí |
| VLAN ID | Ninguno |
| MTU | Predeterminado, 1500 |
| Proveedor externo | Ninguno mostrado |
| Nombre para VDSM | `alumnos` |

El camino que estudiaremos es, por tanto, el caso nativo más directo:

```text
VM del alumno
  → vNIC con perfil alumnos
    → bridge/red alumnos en el host
      → uplink sin etiqueta VLAN configurada por OLVM
        → LAN exterior
```

### “Sin VLAN” no significa “sin red”

La interfaz muestra `Etiqueta VLAN: ninguno`. OLVM no añade una etiqueta 802.1Q para esta Logical Network.

Esto no demuestra por sí solo cómo está segmentada toda la infraestructura exterior. Nos dice lo que hace OLVM en este punto: entregar el tráfico sin una VLAN configurada para `alumnos`.

### El mismo nombre en Data Centers distintos

La lista muestra:

- `ovirtmgmt` en el Data Center `Default`;
- `ovirtmgmt` en el Data Center `Curso`.

Aunque se llamen igual, no son un único objeto global compartido entre ambos Data Centers. El contexto forma parte de la identidad administrativa de la red.

```text
Default / ovirtmgmt ≠ Curso / ovirtmgmt
```

El Data Center `Curso` aparece **Funcionando**, con almacenamiento compartido y compatibilidad 4.7. `Default` aparece **No inicializado**. Es coherente con que el laboratorio operativo sea `Curso` y con que `Default` no tenga todavía la combinación necesaria de Cluster, host y Data Domain activos.

No borraremos `Default` para “limpiar” la pantalla: su estado nos sirve como ejemplo didáctico de Data Center no inicializado.

## Plano de control y plano de datos

### Configuración

```text
Administrador
   ↓
Engine
   ↓
VDSM
   ↓
nmstate / NetworkManager
   ↓
configuración de red del host
```

### Tráfico de la VM

```text
vNIC/TAP ↔ bridge ↔ NIC/bond/VLAN ↔ red física
```

Una vez configurada la red, cada paquete de la VM no viaja al Engine para que este decida qué hacer. El Engine no está en el camino de datos.

Comparación con vSphere:

```text
vCenter administra, pero no conmuta cada trama.
OLVM Engine administra, pero no conmuta cada trama.
```

## Las piezas, una a una

### NIC física

Es el puerto real o virtual presentado al host: `eno1`, `ens3`, `enp1s0` u otro nombre predecible.

En nuestro laboratorio los propios hosts OLVM pueden ser máquinas virtuales. En ese caso la “NIC física” que ve Oracle Linux es una vNIC proporcionada por el hipervisor exterior. Para OLVM sigue siendo el uplink del host.

### Bond

Agrupa varias NICs con un objetivo de redundancia y, según el modo, distribución de tráfico.

```text
eno1 ─┐
      ├── bond0
eno2 ─┘
```

No basta con crear el bond en OLVM. El modo elegido debe ser compatible con la configuración del switch físico. Por ejemplo, un diseño con LACP requiere coordinación con los puertos del switch.

Comparación aproximada:

```text
Linux bond ≈ NIC Teaming de ESXi
```

La equivalencia no significa que los algoritmos, nombres o procedimientos sean iguales.

### VLAN

Una VLAN separa dominios de broadcast mediante una etiqueta 802.1Q.

```text
bond0
 ├── VLAN 100 → gestión
 ├── VLAN 200 → VMs aplicación
 └── VLAN 300 → almacenamiento
```

Si OLVM configura una red con VLAN 200 sobre un uplink, el puerto del switch exterior debe transportar esa VLAN etiquetada. OLVM no reconfigura mágicamente el switch físico.

### Bridge Linux

Es un switch de capa 2 implementado por el kernel Linux.

Sus puertos pueden ser:

- una interfaz física;
- un bond;
- una interfaz VLAN;
- interfaces TAP asociadas a vNICs de VMs.

El bridge aprende direcciones MAC y construye una tabla de reenvío, del mismo modo conceptual que un switch.

```text
                  bridge lógico en el host
             ┌────────────────────────────┐
VM1 vNIC ─ TAP1                            │
VM2 vNIC ─ TAP2                            ├── uplink físico
             └────────────────────────────┘
```

### Logical Network

Es el objeto de OLVM que representa una red con una función y una configuración coherente dentro del Data Center/Cluster.

Puede definir o expresar, según el caso:

- que es una red para VMs;
- VLAN ID;
- MTU;
- si es necesaria para el Cluster;
- función de management, migration, display u otra;
- configuración IP del host cuando corresponde;
- QoS u otras políticas.

No confundimos:

```text
Logical Network  = intención y objeto administrado por OLVM
Bridge Linux     = mecanismo de capa 2 creado/configurado en el host
VLAN             = segmentación 802.1Q
```

## Red con VLAN y red sin VLAN

### Sin VLAN

```text
VM vNIC
   ↓
TAP
   ↓
bridge
   ↓
eno1
   ↓
switch: tráfico sin etiqueta
```

### Con VLAN

```text
VM vNIC
   ↓
TAP
   ↓
bridge de la Logical Network
   ↓
interfaz VLAN sobre eno1/bond0
   ↓
switch: tráfico etiquetado con la VLAN correspondiente
```

La VM normalmente no necesita saber que existe una VLAN. Ve una Ethernet normal. La etiqueta se maneja en la capa del host/uplink definida para esa Logical Network.

## ¿Dónde están las direcciones IP?

Esta distinción evita muchos errores.

### IP de la VM

Está configurada dentro del sistema operativo invitado, en su vNIC.

### IP del host para una Logical Network

Cuando el host necesita participar en esa red —por ejemplo management o migration— la configuración IP se asocia a la interfaz lógica correspondiente del host, normalmente el bridge o la interfaz resultante, no a la vNIC de una VM.

### Red exclusiva para tráfico de VMs

El host puede conmutar tráfico de VMs sin necesitar una IP propia en esa red.

```text
El bridge puede transportar tramas sin actuar como endpoint IP.
```

Pregunta para el grupo:

> Si el host no tiene IP en la red de VMs, ¿pueden las VMs comunicarse? Sí. El bridge funciona en capa 2.

## Dos VMs en el mismo host

Si ambas vNICs están conectadas al mismo bridge y pertenecen a la misma red:

```text
VM1 → TAP1 → bridge → TAP2 → VM2
```

El tráfico puede permanecer dentro del host. No necesita salir por la NIC física para volver a entrar.

## Dos VMs en hosts distintos

```text
VM1
 ↓
TAP → bridge del Host A → uplink A
 ↓
switch físico / red de capa 2
 ↓
uplink B → bridge del Host B → TAP
 ↓
VM2
```

Sin una capa adicional, la red física debe proporcionar la conectividad de capa 2 o VLAN necesaria entre ambos hosts.

Esta es la respuesta corta a la pregunta “¿cómo viaja sin un switch virtual avanzado?”:

> **Viaja como Ethernet normal: bridge Linux, uplink y switches físicos.**

## Redes necesarias y estado del host

Una Logical Network puede marcarse como requerida para un Cluster. Si un host no tiene correctamente configurada esa red, OLVM puede considerarlo no operativo o mostrar la red fuera de sincronía.

Esto no es un fallo caprichoso. OLVM intenta impedir que se ejecuten o migren VMs a un host que no dispone de la conectividad exigida.

Comparación con vSphere:

| OLVM | vSphere aproximado | Matiz |
|---|---|---|
| Logical Network | Port group, en sentido funcional | En OLVM está ligada al modelo Data Center/Cluster |
| Bridge Linux | vSwitch estándar, de forma conceptual | Implementación distinta |
| Bond | NIC Teaming | Modos y gestión distintos |
| vNIC | Adaptador de red virtual | Concepto equivalente |
| Red requerida | Consistencia exigida en hosts | La validación y estados son propios de OLVM |

## Antes de ejecutar comandos: identificar la capa

Nuestra instalación tiene tres sistemas Linux uno dentro de otro. Un mismo comando puede funcionar en los tres y mostrar cosas completamente diferentes.

| Prompt o sistema | Qué es | Quién administra su red | Qué buscamos allí |
|---|---|---|---|
| `ivan@worker4` | Nodo Ubuntu/Kubernetes y primer hipervisor | Netplan, Linux y el playbook exterior | `br-olvm`, NIC USB, macvtap y vNIC de `olvm-host2` |
| `ivan@olvm-host2` | VM Oracle Linux que OLVM trata como host KVM | Engine → VDSM → nmstate/NetworkManager | `ovirtmgmt`, `alumnos`, `eth0`, `eth1` y TAPs de VMs |
| `alumno@vm-alumnoN` | VM anidada del alumno | Sistema operativo invitado | vNIC, IP, gateway, rutas y DNS del invitado |

Regla para toda la práctica:

> **Antes de interpretar una interfaz, leer el prompt. `worker4` y `olvm-host2` no son el mismo host.**

## Por qué `nmstatectl show` no funciona en `worker4`

La pantalla de OLVM muestra una versión de nmstate para `olvm-host2`, porque ese es el host Oracle Linux incorporado al Engine.

No demuestra que `worker4`, que es el Ubuntu exterior, tenga instalado el programa `nmstatectl`. La red exterior se creó con Netplan y comandos de Linux.

```text
worker4       → Netplan + ip + bridge + virsh
olvm-host2    → VDSM + nmstate/NetworkManager + ip + bridge
```

Por tanto, en `worker4` utilizaremos:

```bash
ip -br link
ip -br address
ip link show type bridge
bridge link
bridge fdb show br br-olvm
sudo virsh domiflist olvm-host2
```

Dentro de `olvm-host2` podemos comprobar primero si la utilidad existe:

```bash
command -v nmstatectl
```

Si existe, `nmstatectl show` aporta una vista declarativa. Si no existe o su versión no ofrece ese subcomando, la práctica continúa con:

```bash
ip -br link
ip -br address
bridge link
nmcli device status
nmcli connection show
```

El aprendizaje del día no dependerá de `nmstatectl`.

---

## Cómo leer `ip -br link`

El comando muestra una vista abreviada del estado de enlace:

```text
NOMBRE       ESTADO      MAC                    FLAGS
```

### Nombre y relaciones indicadas con `@`

El primer campo identifica la interfaz. Cuando aparece `@`, expresa una relación:

- `macvtap3@enp1s0f0`: `macvtap3` está construido directamente sobre `enp1s0f0`;
- `cali...@if2`: extremo del host de una pareja virtual cuyo otro extremo tiene índice 2 dentro de otro namespace, normalmente un pod.

### Estado `UP` o `UNKNOWN`

- `UP`: la interfaz dispone de una noción de carrier/enlace y está operativa.
- `UNKNOWN`: el kernel no puede obtener un estado físico tradicional. Es normal en loopback, bridges, VXLAN y TAPs.

`UNKNOWN` no significa automáticamente “averiada”. Para una interfaz virtual comprobamos también los flags `UP` y `LOWER_UP` y su pertenencia al bridge.

### Dirección MAC

Es la dirección de capa 2 de esa interfaz en el sistema donde ejecutamos el comando. En una vNIC virtual, la MAC del lado host puede no coincidir exactamente con la MAC presentada al invitado.

### Flags principales

- `UP`: administrativamente habilitada;
- `LOWER_UP`: la capa inferior está disponible;
- `BROADCAST`: admite broadcast;
- `MULTICAST`: admite multicast;
- `PROMISC`: puede recibir tramas que no van dirigidas a su propia MAC;
- `ALLMULTI`: acepta todo el tráfico multicast.

## Interpretación de la salida real de `worker4`

### `lo`

```text
lo  UNKNOWN  00:00:00:00:00:00  <LOOPBACK,UP,LOWER_UP>
```

Es el loopback del propio `worker4`. No representa ninguna red de OLVM.

### `enp1s0f0`

```text
enp1s0f0  UP  0c:4d:e9:9c:00:0b
```

Es la NIC principal de `worker4`. Tiene la IP del nodo, transporta tráfico de Kubernetes y sirve como dispositivo inferior del macvtap de la primera NIC de `olvm-host2`.

No pertenece a `br-olvm`.

### `enxf8e43b59dd23`

```text
enxf8e43b59dd23  UP  f8:e4:3b:59:dd:23
```

Es la segunda NIC física/USB de `worker4`. El nombre incorpora su MAC porque Linux la identifica por el adaptador USB.

No tiene IP propia. Es un puerto de capa 2 de `br-olvm` y proporciona la salida física de las VMs de alumnos.

### Interfaces de Calico

```text
vxlan.calico
cali68f528877f6@if2
calib90cf7d423d@if2
...
```

Pertenecen a Kubernetes:

- `vxlan.calico` forma parte de la conectividad entre nodos y pods;
- cada `cali...` suele ser el extremo del host conectado al namespace de un pod.

No forman parte del camino OLVM de la red `alumnos`. Tampoco deben borrarse o modificarse durante la práctica.

El playbook crea `br-olvm` manualmente precisamente para no aplicar de nuevo toda la configuración de Netplan y alterar las rutas que mantiene Calico.

### `macvtap3@enp1s0f0`

```text
macvtap3@enp1s0f0  UP  52:54:00:b3:ed:e9
```

Es el dispositivo exterior conectado a la primera vNIC de `olvm-host2`:

```text
macvtap3 → direct → enp1s0f0
```

Dentro de `olvm-host2`, la vNIC con MAC `52:54:00:b3:ed:e9` debería corresponder a `eth0`, que soporta la red de gestión `ovirtmgmt` y la IP del host OLVM.

Los flags `ALLMULTI` y `PROMISC` son coherentes con una interfaz virtual preparada para recibir más tráfico que el dirigido estrictamente a su propia MAC. No convierten macvtap en un bridge Linux tradicional ni eliminan todas sus limitaciones para transportar MAC anidadas.

### `br-olvm`

```text
br-olvm  UP  7a:0e:47:a0:19:99
```

Es el bridge Linux exterior creado en `worker4`.

Su MAC pertenece al propio bridge. No es la MAC de la VM de un alumno ni la de `olvm-host2`.

El bridge puede conmutar tramas sin tener una IPv4 de servicio. Que aparezca una dirección `fe80::` no cambia esa conclusión: es una IPv6 link-local creada automáticamente.

### `vnet0`

```text
vnet0  UNKNOWN  fe:54:00:31:11:21
```

Es el puerto TAP del lado `worker4` conectado a la segunda vNIC de `olvm-host2`.

Libvirt muestra para el invitado:

```text
vnet0 → bridge → br-olvm → MAC invitada 52:54:00:31:11:21
```

Mientras que `ip link` muestra en el lado host:

```text
MAC del TAP: fe:54:00:31:11:21
```

La diferencia inicial `52`/`fe` es normal en interfaces TAP creadas por libvirt. La MAC `52:54...` es la presentada a la vNIC del invitado; la `fe:54...` identifica el extremo TAP del host. No es una colisión ni un cambio accidental.

El estado `UNKNOWN` también es normal: un TAP no tiene cable ni transceptor físico. Los flags `UP` y `LOWER_UP`, junto con su estado `forwarding` en el bridge, son las comprobaciones útiles.

---

## Cómo leer `ip -br address` en `worker4`

Este comando muestra direcciones de capa 3. No dice por sí solo qué puertos forman parte de un bridge.

### Dirección principal

```text
enp1s0f0  UP  192.168.2.107/24
```

`192.168.2.107` es la IP de `worker4`, no la de `olvm-host2`.

La dirección de `olvm-host2` está dentro de la VM y no aparece como IPv4 local de `worker4`, aunque su tráfico atraviese `macvtap3` y `enp1s0f0`.

### NIC USB sin dirección

```text
enxf8e43b59dd23  UP
```

La ausencia de una dirección después de `UP` es intencionada. La interfaz actúa como puerto de `br-olvm`; no necesita comportarse como endpoint IP.

### Dirección de Calico

```text
vxlan.calico  UNKNOWN  10.10.199.128/32
```

Pertenece a Kubernetes. El `/32` identifica un endpoint de Calico, no una red OLVM ni una dirección que debamos trasladar a `alumnos`.

### Direcciones `fe80::`

Las direcciones `fe80::` de `br-olvm`, `vnet0`, `macvtap3` y otras interfaces son IPv6 link-local. Solo tienen alcance en el enlace y normalmente se crean de forma automática.

Cuando decimos que `br-olvm` “no tiene IP” queremos decir con precisión:

> **No tiene una IPv4 ni una dirección de servicio configurada para participar en la LAN como endpoint. Su función prevista es conmutar capa 2.**

---

## Cómo leer `bridge link` en `worker4`

La salida real es:

```text
3:  enxf8e43b59dd23 ... master br-olvm state forwarding priority 32 cost 5
34: vnet0            ... master br-olvm state forwarding priority 32 cost 2
```

### `3:` y `34:`

Son índices internos asignados por el kernel. Pueden cambiar al recrear una interfaz o reiniciar. No son VLAN IDs ni números de puerto físico.

### `master br-olvm`

Confirma que ambas interfaces son puertos del mismo bridge:

```text
br-olvm
 ├── enxf8e43b59dd23   puerto hacia la LAN
 └── vnet0             puerto hacia olvm-host2
```

### `state forwarding`

El bridge puede reenviar tramas por esos puertos. Es el estado que esperamos para el camino de datos activo.

### `mtu 1500`

Ambos puertos utilizan MTU 1500, coherente con la Logical Network `alumnos`, que muestra MTU predeterminado 1500.

### `priority 32` y `cost`

Son parámetros relacionados con bridge/STP. El playbook desactiva STP en `br-olvm`, por lo que los costes 5 y 2 no están seleccionando actualmente un camino alternativo.

No debemos interpretar `cost 2` como “vnet0 es dos veces más rápido”.

## Camino exacto demostrado en `worker4`

```text
TRÁFICO SALIENTE
VM alumno
  → bridge alumnos dentro de olvm-host2
    → eth1 de olvm-host2
      → vNIC invitada 52:54:00:31:11:21
        → vnet0 de worker4
          → br-olvm
            → enxf8e43b59dd23
              → LAN

TRÁFICO ENTRANTE
LAN
  → enxf8e43b59dd23
    → br-olvm
      → vnet0
        → eth1 de olvm-host2
          → bridge alumnos
            → TAP/vNIC de la VM destinataria
```

`worker4` solo puede mostrarnos hasta `vnet0`. Para ver las TAP de `vm-alumnoN` debemos entrar en `olvm-host2` y ejecutar allí `bridge link` o `bridge fdb show`.

## Comprobaciones adicionales de solo lectura en `worker4`

### Confirmar a qué VM pertenece `vnet0`

```bash
sudo virsh domiflist olvm-host2
```

La salida real confirma:

```text
macvtap3  direct  enp1s0f0  virtio  52:54:00:b3:ed:e9
vnet0     bridge  br-olvm   virtio  52:54:00:31:11:21
```

### Ver las MAC aprendidas

```bash
sudo bridge fdb show br br-olvm
```

Después de generar tráfico desde una VM de alumno esperamos aprender:

- las MAC de las VMs detrás de `vnet0`;
- las MAC de gateway y dispositivos de la LAN detrás de `enxf8e43b59dd23`.

La FDB no es una tabla ARP, no muestra IPs y sus entradas dinámicas pueden caducar.

---

## Demostración dentro de `olvm-host2`

Ahora entramos realmente en el host administrado por OLVM:

```bash
hostname
ip -br link
ip -br address
ip link show type bridge
bridge link
bridge fdb show
nmcli device status
nmcli connection show
```

El primer `hostname` evita analizar por error la capa exterior.

Buscamos las MAC conocidas:

```text
52:54:00:b3:ed:e9 → primera vNIC, previsiblemente eth0 / ovirtmgmt
52:54:00:31:11:21 → segunda vNIC, previsiblemente eth1 / alumnos
```

Dentro de este host esperamos encontrar otro esquema:

```text
olvm-host2
 ├── bridge ovirtmgmt
 │    └── eth0 → macvtap3 exterior → enp1s0f0
 │
 └── bridge alumnos
      ├── eth1 → vnet0 exterior → br-olvm → NIC USB
      ├── TAP de vm-alumnoX
      └── TAP de vm-alumnoY
```

Este es el segundo nivel de bridges. El `br-olvm` de `worker4` y el bridge `alumnos` de `olvm-host2` no son el mismo objeto.

No editaremos ni activaremos conexiones con `nmcli`. OLVM debe seguir siendo la fuente de verdad para las redes del host interior:

> **Si cambiamos manualmente una red administrada por OLVM, podemos crear una discrepancia entre el estado esperado por el Engine y el estado real del host.**

## Mini ejercicio: dibujar antes de buscar

Elegimos una Logical Network real y completamos:

```text
Nombre en OLVM:        ____________________
VM Network:            sí / no
VLAN ID:               ____________________
vNIC Profile:          ____________________
Bridge del host:       ____________________
Uplink/bond:           ____________________
Puerto/switch físico:  ____________________
IP del host:           ____________________
```

Si no hay IP del host, no escribimos “error”. Primero preguntamos si el host necesita ser endpoint IP en esa red.

---

# Bloque 5 · vNIC Profiles, MAC, SR-IOV y laboratorio de red

## Tres objetos que no debemos mezclar

```text
Logical Network
      ↓ usa
vNIC Profile
      ↓ selecciona
vNIC concreta de una VM
```

Una forma rápida de explicarlo:

- **Logical Network:** a qué red queremos conectar.
- **vNIC Profile:** con qué reglas o modalidad se permite esa conexión.
- **vNIC:** el adaptador concreto de una VM.

Comparación aproximada con vSphere:

```text
Logical Network + vNIC Profile ≈ parte de lo que solemos asociar a un port group
```

No es una correspondencia uno a uno. En OLVM el perfil es un objeto explícito y seleccionable por la vNIC.

## Qué puede aportar un vNIC Profile

Dependiendo de la versión y configuración, puede asociar:

- una Logical Network;
- QoS de red;
- port mirroring;
- filtros de red;
- passthrough;
- características relacionadas con SR-IOV;
- permisos de uso.

Por eso dos VMs en la misma Logical Network pueden usar perfiles distintos.

Ejemplo:

```text
Logical Network: produccion
 ├── Profile: produccion-estandar
 ├── Profile: produccion-limitado
 └── Profile: produccion-sriov
```

La red es la misma; cambia la forma de conexión o la política aplicada.

## Nuestro perfil real: `alumnos`

La Logical Network `alumnos` tiene actualmente un perfil vNIC también llamado `alumnos`.

| Propiedad | Valor observado |
|---|---|
| Perfil | `alumnos` |
| Logical Network | `alumnos` |
| Data Center | `Curso` |
| Compatibilidad | 4.7 |
| QoS | Ilimitado / sin política seleccionada |
| Filtro de red | `vdsm-no-mac-spoofing` |
| Passthrough/transferencia | No activado |
| Migrable | Sí, según la modalidad mostrada |
| Perfil de conmutación por error | Ninguno |
| Port mirroring | No activado |

Las capturas muestran **14 VMs** asociadas a este perfil, incluidas las imágenes base y `vm-alumno1` a `vm-alumno10`.

Esto convierte el perfil en un objeto con dependencias reales:

> **No modificamos ni borramos `alumnos` como si fuese un objeto vacío. Primero revisamos qué VMs y templates lo consumen.**

### El filtro `vdsm-no-mac-spoofing`

Este filtro ayuda a impedir que una VM emita tráfico usando direcciones MAC de origen distintas de las autorizadas para su vNIC.

Es una protección razonable para VMs normales, pero debemos recordarla si una VM va a realizar funciones especiales, por ejemplo:

- virtualización anidada con otras MAC detrás;
- bridging dentro del invitado;
- determinadas configuraciones de alta disponibilidad con MAC virtual;
- herramientas que intenten modificar la MAC de origen.

Si ese tipo de VM no comunica, no desactivamos el filtro globalmente por reflejo. Confirmamos primero el requisito y creamos un perfil específico con el alcance mínimo necesario.

### La combinación actual

En el laboratorio, el objeto completo queda así:

```text
Data Center Curso
  └── Logical Network alumnos
       ├── VM Network: sí
       ├── VLAN: ninguna
       ├── MTU: 1500
       └── vNIC Profile alumnos
            ├── QoS: ilimitado
            ├── filtro: vdsm-no-mac-spoofing
            ├── mirroring: no
            └── 14 VMs asociadas
```

## MAC address y MAC Pool

Cada vNIC necesita una dirección MAC única en el dominio de capa 2.

OLVM puede asignarla desde un MAC Pool. El pool:

- define rangos administrados;
- reduce colisiones;
- permite automatizar la creación de vNICs;
- puede controlar si se permiten MAC duplicadas, según configuración.

No debemos escribir una MAC manual sin conocer el alcance de la red. Una colisión puede producir conectividad intermitente y tablas MAC inestables, un problema especialmente desagradable de diagnosticar.

## SR-IOV: solo el concepto necesario hoy

SR-IOV permite que una NIC física exponga Virtual Functions que pueden asignarse de forma más directa a VMs.

```text
NIC física compatible
 ├── PF: Physical Function
 ├── VF 1 → VM1
 ├── VF 2 → VM2
 └── VF 3 → VM3
```

Ventaja principal:

- menor sobrecarga y muy buen rendimiento/latencia.

Costes y restricciones:

- hardware y drivers compatibles;
- configuración previa de PF/VF;
- capacidad finita;
- menor flexibilidad que una vNIC conectada a bridge;
- posibles restricciones para migración en vivo y otras funciones.

No lo configuraremos hoy. Lo utilizamos para entender por qué un vNIC Profile puede representar un camino distinto al bridge normal.

## Laboratorio de red · Parte 1: portal

Elegimos una VM de laboratorio encendida.

### Paso 1 · Identificar su vNIC

En la VM abrimos **Network Interfaces** y anotamos:

- nombre de la vNIC;
- modelo/tipo de interfaz;
- MAC;
- estado conectado/desconectado;
- Logical Network;
- vNIC Profile;
- información de IP reportada, si está disponible.

La IP que el portal muestra puede haber sido comunicada por el guest agent. La configuración autoritativa sigue estando dentro del invitado o en el sistema que gestione su red.

### Paso 2 · Abrir el vNIC Profile

Comprobamos:

- Logical Network asociada;
- QoS;
- filtros;
- passthrough;
- port mirroring;
- permisos, si están configurados.

Pregunta:

> ¿Cambiar de perfil significa necesariamente cambiar de red?

No. Puede cambiar la política manteniendo la misma Logical Network. También puede elegirse un perfil perteneciente a otra red, pero entonces sí cambia la conexión lógica.

### Paso 3 · Abrir la Logical Network

Comprobamos:

- Data Center y Clusters;
- VLAN ID;
- MTU;
- si es VM Network;
- si es requerida;
- funciones asignadas;
- estado en cada host.

### Paso 4 · Ver la configuración del host

En **Hosts → Network Interfaces → Setup Host Networks** observamos el diagrama de NICs, bonds y redes.

No arrastramos ni guardamos cambios. Usamos la pantalla para responder:

- ¿sobre qué NIC o bond está la red?
- ¿lleva VLAN?
- ¿tiene IP el host en esa red?
- ¿aparece sincronizada?

## Laboratorio de red · Parte 2: host

Localizamos primero el host OLVM donde corre la VM. No entramos en `worker3` o `worker4`, sino en `olvm-host1` u `olvm-host2`.

Por SSH, en el host OLVM interior:

```bash
hostname
ip -br link
ip -br address
bridge link
bridge fdb show
nmcli device status
nmcli connection show
```

El primer `hostname` evita analizar por error la capa exterior.

Intentamos reconstruir:

```text
vNIC de la VM
   ↓ MAC
TAP/vnet del host
   ↓ master
bridge
   ↓
VLAN, bond o NIC
```

### Cómo relacionar una MAC con el bridge

La MAC anotada en el portal puede buscarse en la FDB:

```bash
bridge fdb show
```

Si aparece asociada a un puerto TAP/vnet, hemos unido la vista de OLVM con el plano de datos de Linux.

No todas las entradas serán permanentes y una MAC puede no aparecer hasta que haya tráfico. Generar un ping desde la VM puede ayudar a que el bridge aprenda la dirección.

## Laboratorio de red · Parte 3: dentro de la VM

En el invitado Linux:

```bash
ip -br link
ip -br address
ip route
ip neigh
```

Relacionamos:

- MAC del invitado con la vNIC del portal;
- dirección IP con su prefijo;
- puerta de enlace con la tabla de rutas;
- vecinos con ARP/ND;
- conectividad con la red física.

Pruebas graduadas:

```text
1. Estado del enlace
2. IP y prefijo
3. Gateway configurado
4. Ping a una VM del mismo segmento
5. Ping al gateway
6. Ping a un destino remoto por IP
7. Resolución DNS
```

No empezamos por `ping google.com`: mezcla enlace, red, routing, firewall y DNS en una sola prueba.

## Práctica opcional controlada · Crear un perfil

Solo si existe una Logical Network de laboratorio y sabemos qué VM podemos modificar:

1. Creamos un vNIC Profile con un nombre explícito, por ejemplo `lab-estandar`.
2. No activamos SR-IOV, mirroring ni filtros todavía.
3. Lo asociamos a la Logical Network de laboratorio.
4. Revisamos sus permisos.
5. Creamos o modificamos una vNIC de la VM de pruebas para usarlo.
6. Comprobamos conexión desde el portal y dentro de la VM.
7. Documentamos el cambio.

No creamos una VLAN nueva si el switch exterior no está preparado. Una VLAN correcta en OLVM pero ausente en el trunk físico produce una configuración aparentemente limpia y una red completamente inútil.

## Tres averías para razonar

### Caso A · Dos VMs del mismo host se comunican, pero no con otra del segundo host

El bridge local y las vNICs probablemente funcionan. Investigamos:

- uplink del host;
- VLAN permitida;
- trunk del switch;
- bridge/red equivalente en el segundo host;
- consistencia de MTU;
- seguridad del switch o filtrado.

### Caso B · El host tiene management, pero una VM no comunica

Management y la red de la VM pueden ser objetos distintos. Investigamos:

- vNIC conectada;
- perfil correcto;
- Logical Network presente en el host;
- bridge y TAP;
- configuración IP de la VM;
- firewall del invitado.

### Caso C · Una VM llega al gateway por IP, pero no resuelve nombres

La capa 2, la IP local y el camino al gateway funcionan. Investigamos DNS, no recreamos el bridge.

---

# Bloque 6 · Caso integrado y troubleshooting

## Escenario

Tenemos dos hosts OLVM, un Data Domain NFS y una Logical Network para VMs.

Una VM arranca correctamente en el Host 1. Después de migrarla al Host 2:

- el disco sigue visible;
- el sistema operativo arranca;
- la VM puede comunicarse con otra VM del Host 2;
- no alcanza el gateway de su red.

## Razonamiento

### Lo que sabemos que funciona

- el Storage Domain NFS es accesible desde Host 2;
- el disco virtual se presenta correctamente;
- el sistema invitado arranca;
- la vNIC, su TAP y el bridge local permiten comunicación dentro de Host 2.

### Lo que no debemos culpar primero

- el SPM;
- el formato del disco;
- el servidor NFS;
- el Engine como camino de cada paquete;
- el DNS, porque todavía no alcanzamos ni el gateway por IP.

### Candidatos probables

- uplink equivocado en Host 2;
- VLAN no permitida en el puerto físico de Host 2;
- Logical Network fuera de sincronía;
- MTU o bond incoherente;
- filtrado de capa 2;
- configuración distinta entre hosts.

## Procedimiento común de diagnóstico

Usaremos siempre esta secuencia:

```text
SÍNTOMA
   ↓
OBJETO afectado
   ↓
ALCANCE: una VM, un host, una red o todo el Data Center
   ↓
ESTADO y eventos en el portal
   ↓
CAMINO lógico dibujado
   ↓
COMPROBACIÓN en cada capa
   ↓
CAMBIO mínimo y reversible
   ↓
VALIDACIÓN y documentación
```

## La prueba que divide el problema

Cuando podamos, elegimos una prueba que separe el camino en dos.

Ejemplos:

- ¿El export NFS monta en otro host?
- ¿Las VMs se comunican dentro del mismo host?
- ¿La VM llega al gateway por IP?
- ¿La MAC aparece en la FDB del bridge?
- ¿El fallo sigue a la VM o permanece en el host?

Estas pruebas aportan más información que reiniciar todos los componentes.

## Eventos y logs: orden de consulta

1. Eventos de la VM, Host o Storage Domain en el portal.
2. Tareas en curso o fallidas.
3. Estado de red/storage en el host afectado.
4. Log de VDSM para la ejecución en el host.
5. Log del Engine para la decisión u orquestación.
6. Logs del backend NFS, switch o sistema invitado según la capa.

La pregunta clave no es “¿qué log conozco?”, sino:

> **¿Qué componente tomó la decisión y cuál intentó ejecutarla?**

Si el Engine ordena y VDSM ejecuta, buscamos la decisión en Engine y el resultado local en VDSM.

## Mini casos de cierre

### 1. Storage Domain activo, VM encendida y aplicación lenta

`Active` confirma disponibilidad administrativa, no garantiza latencia baja. Medimos invitado, host, NFS, red de storage y backend.

### 2. Storage Domain inaccesible solo desde Host 2

Comparamos Host 2 con un host sano: DNS, rutas, firewall, interfaz de storage, versión/opciones de montaje y logs de VDSM.

### 3. Logical Network requerida ausente en Host 2

No forzamos la migración. Corregimos la asignación física/lógica y sincronizamos la red del host.

### 4. VM con vNIC conectada, pero sin dirección IP

“Conectada” confirma el enlace virtual, no que DHCP haya respondido ni que exista configuración estática válida. Entramos en el invitado.

### 5. Tras crear una VLAN, ninguna VM sale del host

Revisamos el trunk del switch y la VLAN permitida antes de reconstruir el bridge.

---

# Resumen para la pizarra

## Storage

```text
Backend NFS
  → export
    → montaje en cada host
      → Data Storage Domain
        → Virtual Disk
          → dispositivo dentro de la VM
```

- NFS es almacenamiento de ficheros compartido.
- Cada host necesita acceso al export.
- El SPM coordina metadatos y operaciones; no transporta toda la E/S.
- Un Storage Domain activo no equivale a rendimiento perfecto.
- No se manipulan manualmente los ficheros internos del dominio.

## Networking

```text
vNIC
  → TAP/vnet
    → bridge Linux
      → VLAN/bond/NIC
        → switch físico
```

- Logical Network define la red en OLVM.
- vNIC Profile define cómo se conecta una vNIC a esa red.
- El bridge conmuta en capa 2.
- La IP de la VM vive dentro del invitado.
- Una red de VMs no exige que el host tenga IP en ella.
- Entre hosts, la red física debe transportar la VLAN o segmento.
- Engine configura; no está en el camino de cada trama.

---

# Guion de tiempos para el instructor

Este documento contiene más material del que debemos leer literalmente. La clase se imparte alternando explicación, portal, terminal y preguntas.

## Primer tramo · 0:00–1:40

- 0:00–0:15: recuperación del mapa del día 1.
- 0:15–0:30: inventario real en el portal.
- 0:30–0:55: capas de storage y Storage Domain.
- 0:55–1:15: NFS frente a bloques y local.
- 1:15–1:30: Virtual Disks, thin/preallocated y snapshots.
- 1:30–1:40: SPM y estados.

Pregunta de control antes de la pausa:

> ¿Puede una VM del Host 2 leer su disco NFS sin pasar por el SPM?

## Pausa · 1:40–1:55

## Segundo tramo · 1:55–2:45

- 1:55–2:10: inventario del Data Domain NFS.
- 2:10–2:25: seguir un disco desde la VM.
- 2:25–2:35: observar montajes en el host.
- 2:35–2:45: diagnóstico guiado y repaso.

Si se realiza la creación del disco de laboratorio, reducir cinco minutos del inventario y cinco del diagnóstico; no correr para completar una operación destructiva.

## Tercer tramo · 2:45–3:40

- 2:45–3:00: camino de control y camino de datos.
- 3:00–3:20: NIC, bond, VLAN y bridge.
- 3:20–3:35: Logical Networks e IPs.
- 3:35–3:40: comparación con vSphere.

## Pausa · 3:40–3:55

## Cuarto tramo · 3:55–4:40

- 3:55–4:10: vNIC Profiles y MAC Pool.
- 4:10–4:15: concepto de SR-IOV.
- 4:15–4:30: seguir una vNIC del portal al host.
- 4:30–4:40: pruebas dentro de la VM.

## Cierre · 4:40–5:00

- 4:40–4:52: caso integrado de migración y red.
- 4:52–4:57: cinco preguntas rápidas.
- 4:57–5:00: resumen de rutas y anticipo del día siguiente.

---

# Cinco preguntas orales para terminar

1. ¿Qué objeto de OLVM equivale aproximadamente a un datastore?  
   **Storage Domain.**

2. ¿Por dónde accede una VM a sus datos: a través del SPM o desde su host?  
   **Desde el host donde se ejecuta; el SPM coordina.**

3. ¿Puede un bridge transportar tráfico de VMs sin tener una IP propia?  
   **Sí, porque conmuta en capa 2.**

4. ¿Qué diferencia hay entre Logical Network y vNIC Profile?  
   **La primera define la red; el segundo define cómo puede conectarse una vNIC.**

5. Dos VMs se ven en el mismo host, pero no entre hosts. ¿Qué frontera investigamos?  
   **El uplink y la red física/VLAN entre los hosts.**

---

# Lo que debe quedar claro al terminar el día 2

No necesitamos memorizar cada pantalla. Necesitamos poder recorrer dos caminos sin perdernos:

```text
DISCO
VM → Virtual Disk → Storage Domain → NFS → backend

RED
VM → vNIC → profile → Logical Network → TAP → bridge → uplink
```

Si sabemos dibujar el camino, podemos preguntar en qué capa aparece el primer resultado incorrecto. Ese es el comienzo de una administración seria de OLVM.
