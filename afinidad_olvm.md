# Afinidad en OLVM · Conceptos, explicación y operación

Este documento es un guion independiente para explicar la afinidad en Oracle Linux Virtualization Manager y mostrar cómo se configura desde el portal. La afinidad no es una propiedad de Linux que tengamos que crear manualmente en los hosts: es una decisión de colocación administrada por **Engine**.

---

# 1. La pregunta que ordena todo el tema

Cuando una VM debe arrancar, migrar o recuperarse después de un fallo, Engine tiene que decidir:

> **¿En qué host del clúster puede y conviene ejecutar esta VM?**

La respuesta la calcula el **scheduler** de Engine. La afinidad introduce condiciones o preferencias en esa decisión.

```text
Administrador
      ↓
Configura la regla en Engine
      ↓
Engine la guarda en su base de datos
      ↓
El scheduler evalúa los hosts posibles
      ↓
Engine elige un host
      ↓
VDSM ejecuta en ese host la orden recibida
```

Por tanto:

- la regla se crea en el portal de OLVM o mediante la API;
- Engine la conserva y la evalúa;
- VDSM no decide la política de afinidad;
- no hay que crear un fichero de afinidad en el host;
- la regla no configura bridges, VLAN, storage ni el sistema operativo invitado.

## Comparación con vSphere

La idea se parece a las reglas de **vSphere DRS**:

| OLVM | vSphere |
|---|---|
| Scheduler de Engine | DRS |
| Afinidad positiva entre VMs | Keep Virtual Machines Together |
| Afinidad negativa entre VMs | Separate Virtual Machines |
| Afinidad VM–host | VM/Host Rules |
| Regla preferente | Should run / soft rule |
| Regla obligatoria | Must run / hard rule |

La comparación es conceptual. Los nombres, los módulos y la forma de aplicar la política no son idénticos.

---

# 2. Cómo el scheduler elige un host

El scheduler no se limita a buscar el host con menos CPU. Trabaja, simplificando, en dos pasos.

```text
Todos los hosts del clúster
            ↓
Filtros: eliminan hosts imposibles
            ↓
Pesos: ordenan los hosts que quedan
            ↓
Engine selecciona el destino
```

## Filtros: condiciones obligatorias

Un filtro responde «válido» o «no válido». Puede descartar un host porque:

- no está en estado `Up`;
- su CPU no es compatible con el clúster o con la VM;
- no dispone de la memoria necesaria;
- no tiene una red lógica obligatoria;
- no puede acceder al Storage Domain;
- no cumple una configuración de pinning o NUMA;
- carece de un dispositivo de host requerido;
- incumple una regla de afinidad marcada como obligatoria.

Si todos los hosts son descartados, la VM no arranca o no puede migrar. No significa que el scheduler esté averiado: puede estar obedeciendo exactamente las restricciones definidas.

## Pesos: preferencias

Una vez descartados los destinos imposibles, los módulos de peso puntúan los hosts válidos. Pueden valorar, por ejemplo:

- utilización de CPU;
- memoria disponible;
- número de VMs;
- ahorro de energía;
- afinidades preferentes.

Un peso empeora o mejora la posición de un host, pero no tiene por qué eliminarlo.

## Balanceador

La política de planificación del clúster también puede decidir cuándo existe un desequilibrio suficiente para proponer migraciones. Esto es diferente de elegir el destino durante un arranque, aunque ambos comportamientos forman parte de la política de scheduling.

---

# 3. Qué es un grupo de afinidad

Un **grupo de afinidad** es un objeto de Engine, definido dentro de un clúster, que puede reunir:

- una o varias VMs;
- opcionalmente, uno o varios hosts;
- una regla de relación entre las VMs;
- opcionalmente, una regla entre esas VMs y los hosts;
- el carácter preferente u obligatorio de cada relación.

El grupo pertenece al ámbito del **clúster**. No relaciona VMs de clústeres diferentes. Si una VM se mueve a otro clúster, deja de formar parte de los grupos del clúster anterior.

La regla se tiene en cuenta en la siguiente decisión de colocación: arranque, migración o recuperación. Crear el grupo no obliga a reiniciar el sistema operativo invitado.

---

# 4. Las cuatro relaciones que hay que dominar

## 4.1 VM–VM positiva: ejecutar juntas

Engine intenta colocar las VMs del grupo en el mismo host.

```text
VM aplicación ─┐
               ├── Host 1
VM caché ──────┘
```

Puede reducir latencia entre dos VMs que se comunican mucho. La contrapartida es que concentra el riesgo: si cae el host, caen ambas.

## 4.2 VM–VM negativa: ejecutar separadas

Engine intenta repartir las VMs del grupo entre hosts distintos.

```text
VM nodo 1 ─── Host 1
VM nodo 2 ─── Host 2
```

Es el caso habitual para:

- nodos redundantes de una aplicación;
- controladores o balanceadores duplicados;
- réplicas que no interesa perder con el mismo fallo físico.

Separar VMs no crea por sí solo alta disponibilidad. La aplicación debe soportar la redundancia y OLVM necesita igualmente storage, fencing y capacidad de recuperación adecuados.

## 4.3 VM–host positiva: usar determinados hosts

Las VMs deben o prefieren ejecutarse en los hosts incluidos en el grupo.

Ejemplos:

- hosts con GPU;
- hosts autorizados por una licencia;
- servidores con una generación concreta de CPU;
- hosts situados en una ubicación determinada.

## 4.4 VM–host negativa: evitar determinados hosts

Las VMs deben o prefieren ejecutarse fuera de los hosts incluidos.

Ejemplos:

- evitar servidores reservados para una carga crítica;
- excluir una generación de hardware con una incompatibilidad conocida;
- impedir que un entorno de pruebas ocupe hosts reservados para producción.

---

# 5. `Enforcing`: preferencia frente a obligación

La casilla **Enforcing** es la decisión más importante de una regla.

| Enforcing | Tipo práctico | Qué ocurre si no se puede cumplir |
|---:|---|---|
| Desactivado | Preferente o `soft` | Engine intenta cumplirla, pero puede incumplirla para arrancar o migrar la VM |
| Activado | Obligatoria o `hard` | Engine elimina los hosts que la incumplen; la VM puede quedarse sin destino |

En algunas versiones del formulario aparecen controles separados para la relación **VM–VM** y para la relación **VM–host**. Cada relación debe analizarse por separado.

## Ejemplo: dos nodos redundantes

Con anti-afinidad VM–VM negativa y preferente, OLVM intenta separarlos. Si solo queda un host disponible, permite ejecutar ambos allí y mantener el servicio degradado.

Con anti-afinidad negativa y obligatoria, uno de los nodos puede quedarse apagado porque ya no existe un segundo destino válido.

La pregunta no es solo técnica:

> **¿Es más importante mantener la separación o mantener el servicio, aunque sea degradado?**

## Ejemplo: licencia ligada a hardware

Si una aplicación solo puede ejecutarse legalmente en dos hosts, una afinidad VM–host positiva y obligatoria representa una restricción real. Si los dos hosts están caídos, que la VM no arranque en otro servidor es el comportamiento esperado.

## Recomendación docente y operativa

Empezar con reglas preferentes, observar su comportamiento y convertirlas en obligatorias solo cuando exista un requisito que justifique dejar una VM sin arrancar. En producción, una regla `hard` debe revisarse junto con la capacidad N+1 y los escenarios de mantenimiento.

---

# 6. Etiquetas de afinidad

Una **etiqueta de afinidad** relaciona VMs y hosts mediante un nombre común. Si una VM requiere una etiqueta, el scheduler solo considera los hosts que tienen esa misma etiqueta. Se comporta como una relación VM–host positiva y obligatoria, evaluada por el módulo `Label` de la política de scheduling.

Ejemplo:

```text
Etiqueta: gpu

VM-IA ─────────────┐
                   ├── Solo hosts con etiqueta gpu
Host-03 ─ gpu ─────┤
Host-04 ─ gpu ─────┘
```

Es útil para expresar conjuntos reutilizables como:

- `gpu`;
- `licencia-oracle`;
- `cpu-nueva`;
- `zona-a`;
- `baja-latencia`.

## No confundir tres tipos de etiqueta

| Objeto | Para qué sirve |
|---|---|
| Etiqueta de afinidad | Condiciona la colocación de VMs sobre hosts |
| Etiqueta administrativa o tag | Clasifica objetos, ayuda a buscar y puede participar en permisos |
| Etiqueta de red | Relaciona redes lógicas con interfaces de host |

Que las tres se llamen «etiqueta» no significa que intervengan en el mismo mecanismo.

## Grupo o etiqueta

| Necesidad | Objeto más adecuado |
|---|---|
| Mantener juntas o separar VMs concretas | Grupo de afinidad |
| Preferir u obligar una relación VM–host | Grupo de afinidad |
| Reutilizar una clase de hosts para varias VMs | Etiqueta de afinidad |
| Crear una relación VM–host positiva estricta y sencilla | Etiqueta de afinidad |

Las etiquetas son cómodas, pero menos expresivas: no sustituyen las reglas VM–VM ni una preferencia `soft`.

---

# 7. La política de planificación del clúster

Los grupos y las etiquetas solo pueden influir en la colocación si la política de scheduling aplicada al clúster contiene los módulos que los interpretan.

| Módulo | Función |
|---|---|
| `VmAffinityGroups` | Evalúa la afinidad entre VMs |
| `VmToHostsAffinityGroups` | Evalúa la afinidad entre VMs y hosts |
| `Label` | Evalúa las etiquetas de afinidad |

No es necesario crear una política personalizada para una demostración básica si la política elegida ya incorpora estos módulos.

## Ver la política aplicada al clúster

En el portal de administración:

1. Ir a **Cómputo → Clústeres**.
2. Seleccionar el clúster, por ejemplo `Curso`.
3. Pulsar **Editar**.
4. Abrir **Política de planificación**. Según la traducción puede aparecer como **Scheduling Policy** o **Política de programación**.
5. Anotar la política seleccionada.

## Revisar o crear políticas

1. Ir a **Administración → Configurar**.
2. Abrir **Políticas de planificación**.
3. Revisar los filtros, pesos, balanceador y propiedades de la política.

No conviene modificar la política del clúster durante la clase sin haber comprobado antes el impacto. Una política personalizada incorrecta puede afectar a todas las decisiones de colocación del clúster.

---

# 8. Cómo crear y operar un grupo de afinidad en OLVM

Los nombres pueden variar ligeramente con la traducción del portal, pero la secuencia en OLVM 4.5 es la siguiente.

## Antes de empezar

Comprobar que:

- las VMs pertenecen al mismo clúster;
- existen al menos dos hosts `Up` para demostrar anti-afinidad;
- las VMs de la prueba son prescindibles o controladas;
- conocemos la política de scheduling aplicada;
- el requisito se puede expresar primero como preferencia.

## Procedimiento A · Crear una anti-afinidad entre dos VMs

1. Ir a **Cómputo → Máquinas virtuales**.
2. Seleccionar una de las VMs y pulsar su nombre para abrir el detalle.
3. Abrir la pestaña **Grupos de afinidad**.
4. Pulsar **Nuevo**.
5. Escribir un nombre descriptivo, por ejemplo `separar-nodos-web`.
6. Añadir una descripción que explique el motivo de negocio.
7. En **Regla de afinidad de VM**, seleccionar **Negativa**.
8. Para la primera prueba, dejar **Enforcing** desactivado.
9. Añadir las dos VMs mediante el botón `+` o el selector disponible.
10. Pulsar **Aceptar**.

Resultado esperado: cuando Engine vuelva a colocar las VMs, preferirá hosts diferentes, pero podrá reunirlas si no existe otro destino válido.

> Crear la regla no migra necesariamente las VMs de inmediato. La regla participa en las decisiones del scheduler y el balanceador de la política determina si debe corregirse una distribución existente.

## Procedimiento B · Añadir una relación VM–host

En el mismo formulario del grupo:

1. Localizar **Regla de afinidad de host**.
2. Elegir **Positiva** para utilizar los hosts seleccionados o **Negativa** para evitarlos.
3. Decidir si la relación será preferente o **Enforcing**.
4. Añadir los hosts que forman el conjunto de inclusión o exclusión.
5. Guardar el grupo.

Ejemplo: añadir `host03` y `host04`, elegir relación positiva y dejarla preferente hace que Engine los favorezca sin impedir completamente el uso de otro host.

## Procedimiento C · Editar o borrar el grupo

1. Abrir una VM participante.
2. Entrar en **Grupos de afinidad**.
3. Seleccionar el grupo.
4. Usar **Editar** para cambiar miembros, signo o `Enforcing`.
5. Usar **Eliminar** solo después de comprobar qué VMs dependen de él.

Eliminar el grupo elimina la regla de colocación; no borra las VMs ni los hosts.

---

# 9. Cómo crear y operar una etiqueta de afinidad

## Crear la etiqueta desde el clúster

1. Ir a **Cómputo → Clústeres**.
2. Abrir el nombre del clúster.
3. Entrar en **Etiquetas de afinidad**.
4. Pulsar **Nueva**.
5. Asignar un nombre, por ejemplo `gpu`.
6. Seleccionar las VMs y los hosts que compartirán la etiqueta.
7. Pulsar **Aceptar**.

También puede añadirse una etiqueta existente al editar una VM o un host y abrir su apartado **Etiquetas de afinidad**.

## Qué comprobar después

- La VM tiene la etiqueta esperada.
- Al menos un host `Up` tiene esa misma etiqueta.
- La política del clúster contiene el filtro `Label`.
- El host etiquetado cumple también CPU, memoria, redes, storage y cualquier otro requisito.

La etiqueta no convierte en válido un host que falla otro filtro. Solo añade una condición más.

---

# 10. Cómo verificar el comportamiento

La comprobación debe demostrar el resultado sin provocar un fallo artificial.

## Prueba segura de una regla preferente

1. Anotar en qué host se ejecuta cada VM.
2. Crear el grupo con `Enforcing` desactivado.
3. Migrar manualmente una VM prescindible o apagarla y arrancarla, según el laboratorio.
4. Revisar el host de destino.
5. Consultar **Eventos** si Engine elige otro destino o rechaza la operación.
6. Repetir la prueba con uno de los hosts en mantenimiento solo si el entorno lo permite.

No hay que concluir que la regla está mal solo porque una regla preferente se incumpla. Precisamente una regla `soft` autoriza a Engine a priorizar la continuidad.

## Qué debe explicar el alumno

Al terminar la prueba debe poder responder:

- qué objeto se creó;
- si la relación era VM–VM o VM–host;
- si era positiva o negativa;
- si era preferente u obligatoria;
- qué hosts descartó o favoreció el scheduler;
- qué ocurriría si solo quedara un host disponible.

---

# 11. Relación con HA, migración y mantenimiento

La afinidad no arranca una VM por sí misma. Cuando HA, una migración o una operación manual requieren colocarla, el scheduler vuelve a evaluar las reglas.

## Durante un fallo

- Con una regla preferente, Engine puede incumplirla para recuperar el servicio.
- Con una regla obligatoria, puede no quedar ningún host válido y la VM permanecerá apagada.

## Durante mantenimiento

Al poner un host en mantenimiento, las VMs migrables necesitan destinos que cumplan todas sus restricciones. Una regla `hard`, una etiqueta o un dispositivo ligado a un host pueden impedir la evacuación.

## Lo que afinidad no sustituye

- HA habilitada en la VM;
- fencing funcional;
- almacenamiento compartido accesible;
- redes disponibles en los hosts de destino;
- reserva de capacidad suficiente;
- redundancia real de la aplicación.

Una buena regla de anti-afinidad sin capacidad N+1 puede convertirse en un bloqueo durante la avería que pretendía mitigar.

---

# 12. Diagnóstico: la VM no encuentra host

Empezar por **Eventos** y leer el mensaje completo. Después revisar las restricciones en este orden:

1. ¿Hay algún host `Up` en el clúster?
2. ¿Cumple la familia de CPU y la versión de compatibilidad?
3. ¿Tiene memoria suficiente para la memoria física garantizada?
4. ¿Dispone de todas las redes obligatorias de la VM?
5. ¿Accede a los Storage Domains necesarios?
6. ¿La VM es migrable o tiene pinning, NUMA o dispositivos ligados al host?
7. ¿Existe una regla de afinidad obligatoria que lo descarte?
8. ¿Existe una etiqueta de afinidad sin ningún host válido asociado?
9. ¿La política de scheduling contiene los módulos adecuados?
10. ¿Dos reglas se contradicen?

## Ejemplo de contradicción

Supongamos que:

- `VM-A` tiene la etiqueta `gpu`;
- solo `host03` tiene esa etiqueta;
- otro grupo obliga a `VM-A` a evitar `host03`.

El filtro de etiqueta descarta todos los hosts salvo `host03`, mientras que la afinidad negativa descarta `host03`. El resultado es **cero candidatos**. El problema no se resuelve reiniciando VDSM: hay que corregir la política contradictoria o aportar otro host que cumpla los requisitos.

---

# 13. Casos para discutir en clase

## Caso 1 · Dos controladores redundantes

Objetivo: evitar perder ambos con un solo host.

Propuesta inicial: afinidad VM–VM negativa y preferente.

Pregunta: si solo queda un host, ¿preferimos ejecutar ambos controladores juntos o dejar uno apagado?

## Caso 2 · Aplicación con licencia física

Objetivo: limitar la ejecución a dos hosts autorizados.

Propuesta: afinidad VM–host positiva y obligatoria.

Pregunta: ¿cómo afectará al mantenimiento simultáneo de esos dos hosts?

## Caso 3 · Aplicación y caché con mucho tráfico

Objetivo: reducir latencia.

Propuesta: afinidad VM–VM positiva y preferente.

Pregunta: ¿aceptamos concentrar ambas cargas en el mismo dominio de fallo?

## Caso 4 · VMs que requieren GPU

Objetivo: utilizar solo hosts preparados.

Propuesta: etiqueta de afinidad `gpu` o grupo VM–host positivo obligatorio.

Pregunta: además de la afinidad, ¿la VM utiliza passthrough de un dispositivo que limite la migración?

---

# 14. Guion de explicación y demostración —25 minutos—

## Minutos 0–5 · Idea principal

Explicar que Engine decide el host y que los filtros descartan mientras los pesos prefieren. Compararlo con DRS.

## Minutos 5–10 · Matriz de reglas

Dibujar las cuatro relaciones:

- VM–VM positiva;
- VM–VM negativa;
- VM–host positiva;
- VM–host negativa.

Añadir a cada una la decisión `soft` frente a `hard`.

## Minutos 10–18 · Portal

Crear un grupo VM–VM negativo y preferente con dos VMs del laboratorio. Enseñar dónde se seleccionan las VMs, los hosts y `Enforcing`.

## Minutos 18–22 · Verificación

Mostrar el host actual de cada VM, realizar una operación controlada y leer los eventos. Si Engine incumple una regla preferente, utilizarlo para explicar que no es un error.

## Minutos 22–25 · Riesgo operativo

Convertir verbalmente la regla en obligatoria y preguntar qué ocurriría con un único host `Up`. Cerrar relacionando afinidad, HA, mantenimiento y capacidad N+1.

---

# 15. Resumen para la pizarra

```text
Afinidad = reglas de colocación evaluadas por Engine

Positiva VM–VM  → juntas
Negativa VM–VM  → separadas
Positiva VM–host → dentro del conjunto
Negativa VM–host → fuera del conjunto

Enforcing desactivado → preferencia
Enforcing activado    → obligación; puede impedir el arranque

Etiqueta de afinidad → relación VM–host positiva y estricta

Afinidad no equivale a HA
Una regla hard sin capacidad suficiente puede dejar la VM apagada
```
