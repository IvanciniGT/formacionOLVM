# Día 3 · Preguntas y respuestas

Estas preguntas acompañan a `dia3_conceptos.md`. El objetivo es seguir el ciclo de vida completo de una VM y distinguir objetos que en el portal aparecen muy próximos.

Cada pregunta tiene una única respuesta correcta. Después de responder, hay que justificar por qué las demás opciones no encajan.

---

# Anatomía y ciclo de vida de una VM

## 1. Una VM aparece en estado `Down`. ¿Qué significa?

A. Se han eliminado su definición y sus discos.  
B. La VM está definida en OLVM, pero no tiene un proceso QEMU ejecutándose.  
C. VDSM ha desaparecido del host.  
D. Su Storage Domain está necesariamente inactivo.

**Respuesta correcta: B.**

- A confunde apagado con eliminación.
- B separa correctamente definición y ejecución.
- C no se deduce del estado de una VM.
- D podría impedir un arranque, pero no es el significado de `Down`.

## 2. ¿Dónde se ejecuta una VM encendida?

A. Dentro del proceso Java del Engine.  
B. En el host seleccionado, mediante QEMU/KVM.  
C. Dentro del servidor NFS.  
D. En el host que ejerce siempre de SPM.

**Respuesta correcta: B.**

- A convierte al plano de control en hipervisor.
- B identifica el plano de ejecución.
- C confunde almacenamiento y cómputo.
- D confunde coordinación de storage con ubicación de la VM.

## 3. Engine muestra una dirección IP en la ficha de una VM. ¿Qué conclusión es correcta?

A. Engine ha configurado necesariamente esa IP.  
B. La IP puede haber sido reportada por el Guest Agent; su configuración pertenece al invitado.  
C. La IP identifica el Storage Domain.  
D. La IP es siempre la del bridge del host.

**Respuesta correcta: B.**

- A atribuye a Engine una configuración que puede proceder de DHCP, cloud-init o del propio invitado.
- B distingue información reportada y configuración.
- C mezcla networking y storage.
- D confunde endpoint invitado e interfaz del host.

## 4. ¿Qué expresa la memoria física garantizada de una VM?

A. El espacio de swap creado dentro del invitado.  
B. Un compromiso de recursos considerado por la plataforma para esa VM.  
C. La memoria máxima del host.  
D. La caché disponible en el servidor NFS.

**Respuesta correcta: B.**

- A pertenece al sistema operativo invitado.
- B describe su función de planificación.
- C es un límite físico distinto.
- D no guarda relación con la memoria de la VM.

## 5. Una VM tiene 2 sockets, 2 cores por socket y 1 thread por core. ¿Cuántas vCPU presenta?

A. 2.  
B. 3.  
C. 4.  
D. 8.

**Respuesta correcta: C.**

`2 × 2 × 1 = 4` vCPU.

## 6. ¿Qué diferencia principal existe entre VDSM y QEMU Guest Agent?

A. VDSM se ejecuta en el host; Guest Agent se ejecuta dentro de la VM.  
B. Ambos son nombres diferentes del mismo servicio.  
C. Guest Agent selecciona el host y VDSM configura la IP invitada.  
D. VDSM solo funciona en Windows y Guest Agent solo en Linux.

**Respuesta correcta: A.**

- A coloca cada agente en su capa.
- B elimina una diferencia arquitectónica esencial.
- C intercambia funciones y atribuye configuración IP a VDSM.
- D es falsa.

## 7. ¿Qué afirmación sobre VirtIO y Guest Agent es correcta?

A. VirtIO y Guest Agent son exactamente el mismo paquete.  
B. VirtIO proporciona dispositivos/controladores eficientes; Guest Agent informa y coordina acciones desde el invitado.  
C. Sin Guest Agent no puede existir ningún disco virtual.  
D. Guest Agent sustituye a los drivers de red.

**Respuesta correcta: B.**

- A confunde dispositivo/controlador y proceso agente.
- B describe sus papeles.
- C es falsa: la VM puede usar discos sin agente.
- D confunde coordinación con controlador.

## 8. ¿Por qué debe preferirse un apagado ordenado frente a Power Off?

A. Porque Power Off borra siempre la VM.  
B. Porque el invitado puede cerrar aplicaciones y filesystems antes de detenerse.  
C. Porque Shutdown mueve la VM al SPM.  
D. Porque Power Off cambia la MAC.

**Respuesta correcta: B.**

- A confunde detener y borrar.
- B protege la consistencia del invitado.
- C atribuye al SPM una función inexistente.
- D no es una consecuencia normal.

## 9. OLVM muestra un disco como conectado. ¿Qué demuestra?

A. Que el invitado lo ha particionado, formateado y montado.  
B. Que el dispositivo virtual está presentado a la VM; el uso dentro del invitado debe comprobarse aparte.  
C. Que se ha creado un nuevo Storage Domain.  
D. Que el disco es un Direct LUN.

**Respuesta correcta: B.**

- A salta varias operaciones del invitado.
- B delimita correctamente la evidencia del portal.
- C confunde Virtual Disk y Storage Domain.
- D no se deduce del estado conectado.

## 10. Una vNIC está conectada y enlazada, pero la VM no tiene IP. ¿Qué interpretación es correcta?

A. Es imposible: conectada significa que DHCP ha respondido.  
B. El enlace virtual puede funcionar aunque falte DHCP o configuración IP invitada.  
C. El SPM debe asignarle una IP.  
D. Hay que borrar el vNIC Profile.

**Respuesta correcta: B.**

- A confunde capa 2 y configuración IP.
- B separa enlace y capa 3.
- C atribuye red al SPM.
- D es destructiva y no parte de la evidencia.

## 11. Una modificación aparece como pendiente. ¿Qué debemos comprobar?

A. Si requiere un apagado/arranque para materializarse en la VM en ejecución.  
B. Si el servidor NFS puede resolver DNS público.  
C. Si el usuario ejerce de SPM.  
D. Si el snapshot es un backup.

**Respuesta correcta: A.**

- A distingue definición futura y estado actual.
- B, C y D no explican el indicador de cambio pendiente.

---

# Snapshots

## 12. ¿Qué describe mejor un snapshot?

A. Una copia independiente almacenada necesariamente fuera del Storage Domain.  
B. Un punto del estado de la VM que permite volver atrás y depende de sus imágenes y almacenamiento.  
C. Un backup completo del Engine.  
D. Una plantilla que cualquier usuario puede utilizar.

**Respuesta correcta: B.**

- A describe una independencia que el snapshot no garantiza.
- B recoge su finalidad y dependencia.
- C confunde VM y Engine.
- D confunde snapshot y template/permisos.

## 13. ¿Por qué un snapshot no sustituye a un backup?

A. Porque no puede contener discos.  
B. Porque suele depender del original y no protege frente a la pérdida completa de ese almacenamiento.  
C. Porque solo existe en la memoria del Engine.  
D. Porque se elimina cada vez que la VM arranca.

**Respuesta correcta: B.**

- A es falsa: puede incluir discos.
- B explica el riesgo principal.
- C ignora las imágenes del Storage Domain.
- D no describe su ciclo de vida.

## 14. ¿Qué puede aportar el Guest Agent durante un snapshot en vivo?

A. Ayudar a congelar y coordinar el filesystem del invitado.  
B. Copiar el Storage Domain a otro Data Center.  
C. Cambiar automáticamente la aplicación a modo backup.  
D. Convertir NFS en almacenamiento de bloques.

**Respuesta correcta: A.**

- A puede mejorar la consistencia del filesystem.
- B no es función del agente.
- C no está garantizado para cualquier aplicación.
- D mezcla tecnologías.

## 15. ¿Qué implica guardar la memoria en un snapshot?

A. Solo se conserva el fichero de configuración de la VM.  
B. Se incorpora también el estado de ejecución, con mayor tiempo y consumo.  
C. El snapshot se convierte en template.  
D. La VM deja de necesitar discos.

**Respuesta correcta: B.**

- A omite el estado de memoria.
- B describe la opción y su coste.
- C y D son falsas.

## 16. Durante la restauración de un snapshot, ¿para qué sirve `Preview`?

A. Para probar el estado seleccionado antes de decidir Commit o Undo.  
B. Para borrar todos los snapshots posteriores automáticamente.  
C. Para migrar la VM al host SPM.  
D. Para exportar un OVA.

**Respuesta correcta: A.**

- A permite validar antes de aceptar.
- B no es la finalidad de Preview.
- C y D pertenecen a operaciones diferentes.

## 17. Después de Preview, ¿qué diferencia hay entre `Commit` y `Undo`?

A. Commit conserva la restauración; Undo abandona la previsualización y vuelve al estado previo.  
B. Commit borra la VM; Undo borra el Storage Domain.  
C. Commit crea un usuario; Undo elimina un rol.  
D. No existe ninguna diferencia.

**Respuesta correcta: A.**

- A identifica las dos decisiones.
- B y C inventan efectos destructivos ajenos.
- D elimina una decisión esencial.

## 18. ¿Qué efecto puede tener acumular snapshots durante mucho tiempo?

A. Reduce siempre el uso de almacenamiento.  
B. Puede aumentar cadenas, consumo, trabajo de merge e impacto de rendimiento.  
C. Convierte automáticamente los discos en RAW preallocated.  
D. Mejora la redundancia física del NFS.

**Respuesta correcta: B.**

- A afirma lo contrario del riesgo habitual.
- B resume los costes operativos.
- C no sucede automáticamente.
- D no modifica la infraestructura física.

## 19. ¿Qué diferencia existe entre restaurar un snapshot y clonar desde él?

A. Restaurar afecta al estado de la VM original; clonar crea otra VM.  
B. Son exactamente la misma operación.  
C. Clonar elimina la VM original.  
D. Restaurar crea necesariamente un OVA.

**Respuesta correcta: A.**

- A separa cambio del original y creación de otro objeto.
- B, C y D son falsas.

---

# Templates, cloud-init y pools

## 20. ¿Para qué sirve principalmente un template?

A. Para coordinar los metadatos de todos los Storage Domains.  
B. Para proporcionar una base reutilizable de configuración, discos y software.  
C. Para transportar la RAM durante la migración.  
D. Para sustituir al Guest Agent.

**Respuesta correcta: B.**

- A corresponde al ámbito de SPM/storage.
- B define un template.
- C corresponde a migración.
- D confunde despliegue y comunicación invitada.

## 21. ¿Por qué se sella una VM antes de crear un template?

A. Para evitar que los clones repitan identidades como machine-id, hostname o claves SSH.  
B. Para aumentar automáticamente el número de vCPU.  
C. Para crear automáticamente una VLAN.  
D. Para asignarle el rol SPM.

**Respuesta correcta: A.**

- A evita duplicidad de identidad.
- B, C y D no son efectos del sellado.

## 22. ¿Qué secuencia representa mejor la personalización mediante cloud-init?

A. Engine proporciona metadata/ConfigDrive y cloud-init la aplica dentro del invitado.  
B. SPM configura directamente `/etc/hostname` mediante NFS.  
C. VDSM entra por SSH y edita todos los ficheros.  
D. El bridge Linux genera usuarios y contraseñas.

**Respuesta correcta: A.**

- A recorre las dos capas reales.
- B atribuye personalización al SPM.
- C no es el mecanismo normal de cloud-init.
- D atribuye configuración del SO a capa 2.

## 23. OLVM ha presentado correctamente el ConfigDrive, pero el hostname no cambia. ¿Dónde investigamos primero?

A. Logs y estado de cloud-init dentro del invitado.  
B. FDB del bridge.  
C. Prioridad del SPM.  
D. Certificado del Engine.

**Respuesta correcta: A.**

- A comprueba la capa que debe consumir los datos.
- B pertenece a forwarding Ethernet.
- C pertenece a storage.
- D no explica que el invitado ignore metadata ya entregada.

## 24. ¿Qué caracteriza a una VM dependiente de la imagen de un template?

A. Utiliza una base de solo lectura y capas copy-on-write.  
B. No tiene discos.  
C. No necesita Storage Domain.  
D. Siempre es un backup independiente.

**Respuesta correcta: A.**

- A describe la dependencia de imagen.
- B y C son falsas.
- D confunde despliegue y backup.

## 25. ¿Qué caracteriza a un clone independiente?

A. Comparte obligatoriamente todas las escrituras con la VM origen.  
B. Dispone de una copia independiente de sus discos, con mayor tiempo y consumo de creación.  
C. Solo puede arrancar en el host SPM.  
D. Conserva necesariamente la misma MAC.

**Respuesta correcta: B.**

- A contradice la independencia.
- B resume el intercambio entre autonomía, tiempo y capacidad.
- C confunde ubicación y SPM.
- D produciría conflictos y no es el comportamiento esperado.

## 26. ¿Thin provisioning y linked/dependent clone significan lo mismo?

A. Sí, siempre.  
B. No: uno describe asignación de bloques y el otro dependencia de una imagen base.  
C. Sí, pero solo sobre NFS.  
D. No, porque thin provisioning es una función de networking.

**Respuesta correcta: B.**

- A y C mezclan dos dimensiones distintas.
- B las separa.
- D coloca thin provisioning en la capa equivocada.

## 27. ¿Qué es un VM Pool?

A. Un conjunto de hosts que comparten CPU.  
B. Un conjunto de VMs derivadas de un template y ofrecidas bajo demanda.  
C. Una agrupación de LUNs iSCSI.  
D. Una colección de vNIC Profiles.

**Respuesta correcta: B.**

- A describe parcialmente un Cluster.
- B define el pool.
- C pertenece a storage.
- D pertenece a networking.

## 28. ¿Qué implica el comportamiento stateless de una VM de pool usada desde VM Portal?

A. Los cambios temporales no deben considerarse persistentes entre asignaciones/ciclos del pool.  
B. La VM no utiliza memoria RAM.  
C. La VM carece de sistema operativo.  
D. Todos los usuarios comparten simultáneamente la misma sesión.

**Respuesta correcta: A.**

- A explica por qué no se guardan datos importantes localmente.
- B y C son falsas.
- D contradice la asignación controlada del pool.

## 29. ¿Qué ventaja y coste tienen las VMs prearrancadas de un pool?

A. Reducen la espera, pero consumen recursos mientras están preparadas.  
B. Eliminan la necesidad de hosts.  
C. Crean backups automáticos sin consumir espacio.  
D. Sustituyen las cuotas por QoS.

**Respuesta correcta: A.**

- A recoge el beneficio y el coste.
- B, C y D son falsas.

---

# Usuarios, roles, permisos y cuotas

## 30. ¿Cómo se construye conceptualmente un permiso en OLVM?

A. Usuario/grupo + rol + objeto.  
B. VM + SPM + VLAN.  
C. Usuario + snapshot + bridge.  
D. Data Center + Guest Agent + NFS.

**Respuesta correcta: A.**

- A identifica sujeto, capacidades y alcance.
- B, C y D mezclan objetos sin formar una autorización.

## 31. ¿Qué diferencia hay entre privilegio, rol y permiso?

A. El privilegio es una acción; el rol agrupa privilegios; el permiso asigna un rol a un sujeto sobre un objeto.  
B. Son tres nombres del mismo objeto.  
C. El rol es una cuota de almacenamiento.  
D. El permiso solo contiene una contraseña.

**Respuesta correcta: A.**

- A describe el modelo RBAC.
- B elimina las capas.
- C confunde autorización y capacidad.
- D confunde autenticación y autorización.

## 32. ¿Qué describe mejor `UserVmManager` aplicado sobre `vm-alumno1`?

A. Permite administrar esa VM dentro de los privilegios del rol.  
B. Convierte al usuario automáticamente en SuperUser de todo OLVM.  
C. Le asigna la red de migración.  
D. Lo convierte en SPM.

**Respuesta correcta: A.**

- A conserva el alcance del objeto.
- B confunde un permiso concreto con administración global.
- C y D no pertenecen al rol.

## 33. ¿Por qué importa el objeto sobre el que se asigna un rol?

A. Porque determina el alcance y la posible herencia del permiso.  
B. Porque cambia el formato del disco a QCOW2.  
C. Porque define la versión de NFS.  
D. Porque crea un nuevo usuario en Linux.

**Respuesta correcta: A.**

- A explica el alcance.
- B, C y D son efectos ajenos a un permiso.

## 34. ¿Qué relación existe entre permiso, cuota y QoS?

A. Permiso autoriza; cuota limita cantidad consumible; QoS limita o regula ritmo/prioridad.  
B. Los tres objetos son sinónimos.  
C. QoS crea usuarios y cuota crea bridges.  
D. El permiso sustituye siempre a la capacidad física.

**Respuesta correcta: A.**

- A separa tres preguntas diferentes.
- B y C mezclan objetos.
- D ignora límites reales de infraestructura.

## 35. ¿Qué sucede normalmente en modo de cuota `Audit`?

A. Se evalúan y registran incumplimientos sin aplicar el bloqueo estricto de `Enforced`.  
B. Se eliminan automáticamente las VMs que superen la cuota.  
C. Se desactiva el Data Center.  
D. Se limita el ancho de banda de cada vNIC.

**Respuesta correcta: A.**

- A permite validar una política antes de imponerla.
- B no es el comportamiento de Audit.
- C confunde cuota y estado del Data Center.
- D corresponde a QoS de red.

## 36. Un usuario puede utilizar una plantilla, pero no crear el disco de la nueva VM. ¿Qué falta puede explicar el problema?

A. Permiso de creación de disco sobre el Storage Domain o cuota suficiente.  
B. El rol SPM en su cuenta.  
C. Una MAC idéntica a la plantilla.  
D. Acceso SSH al Engine.

**Respuesta correcta: A.**

- A identifica dos validaciones independientes.
- B confunde rol técnico de host y autorización de usuario.
- C sería un riesgo, no un requisito.
- D no forma parte del autoservicio normal.

---

# Live migration

## 37. En una live migration con NFS compartido, ¿qué se transfiere principalmente entre hosts?

A. Memoria y estado de ejecución.  
B. Todo el Storage Domain NFS.  
C. La base de datos completa del Engine.  
D. El rol SPM.

**Respuesta correcta: A.**

- A describe la migración de computación.
- B y C serían transferencias distintas y enormes.
- D no forma parte de la VM.

## 38. ¿Qué ocurre con el disco virtual almacenado en NFS durante una migración normal de computación?

A. Permanece en el Storage Domain compartido y el host destino accede a él.  
B. Se copia siempre entero por la red de management.  
C. Se convierte en local storage.  
D. Se monta directamente dentro del Engine.

**Respuesta correcta: A.**

- A utiliza la propiedad compartida del dominio.
- B confunde migración de ejecución y storage migration.
- C rompería el acceso compartido.
- D convierte al Engine en camino de datos.

## 39. ¿Qué condición es necesaria para migrar una VM entre dos hosts?

A. Ambos deben estar en el mismo Cluster, activos y con acceso a redes y storage necesarios.  
B. Ambos deben ser SPM simultáneamente.  
C. El destino debe tener la misma dirección IP que el origen.  
D. La VM debe carecer de vNIC.

**Respuesta correcta: A.**

- A reúne requisitos fundamentales.
- B es imposible: hay un SPM activo por Data Center/pool.
- C produciría un conflicto de host.
- D no es necesaria.

## 40. ¿Cuál de estos elementos puede bloquear una migración?

A. Un dispositivo de host o passthrough que no esté disponible en el destino.  
B. Que la VM tenga un nombre.  
C. Que Engine conserve su UUID.  
D. Que el disco esté en un Storage Domain compartido.

**Respuesta correcta: A.**

- A crea una dependencia física del host origen.
- B y C son propiedades normales.
- D facilita la migración.

## 41. Si no se ha definido una red específica con rol de migración, ¿qué red se utiliza normalmente?

A. La red de gestión.  
B. La red del servidor NFS obligatoriamente.  
C. La consola VNC del alumno.  
D. Una red nueva creada automáticamente para cada migración.

**Respuesta correcta: A.**

- A es el fallback habitual.
- B puede compartir infraestructura, pero no se selecciona por ser NFS.
- C no es una red de host para migración.
- D no se crea automáticamente.

## 42. Una migración manual ha funcionado. ¿Qué no demuestra por sí sola?

A. Que HA y fencing estén correctamente configurados.  
B. Que el destino tenía recursos suficientes en ese momento.  
C. Que ambos hosts podían acceder al disco.  
D. Que la VM tenía permitido migrar.

**Respuesta correcta: A.**

- A requiere mecanismos adicionales, especialmente fencing y configuración de HA.
- B, C y D sí quedan respaldadas para esa operación concreta y ese momento.

---

# Parámetros de creación, HA y agentes del invitado

## 43. Al crear una VM, ¿seleccionar «AlmaLinux 9.x x64» en el campo Sistema operativo instala AlmaLinux?

A. Sí, Engine descarga e instala automáticamente el sistema operativo.  
B. No; declara el tipo de sistema esperado, pero el contenido procede del disco, la plantilla o una ISO.  
C. Sí, siempre que el Storage Domain sea NFS.  
D. No, porque ese campo solo modifica el nombre visible de la VM.

**Respuesta correcta: B.**

El campo Sistema operativo es una declaración sobre el invitado que esperamos ejecutar. OLVM puede emplearla para escoger valores predeterminados compatibles, presentar opciones adecuadas y describir correctamente la VM, pero no copia los archivos de AlmaLinux ni realiza una instalación. El sistema real debe existir ya en el disco de una plantilla o imagen, o instalarse arrancando desde una ISO o por PXE.

Por eso A y C son falsas: ni Engine descarga el sistema por elegirlo ni NFS cambia ese comportamiento. D tampoco es correcta, porque el campo sí tiene significado técnico; simplemente no sustituye al medio de instalación.

## 44. Una VM tiene 1024 MiB de memoria y 4096 MiB de memoria máxima. ¿Qué significa?

A. Arranca con 1024 MiB y puede crecer hasta 4096 MiB si la configuración y el invitado lo permiten.  
B. Reserva 4096 MiB completos en el host al arrancar.  
C. Arranca con 4096 MiB, pero el invitado solo puede ver 1024 MiB.  
D. Reparte 4096 MiB entre todas las VMs del Cluster.

**Respuesta correcta: A.**

Los 1024 MiB representan la memoria definida con la que arranca la VM. Los 4096 MiB son el techo hasta el que podría ampliarse, por ejemplo mediante hot plug de memoria si la versión de compatibilidad, la configuración y el sistema invitado lo soportan. Ese máximo no equivale a una reserva inmediata de 4 GiB.

La cantidad que OLVM se compromete a mantener se expresa mediante la memoria física garantizada, que es otro valor. En el aula, definida y garantizada son ambas 1024 MiB: la VM comienza con 1 GiB y no hay margen práctico para que ballooning recupere memoria por debajo de esa garantía.

## 45. En una VM sin política de pinning, ¿qué representa normalmente una vCPU?

A. Un core físico reservado permanentemente.  
B. Un socket físico completo.  
C. Una CPU virtual que el planificador ejecuta sobre CPUs físicas disponibles.  
D. Un proceso que solo puede ejecutarse en el Engine.

**Respuesta correcta: C.**

Una vCPU es una unidad de ejecución que QEMU presenta al invitado y que el planificador del host ejecuta sobre los procesadores físicos disponibles. Sin pinning, puede ejecutarse en distintos hilos físicos a lo largo del tiempo y competir con otras vCPU. No implica una correspondencia exclusiva de uno a uno con un core.

La dedicación aparece cuando se configura una política de CPU adecuada, pinning explícito y, según el objetivo, aislamiento de CPUs en el host. El Engine decide la colocación, pero la ejecución real ocurre en el host KVM; por eso D confunde plano de gestión y plano de datos.

## 46. En el formulario de creación, ¿qué indica el icono de cadena situado junto a determinados parámetros?

A. Que la VM es un linked clone de la plantilla.  
B. Que el valor está vinculado a un tipo de instancia y puede heredarse de él.  
C. Que el disco está cifrado.  
D. Que la VM no puede migrarse.

**Respuesta correcta: B.**

El icono de cadena relaciona el parámetro con un tipo de instancia. Un tipo de instancia agrupa valores de hardware virtual —por ejemplo memoria, CPU o dispositivos— para reutilizarlos de forma coherente. Si se rompe el vínculo o se elige Personalizado, la VM conserva su propio valor y deja de seguir esa definición para el parámetro correspondiente.

No describe la dependencia de los discos respecto a una plantilla. Esa relación se decide en Asignación de recursos mediante Thin o Clone. Tampoco indica cifrado ni una restricción de migración; son conceptos independientes.

## 47. Al crear una VM desde una plantilla, ¿qué diferencia fundamental existe entre Thin y Clone?

A. Thin conserva dependencia de la imagen base; Clone crea una copia de disco independiente.  
B. Thin usa NFS y Clone solo puede usar iSCSI.  
C. Thin impide snapshots y Clone los exige.  
D. No existe diferencia en el almacenamiento.

**Respuesta correcta: A.**

Con Thin, la nueva VM utiliza una capa de escritura propia sobre la imagen base de la plantilla. Se crea con rapidez y consume inicialmente poco espacio, pero mantiene una dependencia de esa cadena de imágenes. Con Clone se copian los datos necesarios para producir un disco independiente; tarda más y consume más almacenamiento, pero elimina esa dependencia operativa de la base.

La decisión no depende de que el Storage Domain sea NFS o iSCSI. Tampoco significa que uno de los modos prohíba snapshots. El punto clave es la relación entre el disco nuevo y la imagen base de la plantilla.

## 48. ¿Qué describe correctamente una asignación `Clone + QCOW2`?

A. La VM obtiene un disco independiente de la plantilla almacenado en formato QCOW2.  
B. La VM comparte necesariamente el mismo disco escribible con la plantilla.  
C. Se crea un disco RAW dependiente de la plantilla.  
D. Se clona el host KVM completo dentro del Storage Domain.

**Respuesta correcta: A.**

Clone y QCOW2 responden a preguntas distintas. Clone indica que el disco resultante deja de depender de la imagen base de la plantilla. QCOW2 indica el formato en el que se almacena ese disco virtual y permite funciones como asignación dinámica y snapshots internos al formato. Por tanto, un clon independiente puede ser perfectamente QCOW2.

No se comparte un disco escribible con la plantilla y no se está clonando el host. Tampoco debe confundirse independencia con formato RAW: un disco independiente puede utilizar QCOW2 o RAW según las opciones admitidas y seleccionadas.

## 49. En la asignación de discos, ¿son equivalentes Destino y Perfil del disco?

A. Sí; ambos campos seleccionan exactamente el mismo objeto.  
B. No; Destino selecciona el Storage Domain y el perfil aplica políticas como QoS sobre el disco.  
C. No; Destino selecciona una red y el perfil selecciona una VLAN.  
D. Sí, pero únicamente cuando el almacenamiento es NFS.

**Respuesta correcta: B.**

Destino responde a «¿dónde se guardará el disco?» y selecciona un Storage Domain, por ejemplo `curso-nfs`. Perfil del disco responde a «¿con qué política podrá utilizar ese almacenamiento?» y puede asociar QoS de almacenamiento u otras restricciones administrativas. Son objetos diferentes aunque en el aula aparezcan con nombres iguales, lo que puede inducir a error.

El tipo NFS no fusiona ambos conceptos y ninguno selecciona redes o VLAN. Para entender una incidencia de rendimiento hay que comprobar tanto el dominio que aloja el disco como el perfil aplicado.

## 50. ¿Marcar una VM como «Altamente disponible» garantiza por sí solo su recuperación automática?

A. Sí, porque la casilla instala fencing en los hosts.  
B. Sí, siempre que la VM tenga Guest Agent.  
C. No; también deben existir hosts elegibles, storage y redes accesibles, capacidad suficiente y mecanismos de aislamiento válidos.  
D. No, porque OLVM no admite alta disponibilidad de VMs.

**Respuesta correcta: C.**

La casilla expresa que Engine debe tratar de reiniciar la VM en otro host cuando corresponda, pero no crea por sí sola las condiciones necesarias. Debe existir otro host activo y compatible, con CPU y memoria suficientes, acceso a los Storage Domains y a las redes de la VM. En un diseño normal también se necesita gestión de energía y fencing para confirmar que el host dudoso ha quedado aislado antes de arrancar otra instancia.

El Guest Agent aporta visibilidad y operaciones útiles, pero no sustituye esos requisitos. En el laboratorio anidado, marcar la casilla sirve para estudiar la política; no demuestra que todo el circuito de HA esté validado.

## 51. ¿Basta con añadir un dispositivo watchdog a la VM para que el mecanismo funcione correctamente?

A. Sí; QEMU detecta por sí solo si el sistema invitado se ha bloqueado.  
B. No; el invitado necesita el driver y un daemon configurado que renueve el temporizador.  
C. Sí, pero solo si se instala `qemu-guest-agent` en el Engine.  
D. No; también es obligatorio utilizar OVN.

**Respuesta correcta: B.**

OLVM presenta a la VM un watchdog virtual, habitualmente el modelo `i6300esb`, y configura qué acción debe realizarse si expira. Dentro del invitado, el driver del kernel expone `/dev/watchdog` y un daemon debe abrirlo y renovar periódicamente el temporizador. Si el sistema se bloquea o el daemon deja de alimentarlo, QEMU aplica la acción configurada, como reset o power off.

El dispositivo por sí solo no sabe distinguir un sistema sano de uno bloqueado. `qemu-guest-agent` y OVN no forman parte de esta cadena. En AlmaLinux 9, el daemon procede del paquete `watchdog`.

## 52. ¿Qué efecto puede tener una regla de afinidad obligatoria que ningún host pueda satisfacer?

A. Puede impedir el arranque o la migración de la VM por no existir un destino válido.  
B. Se convierte automáticamente en una recomendación débil.  
C. Crea un host nuevo de forma automática.  
D. Solo cambia el orden en que se muestran las VMs.

**Respuesta correcta: A.**

Una regla obligatoria —enforcing— es una condición dura para el planificador. Si exige separar dos VMs pero solo queda un host, o fija una VM a un conjunto de hosts que están inactivos o sin recursos, Engine no puede escoger un destino válido. El resultado puede ser un fallo de arranque, migración o recuperación HA.

Una regla preferente o débil sí puede incumplirse cuando no existe alternativa, pero OLVM no rebaja automáticamente una regla obligatoria ni crea infraestructura nueva. Por eso hay que evaluar estas reglas también durante mantenimientos y fallos, no solo en operación normal.

## 53. ¿Qué diferencia esencial hay entre un VM lease y el fencing?

A. El lease limita CPU y el fencing limita memoria.  
B. Son dos nombres para el mismo bloqueo almacenado en NFS.  
C. El lease protege la ejecución de una VM mediante almacenamiento compartido; el fencing aísla un host dudoso apagándolo o reiniciándolo.  
D. El lease protege el Engine y el fencing protege únicamente el SPM.

**Respuesta correcta: C.**

El VM lease es un arrendamiento asociado a una VM y guardado en un Storage Domain. Mientras se renueva, ayuda a impedir que la misma VM se considere arrancable simultáneamente en otro host tras un fallo ambiguo. Su alcance es esa VM y depende del almacenamiento compartido que conserva el lease.

El fencing actúa sobre el host completo a través de su gestión de energía o BMC: lo apaga, reinicia o aísla para garantizar que ya no ejecuta ninguna VM. Son protecciones complementarias contra la doble ejecución; el lease no es una cuota ni sustituye automáticamente al fencing en todos los diseños.

## 54. ¿Qué paquete proporciona el daemon watchdog dentro de una VM AlmaLinux 9?

A. `watchdog`  
B. `qemu-guest-agent`  
C. `vdsm`  
D. `openvswitch`

**Respuesta correcta: A.**

El paquete de AlmaLinux 9 se llama `watchdog`. Instala el daemon que puede supervisar comprobaciones configuradas, alimentar `/dev/watchdog` y ejecutarse mediante `watchdog.service`; su configuración principal está en `/etc/watchdog.conf`. Antes de activarlo hay que comprobar que OLVM presenta el dispositivo, que el kernel lo detecta y que la acción elegida es segura para la práctica.

`qemu-guest-agent` ofrece comunicación de gestión entre host e invitado, VDSM pertenece al host OLVM y Open vSwitch trata la conmutación de red. Ninguno sustituye al daemon watchdog dentro de la VM.

## 55. En AlmaLinux 9, ¿dónde se incluye normalmente el módulo `virtio_balloon`?

A. En un paquete independiente llamado `virtio-balloon-agent`.  
B. Dentro de `qemu-guest-agent`.  
C. Dentro del paquete `watchdog`.  
D. En los módulos del kernel, normalmente proporcionados por `kernel-modules-core`.

**Respuesta correcta: D.**

`virtio_balloon` es un driver del kernel Linux, no un daemon de usuario. En una instalación normal de AlmaLinux 9 se distribuye con los módulos del kernel, habitualmente en `kernel-modules-core`; no existe la necesidad ordinaria de instalar un supuesto agente `virtio-balloon-agent`. Puede comprobarse con `modinfo virtio_balloon`, `lsmod` y `rpm -qf` sobre la ruta del módulo.

El paquete `qemu-guest-agent` cubre otro canal de comunicación y `watchdog` instala el supervisor del temporizador. Conviene separar esos componentes aunque todos aparezcan como herramientas de integración de una VM KVM.

## 56. ¿Es necesario `qemu-guest-agent` para que funcione memory ballooning?

A. Sí; el Guest Agent reserva y libera directamente todas las páginas.  
B. No; ballooning utiliza el dispositivo VirtIO y el driver `virtio_balloon` del invitado.  
C. Sí, pero únicamente con discos QCOW2.  
D. No, porque ballooning no necesita ningún componente dentro del invitado.

**Respuesta correcta: B.**

Ballooning emplea un dispositivo VirtIO presentado por QEMU y el driver `virtio_balloon` dentro del sistema invitado. Cuando el host necesita recuperar RAM, el balloon se infla: el driver reserva páginas dentro de la VM y QEMU puede devolver esa memoria al host. Por eso sí necesita cooperación del invitado, pero no a través de `qemu-guest-agent`.

El Guest Agent proporciona información y operaciones como apagado ordenado o coordinación de determinadas tareas, pero es un canal separado. El formato del disco tampoco interviene. Además, si memoria definida y garantizada son iguales, como en la captura del aula, el dispositivo existe pero no ofrece margen práctico de recuperación por debajo de esa garantía.

---

# Casos cortos de razonamiento

## Caso 1 · El portal muestra la VM `Up`, pero no aparece ninguna IP

**Respuesta esperada:**

Comprobar si el Guest Agent está instalado y activo, si la vNIC está conectada y enlazada, si utiliza el perfil correcto y si el invitado tiene DHCP o configuración estática. También se revisan consola, `ip -br address`, rutas y logs del Guest Agent. Reiniciar no es la primera prueba.

## Caso 2 · El snapshot permanece en `Image Locked`

**Respuesta esperada:**

No iniciar otro snapshot, clon, borrado o movimiento del disco. Revisar tareas y eventos del portal, capacidad del Storage Domain, latencia NFS y logs de VDSM/Engine según el alcance. Un merge puede tardar sin estar bloqueado definitivamente.

## Caso 3 · Diez VMs creadas desde un template muestran el mismo hostname y las mismas claves SSH

**Respuesta esperada:**

El molde no se selló/generalizó correctamente o la personalización de primer arranque no se aplicó. Revisar machine-id, claves SSH de host, estado de cloud-init y configuración Initial Run. No corregir únicamente el nombre visible en Engine.

## Caso 4 · Un usuario ve la plantilla, pero no puede crear una VM

**Respuesta esperada:**

Separar permiso de uso de template, creación de VM, creación de disco sobre el Storage Domain y uso del vNIC Profile. Después comprobar cuota, capacidad de CPU/RAM y espacio real. Ver la plantilla solo demuestra una parte.

## Caso 5 · La VM migra a Host 2, conserva el disco y la MAC, pero pierde conectividad

**Respuesta esperada:**

Priorizar la materialización de la Logical Network en Host 2: perfil, TAP, bridge, uplink, VLAN, MTU y DHCP. Que el disco funcione demuestra storage, no red de VM.

## Caso 6 · Se pierde por completo el servidor NFS

**Respuesta esperada:**

Los snapshots guardados en ese mismo Storage Domain dependen del almacenamiento perdido. No proporcionan una copia independiente ni otro dominio de fallo. Hace falta un backup externo con retención y procedimiento de recuperación probado.
