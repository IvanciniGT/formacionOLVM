# Día 4 · Preguntas de repaso y tipo examen

Estas preguntas acompañan a `dia4_conceptos.md`. Cada pregunta tiene una única respuesta correcta. Además de marcarla, el alumno debe justificar por qué las otras opciones no encajan.

---

# Scheduler y afinidad

## 1. ¿Qué diferencia principal existe entre un filtro y un peso del scheduler?

A. Un filtro elimina hosts incompatibles; un peso ordena los hosts que siguen siendo válidos.  
B. Un filtro solo afecta a redes y un peso solo a discos.  
C. Los dos eliminan siempre el host.  
D. Los dos son únicamente informativos.

## 2. ¿Qué intenta conseguir una regla VM–VM negativa?

A. Ejecutar las VMs en hosts distintos.  
B. Ejecutar todas las VMs en el mismo host.  
C. Excluir los hosts incluidos en el grupo.  
D. Impedir que las VMs tengan red.

## 3. ¿Qué significa marcar `Enforcing` en una regla de afinidad?

A. La regla pasa a ser obligatoria y puede impedir el arranque si no existe un destino válido.  
B. La regla solo se utiliza para ordenar la pantalla.  
C. Engine puede ignorarla siempre.  
D. La VM se convierte automáticamente en altamente disponible.

## 4. Una regla VM–Host positiva contiene Host 1 y Host 2. ¿Qué expresa?

A. La VM debe o prefiere ejecutarse en los hosts incluidos, según sea obligatoria o preferente.  
B. La VM debe evitar siempre Host 1 y Host 2.  
C. Host 1 y Host 2 deben ejecutar la misma VM simultáneamente.  
D. Los hosts pasan a compartir una dirección IP.

## 5. ¿Qué diferencia una etiqueta de afinidad de un tag administrativo?

A. La etiqueta de afinidad condiciona la selección de host; el tag clasifica objetos, pero no decide su colocación.  
B. No existe ninguna diferencia.  
C. El tag configura el scheduler y la etiqueta solo cambia el icono.  
D. Ambos crean automáticamente una red lógica.

---

# Alta disponibilidad

## 6. ¿Qué describe mejor la alta disponibilidad de una VM en OLVM?

A. Una segunda VM ejecutándose permanentemente en paralelo.  
B. El reinicio de la VM en un host elegible después de detectar y aislar un fallo.  
C. Una copia diaria de sus discos.  
D. Una migración que nunca interrumpe el guest.

## 7. ¿Qué diferencia principal existe entre migración en vivo y recuperación HA?

A. En una migración planificada colaboran origen y destino; en HA el origen puede no responder y la VM se arranca de nuevo.  
B. HA solo mueve discos y la migración solo mueve memoria.  
C. Son dos nombres para la misma operación.  
D. HA exige siempre que Engine esté instalado dentro de la VM.

## 8. Un host aparece `Non Operational`. ¿Qué interpretación es la más correcta?

A. Engine no tiene ninguna comunicación con el host.  
B. Engine se comunica con el host, pero detecta una condición de configuración que impide operarlo normalmente.  
C. Todas sus VMs han sido eliminadas.  
D. El BMC ha confirmado necesariamente que está apagado.

## 9. ¿Qué indica un host `Non Responsive`?

A. Engine no puede comunicarse correctamente con VDSM y el estado de la ejecución puede ser incierto.  
B. El host está en mantenimiento planificado.  
C. Solo ha fallado una vNIC de una VM.  
D. El host está apagado de forma verificada.

## 10. ¿Cuál es el objetivo principal del fencing?

A. Aumentar la CPU de las VMs.  
B. Aislar un host dudoso y garantizar que no pueda seguir ejecutando las VMs antes de recuperarlas en otro.  
C. Convertir un dominio NFS en almacenamiento local.  
D. Sustituir los backups de Engine.

## 11. ¿Por qué suele intervenir otro host como proxy de fencing?

A. Para ejecutar el agente que accede a la gestión de energía del host afectado cuando Engine no lo hace directamente.  
B. Para copiar todos los discos al proxy.  
C. Porque el SPM debe ser siempre el host que se apaga.  
D. Para instalar el Guest Agent dentro de las VMs.

## 12. ¿Qué limitación tiene la demostración de HA en nuestro laboratorio?

A. NFS no permite que una VM arranque en otro host.  
B. Los hosts son anidados y no existe gestión de energía/BMC real configurada para probar fencing empresarial.  
C. Engine no conoce a los dos hosts.  
D. OLVM no admite HA con dos hosts.

## 13. ¿Qué es un VM lease?

A. Una cuota de espacio para la VM.  
B. Un bloqueo renovable guardado en almacenamiento compartido que ayuda a evitar una doble ejecución.  
C. El contrato de soporte de la VM.  
D. El permiso que convierte al host en SPM.

## 14. ¿Qué componente alimenta normalmente el watchdog virtual de una VM?

A. Un daemon dentro del sistema operativo invitado.  
B. El servidor NFS.  
C. El navegador del administrador.  
D. El SPM mediante la base de datos de Engine.

## 15. ¿Qué ocurre cuando vence el watchdog virtual?

A. VDSM recibe directamente un heartbeat perdido y borra la VM.  
B. QEMU aplica la acción configurada para el dispositivo, como reset o power off.  
C. Se activa automáticamente el fencing del host.  
D. Se crea un snapshot completo.

## 16. ¿Qué significa disponer de capacidad N+1?

A. Tener una VM más que hosts.  
B. Poder perder un host y seguir alojando las cargas previstas con sus restricciones.  
C. Mantener un snapshot por cada VM.  
D. Reservar un vCPU adicional en cada guest.

## 17. Engine deja de estar disponible, pero los hosts y las VMs siguen funcionando. ¿Qué es esperable?

A. Todas las VMs se apagan inmediatamente.  
B. Las VMs que ya se ejecutaban pueden continuar, pero se pierden temporalmente las decisiones y operaciones centrales.  
C. Cada VM se convierte en Engine.  
D. NFS borra los leases de forma automática.

---

# Optimización y recursos

## 18. ¿Para qué sirven los CPU shares?

A. Para fijar una frecuencia de CPU garantizada.  
B. Para establecer prioridad relativa cuando varias VMs compiten por CPU.  
C. Para seleccionar la VLAN de una VM.  
D. Para reservar huge pages en NFS.

## 19. ¿Qué implica el overcommit de CPU?

A. Asignar en total más vCPU que CPU física disponible, confiando en que no todas demanden el máximo a la vez.  
B. Duplicar físicamente cada procesador.  
C. Desactivar el scheduler del host.  
D. Reservar una CPU física completa para Engine.

## 20. ¿Qué precaución requiere contar los threads SMT como cores?

A. Un thread SMT no aporta necesariamente el mismo rendimiento que un núcleo físico completo.  
B. Impide utilizar KVM.  
C. Convierte automáticamente todas las VMs en no migrables.  
D. Obliga a usar almacenamiento de bloques.

## 21. ¿Qué compromiso introduce el CPU pinning?

A. Mejora siempre la migración y la densidad.  
B. Puede aportar consistencia, pero reduce flexibilidad de scheduling y destinos de migración.  
C. Solo cambia el nombre visible de las vCPU.  
D. Sustituye la afinidad entre VMs.

## 22. ¿Por qué es relevante NUMA para VMs grandes?

A. Porque el acceso a memoria local y remota puede tener costes diferentes.  
B. Porque NUMA configura direcciones IP.  
C. Porque elimina la necesidad de RAM física.  
D. Porque decide qué host es SPM.

## 23. ¿Qué expresa la memoria física garantizada?

A. El máximo que la VM podrá tener durante toda su vida.  
B. El compromiso mínimo de memoria que OLVM considera para la planificación de la VM.  
C. El swap configurado dentro del guest.  
D. La memoria reservada para el rol SPM.

## 24. ¿Qué necesita memory ballooning para funcionar de forma útil?

A. Soporte habilitado en la plataforma y un dispositivo/driver compatible en el guest, además de margen entre memoria definida y garantizada.  
B. Únicamente `qemu-guest-agent` en Engine.  
C. Un disco RAW en vez de QCOW2.  
D. Un host que sea SPM.

## 25. Una VM tiene 1024 MiB definidos y 1024 MiB garantizados. ¿Qué margen práctico ofrece ballooning por debajo de esa cifra?

A. 4096 MiB.  
B. Ninguno, porque la cantidad garantizada coincide con la definida.  
C. Depende únicamente del tamaño del disco.  
D. La mitad de la RAM del host.

## 26. ¿Qué intenta hacer KSM?

A. Compartir páginas de memoria idénticas entre procesos o VMs para reducir consumo físico.  
B. Cifrar el tráfico de migración.  
C. Bloquear un host mediante su BMC.  
D. Convertir snapshots en backups.

## 27. ¿Qué compromiso introducen las huge pages?

A. Pueden reducir trabajo de traducción de memoria, pero requieren planificación y pueden reducir flexibilidad.  
B. Siempre liberan toda la memoria no utilizada.  
C. Solo sirven para almacenamiento NFS.  
D. Sustituyen a NUMA y al pinning.

## 28. ¿Qué aportan los I/O threads a QEMU?

A. Separan parte del procesamiento de E/S del hilo principal y pueden mejorar concurrencia.  
B. Envían eventos por correo.  
C. Alimentan el watchdog.  
D. Gestionan el BMC del host.

## 29. ¿Qué consecuencia puede tener un perfil de alto rendimiento con pinning, NUMA y dispositivos directos?

A. Mayor libertad para migrar a cualquier host.  
B. Menos destinos elegibles y más exigencias operativas a cambio de rendimiento más predecible.  
C. Eliminación de todos los requisitos de capacidad.  
D. Conversión automática de NFS en vSAN.

---

# Eventos, tareas y logs

## 30. ¿Qué diferencia mejor un evento de un log?

A. El evento registra un hecho desde la perspectiva de OLVM; el log contiene detalle técnico producido por un componente.  
B. Solo el evento tiene fecha.  
C. Los logs se guardan únicamente dentro de las VMs.  
D. Son exactamente el mismo objeto.

## 31. ¿Qué es una tarea en OLVM?

A. Una operación con duración y estado, como copiar o mover un disco.  
B. Una gráfica histórica.  
C. Una regla del firewall.  
D. Un driver VirtIO.

## 32. ¿Cuál es el log inicial para comprender una decisión del plano de control?

A. `/var/log/ovirt-engine/engine.log`  
B. `/var/log/cloud-init-output.log` de cualquier VM  
C. `/etc/fstab` del servidor NFS  
D. El historial del navegador

## 33. Engine aceptó una operación y la envió al host, pero el host no pudo completarla. ¿Qué log es especialmente relevante?

A. `/var/log/vdsm/vdsm.log` del host elegido.  
B. El historial del navegador.  
C. El log del DHCP de una red no relacionada.  
D. Únicamente el historial del navegador.

## 34. ¿Dónde buscaríamos detalle de QEMU para una VM concreta?

A. En los logs de libvirt/QEMU del host donde se ejecuta o intentó ejecutarse.  
B. Solo en la base de datos de Engine.  
C. En el BMC del servidor NFS.  
D. En la configuración de red del invitado.

## 35. ¿Por qué se anota la hora exacta del incidente?

A. Para correlacionar eventos y logs de varios componentes en una ventana común.  
B. Para que la VM reciba más CPU.  
C. Para elegir el SPM.  
D. Para convertir el evento en una tarea.

## 36. ¿Para qué sirve `ovirt-log-collector`?

A. Para reunir logs de Engine, hosts y otros componentes en un archivo de diagnóstico.  
B. Para reparar automáticamente cualquier fallo.  
C. Para sustituir `engine.log`.  
D. Para crear plantillas OVA.

## 37. Un disco aparece `Image Locked`. ¿Cuál es la primera actuación correcta?

A. Cambiar inmediatamente su estado a mano.  
B. Identificar la tarea que mantiene el bloqueo y comprobar eventos, estado y logs antes de intervenir.  
C. Reiniciar todos los hosts.  
D. Eliminar el Storage Domain.

---

# Troubleshooting

## 38. Una VM está `Up`, pero Engine no muestra su IP. ¿Qué debemos concluir primero?

A. La VM carece necesariamente de dirección IP.  
B. Debemos distinguir entre una IP no configurada y una IP que el Guest Agent no ha reportado.  
C. El dominio NFS está lleno.  
D. El host necesita fencing.

## 39. Una migración es rechazada. ¿Qué pregunta orienta mejor el diagnóstico?

A. ¿Qué condición de elegibilidad incumple cada host candidato?  
B. ¿Podemos reiniciar todos los hosts a la vez?  
C. ¿Qué color tiene el icono de la VM?  
D. ¿Podemos borrar el disco y crearlo de nuevo?

## 40. ¿Cuál es la práctica más segura durante un diagnóstico?

A. Aplicar varios cambios simultáneos para ahorrar tiempo.  
B. Formular una hipótesis, hacer una prueba mínima, cambiar una sola cosa y verificar.  
C. Borrar primero los logs antiguos.  
D. Reiniciar Engine ante cualquier aviso.

---

# Casos cortos de razonamiento

## Caso 1 · Host no disponible y VM HA

El host 2 aparece `Non Responsive`. Una VM crítica marcada como HA no se reinicia en el host 1. ¿Qué datos pedirías antes de actuar y qué condiciones deben cumplirse para recuperarla con seguridad?

## Caso 2 · VM `Up` sin conectividad

La VM está `Up`, pero no aparece su IP y no responde. Diseña un recorrido desde la vNIC hasta la aplicación que permita localizar la capa afectada sin reiniciarla inmediatamente.

## Caso 3 · NFS falla solo en un host

`curso-nfs` aparece correctamente en el host 1, pero da problemas en el host 2. ¿Qué hipótesis priorizarías y qué evidencias de solo lectura recogerías?

## Caso 4 · Consolidación excesiva

Las VMs funcionan bien por la noche, pero durante la mañana varias presentan latencia al mismo tiempo. Hay muchas vCPU y ballooning activo. ¿Cómo distinguirías contención de CPU, memoria y almacenamiento?

## Caso 5 · Disco bloqueado

Tras cancelar aparentemente una creación de plantilla, el disco queda `Image Locked`. ¿Qué no harías y cómo reconstruirías la operación que mantiene el bloqueo?

## Caso 6 · Afinidades incompatibles

Una VM no arranca aunque Host 3 tiene CPU y memoria libres. La VM posee la etiqueta `gpu`, Host 1 está excluido por una regla obligatoria y Host 2 se encuentra en mantenimiento. ¿Cómo demostrarías por qué Host 3 tampoco es candidato y qué alternativas presentarías?
