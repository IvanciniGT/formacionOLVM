# Día 1 · Preguntas y respuestas

Estas preguntas acompañan al material de `dia1.md`. Están pensadas para comprobar comprensión, no sólo memoria. Cada pregunta tiene una única respuesta correcta y explica por qué las demás opciones no encajan.

---

## Preguntas tipo examen

## 1. ¿Qué componente ejecuta las VMs utilizando aceleración del kernel?

A. OLVM Engine  
B. Data Warehouse  
C. QEMU utilizando KVM  
D. Storage Pool Manager

**Respuesta correcta: C.**

- A administra y coordina, pero no ejecuta la VM.
- B conserva histórico y métricas.
- C presenta el entorno virtual y utiliza KVM para acelerarlo.
- D coordina determinadas operaciones de almacenamiento.

## 2. ¿Qué secuencia representa mejor el camino de control?

A. Engine → VDSM → libvirt → QEMU/KVM  
B. Engine → SPM → PostgreSQL → QEMU  
C. VM Portal → DWH → KVM  
D. Engine → VLAN → VDSM → Storage Domain

**Respuesta correcta: A.**

- A refleja las capas principales.
- B mezcla almacenamiento y base de datos con el arranque local.
- C asigna al DWH una función de control que no tiene.
- D trata una VLAN como intermediario de gestión.

## 3. ¿Cuál es la descripción más precisa de un Data Center?

A. Un servidor que ejecuta VMs.  
B. Un contenedor lógico superior de recursos físicos y lógicos.  
C. Un grupo de VMs con el mismo sistema operativo.  
D. Una base con métricas históricas.

**Respuesta correcta: B.**

- A describe un host.
- B coincide con la arquitectura de Oracle.
- C no representa el criterio de agrupación.
- D describe el ámbito del DWH.

## 4. ¿Cuál es la función principal de VDSM?

A. Sustituir a KVM.  
B. Actuar como agente del host para el Engine.  
C. Almacenar histórico.  
D. Proporcionar únicamente la consola.

**Respuesta correcta: B.**

- A es falsa: VDSM no virtualiza CPU.
- B expresa su papel central.
- C corresponde al DWH.
- D reduce incorrectamente sus responsabilidades.

## 5. ¿Qué afirmación describe correctamente al SPM?

A. Todo el I/O de todas las VMs lo atraviesa.  
B. Es la VM que contiene el Engine.  
C. Es un rol de host que coordina metadatos y operaciones de storage.  
D. Es el servicio de autenticación.

**Respuesta correcta: C.**

- A confunde coordinación con camino de datos.
- B confunde SPM con Self-Hosted Engine.
- C resume correctamente su función.
- D pertenece a otro subsistema.

## 6. El Engine falla con VMs funcionando. ¿Qué es más probable?

A. Todas las VMs se apagan inmediatamente.  
B. Las VMs continúan, pero se pierde gestión central y se degradan funciones coordinadas.  
C. El DWH se convierte automáticamente en Engine.  
D. El SPM sustituye al Engine.

**Respuesta correcta: B.**

- A ignora que las VMs se ejecutan en hosts.
- B distingue ejecución y control.
- C asigna al DWH una función inexistente.
- D confunde dos responsabilidades distintas.

## 7. ¿Qué relación hay entre Logical Network y VLAN?

A. Toda Logical Network es obligatoriamente una VLAN.  
B. Son el mismo objeto.  
C. La Logical Network expresa conectividad y puede asociarse o no a un VLAN ID.  
D. Las VLAN sólo existen en PostgreSQL.

**Respuesta correcta: C.**

- A y B mezclan intención de OLVM con mecanismo 802.1Q.
- C separa correctamente ambos niveles.
- D ignora que la red se materializa en hosts y switches.

## 8. ¿Qué interfaz está orientada al autoservicio del usuario final?

A. Administration Portal  
B. VM Portal  
C. `virsh`  
D. Data Warehouse

**Respuesta correcta: B.**

- A administra infraestructura.
- B presenta recursos según permisos de usuario.
- C es una herramienta local de libvirt.
- D no es un portal de operación.

## 9. ¿Qué comparación con vSphere debe considerarse “sin equivalente directo”?

A. Engine y vCenter  
B. Host KVM y ESXi  
C. Storage Domain y Datastore  
D. SPM y un componente único de vSphere

**Respuesta correcta: D.**

- A, B y C son analogías funcionales útiles con límites.
- D es correcta porque forzar una pareja para el SPM genera una explicación falsa.

## 10. ¿Qué caracteriza a Self-Hosted Engine?

A. El Engine se ejecuta como VM con mecanismos hosted-engine específicos.  
B. Cada host ejecuta su propio Engine.  
C. El DWH sustituye a PostgreSQL.  
D. Las VMs se ejecutan dentro del proceso Java del Engine.

**Respuesta correcta: A.**

- A describe la topología.
- B es falsa: sigue existiendo un gestor central.
- C es falsa: el DWH utiliza PostgreSQL.
- D es falsa: las VMs se ejecutan en los hosts KVM.

---
