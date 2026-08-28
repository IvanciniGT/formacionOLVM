# Flujo completo de una máquina virtual en OLVM

Este documento sigue una VM desde su creación hasta su eliminación y separa dos ideas que no deben mezclarse:

- **La definición de la VM:** permanece registrada en Engine aunque esté apagada.
- **La ejecución de la VM:** existe como un proceso QEMU/KVM dentro de un host cuando está encendida.

La comparación aproximada con vSphere sería:

| OLVM | Equivalencia aproximada en vSphere |
|---|---|
| Definición de VM en Engine | VM registrada en vCenter |
| Proceso QEMU/KVM en un host | VM ejecutándose en un host ESXi |
| Live migration | vMotion |
| Template | VM Template |
| QEMU Guest Agent | Parte de VMware Tools |

---

# 1. Flujo general del ciclo de vida

```mermaid
flowchart TD
    Inicio([Necesitamos<br>una VM])

    Inicio --> Origen{¿De dónde<br>parte?}
    Origen -->|Plantilla| Template[Seleccionar<br>template]
    Origen -->|Instalación<br>nueva| Blank[Crear VM<br>Blank]
    Origen -->|Disco<br>existente| Imagen[Subir o importar<br> QCOW2/RAW]
    Origen -->|Appliance| OVA[Importar<br>OVA]

    Template --> Definicion
    Blank --> Definicion
    Imagen --> Definicion
    OVA --> Definicion

    Definicion[Crear definición<br>en Engine]
    Definicion --> Recursos[Asignar CPU, memoria,<br> discos y vNIC]
    Recursos --> Politicas[Aplicar permisos, <br>cuotas y políticas]
    Politicas --> Down[VM en estado<br>Down]

    Down --> Arrancar{¿Arrancar?}
    Arrancar -->|Sí| Validar[Engine valida <br>configuración y capacidad]
    Arrancar -->|No| Espera[La definición <br>permanece guardada]
    Espera --> Arrancar

    Validar --> Host[Scheduler selecciona <br>un host]
    Host --> VDSM[Engine ordena el <br>arranque a VDSM]
    VDSM --> QEMU[VDSM y libvirt crean <br>el proceso QEMU/KVM]
    QEMU --> Up[VM en estado<br>Up]

    Up --> Operacion{Operación durante<br>su vida}
    Operacion -->|Modificar| Hotplug[Hot plug o<br> cambio pendiente]
    Operacion -->|Proteger<br>un punto| Snapshot[Crear<br>snapshot]
    Operacion -->|Mover<br>ejecución| Migrar[Live<br>migration]
    Operacion -->|Apagar| Shutdown[Apagado<br>ordenado]

    Hotplug --> Up
    Snapshot --> Up
    Migrar --> Up
    Shutdown --> Down

    Down --> Final{¿Qué hacemos<br>con ella?}
    Final -->|Volver<br>a usar| Arrancar
    Final -->|Usar como<br>molde| NuevaPlantilla[Sellar y crear<br>template]
    Final -->|Retirar| Borrar[Borrar VM y <br>decidir qué hacer con<br> sus discos]

    NuevaPlantilla --> Template
    Borrar --> Fin([Fin del<br>ciclo])
```

La VM puede recorrer el circuito `Down → Up → Down` muchas veces. Apagarla no elimina su definición ni sus discos.

---

# 2. Plano de control y plano de datos

Engine coordina la VM, pero no transporta continuamente sus paquetes ni sus bloques de disco.

```mermaid
flowchart TB
    Admin[Administrador<br>o usuario]

    subgraph Control[Plano de<br>control]
        direction LR
        Portal[Portal o<br>API]
        Engine[OLVM Engine]
        DB[(Base de datos<br>de Engine)]
        VDSM[VDSM del<br>host]
        Libvirt[libvirt]

        Portal -->|solicitud| Engine
        Engine -->|consulta<br>y actualiza| DB
        Engine -->|orden| VDSM
        VDSM -->|configura| Libvirt
    end

    subgraph Host[Host elegido:<br>ejecución de la VM]
        direction LR
        QEMU[Proceso<br>QEMU/KVM]
        TAP[TAP o<br>vnet]
        Bridge[Bridge<br>Linux]

        QEMU -->|tramas de<br>ida y vuelta| TAP
        TAP --> Bridge
    end

    subgraph Datos[Infraestructura<br>de datos]
        direction LR
        NFS[(Storage Domain<br>NFS)]
        NIC[NIC o bond<br>del host]
        LAN[LAN]

        NIC --> LAN
    end

    Admin --> Portal
    Libvirt -->|crea y<br>controla| QEMU
    QEMU -->|lecturas y<br>escrituras| NFS
    Bridge -->|tramas de<br>ida y vuelta| NIC
```

## Qué circula por cada camino

| Camino | Qué circula |
|---|---|
| Portal → Engine → VDSM → libvirt | Órdenes, configuración, estados y resultados |
| QEMU ↔ Storage Domain NFS | Lecturas y escrituras del disco virtual |
| vNIC → TAP/vnet → bridge → NIC | Tramas de red de la VM |

El Engine no lee cada bloque del disco ni conmuta cada trama de la VM.

---

# 3. Qué ocurre cuando pulsamos Ejecutar

```mermaid
sequenceDiagram
    actor Usuario
    participant Engine as OLVM Engine
    participant Scheduler
    participant VDSM as VDSM<br>del host
    participant Storage as Storage<br>Domain NFS
    participant QEMU as QEMU/KVM
    participant Invitado as Sistema<br>invitado

    Usuario->>Engine: Ejecutar VM
    Engine->>Engine: Comprobar permisos,<br>estado y configuración
    Engine->>Scheduler: Buscar un host<br>compatible
    Scheduler-->>Engine: Host<br>seleccionado
    Engine->>VDSM: Preparar y arrancar<br>la VM
    VDSM->>Storage: Comprobar acceso<br>a los discos
    Storage-->>VDSM: Discos<br>disponibles
    VDSM->>QEMU: Crear proceso y<br>dispositivos virtuales
    QEMU->>Invitado: Presentar CPU, RAM,<br>discos, vNIC y firmware
    Invitado-->>QEMU: Arranque del<br>sistema operativo
    QEMU-->>VDSM: Estado de<br>ejecución
    VDSM-->>Engine: VM Up
    Engine-->>Usuario: Estado y<br>host actual
```

## Validaciones principales

Antes de arrancar, OLVM necesita encontrar una combinación válida:

```mermaid
flowchart TD
    Run[Solicitud de<br>arranque] --> Permiso{¿Usuario<br>autorizado?}
    Permiso -->|No| ErrorPermiso[Rechazar]
    Permiso -->|Sí| Estado{¿VM arrancable?}
    Estado -->|No| ErrorEstado[Rechazar o<br>esperar tarea]
    Estado -->|Sí| CPU{¿CPU<br>compatible?}
    CPU -->|No| SinHost[No existe<br>host válido]
    CPU -->|Sí| RAM{¿RAM<br>suficiente?}
    RAM -->|No| SinHost
    RAM -->|Sí| Storage{¿Storage<br>accesible?}
    Storage -->|No| SinHost
    Storage -->|Sí| Red{¿Redes<br>disponibles?}
    Red -->|No| SinHost
    Red -->|Sí| Reglas{¿Afinidad y políticas <br>cumplidas?}
    Reglas -->|No| SinHost
    Reglas -->|Sí| Arranque[Ordenar arranque<br>a VDSM]
```

Por eso «el host tiene CPU libre» no demuestra que pueda ejecutar la VM. También deben encajar storage, redes, afinidad, permisos y compatibilidad.

---

# 4. Estados principales

```mermaid
stateDiagram-v2
    [*] --> Down: VM creada

    Down --> PoweringUp: Ejecutar
    PoweringUp --> Up: QEMU arrancado
    PoweringUp --> Down: Error de<br>arranque

    Up --> PoweringDown: Apagado<br>ordenado
    PoweringDown --> Down: Invitado<br>detenido

    Up --> Paused: Pausa o<br>error de E/S
    Paused --> Up: Reanudar
    Paused --> Down: Detención

    Up --> Migrating: Live<br>migration
    Migrating --> Up: Ejecución<br>en destino

    Up --> Down: Power Off
    Down --> [*]: Eliminar VM
```

Este diagrama está simplificado. Una operación de discos puede mostrar además estados como `Image Locked` mientras la VM continúa teniendo su propio estado de ejecución.

## Down no significa inexistente

Cuando está `Down` permanecen:

- definición;
- discos;
- vNICs y MAC;
- permisos;
- snapshots;
- políticas;
- historial y eventos.

Lo que no existe es el proceso QEMU ejecutándola en un host.

---

# 5. Creación desde una plantilla

```mermaid
flowchart TD
    Template[Template<br>alma9-aula]
    Template --> Tipo{Asignación de<br> almacenamiento}

    Tipo -->|Fino / Thin| Thin[Crear capa QCOW2<br> de cambios]
    Tipo -->|Clone| Clone[Copiar el disco de<br> la plantilla]

    Thin --> Dependiente[VM dependiente de <br>la imagen base]
    Clone --> Independiente[VM con disco<br>independiente]

    Dependiente --> Def[Crear definición<br>de la VM]
    Independiente --> Def

    Def --> MAC[Asignar nueva<br>MAC]
    MAC --> Inicial{¿Usar cloud-init?}
    Inicial -->|Sí| Metadata[Preparar hostname,<br> usuarios, claves y red]
    Inicial -->|No| SinPersonalizar[Usar configuración<br> ya existente]

    Metadata --> ConfigDrive[Presentar ConfigDrive<br>o metadata]
    ConfigDrive --> CloudInit[cloud-init actúa<br>dentro del invitado]

    CloudInit --> VM[VM personalizada]
    SinPersonalizar --> VM
```

## Diferencia esencial

| Modalidad | Creación y consumo inicial | Relación con la plantilla |
|---|---|---|
| **Thin** | Se crea rápidamente y ocupa poco espacio inicialmente | Depende de la imagen base de la plantilla |
| **Clone** | Tarda más y consume el espacio de la copia | Copia el disco y queda independiente |

El template debe estar sellado para no duplicar hostname, claves SSH, `machine-id` u otras identidades.

---

# 6. Funcionamiento de la VM encendida

```mermaid
flowchart LR
    subgraph Invitado[Máquina<br>virtual]
        App[Aplicación]
        SO[Sistema<br>operativo]
        GA[QEMU<br>Guest Agent]
        VDisk[Dispositivo<br>de disco]
        VNIC[vNIC]
    end

    subgraph Host[Host OLVM]
        QEMU[QEMU/KVM]
        Libvirt[libvirt]
        VDSM[VDSM]
        TAP[TAP /<br>vnet]
        Bridge[Bridge]
    end

    Engine[Engine]
    NFS[(NFS)]
    Red[Red<br>física]

    App --> SO
    SO --> VDisk
    SO --> VNIC
    VDisk --> QEMU
    VNIC --> QEMU
    QEMU --> NFS
    QEMU --> TAP
    TAP --> Bridge
    Bridge --> Red

    GA -->|canal de<br>gestión| QEMU
    Engine --> VDSM
    VDSM --> Libvirt
    Libvirt --> QEMU
```

## Guest Agent frente a VDSM

| Componente | Dónde está | Función principal |
|---|---|---|
| **QEMU Guest Agent** | Dentro de la VM | Informa sobre IP y estado del invitado; permite apagado coordinado y otras operaciones |
| **VDSM** | En el host OLVM | Recibe órdenes de Engine y administra libvirt, las VMs, el almacenamiento y la red del host |

Una VM puede funcionar sin Guest Agent, pero Engine conocerá peor lo que sucede dentro de ella.

---

# 7. Flujo de un snapshot

```mermaid
flowchart TD
    Inicio[VM con<br>estado actual]
    Inicio --> Consistencia{¿Qué consistencia<br>necesitamos?}

    Consistencia -->|VM apagada| Apagada[Estado de discos<br>estable]
    Consistencia -->|VM encendida| Guest[Coordinar con<br>Guest Agent]
    Consistencia -->|Aplicación crítica| App[Usar procedimiento o<br>hook de aplicación]

    Apagada --> Crear[Crear snapshot]
    Guest --> Crear
    App --> Crear

    Crear --> Locked[Operación de imagen<br>Image Locked]
    Locked --> Cadena[Crear nueva relación<br>copy-on-write]
    Cadena --> Continuar[La VM continúa usando<br>una nueva capa]

    Continuar --> Decision{¿Qué queremos<br>hacer?}
    Decision -->|Probar estado<br>anterior| Preview[Preview]
    Preview --> Verificar[Arrancar y<br>verificar]
    Verificar -->|Aceptar| Commit[Commit]
    Verificar -->|Volver al<br>estado previo| Undo[Undo]

    Decision -->|Crear otra VM| Clone[Clone desde<br>snapshot]
    Decision -->|Ya no se<br>necesita| Delete[Eliminar y<br>fusionar]
```

## Snapshot no es backup

```mermaid
flowchart LR
    VM[VM<br>original]
    Snapshot[Snapshot]
    Storage[(Mismo Storage<br>Domain)]
    Fallo[Fallo completo<br>del NFS]

    VM --> Storage
    Snapshot --> Storage
    Fallo --> Perdida[Se pierden original<br>y snapshot]
    Storage --> Perdida
```

El snapshot depende de la cadena y del almacenamiento original. Un backup necesita otra copia, retención y un procedimiento probado de restauración.

---

# 8. Flujo de una live migration

```mermaid
sequenceDiagram
    actor Admin as Administrador
    participant Engine
    participant Origen as VDSM host<br>origen
    participant Destino as VDSM host<br>destino
    participant NFS as NFS<br>compartido
    participant Red as Red de<br>migración

    Admin->>Engine: Migrar VM
    Engine->>Engine: Validar CPU, RAM,<br>redes, storage y políticas
    Engine->>Destino: Preparar<br>recepción
    Destino->>NFS: Comprobar acceso<br>al mismo disco
    NFS-->>Destino: Disco<br>accesible
    Destino-->>Engine: Destino<br>preparado
    Engine->>Origen: Iniciar<br>migración

    loop Copias iterativas
        Origen->>Red: Páginas de memoria<br>y estado
        Red->>Destino: Transferir al QEMU<br>de destino
    end

    Origen->>Destino: Estado final de CPU<br>y dispositivos
    Destino-->>Engine: VM activa<br>en destino
    Engine->>Origen: Retirar ejecución<br>de origen
    Engine-->>Admin: Migración<br>completada
```

## Qué se mueve y qué permanece

```mermaid
flowchart LR
    subgraph SeMueve[Se mueve al<br>host destino]
        RAM[Contenido<br>de RAM]
        CPU[Estado<br>de CPU]
        Devices[Estado de<br>dispositivos]
        Process[Nueva ejecución<br>QEMU]
        NewTap[Nuevo TAP<br>o vnet]
    end

    subgraph Permanece[Permanece como<br>objeto compartido]
        Disk[Disco<br>en NFS]
        UUID[UUID de<br>la VM]
        MAC[MAC de<br>la vNIC]
        Definition[Definición<br>en Engine]
        Permissions[Permisos]
    end
```

En una migración normal con NFS compartido no se copia el disco al host destino. Ambos hosts ya acceden al mismo Storage Domain.

---

# 9. Apagado y eliminación

## Apagado ordenado

```mermaid
sequenceDiagram
    actor Usuario
    participant Engine
    participant VDSM
    participant Guest as Sistema<br>invitado
    participant QEMU

    Usuario->>Engine: Shutdown
    Engine->>VDSM: Solicitar<br>apagado
    VDSM->>Guest: Orden mediante<br>Guest Agent/ACPI
    Guest->>Guest: Detener aplicaciones y<br>desmontar filesystems
    Guest-->>QEMU: Apagado<br>completado
    QEMU-->>VDSM: Proceso<br>finalizado
    VDSM-->>Engine: VM Down
```

## Power Off

`Power Off` fuerza la retirada de la ejecución. Es parecido a cortar la alimentación de un servidor y puede dejar datos o filesystems inconsistentes.

## Eliminar la VM

Al borrar debemos distinguir:

```mermaid
flowchart TD
    Remove[Eliminar<br>VM] --> Definition[Borrar definición<br>de Engine]
    Remove --> DiskDecision{¿Qué hacer con<br>sus discos?}
    DiskDecision -->|Borrar| DeleteDisk[Eliminar imágenes del<br>Storage Domain]
    DiskDecision -->|Conservar<br>o separar| Floating[Dejar disco flotante<br>cuando proceda]
```

Borrar una VM no debe hacerse hasta identificar sus discos, snapshots, permisos y posibles dependencias.

---

# 10. El flujo en nuestra instalación

```mermaid
flowchart TD
    Alumno[Alumno o<br>instructor]
    Portal[Portal<br>OLVM]

    subgraph Worker2[worker2]
        Engine[Engine]
    end

    subgraph Worker3[worker3]
        Host1[olvm-host1]
        Bridge1[br-olvm<br>exterior]
        USB1[NIC USB]
    end

    subgraph Worker4[worker4]
        Host2[olvm-host2]
        Bridge2[br-olvm<br>exterior]
        USB2[NIC USB]
    end

    Maestro[maestro:<br>NFS]
    CursoNFS[(curso-nfs /<br>curso-nfs-2)]
    LAN[LAN]

    Alumno --> Portal
    Portal --> Engine
    Engine -->|orden a<br>VDSM| Host1
    Engine -->|orden a<br>VDSM| Host2

    Host1 -->|acceso<br>NFS| CursoNFS
    Host2 -->|acceso<br>NFS| CursoNFS
    Maestro -->|exporta| CursoNFS

    Host1 -->|vNIC<br>alumnos| Bridge1
    Host2 -->|vNIC<br>alumnos| Bridge2
    Bridge1 --> USB1
    Bridge2 --> USB2
    USB1 --> LAN
    USB2 --> LAN
```

Una VM concreta se ejecutará en `olvm-host1` **o** en `olvm-host2`, no en los dos a la vez.

Su camino simplificado será:

| Camino | Recorrido simplificado |
|---|---|
| **Control** | Portal → Engine → VDSM del host elegido → libvirt → QEMU |
| **Disco** | QEMU → red interna → `curso-nfs`/`curso-nfs-2` → NFS de `maestro` |
| **Red de la VM** | vNIC → TAP/vnet → red lógica `alumnos` → NIC del host OLVM → `br-olvm` del worker exterior → NIC USB → LAN |

Durante una migración cambia el host que ejecuta QEMU y se crea el puerto TAP/vnet correspondiente en el destino. El disco continúa en NFS y la MAC de la VM permanece.

---

# 11. Reglas mentales para diagnosticar una VM

```mermaid
flowchart TD
    Problema[La VM no funciona<br>como esperamos]
    Problema --> Estado{¿Qué estado<br>muestra Engine?}
    Estado --> Definicion{¿La definición<br>es correcta?}
    Definicion --> Host{¿En qué host<br>se ejecuta?}
    Host --> Storage{¿Discos accesibles<br>y sin bloqueo?}
    Storage --> Red{¿vNIC, perfil, TAP<br>y red correctos?}
    Red --> Invitado{¿Qué ve el<br>sistema invitado?}
    Invitado --> Eventos[Correlacionar<br>eventos y logs]
```

Las preguntas se hacen en este orden:

1. ¿Existe la VM y en qué estado está?
2. ¿Cuál es su definición de CPU, memoria, discos y red?
3. ¿En qué host está ejecutándose?
4. ¿Tiene acceso al Storage Domain?
5. ¿Qué camino sigue su vNIC?
6. ¿Guest Agent informa correctamente?
7. ¿El problema está en OLVM, en el host o dentro del invitado?

La frase final es:

> **Engine decide y coordina; VDSM ejecuta la orden; QEMU/KVM ejecuta la VM; NFS guarda sus discos; la red del host transporta sus tramas.**
