# Día 4 · Afinidad, alta disponibilidad, optimización y diagnóstico

---

# Qué vamos a conseguir hoy

Durante los tres primeros días hemos seguido esta construcción:

| Día | Pregunta principal |
|---|---|
| Día 1 | ¿Qué piezas forman OLVM y qué hace cada una? |
| Día 2 | ¿Cómo llegan las VMs al almacenamiento y a la red? |
| Día 3 | ¿Cómo se define, crea, personaliza y opera una VM? |

Hoy cambia la pregunta:

> **¿Qué hace OLVM cuando algo falla, cuando faltan recursos o cuando necesitamos averiguar qué ha ocurrido?**

Al terminar la jornada quiero que podáis:

1. Explicar cómo el scheduler filtra y ordena los hosts candidatos.
2. Crear y razonar reglas de afinidad VM–VM y VM–Host.
3. Diferenciar reglas positivas, negativas, preferentes y obligatorias.
4. Explicar el propósito y las limitaciones de las etiquetas de afinidad.
5. Diferenciar mantenimiento, migración, HA, fencing, watchdog y fault tolerance.
6. Explicar por qué Engine no puede reiniciar una VM en otro host sin resolver antes la posible doble ejecución.
7. Interpretar los parámetros de HA de un host y de una VM.
8. Elegir entre consolidación, flexibilidad y rendimiento sin activar opciones por intuición.
9. Relacionar CPU shares, overcommit, pinning, NUMA, memoria garantizada, ballooning, KSM y huge pages.
10. Distinguir evento, tarea, alerta y log, y seguir una incidencia hasta su causa.

La frase del día será:

> **El scheduler decide dónde puede vivir una VM; HA decide cómo recuperarla sin duplicarla.**

## Puente con vSphere

| OLVM | Referencia útil en vSphere | Matiz |
|---|---|---|
| Host en Maintenance | ESXi Maintenance Mode | Se evacúan las VMs migrables antes del mantenimiento |
| Live migration | vMotion | Es una operación planificada, no una recuperación tras fallo |
| VM altamente disponible | Reinicio mediante vSphere HA | Hay interrupción; no es Fault Tolerance |
| Power Management/Fencing | Aislamiento y control de energía del host | OLVM necesita demostrar que el host dudoso ha dejado de ejecutar |
| Afinidad/anti-afinidad | VM/Host y VM/VM rules | Una regla obligatoria puede eliminar todos los destinos válidos |
| Scheduling policy | DRS, solo como referencia conceptual | Algoritmos y controles no son idénticos |
| CPU shares | CPU shares | Pesan durante contención; no equivalen a reservar cores |
| Physical Memory Guaranteed | Reservation, aproximadamente | El modelo de cálculo no es idéntico |
| Events + logs | Tasks/Events + vCenter/ESXi logs | Hay que correlacionar plano de control, host e invitado |

---

# Reparto de la jornada

| Bloque | Duración | Contenido |
|---|---:|---|
| 1 | 45 min | Scheduler, grupos y etiquetas de afinidad |
| 2 | 55 min | Alta disponibilidad de hosts y VMs |
| Pausa | 15 min | |
| 3 | 45 min | Optimización de CPU, memoria y E/S |
| 4 | 55 min | Eventos, tareas y mapa de logs |
| Pausa | 15 min | |
| 5 | 50 min | Método de troubleshooting y prácticas guiadas |
| 6 | 20 min | Caso integrado y cierre |

Total: **5 horas**, incluidas dos pausas de 15 minutos.

---

# Reglas para el laboratorio

La instalación del aula es útil para observar conceptos, pero no reproduce el aislamiento físico de un centro de datos:

- `olvm-engine` es una VM situada en `worker2`;
- `olvm-host1` y `olvm-host2` son hosts KVM anidados sobre `worker3` y `worker4`;
- ambos hosts acceden a `curso-nfs` y `curso-nfs-2`;
- las VMs utilizan la red lógica `alumnos`;
- los hosts OLVM no disponen de un BMC físico propio configurado en Engine;
- la gestión de energía aparece deshabilitada;
- `worker4` tiene especialmente poca capacidad libre.

Por tanto:

- no simularemos un fallo arrancando dos copias de la misma VM;
- no apagaremos `worker3` ni `worker4`;
- no pondremos un host en `Non Responsive` de forma deliberada;
- no activaremos fencing con datos inventados;
- no cambiaremos overcommit, huge pages, pinning o NUMA en las VMs de los alumnos;
- no limpiaremos ni rotaremos logs durante la clase;
- las demostraciones serán de lectura o se realizarán sobre una VM prescindible.

La ausencia de una práctica destructiva no impide aprender el procedimiento. En producción, una prueba de fencing debe planificarse y comprobarse antes de confiar en HA.

---

# Bloque 1 · Scheduler, grupos y etiquetas de afinidad

Antes de hablar de HA necesitamos responder una pregunta anterior:

> **Cuando una VM debe arrancar o migrar, ¿cómo decide Engine qué host utilizar?**

El scheduler no busca simplemente «el host con menos CPU». Primero descarta los hosts que no pueden ejecutar la VM y después compara los que siguen siendo válidos.

## El proceso de selección

Podemos resumirlo en tres fases:

```text
Todos los hosts del Cluster
            ↓
Filtros: eliminan hosts imposibles
            ↓
Pesos: ordenan los hosts válidos
            ↓
Engine selecciona un destino
```

### Filtros

Un filtro responde **sí o no**. Puede eliminar un host porque:

- no está `Up`;
- no tiene CPU compatible;
- no dispone de memoria suficiente;
- carece de una red lógica requerida;
- no puede acceder al almacenamiento;
- no cumple pinning o NUMA;
- no dispone de un dispositivo de host solicitado;
- incumple una regla de afinidad obligatoria.

### Pesos

Cuando quedan varios hosts válidos, los módulos de peso permiten preferir unos frente a otros:

- menor utilización de CPU;
- mayor memoria disponible;
- distribución uniforme de VMs;
- ahorro de energía;
- cumplimiento de una afinidad preferente;
- compatibilidad más favorable con NUMA o alto rendimiento.

Una regla preferente puede empeorar la puntuación de un host, pero no necesariamente lo elimina. Una regla obligatoria sí puede dejar el conjunto de candidatos vacío.

## Política de scheduling del Cluster

El Cluster tiene una política de scheduling. Las políticas predeterminadas permiten, entre otros comportamientos:

- repartir carga de CPU y memoria;
- repartir el número de VMs;
- concentrar carga para ahorrar energía;
- limitar la actividad durante mantenimiento;
- no aplicar balanceo automático.

En una política personalizada se combinan:

| Elemento | Función |
|---|---|
| Filtros | Eliminan destinos que no cumplen una condición obligatoria |
| Pesos | Puntúan los destinos que continúan siendo válidos |
| Balanceador | Decide cuándo conviene migrar VMs para corregir desequilibrio |
| Propiedades | Ajustan umbrales e intervalos de la política |

No es necesario crear una política personalizada para usar afinidad en un curso básico. Sí debemos comprobar que la política aplicada contiene los módulos que interpretan esas reglas.

## Qué es un grupo de afinidad

Un grupo de afinidad reúne:

- una o varias VMs;
- opcionalmente uno o varios hosts;
- una regla entre las VMs;
- opcionalmente una regla entre las VMs y los hosts;
- el carácter preferente u obligatorio de cada regla.

Los grupos pertenecen al ámbito del Cluster. Si una VM cambia de Cluster, deja de pertenecer a los grupos del Cluster anterior.

## Regla de afinidad entre VMs

### Positiva

Intenta ejecutar las VMs juntas en un mismo host.

```text
VM aplicación ─┐
               ├── Host 1
VM caché ──────┘
```

Puede ser útil cuando dos VMs intercambian mucho tráfico y se desea reducir latencia. También concentra el riesgo: si cae el host, ambas se ven afectadas.

### Negativa

Intenta ejecutar las VMs en hosts diferentes.

```text
VM nodo 1 ── Host 1
VM nodo 2 ── Host 2
```

Es habitual para réplicas, nodos de cluster o servicios redundantes. Separar procesos no crea por sí solo HA: ambas VMs deben estar correctamente diseñadas y la aplicación debe conocer la redundancia.

## Regla de afinidad entre VMs y hosts

Primero se añaden al grupo los hosts relevantes. Después se decide la relación.

### Positiva

Las VMs deben o prefieren ejecutarse **en los hosts incluidos** en el grupo.

Ejemplos:

- hosts con GPU;
- hosts con una licencia determinada;
- servidores con una generación concreta de CPU;
- hosts de una ubicación o zona específica.

### Negativa

Las VMs deben o prefieren ejecutarse **fuera de los hosts incluidos**.

Ejemplos:

- evitar hosts reservados para una carga crítica;
- excluir temporalmente una generación con una incompatibilidad conocida;
- separar entornos con requisitos operativos diferentes.

## Preferente y obligatoria

En la GUI aparece la opción **Enforcing**.

| Enforcing | Tipo práctico | Si la regla no puede cumplirse |
|---:|---|---|
| No | Preferente o `Soft` | Engine puede incumplirla y arrancar la VM |
| Sí | Obligatoria o `Hard` | Engine elimina los hosts incompatibles y puede impedir el arranque |

Esta diferencia expresa una prioridad de negocio.

### Ejemplo: dos nodos redundantes

Con anti-afinidad negativa preferente, OLVM intenta separarlos, pero permite que ambos sobrevivan en un único host durante una avería.

Con anti-afinidad negativa obligatoria, si solo queda un host disponible, uno de los nodos no podrá arrancar.

La decisión es:

> ¿Es más importante mantener la separación o mantener el servicio degradado?

### Ejemplo: licencia física

Si una licencia solo permite ejecutar el software en dos hosts concretos, una afinidad VM–Host positiva y obligatoria puede representar esa restricción. Si ambos hosts están caídos, la VM no debe arrancar en otro servidor.

Aquí la indisponibilidad es el resultado esperado de una restricción consciente, no un fallo del scheduler.

## Matriz de reglas

| Relación | Positiva | Negativa |
|---|---|---|
| VM–VM | Juntar VMs | Separar VMs |
| VM–Host | Ejecutar en los hosts del grupo | Evitar los hosts del grupo |

A cualquiera de las cuatro combinaciones se le puede añadir el carácter preferente u obligatorio.

## Etiquetas de afinidad

Una etiqueta de afinidad proporciona una forma sencilla de relacionar VMs y hosts mediante un nombre común.

Ejemplo:

```text
Etiqueta: gpu

VM-render-01 ─────┐
                  ├── solo hosts etiquetados
Host-GPU-01 ──────┤
Host-GPU-02 ──────┘
```

En términos prácticos, la etiqueta se comporta como una relación VM–Host positiva y obligatoria: la VM etiquetada solo encuentra válidos hosts con esa etiqueta.

### No confundir tres tipos de etiquetas

| Objeto | Para qué sirve |
|---|---|
| Tag administrativo | Clasificar y buscar objetos; no decide dónde arranca la VM |
| Network label | Facilitar el despliegue de redes lógicas sobre NICs de host |
| Affinity label | Restringir la colocación de VMs respecto a hosts |

Una etiqueta de afinidad es cómoda, pero menos expresiva que un grupo: no ofrece por sí misma reglas negativas ni preferencia blanda.

## Dependencia de la política de scheduling

Para que las reglas participen en la selección, la política del Cluster debe contener los módulos correspondientes:

- `VmAffinityGroups` para reglas VM–VM;
- `VmToHostsAffinityGroups` para reglas VM–Host;
- `Label` para etiquetas de afinidad.

Las políticas predeterminadas suelen incorporar los módulos necesarios, pero en una política personalizada hay que comprobarlo. Definir un objeto en la GUI no sirve de nada si la política no lo evalúa.

## Conflictos y conjunto vacío

Una VM puede pertenecer a varios grupos y tener además etiquetas. Las restricciones se acumulan.

Ejemplo:

- etiqueta `gpu`: solo Host 1 y Host 2;
- regla obligatoria negativa: no ejecutar en Host 1;
- Host 2 está en mantenimiento.

Resultado:

```text
Host 1: descartado por afinidad
Host 2: descartado por mantenimiento
Host 3: descartado porque no tiene etiqueta gpu

Hosts candidatos: ninguno
```

La VM no arranca aunque el Cluster tenga CPU y memoria libres. El evento puede parecer un problema de capacidad, pero la causa real es la intersección de restricciones.

## Relación con vSphere

La comparación aproximada es:

| OLVM | vSphere |
|---|---|
| Afinidad VM–VM positiva | Keep Virtual Machines Together |
| Afinidad VM–VM negativa | Separate Virtual Machines |
| Afinidad VM–Host positiva/negativa | VM-Host rules |
| Preferente | Should run |
| Obligatoria | Must run |
| Política de scheduling | Parte del papel conceptual de DRS |

Los nombres y algoritmos no son idénticos, pero la pregunta de diseño es la misma: **preferencia que puede romperse o requisito que puede detener el servicio**.

## Casos empresariales

### Servicio activo/pasivo

Dos nodos deben estar separados normalmente. Una anti-afinidad negativa preferente permite juntarlos temporalmente si solo queda un host.

### Carga con mucho tráfico interno

Dos VMs que intercambian gran cantidad de datos pueden utilizar afinidad positiva preferente. Antes hay que valorar si concentrarlas crea un dominio de fallo excesivo.

### Licenciamiento por host físico

Afinidad VM–Host positiva obligatoria hacia los hosts cubiertos por la licencia. La documentación y las pruebas deben demostrar que no existen caminos alternativos que incumplan la restricción.

### Hardware especializado

Las etiquetas pueden seleccionar hosts con GPU, pero un dispositivo PCI passthrough también condiciona migración y disponibilidad. La etiqueta describe colocación; no crea ni valida el dispositivo.

## Práctica 1 · Diseñar afinidad sin guardar cambios

1. Abrir el Cluster `Curso` y revisar su política de scheduling.
2. Localizar las pestañas **Grupos de afinidad** y **Etiquetas de afinidad**.
3. Abrir el formulario de un grupo nuevo.
4. Identificar por separado regla VM, regla Host y opción **Enforcing**.
5. Diseñar —sin guardar— una anti-afinidad preferente entre dos VMs de prueba.
6. Diseñar —sin guardar— una afinidad obligatoria hacia los dos hosts del aula.
7. Explicar qué ocurriría si uno de los hosts entra en mantenimiento.
8. Cancelar.

### Resultado de la práctica

El alumno debe poder completar:

> «Esta regla es ________ y ________. Si no puede cumplirse, Engine ________.»

---

# Bloque 2 · Alta disponibilidad de hosts y VMs

## Empecemos por separar situaciones

| Situación | ¿El host responde? | ¿La VM sigue ejecutándose? | Respuesta habitual |
|---|---:|---:|---|
| Mantenimiento planificado | Sí | Sí al principio | Migrar o apagar ordenadamente |
| VM apagada por el usuario | Sí | No | Mantenerla `Down` |
| Proceso QEMU termina | Sí | No | VDSM informa; aplicar política correspondiente |
| Invitado bloqueado, QEMU vivo | Sí | Sí, pero no presta servicio | Watchdog o monitorización de aplicación |
| Host `Non Operational` | Sí | Puede | Corregir configuración incompatible |
| Host `Non Responsive` | No desde Engine | Desconocido | Aislar/fencear antes de recuperar |
| Engine caído | Los hosts pueden seguir | Las VMs encendidas continúan | Recuperar Engine; no hay nuevas decisiones centrales |
| Storage NFS perdido | Puede | Puede quedar pausada o fallar E/S | Recuperar storage antes de improvisar sobre la VM |

La palabra más importante de la tabla es **desconocido**. Si Engine pierde comunicación con un host, no puede concluir inmediatamente que todas sus VMs han dejado de ejecutarse.

## HA no es fault tolerance

Una VM altamente disponible puede reiniciarse en otro host después de un fallo. El sistema operativo y las aplicaciones arrancan de nuevo.

Esto implica:

- interrupción del servicio;
- pérdida del contenido de RAM;
- posible recuperación de filesystem o aplicación;
- necesidad de que el servicio soporte el reinicio;
- tiempo de detección, aislamiento, selección y arranque.

Fault tolerance implicaría mantener otra ejecución sincronizada capaz de continuar casi sin interrupción. No es lo que conseguimos marcando **Altamente disponible** en OLVM.

## Migración planificada frente a recuperación HA

### Live migration

El host origen está vivo y coopera:

1. Engine selecciona un destino.
2. VDSM y QEMU preparan la migración.
3. Se copian memoria y estado de ejecución.
4. Se realiza una parada final breve.
5. Continúa la misma VM en el destino.

### Recuperación después de un fallo

El host origen no coopera:

1. Engine detecta pérdida de comunicación.
2. Debe evitar una doble ejecución.
3. Se aísla el host o se valida el mecanismo de lease aplicable.
4. Se elige otro host elegible.
5. La VM arranca desde cero utilizando sus discos compartidos.

La recuperación HA no mueve la memoria que estaba en el host muerto.

## Estado `Non Operational` y estado `Non Responsive`

No son equivalentes.

### `Non Operational`

El host todavía habla con Engine, pero existe una incompatibilidad que impide utilizarlo normalmente. Ejemplos:

- falta una Logical Network requerida;
- una red está fuera de sincronía;
- el host no puede acceder a un Storage Domain necesario;
- existe una incompatibilidad de Cluster o CPU;
- una configuración requerida no se ha materializado.

### `Non Responsive`

Engine no consigue comunicarse con VDSM en el host. Las posibles causas incluyen:

- host apagado;
- fallo de red de gestión;
- VDSM bloqueado o detenido;
- firewall;
- sobrecarga extrema;
- problema del sistema operativo;
- partición de red: Engine no llega al host, pero el host puede seguir ejecutando VMs.

Solo el segundo caso plantea directamente la incertidumbre que exige aislamiento.

## Qué es fencing

Fencing es el mecanismo para apartar de forma verificable un host dudoso. En la práctica suele actuar sobre su dispositivo de gestión de energía:

- IPMI;
- iLO;
- iDRAC;
- controladora de chasis;
- PDU o agente de fencing soportado.

Las acciones habituales son consultar estado, apagar, encender o reiniciar.

> **Fencing no intenta arreglar la VM; garantiza que el host anterior ya no puede seguir ejecutándola.**

## Por qué Engine utiliza un proxy de fencing

Engine decide que debe realizarse la operación, pero el agente de fencing se ejecuta a través de otro host apto como proxy.

El proxy debe estar normalmente:

- en el mismo Cluster o, como alternativa, en el mismo Data Center;
- en estado `Up` o `Maintenance`;
- conectado al dispositivo de gestión del host que se quiere aislar.

Se necesitan al menos dos hosts para que otro host pueda actuar como proxy.

Esto evita convertir a Engine en un equipo que deba tener conectividad directa con cada red de BMC.

## Configuración de Power Management en la GUI

Ruta general:

1. **Cómputo → Hosts**.
2. Seleccionar el host.
3. **Modificar → Gestión de energía**.
4. Activar **Habilitar gestión de energía**.
5. Añadir el fence agent.
6. Indicar dirección, usuario, contraseña, tipo, puerto, slot y opciones.
7. Pulsar **Test**.
8. Comprobar que Engine recibe el estado real del host.

Parámetros relevantes:

| Parámetro | Para qué sirve |
|---|---|
| Kdump integration | Evita fencear el host mientras genera un crash dump del kernel |
| Fence agent | Implementa la comunicación con el BMC o dispositivo de energía |
| Concurrent agents | Permite operar agentes a la vez en configuraciones redundantes |
| Sequential order | Define el orden de intentos |
| Proxy preference | Define dónde busca Engine un host que ejecute el agente |
| Secure | Utiliza el mecanismo seguro admitido por el dispositivo |

Una prueba exitosa hoy no garantiza que las credenciales sigan siendo válidas dentro de seis meses. El cambio de contraseña del BMC debe reflejarse en OLVM y volver a probarse.

## Qué significa el aviso de nuestra instalación

El mensaje:

> La gestión de energía no está habilitada para este host.

es correcto en el aula. `olvm-host1` y `olvm-host2` son VMs anidadas, no servidores físicos con iLO, iDRAC o IPMI propios registrados en Engine.

Consecuencias:

- podemos hacer live migration mientras ambos hosts cooperan;
- podemos estudiar la configuración de una VM HA;
- no podemos afirmar que la recuperación automática ante un host incomunicado sea segura;
- marcar una VM como HA no crea un BMC ni añade fencing;
- no debemos presentar el laboratorio como una validación de HA de producción.

## Parámetros de alta disponibilidad de una VM

En **Modificar VM → Alta disponibilidad** aparecen varios controles.

### Altamente disponible

Indica a Engine que la VM es candidata a recuperación automática según las condiciones y políticas disponibles.

No significa:

- que exista un host alternativo;
- que haya memoria suficiente;
- que el storage esté disponible;
- que las redes estén presentes;
- que el host esté correctamente fenceado;
- que la aplicación vaya a quedar consistente.

### Prioridad

Puede ser baja, media o alta. Se utiliza para ordenar VMs cuando varias deben migrarse o recuperarse y los recursos son limitados.

Una prioridad alta no crea capacidad. Si ningún host puede ejecutar la VM, continuará sin existir un destino válido.

### Acción al reanudar

Las opciones describen qué hacer con una VM que debe reanudarse después de una situación de pausa:

- reanudar automáticamente;
- dejarla pausada;
- finalizarla.

Cuando se configura VM lease, la opción compatible es `KILL`, porque el diseño prioriza impedir ejecuciones ambiguas.

### Destino del VM lease

Selecciona el Storage Domain que guardará el lease de esa VM.

El lease:

- es por VM;
- reside en un volumen especial del almacenamiento compartido;
- debe renovarse mientras la ejecución conserva el derecho sobre él;
- ayuda a impedir el arranque simultáneo en otro host;
- no es el disco virtual;
- no es la metadata del SPM;
- no es una reserva de capacidad;
- no sustituye automáticamente todos los casos de fencing.

La implementación utiliza el stack de locking del host, con componentes como `sanlock`. No debemos explicarlo como «Engine entrega un candado y lo libera cuando decide que la VM murió». En una partición real, Engine puede no saber si el host sigue vivo; precisamente por eso importa la expiración comprobable del lease.

## Watchdog: dónde encaja y dónde no

El watchdog responde a otra pregunta:

> ¿Qué ocurre si el proceso QEMU sigue vivo, pero el sistema operativo invitado está bloqueado?

El flujo correcto es:

1. OLVM presenta un dispositivo watchdog virtual, habitualmente `i6300esb`.
2. El kernel invitado expone `/dev/watchdog`.
3. El daemon `watchdog` alimenta el temporizador.
4. Si deja de hacerlo, el dispositivo expira.
5. QEMU ejecuta la acción configurada, por ejemplo reset o power off.
6. VDSM y Engine observan el nuevo estado resultante.

El daemon no manda latidos periódicos a VDSM. El circuito principal ocurre entre daemon, driver, dispositivo virtual y QEMU.

## Requisitos completos de una recuperación

Antes de confiar en HA debemos comprobar:

| Área | Pregunta |
|---|---|
| Cómputo | ¿Existe otro host activo y compatible? |
| Capacidad | ¿Tiene CPU y memoria para la VM? |
| Storage | ¿Accede al Data Domain que contiene los discos? |
| Red | ¿Tiene todas las Logical Networks y VLAN requeridas? |
| Aislamiento | ¿Puede confirmarse que el host anterior no ejecuta? |
| Políticas | ¿Afinidad, pinning o dispositivos permiten otro destino? |
| Aplicación | ¿Tolera un reinicio y recupera su estado? |
| Operación | ¿Se han probado alertas, runbook y recuperación? |

## Capacidad N+1

Dos hosts no bastan si cada uno está lleno.

Si queremos soportar la pérdida de un host, el resto del Cluster debe disponer de capacidad para absorber las VMs críticas. Esto se conoce habitualmente como diseño N+1.

Ejemplo:

| Host | Capacidad útil | Carga normal |
|---|---:|---:|
| Host A | 100 unidades | 80 |
| Host B | 100 unidades | 80 |

Al caer Host A, Host B solo dispone de 20 unidades. Tener dos hosts no proporciona HA para 80 unidades de carga.

## La afinidad vuelve a evaluarse durante HA

Una VM recuperada no queda exenta de las reglas del scheduler. Las reglas obligatorias siguen eliminando destinos durante un fallo. Por eso una anti-afinidad estricta puede impedir un reinicio aunque quede un host con CPU y memoria.

Antes de marcar una regla como **Enforcing**, debemos probar también los escenarios degradados, no solo el funcionamiento normal.

## Práctica 2 · Auditoría de HA sin provocar un fallo

### En los hosts

1. Abrir `olvm-host1` y `olvm-host2`.
2. Comprobar el aviso de gestión de energía.
3. Abrir **Modificar → Gestión de energía** sin guardar cambios.
4. Identificar campos de agente, proxy, kdump y prueba.
5. Cerrar con **Cancelar**.

### En una VM de demostración

1. Abrir **Modificar → Alta disponibilidad**.
2. Anotar si está marcada como altamente disponible.
3. Revisar prioridad, acción al reanudar y destino de lease.
4. Revisar la pestaña Host y las reglas de afinidad.
5. Comprobar sus Storage Domains y Logical Networks.
6. Responder si existe realmente otro destino válido.
7. Cancelar sin modificar.

### Resultado esperado

El alumno debe poder completar esta frase:

> «La VM está configurada como ________, pero el entorno no ofrece HA completa porque ________.»

---

# Bloque 3 · Optimización de CPU, memoria y E/S

## Optimizar no significa activarlo todo

Hay tres objetivos que compiten:

| Objetivo | Qué favorece | Qué puede sacrificar |
|---|---|---|
| Consolidación | Ejecutar más VMs por host | Rendimiento predecible |
| Flexibilidad | Migrar y recolocar VMs fácilmente | Afinidad con hardware concreto |
| Rendimiento | Reducir latencia y variabilidad | Densidad y movilidad |

Una opción de alto rendimiento puede reducir la cantidad de hosts elegibles. Una opción de overcommit puede aumentar densidad, pero introduce contención.

## CPU virtual y CPU física

Sin pinning, una vCPU es una entidad planificable. El kernel del host decide en qué CPU lógica ejecutarla.

Esto permite:

- compartir CPUs físicas;
- mover hilos entre CPUs;
- consolidar cargas;
- migrar la VM con mayor facilidad.

Pero también introduce:

- espera si hay contención;
- variación de latencia;
- pérdida de localidad de caché;
- dificultad para garantizar tiempos estrictos.

## CPU shares

Los shares expresan peso relativo cuando varias VMs compiten por CPU:

| Nivel OLVM | Valor de referencia |
|---|---:|
| Bajo | 512 |
| Medio | 1024 |
| Alto | 2048 |

No reservan un core y no aceleran una VM cuando no existe contención. Sirven para decidir quién recibe más tiempo de CPU cuando dos o más cargas lo necesitan simultáneamente.

## Overcommit de CPU

Podemos definir más vCPU totales que CPUs lógicas físicas porque no todas las VMs consumen el 100 % al mismo tiempo.

El overcommit funciona bien con cargas:

- intermitentes;
- poco intensivas;
- con picos no coincidentes;
- tolerantes a latencia variable.

Es arriesgado con:

- bases de datos críticas;
- baja latencia;
- cargas sostenidas al 100 %;
- licenciamiento ligado a topología;
- aplicaciones sensibles a jitter.

## Contar threads como cores

La opción de Cluster **Count Threads As Cores** permite al planificador considerar hilos SMT como capacidad para vCPU.

Un hilo SMT no equivale a un core físico completo. Dos threads del mismo core comparten recursos internos. Activarlo mejora consolidación, pero no duplica mágicamente el rendimiento.

## Topología de CPU

La VM presenta:

```text
vCPU = sockets × cores por socket × threads por core
```

La topología puede influir en:

- licenciamiento;
- comportamiento del scheduler invitado;
- NUMA virtual;
- límites del sistema operativo;
- compatibilidad y migración.

No debemos escoger muchos sockets solo porque el total de vCPU sea correcto.

## CPU pinning

Pinning vincula vCPU concretas a CPUs físicas concretas.

Ventajas posibles:

- latencia más predecible;
- mejor localidad de caché;
- aislamiento de cargas críticas;
- alineación con NUMA.

Costes:

- menos hosts válidos;
- migración restringida;
- riesgo de concentrar varias vCPU en la misma CPU física;
- necesidad de coordinarlo con CPUs del emulador y threads de E/S;
- administración más compleja.

Pinning mal diseñado puede rendir peor que el scheduler normal.

## NUMA

En servidores grandes, no toda la RAM está igual de cerca de todas las CPUs. Cada nodo NUMA agrupa CPUs y memoria local.

El objetivo es mantener:

- vCPU cerca de la CPU física elegida;
- memoria de la VM en el nodo correspondiente;
- dispositivos y E/S con la mejor localidad posible.

Una VM que cruza nodos NUMA puede sufrir mayor latencia. Sin embargo, fijar NUMA reduce la libertad del planificador y los destinos de migración.

En nuestro laboratorio anidado no utilizaremos NUMA para demostrar rendimiento: mediríamos varias capas de virtualización, no un servidor físico real.

## Memoria definida, garantizada y máxima

| Valor | Significado |
|---|---|
| Definida | RAM que ve la VM al arrancar |
| Garantizada | Compromiso mínimo utilizado por la plataforma |
| Máxima | Techo de crecimiento configurado |

La memoria máxima no se reserva al arrancar. La memoria garantizada tampoco es swap.

## Overcommit de memoria en el Cluster

Oracle documenta perfiles de planificación:

- **None**: 100 % de la memoria física;
- **Server Load**: permite planificar hasta el 150 %;
- **Desktop Load**: permite planificar hasta el 200 %.

Estos porcentajes no fabrican RAM. Solo permiten que el planificador admita más memoria definida confiando en que no todas las VMs la necesitarán simultáneamente y en mecanismos de recuperación de memoria.

## Ballooning y MoM

Para que OLVM pueda utilizar ballooning deben coincidir dos niveles:

1. El Cluster tiene habilitada la optimización de memory balloon.
2. La VM dispone del dispositivo y del driver `virtio_balloon`.

MoM —Memory Overcommitment Manager— decide cuándo aplicar la política. El límite inferior práctico es la memoria garantizada.

En las VMs del aula:

```text
memoria definida   = 1024 MiB
memoria garantizada = 1024 MiB
```

El dispositivo puede existir, pero no hay margen para recuperar memoria por debajo de 1024 MiB.

## KSM

Kernel Same-page Merging busca páginas de memoria idénticas entre procesos y las comparte mediante copy-on-write.

Puede ahorrar memoria cuando muchas VMs contienen páginas iguales, pero:

- consume CPU al buscar coincidencias;
- su beneficio depende de la carga;
- añade complejidad y posibles consideraciones de seguridad;
- no debe presentarse como memoria gratuita.

MoM puede activar KSM cuando calcula que el ahorro compensa el coste.

## Huge pages

Huge pages utilizan páginas mayores para reducir tablas de páginas y presión sobre la TLB.

Pueden beneficiar cargas grandes y predecibles, pero exigen:

- reservar páginas del tamaño correcto en el host;
- que la memoria de la VM encaje en esas páginas;
- considerar NUMA;
- mantener hosts compatibles;
- aceptar que no puede utilizarse hot plug/hot unplug de memoria para esa VM.

No activaremos huge pages en el aula. En un host anidado y con poca RAM, el coste de una mala reserva supera el valor pedagógico de la prueba.

## Aplicaciones Oracle y memoria dedicada

La documentación de Oracle advierte que, para productos Oracle que requieren memoria dedicada —por ejemplo determinadas cargas de Oracle Database—, no deben utilizarse opciones de overcommit, ballooning o KSM como si fueran transparentes.

La política debe derivarse de requisitos y soporte de la aplicación, no solo de la capacidad técnica de OLVM.

## I/O threads

Separan el procesamiento de E/S VirtIO del hilo principal de emulación de la VM.

Pueden mejorar paralelismo y reducir interferencias cuando la VM genera E/S significativa. También consumen threads del host y deben contemplarse en pinning y capacidad.

## VirtIO-SCSI y multiqueue

- VirtIO-SCSI proporciona un controlador paravirtualizado para discos SCSI virtuales.
- Multiqueue permite varias colas para aprovechar múltiples vCPU en red o almacenamiento.
- Más colas no siempre mejoran una VM pequeña.
- Cada cola añade procesamiento y necesita carga paralela para aportar valor.

## Perfil High Performance

Seleccionar **Optimizado para → Alto rendimiento** cambia varios parámetros y presenta recomendaciones manuales.

El beneficio puede venir acompañado de:

- dispositivos deshabilitados;
- pinning;
- huge pages;
- menor número de destinos válidos;
- restricciones de migración;
- más trabajo de operación.

No debe activarse para resolver una aplicación lenta sin identificar antes CPU, memoria, storage, red y aplicación.

## Práctica 3 · Auditoría de optimización

Sin guardar cambios:

1. Abrir el Cluster `Curso` y revisar **Optimización**.
2. Anotar el perfil de overcommit de memoria.
3. Comprobar si se cuentan threads como cores.
4. Comprobar ballooning y KSM.
5. Abrir una VM del alumno y revisar **Asignación de recursos**.
6. Anotar CPU shares, pinning, balloon, I/O threads, multiqueue y VirtIO-SCSI.
7. Revisar **Host**, afinidad y dispositivos de host.
8. Decidir qué opciones reducirían la posibilidad de migrar.
9. Cancelar.

### Pregunta final de la práctica

> ¿La configuración busca consolidación, flexibilidad o máximo rendimiento? ¿Qué evidencia lo demuestra?

---

# Bloque 4 · Eventos, tareas y logs

Hasta ahora hemos hablado de objetos: hosts, máquinas virtuales, redes y dominios de almacenamiento. Cuando algo falla, esos objetos dejan pistas en varios sitios. La dificultad no suele ser que no haya información, sino saber **qué información mirar primero y cómo relacionarla**.

## Cuatro conceptos que no son lo mismo

| Concepto | Qué representa | Ejemplo |
|---|---|---|
| Evento | Un hecho que OLVM ha registrado | «La VM se ha iniciado» |
| Alerta | Un evento que requiere atención | «El dominio de almacenamiento tiene poco espacio» |
| Tarea | Una operación con duración y estado | Crear un disco, moverlo o importar una plantilla |
| Log | El relato técnico producido por un componente | Una excepción en Engine o una orden recibida por VDSM |

Un evento indica qué observó OLVM, pero no siempre contiene la causa raíz. El diagnóstico aparece al correlacionar **evento, tiempo, objeto y logs**.

## Primera regla: empezar en el portal

Antes de entrar por SSH en todos los servidores:

1. Leer el mensaje completo del evento.
2. Identificar el objeto afectado: VM, host, storage, red o cluster.
3. Anotar la hora, la severidad y, si aparece, el identificador de correlación.
4. Comprobar si existen eventos anteriores relacionados.
5. Abrir el objeto y revisar su estado y sus subpestañas.

El portal ofrece el contexto global que puede faltar en un log aislado.

### Severidades habituales

- **Normal o Info:** actividad esperada.
- **Warning:** situación anómala que puede no haber impedido la operación.
- **Error:** una operación ha fallado o un componente no puede cumplirla.
- **Alert:** condición que requiere atención operativa.

No debemos buscar solo mensajes que contengan `error`. Una advertencia anterior puede explicar el error final.

## El reloj también forma parte del diagnóstico

Engine, hosts, almacenamiento y sistemas invitados deben compartir una referencia horaria coherente. Si los relojes difieren, un mismo incidente parecerá ocurrir en un orden falso.

En cada sistema podemos comenzar por:

```bash
date --iso-8601=seconds
timedatectl
```

Después delimitamos una ventana pequeña alrededor del incidente. Es mejor estudiar cinco minutos precisos que leer un fichero completo sin criterio.

## Mapa de logs por componente

| Pregunta | Lugar inicial | Información esperable |
|---|---|---|
| ¿Qué decidió el plano de control? | `/var/log/ovirt-engine/engine.log` | Decisiones de Engine, validaciones y errores de operaciones |
| ¿Está funcionando Engine? | `journalctl -u ovirt-engine` | Arranque, parada y fallos del servicio |
| ¿Qué recibió o ejecutó el host? | `/var/log/vdsm/vdsm.log` | Órdenes de Engine, storage, red y ciclo de vida de VMs |
| ¿Cómo habló VDSM con libvirt? | `/var/log/vdsm/libvirt.log` | Llamadas y respuestas entre VDSM y libvirt |
| ¿Qué ocurrió en una VM concreta? | journal de libvirt y `/var/log/libvirt/qemu/` | Arranque de QEMU, dispositivos y errores concretos del dominio |
| ¿Qué ocurrió en el invitado? | `journalctl` dentro de la VM | Sistema operativo, red, servicios y aplicaciones del guest |
| ¿Falló la personalización inicial? | `/var/log/cloud-init.log` y `/var/log/cloud-init-output.log` del guest | Ejecución de cloud-init |
| ¿Está accesible NFS? | `findmnt`, `nfsstat -m` y journal del host | Montajes, opciones NFS y errores del cliente |

Los nombres exactos de las unidades de libvirt pueden variar según la versión y su arquitectura modular. Antes de asumir `libvirtd`, podemos comprobar:

```bash
systemctl list-units --type=service | grep -E 'virt|libvirt|vdsm'
```

## Lectura segura de logs

Comandos útiles que no modifican el sistema:

```bash
systemctl --failed
journalctl -u ovirt-engine --since "10 minutes ago"
journalctl -u vdsmd --since "10 minutes ago"
journalctl --since "2026-08-27 10:20:00" --until "2026-08-27 10:25:00"
tail -n 100 /var/log/ovirt-engine/engine.log
tail -n 100 /var/log/vdsm/vdsm.log
grep -iE 'error|warning|failed|timeout' /var/log/vdsm/vdsm.log | tail -n 50
```

Conviene buscar también el nombre y el UUID del objeto. Los nombres son cómodos para las personas; los identificadores reducen ambigüedades.

## Una operación, varios relatos

Cuando solicitamos arrancar una VM:

1. El portal envía la petición a Engine.
2. Engine valida cluster, host, memoria, storage, red y políticas.
3. Engine selecciona un host y envía una orden a VDSM.
4. VDSM prepara los recursos y solicita a libvirt la creación del dominio.
5. libvirt lanza QEMU/KVM.
6. Engine recibe estados y los muestra en el portal.

Si Engine rechaza la operación antes de enviarla al host, el detalle estará principalmente en `engine.log`. Si el host recibió la orden pero no pudo preparar un disco o dispositivo, `vdsm.log` y los logs de libvirt serán más útiles.

## Tareas bloqueadas e imágenes bloqueadas

Las operaciones sobre discos pueden tardar: copiar, mover, crear una plantilla o extender un disco. OLVM protege esos objetos para que no se modifiquen simultáneamente. Durante ese tiempo podemos ver un disco o imagen con estado **Locked**.

`Locked` no significa automáticamente corrupción. Primero debemos averiguar:

- qué tarea lo bloqueó;
- si la tarea sigue activa;
- cuándo comenzó;
- si el host y el dominio de almacenamiento siguen disponibles;
- qué indican los eventos y los logs.

No se debe desbloquear una imagen manualmente mientras la tarea real siga trabajando. Forzar el cambio de estado sin conocer la operación puede permitir acciones incompatibles sobre el mismo disco.

## Recogida conjunta con `ovirt-log-collector`

Cuando un incidente afecta a varios componentes, OLVM dispone de una herramienta que recopila logs de Engine, hosts y base de datos en un único archivo para su análisis o para soporte.

En Engine:

```bash
sudo dnf install ovirt-log-collector
sudo ovirt-log-collector
```

El resultado se deja normalmente bajo `/tmp/logcollector`. La herramienta admite filtros y la opción `--no-postgresql` cuando no se desea recoger información de PostgreSQL.

No es el primer paso para cada aviso. Se utiliza cuando necesitamos una fotografía coherente del entorno o debemos entregar evidencias a soporte. Antes de compartir el archivo hay que tratarlo como información sensible: puede contener nombres, direcciones, configuraciones y fragmentos de logs internos.

## Práctica 4 · Seguir una operación sin provocar un fallo

1. Elegir una VM de laboratorio.
2. Anotar su nombre, host actual, estado y hora.
3. Abrir **Eventos** y observar sus eventos recientes.
4. Realizar únicamente una operación autorizada, por ejemplo apagar y arrancar una VM de prueba.
5. Volver a anotar la hora.
6. En Engine, localizar la operación en `engine.log`.
7. En el host de ejecución, localizarla en `vdsm.log`.
8. Comprobar que los mensajes cuentan partes distintas de la misma historia.

La práctica no busca leer cada línea. Busca contestar:

> ¿Quién pidió la operación, quién la decidió, quién la ejecutó y dónde quedó el resultado?

---

# Bloque 5 · Método de troubleshooting

Diagnosticar no consiste en cambiar cosas hasta que desaparece el síntoma. Consiste en reducir posibilidades con evidencias y conservar la capacidad de volver atrás.

## Método de siete pasos

### 1. Proteger el servicio y los datos

Antes de experimentar, determinar si la siguiente acción puede:

- apagar VMs;
- interrumpir storage o red;
- provocar migraciones;
- eliminar evidencias;
- aumentar el alcance del incidente.

### 2. Definir el síntoma con precisión

«OLVM no funciona» no es un síntoma operativo. Son descripciones mejores:

- «La VM `vm-alumno04` no arranca desde las 10:22».
- «El host aparece Non Responsive, pero las VMs siguen respondiendo».
- «Solo las VMs del host 2 han perdido acceso a la red alumnos».

### 3. Fijar la línea temporal

Anotar cuándo comenzó, qué cambio hubo antes y cuál fue el último estado correcto conocido.

### 4. Delimitar el alcance

¿Afecta a una VM, a todas las VMs de un host, a un cluster, a una red, a un dominio de almacenamiento o a todo Engine?

### 5. Elegir la capa

Podemos recorrer de arriba abajo:

1. Portal y API.
2. Engine.
3. VDSM y libvirt.
4. Host: CPU, memoria, red y mounts.
5. Storage compartido.
6. Sistema operativo invitado.
7. Aplicación.

### 6. Formular una hipótesis comprobable

Ejemplo: «La VM no arranca porque el host elegido no puede montar el dominio NFS». La prueba debe distinguir esa hipótesis de otras: comprobar el estado del dominio, el mount y el error exacto de VDSM.

### 7. Aplicar el cambio mínimo y verificar

Un cambio cada vez, con resultado esperado, comprobación y forma de volver atrás. Finalmente se documentan causa, impacto, acción y prevención.

## Escalera de evidencias

| Nivel | Pregunta |
|---|---|
| Portal | ¿Qué estado global y qué evento ve Engine? |
| Engine | ¿Qué decisión tomó y por qué? |
| Host/VDSM | ¿Recibió la orden y pudo preparar los recursos? |
| libvirt/QEMU | ¿Se pudo crear o mantener el dominio virtual? |
| Red/storage | ¿Existen los caminos de datos necesarios? |
| Guest | ¿El sistema operativo arrancó y configuró sus servicios? |

No saltamos directamente al guest si la VM ni siquiera tiene un proceso QEMU. Tampoco culpamos a Engine si la VM está `Up` y el único fallo es un servicio dentro del invitado.

## Caso A · La VM no arranca

Orden de comprobación:

1. Evento exacto y hora.
2. Estado de la VM, su disco y su cluster.
3. Hosts disponibles y memoria necesaria.
4. Afinidad, pinning, passthrough o restricciones de CPU.
5. Estado del Storage Domain y del disco.
6. Red lógica requerida en el host candidato.
7. `engine.log` alrededor de la hora.
8. Si hubo orden al host, `vdsm.log` y libvirt.

Una VM puede no arrancar aunque el cluster muestre RAM libre: la memoria garantizada, la topología NUMA, el pinning, una regla de afinidad o un dispositivo de host pueden eliminar todos los destinos válidos.

## Caso B · La VM está `Up`, pero no tiene IP

`Up` demuestra que el proceso virtual se ejecuta; no demuestra que el sistema invitado haya terminado de arrancar ni que su red esté configurada.

Comprobamos:

1. vNIC conectada y perfil vNIC correcto.
2. Red lógica presente y sincronizada en el host.
3. TAP/vnet unido al bridge o datapath esperado.
4. VLAN y camino físico, si se utilizan.
5. DHCP o configuración estática dentro del guest.
6. Guest Agent, solo para saber por qué Engine no muestra la IP.
7. Firewall, ruta y servicio que realmente queremos alcanzar.

No ver una IP en el portal puede significar simplemente que el Guest Agent no la ha comunicado. Hay que distinguir **IP no mostrada** de **IP no configurada**.

## Caso C · La migración está bloqueada

Posibles causas:

- no existe otro host válido;
- falta memoria o CPU compatible;
- una red requerida no está operativa en el destino;
- el storage no es accesible desde ambos hosts;
- existe pinning, NUMA estricto o passthrough;
- una regla de afinidad lo impide;
- la VM o un dispositivo se ha marcado como no migrable;
- hay una operación de disco en curso.

La pregunta útil no es «¿por qué OLVM no migra?», sino «¿qué condición de elegibilidad no cumple cada host candidato?».

## Caso D · Un dominio NFS presenta problemas

En nuestro laboratorio el almacenamiento de datos es NFS. Hay que separar:

- servidor NFS;
- red entre cada host y el servidor;
- exportación y permisos;
- mount y opciones NFS;
- estado que conoce VDSM;
- metadatos del Storage Domain;
- carga o latencia.

Comprobaciones de solo lectura en un host:

```bash
findmnt -t nfs,nfs4
nfsstat -m
mount | grep -E ' nfs| nfs4'
journalctl --since "10 minutes ago" | grep -iE 'nfs|timeout|stale|mount'
```

Si falla solo un host, el servidor NFS puede estar sano y el problema estar en la red, el mount o las credenciales de ese host. Si fallan todos a la vez, aumenta la probabilidad de un problema común, pero sigue siendo una hipótesis.

## Caso E · Host `Non Responsive`

Antes de reiniciar:

1. Comprobar si responde por la red de gestión.
2. Determinar si el sistema operativo está vivo.
3. Comprobar si VDSM responde y si el host está saturado.
4. Verificar que Engine puede resolver y alcanzar al host.
5. Evaluar el estado real de las VMs.
6. Comprobar fencing y capacidad N+1 antes de esperar recuperación HA.

Reiniciar un host sin confirmar dónde se ejecutan sus VMs puede agravar una situación ambigua. Precisamente para eliminar esa ambigüedad existe el fencing.

## Práctica 5 · Diagnóstico en papel

Por grupos, elegir uno de estos síntomas:

- VM que no arranca;
- VM `Up` sin conectividad;
- migración rechazada;
- Storage Domain inactivo en un host;
- host `Non Responsive`.

Preparar una hoja con:

1. impacto y alcance;
2. tres hipótesis ordenadas;
3. una evidencia que confirmaría o descartaría cada una;
4. logs y comandos de solo lectura;
5. primera acción segura;
6. condición de escalado.

---

# Bloque 6 · Caso integrado

### Escenario

El host 2 deja de comunicar con Engine. En el portal aparece `Non Responsive`. Una VM marcada como altamente disponible estaba ejecutándose allí. No se reinicia inmediatamente en el host 1.

### Preguntas antes de actuar

1. ¿Sigue funcionando realmente la VM en el host 2?
2. ¿Está caído el host o solo la red/servicio de gestión?
3. ¿Existe fencing y puede confirmar el apagado?
4. ¿Tiene la VM un lease y qué comportamiento se configuró?
5. ¿Puede el host 1 acceder a sus discos y redes?
6. ¿Dispone el host 1 de capacidad suficiente?
7. ¿Hay afinidad, pinning o dispositivos que impidan moverla?

### Razonamiento esperado

OLVM no debe arrancar una segunda copia mientras exista la posibilidad razonable de que la primera continúe escribiendo. El fencing elimina esa ambigüedad. Después, HA necesita además un destino válido, acceso al storage, redes correctas y capacidad.

En nuestro laboratorio no hay gestión de energía real. Por ello podemos explicar y observar parte del proceso, pero no debemos prometer un failover automático equivalente al de una plataforma física con BMC, fencing probado y capacidad N+1.

### Evidencias que recogeríamos

- hora y eventos del host y de la VM;
- conectividad de gestión;
- estado de Engine y VDSM;
- estado real del proceso QEMU;
- configuración de fencing, lease y prioridad HA;
- hosts candidatos y causa de descarte;
- acceso a NFS y redes desde el host 1;
- logs de Engine y VDSM en la misma ventana temporal;
- eventos y cambios anteriores al incidente.

## Resumen para la pizarra

| Pregunta | Respuesta corta |
|---|---|
| ¿Qué hace HA? | Reinicia una VM en otro host tras confirmar el fallo y encontrar destino válido |
| ¿Qué evita el fencing? | Que un host de estado incierto continúe ejecutando la VM |
| ¿Qué añade el lease? | Un bloqueo renovable en storage frente a doble ejecución |
| ¿Qué hace el watchdog? | Detecta falta de respuesta dentro del guest y ejecuta una acción configurada |
| ¿Qué hace una regla obligatoria? | Elimina los hosts que no la cumplen |
| ¿Qué hace una regla preferente? | Cambia la prioridad sin impedir necesariamente el arranque |
| ¿Qué controla el overcommit? | Cuántos recursos virtuales prometemos respecto a los físicos |
| ¿Dónde decide OLVM? | En Engine |
| ¿Dónde se ejecuta? | En VDSM, libvirt y QEMU/KVM del host |
| ¿Dónde empezamos un incidente? | Evento, hora, objeto y alcance |

## Distribución sugerida para el instructor

| Tiempo | Actividad |
|---:|---|
| 00:00–00:10 | Recapitulación del día 3 y objetivos |
| 00:10–00:35 | Cómo selecciona un host el scheduler |
| 00:35–00:55 | Grupos, reglas y etiquetas de afinidad |
| 00:55–01:10 | Práctica 1: diseño de afinidad |
| 01:10–01:50 | HA, fencing, lease, watchdog y capacidad N+1 |
| 01:50–02:05 | Práctica 2: auditoría HA |
| 02:05–02:20 | Descanso |
| 02:20–03:05 | Optimización de CPU, memoria y E/S |
| 03:05–03:30 | Eventos, tareas y mapa de logs |
| 03:30–04:00 | Práctica 4: seguir una operación |
| 04:00–04:15 | Descanso |
| 04:15–04:45 | Método de troubleshooting |
| 04:45–04:55 | Caso integrado |
| 04:55–05:00 | Cierre y preguntas rápidas |

## Preguntas orales de cierre

1. ¿Cuándo una afinidad positiva significa juntar VMs y cuándo significa elegir hosts?
2. ¿Qué diferencia operativa existe entre una regla `Soft` y una regla `Hard`?
3. ¿Por qué una VM marcada como HA puede no reiniciarse en otro host?
4. ¿Dónde buscarías si Engine aceptó un arranque pero el host no pudo crear la VM?
5. ¿Qué sacrifica una VM con pinning y passthrough a cambio de rendimiento o acceso directo?

## Al finalizar el día

El alumno debe ser capaz de:

- explicar el recorrido del scheduler desde filtros hasta pesos;
- diseñar afinidad VM–VM y VM–Host, positiva y negativa;
- decidir conscientemente entre reglas preferentes y obligatorias;
- distinguir etiquetas de afinidad, tags y etiquetas de red;
- explicar HA sin confundirla con tolerancia a fallos continua;
- relacionar fencing, lease, watchdog y capacidad N+1;
- evaluar los principales controles de CPU, memoria y E/S;
- distinguir eventos, tareas y logs;
- seguir una operación desde Engine hasta VDSM, libvirt y QEMU;
- diagnosticar con hipótesis y cambios mínimos;
- reconocer qué limitaciones impiden demostrar HA completa en el aula.
