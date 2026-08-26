# Día 3 · Preguntas de repaso y tipo examen

Estas preguntas acompañan a `dia3_conceptos.md`. El objetivo es seguir el ciclo de vida completo de una VM y distinguir objetos que en el portal aparecen muy próximos.

Cada pregunta tiene una única respuesta correcta. Después de responder, hay que justificar por qué las demás opciones no encajan.

---

# Anatomía y ciclo de vida de una VM

## 1. Una VM aparece en estado `Down`. ¿Qué significa?

A. Se han eliminado su definición y sus discos.  
B. La VM está definida en OLVM, pero no tiene un proceso QEMU ejecutándose.  
C. VDSM ha desaparecido del host.  
D. Su Storage Domain está necesariamente inactivo.

## 2. ¿Dónde se ejecuta una VM encendida?

A. Dentro del proceso Java del Engine.  
B. En el host seleccionado, mediante QEMU/KVM.  
C. Dentro del servidor NFS.  
D. En el host que ejerce siempre de SPM.

## 3. Engine muestra una dirección IP en la ficha de una VM. ¿Qué conclusión es correcta?

A. Engine ha configurado necesariamente esa IP.  
B. La IP puede haber sido reportada por el Guest Agent; su configuración pertenece al invitado.  
C. La IP identifica el Storage Domain.  
D. La IP es siempre la del bridge del host.

## 4. ¿Qué expresa la memoria física garantizada de una VM?

A. El espacio de swap creado dentro del invitado.  
B. Un compromiso de recursos considerado por la plataforma para esa VM.  
C. La memoria máxima del host.  
D. La caché disponible en el servidor NFS.

## 5. Una VM tiene 2 sockets, 2 cores por socket y 1 thread por core. ¿Cuántas vCPU presenta?

A. 2.  
B. 3.  
C. 4.  
D. 8.

## 6. ¿Qué diferencia principal existe entre VDSM y QEMU Guest Agent?

A. VDSM se ejecuta en el host; Guest Agent se ejecuta dentro de la VM.  
B. Ambos son nombres diferentes del mismo servicio.  
C. Guest Agent selecciona el host y VDSM configura la IP invitada.  
D. VDSM solo funciona en Windows y Guest Agent solo en Linux.

## 7. ¿Qué afirmación sobre VirtIO y Guest Agent es correcta?

A. VirtIO y Guest Agent son exactamente el mismo paquete.  
B. VirtIO proporciona dispositivos/controladores eficientes; Guest Agent informa y coordina acciones desde el invitado.  
C. Sin Guest Agent no puede existir ningún disco virtual.  
D. Guest Agent sustituye a los drivers de red.

## 8. ¿Por qué debe preferirse un apagado ordenado frente a Power Off?

A. Porque Power Off borra siempre la VM.  
B. Porque el invitado puede cerrar aplicaciones y filesystems antes de detenerse.  
C. Porque Shutdown mueve la VM al SPM.  
D. Porque Power Off cambia la MAC.

## 9. OLVM muestra un disco como conectado. ¿Qué demuestra?

A. Que el invitado lo ha particionado, formateado y montado.  
B. Que el dispositivo virtual está presentado a la VM; el uso dentro del invitado debe comprobarse aparte.  
C. Que se ha creado un nuevo Storage Domain.  
D. Que el disco es un Direct LUN.

## 10. Una vNIC está conectada y enlazada, pero la VM no tiene IP. ¿Qué interpretación es correcta?

A. Es imposible: conectada significa que DHCP ha respondido.  
B. El enlace virtual puede funcionar aunque falte DHCP o configuración IP invitada.  
C. El SPM debe asignarle una IP.  
D. Hay que borrar el vNIC Profile.

## 11. Una modificación aparece como pendiente. ¿Qué debemos comprobar?

A. Si requiere un apagado/arranque para materializarse en la VM en ejecución.  
B. Si el servidor NFS puede resolver DNS público.  
C. Si el usuario ejerce de SPM.  
D. Si el snapshot es un backup.

---

# Snapshots

## 12. ¿Qué describe mejor un snapshot?

A. Una copia independiente almacenada necesariamente fuera del Storage Domain.  
B. Un punto del estado de la VM que permite volver atrás y depende de sus imágenes y almacenamiento.  
C. Un backup completo del Engine.  
D. Una plantilla que cualquier usuario puede utilizar.

## 13. ¿Por qué un snapshot no sustituye a un backup?

A. Porque no puede contener discos.  
B. Porque suele depender del original y no protege frente a la pérdida completa de ese almacenamiento.  
C. Porque solo existe en la memoria del Engine.  
D. Porque se elimina cada vez que la VM arranca.

## 14. ¿Qué puede aportar el Guest Agent durante un snapshot en vivo?

A. Ayudar a congelar y coordinar el filesystem del invitado.  
B. Copiar el Storage Domain a otro Data Center.  
C. Cambiar automáticamente la aplicación a modo backup.  
D. Convertir NFS en almacenamiento de bloques.

## 15. ¿Qué implica guardar la memoria en un snapshot?

A. Solo se conserva el fichero de configuración de la VM.  
B. Se incorpora también el estado de ejecución, con mayor tiempo y consumo.  
C. El snapshot se convierte en template.  
D. La VM deja de necesitar discos.

## 16. Durante la restauración de un snapshot, ¿para qué sirve `Preview`?

A. Para probar el estado seleccionado antes de decidir Commit o Undo.  
B. Para borrar todos los snapshots posteriores automáticamente.  
C. Para migrar la VM al host SPM.  
D. Para exportar un OVA.

## 17. Después de Preview, ¿qué diferencia hay entre `Commit` y `Undo`?

A. Commit conserva la restauración; Undo abandona la previsualización y vuelve al estado previo.  
B. Commit borra la VM; Undo borra el Storage Domain.  
C. Commit crea un usuario; Undo elimina un rol.  
D. No existe ninguna diferencia.

## 18. ¿Qué efecto puede tener acumular snapshots durante mucho tiempo?

A. Reduce siempre el uso de almacenamiento.  
B. Puede aumentar cadenas, consumo, trabajo de merge e impacto de rendimiento.  
C. Convierte automáticamente los discos en RAW preallocated.  
D. Mejora la redundancia física del NFS.

## 19. ¿Qué diferencia existe entre restaurar un snapshot y clonar desde él?

A. Restaurar afecta al estado de la VM original; clonar crea otra VM.  
B. Son exactamente la misma operación.  
C. Clonar elimina la VM original.  
D. Restaurar crea necesariamente un OVA.

---

# Templates, cloud-init y pools

## 20. ¿Para qué sirve principalmente un template?

A. Para coordinar los metadatos de todos los Storage Domains.  
B. Para proporcionar una base reutilizable de configuración, discos y software.  
C. Para transportar la RAM durante la migración.  
D. Para sustituir al Guest Agent.

## 21. ¿Por qué se sella una VM antes de crear un template?

A. Para evitar que los clones repitan identidades como machine-id, hostname o claves SSH.  
B. Para aumentar automáticamente el número de vCPU.  
C. Para crear automáticamente una VLAN.  
D. Para asignarle el rol SPM.

## 22. ¿Qué secuencia representa mejor la personalización mediante cloud-init?

A. Engine proporciona metadata/ConfigDrive y cloud-init la aplica dentro del invitado.  
B. SPM configura directamente `/etc/hostname` mediante NFS.  
C. VDSM entra por SSH y edita todos los ficheros.  
D. El bridge Linux genera usuarios y contraseñas.

## 23. OLVM ha presentado correctamente el ConfigDrive, pero el hostname no cambia. ¿Dónde investigamos primero?

A. Logs y estado de cloud-init dentro del invitado.  
B. FDB del bridge.  
C. Prioridad del SPM.  
D. Certificado del Engine.

## 24. ¿Qué caracteriza a una VM dependiente de la imagen de un template?

A. Utiliza una base de solo lectura y capas copy-on-write.  
B. No tiene discos.  
C. No necesita Storage Domain.  
D. Siempre es un backup independiente.

## 25. ¿Qué caracteriza a un clone independiente?

A. Comparte obligatoriamente todas las escrituras con la VM origen.  
B. Dispone de una copia independiente de sus discos, con mayor tiempo y consumo de creación.  
C. Solo puede arrancar en el host SPM.  
D. Conserva necesariamente la misma MAC.

## 26. ¿Thin provisioning y linked/dependent clone significan lo mismo?

A. Sí, siempre.  
B. No: uno describe asignación de bloques y el otro dependencia de una imagen base.  
C. Sí, pero solo sobre NFS.  
D. No, porque thin provisioning es una función de networking.

## 27. ¿Qué es un VM Pool?

A. Un conjunto de hosts que comparten CPU.  
B. Un conjunto de VMs derivadas de un template y ofrecidas bajo demanda.  
C. Una agrupación de LUNs iSCSI.  
D. Una colección de vNIC Profiles.

## 28. ¿Qué implica el comportamiento stateless de una VM de pool usada desde VM Portal?

A. Los cambios temporales no deben considerarse persistentes entre asignaciones/ciclos del pool.  
B. La VM no utiliza memoria RAM.  
C. La VM carece de sistema operativo.  
D. Todos los usuarios comparten simultáneamente la misma sesión.

## 29. ¿Qué ventaja y coste tienen las VMs prearrancadas de un pool?

A. Reducen la espera, pero consumen recursos mientras están preparadas.  
B. Eliminan la necesidad de hosts.  
C. Crean backups automáticos sin consumir espacio.  
D. Sustituyen las cuotas por QoS.

---

# Usuarios, roles, permisos y cuotas

## 30. ¿Cómo se construye conceptualmente un permiso en OLVM?

A. Usuario/grupo + rol + objeto.  
B. VM + SPM + VLAN.  
C. Usuario + snapshot + bridge.  
D. Data Center + Guest Agent + NFS.

## 31. ¿Qué diferencia hay entre privilegio, rol y permiso?

A. El privilegio es una acción; el rol agrupa privilegios; el permiso asigna un rol a un sujeto sobre un objeto.  
B. Son tres nombres del mismo objeto.  
C. El rol es una cuota de almacenamiento.  
D. El permiso solo contiene una contraseña.

## 32. ¿Qué describe mejor `UserVmManager` aplicado sobre `vm-alumno1`?

A. Permite administrar esa VM dentro de los privilegios del rol.  
B. Convierte al usuario automáticamente en SuperUser de todo OLVM.  
C. Le asigna la red de migración.  
D. Lo convierte en SPM.

## 33. ¿Por qué importa el objeto sobre el que se asigna un rol?

A. Porque determina el alcance y la posible herencia del permiso.  
B. Porque cambia el formato del disco a QCOW2.  
C. Porque define la versión de NFS.  
D. Porque crea un nuevo usuario en Linux.

## 34. ¿Qué relación existe entre permiso, cuota y QoS?

A. Permiso autoriza; cuota limita cantidad consumible; QoS limita o regula ritmo/prioridad.  
B. Los tres objetos son sinónimos.  
C. QoS crea usuarios y cuota crea bridges.  
D. El permiso sustituye siempre a la capacidad física.

## 35. ¿Qué sucede normalmente en modo de cuota `Audit`?

A. Se evalúan y registran incumplimientos sin aplicar el bloqueo estricto de `Enforced`.  
B. Se eliminan automáticamente las VMs que superen la cuota.  
C. Se desactiva el Data Center.  
D. Se limita el ancho de banda de cada vNIC.

## 36. Un usuario puede utilizar una plantilla, pero no crear el disco de la nueva VM. ¿Qué falta puede explicar el problema?

A. Permiso de creación de disco sobre el Storage Domain o cuota suficiente.  
B. El rol SPM en su cuenta.  
C. Una MAC idéntica a la plantilla.  
D. Acceso SSH al Engine.

---

# Live migration

## 37. En una live migration con NFS compartido, ¿qué se transfiere principalmente entre hosts?

A. Memoria y estado de ejecución.  
B. Todo el Storage Domain NFS.  
C. La base de datos completa del Engine.  
D. El rol SPM.

## 38. ¿Qué ocurre con el disco virtual almacenado en NFS durante una migración normal de computación?

A. Permanece en el Storage Domain compartido y el host destino accede a él.  
B. Se copia siempre entero por la red de management.  
C. Se convierte en local storage.  
D. Se monta directamente dentro del Engine.

## 39. ¿Qué condición es necesaria para migrar una VM entre dos hosts?

A. Ambos deben estar en el mismo Cluster, activos y con acceso a redes y storage necesarios.  
B. Ambos deben ser SPM simultáneamente.  
C. El destino debe tener la misma dirección IP que el origen.  
D. La VM debe carecer de vNIC.

## 40. ¿Cuál de estos elementos puede bloquear una migración?

A. Un dispositivo de host o passthrough que no esté disponible en el destino.  
B. Que la VM tenga un nombre.  
C. Que Engine conserve su UUID.  
D. Que el disco esté en un Storage Domain compartido.

## 41. Si no se ha definido una red específica con rol de migración, ¿qué red se utiliza normalmente?

A. La red de gestión.  
B. La red del servidor NFS obligatoriamente.  
C. La consola VNC del alumno.  
D. Una red nueva creada automáticamente para cada migración.

## 42. Una migración manual ha funcionado. ¿Qué no demuestra por sí sola?

A. Que HA y fencing estén correctamente configurados.  
B. Que el destino tenía recursos suficientes en ese momento.  
C. Que ambos hosts podían acceder al disco.  
D. Que la VM tenía permitido migrar.

---

# Parámetros de creación, HA y agentes del invitado

## 43. Al crear una VM, ¿seleccionar «AlmaLinux 9.x x64» en el campo Sistema operativo instala AlmaLinux?

A. Sí, Engine descarga e instala automáticamente el sistema operativo.  
B. No; declara el tipo de sistema esperado, pero el contenido procede del disco, la plantilla o una ISO.  
C. Sí, siempre que el Storage Domain sea NFS.  
D. No, porque ese campo solo modifica el nombre visible de la VM.

## 44. Una VM tiene 1024 MiB de memoria y 4096 MiB de memoria máxima. ¿Qué significa?

A. Arranca con 1024 MiB y puede crecer hasta 4096 MiB si la configuración y el invitado lo permiten.  
B. Reserva 4096 MiB completos en el host al arrancar.  
C. Arranca con 4096 MiB, pero el invitado solo puede ver 1024 MiB.  
D. Reparte 4096 MiB entre todas las VMs del Cluster.

## 45. En una VM sin política de pinning, ¿qué representa normalmente una vCPU?

A. Un core físico reservado permanentemente.  
B. Un socket físico completo.  
C. Una CPU virtual que el planificador ejecuta sobre CPUs físicas disponibles.  
D. Un proceso que solo puede ejecutarse en el Engine.

## 46. En el formulario de creación, ¿qué indica el icono de cadena situado junto a determinados parámetros?

A. Que la VM es un linked clone de la plantilla.  
B. Que el valor está vinculado a un tipo de instancia y puede heredarse de él.  
C. Que el disco está cifrado.  
D. Que la VM no puede migrarse.

## 47. Al crear una VM desde una plantilla, ¿qué diferencia fundamental existe entre Thin y Clone?

A. Thin conserva dependencia de la imagen base; Clone crea una copia de disco independiente.  
B. Thin usa NFS y Clone solo puede usar iSCSI.  
C. Thin impide snapshots y Clone los exige.  
D. No existe diferencia en el almacenamiento.

## 48. ¿Qué describe correctamente una asignación `Clone + QCOW2`?

A. La VM obtiene un disco independiente de la plantilla almacenado en formato QCOW2.  
B. La VM comparte necesariamente el mismo disco escribible con la plantilla.  
C. Se crea un disco RAW dependiente de la plantilla.  
D. Se clona el host KVM completo dentro del Storage Domain.

## 49. En la asignación de discos, ¿son equivalentes Destino y Perfil del disco?

A. Sí; ambos campos seleccionan exactamente el mismo objeto.  
B. No; Destino selecciona el Storage Domain y el perfil aplica políticas como QoS sobre el disco.  
C. No; Destino selecciona una red y el perfil selecciona una VLAN.  
D. Sí, pero únicamente cuando el almacenamiento es NFS.

## 50. ¿Marcar una VM como «Altamente disponible» garantiza por sí solo su recuperación automática?

A. Sí, porque la casilla instala fencing en los hosts.  
B. Sí, siempre que la VM tenga Guest Agent.  
C. No; también deben existir hosts elegibles, storage y redes accesibles, capacidad suficiente y mecanismos de aislamiento válidos.  
D. No, porque OLVM no admite alta disponibilidad de VMs.

## 51. ¿Basta con añadir un dispositivo watchdog a la VM para que el mecanismo funcione correctamente?

A. Sí; QEMU detecta por sí solo si el sistema invitado se ha bloqueado.  
B. No; el invitado necesita el driver y un daemon configurado que renueve el temporizador.  
C. Sí, pero solo si se instala `qemu-guest-agent` en el Engine.  
D. No; también es obligatorio utilizar OVN.

## 52. ¿Qué efecto puede tener una regla de afinidad obligatoria que ningún host pueda satisfacer?

A. Puede impedir el arranque o la migración de la VM por no existir un destino válido.  
B. Se convierte automáticamente en una recomendación débil.  
C. Crea un host nuevo de forma automática.  
D. Solo cambia el orden en que se muestran las VMs.

## 53. ¿Qué diferencia esencial hay entre un VM lease y el fencing?

A. El lease limita CPU y el fencing limita memoria.  
B. Son dos nombres para el mismo bloqueo almacenado en NFS.  
C. El lease protege la ejecución de una VM mediante almacenamiento compartido; el fencing aísla un host dudoso apagándolo o reiniciándolo.  
D. El lease protege el Engine y el fencing protege únicamente el SPM.

## 54. ¿Qué paquete proporciona el daemon watchdog dentro de una VM AlmaLinux 9?

A. `watchdog`  
B. `qemu-guest-agent`  
C. `vdsm`  
D. `openvswitch`

## 55. En AlmaLinux 9, ¿dónde se incluye normalmente el módulo `virtio_balloon`?

A. En un paquete independiente llamado `virtio-balloon-agent`.  
B. Dentro de `qemu-guest-agent`.  
C. Dentro del paquete `watchdog`.  
D. En los módulos del kernel, normalmente proporcionados por `kernel-modules-core`.

## 56. ¿Es necesario `qemu-guest-agent` para que funcione memory ballooning?

A. Sí; el Guest Agent reserva y libera directamente todas las páginas.  
B. No; ballooning utiliza el dispositivo VirtIO y el driver `virtio_balloon` del invitado.  
C. Sí, pero únicamente con discos QCOW2.  
D. No, porque ballooning no necesita ningún componente dentro del invitado.

---

# Casos cortos de razonamiento

## Caso 1 · El portal muestra la VM `Up`, pero no aparece ninguna IP

¿Qué capas y componentes comprobarías antes de reiniciar la VM?

## Caso 2 · El snapshot permanece en `Image Locked`

¿Qué no debemos hacer y qué evidencias consultaríamos?

## Caso 3 · Diez VMs creadas desde un template muestran el mismo hostname y las mismas claves SSH

¿Qué fase del proceso fue probablemente incorrecta?

## Caso 4 · Un usuario ve la plantilla, pero no puede crear una VM

¿Qué permisos y límites comprobarías por separado?

## Caso 5 · La VM migra a Host 2, conserva el disco y la MAC, pero pierde conectividad

¿Qué frontera del camino priorizarías?

## Caso 6 · Se pierde por completo el servidor NFS

¿Por qué los snapshots almacenados en ese mismo NFS no constituyen una recuperación suficiente?
