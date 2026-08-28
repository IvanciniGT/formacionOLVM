# Simulacro 1 · Respuestas razonadas

**Aprobado:** 34 de 50. Cada pregunta vale un punto. Las preguntas de selección
múltiple solo puntúan si se eligieron todas las respuestas correctas y ninguna
incorrecta.

## Plantilla rápida

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|
| C | B | D | A,C | B | D | C | A | D | B |

| 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 |
|---|---|---|---|---|---|---|---|---|---|
| C | A | B,D | C | A | D | B | C | A | D |

| 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 |
|---|---|---|---|---|---|---|---|---|---|
| B | A,C | D | B | C | D | B | C | D | B |

| 31 | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 |
|---|---|---|---|---|---|---|---|---|---|
| C | D | B | C | D | B,D | C | B | D | C |

| 41 | 42 | 43 | 44 | 45 | 46 | 47 | 48 | 49 | 50 |
|---|---|---|---|---|---|---|---|---|---|
| B | D | C | B | D | C | B | D | C | B,D |

---

## Explicaciones

### 1. C · VDSM

VDSM es el agente del host que recibe las órdenes de Engine y administra libvirt,
QEMU, red y storage. Guest Agent corre dentro de una VM; SPM es un rol de
coordinación de almacenamiento.

### 2. B · DNS participa en certificados y comunicaciones

FQDN y resolución coherente son requisitos de identidad de la plataforma. No es
una preferencia estética: certificados, URLs, SSO y comunicación entre Engine y
hosts dependen de nombres estables.

### 3. D · Coordinar metadatos de storage

SPM coordina las operaciones que modifican imágenes y metadatos del Data Center.
No actúa como proxy por el que pase toda la E/S de las VMs.

### 4. A y C · Escrituras de QEMU y tramas de la vNIC

Ambas transportan datos de la carga. La orden de arranque y la consulta de
permisos pertenecen al plano de control.

### 5. B · Misma cadena y mismo dominio

El snapshot depende de imágenes del mismo Storage Domain. Una pérdida completa
del dominio puede destruir simultáneamente VM y snapshots; un backup debe crear
una copia independiente y recuperable.

### 6. D · TAP/vnet

QEMU envía y recibe tramas mediante una interfaz TAP que libvirt suele nombrar
`vnetN`. Esa interfaz se conecta al bridge o a la infraestructura de switching del
host.

### 7. C · Páginas iguales y copy-on-write

KSM deduplica páginas con el mismo contenido. Ballooning, opción A, utiliza un
driver dentro del invitado para reclamar memoria y es un mecanismo diferente.

### 8. A · Compatibilidad y CPU del Cluster

El Cluster agrupa hosts compatibles y establece versión, CPU y políticas de
scheduling/migración. La IP de una vNIC se configura dentro de la VM o mediante su
personalización.

### 9. D · Eventos y host-deploy

Primero se identifica la fase fallida: repositorios, SSH, certificados, paquetes,
red o compatibilidad. Borrar registros o elevar versiones sin evidencia puede
agravar el problema.

### 10. B · Sujeto, rol y objeto

El rol agrupa privilegios, el sujeto es un usuario o grupo y el objeto determina
el alcance. Cambiar el objeto puede convertir un permiso limitado en uno heredado
por muchos recursos.

### 11. C · El disco permanece en NFS

Con storage compartido se transfieren RAM y estado de ejecución. Origen y destino
abren la misma imagen; no es necesario copiar el disco completo al host destino.

### 12. A · Estado, evento, tarea y hora

Esas evidencias delimitan el síntoma y construyen la línea temporal. Después se
elige el log del componente que decidió y del que ejecutó la operación.

### 13. B y D · Información interna y apagado ordenado

Guest Agent comunica datos y permite determinadas acciones dentro del invitado.
La configuración de bonds corresponde al host; la elección del SPM corresponde a
Engine.

### 14. C · Evitar identidades duplicadas

Una plantilla sin sellar puede propagar hostname, claves SSH, `machine-id` o
configuración de red. Esto genera conflictos y dificulta la automatización.

### 15. A · Configurador soportado de Engine

`engine-setup` instala/configura las opciones elegidas y puede volver a ejecutarse
para reconfiguración soportada. KVM/QEMU ejecutan las VMs; VDSM permanece en los
hosts.

### 16. D · vNIC Profile

El perfil enlaza la vNIC con una Logical Network y puede incorporar QoS, filtro,
passthrough o port mirroring. No asigna por sí solo una IP dentro del invitado.

### 17. B · No existe colocación válida para las tres

Una anti-afinidad hard exige hosts distintos. Tres VMs y dos hosts dejan un
conjunto imposible; OLVM no debe ignorar una regla obligatoria ni crear capacidad.

### 18. C · Histórico de uso y rendimiento

Data Warehouse extrae y transforma información para análisis histórico y Grafana.
No participa en la ejecución de CPU, switching ni fencing.

### 19. A · Plano de control

`engine-backup` protege los componentes incluidos de Engine y sus bases. El backup
de discos de VMs es otro flujo y requiere API o solución de copia apropiada.

### 20. D · Cantidad frente a rendimiento

La cuota limita lo que un consumidor puede asignar en clúster/storage. QoS regula
o prioriza rendimiento de red o E/S mediante perfiles.

### 21. B · Standalone Engine

Lo decisivo no es que Engine sea físico o virtual, sino quién controla el ciclo de
vida de esa VM. Si depende de una plataforma externa, es Standalone para OLVM.

### 22. A y C · Copy-on-write y crecimiento fino

QCOW2 mantiene metadatos, backing files y asignación dinámica. No reserva
obligatoriamente todo el tamaño virtual y precisamente sí necesita metadatos.

### 23. D · Comunica con error de configuración frente a no responder

Un host `Non Operational` puede comunicarse pero incumplir una red obligatoria u
otro requisito. `Non Responsive` significa que Engine no recibe respuesta de
VDSM; no demuestra por sí solo que el host esté apagado.

### 24. B · Las VMs pueden continuar

QEMU/KVM sigue ejecutándolas en los hosts y accediendo a red/storage. Se pierde el
plano de control central hasta recuperar Engine.

### 25. C · Sistema compatible y copia externa

La restauración necesita paquetes/versiones compatibles y un backup disponible
fuera del elemento perdido. Modificar tablas o formatear storage no forma parte de
una restauración normal.

### 26. D · Appliance frente a unidad de almacenamiento

OVA empaqueta metadatos y uno o varios discos. Un disco aislado exige crear o
ajustar la VM, firmware, CPU, memoria y redes alrededor de él.

### 27. B · Dirección y propósito del flujo

Una regla de firewall necesita origen y destino. Abrir cualquier puerto desde
cualquier red amplía innecesariamente la superficie de exposición.

### 28. C · Localidad NUMA

NUMA pinning busca que vCPUs y páginas de memoria queden cerca en el mismo nodo
físico, reduciendo accesos remotos. No resuelve red, certificados ni SPM.

### 29. D · `bridge link`

`ip -br link` enumera interfaces, pero `bridge link` muestra puertos y su master.
La frase `master br-olvm` es la evidencia de pertenencia.

### 30. B · Ningún margen bajo la garantía

Si definida y garantizada coinciden, el host no debe recuperar memoria por debajo
de ese valor. El dispositivo balloon puede existir, pero la política no deja
margen práctico inferior.

### 31. C · Reelección de SPM

SPM es un rol dinámico. Engine puede asignarlo a otro host válido; no está unido
permanentemente a un servidor.

### 32. D · Objeto de alcance

El permiso se aplicó sobre una sola VM. Asignar el mismo rol sobre el Data Center o
Cluster produciría un alcance e herencia distintos.

### 33. B · Log de la VM en el host

Engine explica la decisión central; QEMU/libvirt muestra el fallo concreto al
materializar el dispositivo. Se correlacionan por VM, host y hora.

### 34. C · VM provisional creada por el primer host

El instalador usa libvirt y la appliance antes de que exista el Engine definitivo.
Después Engine registra el host y traslada su imagen al dominio compartido.

### 35. D · LAG/LACP del switch

Un bond 802.3ad exige coherencia con el switch físico. Engine puede configurar el
host, pero no crea por sí solo el LAG de un switch externo no integrado.

### 36. B y D · Replicación remota y failover/failback

DR cubre pérdida del sitio y requiere datos disponibles en otro emplazamiento y un
procedimiento de conmutación. Hot plug y ballooning son funciones locales de VM.

### 37. C · Asignación de una VM ya creada

El pool tiene un tamaño y VMs miembros. El usuario recibe una disponible; no
instala un host ni crea un Data Center al solicitarla.

### 38. B · Excluir al host anterior

Una pérdida de gestión puede ocultar un host todavía activo. Fencing evita arrancar
la misma VM en otro nodo mientras el anterior sigue escribiendo.

### 39. D · Definida pero no ejecutándose

`Down` es un estado de ciclo de vida. No implica borrar discos ni imposibilidad de
arranque posterior.

### 40. C · Data Center

Storage Domains compartidos se adjuntan y coordinan en el ámbito del Data Center.
El Cluster agrupa los hosts que los consumen.

### 41. B · E/S paralela suficiente

Multiqueue puede repartir trabajo entre varias colas y vCPUs. No mejora una carga
pequeña por obligación ni sustituye un backend insuficiente.

### 42. D · Maintenance, intervención, activación y validación

El mantenimiento planificado evacua o detiene cargas antes de intervenir. Después
se activa el host y se comprueban VDSM, redes, storage y eventos.

### 43. C · `ovirt-engine-rename`

El FQDN aparece en certificados y configuración distribuida. Cambiar solo el
hostname deja referencias incoherentes; se usa la herramienta soportada con backup
y DNS preparados.

### 44. B · Puede impedir migración

Una VF está asociada a hardware concreto. La migración requiere una estrategia
SR-IOV migrable/failover soportada o retirar la dependencia.

### 45. D · Administración colectiva

Conceder el permiso al grupo permite que las altas y bajas del directorio se
reflejen sin repetir asignaciones. El rol y su alcance siguen siendo necesarios.

### 46. C · KVM/oVirt frente a Xen

OLVM administra Oracle Linux KVM y procede de oVirt. Oracle VM clásico es otra
familia basada en Xen; no debe usarse su documentación como si fuera OLVM.

### 47. B · Engine genera los metadatos

La GUI traduce campos o YAML a un datasource que consume cloud-init dentro de la
VM. Engine no configura el switch físico ni la BMC.

### 48. D · Evitar dependencias incompatibles

Minimal Install reduce paquetes y repositorios que podrían introducir QEMU,
libvirt o bibliotecas no soportadas. Después se instalan los componentes OLVM desde
los repositorios correctos.

### 49. C · `Non Operational`

El host puede responder y, sin embargo, incumplir la red obligatoria del Cluster.
`Image Locked` corresponde a imágenes de VM, no al estado operativo del host.

### 50. B y D · Elegibilidad y dependencias

Si no existe host válido se revisan capacidad/CPU/afinidad y disponibilidad de
redes/storage. El icono y el número de dashboards no participan en el filtro del
scheduler.

---

## Diagnóstico del resultado

Tras corregir, clasifique cada error por bloque. Si un bloque concentra tres o más
fallos, repase los capítulos correspondientes del
[`manual`](../notas/manual-del-curso.md) antes de realizar el simulacro 2.
