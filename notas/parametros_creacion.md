# Parámetros de creación de una máquina virtual en OLVM

Guía para leer el formulario **Nueva máquina virtual** de la instalación del curso.

Las capturas utilizadas corresponden al Data Center y al Cluster `Curso`, creando una VM desde la plantilla `alma9-aula` el **26 de agosto de 2026**.

El objetivo no es memorizar todos los campos. Queremos saber qué decisión representa cada uno y distinguir:

- lo que define la identidad de la VM;
- lo que consume en los hosts;
- lo que hereda de una plantilla o política;
- lo que afecta al almacenamiento;
- lo que condiciona migración y alta disponibilidad.

---

# Lectura rápida de la VM mostrada

La selección de las capturas puede resumirse así:

```text
Cluster:              Curso
Plantilla:            alma9-aula, versión base 1
Sistema declarado:    AlmaLinux 9 x64
Máquina virtual:      Q35 + UEFI, optimizada como servidor
Memoria inicial:      1024 MiB
Memoria máxima:       4096 MiB
Memoria garantizada:  1024 MiB
CPU:                  1 vCPU
Alta disponibilidad:  desactivada
CPU pinning:          ninguno
Disco:                clone independiente, QCOW2, 10 GiB
Storage Domain:       curso-nfs
Red:                  se configura en la pestaña General
```

La idea importante es que una nueva VM no es solamente «un disco copiado»:

```text
Plantilla
  + definición de hardware virtual
  + recursos y políticas
  + disco o cadena de imágenes
  + vNIC y perfil de red
  + reglas de colocación
= nueva VM administrada por OLVM
```

---

# Elementos comunes del formulario

## Icono de información

El círculo azul con una `i` abre una ayuda breve sobre el parámetro. Es útil, pero no siempre explica sus consecuencias sobre migración, rendimiento o consumo.

## Campos deshabilitados o en gris

Un campo puede aparecer deshabilitado porque:

- depende de otra casilla;
- la plantilla o el Cluster no admiten esa combinación;
- el usuario no tiene el permiso necesario;
- solo puede cambiarse con la VM apagada;
- la opción seleccionada en otro apartado lo hace incompatible.

No debemos concluir automáticamente que el componente no esté instalado.

## Icono de cadena

La cadena pertenece al mecanismo de **tipo de instancia**.

Un tipo de instancia es un perfil de hardware, por ejemplo una combinación predefinida de memoria, vCPU y otros valores. Los campos ligados al perfil muestran la cadena.

- Cadena unida: el campo mantiene el valor definido por el tipo de instancia.
- Cadena rota: ese campo se ha personalizado y la VM deja de coincidir con el perfil.
- `Personalizado`: la VM no está vinculada a un tipo de instancia concreto.

No es una indicación de que la VM sea un linked clone ni de que dependa del template. La dependencia del disco se decide mediante **Fino/Thin** o **Clone** en Asignación de recursos.

Comparación aproximada con vSphere: un tipo de instancia se parece a una talla de hardware reutilizable, no a un VM Template completo. El template incluye también los discos y el sistema preparado.

---

# Cabecera de la nueva VM

La cabecera se repite mientras navegamos por las distintas pestañas. Mantiene visibles las decisiones que condicionan todo lo demás.

## Clúster: `Curso`

Indica el Cluster en el que podrá ejecutarse la VM.

Seleccionar el Cluster determina, entre otras cosas:

- qué hosts pueden arrancarla;
- qué modelo y compatibilidad de CPU tendrá;
- qué redes lógicas están disponibles;
- qué políticas de scheduling se aplican;
- qué firmware y dispositivos son compatibles;
- entre qué hosts puede migrar.

No fija la VM a `olvm-host1` o `olvm-host2`. Solo la sitúa dentro del conjunto de hosts elegibles.

Comparación aproximada: **Cluster de vSphere**.

## Centro de datos: `Curso`

Se muestra como consecuencia del Cluster elegido. El Data Center agrupa Cluster, Storage Domains y Logical Networks bajo la misma compatibilidad.

No es una ubicación física ni un servidor.

## Plantilla: `alma9-aula | base version (1)`

Es el modelo desde el que se crea la VM.

La plantilla aporta:

- uno o varios discos base;
- sistema operativo y software ya preparados;
- firmware y hardware virtual predeterminados;
- modelos de dispositivos;
- valores iniciales de CPU y memoria;
- vNICs y otras opciones reutilizables.

`base version (1)` significa que estamos utilizando la versión raíz, número 1, de la cadena de versiones de la plantilla. No significa que la VM vaya a llamarse `base` ni que sea una instalación vacía.

Si se eligiera `Blank`, se crearía una definición de VM sin un sistema operativo preinstalado. Después habría que instalarlo desde ISO, PXE o adjuntar un disco arrancable.

Comparación aproximada: **VM Template de vSphere**.

## Sistema operativo: `AlmaLinux 9.x x64`

Declara a OLVM qué familia de sistema operativo esperamos ejecutar.

Esta elección puede influir en:

- dispositivos virtuales predeterminados;
- firmware y chipset sugeridos;
- opciones de cloud-init o Sysprep;
- icono y presentación;
- compatibilidad de determinadas operaciones.

Seleccionar `AlmaLinux 9.x x64` **no instala AlmaLinux**. El sistema está realmente dentro del disco procedente de la plantilla.

El valor declarado debe coincidir con el contenido real del disco.

## Tipo de conjunto de chips/firmware: `Q35 con UEFI`

Son dos conceptos agrupados en una selección:

- **Q35:** modelo de chipset/máquina virtual moderno presentado por QEMU.
- **UEFI:** firmware con el que arranca la VM, equivalente moderno de la BIOS tradicional.

Las alternativas habituales son:

- i440FX con BIOS legacy;
- Q35 con BIOS;
- Q35 con UEFI;
- Q35 con UEFI Secure Boot.

El disco instalado debe ser compatible con la modalidad elegida. Un sistema instalado en UEFI puede no arrancar si después se cambia a BIOS, y al contrario.

En esta plantilla mantendremos `Q35 con UEFI`. No cambiaremos firmware después de instalar el sistema salvo que sepamos cómo está particionado y arrancado el disco.

Comparación aproximada: tipo de firmware BIOS/EFI y compatibilidad de hardware virtual en vSphere.

## Optimizado para: `Servidor`

Selecciona un conjunto de valores predeterminados orientado a un tipo de uso.

- **Servidor:** configuración conservadora para una VM persistente de servidor.
- **Escritorio:** valores orientados a una sesión de usuario y dispositivos de escritorio.
- **Alto rendimiento:** modifica varios parámetros y exige revisar CPU, NUMA, memoria, migración y dispositivos.

No es un botón que haga automáticamente rápida cualquier carga. Es un punto de partida que cambia varios defaults.

Para las VMs AlmaLinux del curso, `Servidor` es la elección adecuada.

---

# Pestaña Sistema

## Tamaño de la memoria: `1024 MB`

Es la cantidad de RAM que la VM tendrá al arrancar.

Cuando se inicia:

```text
Engine valida capacidad
        ↓
VDSM crea la VM en un host
        ↓
QEMU presenta 1024 MiB al invitado
```

No significa que exista físicamente un módulo de 1 GiB reservado dentro del servidor. Es memoria del host administrada por KVM/QEMU y por las políticas del Cluster.

En nuestro laboratorio anidado 1 GiB es deliberadamente pequeño, pero permite mantener varias VMs ligeras.

## Memoria máxima: `4096 MB`

Es el límite superior hasta el que podría ampliarse la memoria de la VM mediante las funciones compatibles.

No significa:

- que OLVM reserve ahora 4 GiB;
- que la VM crezca automáticamente hasta 4 GiB;
- que ballooning añada memoria cuando haga falta.

La ampliación en caliente depende del sistema invitado, del hardware virtual y de la configuración. Si no se admite hot plug, puede requerir apagar y volver a arrancar.

## Memoria física garantizada: `1024 MB`

Es el compromiso mínimo de memoria física que OLVM mantiene para esa VM.

Debe estar entre cero y la memoria definida. Afecta a la capacidad de sobreasignar RAM y de recuperarla mediante ballooning.

En las capturas:

```text
Memoria definida:     1024 MiB
Memoria garantizada:  1024 MiB
```

Por tanto, aunque exista un dispositivo de ballooning, no hay margen para reducir la VM por debajo de ese GiB garantizado.

En producción el valor se decide según el mínimo real que necesita la aplicación. Poner garantías altas a todas las VMs reduce el overcommit disponible.

Comparación aproximada: se acerca a una **reserva de memoria** de vSphere, aunque la implementación y las políticas no son idénticas.

## Total de CPUs virtuales: `1`

Es el número total de vCPU que verá el invitado.

En parámetros avanzados puede descomponerse como:

```text
vCPU totales = sockets × cores por socket × threads por core
```

Ejemplo:

```text
2 sockets × 2 cores × 1 thread = 4 vCPU
```

Para el laboratorio se utiliza una sola vCPU para reducir el consumo de los hosts anidados.

Una vCPU no es un core físico reservado. El scheduler de Linux reparte tiempo de CPU entre procesos y VMs, salvo que se configure pinning o dedicación.

## Parámetros avanzados

Expande campos como sockets virtuales, cores por socket y threads por core.

La topología puede importar por:

- licenciamiento por socket;
- NUMA;
- comportamiento del sistema operativo;
- rendimiento y cachés;
- compatibilidad de migración.

No cambiaremos la topología solo para «hacer que parezcan más CPUs». Primero decidimos las vCPU totales y después una topología razonable.

## Tipo de instancia: `Personalizado`

Un tipo de instancia es un perfil de hardware reutilizable. Puede establecer memoria, vCPU y otros valores.

Con `Personalizado` los valores pertenecen directamente a esta VM y no quedan gobernados por una talla externa.

En esta versión el portal sigue mostrando los tipos de instancia, pero upstream los considera una función en retirada. Para el curso nos interesa entenderlos, no construir la planificación alrededor de ellos.

## Diferencia horaria en el reloj del hardware

Configura el desfase del reloj de hardware virtual que ve el invitado.

- Linux suele esperar el reloj hardware en UTC/GMT.
- Windows puede esperar una relación distinta según su configuración.

Esto no sustituye a NTP o chrony dentro de la VM. Una cosa es el reloj virtual presentado al arrancar y otra la sincronización continua del sistema operativo.

Para AlmaLinux mantendremos el valor predeterminado GMT/UTC.

## Política de números de serie

Determina qué valor presentará la VM como número de serie virtual.

Opciones habituales:

- política predeterminada del sistema o del Cluster;
- UUID del host;
- UUID de la VM;
- valor personalizado.

Puede ser relevante para inventario, CMDB o software que identifica la máquina mediante DMI.

### Valor mostrado: `Clúster predeterminado (ID del host)`

La VM hereda la política predeterminada del Cluster, que utiliza el identificador del host.

Hay que tener cuidado con aplicaciones o licencias que esperen un identificador estable después de una migración. Para esos casos se estudia una política basada en el ID de la VM o un serial personalizado.

## Número de serie personalizado

Solo se utiliza cuando la política anterior se establece en un valor personalizado.

No debemos inventar números sin integrarlos con el sistema de inventario o licenciamiento de la organización.

---

# Pestaña Alta disponibilidad

## Altamente disponible

Marca la VM como candidata a recuperación automática cuando su proceso o el host que la ejecuta fallan y OLVM puede determinar el estado con seguridad.

No convierte la aplicación en un clúster y no evita la interrupción. Normalmente la VM se **reinicia** en otro host; no continúa ejecutándose exactamente desde el punto anterior.

Para que funcione correctamente hacen falta, entre otras condiciones:

- otro host disponible;
- almacenamiento compartido accesible;
- redes compatibles;
- capacidad suficiente;
- gestión de energía y fencing fiables;
- configuración de migración compatible.

En nuestro laboratorio no la activaremos como demostración de HA real: los hosts OLVM son VMs anidadas y no tienen gestión de energía/BMC configurada.

Comparación aproximada: **vSphere HA**, no vMotion.

## Destino del dominio de almacenamiento para alquiler de MV

La traducción «alquiler» corresponde a un **VM lease**. Una traducción mucho más útil para entenderlo sería **arrendamiento**, **reserva** o **candado de ejecución de la VM**.

El lease se guarda en un volumen especial de un Storage Domain compartido. El host que ejecuta la VM debe renovarlo periódicamente.

```text
olvm-host1 ejecuta la VM
        ↓
renueva el lease en curso-nfs
        ↓
el lease indica que esa ejecución sigue siendo propietaria de la VM
```

Su objetivo principal es evitar que dos hosts ejecuten simultáneamente la misma VM y escriban sobre los mismos discos.

### Ejemplo de fallo ambiguo

Supongamos que Engine deja de recibir respuesta de `olvm-host1`. Engine todavía no sabe si:

- el host está realmente apagado;
- solo ha perdido la red de gestión;
- continúa ejecutando la VM;
- mantiene acceso a NFS y sigue escribiendo en sus discos.

Arrancar inmediatamente la misma VM en `olvm-host2` sería peligroso.

```text
¿olvm-host1 continúa renovando el lease?
        │
        ├── Sí
        │    olvm-host2 no puede adquirirlo
        │    la segunda ejecución queda bloqueada
        │
        └── No
             el lease termina caducando
             olvm-host2 puede adquirirlo
             la VM puede arrancar en el nuevo host
```

El Storage Domain actúa como punto de coordinación porque es visible desde ambos hosts.

En la captura aparece:

```text
Ningún alquiler de MV
```

Por tanto, no se ha seleccionado un Storage Domain para guardar un lease de esta VM.

Seleccionar `curso-nfs` no movería allí el disco ni crearía un disco de datos adicional. OLVM reservaría en ese dominio la información especial de lease de la VM.

No debemos confundirlo con:

- el disco virtual;
- una cuota de almacenamiento;
- un alquiler comercial;
- el rol SPM.

### Lease, alta disponibilidad y fencing

Son tres mecanismos diferentes:

```text
Altamente disponible
    política: Engine debe intentar recuperar automáticamente la VM

VM lease
    protección por VM: impide dos ejecuciones concurrentes

Fencing
    protección por host: apaga o reinicia el host dudoso mediante BMC
```

El fencing suele utilizar iLO, iDRAC, IPMI u otro sistema de gestión de energía. Actúa sobre el servidor completo. El lease actúa sobre una VM concreta y su capacidad para adquirir el candado del almacenamiento.

Por tanto, la frase correcta es:

> **Un VM lease no configura ni sustituye el fencing del host. Sin embargo, aporta protección a nivel de almacenamiento y puede permitir un reinicio seguro de la VM en otro host en diseños específicos sin power management. Para la HA estándar de hosts, Oracle sigue indicando que se configure power management y fencing.**

En algunos diseños especiales, como determinados clústeres extendidos, Oracle utiliza leases precisamente para permitir failover de VMs sin gestión de energía. Eso no convierte ambos mecanismos en equivalentes.

### En nuestro laboratorio

```text
Altamente disponible:  No
VM lease:              Ninguno
Fencing/BMC:           No configurado
```

No podemos presentar actualmente el aula como una plataforma HA validada.

Podríamos utilizar `curso-nfs` para estudiar el funcionamiento de un lease, pero seguiríamos teniendo estas limitaciones:

- los hosts OLVM son VMs anidadas sin BMC;
- `curso-nfs` y `curso-nfs-2` dependen del mismo servidor y discos físicos;
- si se pierde completamente el NFS, no quedan disponibles ni el disco de la VM ni su lease;
- habría que probar cada escenario de fallo antes de confiar en la recuperación automática.

Cuando se configura un VM lease, OLVM limita **Reanudar acción** a `KILL` para evitar que una VM permanezca pausada indefinidamente mientras otra capa intenta recuperarla.

## Reanudar acción: `Reanudar de manera automática`

Define qué hacer cuando la VM ha quedado pausada por un error de E/S de almacenamiento y después se recupera la conexión.

Opciones conceptuales:

- **AUTO_RESUME:** reanudar automáticamente.
- **LEAVE_PAUSED:** dejarla pausada para que intervenga un administrador.
- **KILL:** si el problema supera el tiempo previsto, detenerla de forma no ordenada para permitir recuperación en otro lugar; es la opción ligada a leases.

No controla el comportamiento normal al apagar la VM ni sustituye a HA. Trata específicamente la recuperación después de una pausa por storage.

Para el laboratorio, la reanudación automática es razonable. En una base de datos crítica la decisión se coordinaría con el equipo de aplicación.

## Prioridad para la cola de ejecución/migración: `Bajo`

Cuando varias VMs esperan para arrancar, migrar o recuperarse, la prioridad ayuda a ordenar la cola.

No reserva CPU ni RAM y no hace que una VM en ejecución sea más rápida.

En un diseño real podrían tener prioridad alta servicios como DNS, autenticación o bases de datos críticas. Las VMs del aula pueden permanecer con prioridad baja.

## Watchdog

Un watchdog es un temporizador virtual que permite detectar que el sistema invitado ha dejado de responder.

No es un único botón. Intervienen tres piezas:

```text
OLVM presenta un dispositivo watchdog i6300esb
        ↓
el kernel de AlmaLinux carga el driver i6300esb
        ↓
el servicio watchdog escribe periódicamente en /dev/watchdog
```

```text
Servicio watchdog del invitado funciona
        ↓ renueva contador
dispositivo virtual no llega a cero

Invitado bloqueado
        ↓ deja de renovar
contador llega a cero
        ↓
acción configurada
```

### Modelo de Watchdog: `Sin Watchdog`

No se presenta un dispositivo watchdog a la VM.

Para configurarlo en una VM de prueba:

1. Apagar la VM.
2. Abrir **Editar → Alta disponibilidad**.
3. Seleccionar el modelo `i6300esb`.
4. Elegir una acción, por ejemplo `reset`.
5. Guardar y arrancar de nuevo la VM.

El dispositivo debe aparecer dentro de AlmaLinux:

```bash
lspci | grep -i watchdog
ls -l /dev/watchdog*
lsmod | grep i6300esb
journalctl -k | grep -i watchdog
```

La salida debería mostrar el dispositivo Intel 6300ESB, el módulo `i6300esb` y `/dev/watchdog` o `/dev/watchdog0`.

### Paquete en AlmaLinux 9

El daemon de espacio de usuario se encuentra en el paquete:

```text
watchdog
```

Se instala desde AppStream:

```bash
sudo dnf install -y watchdog
```

En `/etc/watchdog.conf` comprobamos que el daemon utilice el dispositivo virtual:

```ini
watchdog-device = /dev/watchdog
```

Después habilitamos y arrancamos el servicio:

```bash
sudo systemctl enable --now watchdog.service
systemctl status watchdog.service
journalctl -u watchdog.service
```

Añadir el dispositivo en OLVM sin instalar y configurar el servicio no supervisa mágicamente la VM. El kernel vería el dispositivo, pero ningún proceso renovaría el temporizador de forma controlada.

### Acción de Watchdog

Define qué hará la plataforma si vence el temporizador:

- registrar sin actuar;
- resetear la VM;
- apagarla;
- pausarla;
- generar un volcado y pausarla.

Una configuración típica de laboratorio sería:

```text
Modelo:  i6300esb
Acción:  reset
```

No es una recomendación universal para producción. Una base de datos o un clúster de aplicación puede requerir otra acción y coordinación con sus propios mecanismos de HA.

### Qué puede comprobar el daemon

Además de renovar `/dev/watchdog`, `/etc/watchdog.conf` puede definir pruebas sobre:

- procesos mediante PID files;
- conectividad mediante ping;
- interfaces;
- cambios en ficheros;
- carga del sistema;
- memoria disponible;
- scripts de prueba y reparación.

Si las pruebas dejan de completarse, el daemon deja de renovar el dispositivo y termina ejecutándose la acción elegida en OLVM.

### Prueba segura

La primera prueba debe realizarse únicamente sobre una VM desechable:

1. comprobar que el dispositivo existe;
2. comprobar que `watchdog.service` está activo;
3. elegir una acción conocida;
4. observar los eventos de OLVM;
5. verificar que la VM vuelve al estado esperado.

No provocaremos un kernel panic ni mataremos el daemon en una VM con datos importantes. Un watchdog mal probado puede resetear una VM sana por un error de configuración.

Comparación aproximada: monitorización de guest con una acción de recuperación, no el heartbeat entre hosts de vSphere HA.

---

# Agentes y drivers dentro de una VM AlmaLinux

OLVM no utiliza un único paquete llamado «OLVM Tools» para proporcionar todas las funciones. En Linux hay varios componentes separados.

| Componente | Paquete en AlmaLinux | Función |
|---|---|---|
| QEMU Guest Agent | `qemu-guest-agent` | Comunicación de gestión entre OLVM y el invitado |
| Watchdog daemon | `watchdog` | Alimentar el temporizador y detectar fallos configurados |
| Drivers VirtIO | `kernel-modules-core` y kernel | Red, disco, SCSI, ballooning, consola y RNG |
| cloud-init | `cloud-init` | Personalización durante el primer arranque |
| Agente SPICE | `spice-vdagent` | Integración gráfica en escritorios que utilizan SPICE |

## QEMU Guest Agent

Es el componente más parecido a la parte de gestión de **VMware Tools**.

```bash
sudo dnf install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent.service
systemctl status qemu-guest-agent.service
pgrep -a qemu-ga
```

Permite, según operación y configuración:

- informar direcciones IP;
- obtener información del sistema invitado;
- solicitar apagado o reinicio ordenado;
- coordinar congelación y descongelación de filesystems;
- mejorar la consistencia de determinados snapshots.

No ejecuta el watchdog y no proporciona los drivers VirtIO. Son componentes distintos.

En AlmaLinux 9 no instalaremos el antiguo `ovirt-guest-agent` utilizado por versiones históricas. Utilizamos `qemu-guest-agent`.

## Drivers VirtIO

Los principales drivers vienen con el kernel:

```text
virtio_net       red
virtio_scsi      controlador SCSI paravirtualizado
virtio_blk       disco de bloques
virtio_balloon   memory ballooning
virtio_rng       generador aleatorio
virtio_console   canal y consola VirtIO
```

Podemos observarlos con:

```bash
lspci | grep -i virtio
lsmod | grep virtio
```

No hace falta descargar una ISO de drivers para AlmaLinux. En Windows sí es necesario instalar los drivers VirtIO correspondientes.

## Separación mental

```text
qemu-guest-agent
    OLVM pregunta u ordena acciones dentro de la VM

watchdog
    detecta que el invitado o una prueba ha dejado de responder

VirtIO
    proporciona dispositivos paravirtualizados eficientes

cloud-init
    personaliza la VM durante sus primeros arranques
```

---

# Pestaña Asignación de recursos

Esta pestaña mezcla tres familias distintas:

```text
CPU y scheduling
memoria y dispositivos de rendimiento
creación de los discos desde la plantilla
```

## Perfil de CPU: `Curso`

Es un objeto de política asociado al Cluster. Puede aplicar una QoS de CPU que limite el porcentaje de capacidad de procesamiento disponible para la VM.

El nombre `Curso` no demuestra que exista un límite concreto. Para saberlo hay que abrir el perfil y revisar la QoS asociada.

No es el modelo de CPU del Cluster. El modelo determina compatibilidad de instrucciones; el perfil controla consumo o política.

Comparación aproximada: política o límite de CPU, no EVC.

## Recursos de CPU: `Inhabilitado`

Corresponde a los **CPU shares**, el peso relativo de esta VM cuando existe contención de CPU.

Valores típicos:

- bajo;
- medio;
- alto;
- personalizado;
- inhabilitado/predeterminado.

Los shares no son GHz reservados. Solo influyen cuando varias VMs compiten por tiempo de procesador.

Con `Inhabilitado` y valor `0` no se ha establecido un peso personalizado para esta VM.

Comparación aproximada: **CPU Shares** de vSphere.

## CPU Pinning Policy: `None`

Define si una vCPU puede ejecutarse libremente en los procesadores del host o queda ligada a procesadores físicos concretos.

Opciones que pueden aparecer:

- **None:** sin pinning.
- **Manual:** relación vCPU–pCPU definida por el administrador.
- **Resize and Pin NUMA:** ajusta y fija según la topología NUMA del host.
- **Dedicated:** dedica procesadores físicos a las vCPU.
- **Isolate Threads:** aislamiento más estricto por core/hilo según compatibilidad.

El pinning puede mejorar previsibilidad en cargas especiales, pero reduce flexibilidad, complica NUMA y puede limitar la migración.

Para el aula: `None`.

## Topología de anclado de CPU

Campo utilizado por el pinning manual. Expresa qué vCPU puede ejecutarse sobre qué pCPU.

Ejemplos de sintaxis:

```text
0#0          vCPU 0 sobre pCPU 0
0#0_1#3     vCPU 0 sobre pCPU 0 y vCPU 1 sobre pCPU 3
```

No se rellena si la política es `None`.

En producción se construye después de estudiar:

- sockets y cores reales;
- NUMA;
- hyperthreading/SMT;
- afinidad del dispositivo;
- threads de QEMU;
- política de migración.

## Unificación de memoria —ballooning— habilitada

El término habitual es **memory ballooning**. Se presenta un dispositivo VirtIO dentro del invitado para que el host pueda pedirle que libere determinadas páginas de memoria.

```text
Host necesita recuperar RAM
        ↓
infla el balloon del invitado
        ↓
el driver reserva páginas dentro de la VM
        ↓
QEMU puede devolver memoria al host
```

No es memoria swap y no aumenta automáticamente la memoria máxima de la VM. El invitado sigue viendo presión de memoria: las páginas ocupadas por el balloon dejan de estar disponibles para sus aplicaciones.

### Driver y paquete en AlmaLinux 9

El driver se llama:

```text
virtio_balloon
```

No existe un paquete independiente denominado `balloon` u `OLVM-balloon`. El módulo está incluido en el paquete del kernel:

```text
kernel-modules-core
```

Normalmente ya está instalado. Podemos comprobarlo así:

```bash
modinfo virtio_balloon
lsmod | grep virtio_balloon
lspci | grep -i virtio
journalctl -k | grep -i balloon
```

Para identificar exactamente qué RPM proporciona el módulo cargado:

```bash
rpm -qf "$(modinfo -n virtio_balloon)"
```

La respuesta será similar a:

```text
kernel-modules-core-5.14.0-...el9.x86_64
```

Si el módulo existe pero no está cargado, se puede comprobar su carga manual:

```bash
sudo modprobe virtio_balloon
```

Normalmente se carga automáticamente cuando OLVM presenta el dispositivo a la VM. Cargar el módulo sin que exista el dispositivo virtual no crea capacidad de ballooning por sí solo.

No hace falta `qemu-guest-agent` para mover las páginas del balloon. El camino es diferente:

```text
MoM / política del host
        ↓
libvirt y QEMU
        ↓
dispositivo VirtIO balloon
        ↓
driver virtio_balloon dentro de AlmaLinux
```

El Guest Agent puede aportar otras funciones de gestión, pero no sustituye al driver `virtio_balloon`.

### Relación con la memoria garantizada

La memoria física garantizada marca el límite que OLVM no debería rebasar al recuperar memoria.

En nuestra captura la memoria definida y la garantizada son ambas 1024 MiB. Por tanto, el dispositivo existe, pero no ofrece margen práctico para recuperar memoria por debajo de 1 GiB.

```text
Memoria definida:     1024 MiB
Memoria garantizada:  1024 MiB
Margen recuperable:      0 MiB
```

Para disponer de margen tendría que existir una diferencia, por ejemplo:

```text
Memoria definida:     1024 MiB
Memoria garantizada:   512 MiB
Margen teórico:        512 MiB
```

Ese margen no es memoria gratuita. Si el host infla el balloon mientras la VM necesita realmente el GiB completo, el invitado puede entrar en presión de memoria, hacer reclaim o utilizar swap.

Comparación aproximada: **VMware balloon driver**, incluido tradicionalmente en VMware Tools.

## Dispositivo TPM habilitado

Añade un **Trusted Platform Module virtual** a la VM.

Puede utilizarse para:

- Secure Boot y medición del arranque;
- BitLocker;
- almacenamiento protegido de claves;
- sistemas que exigen TPM 2.0.

En x86 normalmente requiere UEFI. Que la casilla aparezca gris indica que la combinación o el estado actual no permite editarla; no demuestra por sí solo que el host carezca de TPM físico.

Una VM con vTPM añade estado sensible que debe considerarse al migrar, exportar o restaurar.

Para AlmaLinux del aula no lo necesitamos.

Comparación aproximada: **vTPM** de vSphere.

## Hilos de E/S habilitados: `1`

Separa trabajo de E/S de los discos VirtIO en uno o varios threads de QEMU, evitando que todo el procesamiento recaiga en el hilo principal de la VM.

Puede mejorar el rendimiento de disco, especialmente con varias operaciones concurrentes.

No equivale a añadir una vCPU al sistema invitado. Es un thread del proceso QEMU en el host.

Con una VM pequeña, un único hilo de E/S es un valor prudente.

## Varias colas habilitadas

Habilita múltiples colas para las vNIC VirtIO. Permite distribuir el procesamiento de red entre varias vCPU cuando la VM tiene tráfico y paralelismo suficientes.

No es una cola de migraciones ni una cola del Storage Domain.

Con una sola vCPU la VM tiene poco paralelismo y el beneficio será limitado. Podemos conservar el valor heredado, pero no debemos presentarlo como una mejora automática.

Comparación aproximada: multiqueue de un adaptador virtual, relacionado con RSS en el invitado.

## Fino frente a Clone

Esta selección solo aparece cuando la VM se crea desde una plantilla.

### Fino —Thin—

La nueva VM utiliza la imagen de la plantilla como base de solo lectura y escribe sus cambios en una capa propia.

```text
Template base
   ├── cambios VM 1
   ├── cambios VM 2
   └── cambios VM 3
```

Ventajas:

- creación rápida;
- poco consumo inicial;
- adecuado para despliegues masivos o temporales.

Consecuencias:

- la VM depende de la imagen de la plantilla;
- hay que conservar y gestionar correctamente la cadena;
- el formato será QCOW2.

Comparación aproximada: **linked clone**.

### Clone —seleccionado en la captura—

Se crea una copia independiente del disco de la plantilla.

```text
Template base ──copia──> disco independiente de la VM
```

Ventajas:

- la VM deja de depender del disco base del template;
- ciclo de vida más independiente;
- adecuada para VMs duraderas.

Consecuencias:

- tarda más en crearse;
- genera una operación de copia;
- consume más almacenamiento;
- puede aparecer como `Image Locked` hasta que finalice.

En el laboratorio no lanzaremos diez clones simultáneos. Aunque el NFS tenga capacidad, los hosts, la red y el backend son compartidos.

## VirtIO-SCSI activado

Presenta a la VM un controlador SCSI paravirtualizado.

Permite conectar discos mediante VirtIO-SCSI y facilita funciones como:

- buen rendimiento;
- hot plug de discos;
- mayor número de dispositivos;
- identificación y gestión SCSI coherente;
- discard/UNMAP cuando toda la cadena lo soporta.

No es un formato de disco. Un disco QCOW2 puede presentarse mediante VirtIO-SCSI.

El invitado necesita el driver correspondiente. AlmaLinux lo incluye.

## VirtIO-SCSI Multi Queues: `Disabled`

Permite que el controlador VirtIO-SCSI procese E/S mediante varias colas.

Puede mejorar throughput cuando:

- la VM tiene varias vCPU;
- existen varios discos o múltiples flujos de E/S;
- el invitado y los drivers lo soportan;
- el backend es capaz de atender el paralelismo.

Para una VM de una vCPU y 10 GiB del aula, mantenerlo deshabilitado es razonable.

## Asignación de discos

La tabla describe cómo se materializará cada disco de la plantilla para la nueva VM.

### Alias: `alma9-disco`

Nombre administrativo del Virtual Disk dentro de OLVM.

No es:

- el nombre `/dev/sda` o `/dev/vda` dentro del invitado;
- una ruta NFS;
- el nombre del filesystem.

Conviene cambiarlo por un alias que identifique la VM cuando se creen muchas copias, por ejemplo `demo-dia3-sistema`.

### Tamaño virtual: `10 GiB`

Capacidad que verá el invitado.

No demuestra que ya se hayan ocupado físicamente 10 GiB en NFS. El consumo real depende del formato, la política de asignación y los bloques escritos.

### Formato: `QCOW2`

Formato de imagen de QEMU que soporta copy-on-write y snapshots.

Ventajas:

- asignación dinámica de bloques;
- snapshots y cadenas de imágenes;
- creación eficiente.

Costes:

- más metadatos y complejidad que RAW;
- rendimiento diferente según patrón y backend;
- exige vigilar cadenas y capacidad real.

`Clone + QCOW2` significa:

- **Clone:** la VM será independiente del template.
- **QCOW2:** el disco independiente utiliza este formato.

No son opciones contradictorias.

### RAW

Representación más directa del disco, con menos funciones de copy-on-write. Puede favorecer determinados patrones de rendimiento, pero su comportamiento de asignación depende también del backend.

No elegimos RAW automáticamente porque «sea más rápido»; se decide según snapshots, aprovisionamiento y características del almacenamiento.

### Destino: `curso-nfs`

Storage Domain en el que OLVM guardará la imagen del disco.

En nuestro caso es un Data Domain NFS. El disco no se copia dentro del host donde arranca la VM; los dos hosts OLVM acceden al dominio compartido.

Podríamos elegir `curso-nfs-2` para practicar movimiento y selección de dominios, pero ambos exports están respaldados por los mismos discos físicos del laboratorio. No aportan redundancia real.

### Perfil del disco: `curso-nfs`

Objeto de política aplicado al Virtual Disk. Puede asociar una QoS de almacenamiento.

No es el Storage Domain, aunque el perfil predeterminado tenga el mismo nombre.

```text
Destino            = dónde se almacena el disco
Perfil del disco   = qué política de E/S se le aplica
```

Para saber si limita IOPS o throughput hay que abrir el perfil y revisar la QoS asociada. El nombre por sí solo no lo demuestra.

Comparación aproximada: política de almacenamiento aplicada al disco, sin asumir equivalencia completa con SPBM.

---

# Pestaña Icono

## Cargar

Permite asignar un icono personalizado a la VM para reconocerla en Administration Portal y VM Portal.

No modifica el sistema operativo, el firmware ni el rendimiento.

Los formatos admitidos por la interfaz upstream son JPG, PNG y GIF, con límites pequeños de tamaño y dimensiones. Es un elemento visual, no la imagen ISO o QCOW2 de la VM.

## Usar predeterminado

Recupera el icono asociado al sistema operativo declarado o el icono genérico.

La palabra «imagen» puede confundir aquí:

```text
Icono       → dibujo mostrado en el portal
ISO         → CD/DVD virtual de instalación
QCOW2/RAW   → imagen de un disco virtual
Template    → modelo administrado para crear VMs
```

---

# Pestaña Foreman/Satellite

## Proveedor: `Sin proveedor`

Permite asociar una VM Enterprise Linux con un proveedor Foreman o Red Hat Satellite registrado en OLVM.

La integración puede utilizarse para consultar contenido y erratas relevantes para el sistema.

No es:

- el proveedor de red;
- el repositorio Glance;
- el Guest Agent;
- el host donde se ejecutará la VM.

En el aula no hay un proveedor Foreman/Satellite configurado y se mantiene `Sin proveedor`.

En Caixa o Redeia podría tener sentido si existe una plataforma corporativa de lifecycle y parcheado integrada. Seleccionarlo no sustituye al registro y configuración del sistema invitado.

---

# Pestaña Afinidad

La afinidad controla dónde debe o no debe ejecutarse una VM respecto de otras VMs o hosts.

No es afinidad de red ni afinidad con un Storage Domain.

## Seleccione un grupo de afinidad

Añade la VM a un grupo que ya contiene reglas de colocación.

Un grupo puede expresar:

- **VM–VM positiva:** intentar mantener varias VMs juntas.
- **VM–VM negativa:** intentar separarlas.
- **VM–Host positiva:** ejecutar las VMs sobre ciertos hosts.
- **VM–Host negativa:** evitar ciertos hosts.
- **soft:** preferencia que puede incumplirse si no hay alternativa.
- **hard/enforcing:** obligación; si no puede cumplirse, la VM puede no arrancar.

Ejemplos reales:

- separar dos controladores de dominio;
- separar nodos de un clúster de base de datos;
- mantener una aplicación cerca de un host con un dispositivo específico;
- restringir software licenciado a un conjunto de hosts.

Comparación aproximada: reglas **VM/Host Rules** y DRS de vSphere.

## Grupos de afinidad seleccionados

Muestra los grupos a los que se incorporará la nueva VM.

En la captura no hay ninguno. Por tanto, el scheduler utilizará las políticas normales del Cluster, sin reglas adicionales procedentes de esta sección.

## Seleccionar una etiqueta de afinidad

Las etiquetas permiten asociar VMs y hosts mediante un identificador común utilizado por el scheduling.

Son una forma más simple de expresar conjuntos relacionados, pero pueden interactuar o entrar en conflicto con grupos de afinidad.

No deben confundirse con una etiqueta meramente descriptiva o con una VLAN.

## Etiquetas de afinidad seleccionadas

Muestra las etiquetas que se aplicarán. En la captura no se ha seleccionado ninguna.

Para el laboratorio las dejamos vacías. Solo crearíamos una regla después de decidir:

1. qué VMs afecta;
2. qué hosts afecta;
3. si queremos juntar o separar;
4. si la regla es preferida u obligatoria;
5. qué debe ocurrir si ningún host puede cumplirla.

---

# Secciones generales

## General

Identidad de la VM, plantilla, descripción, discos iniciales y vNICs. Aquí se elige normalmente el perfil `alumnos` para la interfaz de red.

## Ejecución inicial

Personalización de primer arranque mediante cloud-init en Linux o Sysprep en Windows.

Puede definir:

- hostname;
- usuario o contraseña inicial;
- claves SSH;
- timezone;
- red y DNS;
- user-data o script.

OLVM presenta los datos; el agente cloud-init dentro del invitado debe procesarlos.

## Consola

Protocolo y dispositivo gráfico de la consola: VNC, SPICE, tipo de vídeo, monitores y opciones de acceso.

La consola funciona aunque la VM todavía no tenga IP. No es lo mismo que SSH o RDP a través de la red del invitado.

## Host

Política de colocación y migración:

- cualquier host del Cluster;
- hosts concretos;
- migración permitida o restringida;
- CPU passthrough;
- NUMA y pinning avanzado.

Fijar una VM a un host puede impedir o limitar la live migration y la recuperación en otro host.

## Opciones de arranque

Orden persistente de dispositivos:

- disco;
- CD-ROM/ISO;
- red PXE.

También permite adjuntar una ISO y mostrar un menú de selección.

No debemos confundir esta configuración persistente con **Run Once**, que aplica parámetros para un solo arranque.

## Generador aleatorio

Añade un dispositivo paravirtualizado `virtio-rng` que proporciona entropía del host al invitado.

Es útil para generación de claves y arranques que necesitan aleatoriedad. Puede limitarse por bytes y periodo para que una VM no consuma toda la fuente compartida.

## Propiedades personalizadas

Parámetros avanzados de QEMU/libvirt expuestos por OLVM, por ejemplo huge pages o determinadas opciones de red y disco.

No se utilizan como cajón de sastre. Pueden afectar migración, rendimiento e integridad y deben documentarse y probarse.
