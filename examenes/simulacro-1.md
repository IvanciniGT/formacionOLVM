# Simulacro 1 · Oracle Linux Virtualization Manager Associate 1Z0-1170

**50 preguntas · 90 minutos · aprobado: 34 respuestas correctas (68 %)**

No consulte las respuestas hasta terminar. Si una pregunta solicita dos opciones,
deben marcarse exactamente las dos.

---

## Hoja de respuestas

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|
|   |   |   |   |   |   |   |   |   |   |

| 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 |
|---|---|---|---|---|---|---|---|---|---|
|   |   |   |   |   |   |   |   |   |   |

| 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 |
|---|---|---|---|---|---|---|---|---|---|
|   |   |   |   |   |   |   |   |   |   |

| 31 | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 |
|---|---|---|---|---|---|---|---|---|---|
|   |   |   |   |   |   |   |   |   |   |

| 41 | 42 | 43 | 44 | 45 | 46 | 47 | 48 | 49 | 50 |
|---|---|---|---|---|---|---|---|---|---|
|   |   |   |   |   |   |   |   |   |   |

---

## Preguntas

### 1. ¿Qué componente recibe órdenes de Engine en cada host y coordina las operaciones con libvirt, red y almacenamiento?

A. QEMU Guest Agent  
B. Data Warehouse  
C. VDSM  
D. SPM

### 2. Antes de instalar Engine se comprueba que su FQDN y el de los hosts resuelven de manera coherente. ¿Cuál es la razón principal?

A. KVM solo admite nombres y no direcciones IP.  
B. DNS participa en certificados, acceso a servicios y comunicación entre componentes.  
C. PostgreSQL exige que todos los hosts pertenezcan al mismo dominio DNS público.  
D. NFS no funciona si se utiliza una dirección IP.

### 3. ¿Cuál es la función principal del Storage Pool Manager?

A. Transportar toda la E/S de las VMs hacia el almacenamiento.  
B. Asignar direcciones IP a las vNIC.  
C. Mantener las métricas históricas de todos los Storage Domains.  
D. Coordinar operaciones y metadatos del almacenamiento del Data Center.

### 4. ¿Cuáles son dos ejemplos de tráfico perteneciente al plano de datos? Seleccione dos.

A. Las escrituras de QEMU sobre el disco NFS.  
B. Una orden de Engine para arrancar una VM.  
C. Las tramas enviadas por la vNIC de una VM.  
D. Una consulta de permisos realizada por Engine.

### 5. ¿Por qué un snapshot almacenado en el mismo Storage Domain no debe considerarse un backup independiente?

A. Porque los snapshots solo pueden contener la memoria de la VM.  
B. Porque depende de la misma cadena de imágenes y del mismo dominio de almacenamiento.  
C. Porque OLVM elimina automáticamente todos los snapshots al apagar la VM.  
D. Porque los snapshots únicamente funcionan con discos RAW.

### 6. En una red OLVM basada en bridge Linux, ¿qué elemento conecta normalmente el proceso QEMU con el bridge del host?

A. El SPM.  
B. El Image I/O Proxy.  
C. El QEMU Guest Agent.  
D. Una interfaz TAP/vnet.

### 7. ¿Qué describe correctamente KSM?

A. Solicita al invitado que libere páginas mediante un dispositivo VirtIO.  
B. Fija cada vCPU a un core físico.  
C. Fusiona páginas de memoria idénticas y utiliza copy-on-write cuando cambian.  
D. Reserva huge pages exclusivamente para la VM Engine.

### 8. ¿Qué propiedad se define principalmente en el ámbito del Cluster?

A. La versión de compatibilidad y el tipo mínimo de CPU compartido por los hosts.  
B. La contraseña root de cada VM.  
C. El contenido del export NFS.  
D. La dirección IP de cada vNIC invitada.

### 9. Engine intenta incorporar un host y termina en `Install Failed`. ¿Qué debe hacerse primero?

A. Borrar manualmente las tablas que contienen el host.  
B. Eliminar todos los Storage Domains del Data Center.  
C. Cambiar la versión del Cluster al valor más alto disponible.  
D. Consultar eventos y los logs de host-deploy para identificar la fase fallida.

### 10. ¿Qué tres elementos forman un permiso en OLVM?

A. Usuario, contraseña y cuota.  
B. Usuario o grupo, rol y objeto de alcance.  
C. Rol, VLAN y Storage Domain.  
D. Grupo, política de scheduling y perfil vNIC.

### 11. Una VM utiliza un disco en NFS compartido y se migra en vivo a otro host. ¿Qué sucede normalmente con el disco?

A. Se copia completo al disco local del destino antes de mover la memoria.  
B. Se convierte siempre de QCOW2 a RAW.  
C. Permanece en NFS y el host destino abre la misma imagen compartida.  
D. Se separa del Storage Domain hasta terminar la migración.

### 12. Una operación falla desde el portal. ¿Cuál es el primer conjunto de evidencias que conviene fijar?

A. Estado del objeto, evento, tarea y hora exacta.  
B. Todas las tablas de PostgreSQL exportadas a CSV.  
C. Una captura completa de tráfico de todos los hosts.  
D. El historial de comandos de todos los usuarios.

### 13. ¿Cuáles son dos funciones propias de QEMU Guest Agent? Seleccione dos.

A. Configurar bonds físicos en el host.  
B. Informar a Engine de datos internos como hostname o IP.  
C. Elegir qué host será SPM.  
D. Coordinar un apagado ordenado del invitado.

### 14. ¿Qué problema evita el sellado correcto de una VM antes de crear una plantilla?

A. Que el Storage Domain utilice thin provisioning.  
B. Que Engine pueda migrar las nuevas VMs.  
C. Que las VMs derivadas repitan claves SSH, `machine-id` o identidad de red.  
D. Que el clúster use más de una NIC.

### 15. ¿Qué afirmación describe mejor `engine-setup`?

A. Configura o reconfigura de forma soportada los componentes de Engine seleccionados.  
B. Es el hipervisor que ejecuta las VMs.  
C. Sustituye a VDSM en todos los hosts.  
D. Crea automáticamente todas las redes y VMs de producción.

### 16. ¿Qué objeto selecciona normalmente una vNIC de una VM para obtener políticas de conexión a una red lógica?

A. Disk Profile.  
B. Storage QoS.  
C. CPU Profile.  
D. vNIC Profile.

### 17. Una anti-afinidad VM–VM está marcada como `Enforcing` para tres VMs en un clúster de dos hosts. ¿Cuál es el resultado más probable si las tres deben ejecutarse?

A. OLVM crea automáticamente un tercer host virtual.  
B. Al menos una VM no podrá colocarse cumpliendo la regla.  
C. Engine ignora siempre las reglas `Enforcing` durante el arranque.  
D. Las tres VMs se ejecutarán en el mismo host.

### 18. ¿Para qué se utiliza principalmente Data Warehouse?

A. Para ejecutar instrucciones de CPU de las VMs.  
B. Para proporcionar fencing a los hosts.  
C. Para conservar y preparar información histórica de uso y rendimiento.  
D. Para conmutar tramas entre vNIC.

### 19. ¿Qué protege principalmente una copia creada con `engine-backup --scope=all`?

A. La configuración y las bases del plano de control incluidas en el alcance.  
B. Todos los bloques de todos los discos de las VMs.  
C. El firmware de las BMC.  
D. Las configuraciones internas de cualquier aplicación invitada.

### 20. ¿Cuál es la diferencia correcta entre cuota y QoS?

A. QoS autentica usuarios; la cuota crea certificados.  
B. La cuota solo se usa en networking; QoS solo en CPU.  
C. Son dos nombres para el mismo objeto.  
D. La cuota limita recursos asignables; QoS regula o prioriza rendimiento.

### 21. Engine está instalado dentro de una VM creada manualmente en otra plataforma y administra hosts OLVM. ¿Qué topología es?

A. Self-Hosted Engine por el hecho de ser una VM.  
B. Standalone Engine, porque su ciclo de vida es externo al OLVM administrado.  
C. Hosted Engine únicamente mientras la VM está encendida.  
D. Un host SPM.

### 22. ¿Cuáles son dos características habituales de QCOW2? Seleccione dos.

A. Copy-on-write y soporte de backing files.  
B. Reserva física obligatoria de todo el tamaño virtual.  
C. Crecimiento fino según los bloques escritos.  
D. Ausencia completa de metadatos.

### 23. ¿Qué diferencia existe entre `Non Operational` y `Non Responsive`?

A. `Non Operational` significa apagado físico; `Non Responsive` significa mantenimiento.  
B. Ambos estados significan exactamente lo mismo.  
C. `Non Responsive` solo se aplica a Storage Domains.  
D. `Non Operational` puede comunicar pero incumple un requisito; `Non Responsive` no responde a Engine.

### 24. ¿Qué ocurre normalmente con las VMs que ya se ejecutan si se detiene Engine pero los hosts y el almacenamiento siguen disponibles?

A. Se destruyen inmediatamente.  
B. Pueden seguir ejecutándose, aunque se pierde o degrada la gestión central.  
C. Se convierten automáticamente en plantillas.  
D. Pierden sus discos porque Engine transportaba cada bloque.

### 25. Se restaurará Engine en un servidor nuevo. ¿Qué acción debe preceder al proceso?

A. Crear manualmente las mismas filas en PostgreSQL.  
B. Formatear todos los Storage Domains.  
C. Preparar una versión compatible y disponer de una copia válida fuera del Engine perdido.  
D. Elevar la compatibilidad del Cluster sin comprobar los hosts.

### 26. ¿Cuál es la diferencia principal entre importar un OVA y adjuntar un disco virtual?

A. Un disco siempre contiene la configuración completa de Engine.  
B. Un OVA solo puede contener una ISO.  
C. No existe diferencia funcional ni de contenido.  
D. Un OVA puede contener definición y varios discos; un disco contiene principalmente bloques de una unidad.

### 27. ¿Por qué debe documentarse un puerto junto con su origen, destino y función?

A. Porque OLVM no utiliza protocolos TCP.  
B. Porque el mismo número puede aparecer en flujos distintos y las reglas necesitan dirección y propósito.  
C. Porque todos los puertos deben abrirse desde cualquier red.  
D. Porque el firewall solo se configura dentro de las VMs.

### 28. ¿Qué problema intenta reducir NUMA pinning en una VM grande?

A. Direcciones MAC duplicadas.  
B. La ausencia de un Storage Domain maestro.  
C. Accesos de una vCPU a memoria remota respecto del nodo físico donde ejecuta.  
D. Certificados de Engine caducados.

### 29. ¿Qué comando muestra explícitamente que `vnet0` pertenece a `br-olvm` mediante la indicación `master br-olvm`?

A. `df -h`  
B. `hostname -f`  
C. `engine-backup`  
D. `bridge link`

### 30. Una VM tiene memoria definida de 4 GiB y memoria garantizada de 4 GiB. ¿Qué margen práctico ofrece ballooning para reducirla por debajo de la memoria garantizada?

A. 4 GiB adicionales.  
B. Ninguno.  
C. Toda la memoria máxima.  
D. Depende únicamente del formato del disco.

### 31. El host que ejerce como SPM entra en mantenimiento. ¿Qué comportamiento se espera?

A. Todas las VMs deben perder sus discos permanentemente.  
B. La red de gestión se convierte en Storage Domain.  
C. Engine puede asignar el rol SPM a otro host válido del Data Center.  
D. El rol queda siempre unido al mismo servidor físico.

### 32. Se concede a un grupo un rol de usuario sobre una única VM. ¿Qué determina principalmente que el permiso no se extienda a todas las VMs?

A. La contraseña de los usuarios.  
B. El formato QCOW2.  
C. La prioridad SPM del host.  
D. El objeto de alcance sobre el que se asignó el rol.

### 33. Engine registra que QEMU no pudo crear un dispositivo al arrancar una VM concreta. ¿Qué log es especialmente relevante después de revisar el evento?

A. Únicamente el log del navegador.  
B. El log QEMU/libvirt de esa VM en el host elegido.  
C. El log de otra VM que arrancó correctamente hace un mes.  
D. El historial de DHCP del servidor NFS.

### 34. Durante `hosted-engine --deploy`, ¿cómo se resuelve inicialmente la ausencia de un Engine que cree su propia VM?

A. El SPM descarga una VM desde cada host de producción.  
B. El switch físico ejecuta temporalmente Engine.  
C. El primer host crea una VM provisional mediante libvirt y configura la appliance.  
D. La base DWH actúa como hipervisor.

### 35. Se configura un bond 802.3ad en un host físico. ¿Qué elemento externo debe configurarse de forma coherente?

A. El filesystem de todas las VMs.  
B. El número de serie virtual de cada invitado.  
C. La base histórica de Engine.  
D. El LAG/LACP de los puertos correspondientes del switch físico.

### 36. ¿Qué dos capacidades pertenecen principalmente a una estrategia de Disaster Recovery y no solo a HA local? Seleccione dos.

A. Hot plug de una vNIC.  
B. Replicación de datos hacia otro sitio.  
C. Ballooning dentro de una VM.  
D. Procedimiento probado de failover/failback del sitio.

### 37. Cuando un usuario solicita una VM de un pool, ¿qué sucede normalmente?

A. El usuario instala automáticamente un host KVM nuevo.  
B. Se crea un Data Center por usuario.  
C. Engine le asigna una de las VMs ya creadas y disponibles del pool.  
D. Se convierte el template en Storage Domain.

### 38. ¿Por qué es necesario fencing antes de recuperar VMs de un host que ha dejado de responder?

A. Para aumentar el tamaño virtual de sus discos.  
B. Para confirmar que el host anterior no continúa ejecutándolas y escribiendo en storage.  
C. Para instalar QEMU Guest Agent.  
D. Para crear métricas históricas.

### 39. Una VM aparece `Down`. ¿Qué se puede afirmar?

A. Todos sus discos han sido eliminados.  
B. Nunca podrá volver a arrancar.  
C. El host que la ejecutaba es necesariamente SPM.  
D. Su definición puede existir en Engine, pero no hay una ejecución QEMU activa.

### 40. ¿En qué ámbito se adjuntan y coordinan los Storage Domains compartidos?

A. vNIC Profile.  
B. Usuario.  
C. Data Center.  
D. QEMU Guest Agent.

### 41. ¿Cuándo puede ser útil VirtIO-SCSI multiqueue?

A. Cuando se necesita resolver DNS inverso.  
B. Cuando una VM con varias vCPUs genera suficiente E/S para aprovechar varias colas.  
C. Cuando Engine ha perdido su certificado.  
D. Para sustituir el servidor NFS.

### 42. ¿Cuál es la secuencia correcta para mantener planificadamente un host que ejecuta VMs migrables?

A. Apagar el switch sin avisar y después crear un snapshot.  
B. Borrar el host de la base antes de migrar.  
C. Desactivar todos los Storage Domains.  
D. Colocarlo en mantenimiento, confirmar evacuación, intervenir, activar y validar.

### 43. ¿Qué herramienta/procedimiento debe utilizarse para cambiar de forma soportada el FQDN de Engine?

A. Editar únicamente `/etc/hostname`.  
B. Renombrar el bridge de una VM.  
C. `ovirt-engine-rename` junto con la preparación y validación requeridas.  
D. Cambiar el alias del disco de Engine.

### 44. ¿Qué consecuencia puede tener conectar una vNIC mediante una Virtual Function SR-IOV no migrable?

A. Convierte automáticamente el disco en RAW.  
B. Puede impedir la live migration de la VM.  
C. Elimina la necesidad de una red física.  
D. Instala el Guest Agent.

### 45. ¿Qué ventaja aporta conceder permisos a grupos procedentes de un proveedor de identidad?

A. Hace innecesarios los roles.  
B. Convierte las cuotas en QoS.  
C. Permite que cualquier miembro sea SuperUser global.  
D. Facilita administrar el acceso de un equipo sin repetir permisos usuario por usuario.

### 46. ¿Qué afirmación distingue correctamente OLVM de Oracle VM clásico?

A. Ambos son exactamente el mismo producto con distinto tema visual.  
B. OLVM utiliza exclusivamente Solaris Zones.  
C. OLVM administra Oracle Linux KVM y procede de la arquitectura oVirt; Oracle VM clásico utilizaba Xen.  
D. Oracle VM clásico es el nombre de VDSM.

### 47. En la GUI se rellenan campos de cloud-init para una VM. ¿Qué hace Engine?

A. Sustituye el kernel del host por YAML.  
B. Genera los metadatos que la VM consumirá en el arranque inicial.  
C. Configura físicamente el switch de acceso.  
D. Cambia la IP de la BMC.

### 48. ¿Por qué se recomienda partir de una instalación mínima de Oracle Linux para Engine y hosts?

A. Porque una instalación mínima no necesita DNS.  
B. Porque elimina la necesidad de actualizar.  
C. Porque incluye automáticamente todas las VMs.  
D. Para evitar paquetes, repositorios y versiones de QEMU/libvirt incompatibles o innecesarios.

### 49. Una red lógica obligatoria no está desplegada en un host del Cluster. ¿Qué estado puede mostrar el host?

A. `Image Locked`.  
B. `Paused`.  
C. `Non Operational`.  
D. `Preview`.

### 50. Una VM no arranca y el evento indica que no existe un host válido. ¿Cuáles son dos comprobaciones prioritarias? Seleccione dos.

A. Color del icono personalizado de la VM.  
B. Capacidad/compatibilidad de los hosts y reglas de afinidad.  
C. Número de dashboards de Grafana.  
D. Disponibilidad de redes y Storage Domains requeridos.

---

**Fin del simulacro.** Anote la hora y pase a
[`simulacro-1-respuestas.md`](simulacro-1-respuestas.md) únicamente después de
terminar.
