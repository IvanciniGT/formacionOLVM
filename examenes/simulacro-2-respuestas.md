# Simulacro 2 · Respuestas razonadas

**Aprobado:** 34 de 50. Las preguntas con varias respuestas requieren seleccionar
el conjunto completo y exacto.

## Plantilla rápida

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|
| A | C | B | D | B,D | A | D | B | C | A |

| 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 |
|---|---|---|---|---|---|---|---|---|---|
| B | D | C | A | D | B | A,C | A | D | B |

| 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 |
|---|---|---|---|---|---|---|---|---|---|
| C | B,D | D | B | A | C | D | B | A | C |

| 31 | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 |
|---|---|---|---|---|---|---|---|---|---|
| A,D | D | A | C | B | D | A | C | B | D |

| 41 | 42 | 43 | 44 | 45 | 46 | 47 | 48 | 49 | 50 |
|---|---|---|---|---|---|---|---|---|---|
| C | A | B | D | C | A | D | B | C | A |

---

## Explicaciones

### 1. A · Ejecución de la VM

La definición vive en Engine; la ejecución activa se materializa como dominio
libvirt y proceso QEMU en un host. Al apagarla desaparece el proceso, no
necesariamente el objeto ni sus discos.

### 2. C · Agotamiento del backend

Thin provisioning permite prometer más capacidad virtual que física. Funciona
mientras el uso real se mantenga dentro del backend; por eso el crecimiento debe
monitorizarse y alertarse.

### 3. B · Bootstrap y enrolado

Durante el alta, Engine prepara el host, instala VDSM, configura certificados y
descubre hardware. No se está creando una cuota ni restaurando DWH.

### 4. D · Consolidar preservando estados restantes

Las capas físicas pueden fusionarse, pero los snapshots que no se eliminan deben
seguir representando el mismo estado lógico. Borrar S1 no autoriza a destruir S2
o S3.

### 5. B y D · Autorización y scheduling

Engine valida el permiso y decide el destino. VDSM aplica localmente la orden,
incluida la preparación de QEMU y la TAP.

### 6. A · Capacidad aparente, no cores nuevos

SMT expone varios hilos por core que comparten recursos. Contarlos como cores
permite mayor densidad de scheduling, pero no duplica las unidades físicas.

### 7. D · La IP pertenece al invitado

El vNIC Profile elige red y políticas de la interfaz. DHCP, cloud-init o la
configuración del sistema operativo asignan la dirección IP.

### 8. B · Gestión frente a consumo

Un rol administrativo modifica objetos de la plataforma. Un rol de usuario
permite utilizar VMs, pools o recursos dentro del objeto donde se concedió.

### 9. C · Deploy

`Deploy` instala la configuración y los agentes de Hosted Engine en el host. Un
host regular puede ejecutar otras VMs, pero no la VM Engine.

### 10. A · Grupo de afinidad

La etiqueta agrupa entidades. El grupo define regla VM–VM o VM–host, polaridad y
si es preferente u obligatoria.

### 11. B · Export y permisos NFS

La resolución funciona; el error apunta a autorización o exportación para la IP
del host. Se revisan export, versión, opciones y acceso desde ese nodo.

### 12. D · La E/S no atraviesa siempre SPM

Cada host accede directamente al Storage Domain para sus VMs. SPM coordina
metadatos y puede cambiar de host sin convertirse en proxy permanente.

### 13. C · Observar tarea y progreso

`Image Locked` protege una imagen durante una operación exclusiva. Si progresa es
un estado correcto; forzar la base o locks puede corromper la cadena.

### 14. A · Independencia de la base

El clone completo ocupa más espacio y tarda más, pero su disco deja de depender de
la plantilla. No evita sellado ni Guest Agent.

### 15. D · Matriz de la release

La versión más nueva del sistema operativo no es necesariamente soportada por la
release instalada. Se consulta la guía antes de fijar repositorios y paquetes.

### 16. B · PCP y `pmlogger`

Performance Co-Pilot expone métricas del host y permite guardar históricos. Sirve
para ampliar lo observado en Portal/DWH con detalle del sistema operativo.

### 17. A y C · Switch y host

El host crea bond/VLAN y el switch debe permitir esa VLAN sobre un LAG coherente.
Una sola capa mal configurada rompe la conectividad.

### 18. A · Base `engine`

La base `engine` conserva inventario y configuración actual. La base histórica
almacena muestras preparadas por DWH.

### 19. D · Ubicación externa

La copia debe sobrevivir al fallo que pretende cubrir. Guardarla solo en el mismo
filesystem del Engine la hace desaparecer con él.

### 20. B · Falta autorización sobre recursos

Autenticar demuestra identidad, no permiso. El usuario necesita un rol de usuario
sobre VM, pool o un objeto superior con herencia apropiada.

### 21. C · Renombrado soportado

El nombre forma parte de certificados y URLs. Se prepara DNS y una copia y se usa
`ovirt-engine-rename`; una edición aislada deja configuración inconsistente.

### 22. B y D · Recopilación coherente para soporte

El colector reúne evidencias correlacionadas. No repara automáticamente ni
garantiza eliminar información sensible, por lo que el archivo se revisa.

### 23. D · Objeto lógico

Un Data Center no tiene que representar un edificio. Es el ámbito lógico que
relaciona clústeres, redes y almacenamiento.

### 24. B · Cloud-init preparado dentro de la imagen

Engine entrega metadatos, pero el invitado necesita cloud-init instalado y
habilitado para consumirlos. RAW o SR-IOV no son requisitos.

### 25. A · Evitar MAC spoofing

El filtro impide que la vNIC emita tramas con direcciones distintas de las
autorizadas. Puede ser incompatible con appliances que legítimamente necesitan
varias MAC y debe elegirse conscientemente.

### 26. C · Puede incumplir una preferencia soft

Una regla soft puntúa, no elimina necesariamente destinos. Si los preferidos no
son válidos, el scheduler puede mantener servicio en otro host compatible.

### 27. D · Fichero de respuestas

`engine-setup` puede generar/consumir respuestas para automatizar una configuración
repetible. El fichero contiene datos sensibles y se protege.

### 28. B · Paquete `watchdog`

El paquete instala el daemon que alimenta el dispositivo presentado a la VM. VDSM
se instala en hosts, no dentro del invitado para esta función.

### 29. A · Gestión y decisiones nuevas

Las cargas ya arrancadas siguen en KVM y acceden directamente a bridge/storage.
Engine ausente impide la operación central y nuevas decisiones coordinadas.

### 30. C · QoS mediante Disk Profile

La política QoS se asocia al perfil y el disco selecciona ese perfil. Así se
reutilizan políticas y permisos sobre un Storage Domain.

### 31. A y D · Dos capas complementarias

El lease utiliza storage compartido para coordinar la VM. Fencing actúa sobre el
host dudoso. Un lease no sustituye universalmente la exclusión física del host.

### 32. D · Informa sin bloquear

Audit mode permite observar superaciones y dimensionar cuotas. Enforcement sí
rechaza las operaciones que exceden límites.

### 33. A · CPU y seguridad

KSM escanea y compara páginas, consumiendo CPU. La deduplicación también tiene
consideraciones de aislamiento; no se activa agresivamente sin medir.

### 34. C · Aislamiento de Engine

Hosted Engine mantiene imagen, locks y metadatos especiales. Un dominio dedicado
evita mezclar su ciclo crítico con cargas generales.

### 35. B · Hardware y VFs preparados

SR-IOV requiere NIC/PF compatible, VFs habilitadas, drivers y red física coherente.
Después OLVM puede asociar perfiles y redes a las funciones disponibles.

### 36. D · Histórico degradado

DWH no ejecuta las VMs ni transporta su E/S. Su pérdida afecta principalmente a
series históricas, informes y dashboards que dependen de esa base.

### 37. A · Operación larga sobre la imagen

Copiar 500 GiB puede tardar. Se observa la tarea, backend y eventos antes de
declarar bloqueo anómalo.

### 38. C · Evitar dos sitios activos sobre los mismos datos

El sitio anterior debe quedar aislado antes de activar el pasivo. Es el mismo
principio de exclusión que fencing, aplicado al desastre de sitio.

### 39. B · Dos objetos, un fallo común

OLVM ve dos dominios, pero ambos dependen del servidor y discos comunes. No son
backup ni redundancia física.

### 40. D · Limpiar y utilizar versiones soportadas

Paquetes externos pueden alterar libvirt/QEMU y hacer el host incompatible. Es más
seguro prepararlo correctamente antes de ejecutar cargas.

### 41. C · Rendimiento frente a flexibilidad

Huge pages reducen traducciones y pueden ayudar a cargas grandes, pero requieren
reserva y dificultan ballooning, overcommit o migración según el diseño.

### 42. A · Compatibilidad y dependencias

La restauración debe reproducir un entorno compatible con el contenido de la copia
y resolver DNS, certificados, bases y acceso a storage/hosts.

### 43. B · Datos frente a gestión

VirtIO proporciona dispositivos paravirtualizados eficientes. Guest Agent comunica
información y acciones del sistema invitado a la plataforma.

### 44. D · MTU extremo a extremo

Jumbo frames solo funcionan de forma fiable si todo el camino admite la MTU. Un
switch o enlace con 1500 puede descartar o fragmentar tráfico.

### 45. C · Alcance e herencia

Un permiso sobre Cluster puede alcanzar muchas VMs y objetos inferiores. Se revisa
mínimo privilegio y se reduce el alcance cuando sea posible.

### 46. A · Mantener el requisito soportado

Usar IPv4 no exige retirar IPv6 del sistema. Se conserva habilitado cuando la guía
lo requiere y se configuran explícitamente los flujos de servicio.

### 47. D · El máximo es el techo

El hot plug no puede superar la memoria máxima expuesta/configurada sin ajustar el
modelo y, según el cambio, reiniciar. Storage y KSM no cambian ese techo.

### 48. B · Conmutación local del bridge

El bridge aprende MAC y envía la trama entre los TAP del mismo host. Engine no
participa en cada paquete.

### 49. C · Agentes HA y estado compartido

Los agentes de los hosts operan aunque el servicio Engine esté caído. Coordinan
estado, score y locks para arrancar una única VM Engine válida.

### 50. A · Correlación temporal

La hora exacta permite relacionar evento, tarea y logs de Engine, VDSM y QEMU.
Borrar o reiniciar primero puede destruir evidencia sin resolver la causa.

---

## Siguiente paso

Compare los errores de ambos simulacros. Un fallo repetido en el mismo bloque
indica una laguna conceptual; un fallo aislado suele ser lectura o confusión entre
componentes. Repase el capítulo correspondiente y repita únicamente después de
poder explicar la respuesta sin memorizar la letra.
