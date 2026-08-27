# Día 4 · Notas para impartir la clase

Estas notas no sustituyen al documento formal. Son un guion para explicar los conceptos en voz alta, decidir dónde detenerse y relacionarlos con lo que los alumnos ya conocen de vSphere.

## Idea central del día

Hoy dejamos de mirar OLVM como una colección de pantallas y empezamos a tratarlo como una plataforma en producción.

La pregunta ya no es solo:

> ¿Cómo creo una VM?

Ahora preguntamos:

> ¿Qué ocurre cuando algo falla, cómo sé qué ha pasado y cómo evito empeorarlo?

La sesión gira alrededor de cuatro ideas:

1. Engine primero descarta hosts imposibles y después ordena los válidos.
2. HA necesita eliminar la ambigüedad antes de arrancar una segunda copia.
3. Optimizar siempre implica aceptar algún compromiso.
4. Un incidente se sigue por capas y por tiempo.

## Apertura —15 minutos—

Empezar preguntando:

- ¿Por qué una VM arranca en Host 1 y no en Host 2?
- ¿Qué ocurre si le decimos que dos VMs deben permanecer separadas?
- ¿Es esa separación una preferencia o una obligación?

Dejar que aparezca la respuesta intuitiva: «Engine selecciona el host que tiene más recursos». Después introducir el matiz:

> Un host puede tener recursos y seguir siendo un destino imposible por red, CPU, afinidad, pinning o dispositivos.

Cuando entiendan cómo se forma el conjunto de candidatos, será natural preguntar qué ocurre durante un fallo.

## 1. Scheduler y afinidad

### La explicación con una lista de candidatos

Dibujar dos cajas:

```text
Filtros → ¿puede ejecutar la VM?
Pesos  → ¿cuál de los válidos conviene más?
```

Un filtro elimina. Un peso puntúa.

Ejemplo con nuestros dos hosts:

```text
Host 1: Up, red alumnos, NFS, memoria suficiente
Host 2: Up, red alumnos, NFS, memoria insuficiente

Resultado: Host 2 queda filtrado; no importa que tenga poca CPU utilizada.
```

Añadir una regla obligatoria puede eliminar un host igual que la falta de memoria. Añadir una preferencia solo cambia el orden si ambos siguen siendo válidos.

### Las cuatro combinaciones

Explicarlas siempre indicando primero **qué dos tipos de objetos relacionamos**:

| Regla | Positiva | Negativa |
|---|---|---|
| VM–VM | Juntar | Separar |
| VM–Host | Ejecutar dentro del grupo de hosts | Ejecutar fuera del grupo |

No decir simplemente «afinidad positiva significa juntar», porque con una regla VM–Host significa seleccionar los hosts incluidos, no juntar objetos en un único servidor.

### Soft frente a Hard

Usar una frase que recuerden:

> Soft expresa una preferencia; Hard acepta dejar la VM apagada.

En la GUI, `Enforcing` convierte la regla en obligatoria.

Ejemplo con dos nodos de aplicación:

- negativa y preferente: normalmente separados; durante la caída de un host pueden convivir;
- negativa y obligatoria: separados siempre; si solo queda un host, uno permanece apagado.

Preguntar a la clase:

> ¿Qué es prioritario: conservar la separación o conservar el servicio degradado?

No existe una respuesta universal. La regla debe representar la decisión de negocio.

### Tres ejemplos que funcionan bien

#### Alta disponibilidad de aplicación

Dos nodos redundantes con anti-afinidad negativa preferente. Se intenta evitar que un único host los derribe, pero se admite convivencia temporal si la infraestructura queda degradada.

#### Rendimiento

Aplicación y caché con afinidad positiva preferente. Pueden reducir tráfico externo, pero se concentran en el mismo dominio de fallo.

#### Licenciamiento o GPU

VM–Host positiva y obligatoria hacia los servidores autorizados o equipados. Si ninguno está disponible, el resultado correcto puede ser que la VM no arranque.

### Etiquetas de afinidad

Explicarlas como el atajo para una relación dura entre VM y host:

```text
VM con etiqueta gpu
        ↓
solo hosts con etiqueta gpu
```

Comparar objetos:

- tag: clasifica y permite buscar;
- network label: despliega redes sobre NICs;
- affinity label: condiciona dónde puede arrancar una VM.

Esta distinción merece una pregunta directa porque los tres conceptos utilizan la palabra «etiqueta» y no hacen lo mismo.

### El conflicto típico

Plantear:

```text
VM etiquetada gpu
Host 1 tiene gpu, pero una regla lo excluye
Host 2 tiene gpu, pero está en mantenimiento
Host 3 está libre, pero no tiene gpu
```

La VM no arranca. No falta capacidad global: la intersección de reglas ha dejado cero destinos.

### Comparación con vSphere

- positiva VM–VM: mantener juntas;
- negativa VM–VM: separar;
- VM–Host: reglas de ejecución sobre grupos de hosts;
- preferente: `Should`;
- obligatoria: `Must`.

La comparación ayuda, pero OLVM necesita que la política de scheduling contenga los módulos que aplican grupos y etiquetas.

### Práctica de afinidad

Abrir un grupo nuevo sin guardarlo:

1. seleccionar dos VMs;
2. configurar VM negativa;
3. observar `Enforcing`;
4. añadir los dos hosts;
5. cambiar la regla Host entre positiva y negativa;
6. pedir que expliquen con palabras cada combinación;
7. cancelar.

Cerrar preguntando:

> Si ahora cae un host, ¿las reglas desaparecen?

La respuesta es no. HA vuelve a pasar la VM por el scheduler.

## 2. Alta disponibilidad

### La explicación simple

Una VM normal se ejecuta en un host. Si ese host falla, el disco sigue estando en el almacenamiento compartido, pero el proceso que ejecutaba la VM ha desaparecido.

OLVM puede volver a arrancarla en otro host, siempre que se cumplan varias condiciones:

- está marcada como HA;
- Engine detecta y confirma el fallo;
- el host anterior ya no puede seguir ejecutándola;
- existe otro host compatible;
- ese host tiene recursos;
- puede acceder al disco y a las redes de la VM.

Marcar HA expresa una intención. No crea capacidad, no repara NFS y no configura por sí solo el BMC.

### Comparación con vSphere

La comparación útil es vSphere HA, no Fault Tolerance.

- En HA existe una interrupción y un nuevo arranque.
- En FT se mantiene otra ejecución sincronizada.

OLVM HA está en la primera categoría.

### Migración y HA tampoco son iguales

En mantenimiento tenemos tiempo: migramos la VM viva y luego intervenimos el host.

En un fallo inesperado ya no podemos pedir colaboración al host averiado. Hay que confirmar que no seguirá ejecutando la VM y arrancarla de nuevo en otro sitio.

Escribir en la pizarra:

| Operación planificada | Fallo inesperado |
|---|---|
| Migración | Reinicio HA |
| Ambos hosts colaboran | El origen puede no responder |
| Sin reinicio del guest | Nuevo arranque del guest |

### Non Operational y Non Responsive

No traducir ambos estados como «host roto».

- `Non Operational`: Engine habla con el host, pero su configuración no es válida para operar. Puede faltar una red requerida o existir un problema de storage/configuración.
- `Non Responsive`: Engine no consigue comunicarse con VDSM. No conoce con seguridad qué está ocurriendo dentro del host.

El segundo es el estado peligroso para HA porque existe incertidumbre.

### Fencing —la metáfora de las llaves—

Si dos personas tienen la llave del mismo coche y no sabemos dónde está una de ellas, no entregamos otra llave y suponemos que todo irá bien.

El fencing retira de forma verificable la capacidad del host dudoso para seguir ejecutando VMs. Normalmente otro host actúa como proxy y llama al agente de fence que controla el BMC del servidor —iLO, iDRAC, IPMI u otro mecanismo—.

Puntos que deben quedar muy claros:

- no es un apagado elegante del sistema operativo;
- no se ejecuta «dentro» del host que no responde;
- necesita conectividad y credenciales de gestión fuera de banda;
- debe probarse antes de necesitarlo;
- con dos hosts, perder uno puede dejar capacidad insuficiente.

### Limitación real de nuestra instalación

En el laboratorio aparecen hosts OLVM anidados y no tenemos un BMC físico real para cada uno. Por eso la GUI avisa de que la gestión de energía no está habilitada.

Podemos enseñar configuración, estados y razonamiento. No debemos presentar una simulación incompleta como prueba de HA empresarial.

En Caixa o Redeia esperaríamos, además:

- hosts físicos;
- dos caminos de gestión cuando sea necesario;
- BMC aislado y controlado;
- fencing probado;
- almacenamiento y red redundantes;
- capacidad N+1 o superior;
- procedimientos y observabilidad.

### VM lease

Explicarlo sin decir que es simplemente «un fichero que bloquea la VM».

El lease es un bloqueo renovable guardado en el almacenamiento compartido. El host que ejecuta la VM lo mantiene vivo mediante la pila de locking. Si pierde la capacidad de renovarlo, el mecanismo ayuda a impedir que dos hosts consideren válida la misma ejecución.

El lease y el fencing atacan el mismo riesgo desde ángulos distintos: doble ejecución, split-brain y posible corrupción.

No confundirlo con:

- el disco de la VM;
- un alquiler comercial;
- una cuota;
- el rol SPM.

### Watchdog

Esta corrección es importante: el watchdog de la VM **no manda latidos a VDSM**.

La VM ve un dispositivo watchdog virtual. Un daemon dentro del invitado lo alimenta periódicamente. Si el guest se bloquea y deja de hacerlo, el dispositivo vence y QEMU aplica la acción configurada, por ejemplo reset o power off. Después VDSM y Engine observan el resultado.

Cadena para la pizarra:

```text
daemon del guest
      ↓ alimenta
watchdog virtual
      ↓ si vence
QEMU ejecuta la acción
      ↓
VDSM y Engine observan el estado
```

Watchdog detecta un guest atascado. Fencing resuelve un host de estado incierto. No son intercambiables.

### Capacidad N+1

Con dos hosts al 70 % no tenemos HA real para todas las cargas. Cuando falle uno, el otro no podrá absorber el 140 %.

Usar una pregunta sencilla:

> Si quito el host más grande, ¿todo lo crítico cabe y sigue cumpliendo sus restricciones?

Eso es más útil que contar hosts.

## Práctica rápida de HA

Sin guardar cambios:

1. Abrir una VM y localizar la casilla HA.
2. Ver prioridad, acción de reanudación, lease y watchdog.
3. Revisar el host y su gestión de energía.
4. Ver afinidad y restricciones de la VM.
5. Preguntar qué impediría reiniciarla en el otro host.

No provocar una caída real del host.

## 3. Optimización

### Entrada al bloque

No presentar cada casilla como una mejora. Presentar cada una como una decisión.

> Si una opción promete más rendimiento, ¿qué flexibilidad, densidad o capacidad de migración estamos pagando?

### CPU

Separar cuatro ideas:

- vCPU: CPU que ve la VM.
- sobreasignación: prometer más vCPU que CPU física.
- shares: prioridad relativa cuando hay contención.
- pinning: limitar determinadas vCPU a CPU físicas concretas.

Los shares no reservan una frecuencia fija. Solo deciden quién recibe más tiempo cuando varias VMs compiten.

El pinning puede reducir variabilidad, pero también restringe el scheduler y puede dificultar una migración.

Recordar que contar threads como cores aumenta la capacidad aparente, pero un hilo SMT no equivale siempre a un núcleo físico completo.

### Memoria

Usar tres cantidades:

- definida: la que ve al arrancar;
- garantizada: la que OLVM debe poder respaldar;
- máxima: techo para hot-plug o crecimiento permitido.

Ballooning necesita soporte en cluster y dispositivo/driver en la VM. El host pide memoria al guest inflando el balloon. No es swap y no inventa RAM.

En la VM que vimos ayer, definida y garantizada eran ambas 1024 MiB. Aunque ballooning estuviera marcado, no había margen práctico por debajo de 1 GiB.

KSM intenta compartir páginas de memoria idénticas. Puede ayudar en cargas parecidas, pero consume CPU y tiene implicaciones de seguridad y rendimiento.

Huge pages reducen trabajo de traducción de memoria, pero deben reservarse y pueden reducir flexibilidad. No activarlas como remedio genérico.

### E/S

I/O threads separan parte del trabajo de disco del hilo principal de QEMU. VirtIO-SCSI y multiqueue pueden mejorar paralelismo, siempre que guest, host y carga lo aprovechen.

Clon y thin no son lo mismo:

- clon: copia independiente del disco de plantilla;
- thin: asignación de espacio según se utiliza.

Una VM puede ser clon completo y usar un formato que crece dinámicamente. No mezclar independencia lógica con asignación física.

### High Performance

Presentarlo como un conjunto de decisiones exigentes: pinning, NUMA, huge pages y dispositivos más directos. Puede aportar consistencia, pero reduce destinos válidos y hace la operación más rígida.

En cargas de Oracle Database, NUMA y huge pages pueden ser importantes, pero deben seguir una arquitectura y medición reales. No se activan porque la VM «parece lenta».

## 4. Eventos, tareas y logs

### Frase de entrada

> El portal cuenta la historia desde el punto de vista de Engine. Los logs muestran cómo la vivió cada componente.

Escribir:

```text
Evento = qué observó OLVM
Tarea  = operación y estado
Log    = detalle técnico
Alerta = evento que requiere atención
```

### El recorrido mínimo

1. Evento exacto.
2. Objeto afectado.
3. Hora y severidad.
4. `engine.log`.
5. Si Engine envió la orden: `vdsm.log` del host.
6. Si VDSM llegó a libvirt/QEMU: logs correspondientes.
7. Si la VM está ejecutándose: logs del guest y de la aplicación.

Insistir en la hora. Leer logs sin acotar tiempo hace que todas las plataformas parezcan estar permanentemente averiadas.

### Mapa de memoria

| Sitio | Para qué |
|---|---|
| `/var/log/ovirt-engine/engine.log` | Decisión central |
| `/var/log/vdsm/vdsm.log` | Ejecución en el host |
| `/var/log/vdsm/libvirt.log` | Conversación VDSM-libvirt |
| `/var/log/libvirt/qemu/` | Una VM concreta |
| journal del guest | Sistema operativo de la VM |
| logs de cloud-init | Solo personalización inicial |

No hace falta memorizar todos los ficheros. Hay que memorizar el camino de control.

### Locked no significa roto

Una imagen se bloquea mientras una operación necesita exclusividad. Si vemos `Locked`, primero buscamos la tarea y su estado. Forzar un desbloqueo puede dejar dos operaciones modificando el mismo objeto.

### Log collector

`ovirt-log-collector` prepara una recogida conjunta para soporte o análisis amplio. No sustituye a pensar ni es necesario para cada evento. El archivo resultante contiene información sensible.

## 5. Troubleshooting

### Regla de oro

Una hipótesis, una prueba, un cambio.

Si reiniciamos Engine, VDSM, la red y NFS a la vez, quizá el servicio vuelva, pero no sabremos qué ocurrió ni si va a repetirse.

### Preguntas que ordenarían cualquier incidente

1. ¿Qué dejó de funcionar exactamente?
2. ¿Desde cuándo?
3. ¿A quién afecta?
4. ¿Qué cambió justo antes?
5. ¿Qué capa podría producir ese alcance?
6. ¿Qué evidencia separa mis dos primeras hipótesis?
7. ¿Cuál es la prueba menos invasiva?

### VM `Up` sin IP

Usar este caso para mostrar que estado de virtualización y estado de aplicación no son lo mismo.

Posibilidades:

- la vNIC está desconectada;
- el perfil vNIC no es correcto;
- la red no está disponible en el host;
- falla VLAN/camino físico;
- el guest no configuró DHCP o estática;
- el Guest Agent no comunica la IP al portal;
- la IP existe, pero firewall o servicio fallan.

No confundir «Engine no muestra la IP» con «la VM no tiene IP».

### NFS

En nuestro entorno el storage es NFS. Si una VM no accede a su disco, comprobar si el problema afecta a:

- un solo host;
- los dos hosts;
- un único dominio;
- todas las exportaciones.

Esa clasificación reduce rápidamente el espacio de búsqueda.

## Caso integrado para cerrar

Plantear:

> Host 2 aparece Non Responsive. Una VM crítica marcada como HA estaba allí y no se reinicia.

Pedir a los alumnos que no den soluciones todavía. Primero deben pedir datos.

Datos que deberían solicitar:

- estado real de la VM;
- conectividad con el host;
- fencing configurado;
- lease de la VM;
- capacidad del host 1;
- redes y NFS disponibles en el host 1;
- afinidad, pinning o dispositivos;
- eventos y logs alrededor de la hora.

Respuesta final:

> HA no es una casilla mágica. Es una cadena completa de detección, aislamiento, protección de datos, elegibilidad y capacidad.

## Cierre —últimos cinco minutos—

Pedir cinco respuestas rápidas:

1. Un filtro… **elimina un host**.
2. Una regla `Soft`… **puede incumplirse**.
3. HA no es… **FT**.
4. Fencing sirve para… **eliminar la ejecución incierta del host**.
5. `engine.log` explica… **decisiones del plano de control**.

Adelantar el día 5 sin desarrollarlo:

> Mañana cerraremos el curso viendo cómo se construye, se protege y se recupera la propia plataforma: instalación, backup, restore y procedimientos de recuperación.
