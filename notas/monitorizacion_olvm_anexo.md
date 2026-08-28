# Anexo rápido · Monitorización de OLVM y del laboratorio

Este fue el resumen preparado inicialmente como anexo del día 4. El desarrollo formal, las prácticas y la explicación de instalación se encuentran ahora en `dia5_conceptos.md` y `dia5_notas_clase.md`. Se conserva este fichero únicamente como hoja rápida de consulta.

---

# Evento, log y métrica

| Elemento | Pregunta que responde | Ejemplo |
|---|---|---|
| Evento | ¿Qué hecho observó OLVM? | El Storage Domain pasó a inactivo |
| Log | ¿Qué ocurrió dentro del componente? | VDSM registró un timeout NFS |
| Métrica | ¿Cuánto y durante cuánto tiempo? | La latencia subió durante diez minutos |

Ninguno sustituye a los demás. Una métrica permite identificar una tendencia; un evento sitúa el impacto en el inventario; un log aporta el detalle técnico.

# Estado actual frente a histórico

Engine conserva el estado operativo y los eventos necesarios para administrar el entorno. El Data Warehouse —DWH— recopila muestras y las consolida para análisis histórico.

| Necesidad | Fuente adecuada |
|---|---|
| Saber dónde corre una VM ahora | Engine |
| Ver un error ocurrido hace unos minutos | Eventos y logs |
| Observar la CPU durante varios días | DWH o plataforma externa |
| Detectar el crecimiento del storage | Métricas históricas |
| Entender la causa de un arranque fallido | Logs correlacionados |

# DWH y Grafana nativos

El Data Warehouse utiliza habitualmente la base `ovirt_engine_history`. Recoge muestras periódicas y genera agregaciones horarias y diarias, lo que permite conservar tendencia sin mantener indefinidamente cada muestra con el máximo detalle.

OLVM puede integrar Grafana con DWH. Esa integración es diferente del portal de administración y también de cualquier Grafana externo que utilice la empresa.

# Performance Co-Pilot

Performance Co-Pilot —PCP— observa métricas del sistema operativo y sus componentes. Ayuda a estudiar presión de CPU, memoria, disco y red cuando la vista de inventario de OLVM no es suficiente.

La observabilidad completa suele combinar:

- OLVM para el contexto de virtualización;
- métricas del host para capacidad física;
- métricas del storage y la red para los caminos de datos;
- métricas del guest y la aplicación para el servicio final.

# Cómo está montado el laboratorio

En el aula existen dos caminos distintos.

## Camino nativo de OLVM

- DWH está instalado y habilitado.
- La integración nativa de Grafana de OLVM no está instalada.

## Camino externo

```text
Engine y hosts OLVM
       ↓ collectd
     puerto 9104
       ↓ consulta periódica
Prometheus en Kubernetes
       ↓
Grafana externo
```

Por tanto, que existan paneles en Grafana no demuestra que esté instalado el Grafana nativo de OLVM.

| Componente | Función en el aula |
|---|---|
| DWH | Histórico propio de OLVM |
| collectd | Recogida y exposición de métricas del sistema |
| Prometheus | Consulta y almacenamiento de series temporales |
| Grafana externo | Visualización |
| Eventos de OLVM | Hechos operativos y errores |
| Logs | Evidencia técnica detallada |

# Notificaciones del laboratorio

`ovirt-engine-notifier` consulta los eventos de Engine y aplica las suscripciones configuradas. En nuestra instalación entrega por SMTP a Mailpit los eventos que cumplen el filtro de severidad establecido.

```text
Componente
    ↓
evento en Engine
    ↓
ovirt-engine-notifier
    ↓ SMTP
Mailpit
```

Si un correo no llega, hay que comprobar el flujo completo:

1. ¿Existe el evento en Engine?
2. ¿Cumple el filtro de la suscripción?
3. ¿Está funcionando Notifier?
4. ¿Puede Notifier alcanzar el servidor SMTP?
5. ¿Aceptó Mailpit el mensaje?

La ausencia de correo no demuestra que el evento no se haya producido.

# Qué conviene vigilar

## Capacidad

- CPU y memoria por host y Cluster;
- sobreasignación de recursos;
- espacio libre y crecimiento de Storage Domains;
- número de destinos válidos para migración;
- margen N+1.

## Salud

- Engine, DWH y VDSM;
- hosts `Non Operational` o `Non Responsive`;
- Storage Domains inactivos;
- redes fuera de sincronía;
- tareas fallidas o bloqueadas.

## Rendimiento

- contención de CPU;
- presión de memoria, ballooning y swap;
- latencia y throughput de almacenamiento;
- errores, drops y saturación de red;
- rendimiento del guest y de la aplicación.

## Disponibilidad

- VMs críticas marcadas como HA;
- fencing configurado y probado;
- leases donde sean necesarios;
- capacidad de reinicio en otro host;
- alertas entregadas y atendidas.

# Una métrica necesita contexto

CPU al 95 % puede ser normal durante un procesamiento intensivo. CPU al 30 % puede ser mala si un único hilo limita la aplicación. Cero errores en OLVM tampoco demuestra que la aplicación responda.

Una alerta útil debe incluir:

- objeto afectado;
- condición y umbral;
- duración;
- impacto esperado;
- procedimiento de actuación;
- responsable.

# Práctica opcional

1. Elegir un host del aula.
2. Ver su utilización actual en OLVM.
3. Localizar el mismo host en el Grafana externo.
4. Comparar instante, intervalo y unidad.
5. Abrir los eventos recientes del host.
6. Explicar qué aporta la gráfica y qué aporta el evento.
7. Revisar el estado de DWH sin modificarlo.
8. Revisar el estado de `ovirt-engine-notifier`.
9. Localizar en Mailpit una notificación de prueba ya existente, si la hubiera.
