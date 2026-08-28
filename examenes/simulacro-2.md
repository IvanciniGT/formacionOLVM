# Simulacro 2 · Oracle Linux Virtualization Manager Associate 1Z0-1170

**50 preguntas · 90 minutos · aprobado: 34 respuestas correctas (68 %)**

Este simulacro utiliza escenarios distintos del primero. Realícelo sin consultar
la documentación y corrija únicamente al terminar.

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

### 1. En un host se observa un proceso QEMU asociado a `vm-app01`. ¿Qué representa?

A. La ejecución activa de esa VM en el host.  
B. La base de datos histórica de la VM.  
C. El rol SPM del Data Center.  
D. El permiso del usuario sobre la VM.

### 2. Un Storage Domain thin tiene 2 TiB físicos y se han creado discos con 5 TiB virtuales. ¿Cuál es el riesgo principal?

A. Engine convertirá automáticamente todos los discos a ISO.  
B. Las VMs no podrán arrancar aunque no hayan escrito datos.  
C. El backend puede agotarse si el consumo real crece hasta superar la capacidad física.  
D. La sobreasignación elimina los snapshots existentes.

### 3. Durante el alta de un host, Engine instala VDSM y distribuye certificados. ¿Qué fase se está realizando?

A. Creación de una cuota.  
B. Bootstrap y enrolado del host.  
C. Consolidación de snapshots.  
D. Restauración de Data Warehouse.

### 4. Una VM tiene snapshots S1, S2 y S3. Se elimina S1. ¿Qué debe preservar OLVM?

A. Únicamente la capa física original, aunque S2 deje de funcionar.  
B. Nada; borrar un snapshot elimina obligatoriamente todos los posteriores.  
C. La memoria RAM actual de cualquier VM del clúster.  
D. El estado lógico de S2, S3 y la capa activa mediante la consolidación necesaria.

### 5. ¿Cuáles son dos responsabilidades de Engine y no de VDSM? Seleccione dos.

A. Crear el proceso QEMU local después de recibir una orden.  
B. Autorizar la solicitud del usuario.  
C. Conectar localmente una TAP a un bridge.  
D. Ejecutar el scheduler y elegir un host.

### 6. Un clúster permite contar threads SMT como cores. ¿Qué afirmación es correcta?

A. Aumenta la capacidad que considera el scheduler, pero no convierte dos threads en dos cores físicos completos.  
B. Duplica físicamente la memoria del servidor.  
C. Desactiva automáticamente NUMA.  
D. Obliga a utilizar discos RAW.

### 7. Se cambia el vNIC Profile de una VM, pero la IP dentro del invitado no cambia. ¿Cuál es la explicación más probable?

A. El perfil vNIC solo afecta a Storage QoS.  
B. SPM debe reiniciar PostgreSQL para asignar la IP.  
C. QEMU Guest Agent siempre conserva la IP anterior.  
D. La dirección IP se configura mediante DHCP/cloud-init/sistema invitado, no por el perfil por sí solo.

### 8. ¿Qué diferencia básica existe entre un rol administrativo y un rol de usuario?

A. El rol de usuario siempre permite modificar hosts.  
B. El administrativo concede privilegios de gestión; el de usuario permite consumir recursos dentro del alcance.  
C. Los roles administrativos solo se aplican a discos RAW.  
D. No existe diferencia en el portal disponible.

### 9. Se añade un segundo host a un entorno Self-Hosted Engine. ¿Qué opción debe seleccionarse en la pestaña Hosted Engine para que pueda ejecutar la VM Engine?

A. None.  
B. Undeploy.  
C. Deploy.  
D. Import ISO.

### 10. En compatibilidad moderna, una affinity label contiene VMs y hosts. ¿Qué define la relación positiva/negativa y hard/soft?

A. El grupo de afinidad que utiliza la etiqueta.  
B. El nombre de la etiqueta.  
C. El formato del Storage Domain.  
D. El Guest Agent de cada VM.

### 11. Todos los hosts resuelven el servidor NFS, pero uno recibe `permission denied` al montar el export. ¿Qué se revisa primero?

A. La topología de vCPU de las VMs.  
B. Exportación, permisos y acceso NFS de la IP de ese host.  
C. La contraseña de `admin@ovirt`.  
D. El icono del Data Center.

### 12. ¿Qué afirmación sobre SPM es falsa?

A. Engine puede elegir otro host si el SPM actual no está disponible.  
B. Coordina operaciones de metadatos del almacenamiento.  
C. Un host SPM también puede ejecutar VMs.  
D. Toda lectura y escritura de todas las VMs atraviesa obligatoriamente el SPM.

### 13. Una plantilla permanece `Image Locked` mientras se crea. ¿Cuál es la respuesta operativa correcta?

A. Cambiar manualmente el estado en PostgreSQL.  
B. Borrar el volumen NFS para liberar el lock.  
C. Revisar la tarea y los eventos; esperar si la operación progresa.  
D. Reiniciar simultáneamente todos los hosts.

### 14. ¿Qué ventaja ofrece crear una VM como clone independiente de una plantilla?

A. Su disco deja de depender de la imagen base de la plantilla.  
B. No consume espacio de almacenamiento.  
C. Conserva siempre el mismo `machine-id`.  
D. Elimina la necesidad de Guest Agent.

### 15. ¿Qué debe hacerse antes de elegir una versión de Oracle Linux para Engine o los hosts?

A. Utilizar siempre la versión numéricamente más alta.  
B. Deshabilitar todos los repositorios de Oracle.  
C. Instalar primero QEMU desde una fuente externa.  
D. Consultar la matriz y los requisitos de la release OLVM que se desplegará.

### 16. El portal indica presión de CPU, pero se necesita histórico detallado del sistema operativo del host. ¿Qué herramienta puede aportar métricas y archivos de rendimiento?

A. `ovirt-engine-rename`.  
B. Performance Co-Pilot y `pmlogger`.  
C. `cloud-init clean`.  
D. Sysprep.

### 17. ¿Cuáles son dos configuraciones que deben ser coherentes al transportar una VLAN sobre un bond? Seleccione dos.

A. La VLAN/LAG permitidos en los puertos del switch.  
B. El fondo de pantalla de la VM.  
C. El bond y la subinterfaz/configuración de red del host.  
D. La escala de Data Warehouse.

### 18. ¿Dónde se conserva principalmente el inventario y estado actual administrado por Engine?

A. En la base de datos `engine`.  
B. En `ovirt_engine_history` exclusivamente.  
C. Dentro del firmware UEFI de las VMs.  
D. En la BMC del host SPM.

### 19. ¿Dónde debe guardarse una copia de Engine para proteger frente a la pérdida completa de su servidor?

A. Únicamente en `/tmp` del propio Engine.  
B. En el mismo filesystem local y sin otra copia.  
C. Dentro de la memoria de una VM.  
D. En una ubicación externa e independiente del fallo protegido.

### 20. Un usuario autentica correctamente en VM Portal, pero no ve ninguna VM. ¿Cuál es la causa más probable?

A. VM Portal exige que el usuario sea SPM.  
B. No tiene un permiso de usuario sobre una VM, pool u objeto con alcance adecuado.  
C. Le falta un perfil de disco administrativo.  
D. Debe instalar VDSM en su equipo.

### 21. Se ha cambiado manualmente el hostname de Engine y ahora fallan certificados y URLs. ¿Cuál era el procedimiento correcto?

A. Modificar solo `/etc/hosts` en un hipervisor.  
B. Cambiar la MAC de todas las VMs.  
C. Preparar DNS y backup y utilizar el procedimiento `ovirt-engine-rename`.  
D. Crear un VM lease.

### 22. ¿Cuáles son dos ventajas de `ovirt-log-collector` durante una incidencia? Seleccione dos.

A. Sustituye automáticamente componentes defectuosos.  
B. Reúne evidencias de Engine y hosts en una recopilación coherente.  
C. Elimina todos los datos sensibles antes de compartir sin revisión.  
D. Facilita entregar a soporte logs correlacionados del mismo entorno.

### 23. ¿Qué describe mejor un Data Center en OLVM?

A. Una habitación física que debe corresponder a un edificio.  
B. Una VM especial que ejecuta PostgreSQL.  
C. Un perfil de vNIC compartido por usuarios.  
D. Un objeto lógico que agrupa storage, redes y clústeres relacionados.

### 24. Una plantilla AlmaLinux debe utilizar cloud-init. ¿Qué requisito existe dentro de la imagen?

A. Que SPM se ejecute como servicio invitado.  
B. Que el paquete/servicio cloud-init esté instalado y preparado para consumir los metadatos.  
C. Que el disco sea obligatoriamente RAW preasignado.  
D. Que la VM tenga una VF SR-IOV.

### 25. ¿Qué evita normalmente el filtro `vdsm-no-mac-spoofing` de un vNIC Profile?

A. Que la VM emita tráfico con direcciones MAC no autorizadas.  
B. Que el host utilice NFS.  
C. Que Engine conserve eventos.  
D. Que se creen snapshots.

### 26. Una afinidad VM–host positiva es soft y los hosts preferidos están llenos. ¿Qué puede hacer el scheduler?

A. Destruir una VM para liberar espacio.  
B. Ignorar todos los filtros de seguridad.  
C. Colocar la VM en otro host válido aunque no sea el preferido.  
D. Convertir la preferencia en fencing.

### 27. Se desea repetir `engine-setup` sin responder interactivamente a las mismas preguntas. ¿Qué mecanismo es apropiado?

A. Un snapshot con memoria de todas las VMs.  
B. Una affinity label.  
C. Un perfil vNIC.  
D. Un fichero de respuestas generado y protegido.

### 28. Se configura un watchdog virtual en una VM AlmaLinux. ¿Qué paquete proporciona el daemon que alimenta `/dev/watchdog`?

A. `ovirt-engine`.  
B. `watchdog`.  
C. `vdsm`.  
D. `sanlock-client` exclusivamente.

### 29. Engine deja de estar disponible, pero las VMs continúan funcionando. ¿Qué operación queda especialmente afectada?

A. Nuevas decisiones centralizadas de scheduling y administración.  
B. La ejecución de cada instrucción ya iniciada en KVM.  
C. Toda conmutación local del bridge Linux.  
D. Toda lectura NFS realizada directamente por los hosts.

### 30. Se crea una política Storage QoS. ¿Cómo se aplica normalmente a un disco?

A. Mediante una etiqueta de red física.  
B. Con una regla de fencing.  
C. Asociándola a un Disk Profile que selecciona el disco/VM.  
D. Instalando Guest Agent.

### 31. ¿Cuáles son dos afirmaciones correctas sobre VM lease y fencing? Seleccione dos.

A. El lease coordina la propiedad de una VM mediante almacenamiento compartido.  
B. El lease reemplaza siempre cualquier necesidad de fencing.  
C. Fencing aumenta el tamaño del disco virtual.  
D. Fencing intenta excluir o reiniciar el host físico dudoso.

### 32. Una cuota está en modo auditoría y un usuario supera el límite definido. ¿Qué comportamiento se espera?

A. El host se apaga mediante BMC.  
B. La VM se convierte en plantilla.  
C. Se cambia automáticamente a un rol administrativo.  
D. Se informa de la superación sin bloquear como lo haría enforcement.

### 33. ¿Cuál es un coste posible de habilitar KSM agresivamente?

A. Consumo de CPU al buscar y comparar páginas, además de consideraciones de seguridad.  
B. Pérdida obligatoria de todas las redes VLAN.  
C. Conversión de NFS en iSCSI.  
D. Eliminación de la base `engine`.

### 34. ¿Por qué se recomienda un Storage Domain dedicado para la VM Self-Hosted Engine?

A. Porque la VM Engine no utiliza discos.  
B. Porque todos los usuarios deben guardar allí sus ISOs.  
C. Para aislar su almacenamiento y metadatos de las cargas generales y del ciclo de vida ordinario.  
D. Para sustituir los hosts adicionales.

### 35. ¿Qué debe existir antes de asociar una Logical Network a Virtual Functions SR-IOV?

A. Un snapshot de memoria por VF.  
B. Una NIC compatible, VFs habilitadas y configuración física/red coherente.  
C. El rol SPM asignado permanentemente al host.  
D. Una base DWH remota.

### 36. Data Warehouse deja de funcionar, pero Engine y los hosts siguen activos. ¿Qué función se degrada principalmente?

A. La ejecución KVM de todas las VMs.  
B. El bridge Linux de cada red.  
C. El acceso de QEMU a los discos actuales.  
D. Las tendencias, informes y monitorización histórica.

### 37. Una plantilla está bloqueada mientras se crea desde una VM de 500 GiB. ¿Qué explica mejor el estado?

A. Una operación de imagen aún está copiando o preparando el disco; se revisa progreso y eventos.  
B. La plantilla ha sido eliminada de forma definitiva.  
C. El usuario debe editar sanlock manualmente.  
D. La red `ovirtmgmt` se ha convertido en ISO.

### 38. Antes de activar un sitio pasivo en un procedimiento DR, ¿qué debe garantizarse respecto al sitio anterior?

A. Que todas sus VMs conservan el mismo icono.  
B. Que KSM está desactivado.  
C. Que está aislado y no puede seguir escribiendo sobre los datos replicados.  
D. Que su Data Warehouse ha aumentado de tamaño.

### 39. Se crean dos Storage Domains NFS en dos exports del mismo servidor y sobre los mismos discos. ¿Qué afirmación es correcta?

A. Proporcionan automáticamente independencia total ante el fallo del servidor.  
B. Son dominios distintos para OLVM, pero comparten el mismo fallo físico.  
C. Uno se convierte en backup del otro sin copiar datos.  
D. Engine los transforma en dos cabinas independientes.

### 40. Un host candidato tiene instalados paquetes QEMU de un repositorio no soportado. ¿Qué acción es adecuada antes de incorporarlo?

A. Añadirlo y corregirlo después de ejecutar VMs.  
B. Ocultar los paquetes cambiando el hostname.  
C. Editar la base Engine para marcarlo compatible.  
D. Limpiar/preparar el host con Minimal Install y repositorios/versiones soportados.

### 41. ¿Qué compromiso introducen las huge pages?

A. Eliminan la necesidad de RAM física.  
B. Asignan direcciones MAC estáticas.  
C. Pueden mejorar traducción de memoria, pero reducen flexibilidad y requieren reserva/alineación.  
D. Sustituyen el almacenamiento compartido.

### 42. ¿Qué debe comprobarse antes de restaurar un backup de Engine sobre un sistema nuevo?

A. Compatibilidad de versión, alcance del backup, DNS y dependencias de base/storage.  
B. Que todos los discos de las VMs sean ISO.  
C. Que el nuevo host tenga el mismo icono.  
D. Que no exista ninguna red física.

### 43. ¿Qué diferencia existe entre drivers VirtIO y QEMU Guest Agent?

A. Guest Agent ejecuta la CPU; VirtIO almacena eventos.  
B. VirtIO optimiza dispositivos del camino de datos; Guest Agent aporta comunicación y acciones dentro del invitado.  
C. Ambos son nombres de un Storage Domain.  
D. VirtIO solo existe en Engine y nunca dentro del guest.

### 44. Una Logical Network utiliza MTU 9000. ¿Qué se necesita para evitar fragmentación o pérdida por MTU incoherente?

A. Cambiar el disco a QCOW2.  
B. Instalar un watchdog.  
C. Convertir el Engine en SPM.  
D. Soporte de la misma MTU en todo el camino relevante, incluidos host y switches.

### 45. Un grupo recibe un permiso sobre el Cluster. ¿Qué debe revisarse antes de guardarlo?

A. Solo la contraseña de un miembro.  
B. El formato de todos los discos.  
C. La herencia hacia objetos inferiores y el mínimo privilegio necesario.  
D. La prioridad SPM.

### 46. Una guía indica que IPv6 debe permanecer habilitado aunque se use IPv4. ¿Qué acción es correcta?

A. Mantenerlo según el requisito soportado y utilizar IPv4 en los flujos configurados.  
B. Eliminar IPv6 del kernel sin comprobar impacto.  
C. Convertir todas las VMs a IPv6.  
D. Deshabilitar DNS.

### 47. Una VM tiene 4 GiB definidos y 4 GiB máximos. Se intenta hot-plug hasta 8 GiB. ¿Qué ocurre?

A. El Storage Domain aporta automáticamente los 4 GiB que faltan.  
B. KSM cambia el máximo sin reinicio.  
C. El número de vCPU se duplica.  
D. El máximo configurado impide alcanzar 8 GiB sin modificar la configuración permitida.

### 48. Dos VMs de la misma red lógica se ejecutan en el mismo host y bridge. ¿Cómo pueden intercambiar tramas?

A. Obligatoriamente mediante Engine y PostgreSQL.  
B. El bridge puede conmutarlas localmente entre sus puertos TAP.  
C. A través del SPM.  
D. Copiando los discos al mismo directorio.

### 49. ¿Qué permite recuperar la VM Engine cuando el servicio Engine está caído en un despliegue Self-Hosted?

A. El QEMU Guest Agent de una VM cualquiera.  
B. Data Warehouse como hipervisor alternativo.  
C. Los agentes `ovirt-ha-agent`/`ovirt-ha-broker`, metadatos y storage compartido.  
D. Un usuario de VM Portal.

### 50. Una VM pasó de `Up` a `Down` a las 03:14. ¿Cuál es el primer paso de diagnóstico más útil?

A. Correlacionar evento, tarea, host y logs alrededor de las 03:14.  
B. Borrar inmediatamente todos sus snapshots.  
C. Reiniciar sin registrar la hora.  
D. Aumentar la compatibilidad del Cluster.

---

**Fin del simulacro.** Continúe con
[`simulacro-2-respuestas.md`](simulacro-2-respuestas.md) únicamente después de
terminar.
