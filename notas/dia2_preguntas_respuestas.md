# Día 2 · Preguntas y respuestas

Estas preguntas acompañan al material de `dia2_conceptos.md`. El objetivo no es aprender frases de memoria, sino recorrer correctamente los caminos de almacenamiento y red.

Cada pregunta tiene una única respuesta correcta. Después de responder, hay que justificar por qué las demás opciones no encajan.

---

# Storage Domains y NFS

## 1. En la instalación del curso, ¿qué describe mejor a `curso-nfs`?

A. Es el host que desempeña el rol SPM.  
B. Es un Data Storage Domain NFS V5 activo que ejerce el rol de dominio maestro.  
C. Es un repositorio OpenStack Glance separado.  
D. Es un filesystem creado dentro de una VM.

**Respuesta correcta: B.**

- A confunde un rol de host con un rol del dominio.
- B recoge los datos mostrados por el portal.
- C describe `ovirt-image-repository`.
- D confunde el filesystem invitado con el almacenamiento administrado por OLVM.

## 2. ¿Qué conclusión puede obtenerse únicamente porque `curso-nfs` y `curso-nfs-2` muestran 3666 GiB totales y 2700 GiB libres?

A. Disponemos con seguridad de 7332 GiB de capacidad física independiente.  
B. Los dos dominios son réplicas automáticas.  
C. Debemos comprobar servidor, export y filesystem antes de sumar la capacidad.  
D. Uno de los dos dominios está inactivo.

**Respuesta correcta: C.**

- A suma capacidad aparente sin verificar el backend.
- B deduce una replicación que la pantalla no demuestra.
- C interpreta la igualdad como una pista que necesita validación.
- D contradice la pantalla: ambos aparecen activos.

## 3. ¿Cuál es el orden correcto desde el backend hasta el sistema invitado?

A. Export NFS → Storage Domain → Virtual Disk → dispositivo de la VM.  
B. Storage Domain → SPM → Engine → filesystem de la VM.  
C. Virtual Disk → servidor NFS → bridge → VM.  
D. SPM → VLAN → NFS → vNIC.

**Respuesta correcta: A.**

- A recorre las capas reales de almacenamiento.
- B coloca Engine y SPM en el camino de datos de la VM.
- C mezcla almacenamiento y networking.
- D une objetos que no forman una cadena de E/S.

## 4. ¿Qué afirmación describe mejor el acceso de una VM a un disco almacenado en NFS?

A. Toda la E/S atraviesa el host SPM.  
B. El host donde corre la VM accede al Storage Domain compartido.  
C. La VM monta directamente el export administrativo de OLVM.  
D. El Engine lee cada bloque y se lo entrega a QEMU.

**Respuesta correcta: B.**

- A confunde coordinación con transporte de datos.
- B describe el camino normal de E/S.
- C omite la capa de Virtual Disk y expondría el backend de forma incorrecta.
- D convierte al Engine en un proxy de datos, función que no realiza.

## 5. ¿Qué función principal tiene el SPM?

A. Resolver DNS para los hosts.  
B. Coordinar metadatos y determinadas operaciones de almacenamiento.  
C. Conmutar las tramas de todas las VMs.  
D. Ejecutar el sistema operativo invitado.

**Respuesta correcta: B.**

- A pertenece a la infraestructura de red/nombres.
- B define su papel.
- C corresponde al plano de red de los hosts.
- D corresponde a QEMU/KVM en el host de la VM.

## 6. Un Storage Domain NFS falla solo en el Host 2. ¿Qué enfoque inicial es más útil?

A. Formatear de nuevo el export.  
B. Reiniciar todos los hosts a la vez.  
C. Comparar DNS, rutas, conectividad, montaje y logs de Host 2 con un host sano.  
D. Eliminar el dominio del Data Center.

**Respuesta correcta: C.**

- A es destructiva y no parte de la evidencia.
- B aumenta el impacto y destruye la comparación.
- C utiliza el alcance del fallo para aislar la capa.
- D no corrige la causa y puede poner en riesgo los datos.

## 7. ¿Qué demuestra el estado `Active` de un Storage Domain?

A. Que su latencia es siempre baja.  
B. Que todas las VMs almacenadas están encendidas.  
C. Que el dominio está disponible administrativamente para el Data Center, no que su rendimiento sea perfecto.  
D. Que el servidor NFS es físicamente redundante.

**Respuesta correcta: C.**

- A confunde disponibilidad con rendimiento.
- B mezcla estado de storage y estado de VMs.
- C establece el alcance correcto del estado.
- D deduce una arquitectura que el portal no demuestra.

## 8. ¿Qué práctica es correcta ante un error de permisos en el export NFS?

A. Aplicar `chmod 777` y dejarlo así.  
B. Verificar identidad de VDSM, ownership, permisos y reglas de export antes de cambiar nada.  
C. Borrar los metadatos del Storage Domain.  
D. Crear una VLAN nueva.

**Respuesta correcta: B.**

- A puede ocultar la causa y degradar la seguridad.
- B sigue un diagnóstico por capas.
- C puede destruir el dominio.
- D no responde a un error de permisos del filesystem.

## 9. ¿Qué significa que `ovirt-image-repository` aparezca como Imagen/OpenStack Glance y separado?

A. Es el Data Domain maestro del Data Center.  
B. Es el SPM de respaldo.  
C. Es un repositorio de imágenes separado, no uno de los dos Data Domains NFS activos.  
D. Todos los discos de VM dependen necesariamente de él.

**Respuesta correcta: C.**

- A describe otro rol y otro dominio.
- B vuelve a confundir dominio con rol de host.
- C interpreta la fila mostrada.
- D atribuye una dependencia que no existe por el mero hecho de aparecer en la lista.

## 10. Dentro de una VM Linux se crea XFS sobre el nuevo disco. ¿Qué se ha creado?

A. Un nuevo Storage Domain de OLVM.  
B. Un filesystem dentro del sistema invitado.  
C. Un export NFS en el host.  
D. Un nuevo SPM.

**Respuesta correcta: B.**

- A confunde el almacenamiento de OLVM con el uso del disco por el invitado.
- B identifica la capa correcta.
- C necesitaría configuración en un servidor NFS, no dentro de la VM.
- D no guarda relación con crear un filesystem.

---

# Networking nativo

## 11. ¿Qué secuencia representa mejor el camino de una trama desde una VM hacia la red física?

A. vNIC → TAP/vnet → bridge Linux → VLAN/bond/NIC → switch físico.  
B. vNIC → Engine → PostgreSQL → switch físico.  
C. vNIC → SPM → NFS → NIC física.  
D. vNIC → Data Warehouse → bridge → DNS.

**Respuesta correcta: A.**

- A recorre el plano de datos del host.
- B coloca el plano de gestión en cada paquete.
- C mezcla red y almacenamiento.
- D atribuye conmutación al DWH.

## 12. Dos VMs están conectadas al mismo bridge del mismo host. ¿Cómo puede circular su tráfico?

A. Tiene que salir siempre al switch físico.  
B. Puede ir de TAP a bridge y de bridge a TAP sin salir del host.  
C. Tiene que atravesar el Engine.  
D. Debe escribirse primero en el Storage Domain.

**Respuesta correcta: B.**

- A ignora la conmutación local del bridge.
- B describe el camino dentro del host.
- C confunde administración con plano de datos.
- D confunde tramas Ethernet con E/S de disco.

## 13. Dos VMs de la misma red están en hosts distintos y no se comunican. Sí se comunican si están en el mismo host. ¿Cuál es la frontera más probable que debemos investigar?

A. El filesystem de la VM.  
B. El SPM y los snapshots.  
C. Uplinks, VLAN y red física entre los hosts.  
D. El Data Warehouse.

**Respuesta correcta: C.**

- A no explica la diferencia entre comunicación local y remota.
- B pertenece al almacenamiento.
- C es la parte del camino que solo interviene al cruzar hosts.
- D no conmuta tráfico.

## 14. ¿Qué diferencia hay entre Logical Network y bridge Linux?

A. Son exactamente el mismo objeto con dos nombres.  
B. La Logical Network expresa la red administrada por OLVM; el bridge es un mecanismo de capa 2 en el host.  
C. El bridge solo existe en la base de datos del Engine.  
D. La Logical Network es un cable físico.

**Respuesta correcta: B.**

- A elimina la diferencia entre intención y materialización.
- B separa correctamente las capas.
- C ignora que el bridge existe en el kernel Linux.
- D confunde objeto lógico con infraestructura física.

## 15. Una Logical Network no tiene VLAN ID. ¿Qué significa necesariamente?

A. No puede transportar tráfico de VMs.  
B. OLVM la transporta obligatoriamente por la VLAN 0.  
C. En ese punto se utiliza tráfico sin una etiqueta VLAN configurada para esa red.  
D. El host debe usar SR-IOV.

**Respuesta correcta: C.**

- A es falsa: puede existir una red de VMs sin VLAN.
- B inventa una etiqueta obligatoria.
- C describe la ausencia de tagging configurado para la Logical Network.
- D no guarda relación necesaria con el VLAN ID.

## 16. ¿Puede un bridge transportar tráfico de VMs sin tener dirección IP propia?

A. Sí, porque puede conmutar tramas en capa 2.  
B. No, todo bridge necesita una puerta de enlace.  
C. Solo si el Engine está instalado en ese host.  
D. Solo si el SPM le asigna una IP.

**Respuesta correcta: A.**

- A distingue conmutación y participación IP.
- B exige una configuración de capa 3 que no siempre es necesaria.
- C no guarda relación con la función del bridge.
- D atribuye al SPM una función de red que no tiene.

## 17. ¿Dónde se configura normalmente la IP de una VM?

A. En el Data Storage Domain.  
B. En el sistema operativo invitado, sobre su vNIC.  
C. En el SPM.  
D. En la base de datos DWH.

**Respuesta correcta: B.**

- A almacena discos, no configura TCP/IP del invitado.
- B identifica el endpoint IP real.
- C coordina storage.
- D almacena información histórica.

## 18. ¿Qué relación describe mejor Logical Network, vNIC Profile y vNIC?

A. La Logical Network indica la red; el perfil define modalidad/política; la vNIC es el adaptador concreto.  
B. Los tres términos son sinónimos.  
C. El perfil reemplaza a la NIC física.  
D. La vNIC crea automáticamente el switch físico.

**Respuesta correcta: A.**

- A asigna a cada objeto su papel.
- B elimina niveles importantes de configuración.
- C confunde política con hardware.
- D atribuye a OLVM control automático sobre el switch exterior.

## 19. ¿Para qué sirve un MAC Pool?

A. Para almacenar discos RAW.  
B. Para administrar rangos de MAC y reducir colisiones al crear vNICs.  
C. Para elegir el host SPM.  
D. Para montar exports NFS.

**Respuesta correcta: B.**

- A pertenece a storage.
- B describe su finalidad.
- C pertenece a la coordinación de almacenamiento.
- D pertenece al acceso NFS.

## 20. ¿Qué afirmación sobre SR-IOV es correcta?

A. Siempre ofrece más flexibilidad que un bridge y no afecta a la migración.  
B. Permite asignar Virtual Functions de una NIC a VMs, con rendimiento alto y posibles restricciones operativas.  
C. Es necesario para toda Logical Network con VLAN.  
D. Sustituye al servidor NFS.

**Respuesta correcta: B.**

- A ignora sus requisitos y restricciones.
- B resume el equilibrio entre rendimiento y flexibilidad.
- C mezcla dos tecnologías independientes.
- D confunde red de datos y almacenamiento.

## 21. ¿Qué describe correctamente la Logical Network `alumnos` del laboratorio?

A. Es una red de storage NFS con VLAN 200.  
B. Es una VM Network del Data Center `Curso`, sin VLAN configurada y con MTU 1500.  
C. Es una red del Data Center `Default` que no pueden usar las VMs.  
D. Es un proveedor de imágenes OpenStack Glance.

**Respuesta correcta: B.**

- A inventa una función y una VLAN que no aparecen.
- B recoge las propiedades observadas.
- C la asigna al Data Center equivocado.
- D confunde networking y repositorios de imágenes.

## 22. La lista muestra `ovirtmgmt` en `Default` y otro `ovirtmgmt` en `Curso`. ¿Qué interpretación es correcta?

A. Es necesariamente el mismo objeto global mostrado dos veces.  
B. Son Logical Networks contextualizadas en Data Centers distintos, aunque compartan nombre.  
C. Uno de ellos es el SPM.  
D. Los dos deben tener la misma configuración física.

**Respuesta correcta: B.**

- A ignora el ámbito del Data Center.
- B diferencia correctamente los objetos.
- C confunde red y almacenamiento.
- D deduce una igualdad de configuración que el nombre no garantiza.

## 23. ¿Qué implica que el perfil `alumnos` utilice `vdsm-no-mac-spoofing`?

A. Limita la capacidad del disco de la VM.  
B. Ayuda a impedir que la VM emita con MAC de origen no autorizadas.  
C. Añade automáticamente una VLAN.  
D. Convierte la vNIC en SR-IOV.

**Respuesta correcta: B.**

- A pertenece a storage.
- B describe la finalidad del filtro.
- C confunde filtrado de MAC y etiquetado 802.1Q.
- D confunde filtro de red con passthrough de hardware.

## 24. Hay 14 VMs asociadas al perfil `alumnos`. ¿Qué consecuencia administrativa tiene?

A. El perfil puede borrarse sin revisar nada porque no contiene discos.  
B. Debemos revisar dependencias e impacto antes de modificarlo o eliminarlo.  
C. Las 14 VMs tienen necesariamente la misma dirección IP.  
D. El perfil pasa automáticamente a ser el Data Domain maestro.

**Respuesta correcta: B.**

- A ignora a sus consumidores.
- B aplica gestión segura de dependencias.
- C confunde perfil compartido e identidad IP.
- D mezcla networking y storage.

## 25. El Data Center `Default` aparece `No inicializado`, mientras `Curso` está `Funcionando`. ¿Cuál es la mejor interpretación?

A. El Engine está averiado.  
B. Todos los hosts han perdido NFS.  
C. `Curso` es el entorno operativo; `Default` no dispone de todos los elementos necesarios para inicializarse.  
D. Una VM no tiene dirección IP.

**Respuesta correcta: C.**

- A no encaja con un Data Center operativo en el mismo Engine.
- B contradice los dos Data Domains activos de `Curso`.
- C relaciona correctamente los estados observados.
- D no determina la inicialización de un Data Center.

---

# Casos cortos de diagnóstico

## Caso 1 · El dominio está activo, pero las VMs sufren pausas de E/S

¿Qué debemos comprobar?

**Respuesta esperada:** latencia y errores en la VM, host, red de almacenamiento, servidor NFS y backend. `Active` no certifica rendimiento.

## Caso 2 · Host 1 monta NFS; Host 2 recibe `Permission denied`

¿Qué comparación hacemos primero?

**Respuesta esperada:** identidad efectiva, resolución, dirección origen, reglas del export, ownership/permisos y opciones de montaje entre ambos hosts.

## Caso 3 · La VM alcanza el gateway por IP, pero no un nombre DNS

¿Qué parte del camino ya está demostrada?

**Respuesta esperada:** enlace virtual, configuración IP básica y routing hasta el gateway. Se investiga DNS antes que bridge, VLAN o Storage Domain.

## Caso 4 · Después de migrar, la VM se comunica localmente en Host 2 pero no sale de él

¿Qué capas priorizamos?

**Respuesta esperada:** bridge-uplink, VLAN permitida, bond, MTU y switch físico del Host 2.

## Caso 5 · Una vNIC aparece conectada, pero no recibe IP

¿Qué no demuestra el estado “conectada”?

**Respuesta esperada:** no demuestra que DHCP funcione, que exista una configuración estática correcta, que haya gateway ni que DNS responda.

---

# Ronda rápida para decir en voz alta

1. `Storage Domain` se parece conceptualmente a… **un datastore**.  
2. Nuestro backend práctico del día 2 es… **NFS**.  
3. El host que coordina operaciones de storage ejerce el rol… **SPM**.  
4. El switch de capa 2 del kernel Linux es… **el bridge**.  
5. La política/modalidad de conexión de una vNIC es… **el vNIC Profile**.  
6. La segmentación etiquetada IEEE 802.1Q es… **la VLAN**.  
7. La tabla de MAC aprendidas por un bridge es… **la FDB**.  
8. La IP de una VM se configura normalmente… **dentro del invitado**.  
9. El Engine pertenece principalmente al plano de… **control/gestión**.  
10. El tráfico entre hosts utiliza… **los uplinks y la red física**.
