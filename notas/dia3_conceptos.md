# Día 3 · Ciclo de vida de una VM, templates, pools y permisos

---

# Qué vamos a conseguir hoy

Los dos primeros días hemos construido la infraestructura mental de OLVM:

```text
Engine → Data Center → Cluster → Host
                         ├── Storage Domains
                         └── Logical Networks
```

Hoy vamos a recorrer esa infraestructura desde el punto de vista del objeto que la utiliza: **la máquina virtual**.

No vamos a instalar el Engine ni nuevos hosts. Tampoco vamos a entrar en OVN, alta disponibilidad o tuning avanzado. El objetivo es seguir una VM durante toda su vida:

```text
definir → crear → arrancar → observar → modificar
       → snapshot → template → desplegar → autorizar → migrar
```

Al terminar el día quiero que podáis:

1. Distinguir la definición de una VM de su proceso en ejecución.
2. Interpretar sus recursos, discos, vNICs, consola y opciones de arranque.
3. Explicar qué aporta el QEMU Guest Agent y qué no hace.
4. Añadir y retirar dispositivos sin confundir la vista de OLVM con la del invitado.
5. Crear y restaurar un snapshot entendiendo sus riesgos.
6. Diferenciar snapshot, clone, template, pool y backup.
7. Explicar el sellado y la personalización con cloud-init.
8. Construir un permiso como combinación de usuario, rol y objeto.
9. Distinguir permisos, cuotas y QoS.
10. Explicar qué se mueve y qué permanece durante una live migration.

La frase del día será:

> **Una VM no es solo un proceso: es una definición, una identidad, unos discos, unas redes y unas políticas que OLVM coordina.**

## Puente rápido con vSphere

Partimos de conceptos que el grupo ya conoce, pero marcando dónde la equivalencia deja de ser exacta:

| En OLVM | Referencia útil en vSphere | Matiz importante |
|---|---|---|
| Objeto VM en Engine | VM registrada en vCenter | La ejecución real ocurre en un host, no en el gestor |
| QEMU Guest Agent | Parte funcional de VMware Tools | Guest Agent no es VDSM; uno vive en la VM y el otro en el host |
| Template | VM Template | En ambos casos hay que evitar duplicar identidades del sistema operativo |
| Clone dependiente | Linked clone, conceptualmente | La implementación y el ciclo de vida no se administran con las mismas reglas |
| VM Pool | Pool de escritorios o reserva de VMs | No tiene un equivalente único en el vSphere básico |
| Live migration | vMotion | Con NFS compartido se mueve la ejecución, no el disco |
| VM Portal | Portal de autoservicio | No equivale al vSphere Client administrativo completo |
| Usuario/grupo + rol + objeto | Principal + rol + objeto de inventario | El alcance y la herencia son tan importantes como el nombre del rol |

La comparación sirve para encontrar la idea conocida. Después volvemos a los objetos y procedimientos reales de OLVM.

---

# Reparto de la jornada

| Bloque | Duración | Contenido |
|---|---:|---|
| 1 | 50 min | Anatomía de una VM y relación con los días anteriores |
| 2 | 60 min | Ciclo de vida, Guest Agent y hot plug de dispositivos |
| Pausa | 15 min | |
| 3 | 55 min | Snapshots: crear, verificar, restaurar y eliminar |
| 4 | 50 min | Templates, sellado, cloud-init, clones y pools |
| Pausa | 15 min | |
| 5 | 40 min | Usuarios, roles, permisos, VM Portal y cuotas |
| 6 | 20 min | Live migration y caso integrado |
| Cierre | 10 min | Repaso y preguntas orales |

Total: **5 horas**, incluidas dos pausas de 15 minutos.

---

# Reglas del laboratorio

La instalación es pequeña y está anidada. Los dos hosts OLVM son VMs con memoria limitada y los usuarios del curso disponen actualmente de privilegios amplios.

Eso cambia la forma de practicar.

## Objetos que no se tocan

- VMs cuyo nombre termine en `-base`.
- Templates utilizadas para crear las VMs de los alumnos.
- VMs de otro alumno.
- Storage Domains `curso-nfs` y `curso-nfs-2` a nivel de activación, mantenimiento o borrado.
- Logical Network `ovirtmgmt`.
- Configuración de hosts, bonds o bridges durante estas prácticas.

## Objeto de demostración

El instructor utilizará una VM prescindible con un nombre inequívoco:

```text
demo-dia3
```

Si ya existe una VM con ese nombre, no se crea otra sin comprobar antes su propietario y propósito.

## Concurrencia

- No crearemos diez templates a la vez.
- No arrancaremos todas las VMs pesadas simultáneamente.
- Snapshots y merges se harán por turnos.
- La live migration será una demostración controlada.
- Antes de una operación esperaremos a que desaparezcan estados como `Image Locked`.

El hecho de que un botón esté habilitado no significa que sea seguro pulsarlo.

---

# Bloque 1 · Anatomía de una máquina virtual

## El objeto y el proceso

Cuando una VM está apagada, OLVM conserva su definición:

- nombre e identificador;
- Cluster;
- CPU y memoria;
- firmware y chipset;
- discos;
- vNICs y MAC;
- dispositivos;
- políticas de migración y alta disponibilidad;
- permisos;
- snapshots;
- afinidad y opciones de ejecución.

Cuando se arranca, esa definición se materializa en un host:

```text
Definición en Engine
        ↓
selección de host
        ↓
Engine → VDSM → libvirt
        ↓
proceso QEMU + KVM
        ↓
VM ejecutándose
```

La VM no vive dentro del Engine. Engine conserva y coordina la definición; el proceso se ejecuta en uno de los hosts.

Comparación con vSphere:

```text
Objeto VM en OLVM        ≈ objeto VM en vCenter
Proceso QEMU en el host  ≈ VMX world/proceso de la VM en ESXi, conceptualmente
```

No utilizaremos esta comparación para deducir nombres de ficheros ni procedimientos internos.

## Estados que debemos reconocer

Los nombres exactos pueden variar con la traducción del portal, pero debemos interpretar al menos:

| Estado | Significado práctico |
|---|---|
| `Down` | La VM está definida, pero no ejecutándose |
| `Powering Up` | Se están preparando recursos y arrancando el proceso |
| `Up` | La VM está ejecutándose |
| `Powering Down` | Está en curso un apagado ordenado o una transición |
| `Paused` | La ejecución está detenida temporalmente |
| `Migrating From/To` | La VM participa en una migración |
| `Image Locked` | Existe una operación sobre sus imágenes; no debemos iniciar otra |
| `Not Responding` | Engine no recibe la información esperada; no equivale automáticamente a sistema operativo caído |

Un estado es una evidencia, no toda la causa.

## Recorrido por la ficha de una VM

Elegimos una VM de laboratorio y respondemos sin modificarla.

### General

- Nombre e ID.
- Estado.
- Host actual.
- Cluster.
- Sistema operativo declarado.
- IP reportada, si existe Guest Agent.
- Memoria, CPU y uptime.

La IP mostrada por Engine es información reportada. No significa que OLVM haya configurado esa IP dentro del invitado.

### Discos

Para cada Virtual Disk:

- alias;
- tamaño virtual;
- tamaño real;
- Storage Domain;
- formato;
- política de asignación;
- interfaz virtual;
- perfil de disco;
- estado y posibilidad de compartirlo.

Recuperamos el camino del día 2:

```text
VM → Virtual Disk → perfil de disco → Storage Domain → NFS
```

Recordatorio:

```text
Storage QoS  = contiene límites de IOPS/caudal
Disk Profile = aplica una QoS y permisos dentro de un Storage Domain
Virtual Disk = selecciona el perfil
```

### Interfaces de red

Para cada vNIC:

- nombre;
- estado conectada/desconectada;
- modelo;
- MAC;
- vNIC Profile;
- Logical Network;
- IP reportada, si está disponible.

Camino del día 2:

```text
VM → vNIC → vNIC Profile → Logical Network
   → TAP/vnet → bridge → uplink
```

### Snapshots

Comprobamos si existe una cadena previa. No creamos uno sin saber:

- qué discos incluye;
- si conserva memoria;
- cuánto tiempo lleva existiendo;
- si una tarea de merge continúa activa.

### Permisos

Vemos qué usuarios y grupos tienen acceso directo o heredado. No confundimos “puedo verla” con “puedo administrar todo el entorno”.

### Aplicaciones e información del invitado

Estos datos dependen del Guest Agent y de lo que el invitado pueda reportar. Una pestaña vacía puede significar agente ausente, detenido o sin comunicación; no demuestra que la VM esté apagada.

## CPU: no son cuatro números intercambiables

Una VM puede expresar su CPU virtual mediante:

```text
vCPU totales = sockets virtuales × cores por socket × threads por core
```

Para la mayoría de cargas modernas interesa una topología sencilla. Sin embargo, la topología puede afectar:

- licenciamiento del software invitado;
- límites del sistema operativo;
- NUMA y rendimiento;
- capacidad de hot plug;
- comparación con la topología física.

No enseñaremos que “más sockets” equivale a “más rendimiento”. Dos configuraciones con el mismo total de vCPU pueden no ser equivalentes para licencias o para el invitado.

Comparación con vSphere: el concepto de sockets y cores virtuales es conocido, pero las políticas y los límites se comprueban en OLVM y en el software licenciado.

## Memoria definida, garantizada y máxima

Debemos distinguir:

- **Memoria definida:** cantidad presentada a la VM al arrancar.
- **Memoria física garantizada:** cantidad que OLVM debe considerar garantizada para la VM.
- **Memoria máxima:** techo que condiciona determinadas ampliaciones en caliente.

La memoria garantizada no es “RAM reservada dentro del sistema operativo invitado”. Es un compromiso de planificación en la plataforma.

En este laboratorio evitaremos ampliar memoria porque los hosts anidados tienen poco margen.

## Firmware, chipset y tipo de sistema operativo

Estas opciones no son decoración.

- BIOS y UEFI siguen caminos de arranque diferentes.
- El chipset determina dispositivos y capacidades presentadas.
- El sistema operativo declarado ayuda a OLVM a ofrecer valores compatibles.
- Una imagen preparada para UEFI puede no arrancar si se crea con BIOS, y al revés.

En nuestra instalación las plantillas disponibles no deben mezclarse a ciegas. Antes de crear una VM comprobamos cómo se preparó su imagen base.

## Comprobación oral del bloque

1. ¿Existe una VM apagada? **Sí: existe su definición aunque no exista el proceso QEMU.**
2. ¿Dónde se ejecuta una VM encendida? **En un host KVM.**
3. ¿La IP mostrada en el portal la asigna necesariamente Engine? **No.**
4. ¿Perfil de disco y QoS son el mismo objeto? **No.**
5. ¿Cambiar la topología de vCPU sin cambiar el total siempre es irrelevante? **No.**

---

# Bloque 2 · Ciclo de vida, Guest Agent y dispositivos

## Arrancar una VM

Al pulsar **Run**, Engine debe validar:

- permiso del usuario;
- estado de la VM;
- Cluster y hosts candidatos;
- CPU compatible;
- memoria disponible;
- redes necesarias;
- acceso a sus Storage Domains;
- afinidad, pinning y otras restricciones.

Después selecciona un host y solicita a VDSM que materialice la VM.

Un fallo de arranque no se diagnostica mirando solamente el botón. Leemos el evento y preguntamos qué validación no se cumplió.

## Apagado ordenado, reinicio y corte de energía

No son la misma operación.

| Operación | Intención |
|---|---|
| Shutdown | Solicitar al sistema invitado un apagado ordenado |
| Reboot | Solicitar un reinicio ordenado |
| Power Off/Stop | Cortar la ejecución cuando el invitado no coopera |
| Pause/Suspend | Detener temporalmente la ejecución según la función disponible |

La opción de corte forzado se parece a retirar la alimentación de un servidor. Puede dejar aplicaciones o filesystems en estado inconsistente.

> **Que OLVM pueda detener el proceso no significa que el sistema invitado haya cerrado bien sus datos.**

## QEMU Guest Agent

El Guest Agent se ejecuta dentro de la VM y se comunica con la capa de virtualización mediante un canal virtual, no mediante SSH.

Puede aportar:

- nombre del invitado;
- sistema operativo;
- direcciones IP;
- información de aplicaciones y uso de recursos;
- apagado y reinicio coordinados;
- congelación y descongelación de filesystem para determinadas operaciones;
- mayor consistencia en snapshots en vivo.

No hace estas cosas:

- no sustituye a VDSM;
- no convierte la VM en un host OLVM;
- no asigna por sí mismo una dirección IP;
- no sustituye a DHCP, cloud-init o NetworkManager dentro del invitado;
- no reemplaza un sistema de monitorización de aplicaciones;
- no es el driver VirtIO del disco o de la red.

Comparación aproximada:

```text
QEMU Guest Agent ≈ parte de VMware Tools
```

La analogía es útil para la función, no para paquetes ni protocolos.

## Práctica 1 · Verificar el Guest Agent

En una VM Linux adecuada:

```bash
systemctl is-active qemu-guest-agent
systemctl status qemu-guest-agent --no-pager
```

Desde el portal observamos:

- sistema operativo reportado;
- hostname;
- direcciones IP;
- aplicaciones, si aparecen;
- respuesta a un apagado ordenado.

Si la VM de CirrOS no tiene agente, no intentamos instalar paquetes a ciegas. La comparamos con una VM AlmaLinux o Debian preparada para el laboratorio.

## VirtIO y Guest Agent no son lo mismo

**VirtIO** proporciona dispositivos virtuales eficientes, por ejemplo:

- `virtio-net` para red;
- `virtio-blk` o `virtio-scsi` para discos;
- canal serie VirtIO para comunicación con el agente.

El Guest Agent es un proceso dentro del invitado. Los drivers VirtIO permiten usar dispositivos virtuales. Una VM puede utilizar discos y red VirtIO aunque el Guest Agent esté detenido.

## Hot plug

Hot plug significa añadir o conectar un dispositivo mientras la VM está encendida. Hot unplug significa retirarlo en caliente.

La posibilidad real depende de tres capas:

```text
OLVM permite la operación
        +
hardware virtual/configuración compatible
        +
sistema operativo invitado compatible
```

No basta con que la interfaz muestre un botón.

## Práctica 2 · Añadir un disco

Objetivo: observar la misma operación desde OLVM y desde el invitado.

### Antes de crear

Confirmamos:

- VM correcta;
- Storage Domain elegido;
- capacidad libre;
- tamaño pequeño, por ejemplo 1 GiB;
- thin provisioning;
- interfaz VirtIO-SCSI o la compatible con la VM;
- perfil de disco sin una QoS accidental;
- opción shareable desactivada.

No utilizamos el valor predeterminado sin leerlo.

### En OLVM

1. Abrir la VM de laboratorio.
2. Ir a **Disks**.
3. Crear o adjuntar el disco.
4. Elegir expresamente `curso-nfs-2` para distinguirlo del disco del sistema, si el instructor lo autoriza.
5. Esperar a que el disco quede `OK`.
6. Conectarlo en caliente si la VM y la interfaz lo permiten.

### Dentro del invitado

```bash
lsblk
dmesg | tail -n 30
```

En una VM completa también podemos identificarlo con:

```bash
ls -l /dev/disk/by-id/
```

No suponemos que será siempre `/dev/vdb`. Lo identificamos por tamaño y por el cambio observado.

Si se autoriza escritura:

```bash
sudo mkfs.xfs /dev/vdX
sudo mkdir -p /mnt/dia3
sudo mount /dev/vdX /mnt/dia3
findmnt /mnt/dia3
```

`/dev/vdX` es un marcador. Debe sustituirse por el dispositivo comprobado; no se copia literalmente.

### Antes de retirar

```bash
sudo umount /mnt/dia3
```

Después se desconecta desde OLVM. Borrar el disco es una operación distinta y no se realiza hasta confirmar que el objeto es el de la práctica.

## Qué hemos demostrado

```text
Portal: Virtual Disk conectado
          ↓
Host: QEMU presenta un dispositivo virtual
          ↓
Invitado: aparece un dispositivo de bloques
          ↓
Invitado: crea su propio filesystem
```

El filesystem creado dentro de la VM no es un nuevo Storage Domain.

## Práctica 3 · Estado de una vNIC

Sobre una vNIC de laboratorio distinguimos:

- **plugged/unplugged:** dispositivo presentado o retirado de la VM;
- **linked/unlinked:** enlace virtual conectado o desconectado;
- perfil vNIC seleccionado;
- configuración IP dentro del invitado.

Una vNIC puede existir y estar desconectada. También puede estar conectada y no tener IP.

No cambiaremos la vNIC de `ovirtmgmt`. Utilizaremos la red `alumnos` y verificaremos el perfil antes de aceptar.

## Cambios pendientes

Algunas modificaciones se aplican en caliente; otras quedan pendientes hasta el siguiente apagado y arranque.

Cuando aparece un indicador de cambios pendientes preguntamos:

1. ¿La definición de Engine ya cambió?
2. ¿La VM en ejecución utiliza el valor viejo o el nuevo?
3. ¿Hace falta reboot o apagado completo?

No reiniciamos a ciegas para “que se arregle”.

---

# Bloque 3 · Snapshots

## Qué es un snapshot

Un snapshot conserva una vista de la configuración y de los discos seleccionados en un momento concreto. Opcionalmente puede incluir memoria.

```text
Disco base
    ↓
snapshot / nueva capa copy-on-write
    ↓
escrituras posteriores
```

Las escrituras nuevas no sustituyen inmediatamente la imagen anterior; se construye una cadena que permite volver a un estado previo.

## Qué no es

Un snapshot no es:

- una copia independiente en otro sistema;
- una protección frente a pérdida completa del Storage Domain;
- una política de retención;
- un backup del Engine;
- una garantía de consistencia de la aplicación;
- algo que deba conservarse indefinidamente.

> **Snapshot sirve para volver atrás; backup sirve para sobrevivir a la pérdida del original.**

## Consistencia

Podemos distinguir:

- **Crash-consistent:** parecido al estado de los discos después de un corte de alimentación.
- **Filesystem-consistent:** el filesystem ha coordinado sus escrituras.
- **Application-consistent:** la aplicación también ha vaciado o coordinado su estado.

El Guest Agent puede ayudar a congelar el filesystem, pero no entiende automáticamente todas las bases de datos o aplicaciones. Una base de datos puede requerir hooks o procedimientos propios.

## Snapshot con memoria

Guardar memoria permite recuperar también el estado de ejecución, pero:

- aumenta el tiempo y el espacio necesario;
- añade complejidad;
- no siempre es necesario;
- no convierte el snapshot en backup.

En la práctica de hoy haremos un snapshot de discos sin memoria para que el resultado sea fácil de explicar.

## Preview, Commit y Undo

La restauración no debe explicarse como un botón mágico.

1. **Preview:** arrancamos temporalmente desde el estado elegido para comprobarlo.
2. **Commit:** aceptamos ese estado como estado restaurado de la VM.
3. **Undo:** abandonamos la previsualización y volvemos al estado anterior a ella.

Para restaurar, la VM debe estar apagada según el procedimiento documentado.

En vSphere suele pensarse directamente en **Revert**. En OLVM conviene enseñar las tres decisiones por separado: primero probamos con `Preview`, después elegimos entre aceptar con `Commit` o abandonar con `Undo`.

## Práctica 4 · Crear y restaurar un snapshot

Utilizamos `demo-dia3` o la VM expresamente autorizada.

### Fase A · Estado inicial

Dentro de la VM:

```bash
date | sudo tee /root/estado-dia3.txt
sudo cat /root/estado-dia3.txt
```

Anotamos el contenido.

### Fase B · Snapshot

1. Verificar Guest Agent si el snapshot será en vivo.
2. Abrir **Snapshots**.
3. Crear uno con descripción explícita:

```text
antes-de-modificar-estado-dia3
```

4. Incluir el disco del sistema.
5. No guardar memoria.
6. Esperar a que desaparezca `Image Locked`.

### Fase C · Modificación

```bash
echo MODIFICADO | sudo tee /root/estado-dia3.txt
sudo cat /root/estado-dia3.txt
```

### Fase D · Restauración controlada

1. Apagar ordenadamente la VM.
2. Seleccionar el snapshot.
3. Ejecutar **Preview**.
4. Arrancar la VM y comprobar el fichero.
5. Apagarla de nuevo.
6. Elegir conscientemente:
   - **Commit** para conservar el estado restaurado;
   - **Undo** para volver al estado anterior al Preview.

Antes de pulsar, el alumno debe decir qué resultado espera.

### Fase E · Limpieza

Cuando el snapshot ya no sea necesario, se elimina de forma controlada. El merge puede consumir E/S y mantener la imagen bloqueada durante un tiempo.

No se inicia otra operación de disco hasta que finalice.

## Efectos de acumular snapshots

- cadenas más largas;
- más consumo de capacidad;
- más trabajo para merges;
- posibles impactos de rendimiento;
- más dificultad para interpretar dependencias;
- mayor ventana de riesgo operativo.

Una política sensata define propósito, duración, responsable y eliminación.

## Clonar desde un snapshot

OLVM permite crear otra VM a partir de un snapshot. Eso no restaura la VM original: crea un nuevo objeto con identidad propia.

```text
Restore/Commit → cambia el estado de la VM original
Clone snapshot → crea otra VM
```

En el laboratorio solo lo mostrará el instructor si hay memoria y almacenamiento suficientes.

---

# Bloque 4 · Templates, personalización y pools

## De VM a template

Un template es una imagen y configuración base reutilizable para crear VMs coherentes.

Puede capturar:

- discos y software instalado;
- CPU y memoria predeterminadas;
- tipo de sistema operativo;
- tipo de vNIC y controladores;
- opciones de consola y dispositivos;
- configuración inicial reutilizable.

No debe capturar identidades únicas que luego se repitan.

## Sellado

Sellar una VM significa retirar o preparar datos específicos antes de convertirla en template.

Según sistema operativo y procedimiento pueden incluir:

- machine-id;
- claves SSH de host;
- leases DHCP;
- hostname;
- reglas persistentes de red;
- estado de cloud-init;
- logs e historial;
- identificadores propios de Windows mediante Sysprep.

El objetivo es:

> **Muchas VMs con el mismo punto de partida, pero no muchas máquinas con la misma identidad.**

No ejecutaremos `virt-sysprep` sobre una VM del alumno durante la clase. El sellado es destructivo respecto a la identidad del molde y se hace sobre una VM preparada para ello.

## Nuestra instalación

El laboratorio dispone de imágenes y VMs base ligeras, entre ellas CirrOS, Alpine, Debian y AlmaLinux. También existe una preparación específica para el aula.

Los playbooks reflejan decisiones relevantes:

- CirrOS es la opción mínima para ciclo de vida y consola.
- Las imágenes cloud utilizan cloud-init cuando está verificado.
- El molde del aula elimina identidad antes de convertirse en template.
- Los alumnos ya disponen de VMs creadas para evitar una tormenta de clones.

No utilizaremos las VMs `*-base` como objetos de práctica.

## Crear una VM desde template

El proceso conceptual es:

```text
Template
   ↓
nueva definición de VM
   ↓
discos dependientes o clonados
   ↓
nueva MAC
   ↓
personalización inicial
```

Las VMs creadas desde un template reciben MAC independientes. La identidad dentro del sistema operativo debe regenerarse o personalizarse mediante el proceso de sellado y primer arranque.

## Thin/dependiente frente a clone/independiente

### VM dependiente de la imagen del template

- creación rápida;
- menor consumo inicial;
- utiliza una base de solo lectura y capas copy-on-write;
- mantiene dependencia de la imagen del template;
- la cadena y sus limitaciones deben entenderse.

### Clone independiente

- copia independiente de los discos;
- tarda y ocupa más;
- reduce la dependencia de la imagen base;
- puede ser preferible para cargas con vida larga o requisitos operativos concretos.

No confundimos **thin provisioning** con **linked clone**. Uno describe asignación de bloques; el otro, dependencia de una imagen base.

## Cloud-init

Cloud-init personaliza una imagen preparada durante los primeros arranques.

OLVM puede proporcionar mediante Initial Run datos como:

- hostname;
- usuario y contraseña;
- clave SSH;
- configuración de red;
- timezone;
- script o user-data.

Camino conceptual:

```text
Datos Initial Run en Engine
        ↓
ConfigDrive / metadata presentada a la VM
        ↓
cloud-init dentro del invitado
        ↓
configuración del sistema operativo
```

Si la VM no aplica los datos, investigamos dentro del invitado:

```bash
cloud-init status --long
sudo journalctl -u cloud-init --no-pager
sudo tail -n 100 /var/log/cloud-init-output.log
```

Que OLVM haya creado correctamente el ConfigDrive no demuestra que cloud-init haya aceptado y aplicado la configuración.

Comparación aproximada:

```text
cloud-init + Initial Run ≈ Guest Customization para Linux
Sysprep                  ≈ personalización/generalización de Windows
```

## Práctica 5 · Desplegar una VM de demostración

Solo el instructor crea `demo-dia3`.

1. Seleccionar un template verificado.
2. Elegir el Cluster `Curso`.
3. Asignar memoria y vCPU mínimos.
4. Revisar el Storage Domain y la política de disco.
5. Seleccionar el perfil vNIC `alumnos`.
6. Configurar un hostname nuevo mediante Initial Run si la imagen lo admite.
7. Crear la VM y observar `Image Locked`.
8. Arrancarla cuando los discos estén listos.
9. Comprobar consola, red e información del Guest Agent.

Si cloud-init falla, no se improvisa cambiando diez parámetros. Conservamos la evidencia y localizamos si falló:

- entrega de metadata;
- datasource;
- red del invitado;
- sintaxis del user-data;
- servicio cloud-init.

## VM Pool

Un pool es un conjunto de VMs creadas desde un mismo template y ofrecidas bajo demanda a usuarios autorizados.

```text
Template
   ↓
VM Pool
 ├── VM disponible
 ├── VM disponible
 └── VM asignada temporalmente a un usuario
```

Puede ser útil para:

- aulas;
- escritorios virtuales;
- puestos de atención;
- laboratorios efímeros;
- usuarios que necesitan “una VM”, no una VM con nombre fijo.

## Stateless

Cuando se accede a VMs de pool desde VM Portal, el diseño está orientado a entregar la VM desde su estado base y descartar los cambios temporales según el ciclo de asignación.

No se utiliza para guardar información importante dentro de la VM sin almacenamiento externo o persistente.

## VMs prearrancadas

Un pool puede mantener algunas VMs preparadas para reducir el tiempo de espera del usuario. El beneficio tiene un coste:

- consumen memoria y CPU mientras esperan;
- ocupan capacidad del clúster;
- deben dimensionarse según demanda real.

## Por qué el aula actual no utiliza un pool como mecanismo principal

El laboratorio entrega una VM concreta a cada alumno con `UserVmManager`, lo que permite practicar snapshots y administración de su VM.

Un pool está pensado para asignación bajo demanda y sus permisos predeterminados son más limitados. Además, el laboratorio anidado tiene poco margen para mantener VMs prearrancadas.

El pool se explicará y, como máximo, se demostrará con dos VMs ligeras. No sustituiremos las VMs actuales del aula.

## OVA

Un OVA empaqueta una máquina virtual para transporte o intercambio. No debe confundirse con:

- snapshot;
- template dentro del mismo Engine;
- backup del Engine;
- copia de seguridad de todos los datos de una aplicación.

La exportación/importación se deja como demostración opcional si el tiempo y la capacidad lo permiten.

## Tabla que debe quedar clara

| Objeto | Finalidad principal | ¿Nuevo objeto? | Dependencia típica |
|---|---|---:|---|
| Snapshot | Volver a un punto anterior | No necesariamente | Cadena de la VM original |
| Clone desde snapshot | Crear otra VM desde ese estado | Sí | Según tipo de clon |
| Template | Base administrada y reutilizable | Sí | Imagen base de solo lectura |
| VM desde template | Desplegar una instancia | Sí | Dependiente o independiente |
| Pool | Entregar VMs bajo demanda | Varias VMs | Template y política del pool |
| OVA | Transportar una VM/appliance | Fichero exportado | Formato de intercambio |
| Backup | Recuperarse incluso perdiendo el original | Copia externa | Repositorio y política de backup |

---

# Bloque 5 · Usuarios, roles, permisos y cuotas

## La ecuación de autorización

En OLVM un permiso se entiende así:

```text
usuario o grupo + rol + objeto = permiso
```

Ejemplo:

```text
alumno1 + UserVmManager + vm-alumno1
```

Significa que `alumno1` puede realizar sobre `vm-alumno1` las acciones incluidas en `UserVmManager`.

No significa que tenga ese rol sobre todas las VMs.

## Privilegio, rol y permiso

- **Privilegio:** acción elemental autorizable.
- **Rol:** conjunto de privilegios.
- **Permiso:** asignación de un rol a un usuario/grupo sobre un objeto.

No asignamos normalmente cientos de privilegios uno a uno. Construimos o reutilizamos roles y los aplicamos con un alcance.

## Roles administrativos y roles de usuario

### Roles administrativos

Permiten administrar objetos desde Administration Portal.

Ejemplos:

- `SuperUser`;
- `DataCenterAdmin`;
- `ClusterAdmin`;
- `NetworkAdmin`;
- otros roles especializados.

### Roles de usuario

Permiten consumir o administrar recursos desde VM Portal dentro del alcance concedido.

Ejemplos:

- `UserRole`: acceso y uso de VMs y pools según permisos;
- `UserVmManager`: administración de una VM concreta, incluidos snapshots;
- `PowerUserRole`: creación y administración de VMs/templates dentro del alcance;
- `VmCreator`;
- `DiskCreator`;
- `TemplateCreator`;
- `VnicProfileUser`.

Los nombres no se memorizan aislados. Abrimos el rol y revisamos sus privilegios reales en la versión instalada.

## Alcance e herencia

Un permiso puede aplicarse en distintos niveles:

```text
Sistema
 └── Data Center
      ├── Cluster
      │    └── VM
      ├── Storage Domain
      └── Logical Network / vNIC Profile
```

Un permiso superior puede heredarse hacia objetos inferiores. Por eso conceder un rol aparentemente razonable en `System` puede dar acceso a todo el entorno.

Siempre preguntamos:

1. ¿A quién?
2. ¿Qué rol?
3. ¿Sobre qué objeto?
4. ¿Qué se hereda?

## Situación real del laboratorio

Los playbooks conceden `UserVmManager` a cada alumno sobre su VM. También configuran actualmente un rol administrativo amplio para facilitar un curso de administración.

Consecuencia:

- el permiso fino sirve como ejemplo correcto;
- el rol administrativo amplio puede hacer que el alumno vea y modifique más objetos;
- no podemos confiar en que el portal impida tocar la VM de otro alumno;
- en producción no concederíamos `SuperUser` como solución general de autoservicio.

No retiraremos estos permisos durante la clase sin haber validado previamente el acceso alternativo.

## Práctica 6 · Leer un permiso

Sin modificar:

1. Abrir `vm-alumnoN`.
2. Entrar en **Permissions**.
3. Localizar el usuario correspondiente.
4. Identificar `UserVmManager`.
5. Preguntar si el permiso es directo o heredado.
6. Abrir la definición del rol y revisar acciones.
7. Comparar lo que vería ese usuario sin el rol administrativo global.

El objetivo no es crear usuarios nuevos, sino leer correctamente una autorización.

## VM Portal

VM Portal presenta al usuario los recursos que sus permisos le permiten utilizar.

No es una versión reducida técnicamente idéntica a Administration Portal. Tiene otro propósito:

- arrancar y apagar VMs autorizadas;
- abrir consola;
- crear o editar cuando el rol lo permita;
- solicitar VMs de pools;
- trabajar sin exponer toda la administración física.

Comparación aproximada:

```text
Administration Portal ≈ vSphere Client para administradores
VM Portal             ≈ portal de autoservicio para consumidores
```

## Cuota

Una cuota limita el consumo de recursos dentro de un Data Center.

Puede controlar:

- vCPU;
- memoria;
- almacenamiento global o por Storage Domain.

La cuota no contiene privilegios. Un usuario puede tener permiso para crear una VM y no disponer de cuota suficiente para consumir los recursos solicitados.

```text
Permiso = ¿puedes realizar la acción?
Cuota   = ¿cuánto recurso puedes consumir?
QoS     = ¿a qué ritmo o con qué prioridad puede usarlo?
```

## Modos de cuota

- **Disabled:** no se aplican límites de cuota.
- **Audit:** se evalúan y registran incumplimientos sin utilizarlos como bloqueo estricto.
- **Enforced:** una operación que supera la cuota puede ser rechazada.

Antes de diseñar cuotas comprobamos el modo del Data Center. Crear una cuota en un Data Center con el modo deshabilitado no produce el control esperado.

## Nuestra instalación

Los parámetros del laboratorio no crean una cuota OLVM para cada alumno. El control práctico se basa en VMs ya dimensionadas y en no ejecutar demasiadas cargas simultáneamente.

No afirmamos el modo actual del Data Center sin abrirlo y comprobarlo.

## Caso de razonamiento

Un usuario puede ver una plantilla, pero al crear una VM recibe un error.

Comprobamos por separado:

- permiso para usar la plantilla;
- permiso para crear VM en el alcance;
- permiso para crear disco en el Storage Domain;
- permiso para utilizar el vNIC Profile;
- cuota disponible;
- capacidad real de host y almacenamiento.

“Tiene acceso a la plantilla” no demuestra que cumpla las demás condiciones.

---

# Bloque 6 · Live migration

## La operación que une los tres días

Una live migration mueve una VM encendida entre hosts del mismo Cluster minimizando la interrupción.

Es el equivalente conceptual de **vMotion**. La equivalencia acaba ahí: para diagnosticarla utilizaremos los roles de red, estados, eventos y componentes propios de OLVM.

```text
Antes
olvm-host1: proceso QEMU + RAM activa
olvm-host2: destino preparado

Durante
RAM y estado → red de migración/management → destino

Después
olvm-host1: proceso retirado
olvm-host2: proceso QEMU activo
```

## Qué se mueve

- contenido de memoria;
- estado de CPU y dispositivos virtuales;
- ejecución del proceso al host destino;
- puerto TAP/vnet correspondiente a la vNIC.

## Qué permanece

- definición de la VM en Engine;
- identidad y UUID;
- MAC de sus vNICs;
- discos en el Storage Domain NFS compartido;
- Logical Network a la que está conectada;
- permisos.

Con almacenamiento compartido, la migración de computación no copia el disco al otro host. Ambos hosts ya pueden acceder al mismo Storage Domain.

## Quién participa

```text
Engine            → valida y coordina
VDSM origen       → prepara y ejecuta la salida
VDSM destino      → prepara y ejecuta la recepción
libvirt/QEMU      → transfieren estado de ejecución
red de migración  → transporta memoria/estado
NFS compartido    → mantiene los discos accesibles
```

El SPM no se convierte en proxy de memoria ni de E/S de la VM.

## Requisitos básicos

- hosts `Up` y en el mismo Cluster;
- CPU compatible;
- memoria y capacidad disponibles en destino;
- mismas redes y VLAN necesarias;
- acceso al Storage Domain de la VM;
- modo de migración permitido;
- ausencia de pinning o dispositivos incompatibles;
- conectividad de migración.

Si no existe una red con rol de migración, se utiliza normalmente la red de gestión.

## Elementos que pueden bloquearla

- VM fijada a un host;
- Direct LUN o dispositivo host no disponible en destino;
- passthrough/SR-IOV sin diseño migrable;
- red lógica ausente o fuera de sincronía;
- CPU incompatible;
- falta de RAM;
- almacenamiento no compartido;
- otra operación que mantiene la imagen bloqueada.

## Práctica 7 · Migración controlada

Solo el instructor ejecuta la migración.

### Antes

1. Elegir una VM pequeña y prescindible.
2. Confirmar Guest Agent y conectividad.
3. Anotar host actual.
4. Confirmar que ambos hosts están `Up`.
5. Confirmar acceso al mismo Storage Domain y a la red `alumnos`.
6. Evitar que la VM tenga una operación de snapshot o disco activa.

Dentro de la VM, si existe conectividad:

```bash
ping -i 0.2 IP_DE_PRUEBA
```

### Durante

- Pulsar **Migrate**.
- Elegir destino explícito para la demostración.
- Observar progreso y eventos.
- Comprobar si el ping pierde paquetes.
- No iniciar otra migración simultánea.

### Después

- Comprobar el nuevo host en el portal.
- Verificar que la VM conserva MAC e IP.
- Verificar que el disco continúa en el mismo Storage Domain.
- Comprobar aplicación o servicio.
- Relacionar el nuevo TAP/vnet con el bridge del destino si se dispone de acceso al host.

## Qué demuestra una migración correcta

Demuestra que, para esa VM y en ese momento:

- CPU y recursos eran compatibles;
- el destino accedía al storage;
- la red lógica estaba disponible;
- el camino de migración funcionaba;
- VDSM/libvirt coordinaron la operación.

No demuestra que HA y fencing estén correctamente configurados. En nuestro laboratorio la gestión de energía no está habilitada porque los hosts OLVM son VMs sin BMC real.

---

# Caso integrado

Una VM creada desde template arranca, pero después de migrarla:

- conserva su disco;
- aparece `Up`;
- Engine muestra la misma MAC;
- no recibe IP;
- el Guest Agent deja de mostrar información.

## Lo que sabemos

- la creación desde template produjo una definición utilizable;
- la VM ha podido arrancar en destino;
- el Storage Domain es accesible;
- la definición y el disco sobreviven a la migración.

## Lo que investigamos

1. ¿Existe la Logical Network en el host destino?
2. ¿Está sincronizada y sobre el uplink correcto?
3. ¿La vNIC está linked y utiliza el perfil esperado?
4. ¿Aparece el TAP en el bridge del destino?
5. ¿Llega DHCP a ese host?
6. ¿El sistema invitado conserva configuración de red?
7. ¿Está activo el Guest Agent o solo dejó de tener red?

## Lo que no culpamos primero

- el SPM;
- el template por el mero hecho de existir;
- la QoS de almacenamiento;
- el Engine como router de paquetes;
- el NFS si la VM ha arrancado correctamente.

La operación une los caminos anteriores:

```text
DEFINICIÓN
Template → VM → permisos

STORAGE
VM → Virtual Disk → NFS compartido

RED
vNIC → perfil → Logical Network → bridge del nuevo host

CONTROL
Engine → VDSM origen/destino → libvirt/QEMU
```

---

# Resumen para la pizarra

## Ciclo de vida

```text
Engine conserva definición y políticas
Host ejecuta QEMU/KVM
Guest Agent informa desde dentro
```

## Dispositivo

```text
OLVM adjunta Virtual Disk
  → QEMU presenta dispositivo
    → invitado detecta bloque
      → invitado crea filesystem
```

## Imagen

```text
Snapshot = volver atrás
Template = crear de nuevo
Pool     = entregar bajo demanda
Backup   = sobrevivir a perder el original
```

## Autorización

```text
usuario/grupo + rol + objeto = permiso

permiso ≠ cuota ≠ QoS
```

## Migración

```text
se mueve:    RAM + estado + ejecución
permanece:   disco NFS + identidad + MAC + definición
```

---

# Guion de tiempos para el instructor

Este material no se lee literalmente. Alternamos explicación, portal, consola, terminal y preguntas.

## Primer tramo · 0:00–1:50

- 0:00–0:10: tres preguntas de recuperación del día 2.
- 0:10–0:20: perfil/QoS y rol de red, solo como cierre.
- 0:20–0:40: definición frente a proceso; estados.
- 0:40–1:10: ficha completa de una VM.
- 1:10–1:25: Guest Agent frente a VirtIO y VDSM.
- 1:25–1:50: hot plug de disco y comprobación en invitado.

Pregunta de control:

> Si OLVM dice que el disco está conectado, ¿demuestra que la VM lo ha formateado y montado?

## Pausa · 1:50–2:05

## Segundo tramo · 2:05–3:00

- 2:05–2:20: snapshot, cadena y consistencia.
- 2:20–2:45: práctica de creación, modificación y Preview.
- 2:45–3:00: Commit/Undo, limpieza y snapshot frente a backup.

## Tercer tramo · 3:00–3:50

- 3:00–3:15: template y sellado.
- 3:15–3:30: VM dependiente frente a clone.
- 3:30–3:40: cloud-init y evidencias de fallo.
- 3:40–3:50: pool, stateless y prearrancadas.

## Pausa · 3:50–4:05

## Cuarto tramo · 4:05–4:45

- 4:05–4:20: usuario, rol, objeto y herencia.
- 4:20–4:30: situación real de permisos del laboratorio.
- 4:30–4:38: VM Portal.
- 4:38–4:45: cuota frente a permiso y QoS.

## Migración y cierre · 4:45–5:00

- 4:45–4:55: migración controlada de una VM ligera.
- 4:55–5:00: cinco preguntas rápidas.

Si una operación de snapshot, template o migración tarda más de lo previsto, se utiliza el tiempo para leer eventos. No iniciamos una segunda operación para “desatascar” la primera.

---

# Cinco preguntas orales para terminar

1. ¿Qué diferencia hay entre una VM apagada y una inexistente?  
   **La apagada conserva su definición, discos, redes, políticas y permisos.**

2. ¿Guest Agent y VDSM son el mismo agente?  
   **No. Guest Agent vive dentro de la VM; VDSM vive en el host.**

3. ¿Por qué un snapshot no es un backup?  
   **Porque depende del almacenamiento y de la cadena original y no protege frente a su pérdida.**

4. ¿Cómo se construye un permiso?  
   **Usuario o grupo, más rol, más objeto.**

5. ¿Qué ocurre con el disco NFS durante una live migration normal?  
   **Permanece en el Storage Domain compartido; se mueve la ejecución y la memoria.**

---

# Lo que debe quedar claro al terminar el día 3

```text
VM
├── definición y políticas en Engine
├── proceso QEMU/KVM en un host
├── discos en Storage Domains
├── vNICs en Logical Networks
├── información interna mediante Guest Agent
├── puntos de retorno mediante snapshots
├── despliegue repetible mediante templates/pools
└── acceso controlado mediante permisos y cuotas
```

Si el alumno puede seguir una VM desde el template hasta una migración, explicar cada dependencia y decir qué objeto autoriza cada operación, ya no está memorizando pantallas: está administrando OLVM.
