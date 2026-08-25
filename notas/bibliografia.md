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

# Criterio de uso de las fuentes

- Para comportamiento específico de OLVM, se prioriza Oracle.
- Para detalles internos heredados de oVirt que Oracle no repite, se consulta oVirt.
- Para KVM, bridge, NFS y NetworkManager, se utilizan las fuentes primarias del componente o de la distribución.
- Una analogía con vSphere sirve para trasladar el concepto, pero no sustituye al procedimiento oficial de OLVM.
- Antes de ejecutar una operación administrativa se debe verificar que la guía corresponde a la versión instalada.
