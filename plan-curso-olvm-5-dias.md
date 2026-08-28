# Propuesta de reparto — Formación OLVM / Oracle 1Z0-1170

**Duración:** 5 días, del lunes 24 al viernes 28 de agosto de 2026  
**Horario:** 5 horas por día, 25 horas totales  
**Versión del documento:** borrador 0.1 para validación  
**Base técnica:** Oracle Linux Virtualization Manager 4.5 y objetivos del examen 1Z0-1170

## Criterio del reparto

El día 1 es principalmente teórico y no utiliza la instalación disponible. Primero se construye un modelo mental sólido de OLVM, su arquitectura, su ecosistema y sus herramientas. La plataforma existente empieza a utilizarse, como pronto, en el día 2. La instalación se desplaza al día 4, cuando el alumnado ya conoce los componentes que está instalando.

Cada jornada reserva **4 horas y 30 minutos de contenido y práctica** y **30 minutos de pausas**. Si las cinco horas indicadas son íntegramente lectivas, los 30 minutos podrán utilizarse para ampliar los laboratorios o resolver dudas.

El reparto no sigue literalmente el orden de la playlist oficial. Conserva todos sus bloques, pero los ordena de forma que los conceptos precedan a los procedimientos.

Como el alumnado ya conoce VMware vSphere, las explicaciones utilizarán comparaciones sistemáticas con vSphere como puente didáctico. Se indicarán siempre las diferencias para evitar presentar analogías aproximadas como equivalencias exactas.

---

## Día 1 — Fundamentos, arquitectura y ecosistema OLVM

**Objetivo del día:** que el alumnado pueda explicar qué es OLVM, qué administra, qué componentes intervienen y cómo se relacionan los principales objetos de la plataforma.

| Bloque | Duración | Contenido |
|---|---:|---|
| 1. OLVM en el ecosistema de virtualización | 45 min | Virtualización de servidor; diferencia entre KVM y OLVM; relación Oracle Linux, KVM, QEMU, libvirt, VDSM, oVirt y OLVM; diferencia frente al antiguo Oracle VM basado en Xen; primer mapa comparativo con VMware vSphere. |
| 2. Arquitectura y flujo de gestión | 55 min | Engine como plano de control; hosts como plano de ejecución; flujo Engine → VDSM → libvirt/QEMU-KVM → VM; Engine DB y Data Warehouse; instalación estándar frente a Self-Hosted Engine a nivel conceptual. |
| 3. Modelo de objetos de OLVM | 50 min | Data Center, Cluster, Host, VM y relaciones entre ellos; compatibilidad; recursos compartidos; estados básicos; comparación razonada con vCenter, vSphere Datacenter, vSphere Cluster y ESXi; papel del Storage Pool Manager y diferencias frente a los mecanismos de coordinación de almacenamiento de vSphere. |
| 4. Almacenamiento y red: primer mapa mental | 45 min | Storage físico → Storage Domain → disco virtual → filesystem invitado; NIC/bond/VLAN/bridge/logical network/vNIC; por qué una red lógica no es una VLAN. |
| 5. Herramientas, interfaces y consolidación conceptual | 75 min | Mapa funcional de Administration Portal, VM Portal, consola de VM, API REST, automatización y herramientas de diagnóstico; comparación con vSphere Client, vCenter APIs y acceso a hosts ESXi; qué problema resuelve cada interfaz; recorrido teórico de una operación desde el usuario hasta KVM; consolidación mediante esquemas y casos breves, sin acceder a la instalación. |

**Actividades y resultados (sin usar la instalación):**

- Dibujar una arquitectura OLVM de referencia.
- Clasificar componentes en plano de control, plano de ejecución, almacenamiento, red y observabilidad.
- Resolver ejercicios de relación entre Data Center, Cluster, Host, Storage Domain, Logical Network y VM.
- Seguir sobre un diagrama el recorrido de operaciones como crear, arrancar o migrar una VM.
- Asociar cada interfaz y herramienta con sus usuarios, responsabilidades y casos de uso.
- Completar una tabla OLVM ↔ vSphere que incluya equivalencias aproximadas y diferencias importantes.
- Elaborar una primera tabla “componente, responsabilidad, dónde se ejecuta”.

**Conceptos que deben quedar cerrados:** OLVM no es el hipervisor; Engine no ejecuta las VMs; VDSM no sustituye a KVM; un Data Center, un Cluster y un Host no son sinónimos; el SPM coordina operaciones de almacenamiento, pero el I/O normal de todas las VMs no atraviesa el SPM.

---

## Día 2 — Storage Domains y gestión de redes

**Objetivo del día:** comprender cómo OLVM presenta almacenamiento y conectividad a hosts y máquinas virtuales, y realizar operaciones controladas sobre la plataforma existente.

| Bloque | Duración | Contenido |
|---|---:|---|
| 1. Arquitectura de almacenamiento | 55 min | Data domains; file storage frente a block storage; NFS, iSCSI, Fibre Channel, GlusterFS y almacenamiento local; metadatos, imágenes, discos, snapshots e ISO; activación y asociación al Data Center. |
| 2. SPM y operaciones de almacenamiento | 45 min | Elección y función del SPM; coordinación de metadatos y operaciones serializadas; acceso de los hosts; almacenamiento compartido; restricciones de local storage; impacto sobre migración, scheduling, fencing y HA. |
| 3. Arquitectura de red | 55 min | NIC física, bond, VLAN, bridge, logical network y vNIC; redes de gestión, VMs, migración, display y almacenamiento; diseño, redundancia y separación de tráfico. |
| 4. Funciones avanzadas de red | 45 min | MAC pools y MAC específica; perfiles vNIC; SR-IOV, Physical Functions y Virtual Functions; rendimiento frente a movilidad y flexibilidad. |
| 5. Laboratorio de storage y networking | 70 min | Inspección y, si el entorno lo permite, creación de una red lógica y un recurso de almacenamiento de prueba; asociación al nivel correcto; validación desde host y VM; diagnóstico de estados no operativos. |

**Práctica/resultados:**

- Clasificar el almacenamiento disponible como archivo, bloque o local.
- Seguir un disco desde el backend hasta la VM.
- Identificar las redes lógicas, sus roles y su realización física en un host.
- Explicar qué funciones se pierden al usar local storage o determinadas configuraciones SR-IOV.
- Preparar el diseño del laboratorio iSCSI que se desarrollará como práctica ampliada.

---

## Día 3 — Máquinas virtuales, permisos, optimización y alta disponibilidad

**Objetivo del día:** administrar el ciclo de vida de una VM entendiendo sus dependencias, delegar acceso correctamente y distinguir rendimiento, migración y alta disponibilidad.

| Bloque | Duración | Contenido |
|---|---:|---|
| 1. VM, hardware virtual y almacenamiento | 55 min | Creación y ciclo de vida; vCPU, memoria, firmware y dispositivos; discos virtuales; interfaces VirtIO; QEMU Guest Agent; diferencia entre agente invitado y driver. |
| 2. Snapshot, clon, template, pool y OVA | 55 min | Diferencias conceptuales; dependencias de disco; thin/full provisioning cuando aplique; creación de templates; pools para escenarios VDI; exportación e importación. |
| 3. Usuarios, grupos, roles y cuotas | 50 min | RBAC: sujeto + rol + objeto; privilegios y permisos; alcance y herencia; roles administrativos y de usuario; VM Portal; cuotas como control de consumo, no como sustituto de la autorización. |
| 4. Optimización | 50 min | CPU y memoria; VirtIO; NUMA, pinning, ballooning, huge pages y sobreasignación únicamente donde estén soportados y sean aplicables; políticas de scheduling, afinidad y anti-afinidad; migración. |
| 5. HA, fencing y hot plug | 60 min | Host HA y VM HA; power management y fencing; prioridad de reinicio; live migration frente a reinicio tras fallo; requisitos de almacenamiento y red; dispositivos que pueden conectarse en caliente y restricciones por versión/SO invitado. |

**Práctica/resultados:**

- Crear o clonar una VM de laboratorio.
- Instalar/verificar guest agent y drivers VirtIO cuando corresponda.
- Ejecutar el flujo VM → snapshot → template → pool.
- Asignar un permiso con alcance limitado y comprobarlo desde VM Portal.
- Simular sobre papel y, si el laboratorio lo permite, observar migración, mantenimiento de host y reinicio HA.

---

## Día 4 — Instalación del Engine, hosts KVM y Self-Hosted Engine

**Objetivo del día:** entender y reproducir el despliegue después de conocer qué función cumple cada componente instalado.

| Bloque | Duración | Contenido |
|---|---:|---|
| 1. Diseño y requisitos previos | 50 min | Topologías estándar y Self-Hosted Engine; dimensionamiento; versiones compatibles; CPU/virtualización; DNS directo e inverso; FQDN, tiempo, certificados, repositorios, puertos y firewall. |
| 2. Instalación estándar del Engine | 60 min | Preparación de Oracle Linux; repositorios y paquetes; prechecks; `ovirt-engine`; `engine-setup`; base de datos, PKI y acceso inicial; validaciones posteriores. |
| 3. Preparación y alta de un host KVM | 50 min | Instalación mínima; kernel y repositorios; virtualización; red y almacenamiento; despliegue de VDSM; registro desde Administration Portal; estados de instalación y activación. |
| 4. Self-Hosted Engine | 50 min | Arquitectura, almacenamiento y agentes HA; despliegue; mínimo de hosts para HA; mantenimiento y recuperación conceptual de la Engine VM; diferencias frente al Engine independiente. |
| 5. Demostración y troubleshooting | 60 min | Reconstrucción guiada de la instalación existente o despliegue en máquinas separadas de práctica; revisión de DNS, certificados, firewall, repositorios, servicios y logs ante fallos típicos. |

**Práctica/resultados:**

- Documentar la topología instalada y justificarla.
- Ejecutar comprobaciones previas sin alterar la plataforma principal.
- Añadir un host de laboratorio si existe capacidad preparada para ello.
- Explicar el orden de operaciones y dónde buscar ante un fallo de instalación.

**Precaución:** la instalación o reinstalación real no debe realizarse sobre el Engine o los hosts que sostienen el laboratorio compartido. Se utilizarán nodos separados, nested virtualization o una demostración preparada.

---

## Día 5 — Eventos, logs, observabilidad y recuperación

**Objetivo del día:** diagnosticar la plataforma por capas, comprender sus datos operativos y ejecutar una estrategia básica de protección y recuperación del Manager.

| Bloque | Duración | Contenido |
|---|---:|---|
| 1. Eventos y notificaciones | 40 min | Eventos, tareas, auditoría y severidad; notificaciones; correo y SNMP traps; diferencia entre evento, métrica y log. |
| 2. Mapa de logs y diagnóstico | 65 min | Engine, VDSM, libvirt, QEMU, PostgreSQL, DWH y sistema; servicios responsables; método de diagnóstico desde el síntoma hasta la capa; correlación temporal e identificadores. |
| 3. Monitoring y observabilidad | 60 min | Engine DB frente a DWH; métricas históricas; PostgreSQL orientado al diagnóstico; Grafana; `ovirt-log-collector`; actividad, conexiones, bloqueos y tamaño sin convertir el bloque en un curso de DBA. |
| 4. Backup, restore y disaster recovery | 75 min | `engine-backup`; alcance, archivos y automatización; restauración sobre host limpio; certificados y base de datos; cambio de nombre del Engine; DWH remoto; estrategias active/passive y active/active a nivel conceptual. |
| 5. Caso integrado y repaso | 30 min | Caso de caída o degradación: eventos → logs → diagnóstico → decisión de recuperación; repaso por peso del examen y preguntas de conceptos/orden de operaciones. |

**Práctica/resultados:**

- Resolver una incidencia guiada utilizando eventos y logs.
- Localizar datos actuales frente a históricos.
- Generar y verificar un backup del Manager sin detener el servicio.
- Explicar un restore completo sobre un Engine limpio.
- Realizar una prueba final corta y corregirla razonadamente.

---

## Cobertura respecto al examen

| Objetivo oficial | Peso | Día principal | Refuerzo |
|---|---:|---:|---:|
| Arquitectura y componentes | 17 % | 1 | 2, 4 y 5 |
| Instalación de Engine y KVM Host | 16 % | 4 | 1 |
| Storage Domains y networking | 19 % | 2 | 1, 3 y 4 |
| Administración de VMs | 10 % | 3 | 2 |
| Usuarios y permisos | 9 % | 3 | 1 |
| Optimización, eventos y logs | 20 % | 3 y 5 | 1 |
| Recovery | 9 % | 5 | 4 |

La distribución reserva días completos a los bloques de mayor peso sin perder la secuencia pedagógica. El 20 % de optimización, eventos y logs se reparte entre los días 3 y 5 porque mezcla dos áreas operativas diferentes.

## Validaciones técnicas realizadas

- La documentación vigente de Oracle presenta **OLVM Release 4.5** como plataforma de gestión de entornos Oracle Linux KVM basada en oVirt.
- La jerarquía Data Center → Cluster → Host, junto con Storage Domains y Logical Networks, coincide con la arquitectura oficial.
- Oracle documenta Administration Portal, VM Portal y API REST como interfaces de la plataforma.
- La documentación actual mantiene instalaciones estándar y Self-Hosted Engine. El Engine estándar no debe compartir host con un KVM host gestionado por él.
- Oracle documenta NFS, iSCSI, Fibre Channel, GlusterFS y local storage, pero sus requisitos y limitaciones deben comprobarse contra la versión concreta del laboratorio.
- Oracle documenta `engine-backup`, Engine DB, DWH, Grafana, eventos, notificaciones y soluciones de disaster recovery, por lo que se mantienen en el día 5.
- Los requisitos exactos de Oracle Linux, kernel, compatibilidad de clúster y GlusterFS se fijarán después de inventariar las versiones reales del laboratorio.

## Decisiones pendientes antes de cerrar la versión 1.0

1. Confirmar si las 5 horas diarias incluyen las pausas.
2. Inventariar la instalación existente antes del día 2: versión de OLVM, sistema del Engine, número y versión de hosts, almacenamiento, redes y VMs disponibles.
3. Decidir qué prácticas pueden modificar el entorno compartido y cuáles necesitan nodos o máquinas virtuales independientes.
4. Determinar si el objetivo principal es operación real, examen 1Z0-1170 o un equilibrio explícito entre ambos.
5. Confirmar si el día 4 incluirá una instalación real completa o una demostración guiada apoyada por ejercicios parciales.

## Fuentes oficiales de referencia

- [Oracle Linux Virtualization Manager Documentation](https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/)
- [Architecture and Planning Guide](https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/)
- [Getting Started Guide](https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/)
- [Administrator's Guide](https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/)
- [Self-Hosted Engine Deployment](https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-hosted-engine-deploy.html)
- [Backup and Restore](https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/getstarted-backup-restore.html)
