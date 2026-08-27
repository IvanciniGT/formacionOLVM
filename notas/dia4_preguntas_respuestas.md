# Día 4 · Preguntas y respuestas razonadas

Estas preguntas acompañan a `dia4_conceptos.md`. Cada una tiene una única respuesta correcta. La explicación no se limita a identificar una letra: relaciona la respuesta con el funcionamiento real de OLVM.

---

# Scheduler y afinidad

## 1. ¿Qué diferencia principal existe entre un filtro y un peso del scheduler?

A. Un filtro elimina hosts incompatibles; un peso ordena los hosts que siguen siendo válidos.  
B. Un filtro solo afecta a redes y un peso solo a discos.  
C. Los dos eliminan siempre el host.  
D. Los dos son únicamente informativos.

**Respuesta correcta: A.**

Los filtros representan requisitos que el destino debe cumplir: estado, CPU, memoria, red, storage o reglas obligatorias. Después, los pesos comparan los destinos que han sobrevivido, por ejemplo favoreciendo menor carga o una afinidad preferente. Confundir ambos impide comprender por qué a veces ningún host aparece disponible.

## 2. ¿Qué intenta conseguir una regla VM–VM negativa?

A. Ejecutar las VMs en hosts distintos.  
B. Ejecutar todas las VMs en el mismo host.  
C. Excluir los hosts incluidos en el grupo.  
D. Impedir que las VMs tengan red.

**Respuesta correcta: A.**

La regla negativa VM–VM busca separar las VMs del grupo para que no compartan el mismo dominio de fallo. B corresponde a afinidad positiva; C sería una regla VM–Host negativa. La regla de colocación no configura la conectividad de las VMs.

## 3. ¿Qué significa marcar `Enforcing` en una regla de afinidad?

A. La regla pasa a ser obligatoria y puede impedir el arranque si no existe un destino válido.  
B. La regla solo se utiliza para ordenar la pantalla.  
C. Engine puede ignorarla siempre.  
D. La VM se convierte automáticamente en altamente disponible.

**Respuesta correcta: A.**

`Enforcing` convierte la regla en `Hard`: los hosts que no la cumplen quedan filtrados. Sin `Enforcing`, la regla es `Soft` y el scheduler puede incumplirla para mantener el servicio. La opción no activa HA y debe utilizarse solo cuando el requisito justifica dejar una VM apagada.

## 4. Una regla VM–Host positiva contiene Host 1 y Host 2. ¿Qué expresa?

A. La VM debe o prefiere ejecutarse en los hosts incluidos, según sea obligatoria o preferente.  
B. La VM debe evitar siempre Host 1 y Host 2.  
C. Host 1 y Host 2 deben ejecutar la misma VM simultáneamente.  
D. Los hosts pasan a compartir una dirección IP.

**Respuesta correcta: A.**

En una relación VM–Host, positiva significa «dentro del conjunto». Si es preferente, Engine intentará utilizar esos hosts; si es obligatoria, descartará los demás. No implica ejecutar dos copias ni modifica la red de gestión.

## 5. ¿Qué diferencia una etiqueta de afinidad de un tag administrativo?

A. La etiqueta de afinidad condiciona la selección de host; el tag clasifica objetos, pero no decide su colocación.  
B. No existe ninguna diferencia.  
C. El tag configura el scheduler y la etiqueta solo cambia el icono.  
D. Ambos crean automáticamente una red lógica.

**Respuesta correcta: A.**

El tag sirve para organización, búsqueda y administración. La etiqueta de afinidad participa en el filtro de colocación y vincula VMs con hosts etiquetados, de forma equivalente a una relación positiva obligatoria. Tampoco debe confundirse con una etiqueta de red, que facilita desplegar Logical Networks sobre NICs.

---

# Alta disponibilidad

## 6. ¿Qué describe mejor la alta disponibilidad de una VM en OLVM?

A. Una segunda VM ejecutándose permanentemente en paralelo.  
B. El reinicio de la VM en un host elegible después de detectar y aislar un fallo.  
C. Una copia diaria de sus discos.  
D. Una migración que nunca interrumpe el guest.

**Respuesta correcta: B.**

OLVM HA recupera el servicio arrancando de nuevo la VM en otro host; normalmente existe una interrupción y el sistema operativo invitado realiza un nuevo boot. A describe tolerancia a fallos con una réplica activa, C es backup y D describe una migración planificada, no la recuperación después de perder el origen.

## 7. ¿Qué diferencia principal existe entre migración en vivo y recuperación HA?

A. En una migración planificada colaboran origen y destino; en HA el origen puede no responder y la VM se arranca de nuevo.  
B. HA solo mueve discos y la migración solo mueve memoria.  
C. Son dos nombres para la misma operación.  
D. HA exige siempre que Engine esté instalado dentro de la VM.

**Respuesta correcta: A.**

En una migración en vivo, el host de origen sigue funcionando y transfiere el estado de ejecución al destino. En un fallo, ese host puede no colaborar: OLVM debe resolver la incertidumbre, aislarlo cuando corresponda y arrancar otra vez la VM desde el almacenamiento compartido.

## 8. Un host aparece `Non Operational`. ¿Qué interpretación es la más correcta?

A. Engine no tiene ninguna comunicación con el host.  
B. Engine se comunica con el host, pero detecta una condición de configuración que impide operarlo normalmente.  
C. Todas sus VMs han sido eliminadas.  
D. El BMC ha confirmado necesariamente que está apagado.

**Respuesta correcta: B.**

`Non Operational` suele indicar que Engine recibe información del host pero encuentra una condición inválida, como una red obligatoria ausente o un problema de acceso a storage. La ausencia o pérdida de comunicación se aproxima más a `Non Responsive`; ninguno de estos estados implica por sí solo eliminación de VMs.

## 9. ¿Qué indica un host `Non Responsive`?

A. Engine no puede comunicarse correctamente con VDSM y el estado de la ejecución puede ser incierto.  
B. El host está en mantenimiento planificado.  
C. Solo ha fallado una vNIC de una VM.  
D. El host está apagado de forma verificada.

**Respuesta correcta: A.**

Engine ha perdido la comunicación de gestión con el host. Eso no confirma que esté sin alimentación: podría seguir ejecutando VMs mientras falla la red de gestión o VDSM. Por eso no se debe iniciar una segunda copia hasta resolver la incertidumbre mediante los mecanismos previstos.

## 10. ¿Cuál es el objetivo principal del fencing?

A. Aumentar la CPU de las VMs.  
B. Aislar un host dudoso y garantizar que no pueda seguir ejecutando las VMs antes de recuperarlas en otro.  
C. Convertir un dominio NFS en almacenamiento local.  
D. Sustituir los backups de Engine.

**Respuesta correcta: B.**

Fencing actúa fuera del sistema operativo del host, normalmente mediante su controlador de gestión, para apagarlo o reiniciarlo de forma verificable. Su objetivo es evitar dos ejecuciones concurrentes de una misma VM y proteger el almacenamiento compartido; no es una función de capacidad ni de backup.

## 11. ¿Por qué suele intervenir otro host como proxy de fencing?

A. Para ejecutar el agente que accede a la gestión de energía del host afectado cuando Engine no lo hace directamente.  
B. Para copiar todos los discos al proxy.  
C. Porque el SPM debe ser siempre el host que se apaga.  
D. Para instalar el Guest Agent dentro de las VMs.

**Respuesta correcta: A.**

Engine selecciona un host operativo que pueda ejecutar el agente de fence y alcanzar el BMC del host dudoso. El proxy no recibe los discos ni tiene por qué ser SPM; simplemente proporciona el punto de ejecución fiable de la acción de aislamiento.

## 12. ¿Qué limitación tiene la demostración de HA en nuestro laboratorio?

A. NFS no permite que una VM arranque en otro host.  
B. Los hosts son anidados y no existe gestión de energía/BMC real configurada para probar fencing empresarial.  
C. Engine no conoce a los dos hosts.  
D. OLVM no admite HA con dos hosts.

**Respuesta correcta: B.**

El aula permite estudiar objetos, estados, storage compartido y configuración HA, pero los hosts OLVM son VMs anidadas sin un BMC físico configurado. NFS sí permite que ambos hosts accedan al disco; la limitación es demostrar de extremo a extremo el aislamiento y la recuperación propios de una instalación física.

## 13. ¿Qué es un VM lease?

A. Una cuota de espacio para la VM.  
B. Un bloqueo renovable guardado en almacenamiento compartido que ayuda a evitar una doble ejecución.  
C. El contrato de soporte de la VM.  
D. El permiso que convierte al host en SPM.

**Respuesta correcta: B.**

El host que ejecuta la VM mantiene el lease mediante el mecanismo de locking sobre storage compartido. Si deja de renovarlo, el lease ayuda a determinar que esa ejecución ya no conserva el derecho válido. No es el disco, una cuota, un contrato ni el rol coordinador SPM.

## 14. ¿Qué componente alimenta normalmente el watchdog virtual de una VM?

A. Un daemon dentro del sistema operativo invitado.  
B. El servidor NFS.  
C. El navegador del administrador.  
D. El SPM mediante la base de datos de Engine.

**Respuesta correcta: A.**

El guest dispone de un driver y un daemon watchdog que renuevan periódicamente el temporizador del dispositivo virtual. Si el invitado se bloquea y deja de alimentarlo, vence. No es un latido enviado desde el navegador, NFS o SPM.

## 15. ¿Qué ocurre cuando vence el watchdog virtual?

A. VDSM recibe directamente un heartbeat perdido y borra la VM.  
B. QEMU aplica la acción configurada para el dispositivo, como reset o power off.  
C. Se activa automáticamente el fencing del host.  
D. Se crea un snapshot completo.

**Respuesta correcta: B.**

El dispositivo watchdog forma parte del hardware virtual que presenta QEMU. Al expirar, QEMU aplica la acción definida y VDSM/Engine observan después el nuevo estado. Watchdog actúa ante un guest atascado; fencing actúa sobre un host dudoso.

## 16. ¿Qué significa disponer de capacidad N+1?

A. Tener una VM más que hosts.  
B. Poder perder un host y seguir alojando las cargas previstas con sus restricciones.  
C. Mantener un snapshot por cada VM.  
D. Reservar un vCPU adicional en cada guest.

**Respuesta correcta: B.**

No basta con contar dos o más hosts. Hay que retirar del cálculo un host —habitualmente el relevante para el diseño— y comprobar que las VMs que deban recuperarse caben en los restantes, respetando memoria, CPU, afinidad, redes, storage y dispositivos.

## 17. Engine deja de estar disponible, pero los hosts y las VMs siguen funcionando. ¿Qué es esperable?

A. Todas las VMs se apagan inmediatamente.  
B. Las VMs que ya se ejecutaban pueden continuar, pero se pierden temporalmente las decisiones y operaciones centrales.  
C. Cada VM se convierte en Engine.  
D. NFS borra los leases de forma automática.

**Respuesta correcta: B.**

QEMU/KVM ejecuta las VMs en los hosts, no dentro de Engine. La caída del plano de control no obliga a detener procesos que ya funcionan, pero impide o limita tareas centrales como nuevas decisiones de scheduling, administración y ciertas recuperaciones hasta restaurar Engine.

---

# Optimización y recursos

## 18. ¿Para qué sirven los CPU shares?

A. Para fijar una frecuencia de CPU garantizada.  
B. Para establecer prioridad relativa cuando varias VMs compiten por CPU.  
C. Para seleccionar la VLAN de una VM.  
D. Para reservar huge pages en NFS.

**Respuesta correcta: B.**

Los shares adquieren efecto cuando existe contención: una VM con mayor peso obtiene una proporción superior del tiempo de CPU frente a otra con menor peso. No expresan GHz garantizados y no guardan relación con red, NFS o huge pages.

## 19. ¿Qué implica el overcommit de CPU?

A. Asignar en total más vCPU que CPU física disponible, confiando en que no todas demanden el máximo a la vez.  
B. Duplicar físicamente cada procesador.  
C. Desactivar el scheduler del host.  
D. Reservar una CPU física completa para Engine.

**Respuesta correcta: A.**

El overcommit mejora consolidación si los picos no coinciden. Si demasiadas VMs demandan CPU simultáneamente aparece contención y aumenta la latencia. Es una decisión de capacidad que debe medirse, no la creación de recursos físicos adicionales.

## 20. ¿Qué precaución requiere contar los threads SMT como cores?

A. Un thread SMT no aporta necesariamente el mismo rendimiento que un núcleo físico completo.  
B. Impide utilizar KVM.  
C. Convierte automáticamente todas las VMs en no migrables.  
D. Obliga a usar almacenamiento de bloques.

**Respuesta correcta: A.**

Los threads de un mismo núcleo comparten recursos internos. Contarlos como capacidad puede ser válido para determinadas cargas, pero sobreestima el rendimiento si se asume que cada hilo equivale a un core completo. El efecto debe validarse con la carga real.

## 21. ¿Qué compromiso introduce el CPU pinning?

A. Mejora siempre la migración y la densidad.  
B. Puede aportar consistencia, pero reduce flexibilidad de scheduling y destinos de migración.  
C. Solo cambia el nombre visible de las vCPU.  
D. Sustituye la afinidad entre VMs.

**Respuesta correcta: B.**

El pinning relaciona vCPU con CPU físicas concretas, lo que puede reducir variabilidad y ayudar a cargas sensibles. A cambio obliga a encontrar una topología compatible y puede impedir o complicar migraciones. No reemplaza las reglas de colocación entre VMs.

## 22. ¿Por qué es relevante NUMA para VMs grandes?

A. Porque el acceso a memoria local y remota puede tener costes diferentes.  
B. Porque NUMA configura direcciones IP.  
C. Porque elimina la necesidad de RAM física.  
D. Porque decide qué host es SPM.

**Respuesta correcta: A.**

En un servidor NUMA, CPU y memoria se agrupan en nodos; acceder a la memoria del nodo local suele ser más eficiente. Una VM grande mal situada puede atravesar nodos y sufrir latencia. NUMA pertenece a CPU/memoria, no a red ni al rol SPM.

## 23. ¿Qué expresa la memoria física garantizada?

A. El máximo que la VM podrá tener durante toda su vida.  
B. El compromiso mínimo de memoria que OLVM considera para la planificación de la VM.  
C. El swap configurado dentro del guest.  
D. La memoria reservada para el rol SPM.

**Respuesta correcta: B.**

OLVM usa la memoria garantizada al decidir si un host puede alojar la VM y qué parte no debería recuperarse mediante mecanismos de consolidación. La memoria máxima es otro valor; el swap del guest y el rol coordinador SPM pertenecen a conceptos diferentes.

## 24. ¿Qué necesita memory ballooning para funcionar de forma útil?

A. Soporte habilitado en la plataforma y un dispositivo/driver compatible en el guest, además de margen entre memoria definida y garantizada.  
B. Únicamente `qemu-guest-agent` en Engine.  
C. Un disco RAW en vez de QCOW2.  
D. Un host que sea SPM.

**Respuesta correcta: A.**

El host solicita al driver `virtio_balloon` que reserve páginas dentro del invitado, permitiendo recuperar memoria física. Para obtener ahorro debe existir memoria recuperable por encima de la garantizada. QEMU Guest Agent no es quien ejecuta este mecanismo y el formato de disco o SPM no lo determinan.

## 25. Una VM tiene 1024 MiB definidos y 1024 MiB garantizados. ¿Qué margen práctico ofrece ballooning por debajo de esa cifra?

A. 4096 MiB.  
B. Ninguno, porque la cantidad garantizada coincide con la definida.  
C. Depende únicamente del tamaño del disco.  
D. La mitad de la RAM del host.

**Respuesta correcta: B.**

Aunque el dispositivo esté habilitado, OLVM no debería reducir el respaldo de la VM por debajo de lo garantizado. Al coincidir definido y garantizado, el margen de recuperación es nulo en ese punto. Esto demuestra por qué una casilla activa no implica un beneficio efectivo.

## 26. ¿Qué intenta hacer KSM?

A. Compartir páginas de memoria idénticas entre procesos o VMs para reducir consumo físico.  
B. Cifrar el tráfico de migración.  
C. Bloquear un host mediante su BMC.  
D. Convertir snapshots en backups.

**Respuesta correcta: A.**

KSM localiza páginas con el mismo contenido y las fusiona mediante copy-on-write. Puede ahorrar memoria en VMs parecidas, pero consume CPU y requiere valorar impacto de rendimiento y seguridad. No participa en migración, fencing ni protección externa de datos.

## 27. ¿Qué compromiso introducen las huge pages?

A. Pueden reducir trabajo de traducción de memoria, pero requieren planificación y pueden reducir flexibilidad.  
B. Siempre liberan toda la memoria no utilizada.  
C. Solo sirven para almacenamiento NFS.  
D. Sustituyen a NUMA y al pinning.

**Respuesta correcta: A.**

Las páginas grandes reducen entradas y fallos de TLB en determinadas cargas. Sin embargo, hay que reservarlas y mantener tamaños compatibles en los hosts candidatos; una configuración rígida puede dificultar el arranque o la migración. Complementan, no reemplazan, el diseño NUMA.

## 28. ¿Qué aportan los I/O threads a QEMU?

A. Separan parte del procesamiento de E/S del hilo principal y pueden mejorar concurrencia.  
B. Envían eventos por correo.  
C. Alimentan el watchdog.  
D. Gestionan el BMC del host.

**Respuesta correcta: A.**

Los I/O threads permiten que determinadas operaciones de dispositivos no dependan exclusivamente del hilo principal de QEMU. El beneficio depende de carga, drivers, colas y storage; no son un servicio de alertas, watchdog o gestión física.

## 29. ¿Qué consecuencia puede tener un perfil de alto rendimiento con pinning, NUMA y dispositivos directos?

A. Mayor libertad para migrar a cualquier host.  
B. Menos destinos elegibles y más exigencias operativas a cambio de rendimiento más predecible.  
C. Eliminación de todos los requisitos de capacidad.  
D. Conversión automática de NFS en vSAN.

**Respuesta correcta: B.**

Estas opciones acercan la VM a recursos físicos concretos. Eso puede reducir jitter, pero obliga a disponer de hosts equivalentes y limita las acciones del scheduler. Alto rendimiento no significa capacidad infinita ni cambia la tecnología de almacenamiento.

---

# Eventos, tareas y logs

## 30. ¿Qué diferencia mejor un evento de un log?

A. El evento registra un hecho desde la perspectiva de OLVM; el log contiene detalle técnico producido por un componente.  
B. Solo el evento tiene fecha.  
C. Los logs se guardan únicamente dentro de las VMs.  
D. Son exactamente el mismo objeto.

**Respuesta correcta: A.**

Un evento facilita una visión operativa central: objeto, severidad, hora y resultado. El log narra el trabajo interno de Engine, VDSM, libvirt u otro componente. Ambos tienen tiempo y existen logs tanto en Engine y hosts como dentro del guest.

## 31. ¿Qué es una tarea en OLVM?

A. Una operación con duración y estado, como copiar o mover un disco.  
B. Una gráfica histórica.  
C. Una regla del firewall.  
D. Un driver VirtIO.

**Respuesta correcta: A.**

Una tarea representa trabajo que puede estar pendiente, en curso, completado o fallido. Es especialmente importante en operaciones de almacenamiento porque puede mantener objetos bloqueados. Las demás respuestas pertenecen a monitorización, seguridad o dispositivos del guest.

## 32. ¿Cuál es el log inicial para comprender una decisión del plano de control?

A. `/var/log/ovirt-engine/engine.log`  
B. `/var/log/cloud-init-output.log` de cualquier VM  
C. `/etc/fstab` del servidor NFS  
D. El historial del navegador

**Respuesta correcta: A.**

`engine.log` recoge validaciones, selección, envío de órdenes y errores del plano central. Cloud-init solo relata personalización dentro de una VM, `fstab` es configuración y el historial del navegador no contiene la decisión interna tomada por Engine.

## 33. Engine aceptó una operación y la envió al host, pero el host no pudo completarla. ¿Qué log es especialmente relevante?

A. `/var/log/vdsm/vdsm.log` del host elegido.  
B. El historial del navegador.  
C. El log del DHCP de una red no relacionada.  
D. Únicamente el historial del navegador.

**Respuesta correcta: A.**

VDSM recibe las órdenes de Engine y prepara red, storage y definición de la VM en el host. Su log permite saber qué intentó, qué dependencia falló y qué respuesta devolvió. Después puede ser necesario bajar a libvirt o QEMU, pero VDSM es el primer punto de la capa host.

## 34. ¿Dónde buscaríamos detalle de QEMU para una VM concreta?

A. En los logs de libvirt/QEMU del host donde se ejecuta o intentó ejecutarse.  
B. Solo en la base de datos de Engine.  
C. En el BMC del servidor NFS.  
D. En la configuración de red del invitado.

**Respuesta correcta: A.**

El proceso QEMU se crea en el host y sus mensajes se encuentran en el journal o bajo el directorio de logs de libvirt/QEMU, según la versión. La base de datos de Engine, el BMC y la configuración de red del invitado no contienen ese relato de creación del dominio virtual.

## 35. ¿Por qué se anota la hora exacta del incidente?

A. Para correlacionar eventos y logs de varios componentes en una ventana común.  
B. Para que la VM reciba más CPU.  
C. Para elegir el SPM.  
D. Para convertir el evento en una tarea.

**Respuesta correcta: A.**

Una operación deja rastros en Engine, host, libvirt, storage y guest. La hora y una sincronización correcta permiten reconstruir el orden causal. Sin una ventana temporal se mezclan mensajes antiguos con el incidente actual y aumenta el ruido.

## 36. ¿Para qué sirve `ovirt-log-collector`?

A. Para reunir logs de Engine, hosts y otros componentes en un archivo de diagnóstico.  
B. Para reparar automáticamente cualquier fallo.  
C. Para sustituir `engine.log`.  
D. Para crear plantillas OVA.

**Respuesta correcta: A.**

La herramienta crea un paquete coherente de evidencias para análisis o soporte. No interpreta ni repara automáticamente el problema, no sustituye la lectura de `engine.log` y tampoco crea plantillas OVA. Su salida debe tratarse como sensible.

## 37. Un disco aparece `Image Locked`. ¿Cuál es la primera actuación correcta?

A. Cambiar inmediatamente su estado a mano.  
B. Identificar la tarea que mantiene el bloqueo y comprobar eventos, estado y logs antes de intervenir.  
C. Reiniciar todos los hosts.  
D. Eliminar el Storage Domain.

**Respuesta correcta: B.**

El bloqueo puede ser legítimo mientras una copia, snapshot, movimiento o plantilla continúa trabajando. Antes de forzarlo hay que reconstruir la tarea y confirmar que ha terminado o quedado realmente huérfana. Las otras acciones son invasivas y pueden provocar corrupción o aumentar el impacto.

---

# Troubleshooting

## 38. Una VM está `Up`, pero Engine no muestra su IP. ¿Qué debemos concluir primero?

A. La VM carece necesariamente de dirección IP.  
B. Debemos distinguir entre una IP no configurada y una IP que el Guest Agent no ha reportado.  
C. El dominio NFS está lleno.  
D. El host necesita fencing.

**Respuesta correcta: B.**

`Up` indica que QEMU ejecuta la VM, pero la IP que muestra Engine suele depender de la información del Guest Agent. Debemos comprobar consola, interfaz, configuración, DHCP/rutas y estado del agente antes de concluir que no existe IP. Storage y fencing no se deducen de ese síntoma.

## 39. Una migración es rechazada. ¿Qué pregunta orienta mejor el diagnóstico?

A. ¿Qué condición de elegibilidad incumple cada host candidato?  
B. ¿Podemos reiniciar todos los hosts a la vez?  
C. ¿Qué color tiene el icono de la VM?  
D. ¿Podemos borrar el disco y crearlo de nuevo?

**Respuesta correcta: A.**

El scheduler filtra destinos por CPU compatible, memoria, redes, storage, afinidad, pinning, NUMA y dispositivos, entre otras condiciones. Revisar la razón de descarte transforma un mensaje genérico en una causa comprobable. Reiniciar o borrar datos antes de identificarla es injustificado.

## 40. ¿Cuál es la práctica más segura durante un diagnóstico?

A. Aplicar varios cambios simultáneos para ahorrar tiempo.  
B. Formular una hipótesis, hacer una prueba mínima, cambiar una sola cosa y verificar.  
C. Borrar primero los logs antiguos.  
D. Reiniciar Engine ante cualquier aviso.

**Respuesta correcta: B.**

Una secuencia controlada conserva evidencias, reduce impacto y permite atribuir el resultado a una acción concreta. Los cambios múltiples y los reinicios indiscriminados pueden ocultar la causa, ampliar el incidente y dejarlo listo para repetirse.

---

# Casos cortos de razonamiento

## Caso 1 · Host no disponible y VM HA

El host 2 aparece `Non Responsive`. Una VM crítica marcada como HA no se reinicia en el host 1. ¿Qué datos pedirías antes de actuar y qué condiciones deben cumplirse para recuperarla con seguridad?

### Respuesta razonada

Primero hay que determinar si el host o la VM siguen realmente ejecutándose, si solo se ha perdido la red de gestión y si el fencing puede confirmar el aislamiento. Revisaríamos hora, eventos, comunicación de Engine con VDSM, acceso al BMC, lease de la VM y proceso QEMU cuando pueda comprobarse de forma segura.

Después confirmaríamos que el host 1 es un destino válido: CPU compatible, memoria suficiente, acceso a `curso-nfs`, red lógica disponible, ausencia de reglas de afinidad o pinning incompatibles y dispositivos requeridos. Marcar HA no elimina ninguno de esos requisitos. En el aula, la falta de fencing real limita la demostración automática.

## Caso 2 · VM `Up` sin conectividad

La VM está `Up`, pero no aparece su IP y no responde. Diseña un recorrido desde la vNIC hasta la aplicación que permita localizar la capa afectada sin reiniciarla inmediatamente.

### Respuesta razonada

Comprobaríamos que la vNIC está enlazada y conectada al perfil correcto, que la red lógica está presente y sincronizada en el host, y que TAP/vnet está conectado al bridge o camino esperado. Después revisaríamos VLAN y enlace físico si aplican.

En la consola del guest verificaríamos interfaz, dirección, ruta, DNS y firewall; luego el servicio final. También revisaríamos el Guest Agent: su ausencia puede explicar por qué Engine no muestra la IP, aunque la red funcione. Cada prueba separa capa virtual, camino físico, configuración invitada y aplicación.

## Caso 3 · NFS falla solo en un host

`curso-nfs` aparece correctamente en el host 1, pero da problemas en el host 2. ¿Qué hipótesis priorizarías y qué evidencias de solo lectura recogerías?

### Respuesta razonada

Al funcionar desde el host 1, el servidor y la exportación no están completamente caídos. Priorizaríamos red entre host 2 y NFS, resolución de nombre, firewall, opciones o estado del mount, permisos vistos por ese cliente y estado de VDSM.

Recogeríamos eventos con hora, `findmnt -t nfs,nfs4`, `nfsstat -m`, journal filtrado por NFS/timeouts y `vdsm.log`, comparándolo con el host sano. No desmontaríamos ni montaríamos a mano un dominio gestionado por OLVM como primera prueba.

## Caso 4 · Consolidación excesiva

Las VMs funcionan bien por la noche, pero durante la mañana varias presentan latencia al mismo tiempo. Hay muchas vCPU y ballooning activo. ¿Cómo distinguirías contención de CPU, memoria y almacenamiento?

### Respuesta razonada

Correlacionaríamos el mismo intervalo en métricas de hosts, VMs y storage. Para CPU observaríamos utilización, colas/espera y coincidencia de picos; para memoria, presión del host, memoria garantizada, actividad del balloon, swap y KSM; para storage, latencia, IOPS, throughput y timeouts NFS.

Que haya muchas vCPU no prueba contención y que ballooning esté activo no prueba que haya liberado memoria. Comparar la evolución simultánea y el alcance evita cambiar CPU cuando el cuello real está en NFS, o culpar al storage cuando el host está intercambiando memoria.

## Caso 5 · Disco bloqueado

Tras cancelar aparentemente una creación de plantilla, el disco queda `Image Locked`. ¿Qué no harías y cómo reconstruirías la operación que mantiene el bloqueo?

### Respuesta razonada

No cambiaríamos a mano el estado, no borraríamos el disco y no reiniciaríamos todos los hosts sin saber si la copia continúa. Localizaríamos la tarea, su hora de inicio, VM/template y host involucrados; después correlacionaríamos eventos, `engine.log`, `vdsm.log` y estado del Storage Domain.

Solo tras confirmar que la operación ya no existe y que el bloqueo quedó huérfano aplicaríamos un procedimiento documentado para esa versión, con backup de la información necesaria y posibilidad de escalado a soporte.

## Caso 6 · Afinidades incompatibles

Una VM no arranca aunque Host 3 tiene CPU y memoria libres. La VM posee la etiqueta `gpu`, Host 1 está excluido por una regla obligatoria y Host 2 se encuentra en mantenimiento. ¿Cómo demostrarías por qué Host 3 tampoco es candidato y qué alternativas presentarías?

### Respuesta razonada

Reconstruiríamos la selección host por host. Host 1 queda eliminado por la regla obligatoria; Host 2, por su estado de mantenimiento; Host 3, porque la etiqueta de afinidad `gpu` actúa como requisito de colocación y el host no la posee. La capacidad libre de Host 3 no puede compensar un filtro incumplido.

Después verificaríamos que la política del Cluster contiene el módulo `Label` y revisaríamos posibles conflictos con grupos de afinidad. Las alternativas dependen del requisito real: devolver Host 2 a servicio, etiquetar un host que realmente tenga el hardware, convertir una restricción innecesaria en preferente mediante un grupo, o mantener la VM apagada si GPU/licencia es obligatoria. No eliminaríamos una regla sin validar por qué existe.
