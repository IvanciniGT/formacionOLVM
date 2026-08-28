# Día 5 · Monitorización e instalación de OLVM

---

# Qué vamos a conseguir hoy

Hoy cerraremos primero lo que quedó pendiente de afinidad y alta disponibilidad. Para ese tramo utilizaremos los documentos del día 4 y no repetiremos aquí su contenido.

El material nuevo responde a dos preguntas:

> **¿Cómo sabemos qué está pasando en una plataforma OLVM?**

> **¿Qué decisiones y comprobaciones hacen falta para instalarla correctamente?**

Al terminar la jornada quiero que podáis:

1. Diferenciar estado, evento, tarea, alerta, log y métrica.
2. Explicar qué guardan las bases `engine` y `ovirt_engine_history`.
3. Seguir el camino de los datos desde Engine hasta DWH y Grafana.
4. Elegir la fuente adecuada para investigar un problema de VM, host, red o almacenamiento.
5. Utilizar consultas PostgreSQL de lectura para comprobar tamaño, sesiones, actividad y bloqueos.
6. Explicar para qué sirven Notifier, SNMP, PCP y `ovirt-log-collector`.
7. Comparar una instalación Standalone Engine con una Self-Hosted Engine.
8. Preparar una lista razonada de requisitos de Engine, hosts, DNS, tiempo, red, firewall y storage.
9. Diferenciar instalar paquetes de configurar la plataforma mediante `engine-setup`.
10. Explicar qué ocurre realmente cuando se añade un host desde el Administration Portal.
11. Validar una instalación antes de crear cargas de producción.
12. Reconocer los puntos especialmente relevantes para el examen 1Z0-1170 sin reducir la instalación a memorizar comandos.

La frase del día será:

> **Monitorizar es relacionar síntomas con componentes; instalar es convertir decisiones de arquitectura en una plataforma verificable.**

## Puente con vSphere

| OLVM | Referencia útil en vSphere | Matiz |
|---|---|---|
| Engine | vCenter Server | Es el plano de control; las VMs se ejecutan en los hosts |
| Base `engine` | Base de datos de vCenter | Conserva configuración y estado de la plataforma |
| DWH + `ovirt_engine_history` | Histórico de rendimiento de vCenter, aproximadamente | OLVM separa explícitamente el histórico en otra base y un proceso ETL |
| Monitoring Portal / Grafana | Performance charts y plataformas externas | No es el Administration Portal y puede coexistir con otro Grafana empresarial |
| VDSM y `vdsm.log` | Agentes y logs de gestión del ESXi | VDSM ejecuta en el host las órdenes de Engine |
| Standalone Engine | vCenter fuera del Cluster administrado, como analogía | El Engine puede ser físico o VM, pero no debe depender del entorno que administra |
| Self-Hosted Engine | vCenter ejecutado dentro del Cluster | En OLVM existe un despliegue y mecanismo HA específico para la VM del Engine |
| Añadir un host | Incorporar un ESXi a vCenter | Engine instala y configura VDSM y establece confianza; no es un alta puramente administrativa |

---

# Reparto de las cinco horas

| Bloque | Duración | Contenido |
|---|---:|---|
| 0 | 40 min | Cierre de afinidad y HA con los documentos del día 4 |
| 1 | 60 min | Observabilidad: estado, eventos, logs, métricas, DWH y Grafana |
| Pausa | 15 min | |
| 2 | 75 min | Diseño de la instalación: topologías, requisitos, DNS, tiempo, red y firewall |
| Pausa | 15 min | |
| 3 | 60 min | Flujo de instalación: Engine, `engine-setup`, Self-Hosted Engine y alta de hosts |
| 4 | 35 min | Validación, caso integrado y repaso orientado al examen |

Total: **5 horas**, incluidas dos pausas de 15 minutos.

---

# Límites del laboratorio

La instalación del aula sirve para observar componentes reales, pero no representa un despliegue físico empresarial:

- `olvm-engine` se ejecuta como una VM en `worker2`;
- esa VM no es una Self-Hosted Engine administrada por los hosts OLVM;
- `olvm-host1` y `olvm-host2` son hosts KVM anidados sobre `worker3` y `worker4`;
- las VMs utilizan almacenamiento compartido NFS;
- los hosts anidados no tienen BMC físico ni fencing empresarial configurado;
- DWH está instalado;
- la integración nativa de Grafana de OLVM no está instalada;
- existe un camino externo mediante `collectd`, Prometheus y un Grafana externo.

Por ello hoy:

- observaremos servicios, bases, eventos y métricas sin modificar datos;
- no activaremos logging SQL detallado de forma permanente;
- no ejecutaremos mantenimiento de PostgreSQL, `VACUUM` ni borrados;
- no reinstalaremos Engine ni los hosts durante la clase;
- no modificaremos el firewall del aula;
- reconstruiremos el proceso de instalación como ejercicio de arquitectura y validación.

La instalación del laboratorio permite hacer una comparación muy útil:

> **Una VM que contiene Engine no es automáticamente una Self-Hosted Engine.**

En nuestro caso, el ciclo de vida de `olvm-engine` depende de la capa externa que ejecuta `worker2`. Los hosts OLVM administrados no arrancan ni protegen esa VM mediante el mecanismo Hosted Engine.

---

# Bloque 1 · Monitorización y observabilidad

# 1. Monitorizar no es mirar una sola pantalla

Una plataforma puede mostrar todos los objetos en verde y seguir prestando un servicio deficiente. También puede existir un pico de CPU sin que haya una avería.

Necesitamos separar seis conceptos:

| Elemento | Pregunta que responde | Ejemplo |
|---|---|---|
| Estado | ¿Cómo considera OLVM el objeto ahora? | Host `Up`, Storage Domain `Active` |
| Evento | ¿Qué hecho registró Engine? | Falló una migración |
| Tarea | ¿Qué operación está ejecutándose o terminó? | Crear un disco, mover una imagen |
| Alerta | ¿Qué condición requiere atención? | Poco espacio libre en storage |
| Log | ¿Qué ocurrió dentro del componente? | Timeout NFS en `vdsm.log` |
| Métrica | ¿Cuánto, cuándo y durante cuánto tiempo? | Latencia de storage durante una hora |

No son sinónimos:

- el estado muestra la fotografía actual;
- el evento sitúa un cambio en el tiempo;
- la tarea representa una operación;
- el log ofrece el detalle interno;
- la métrica permite observar tendencia y magnitud;
- la alerta aplica una condición a un estado o una métrica.

## Un ejemplo completo

Un alumno informa de que su VM ha tardado mucho en arrancar.

```text
Estado actual: la VM está Up
Evento: el arranque tardó y hubo reintentos
Tarea: la activación del disco terminó
Métrica: la latencia de NFS subió durante diez minutos
Log de VDSM: aparecen esperas al acceder al volumen
Log del servidor NFS: confirma saturación en el mismo intervalo
```

Mirar únicamente el estado final habría ocultado el problema.

# 2. Las capas que debemos observar

OLVM no es un único proceso. La investigación cambia según la capa afectada.

| Capa | Objetos principales | Qué observamos |
|---|---|---|
| Servicio | Aplicación y sistema operativo invitado | Respuesta, errores, procesos, disco y red del guest |
| VM | Definición, QEMU, vCPU, RAM, discos, vNIC | Estado, recursos, migración, snapshots, Guest Agent |
| Host | VDSM, libvirt, KVM, NICs, CPU, RAM | Salud, contención, logs, acceso a redes y storage |
| Storage | Storage Domain, NFS, volumen, servidor | Capacidad, latencia, errores y conectividad |
| Red | Logical Network, bridge, NIC, bond, switches | Link, MTU, VLAN, pérdidas, errores y saturación |
| Control | Engine, base de datos, DWH, servicios | Eventos, tareas, decisiones, históricos y configuración |

La regla práctica es:

> **La capa donde se ve el síntoma no tiene por qué ser la capa que lo causa.**

Una VM lenta puede tener CPU libre y estar esperando a NFS. Un host `Non Operational` puede funcionar perfectamente como Linux, pero no cumplir una red obligatoria del Cluster.

# 3. Estado actual e histórico

Engine necesita conocer el estado actual para administrar. El análisis de tendencia necesita conservar muestras durante mucho más tiempo. OLVM separa ambas responsabilidades.

## Base de datos `engine`

Conserva información persistente necesaria para el plano de control:

- inventario de Data Centers, Clusters, hosts, VMs, redes y Storage Domains;
- configuración y relaciones entre objetos;
- estados operativos;
- usuarios, roles y permisos;
- eventos y datos necesarios para las decisiones de Engine;
- información de rendimiento reciente utilizada por la plataforma.

No debemos consultar o modificar sus tablas como si fueran una API pública. Para automatizar la plataforma se utiliza la API de OLVM. Las consultas PostgreSQL que veremos son diagnósticas y de solo lectura.

## Base de datos `ovirt_engine_history`

Es la base histórica de Data Warehouse. Conserva:

- cambios de configuración a lo largo del tiempo;
- muestras estadísticas de Data Centers, Clusters, hosts y VMs;
- agregaciones que permiten ampliar el periodo de conservación;
- datos destinados a informes y paneles.

La diferencia esencial es:

| `engine` | `ovirt_engine_history` |
|---|---|
| Fuente operativa del plano de control | Fuente histórica y analítica |
| Estado y configuración vigentes | Evolución y estadísticas en el tiempo |
| Engine escribe para administrar | DWH carga datos mediante ETL |
| No es la fuente adecuada para informes pesados | Está pensada para históricos e informes |

# 4. Qué hace Data Warehouse

El servicio de DWH realiza un proceso ETL:

```text
Extract
Lee cambios y estadísticas de la base engine
            ↓
Transform
Adapta los datos al modelo histórico
            ↓
Load
Los inserta en ovirt_engine_history
```

Además de copiar muestras, registra cambios de entidades:

- alta de un objeto;
- cambio de sus propiedades;
- retirada del objeto, conservando la referencia histórica.

La unidad del servicio suele ser `ovirt-engine-dwhd`. El nombre del paquete, el servicio, la base de datos y la función no deben confundirse:

| Nombre | Qué es |
|---|---|
| `ovirt-engine-dwh` | Paquete o componente de Data Warehouse |
| `ovirt-engine-dwhd` | Servicio que ejecuta la carga histórica |
| `ovirt_engine_history` | Base PostgreSQL de histórico |
| Grafana | Herramienta que consulta y visualiza esos datos |

## Frecuencia y retención

Las muestras se recogen periódicamente y después pueden consolidarse en agregaciones horarias y diarias. Conservar cada muestra fina durante años tendría un coste excesivo.

```text
Muestras detalladas
        ↓ consolidación
Datos horarios
        ↓ consolidación
Datos diarios
```

La política exacta se decide durante la instalación mediante la escala de DWH y debe verificarse en la versión instalada. El principio que debemos recordar es:

- más detalle y más retención consumen más almacenamiento y recursos;
- las agregaciones reducen detalle, pero permiten observar tendencias largas;
- un entorno grande puede justificar separar DWH o la base histórica, siempre que la versión y el soporte lo permitan.

Como referencia para la versión documentada el **28 de agosto de 2026**:

| Escala DWH | Muestras detalladas | Agregación horaria | Agregación diaria |
|---|---:|---:|---:|
| Basic —predeterminada— | 24 horas | 1 mes | No se conserva |
| Full | 24 horas | 2 meses | 5 años |

`Full` puede requerir separar DWH en entornos grandes. El dato de examen es la relación entre escala, retención y capacidad; la decisión de producción exige volver a consultar la guía de la release instalada.

# 5. Grafana nativo y Grafana externo

Grafana no sustituye a Engine. Consulta una fuente de métricas y presenta paneles.

## Integración nativa de OLVM

```text
Engine
   ↓ estado y estadísticas
Base engine
   ↓ ETL de DWH
ovirt_engine_history
   ↓ consultas
Monitoring Portal / Grafana
```

La integración puede configurarse desde `engine-setup` y utiliza DWH como fuente. El acceso habitual se realiza mediante el Monitoring Portal y puede integrarse con el sistema de identidad de Engine.

## Camino externo del aula

```text
Engine y hosts
      ↓
collectd
      ↓
Prometheus
      ↓
Grafana externo
```

| Camino nativo | Camino externo del aula |
|---|---|
| Conoce el modelo histórico de OLVM | Recoge series de los sistemas expuestos |
| Consulta `ovirt_engine_history` | Consulta Prometheus |
| Se integra con Monitoring Portal | Se integra con la observabilidad general del laboratorio |
| Ofrece contexto propio de OLVM | Permite combinar OLVM con Kubernetes y otros sistemas |

Que exista un panel de Grafana no demuestra qué camino se está utilizando. Debemos mirar el origen de datos.

# 6. Qué mirar desde el portal

El portal es el primer punto de orientación, no siempre el último.

## A nivel de Data Center y Cluster

- estado general;
- hosts disponibles;
- redes obligatorias;
- Storage Domains activos;
- uso agregado de CPU y memoria;
- capacidad N+1 y destinos de migración;
- eventos recientes.

## A nivel de host

- estado `Up`, `Maintenance`, `Non Operational` o `Non Responsive`;
- carga de CPU y memoria;
- número y distribución de VMs;
- dispositivos y NICs;
- redes fuera de sincronía;
- acceso al almacenamiento;
- eventos y tareas recientes.

## A nivel de VM

- estado y host actual;
- consumo de CPU, memoria y red;
- Guest Agent y direcciones observadas;
- discos, snapshots y operaciones pendientes;
- eventos de arranque, parada y migración;
- razón de fallo si no existe un host elegible.

## A nivel de Storage Domain

- estado y conectividad;
- espacio total, utilizado y libre;
- operaciones sobre imágenes;
- discos bloqueados;
- latencia y errores del camino NFS fuera de OLVM.

# 7. Eventos, correo y SNMP

Engine registra eventos y estos pueden mostrarse en el portal. El servicio de notificaciones puede seleccionar eventos y enviarlos a sistemas externos.

```text
Componente o acción
        ↓
Evento registrado por Engine
        ↓
Suscripción y filtro
        ↓
ovirt-engine-notifier
        ↓
Correo o trap SNMP
```

El evento y la notificación no son lo mismo. Si no llega un correo debemos comprobar:

1. que el evento existe;
2. que su severidad y código cumplen la suscripción;
3. que `ovirt-engine-notifier` está activo;
4. que la resolución DNS y la ruta al servidor SMTP funcionan;
5. que el servidor SMTP aceptó el mensaje;
6. que el destinatario y sus permisos están correctamente configurados.

SNMP permite entregar traps a una plataforma de gestión, pero no convierte OLVM en un sistema SNMP de sondeo completo. En empresas grandes, los eventos suelen entrar en una plataforma central junto con red, almacenamiento, hardware y aplicaciones.

# 8. Mapa mínimo de logs

| Componente | Log o fuente principal | Cuándo empezar por ahí |
|---|---|---|
| Engine | `/var/log/ovirt-engine/engine.log` | Decisiones, errores del portal, API, base de datos y operaciones |
| Configuración de Engine | `/var/log/ovirt-engine/setup/` | Fallos o decisiones de `engine-setup` |
| VDSM | `/var/log/vdsm/vdsm.log` | Operaciones del host, storage, red, migración y VMs |
| libvirt | journal y logs de libvirt/QEMU | Definición y ejecución del dominio virtual |
| QEMU de una VM | Logs por dominio en el host | Arranque, dispositivos y fallos del proceso de la VM |
| DWH | journal y logs del servicio DWH | Falta de histórico o retraso de ETL |
| PostgreSQL | journal y logs de PostgreSQL | Conexiones, bloqueos, espacio y errores de base |
| Sistema | `journalctl` | Kernel, red, discos, servicios y errores generales |

Método recomendado:

```text
Identificar objeto y hora
          ↓
Localizar evento o tarea
          ↓
Decidir qué componente actuaba
          ↓
Abrir su log en ese intervalo
          ↓
Correlacionar con host, storage, red y guest
```

# 9. `ovirt-log-collector`

`ovirt-log-collector` prepara un archivo de diagnóstico con información de Engine, hosts y bases de datos. Es especialmente útil para entregar evidencias coherentes a soporte.

Flujo básico:

```text
Instalar la herramienta en Engine
             ↓
Seleccionar alcance y opciones
             ↓
Recoger Engine, hosts y base de datos
             ↓
Generar archivo en /tmp/logcollector
             ↓
Revisar datos sensibles antes de compartir
```

Comandos de referencia:

```bash
sudo dnf install ovirt-log-collector
sudo ovirt-log-collector -h
sudo ovirt-log-collector
```

Sin parámetros, la herramienta intenta recopilar un conjunto amplio, incluida información PostgreSQL. La opción `--no-postgresql` permite excluirla cuando corresponda.

El recolector:

- no repara la plataforma;
- no sustituye a localizar la hora del incidente;
- puede generar un archivo grande;
- puede contener nombres, direcciones, configuración y fragmentos sensibles;
- debe ejecutarse con una política clara de custodia y entrega.

# 10. Performance Co-Pilot

Performance Co-Pilot —PCP— proporciona métricas del sistema operativo y permite conservar archivos de rendimiento con `pmlogger`. Resulta útil cuando el portal indica que existe presión, pero necesitamos detalle del host:

- utilización y espera de CPU;
- memoria y swap;
- E/S y latencia de dispositivos;
- interfaces y errores de red;
- comportamiento del kernel.

PCP no conoce por sí solo toda la semántica de OLVM. Su fuerza está en combinar:

```text
Contexto de OLVM
      +
Métricas del sistema
      +
Logs y eventos
      =
Diagnóstico con contexto
```

# 11. PostgreSQL como fuente de diagnóstico

El objetivo no es administrar PostgreSQL en profundidad. Queremos responder cuatro preguntas seguras:

1. ¿Existen las bases esperadas y cuánto ocupan?
2. ¿Cuántas sesiones hay y en qué estado?
3. ¿Hay consultas activas o esperando?
4. ¿Existen bloqueos concedidos o pendientes?

## Tamaño de las bases

```bash
sudo /usr/share/ovirt-engine/dbscripts/engine-psql.sh -c \
"SELECT datname AS db_name,
        pg_size_pretty(pg_database_size(datname)) AS db_usage
   FROM pg_database
  ORDER BY datname;"
```

Cómo interpretar la salida:

| Columna | Significado |
|---|---|
| `db_name` | Nombre de la base, por ejemplo `engine` u `ovirt_engine_history` |
| `db_usage` | Tamaño presentado en unidades legibles |

El tamaño por sí solo no indica un problema. Necesitamos tendencia, espacio libre del filesystem y comparación con la retención prevista.

## Sesiones agrupadas

```bash
sudo /usr/share/ovirt-engine/dbscripts/engine-psql.sh -c \
"SELECT datname, usename, state, count(*) AS sessions
   FROM pg_stat_activity
  GROUP BY datname, usename, state
  ORDER BY datname, usename, state;"
```

Interpretación básica:

- `active`: está ejecutando una consulta;
- `idle`: mantiene la conexión sin ejecutar en ese instante;
- `idle in transaction`: ha dejado una transacción abierta; si persiste merece investigación;
- muchas conexiones no significan automáticamente saturación.

## Actividad no inactiva

```bash
sudo /usr/share/ovirt-engine/dbscripts/engine-psql.sh -c \
"SELECT pid, datname, usename, state,
        wait_event_type, wait_event,
        now() - query_start AS age
   FROM pg_stat_activity
  WHERE state IS DISTINCT FROM 'idle'
  ORDER BY query_start;"
```

Debemos observar:

- cuánto tiempo lleva activa la sesión;
- si espera por un evento;
- si el momento coincide con una operación de OLVM;
- si el patrón se repite.

## Resumen de bloqueos

```bash
sudo /usr/share/ovirt-engine/dbscripts/engine-psql.sh -c \
"SELECT locktype, mode, granted, count(*) AS locks
   FROM pg_locks
  GROUP BY locktype, mode, granted
  ORDER BY locktype, mode, granted;"
```

`granted = false` indica una solicitud pendiente. No debemos cancelar sesiones ni procesos solo por verla: primero hay que identificar la operación, duración, bloqueador e impacto.

## Precauciones

- utilizar consultas de solo lectura;
- no modificar tablas de Engine manualmente;
- no copiar contraseñas de configuración a apuntes o tickets;
- no activar logging SQL detallado de forma indefinida;
- no ejecutar mantenimiento de base durante la clase;
- antes de una operación de mantenimiento real, seguir el procedimiento de la versión y disponer de backup probado.

## Logging PostgreSQL para una ventana de diagnóstico

Antes de cambiar nada podemos consultar la configuración efectiva:

```sql
SHOW log_statement;
SHOW log_min_duration_statement;
SHOW log_duration;
SHOW log_connections;
SHOW log_disconnections;
SHOW log_directory;
```

Los parámetros responden a preguntas diferentes:

| Parámetro | Qué registra |
|---|---|
| `log_statement` | Sentencias según el nivel `none`, `ddl`, `mod` o `all` |
| `log_min_duration_statement` | Sentencias que superan una duración mínima |
| `log_duration` | Duración de cada sentencia completada |
| `log_connections` | Aperturas de conexión |
| `log_disconnections` | Cierres y duración de sesiones |

Para una incidencia de rendimiento suele ser menos agresivo registrar temporalmente consultas lentas que utilizar `log_statement = 'all'`. El procedimiento de producción es:

1. definir la hipótesis y la ventana de captura;
2. comprobar espacio libre, rotación y ubicación del log;
3. registrar los valores actuales;
4. aplicar únicamente los parámetros acordados con el DBA o soporte;
5. reproducir o esperar el problema;
6. recopilar el intervalo relevante;
7. restaurar los valores originales;
8. verificar que el volumen y la carga vuelven a la normalidad.

`log_statement = 'all'` puede registrar cada consulta y datos sensibles. No se habilita por curiosidad ni se deja activo después de la práctica. En el aula solo mostraremos los valores con `SHOW`; no cambiaremos la configuración.

# 12. Qué sería una monitorización útil en producción

Una buena plataforma no alerta por todo. Debe relacionar condición, duración e impacto.

## Capacidad

- CPU y memoria disponibles por host y Cluster;
- capacidad N+1;
- crecimiento de los Storage Domains;
- capacidad de DWH y PostgreSQL;
- número de destinos válidos para migrar o reiniciar VMs.

## Salud

- Engine, VDSM, DWH, PostgreSQL y servicios de consola;
- hosts `Non Responsive` o `Non Operational`;
- Storage Domains inactivos;
- redes fuera de sincronía;
- tareas bloqueadas o fallidas.

## Rendimiento

- contención de CPU;
- presión de memoria, ballooning y swap;
- latencia y throughput del almacenamiento;
- errores, drops y saturación de red;
- tiempos de las operaciones de Engine;
- rendimiento observado dentro del guest.

## Disponibilidad

- VMs críticas y políticas HA;
- fencing configurado y probado;
- capacidad de recuperación;
- entrega real de notificaciones;
- backup de Engine y prueba de restauración.

Una alerta útil debe indicar:

- objeto afectado;
- condición y umbral;
- duración;
- posible impacto;
- procedimiento de actuación;
- equipo responsable.

---

# Bloque 2 · Diseñar la instalación antes de ejecutar comandos

# 13. El examen y la realidad

En la distribución aportada para **1Z0-1170**, instalación y configuración de Engine/host representan un **16 %**. Monitorización se incluye en el bloque de optimización, eventos y logs, que representa un **20 %**. Por eso hoy aparecen datos concretos de versión, puertos y componentes, pero siempre unidos a su función.

El objetivo oficial de instalación se puede resumir en cinco apartados:

1. requisitos de Engine y hosts;
2. requisitos de firewall;
3. configuración de Engine;
4. configuración de Self-Hosted Engine;
5. incorporación de un host KVM.

Para aprobar no basta con recordar `dnf install`. Para operar una plataforma tampoco. Cada apartado representa una decisión:

| Pregunta | Consecuencia |
|---|---|
| ¿Dónde vive Engine? | Disponibilidad y dependencia del plano de control |
| ¿Qué versión está soportada? | Repositorios, kernel, QEMU, libvirt y compatibilidad |
| ¿Cómo resuelve DNS? | Certificados, SSO, hosts y acceso al portal |
| ¿Cómo se sincroniza la hora? | TLS, autenticación, correlación de logs y clúster |
| ¿Qué flujos permite el firewall? | Gestión, migración, consola, storage y carga de imágenes |
| ¿Dónde están las bases y DWH? | Rendimiento, soporte, backup y recuperación |
| ¿Qué storage se usará? | Disponibilidad de VMs y Self-Hosted Engine |
| ¿Cómo se aislará un host fallido? | Fiabilidad de HA |

# 14. Dos topologías de Engine

## Standalone Engine

Engine se instala en un sistema independiente de los hosts que administra. Puede ser un servidor físico o una VM, pero esa VM no debe depender del mismo entorno OLVM que controla.

```text
Sistema externo
      ↓
Engine + DB + DWH
      ↓ gestión
Hosts KVM administrados
      ↓
VMs de negocio
```

Ventajas:

- arranque y recuperación del plano de control fáciles de razonar;
- menor problema de dependencia circular;
- mantenimiento de Engine separado de los hosts administrados;
- adecuado cuando ya existe otra plataforma fiable para alojarlo.

Inconvenientes:

- la disponibilidad de Engine depende de esa infraestructura externa;
- hay que proteger y respaldar ese sistema por separado;
- si es un único servidor físico, puede convertirse en un punto de fallo.

## Self-Hosted Engine

Engine se ejecuta como una VM especial dentro de los hosts OLVM y se despliega con un mecanismo específico de Hosted Engine.

```text
Hosts OLVM
   ↓ ejecutan y protegen
VM del Engine
   ↓ administra
Los mismos hosts OLVM
```

El aparente círculo se resuelve mediante agentes, metadatos y un procedimiento de bootstrap. Los hosts pueden gestionar el arranque de la VM del Engine incluso cuando el propio Engine todavía no está disponible.

Ventajas:

- el plano de control puede aprovechar la HA de varios hosts;
- no requiere una plataforma de virtualización externa para Engine;
- permite mantener el Engine dentro del entorno OLVM.

Inconvenientes:

- el despliegue y la recuperación son más complejos;
- requiere storage compartido disponible desde el inicio;
- el mantenimiento del Engine necesita comprender el modo de mantenimiento global;
- una avería simultánea de hosts, red y storage puede afectar tanto al control como a las cargas.

## La elección no es «físico frente a virtual»

La diferencia real es quién controla el ciclo de vida de la VM de Engine:

| Situación | Tipo |
|---|---|
| Engine físico separado | Standalone |
| Engine VM en VMware y administra OLVM | Standalone |
| Engine VM en KVM manual fuera de OLVM | Standalone |
| Engine VM desplegada mediante Hosted Engine en los hosts administrados | Self-Hosted |
| Engine VM del aula ejecutada en `worker2` | Standalone desde el punto de vista de OLVM |

# 15. Inventario previo a la instalación

Antes de instalar necesitamos una ficha de diseño.

## Plataforma y soporte

- versión de OLVM elegida;
- versión y edición de Oracle Linux para Engine;
- versión y kernel soportados para los hosts;
- compatibilidad del Cluster;
- hardware certificado cuando corresponda;
- repositorios oficiales requeridos;
- características en soporte completo o Technology Preview.

Los números exactos cambian. El hábito correcto es consultar la matriz vigente antes de cada despliegue o ampliación.

## Fotografía de requisitos oficiales —28 de agosto de 2026—

Esta tabla ayuda a estudiar la versión actual. No debe reutilizarse en un proyecto posterior sin volver a comprobarla.

| Rol | Sistema indicado en la guía actual | Matiz |
|---|---|---|
| Standalone Engine | Oracle Linux 8.8 o posterior dentro de 8.x, u Oracle Linux 9.7 o posterior dentro de 9.x | `Minimal Install`; no convertirlo además en host KVM administrado |
| Host KVM | Oracle Linux 8.8 o posterior dentro de 8.x, u Oracle Linux 9.6 o posterior dentro de 9.x | `Minimal Install`; los hosts OL9 requieren compatibilidad de Cluster 4.7 y un nivel mínimo de Engine |
| Host Self-Hosted Engine | Oracle Linux 8.8 o posterior dentro de 8.x, u Oracle Linux 9.6 o posterior dentro de 9.x | los hosts que proporcionan HA a Engine deben ejecutar la misma release de Oracle Linux |

La documentación también establece en este momento:

- para un Engine de tamaño pequeño, mínimo de 2 cores, 4 GB de RAM y 25 GB locales; recomienda 4 cores, 16 GB y 50 GB;
- para un host KVM, mínimo de CPU de 64 bits con dos cores, 2 GB de RAM, una NIC de 1 Gbps y 60 GB locales;
- se recomiendan varias CPU y dos o más NICs en el host;
- el host KVM necesita VT-x o AMD-V y NX habilitados;
- los hosts no deben ejecutar aplicaciones ajenas ni watchdogs de terceros que interfieran con VDSM.

Son umbrales de instalación y soporte, no un dimensionamiento sensato de producción. En un entorno de CaixaBank o Redeia, capacidad, redundancia, seguridad, crecimiento y dominio de fallo pesan mucho más que alcanzar el mínimo.

## Capacidad

- número inicial y futuro de hosts;
- número y tipo de VMs;
- CPU, memoria y disco de Engine;
- crecimiento de las bases y DWH;
- capacidad N+1 del Cluster;
- ancho de banda de gestión, migración, storage y VMs.

El mínimo permite instalar. El recomendado permite operar con margen.

## Identidad y nombres

- FQDN definitivo de Engine;
- FQDN de cada host;
- resolución directa e inversa coherente;
- direcciones IP estables;
- dominio de búsqueda;
- certificados y autoridad de confianza;
- integración con Keycloak, LDAP o Active Directory si se utiliza.

Cambiar el nombre de Engine después no equivale a editar `/etc/hosts`: el FQDN participa en certificados, SSO, URLs y configuración distribuida.

## Tiempo

- fuente NTP o Chrony corporativa;
- zona horaria administrativa;
- sincronización de Engine, hosts, storage y servicios de identidad;
- monitorización del desfase.

Un desfase importante puede producir fallos de autenticación y certificados, además de destruir la correlación temporal de un incidente.

## Redes

- red de gestión;
- redes de migración, storage, display y VMs cuando se separen;
- VLAN, MTU y bonds;
- redundancia de NICs y switches;
- rutas y gateways;
- DNS, NTP, repositorios, SMTP y servicios de identidad;
- acceso al BMC por una red de gestión fuera de banda.

## Almacenamiento

- tipo de Storage Domain y soporte en la versión;
- capacidad y rendimiento;
- redundancia del servidor o cabina;
- conectividad desde todos los hosts;
- permisos y exportaciones NFS si se utiliza NFS;
- storage de la Self-Hosted Engine si corresponde;
- backup de Engine en una ubicación independiente.

# 16. Requisitos conceptuales del host de Engine

El host de Engine necesita:

- instalación mínima y limpia de Oracle Linux soportado;
- CPU, memoria y disco acordes al tamaño del entorno;
- FQDN estable y resoluble;
- tiempo sincronizado;
- repositorios soportados y sin paquetes que introduzcan conflictos;
- conectividad hacia todos los hosts y servicios necesarios;
- firewall preparado;
- backup y capacidad de recuperación.

Regla clave:

> **Un Standalone Engine no se configura también como host KVM administrado.**

Engine ejecuta servicios sensibles: aplicación, base, DWH, SSO, proxies y certificados. Instalar software arbitrario o habilitar repositorios no soportados puede alterar dependencias de PostgreSQL, Java, QEMU o librerías.

# 17. Requisitos conceptuales de un host KVM

Un host necesita:

- Oracle Linux y kernel soportados;
- instalación `Minimal Install`;
- extensiones Intel VT-x o AMD-V habilitadas;
- NX o equivalente;
- CPU de 64 bits y capacidad suficiente;
- memoria para las VMs y reserva para el propio host;
- disco local para sistema, logs, volcados y metadatos;
- NICs y redundancia conforme al diseño;
- acceso a redes, storage, DNS, NTP y Engine;
- repositorios correctos;
- SELinux y firewall en el estado soportado;
- BMC accesible y fencing preparado en producción.

El host debe dedicarse a virtualización. Añadir aplicaciones empresariales ajenas aumenta el consumo, las dependencias y la superficie de fallo.

En el aula los hosts son anidados. Esto ayuda a enseñar OLVM, pero no convierte la virtualización anidada en la arquitectura de producción recomendada.

# 18. DNS, FQDN y certificados

La comprobación básica de Engine es:

```bash
hostname -f
```

Debe devolver el FQDN que se utilizará en `engine-setup`.

Comprobaciones de diseño:

```bash
getent hosts engine.ejemplo.local
getent hosts host01.ejemplo.local
timedatectl
```

El archivo `/etc/hosts` puede resolver un laboratorio pequeño, pero en producción interesa un DNS consistente para todos los participantes.

Errores típicos:

- Engine se identifica con un nombre corto;
- el navegador utiliza un alias no incluido en el certificado;
- Engine resuelve el host a una IP diferente de la que usa el host;
- la resolución inversa devuelve un nombre incoherente;
- se cambia la IP sin revisar certificados, firewall y rutas;
- NTP no está sincronizado.

# 19. Firewall: pensar en flujos

Memorizar una lista sin origen y destino conduce a errores. Una regla de firewall tiene al menos cinco dimensiones:

```text
origen + destino + protocolo + puerto + finalidad
```

## Flujos principales

| Origen | Destino | Puertos representativos | Finalidad |
|---|---|---|---|
| Navegadores y clientes API | Engine | TCP 80/443 | Portal, API y servicios web |
| Engine | Host KVM | TCP 22 | Bootstrap inicial por SSH, si se utiliza |
| Engine y hosts | Host KVM | TCP 54321 | Comunicación con VDSM |
| Host KVM | Host KVM | TCP 16514 | Migración mediante libvirt |
| Host KVM | Host KVM | TCP 49152–49216 | Migración y operaciones VDSM |
| Clientes de consola | Host KVM | TCP 5900–6923 | Consolas VNC/RDP según configuración |
| Clientes | Proxy WebSocket | TCP 6100 | Consolas noVNC/HTML5, si se utiliza |
| Cliente o proxy | Engine/host | TCP 54323/54322 | Carga de imágenes mediante Image I/O Proxy |
| Engine o DWH | PostgreSQL | TCP 5432 | Base remota, si se configura |
| Hosts | Hosts | UDP 6081 | OVN, solo si el proveedor está habilitado |
| Hosts | Storage NFS | Según NFS y versión | Acceso al almacenamiento compartido |

Esta tabla es un mapa de estudio, no una política para copiar sin revisión. Hay flujos opcionales y puertos configurables. NFS, consolas, bases remotas, OVN, SNMP y componentes separados cambian los requisitos.

## Configuración automática

`engine-setup` puede configurar el firewall del host de Engine. Al añadir un host, Engine también puede configurar el firewall del host KVM.

Esto no significa que OLVM configure:

- el firewall de red corporativo;
- las ACL de los switches;
- el firewall de la cabina o servidor NFS;
- las reglas entre zonas de seguridad;
- la red de BMC.

En un entorno empresarial hay que entregar una matriz de flujos antes del cambio.

## Dos precauciones importantes

- La configuración automática puede alterar reglas existentes; debe acordarse con el equipo responsable.
- IPv6 debe permanecer habilitado en el sistema de Engine aunque la red de servicio use IPv4.

# 20. Repositorios y paquetes

Hay tres pasos distintos:

```text
Habilitar repositorios soportados
             ↓
Instalar el paquete release de OLVM
             ↓
Instalar o desplegar los componentes
```

El paquete `oracle-ovirt-release-45-*` configura repositorios y módulos adecuados a la versión. No instala por sí solo un Engine completo.

En Engine, el paquete central es:

```bash
sudo dnf install ovirt-engine
```

En un host, Engine puede instalar VDSM y los paquetes requeridos durante el alta. Antes se prepara Oracle Linux, repositorios, kernel y conectividad.

No debemos copiar nombres de repositorios de otra versión sin comprobar:

- versión mayor de Oracle Linux;
- versión de OLVM;
- kernel requerido;
- repositorios de KVM;
- compatibilidad del Cluster;
- estado de soporte de cada función.

---

# Bloque 3 · Flujo de instalación

# 21. Instalación de Standalone Engine

El proceso conceptual es:

```text
1. Diseñar y documentar
          ↓
2. Instalar Oracle Linux mínimo
          ↓
3. Configurar FQDN, DNS, IP y tiempo
          ↓
4. Habilitar repositorios soportados
          ↓
5. Instalar ovirt-engine
          ↓
6. Ejecutar engine-setup
          ↓
7. Validar servicios, portal y certificados
          ↓
8. Crear o ajustar Data Center y Cluster
          ↓
9. Preparar y añadir hosts
          ↓
10. Configurar redes, storage y fencing
          ↓
11. Probar y obtener backup inicial
```

Los pasos 5 y 6 no son equivalentes:

| Instalación del paquete | `engine-setup` |
|---|---|
| Copia software y dependencias | Toma decisiones de configuración |
| No define el FQDN final | Configura nombres, certificados y URLs |
| No crea toda la plataforma operativa | Configura Engine y sus servicios |
| No decide bases, DWH o Grafana | Crea o conecta bases y componentes |
| No abre necesariamente los flujos | Puede configurar el firewall local |

# 22. Qué decide `engine-setup`

`engine-setup` es un asistente de configuración ejecutado en terminal. Pregunta, valida, resume y después aplica.

Áreas principales:

## Engine

- configurar Engine en este host;
- FQDN definitivo;
- contraseña del administrador;
- certificados y PKI interna;
- puertos y servicios web.

## Base de datos

- base `engine` local automática;
- base local administrada manualmente;
- base remota, si la versión y el soporte lo permiten;
- nombre, usuario, host, puerto y cifrado.

Para una instalación habitual, la opción local automática reduce variables. Separar PostgreSQL puede reducir carga, pero añade red, TLS, backup coordinado, latencia, firewall y un dominio de fallo adicional.

## DWH

- configurar o no DWH en este host;
- base `ovirt_engine_history`;
- escala y retención;
- ubicación local o remota según soporte.

## Grafana

- instalar la integración de monitorización;
- configurar credenciales iniciales e integración de acceso;
- conectar con DWH.

Que la opción exista no garantiza que esté instalada en un entorno automatizado: hay que comprobar los servicios y la configuración reales.

## Identidad

- Keycloak como mecanismo moderno de SSO;
- dominio interno y administrador;
- posterior federación con AD o LDAP si se requiere.

## Consolas

- WebSocket Proxy para noVNC/HTML5;
- proxy de consola serie;
- componentes locales o remotos.

## Red virtual opcional

- proveedor OVN;
- relación con clusters y hosts;
- compatibilidad y estado de soporte de la versión.

OVN no es obligatorio para el networking nativo con bridges que hemos utilizado en el curso.

## Firewall

- detección del gestor de firewall;
- configuración automática o manual;
- resumen de puertos necesarios.

# 23. Qué produce `engine-setup`

Al finalizar correctamente obtenemos:

- servicios de Engine configurados;
- bases creadas o conectadas;
- certificados y CA interna;
- usuario administrador;
- URL del portal;
- configuración de DWH, SSO, proxies y Grafana según respuestas;
- reglas locales de firewall si se autorizó;
- log detallado del proceso;
- archivo de respuestas reutilizable para reconfiguración o automatización.

El log de Setup se encuentra bajo:

```text
/var/log/ovirt-engine/setup/
```

El resumen previo a aplicar es una última oportunidad para detectar:

- FQDN incorrecto;
- bases en el host equivocado;
- componentes opcionales no deseados;
- firewall manual sin matriz preparada;
- escala DWH insuficiente o excesiva.

# 24. Validación inicial de Engine

No basta con que `engine-setup` termine con éxito.

## Sistema

```bash
hostname -f
timedatectl
dnf repolist
```

## Servicios

```bash
systemctl status ovirt-engine
systemctl status postgresql
systemctl status ovirt-engine-dwhd
```

El nombre exacto de la unidad PostgreSQL puede variar según la versión. Solo comprobamos DWH o Grafana si se configuraron.

## Funcional

- abrir el portal mediante el FQDN;
- validar la cadena del certificado desde el cliente;
- iniciar sesión;
- comprobar eventos de instalación;
- confirmar que DWH comienza a recibir datos;
- comprobar el Monitoring Portal si se instaló;
- verificar resolución y acceso a los futuros hosts.

## Operativo

- guardar de forma segura el archivo de respuestas y la documentación;
- registrar las decisiones de instalación;
- realizar un backup inicial de Engine;
- copiar el backup fuera del host;
- definir responsables, monitorización y renovación de certificados.

# 25. Flujo de Self-Hosted Engine

Self-Hosted Engine añade una fase de bootstrap.

```text
1. Preparar varios hosts compatibles
             ↓
2. Preparar red, DNS y storage compartido
             ↓
3. Ejecutar prechecks
             ↓
4. Iniciar despliegue desde un primer host
             ↓
5. Crear la VM de Engine desde el appliance
             ↓
6. Ejecutar la configuración de Engine dentro de la VM
             ↓
7. Registrar el primer host y el storage
             ↓
8. Añadir más hosts Hosted Engine
             ↓
9. Comprobar HA de la VM de Engine
             ↓
10. Realizar backup y prueba operativa
```

Elementos que deben existir antes:

- un mínimo de dos hosts Hosted Engine para disponer de HA; la guía de OLVM 4.5 limita a siete los hosts desplegados con ese rol, aunque pueden existir hosts regulares adicionales;
- FQDN e IP definitivos de la VM de Engine;
- resolución DNS desde hosts y clientes;
- almacenamiento compartido compatible;
- red común y MTU coherente;
- hosts con versiones compatibles;
- recursos reservados para Engine;
- método de fencing y capacidad suficiente.

El rango **de dos a siete hosts Hosted Engine** es relevante para el examen de la versión actual. En una implantación futura se confirma siempre en la guía vigente.

## Appliance u OVA de Engine

El despliegue utiliza una imagen preparada para crear la VM de Engine. No es una VM ordinaria creada manualmente y después marcada como especial.

## Agentes Hosted Engine

Los hosts mantienen metadatos y conocen el estado de la VM del Engine. Si Engine no responde, los hosts pueden coordinar su arranque en un destino válido.

## Mantenimiento global

Antes de determinadas operaciones sobre la VM de Engine se activa el mantenimiento global para impedir que los agentes interpreten la parada planificada como un fallo que deben corregir.

```bash
hosted-engine --set-maintenance --mode=global
hosted-engine --vm-status
hosted-engine --set-maintenance --mode=none
```

Estos comandos son de referencia. No se ejecutan en el aula porque nuestra VM de Engine no es una Self-Hosted Engine.

# 26. Preparación de un host KVM

Proceso previo:

```text
Oracle Linux Minimal Install
            ↓
FQDN, DNS, IP y tiempo
            ↓
Repositorios y paquete release
            ↓
Kernel y módulos requeridos
            ↓
Conectividad y firewall
            ↓
Precheck
            ↓
Alta desde Engine
```

La herramienta de precomprobación ayuda a detectar:

- instalación no mínima;
- repositorios adicionales;
- kernel incorrecto;
- módulos ausentes;
- estado de firewall y SELinux;
- FQDN incorrecto;
- paquetes de host ya instalados de forma inesperada.

Comando de referencia en las versiones actuales:

```bash
sudo olvm-pre-check.py
```

Un `WARN` requiere entender la causa; un `FAIL` debe resolverse. No se trata de ocultar la advertencia hasta que el informe se vea verde.

# 27. Qué ocurre al añadir un host

Desde el portal se seleccionan Data Center y Cluster, y se proporciona:

- nombre del objeto;
- FQDN o IP;
- puerto SSH;
- autenticación por clave pública o contraseña;
- configuración automática del firewall, si se autoriza;
- parámetros opcionales, incluida gestión de energía.

Después, Engine no se limita a insertar una fila:

```text
Engine alcanza el host por SSH
             ↓
Comprueba identidad y requisitos
             ↓
Instala y configura VDSM y dependencias
             ↓
Establece certificados y confianza
             ↓
Configura servicios y firewall autorizado
             ↓
VDSM informa hardware, redes y storage
             ↓
Engine valida compatibilidad con el Cluster
             ↓
Estado Installing pasa a Up
```

Si queda en `Installing`, debemos buscar en este orden:

1. DNS y FQDN;
2. SSH y autenticación;
3. repositorios y resolución de dependencias;
4. tiempo y certificados;
5. firewall;
6. servicios y logs de VDSM;
7. compatibilidad de CPU, kernel y Cluster;
8. redes o storage obligatorios.

Después de que Engine gestione el host, los cambios de red persistentes se realizan desde el Administration Portal o la API. Modificar `nmcli` o archivos por fuera puede crear divergencia entre el estado deseado de Engine y la configuración real.

# 28. Orden de construcción de la plataforma

Una secuencia razonable después de instalar Engine es:

1. revisar Data Center y Cluster;
2. fijar versión de compatibilidad y política de CPU;
3. añadir al menos dos hosts cuando se busca HA;
4. configurar redes de host desde Engine;
5. adjuntar el almacenamiento compartido;
6. configurar gestión de energía y probar fencing;
7. validar migración entre hosts;
8. cargar imágenes o crear VMs de prueba;
9. comprobar DWH, eventos y notificaciones;
10. realizar backup de Engine y documentar restauración.

El Data Center y Cluster `Default` que aparecen tras la instalación ayudan a comenzar. No sustituyen al diseño de nombres, redes, compatibilidad y políticas de producción.

# 29. Instalación automatizada

Ansible puede ejecutar los mismos pasos de forma repetible, pero no elimina las decisiones.

Una automatización correcta debe:

- declarar versiones y repositorios;
- comprobar DNS y tiempo;
- validar variables antes de aplicar;
- proteger contraseñas y claves;
- conservar logs y archivos de respuesta;
- ser idempotente cuando sea posible;
- distinguir preparación del sistema, instalación, configuración y validación;
- detenerse ante un estado inesperado;
- documentar qué queda bajo control de Engine y qué pertenece a la infraestructura externa.

Automatizar un valor incorrecto solo permite equivocarse de manera más consistente.

---

# Bloque 4 · Caso integrado y repaso

# 30. Caso: nueva plataforma para una empresa grande

Requisitos iniciales:

- varios racks y decenas de hosts;
- redes separadas para gestión, migración, storage y VMs;
- Active Directory corporativo;
- almacenamiento compartido redundante;
- plataforma corporativa de monitorización;
- obligación de mantener servicio durante el fallo de un host;
- equipos distintos para sistemas, red, storage y seguridad.

## Decisiones que debemos documentar

### Engine

- Standalone sobre una plataforma externa o Self-Hosted;
- capacidad inicial y crecimiento;
- ubicación de bases y DWH;
- FQDN, certificados y backup;
- dependencia de un único rack o cabina.

### Hosts

- versión y compatibilidad;
- grupos de CPU;
- número de hosts por Cluster;
- capacidad N+1;
- BMC y fencing;
- reserva para el propio hipervisor.

### Red

- VLAN y MTU de cada tráfico;
- bonds y switches redundantes;
- matriz de firewall por flujo;
- rutas hacia DNS, NTP, AD, SMTP, repositorios y monitorización;
- red fuera de banda.

### Storage

- protocolo y dominio;
- capacidad, IOPS, latencia y crecimiento;
- redundancia de red y cabina;
- acceso coherente desde todos los hosts;
- ubicación del Hosted Engine si se elige.

### Observabilidad

- eventos de OLVM hacia la plataforma corporativa;
- métricas nativas y externas;
- retención de DWH;
- paneles de capacidad N+1, storage y migraciones;
- procedimiento para recoger logs;
- sincronización horaria de todas las capas.

# 31. Errores de diseño que debemos detectar

1. Instalar primero y decidir el FQDN después.
2. Usar una VM de Engine alojada en el propio entorno y llamarla Standalone sin resolver su arranque.
3. Dimensionar con mínimos y no reservar crecimiento para DWH.
4. Abrir puertos solo desde Engine hacia hosts y olvidar migración host a host.
5. Configurar una única NIC sin reconocer el dominio de fallo.
6. Dar por hecho que instalar el paquete release instala Engine.
7. Dar por hecho que añadir un host solo lo registra.
8. Modificar después la red del host con `nmcli` fuera de Engine.
9. Configurar HA sin BMC, fencing probado o capacidad N+1.
10. Instalar Grafana sin identificar su origen de datos.
11. Guardar el backup de Engine únicamente en el propio host.
12. Activar logging detallado y olvidarlo habilitado.

# 32. Lo que conviene recordar para el examen

## Monitorización

- `engine` representa estado y configuración operativos.
- `ovirt_engine_history` contiene histórico de DWH.
- DWH realiza ETL desde Engine hacia la base histórica.
- Grafana consulta DWH; no sustituye al Administration Portal.
- `engine.log` y `vdsm.log` pertenecen a planos diferentes.
- Notifier puede entregar eventos por correo y SNMP.
- `ovirt-log-collector` reúne evidencias de Engine, hosts y bases.
- PostgreSQL puede comprobarse mediante consultas de solo lectura.

## Instalación

- Engine y host KVM tienen requisitos distintos.
- utilizar Oracle Linux, kernel, repositorios y compatibilidad soportados.
- `Minimal Install` evita conflictos de paquetes.
- Standalone Engine no se convierte también en host administrado.
- el FQDN, DNS, tiempo y firewall se resuelven antes de instalar.
- `dnf install ovirt-engine` instala software; `engine-setup` configura la plataforma.
- Self-Hosted Engine es una VM especial gestionada mediante Hosted Engine y storage compartido.
- añadir un host despliega y configura VDSM; el estado pasa por `Installing` antes de `Up`.
- las reglas de firewall se entienden por origen, destino, puerto, protocolo y finalidad.

# 33. Cierre

Una plataforma operable necesita dos disciplinas que suelen separarse demasiado:

```text
Instalación bien diseñada
          ↓
Componentes y dependencias conocidos
          ↓
Monitorización con contexto
          ↓
Diagnóstico reproducible
          ↓
Recuperación fiable
```

Si no sabemos cómo se instaló, cuesta interpretar lo que monitorizamos. Si no monitorizamos, no sabemos si las decisiones de instalación siguen siendo válidas.

La idea final del curso es:

> **Engine expresa el estado deseado y coordina; los hosts ejecutan; storage y red transportan; la observabilidad nos permite comprobar que el conjunto sigue cumpliendo su propósito.**
