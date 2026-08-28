# Bibliografía y fuentes de la formación OLVM

Fuentes de apoyo para preparar y ampliar el material. Se prioriza documentación oficial de Oracle; cuando Oracle remite a documentación upstream de oVirt, se indica expresamente.

Última revisión de enlaces: **25 de agosto de 2026**.

---

# Día 1 · Fundamentos, arquitectura y ecosistema

## Documentación principal

1. **Oracle Linux Virtualization Manager Documentation**  
   Portal general de documentación, versiones, notas de versión y guías.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/>

2. **Oracle Linux Virtualization Manager: Architecture and Planning Guide**  
   Arquitectura general, componentes y planificación.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/>

3. **OLVM Engine Architecture**  
   Función del Engine y relación con los hosts.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-arch-engine.html>

4. **Host Architecture**  
   Arquitectura del host KVM y componentes implicados.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-arch-host-arch.html>

5. **Data Warehouse and Database Architecture**  
   Bases de datos del Engine y del Data Warehouse.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-arch-dwh-db.html>

6. **Administration Interfaces and Portals**  
   Administration Portal, VM Portal y otras interfaces de acceso.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-arch-admin-interface.html>

7. **Self-Hosted Engine Deployment**  
   Despliegue y conceptos del Engine ejecutado como VM administrada.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-hosted-engine-deploy.html>

## Proyectos base

8. **KVM documentation**  
   Documentación del subsistema de virtualización del kernel Linux.  
   <https://www.kernel.org/doc/html/latest/virt/kvm/index.html>

9. **QEMU documentation**  
   Emulación, proceso de la VM y dispositivos virtuales.  
   <https://www.qemu.org/docs/master/>

10. **libvirt documentation**  
    API y herramientas de gestión de virtualización.  
    <https://libvirt.org/docs.html>

11. **oVirt documentation**  
    Documentación del proyecto upstream sobre el que se basa OLVM.  
    <https://www.ovirt.org/documentation/>

---

# Día 2 · Storage Domains, NFS y networking nativo

## Almacenamiento

1. **OLVM Storage Architecture**  
   Storage Domains, SPM, imágenes y conceptos de almacenamiento compartido.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-storage.html>

2. **oVirt Administration Guide**  
   Operaciones detalladas sobre Storage Domains, virtual disks, snapshots y tareas administrativas. Oracle remite a esta documentación upstream para procedimientos que no duplica en sus guías.  
   <https://www.ovirt.org/documentation/administration_guide/>

3. **Red Hat Enterprise Linux 9: Managing file systems — NFS**  
   Conceptos y administración del cliente/servidor NFS sobre la base Linux de los hosts.  
   <https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/mounting-nfs-shares_managing-file-systems>

## Networking

4. **OLVM Network Architecture**  
   Logical Networks, host networking y flujo conceptual de red.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-networks.html>

5. **OLVM Host Architecture**  
   Papel de VDSM y componentes del host; útil para relacionar Engine, host y plano de datos.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-arch-host-arch.html>

6. **Linux bridge documentation**  
   Bridge, FDB, aprendizaje de MAC y comportamiento del switch de capa 2 del kernel.  
   <https://www.kernel.org/doc/html/latest/networking/bridge.html>

7. **NetworkManager documentation**  
   Perfiles y estado de red en Linux. En las redes administradas por OLVM se utiliza para observar y comprender, no para sustituir al Engine como fuente de verdad.  
   <https://networkmanager.dev/docs/>

8. **nmstate documentation**  
   Modelo declarativo utilizado para describir y aplicar estado de red en los hosts.  
   <https://nmstate.io/>

## Fuentes de la instalación del curso

Las siguientes evidencias proceden del Administration Portal del laboratorio y no de ejemplos externos:

- Data Center `Curso`, compartido, funcionando, compatibilidad 4.7.
- Data Center `Default`, compartido, no inicializado.
- Data Domains NFS V5 `curso-nfs` y `curso-nfs-2`, ambos activos.
- `curso-nfs` con rol de Data Domain maestro.
- Logical Network `alumnos`, VM Network sin VLAN, MTU 1500.
- vNIC Profile `alumnos`, con filtro `vdsm-no-mac-spoofing` y 14 VMs asociadas.

Estas capturas reflejan el estado observado el **25 de agosto de 2026**. Si la instalación cambia, prevalece la observación actual del portal y de los hosts.

---

# Día 3 · Ciclo de vida de una VM, templates, pools y permisos

## Máquinas virtuales y operaciones

1. **OLVM Virtual Machine Architecture**
   Relación entre la definición de una VM, QEMU/KVM, dispositivos VirtIO, Guest Agent, templates, snapshots y VM Pools.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-virtual-machines.html>

2. **OLVM Administration: Virtual Machine Tasks**
   Creación y administración de VMs, instalación del Guest Agent, snapshots, templates, importación y live migration.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-vm-tasks.html>

3. **OLVM Administration: Hot Plugging Virtual Devices**
   Conexión y desconexión en caliente de discos, vNICs y otros dispositivos compatibles.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-hot-plug-devices.html>

4. **OLVM Host Architecture**
   Papel de VDSM, libvirt y QEMU en la ejecución y supervisión de las VMs.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-arch-host-arch.html>

5. **OLVM Administration: Host Tasks**
   Mantenimiento de hosts y evacuación o migración de sus VMs.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-host-tasks.html>

## Usuarios y delegación

6. **OLVM User Permissions Architecture**
   Modelo de usuarios, grupos, roles, permisos y herencia sobre objetos.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-user-permissions.html>

7. **OLVM Administration Portal User Management**
   Gestión administrativa de usuarios y permisos desde el portal.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-admin-portal-user-management.html>

8. **oVirt Virtual Machine Management Guide**
   Referencia upstream para roles de VM, creación desde templates, pools, sealing, cloud-init y operaciones del ciclo de vida.
   <https://www.ovirt.org/documentation/virtual_machine_management_guide/>

9. **Introduction to the VM Portal**
   Alcance del portal de autoservicio y operaciones disponibles para los usuarios finales.
   <https://www.ovirt.org/documentation/doc-Introduction_to_the_VM_Portal/>

10. **oVirt Administration Guide**
    Referencia upstream para cuotas, modos de aplicación, QoS y administración avanzada.
    <https://www.ovirt.org/documentation/administration_guide/>

## Alta disponibilidad, agentes y drivers del invitado

11. **OLVM Administration: Optimizing Clusters, Hosts and Virtual Machines**
    Parámetros de recursos, memoria, CPU, pinning y optimización de máquinas virtuales.
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-optimize-clus-host-vm.html>

12. **Oracle Linux KVM: Add Watchdog Device to KVM Instance**
    Funcionamiento del watchdog virtual, modelo `i6300esb` y acciones cuando expira el temporizador.
    <https://docs.oracle.com/en/operating-systems/oracle-linux/8/kvm-user/kvm-config_vm_watchdog.html>

13. **AlmaLinux 9 AppStream: paquete `watchdog`**
    Contenido del paquete que proporciona el daemon y la unidad `watchdog.service` dentro del invitado.
    <https://www.rpmfind.net/linux/RPM/almalinux/9.8/appstream/x86_64/watchdog-5.16-2.el9.x86_64.html>

14. **AlmaLinux 9 BaseOS: paquete `kernel-modules-core`**
    Relación de módulos del kernel, incluido `virtio_balloon`, utilizado para memory ballooning.
    <https://www.rpmfind.net/linux/RPM/almalinux/9.7/baseos/x86_64/kernel-modules-core-5.14.0-611.34.1.el9_7.x86_64.html>

## Fuentes de la instalación del curso

El material del día 3 también se ha ajustado a la configuración observada en el laboratorio y a los playbooks que lo construyen:

- dos hosts OLVM anidados, con capacidad limitada, por lo que las operaciones pesadas se hacen de forma escalonada;
- almacenamiento NFS compartido mediante `curso-nfs` y `curso-nfs-2`;
- red lógica y perfil vNIC `alumnos` para las VMs del curso;
- VMs asignadas individualmente a los alumnos y varias imágenes ligeras de referencia;
- permisos `UserVmManager` sobre la VM asignada y, en el estado actual del aula, privilegios administrativos globales adicionales.

La última circunstancia obliga a tratar el VM Portal y la delegación como una práctica controlada: la visibilidad real de un alumno puede ser mayor de la que tendría en un diseño de producción con mínimo privilegio.

---

# Día 4 · Afinidad, alta disponibilidad, optimización y diagnóstico

## Scheduler y afinidad

1. **OLVM Architecture: High Availability and Optimization**
   Políticas predeterminadas de scheduling, balanceo, migración y relación con la disponibilidad del Cluster.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-ha-optimize.html>

2. **OLVM Administration: Creating a Scheduling Policy**
   Composición de una política mediante filtros, pesos, balanceador y propiedades.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-admin-schedule-policy-create.html>

3. **oVirt Virtual Machine Management Guide: Affinity Groups and Affinity Labels**
   Referencia upstream detallada para reglas VM–VM y VM–Host, polaridad positiva/negativa, `Enforcing`, etiquetas, ejemplos y resolución de conflictos.
   <https://www.ovirt.org/documentation/virtual_machine_management_guide/>

## Alta disponibilidad y optimización

4. **OLVM Administration: Optimizing Clusters, Hosts and Virtual Machines**
   Referencia principal para fencing y hosts proxy, alta disponibilidad de VMs, VM leases, prioridad de migración/reinicio, overcommit, CPU shares, pinning, NUMA, ballooning, KSM, huge pages, I/O threads, multiqueue y perfiles de alto rendimiento.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-optimize-clus-host-vm.html>

5. **OLVM Administration: Host Tasks**
   Estados y operaciones de los hosts, mantenimiento, gestión de energía y tareas administrativas relacionadas con la disponibilidad.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-host-tasks.html>

6. **OLVM Architecture: Hosts**
   Arquitectura del host y relación entre Engine, VDSM, libvirt y QEMU/KVM.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-hosts.html>

7. **Oracle Linux KVM: Add Watchdog Device to KVM Instance**
   Dispositivo watchdog virtual, daemon dentro del invitado y acciones ejecutadas al vencer el temporizador.
   <https://docs.oracle.com/en/operating-systems/oracle-linux/8/kvm-user/kvm-config_vm_watchdog.html>

## Eventos, logs y troubleshooting

8. **OLVM Architecture: Event Logs**
   Ubicación y finalidad de los principales logs de Engine y VDSM.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-event-logs.html>

9. **OLVM Release Notes: Obtaining the Log Files**
   Instalación y uso de `ovirt-log-collector`, alcance de la recogida y ubicación del archivo resultante.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/relnotes/relnotes-obtainingTheLogFiles.html>

10. **oVirt Developer Guide: VDSM Log Files**
   Referencia upstream para `vdsm.log`, logs de libvirt y diagnóstico de las operaciones ejecutadas en los hosts.
   <https://www.ovirt.org/develop/developer-guide/vdsm/log-files.html>

11. **oVirt Developer Guide: libvirt**
   Relación de VDSM con libvirt y log de las llamadas realizadas por VDSM.
   <https://www.ovirt.org/develop/projects/libvirt.html>

12. **OLVM Administration: Storage Tasks**
   Estados y operaciones de Storage Domains utilizados para interpretar incidencias de NFS, discos e imágenes.
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-storage-tasks.html>

## Fuentes de la instalación del curso

El contenido del día 4 se ha contrastado con la automatización y el estado observado del laboratorio:

- Engine se ejecuta como VM en `worker2`; los hosts OLVM anidados se ejecutan en `worker3` y `worker4`;
- los datos de las VMs se encuentran en los Storage Domains NFS `curso-nfs` y `curso-nfs-2`;
- los hosts anidados no tienen configurado un BMC físico, por lo que la gestión de energía y el fencing real no están disponibles.

Estas características se utilizan para diferenciar lo que el aula permite observar de lo que exigiría una arquitectura física empresarial con gestión fuera de banda, redundancia y capacidad N+1 probadas.

# Día 5 · Monitorización e instalación

## Monitorización, histórico y diagnóstico

1. **OLVM Administration: Monitoring and Diagnostics**  
   Referencia principal para Monitoring Portal, Grafana, Event Notifier, correo, SNMP, Performance Co-Pilot y diagnóstico de la plataforma.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-monitoring.html>

2. **OLVM Architecture: System State and History**  
   Diferencia entre la base operativa `engine`, la base histórica `ovirt_engine_history` y el proceso ETL de Data Warehouse.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-sys-state-hist.html>

3. **OLVM Architecture: Data Warehouse and Databases**  
   Arquitectura de las bases PostgreSQL, cambios de entidades y estadísticas conservadas por DWH.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-arch-dwh-db.html>

4. **OLVM Architecture: Data Visualization with Grafana**  
   Origen de datos de la integración nativa, periodicidad de recogida, agregaciones y escalas de retención.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-grafana.html>

5. **OLVM Administration: Databases**  
   Identificación de las bases, consulta oficial de tamaños y procedimiento soportado de mantenimiento. En clase solo se utilizan consultas de lectura.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-data-bases-tasks.html>

6. **OLVM Architecture: Event Logging and Notifications**  
   Función de `engine.log`, `vdsm.log`, eventos del portal, correo, SNMP y relación con `ovirt-log-collector`.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-ap-event-logs.html>

7. **OLVM Release Notes: Obtaining the Log Files**  
   Instalación y ejecución de `ovirt-log-collector`, alcance por defecto, exclusión de PostgreSQL y ubicación del archivo resultante.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/relnotes/relnotes-obtainingTheLogFiles.html>

8. **PostgreSQL: The Statistics Collector**  
   Referencia primaria para `pg_stat_activity`, `pg_locks` y las vistas de estadísticas utilizadas en las consultas de diagnóstico.  
   <https://www.postgresql.org/docs/current/monitoring-stats.html>

9. **PostgreSQL: Error Reporting and Logging**  
   Referencia primaria para `log_statement`, `log_min_duration_statement`, conexiones, duración, destinos y rotación de logs.  
   <https://www.postgresql.org/docs/current/runtime-config-logging.html>

## Arquitectura y requisitos de instalación

10. **OLVM Architecture: Engine Host Requirements**  
   Requisitos de sistema, capacidad y red del host de Engine, además de la separación entre Standalone Engine y host KVM.  
   <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-manager-reqs.html>

11. **OLVM Architecture: KVM Host Requirements**  
    Requisitos de sistema operativo, kernel, CPU, virtualización por hardware, memoria, disco y red de los hosts.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-kvm-reqs.html>

12. **OLVM Architecture: Firewall Requirements**  
    Matrices oficiales para Engine, hosts y componentes remotos, con origen, destino, protocolo, puerto y finalidad.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-firewall-reqs.html>

13. **OLVM Architecture: Installation Scalability Limits**  
    Límites y categorías de tamaño que deben revisarse al dimensionar Engine, Clusters, hosts y VMs.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/arch/architecture-olv-install-scalability.html>

## Instalación y configuración de Engine

14. **OLVM Getting Started: Installation and Configuration**  
    Orden general de despliegue: Engine, hosts KVM, redes, almacenamiento y máquinas virtuales.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-manager-install.html>

15. **OLVM Getting Started: Install the Engine**  
    Preparación del sistema, repositorios, paquete `ovirt-engine`, ejecución de `engine-setup` y opciones de bases, DWH, Grafana, Keycloak, proxies, OVN, FQDN y firewall.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-manager-install-prepare.html>

16. **OLVM Administration: Keycloak Integration and Management**  
    SSO, federación de usuarios y grupos, Active Directory, MFA y acceso al Monitoring Portal.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/admin-keycloak.html>

## Self-Hosted Engine y hosts KVM

17. **OLVM Getting Started: Self-Hosted Engine Deployment**  
    Arquitectura, alta disponibilidad, requisitos generales y mínimo de dos hosts para la VM de Engine.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-hosted-engine-deploy.html>

18. **OLVM Getting Started: Self-Hosted Engine Prerequisites**  
    Requisitos de FQDN, release común, espacio, storage dedicado y rango de dos a siete hosts KVM para el despliegue.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-he-deploy-prereqs.html>

19. **OLVM Getting Started: Deploy the Self-Hosted Engine**  
    Preparación, prechecks, appliance de Engine, bootstrap, storage compartido y despliegue de los hosts Hosted Engine.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-he-deploy-tasks.html>

20. **OLVM Getting Started: Configure a KVM Host**  
    Preparación de Oracle Linux, repositorios, `olvm-pre-check.py`, alta desde el portal, autenticación SSH, instalación de VDSM y paso de `Installing` a `Up`.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/getstart/getstarted-manager-install-kvm.html>

21. **OLVM Administration: Backup and Restore**  
    Backup inicial y periódico de configuración, base `engine` y DWH mediante `engine-backup`. Se utiliza como comprobación de aceptación, no como desarrollo completo del bloque de recuperación.  
    <https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/admin/getstarted-backup-restore.html>

## Fuentes de la instalación del curso

El contenido de monitorización se ha ajustado a la instalación observada el **28 de agosto de 2026**:

- `olvm-engine` se ejecuta como VM externa al Cluster OLVM, alojada en `worker2`;
- DWH está instalado y mantiene la base histórica;
- la integración nativa de Grafana no está instalada;
- `collectd` expone métricas que consulta Prometheus y visualiza un Grafana externo;
- `ovirt-engine-notifier` entrega notificaciones de laboratorio mediante SMTP a Mailpit;
- los hosts OLVM son VMs anidadas y utilizan Storage Domains NFS.

Estas características sirven para comparar una instalación real del aula con una arquitectura empresarial. Los números de versión, nombres de repositorios, kernels soportados, puertos opcionales y funciones en Technology Preview deben verificarse siempre en la documentación de la versión que se vaya a desplegar.

---

# Examen 1Z0-1170

1. **Oracle Linux Virtualization Manager 4.5 Associate Certification**  
   Ficha oficial del examen y referencia que debe comprobarse antes de reservarlo.  
   <https://education.oracle.com/oracle-linux-virtualization-manager-associate/pexam_1Z0-1170>

2. **Oracle Virtualization Blog: New Training to Learn About Oracle Virtualization**  
   Anuncio oficial que publica 50 preguntas y un 68 % de aciertos como nota de corte.  
   <https://blogs.oracle.com/virtualization/new-training-to-learn-about-oracle-virtualization>

3. **Oracle Certification: Score Review FAQ**  
   Criterios generales de puntuación y ausencia de penalización por respuestas incorrectas.  
   <https://docs.oracle.com/en/education/customer-success/digital-learning-kit/ml-cert-faq/topics/ScoreReview.html>

---

# Criterio de uso de las fuentes

- Para comportamiento específico de OLVM, se prioriza Oracle.
- Para detalles internos heredados de oVirt que Oracle no repite, se consulta oVirt.
- Para KVM, bridge, NFS y NetworkManager, se utilizan las fuentes primarias del componente o de la distribución.
- Una analogía con vSphere sirve para trasladar el concepto, pero no sustituye al procedimiento oficial de OLVM.
- Antes de ejecutar una operación administrativa se debe verificar que la guía corresponde a la versión instalada.
