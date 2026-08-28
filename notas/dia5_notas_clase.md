# Día 5 · Notas para impartir la clase

Estas notas acompañan a `dia5_conceptos.md`. No son un resumen del documento formal: indican cómo contarlo, qué mostrar y dónde insistir.

---

# Idea central del día

Hoy deben unir dos perspectivas:

1. cómo observar una plataforma ya en funcionamiento;
2. cómo diseñar e instalar una plataforma que después pueda operarse.

La relación entre ambas es directa:

> Si no conocemos las decisiones de instalación, una alerta carece de contexto. Si no monitorizamos, no sabemos si aquellas decisiones siguen funcionando.

La jornada contiene un cierre de afinidad y HA. Usar para ello `dia4_conceptos.md` y `dia4_notas_clase.md`; no intentar comprimir de nuevo todos los conceptos en estas notas.

---

# Preparación antes de empezar

Abrir con antelación:

- Administration Portal;
- eventos recientes de Engine;
- detalle de un host y una VM;
- detalle de `curso-nfs`;
- terminal de Engine con acceso de solo lectura;
- Grafana externo del laboratorio;
- Mailpit, si sigue disponible;
- `dia5_conceptos.md` en la parte del mapa de observabilidad;
- documentación oficial de requisitos y firewall por si preguntan por una versión exacta.

No preparar una reinstalación en directo. La sesión de instalación debe parecer una revisión de arquitectura y un ensayo de preproducción, no una sucesión de comandos que los alumnos copian sin entender.

---

# 00:00–00:40 · Cierre de afinidad y HA

## Objetivo

Cerrar lo pendiente del día 4 sin consumir la sesión nueva.

## Secuencia sugerida

### Minutos 0–15 · Afinidad

Antes de hablar de HA, recuperar la pregunta que debe ordenar todo el bloque:

> Cuando una VM debe arrancar, migrar o recuperarse, ¿cómo decide Engine qué host utilizar?

No responder «elige el host con menos carga». El scheduler trabaja en dos pasos:

```text
Todos los hosts del Cluster
            ↓
Filtros: eliminan destinos imposibles
            ↓
Pesos: ordenan los destinos válidos
            ↓
Engine selecciona un host
```

Una regla de afinidad **obligatoria** puede actuar como filtro y eliminar un host. Una afinidad **preferente** modifica la prioridad, pero permite utilizar el host si no existe una alternativa mejor.

#### Grupo de afinidad

Un grupo de afinidad relaciona una o varias VMs con:

- otras VMs;
- uno o varios hosts;
- una regla positiva o negativa;
- el carácter preferente u obligatorio de la regla.

El grupo pertenece al Cluster. No configura la red de las VMs, no reserva CPU y no proporciona alta disponibilidad por sí solo: expresa **dónde se desea o se permite ejecutar una carga**.

#### Las cuatro relaciones que hay que distinguir

| Relación | Positiva | Negativa |
|---|---|---|
| VM–VM | Intentar juntar las VMs | Intentar separar las VMs |
| VM–Host | Ejecutar en los hosts incluidos | Evitar los hosts incluidos |

No decir simplemente «afinidad positiva significa juntar». Eso es correcto para VM–VM, pero en una regla VM–Host significa seleccionar los hosts incluidos en el grupo.

Ejemplos para contarlo:

- **VM–VM positiva:** aplicación y caché que intercambian mucho tráfico. Reduce distancia, pero concentra el dominio de fallo.
- **VM–VM negativa:** dos nodos redundantes que interesa mantener en hosts diferentes.
- **VM–Host positiva:** una carga que solo puede ejecutarse en hosts con una licencia o un hardware determinado.
- **VM–Host negativa:** una VM que debe evitar hosts reservados o afectados por una incompatibilidad.

#### Preferente frente a obligatoria

En la GUI, la opción decisiva es **Enforcing**:

| `Enforcing` | Comportamiento | Si la regla no puede cumplirse |
|---:|---|---|
| No | Preferente o `soft` | Engine puede incumplirla y arrancar la VM |
| Sí | Obligatoria o `hard` | Engine descarta los destinos incompatibles y puede dejar la VM apagada |

Repasar con este caso:

```text
Dos nodos de aplicación
Dos hosts disponibles
Anti-afinidad negativa preferente
```

Preguntar:

- ¿qué intenta hacer el scheduler normalmente?
- ¿qué ocurre si solo queda un host?
- ¿qué cambiaría al marcar `Enforcing`?
- ¿qué diferencia hay entre una regla VM–VM y una VM–Host?

La respuesta que interesa no es la definición, sino la decisión de negocio:

> Preferente acepta degradar la separación para conservar servicio. Obligatoria acepta dejar una VM apagada para conservar la restricción.

#### Etiquetas de afinidad

Una etiqueta de afinidad relaciona VMs y hosts mediante un nombre común. Por ejemplo, una VM con la etiqueta `gpu` solo debe encontrar válidos los hosts etiquetados como `gpu`.

Explicar inmediatamente lo que **no** es:

| Objeto | Función |
|---|---|
| Tag administrativo | Clasificar y buscar objetos |
| Network label | Ayudar a desplegar Logical Networks sobre NICs |
| Affinity label | Condicionar la colocación de una VM respecto a hosts |

La etiqueta es sencilla, pero menos expresiva que un grupo: no representa por sí sola una relación negativa ni una preferencia blanda.

#### Conflictos y ausencia de destinos

Plantear en la pizarra:

```text
VM requiere etiqueta gpu
Host 1 tiene gpu, pero una regla obligatoria lo excluye
Host 2 tiene gpu, pero está en mantenimiento
Host 3 tiene capacidad, pero no tiene gpu

Resultado: ningún host candidato
```

La conclusión es importante para diagnóstico:

> Que el Cluster tenga CPU y memoria libres no significa que exista un host válido. Las restricciones se intersectan.

#### Comparación con vSphere

| OLVM | vSphere |
|---|---|
| Afinidad VM–VM positiva | Keep Virtual Machines Together |
| Afinidad VM–VM negativa | Separate Virtual Machines |
| Afinidad VM–Host | VM/Host Rules |
| Preferente | Should run |
| Obligatoria | Must run |
| Scheduler del Cluster | Parte del papel conceptual de DRS |

Cerrar el apartado enlazando con HA:

> Cuando una VM debe recuperarse, el scheduler vuelve a evaluar afinidad, redes, almacenamiento, CPU, memoria y dispositivos. Marcar una VM como altamente disponible no anula ninguna regla obligatoria.

### Minutos 15–35 · HA

Seguir el camino:

```text
Host deja de responder
        ↓
Engine pierde certeza
        ↓
Fencing elimina la ejecución dudosa
        ↓
Scheduler busca un destino
        ↓
La VM arranca de nuevo
```

Volver a señalar que el aula no tiene BMC físico en los hosts anidados. No simular el fencing desconectando sistemas externos.

### Minutos 35–40 · Puente

Cerrar con una pregunta:

> Si una recuperación tarda tres minutos, ¿cómo demostramos en qué paso se consumió el tiempo?

La respuesta abre el tema nuevo: eventos, logs y métricas.

---

# 00:40–01:40 · Monitorización y observabilidad

# 1. Empezar por un síntoma, no por Grafana

Plantear:

> Una VM está lenta. ¿Dónde miraríais primero?

Anotar las respuestas sin corregir todavía: CPU, RAM, host, storage, red, logs, aplicación.

Después explicar que todas pueden ser correctas, pero necesitamos ordenar la investigación:

```text
Objeto afectado
      ↓
Momento exacto
      ↓
Operación que estaba ocurriendo
      ↓
Capa responsable
      ↓
Evidencia adecuada
```

# 2. Estado, evento, tarea, log y métrica

No leer la tabla. Construirla verbalmente con un arranque lento:

- ahora la VM está `Up`: estado;
- a las 10:04 falló un primer arranque: evento;
- la activación del disco tardó: tarea;
- VDSM esperó al volumen NFS: log;
- la latencia subió en ese intervalo: métrica;
- se superó el umbral durante cinco minutos: alerta.

Comparación con vSphere:

- estado del objeto;
- Tasks and Events;
- logs de vCenter y ESXi;
- gráficas de rendimiento.

La estructura mental es equivalente aunque los componentes no se llamen igual.

# 3. Demostración breve en el portal

Elegir una VM de prueba y abrir:

1. su resumen;
2. el host actual;
3. eventos;
4. discos;
5. métricas visibles.

Pedir que indiquen qué información es actual y cuál representa historia.

Después abrir el Storage Domain NFS y recordar:

> OLVM conoce el estado lógico del dominio. Para demostrar latencia del servidor NFS quizá necesitemos métricas y logs fuera de Engine.

# 4. Las dos bases

Dibujar dos cajas grandes:

```text
engine
Estado y configuración

ovirt_engine_history
Histórico y estadísticas
```

Preguntar dónde buscarían:

- el host actual de una VM;
- la utilización de CPU de la semana pasada;
- el nombre de una red lógica;
- la tendencia de crecimiento de un Storage Domain.

Introducir DWH como el movimiento entre ambos mundos:

```text
Engine DB → ETL de DWH → History DB → Grafana
```

No convertir DWH en una explicación de esquemas SQL. Deben recordar función, dirección del flujo y consecuencia de que el servicio se detenga.

Si DWH se detiene:

- Engine puede seguir administrando;
- la plataforma no pierde inmediatamente la ejecución de VMs;
- el histórico deja de actualizarse o queda retrasado;
- Grafana puede mostrar datos antiguos o huecos.

# 5. Grafana nativo frente al del aula

Este punto es importante porque visualmente ambos se llaman Grafana.

Preguntar:

> Si vemos un dashboard de Engine en Grafana, ¿demuestra que Grafana consulta DWH?

Respuesta: no.

Explicar los dos caminos:

```text
Nativo:   Engine → DWH → ovirt_engine_history → Grafana

Aula:     sistemas → collectd → Prometheus → Grafana externo
```

Mostrar en el Grafana externo su datasource, si puede hacerse sin modificar nada. El objetivo no es aprender PromQL; es demostrar que el origen de datos define qué significa el panel.

# 6. PostgreSQL de solo lectura

Antes de ejecutar nada, decir:

> No vamos a tocar las tablas de Engine. Vamos a preguntar al servidor PostgreSQL por su propia salud.

Ejecutar como máximo las cuatro consultas del documento formal.

## Tamaño

Detenerse en dos cuestiones:

- un tamaño grande no es por sí mismo un error;
- lo relevante es crecimiento, espacio disponible y política de retención.

## Sesiones

Explicar `active`, `idle` e `idle in transaction` sin dramatizar. Una conexión inactiva puede formar parte de un pool normal.

## Esperas y bloqueos

Si no existen bloqueos pendientes, es una buena salida. No hace falta provocar uno.

Frase de seguridad:

> Ver un bloqueo no autoriza a matar el proceso; primero identificamos quién espera, quién bloquea, desde cuándo y qué operación representa.

# 7. Alertas y notificaciones

Abrir eventos y, si existe un mensaje de prueba, mostrar el recorrido hacia Mailpit:

```text
evento → suscripción → notifier → SMTP → Mailpit
```

Preguntar qué revisarían si el evento aparece en el portal pero no llega el correo.

Dejar SNMP como mecanismo de integración con la gestión corporativa. En organizaciones grandes, lo normal es que los equipos no permanezcan mirando constantemente la GUI de OLVM.

# 8. Cierre del bloque

Pedir una frase por cada herramienta:

- Administration Portal orienta;
- Engine DB mantiene estado;
- DWH construye histórico;
- Grafana visualiza;
- logs explican;
- Notifier entrega eventos;
- `ovirt-log-collector` reúne evidencias;
- PCP detalla rendimiento del sistema.

---

# 01:40–01:55 · Pausa

---

# 01:55–03:10 · Diseño de la instalación

# 1. No empezar por `dnf`

Abrir el bloque diciendo:

> La primera decisión de una instalación de OLVM no es qué paquete instalar. Es qué dependencia aceptamos para Engine.

Presentar Standalone y Self-Hosted Engine.

# 2. Standalone no significa físico

Usar el laboratorio:

- `olvm-engine` es una VM;
- vive en `worker2`;
- no está controlada por los hosts OLVM;
- desde el punto de vista de OLVM es Standalone.

Preguntar:

> Si apagamos toda la plataforma OLVM, ¿quién arranca la VM de Engine del aula?

Respuesta: la capa externa de `worker2`, no el mecanismo Hosted Engine.

Después preguntar:

> ¿Y en Self-Hosted Engine?

Respuesta: los hosts y agentes Hosted Engine coordinan el arranque de la VM especial de Engine.

# 3. Comparación de topologías

Construir la tabla en la pizarra:

| Decisión | Standalone | Self-Hosted |
|---|---|---|
| Quién aloja Engine | Infraestructura externa | Hosts OLVM |
| Dependencia de storage OLVM al arrancar | No necesariamente | Sí, storage compartido de Hosted Engine |
| Complejidad de bootstrap | Menor | Mayor |
| HA del Engine | La proporciona el entorno externo | La proporcionan los hosts Hosted Engine |
| Mantenimiento especial | No Hosted Engine | Modo global en determinadas operaciones |

No presentar una opción como siempre mejor. La organización puede disponer ya de una plataforma corporativa adecuada para un Standalone Engine.

# 4. Taller verbal de preinstalación

Dividir la pizarra en ocho áreas:

```text
Soporte | Capacidad | DNS | Tiempo
Red     | Firewall  | Storage | Identidad
```

Pedir a los alumnos que aporten al menos dos comprobaciones en cada área.

## Soporte

- versión exacta de Oracle Linux;
- kernel;
- repositorios;
- compatibilidad del Cluster;
- función soportada o Technology Preview.

Insistir en que los números exactos de versión se consultan. En examen puede aparecer una versión concreta, pero en producción prevalece la matriz vigente.

## Capacidad

Diferenciar mínimo de recomendado. El mínimo puede permitir arrancar `engine-setup` y seguir siendo inadecuado para miles de VMs o una retención DWH amplia.

## DNS

Mostrar `hostname -f` y explicar por qué el FQDN termina en:

- certificado;
- URL del portal;
- SSO;
- configuración de hosts;
- logs y notificaciones.

## Tiempo

Relacionarlo con TLS, tickets de autenticación y correlación de logs. Esta relación suele recordarse mejor que «hay que instalar Chrony».

## Red y storage

No rediseñar el día 2. Solo comprobar que instalación necesita decisiones previas:

- red de gestión;
- migración;
- storage;
- VMs;
- BMC;
- NFS o cabina accesible desde todos los hosts.

# 5. Firewall por conversación

Evitar recitar puertos. Dibujar participantes:

```text
Cliente — Engine — Host 1 — Host 2 — Storage
```

Preguntar qué conversaciones existen:

- cliente con portal;
- Engine con VDSM;
- Engine con host durante bootstrap;
- host con host durante migración;
- cliente con host o proxy para consola;
- host con NFS;
- Engine/DWH con PostgreSQL si es remoto.

Después asociar los puertos representativos del documento.

Puntos que suelen confundirse:

- 54321 es VDSM, no PostgreSQL;
- 5432 es PostgreSQL;
- migración exige tráfico host a host;
- abrir el portal no habilita las consolas;
- la configuración automática del firewall local no cambia el firewall corporativo;
- OVN añade tráfico opcional, no está presente por usar OLVM nativo.

# 6. Ejercicio de matriz de firewall

Proponer este requisito:

```text
Engine en red de gestión
Hosts en dos racks
PostgreSQL local
NFSv4 compartido
Consola HTML5 mediante proxy local
Sin OVN
```

Pedir que tachen flujos innecesarios:

- PostgreSQL remoto: no;
- OVN: no;
- Engine a host por VDSM: sí;
- host a host por migración: sí;
- host a NFS: sí;
- cliente a portal y proxy: sí.

La habilidad examinable es relacionar función y flujo, no abrir todos los puertos «por si acaso».

# 7. Repositorios

Explicar los tres niveles:

```text
Repositorios → paquete release → componente
```

El paquete release configura el conjunto de repositorios. En un host no instala Engine. En Engine, `ovirt-engine` instala el software, pero aún falta `engine-setup`.

Cerrar el bloque con una pregunta:

> ¿Qué cosas debemos conocer antes de ejecutar el primer comando?

La lista debe incluir topología, nombres, versiones, red, storage, firewall, identidad, capacidad y recuperación.

---

# 03:10–03:25 · Pausa

---

# 03:25–04:25 · Flujo de instalación

# 1. Standalone Engine paso a paso

Mostrar el flujo de once pasos del documento formal. No ejecutar cada comando.

Concentrarse en tres fronteras:

```text
Preparar sistema
      ↓
Instalar software
      ↓
Configurar plataforma
```

## Preparar

- Oracle Linux mínimo;
- FQDN e IP;
- DNS y tiempo;
- repositorios;
- firewall planificado.

## Instalar

- paquete release;
- `ovirt-engine`;
- dependencias y módulos requeridos por la versión.

## Configurar

- `engine-setup`;
- bases;
- DWH y Grafana;
- SSO;
- proxies;
- certificados;
- firewall local.

# 2. Recorrer `engine-setup` como árbol de decisiones

No leer todas las preguntas del asistente. Para cada grupo preguntar «¿qué consecuencia tiene?».

## FQDN

Si está mal, no es cosmética. Afecta certificados, acceso y referencias distribuidas.

## Base local o remota

La base local reduce complejidad. La remota puede aislar carga, pero añade:

- una nueva dependencia de red;
- credenciales y cifrado;
- firewall 5432;
- backup coordinado;
- latencia;
- soporte dependiente de versión.

## DWH y escala

Relacionar tamaño del entorno y retención. El histórico no debe ocupar todo el filesystem de Engine.

## Grafana

Conecta con DWH. Recordar que en el aula no se instaló la integración nativa aunque exista otro Grafana.

## Keycloak

Es el mecanismo moderno de identidad y SSO. La federación con AD no se improvisa durante `engine-setup`, pero la elección inicial prepara la arquitectura.

## OVN

Es opcional y dependiente de versión. No hace falta para el camino de Linux bridges estudiado. No abrir una digresión sobre Geneve.

## Firewall automático

Puede ser cómodo en un laboratorio. En producción debe estar acordado porque altera la configuración local.

# 3. Qué guardar de Setup

Enseñar que el resultado incluye:

- log del proceso;
- archivo de respuestas;
- URL;
- credenciales iniciales;
- certificados;
- decisiones de bases y componentes.

No mostrar contraseñas reales de la instalación.

# 4. Self-Hosted Engine

Explicarla como bootstrap, no como «instalar Engine dentro de una VM».

```text
Primer host preparado
      ↓
Appliance crea VM de Engine
      ↓
Engine se configura dentro
      ↓
Engine incorpora el host y storage
      ↓
Se añaden más hosts Hosted Engine
```

Tres ideas que deben quedar:

1. necesita storage compartido desde el despliegue;
2. la VM es especial y los agentes conocen su estado;
3. el mantenimiento global evita recuperaciones automáticas durante una parada planificada.

# 5. Alta de un host

Usar la pantalla de creación de host sin guardar cambios.

Recorrer:

- Cluster;
- nombre;
- FQDN;
- SSH;
- clave pública recomendada;
- firewall automático;
- power management.

Después explicar el trabajo oculto durante `Installing`:

- conexión;
- instalación de VDSM;
- certificados;
- inventario;
- validación de CPU, red y storage;
- paso a `Up`.

Compararlo con vSphere: añadir ESXi a vCenter también establece gestión y confianza, pero en OLVM debemos reconocer explícitamente el papel de VDSM y el bootstrap por SSH.

# 6. Caso de host atascado

Presentar:

```text
Host nuevo
Estado Installing durante 20 minutos
Engine resuelve su FQDN
SSH falla
```

Preguntar si tiene sentido investigar NFS primero. Respuesta: no; el flujo todavía no ha superado el bootstrap.

Cambiar el caso:

```text
SSH funciona
VDSM está instalado
El host queda Non Operational
La red de gestión requerida no está asignada
```

Ahora sí hay comunicación, pero la configuración no cumple el Cluster. Relacionarlo con los estados vistos el día 4.

---

# 04:25–05:00 · Validación, caso y examen

# 1. Checklist de aceptación

Pedir al grupo que no diga «el portal abre» como única prueba.

Construir cuatro columnas:

| Sistema | Control | Datos | Operación |
|---|---|---|---|
| DNS, NTP, repos | Portal, API, Engine | Storage y redes | Eventos y backup |
| Servicios | Hosts Up | Migración | DWH y alertas |
| Filesystems | Certificados | Consolas | Documentación |

## Pruebas mínimas

- acceder por FQDN y validar certificado;
- añadir hosts y comprobar `Up`;
- adjuntar storage compartido;
- desplegar redes desde Engine;
- crear una VM de prueba;
- abrir consola;
- migrarla si la arquitectura lo permite;
- observar eventos y métricas;
- probar una notificación;
- crear y extraer un backup inicial;
- comprobar fencing de manera planificada en producción.

# 2. Caso integrado

Plantear una empresa con 40 hosts, dos racks y almacenamiento redundante.

Preguntar en este orden:

1. ¿Standalone o Self-Hosted y por qué?
2. ¿Qué debe existir en DNS antes de instalar?
3. ¿Qué flujos son host a host?
4. ¿Qué debe monitorizarse para demostrar N+1?
5. ¿Dónde observaríamos tendencia de CPU del último mes?
6. ¿Qué recogeríamos para un SR de Oracle?
7. ¿Qué prueba falta si todas las VMs arrancan pero nunca se probó fencing?

No buscar una arquitectura única. Evaluar si justifican dependencias, dominios de fallo y procedimiento de recuperación.

# 3. Repaso de examen

Utilizar los documentos de preguntas. Seleccionar al menos:

- una pregunta sobre `engine` frente a `ovirt_engine_history`;
- una sobre DWH y Grafana;
- una sobre `ovirt-log-collector`;
- una sobre Standalone frente a Self-Hosted;
- una sobre firewall host a host;
- una sobre `dnf install ovirt-engine` frente a `engine-setup`;
- una sobre el estado `Installing` de un host.

Pedir siempre que descarten las opciones incorrectas. El examen suele mezclar un componente real con una función que pertenece a otro.

# 4. Cierre final

Terminar con cuatro frases para que las completen:

```text
Engine sirve para...
DWH sirve para...
engine-setup sirve para...
VDSM sirve para...
```

Respuestas esperadas:

- Engine mantiene el plano de control y coordina;
- DWH construye histórico para análisis;
- `engine-setup` convierte paquetes instalados en una plataforma configurada;
- VDSM ejecuta y reporta en cada host las operaciones ordenadas por Engine.

Última idea:

> Una instalación no termina cuando aparece el portal. Termina cuando la arquitectura está validada, monitorizada, documentada y recuperable.

---

# Plan B si falla alguna demostración

## No hay datos en Grafana externo

Usar el flujo conceptual y revisar el datasource sin intentar reparar Prometheus durante la clase.

## DWH no está activo

Convertirlo en ejercicio: Engine sigue administrando, pero el histórico no avanza. Revisar estado y logs sin reiniciar si no se conoce el impacto.

## Las consultas PostgreSQL no funcionan

No pedir contraseñas ni improvisar acceso. Mostrar las consultas y explicar la salida esperada.

## Mailpit no contiene mensajes

Seguir verbalmente la cadena evento–suscripción–Notifier–SMTP. La ausencia del mensaje es un ejemplo válido de diagnóstico.

## No se puede abrir la pantalla de alta de host

Usar el flujo formal. No añadir un host ficticio ni reutilizar credenciales reales en una pantalla compartida.

## Falta tiempo

Mantener obligatoriamente:

1. Engine DB frente a DWH;
2. Grafana nativo frente al externo;
3. Standalone frente a Self-Hosted;
4. requisitos DNS/tiempo/firewall;
5. paquete frente a `engine-setup`;
6. qué ocurre al añadir un host.

Reducir las consultas PostgreSQL a tamaño y sesiones, y dejar el caso de arquitectura como ejercicio final.
