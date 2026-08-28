# Día 5 · Preguntas de repaso y tipo examen

Estas preguntas acompañan a `dia5_conceptos.md`. Cada pregunta tiene una única respuesta correcta. Además de elegirla, el alumno debe justificar por qué las demás opciones no describen el componente o el flujo indicado.

---

# Monitorización, DWH y Grafana

## 1. ¿Qué diferencia principal existe entre un estado y un evento?

A. El estado describe cómo está el objeto ahora; el evento registra un hecho en un momento determinado.  
B. El estado solo existe en PostgreSQL y el evento solo en el navegador.  
C. Un evento siempre contiene una serie temporal.  
D. No existe ninguna diferencia operativa.

## 2. Una VM está `Up`, pero el usuario afirma que tardó diez minutos en arrancar. ¿Qué enfoque es el más adecuado?

A. Concluir que no hubo ningún problema porque el estado actual es correcto.  
B. Correlacionar eventos, tareas, logs y métricas en el intervalo del arranque.  
C. Reiniciar inmediatamente todos los hosts.  
D. Modificar directamente la base `engine`.

## 3. ¿Qué almacena principalmente la base PostgreSQL `engine`?

A. El estado persistente, la configuración y la información operativa necesaria para administrar OLVM.  
B. Únicamente copias completas de los discos virtuales.  
C. Solo métricas diarias de hace varios años.  
D. Los archivos ISO cargados por los usuarios.

## 4. ¿Cuál es la finalidad principal de `ovirt_engine_history`?

A. Guardar el histórico de configuración y estadísticas preparado por Data Warehouse.  
B. Sustituir los discos de las VMs.  
C. Ejecutar VDSM.  
D. Mantener las reglas del firewall de los hosts.

## 5. ¿Qué describe mejor el trabajo de DWH?

A. Extraer datos de `engine`, transformarlos y cargarlos en `ovirt_engine_history`.  
B. Migrar en vivo las VMs entre hosts.  
C. Convertir una red lógica en una VLAN física.  
D. Realizar fencing del host que no responde.

## 6. El servicio DWH se detiene, pero Engine y los hosts continúan funcionando. ¿Qué efecto es más probable?

A. Todas las VMs se apagan inmediatamente.  
B. Engine puede seguir administrando, pero el histórico deja de actualizarse o queda retrasado.  
C. El Storage Domain se convierte en local.  
D. El host SPM pierde necesariamente la alimentación.

## 7. ¿De qué fuente obtiene los datos la integración nativa de Grafana de OLVM?

A. De la base histórica de Data Warehouse.  
B. Directamente de los discos QCOW2.  
C. Del BMC de cada servidor exclusivamente.  
D. De las reglas de afinidad del scheduler, sin base de datos.

## 8. En el aula existe Grafana, pero no está instalada la integración nativa de OLVM. ¿Cómo es posible?

A. Grafana externo consulta Prometheus, alimentado por métricas expuestas mediante `collectd`.  
B. Toda instalación de VDSM incluye necesariamente Grafana.  
C. NFS genera automáticamente el Monitoring Portal.  
D. QEMU dibuja los paneles en el navegador.

## 9. ¿Qué ventaja proporcionan las agregaciones horarias y diarias de DWH?

A. Permiten conservar tendencias largas con menos detalle y menor coste que todas las muestras finas.  
B. Garantizan que nunca crezca la base histórica.  
C. Sustituyen la sincronización NTP.  
D. Eliminan la necesidad de monitorizar el almacenamiento.

## 10. ¿Cuál es la fuente más adecuada para saber en qué host se ejecuta una VM ahora mismo?

A. El estado actual de Engine.  
B. Una agregación diaria de DWH.  
C. Un backup antiguo de Engine.  
D. El log de instalación de Oracle Linux.

---

# Eventos, logs y diagnóstico

## 11. ¿Dónde buscarías primero el detalle de una operación de storage ordenada a un host que ha fallado?

A. En `vdsm.log` del host implicado, correlacionado con el evento de Engine.  
B. En el historial del navegador del alumno.  
C. En el disco virtual de otra VM.  
D. En el BMC del servidor NFS exclusivamente.

## 12. ¿Qué función cumple `ovirt-engine-notifier`?

A. Procesar suscripciones de eventos y enviar notificaciones, por ejemplo mediante correo o SNMP.  
B. Ejecutar las VMs en lugar de QEMU.  
C. Crear Storage Domains NFS.  
D. Sustituir la base `engine`.

## 13. El evento aparece en Engine, pero no llega el correo. ¿Qué comprobación es la más razonable?

A. Revisar suscripción, filtro, servicio Notifier y conectividad con SMTP.  
B. Borrar el evento de la base.  
C. Reiniciar todas las VMs del Cluster.  
D. Cambiar el formato de sus discos a RAW.

## 14. ¿Para qué se utiliza `ovirt-log-collector`?

A. Para recopilar de forma coordinada información de Engine, hosts y bases y preparar un archivo de diagnóstico.  
B. Para aplicar parches dentro de todos los invitados.  
C. Para convertir Standalone Engine en Self-Hosted Engine.  
D. Para fusionar snapshots.

## 15. ¿Qué hace la opción `--no-postgresql` de `ovirt-log-collector`?

A. Excluye la recogida de información de PostgreSQL.  
B. Desinstala PostgreSQL del Engine.  
C. Crea una base PostgreSQL remota.  
D. Detiene permanentemente DWH.

## 16. ¿Qué aporta Performance Co-Pilot —PCP—?

A. Métricas detalladas del sistema operativo sobre CPU, memoria, disco y red.  
B. Un hipervisor alternativo a KVM.  
C. Una copia activa de la VM de Engine.  
D. La configuración de etiquetas de afinidad.

## 17. En `pg_stat_activity`, ¿qué situación merece investigación si persiste durante mucho tiempo?

A. Una sesión `idle in transaction`.  
B. La existencia de la base `postgres`.  
C. Una conexión cerrada correctamente.  
D. Que el nombre de la base sea `engine`.

## 18. Una consulta a `pg_locks` muestra una fila con `granted = false`. ¿Qué significa?

A. Existe una solicitud de bloqueo pendiente; debe investigarse su duración y bloqueador antes de actuar.  
B. PostgreSQL ha borrado todos los datos.  
C. La VM de Engine está necesariamente apagada.  
D. El host ha pasado a ser SPM.

## 19. ¿Por qué no debe activarse indefinidamente el logging SQL detallado de PostgreSQL?

A. Puede generar carga, mucho volumen y registrar información sensible.  
B. Desactiva la virtualización de CPU.  
C. Convierte las VMs en templates.  
D. Impide utilizar NFSv4 por diseño.

## 20. ¿Cuál es la conducta correcta al consultar la base de Engine para diagnóstico?

A. Utilizar consultas de solo lectura y la API para modificar la plataforma.  
B. Actualizar tablas manualmente cuando la GUI no muestre lo esperado.  
C. Compartir las contraseñas en el ticket.  
D. Eliminar sesiones sin identificar su operación.

---

# Topologías y requisitos de instalación

## 21. Engine se ejecuta como VM en una plataforma externa y administra hosts OLVM. ¿Qué topología describe?

A. Standalone Engine.  
B. Self-Hosted Engine necesariamente.  
C. Un host SPM.  
D. Un VM Pool.

## 22. ¿Qué convierte realmente una VM de Engine en Self-Hosted Engine?

A. Haber sido desplegada mediante Hosted Engine y estar gestionada por sus hosts y agentes específicos.  
B. Tener un disco QCOW2.  
C. Ejecutarse sobre cualquier hipervisor.  
D. Utilizar una dirección IP estática.

## 23. En la guía actual de OLVM 4.5, ¿cuántos hosts Hosted Engine se contemplan para el despliegue?

A. Un mínimo de dos y un máximo de siete hosts con ese rol.  
B. Exactamente un host.  
C. Un mínimo de ocho y sin máximo.  
D. El número de hosts no influye en la alta disponibilidad de Engine.

## 24. ¿Puede un Standalone Engine configurarse además como host KVM administrado por ese Engine?

A. No; deben mantenerse separados esos roles.  
B. Sí, es la configuración recomendada para producción.  
C. Solo si todas las VMs usan RAW.  
D. Solo mientras DWH esté detenido.

## 25. ¿Por qué se recomienda `Minimal Install` para Engine y hosts?

A. Para partir de una base soportada y evitar paquetes o versiones que entren en conflicto con OLVM, QEMU y libvirt.  
B. Porque OLVM no utiliza red.  
C. Porque instala automáticamente todas las VMs.  
D. Porque deshabilita KVM.

## 26. ¿Qué debe comprobarse respecto a la CPU de un host KVM?

A. Que las extensiones de virtualización y NX estén soportadas y habilitadas.  
B. Que tenga el mismo número de serie que Engine.  
C. Que no admita 64 bits.  
D. Que todos los cores estén dedicados a una sola VM.

## 27. ¿Por qué es crítico decidir el FQDN de Engine antes de instalar?

A. Porque participa en certificados, SSO, URLs y configuración distribuida.  
B. Porque determina el formato de los discos virtuales.  
C. Porque selecciona el host SPM.  
D. Porque reemplaza la dirección MAC de las VMs.

## 28. ¿Qué problemas puede causar una hora incoherente entre Engine y hosts?

A. Fallos de certificados o autenticación y dificultad para correlacionar logs.  
B. Conversión automática de NFS en iSCSI.  
C. Creación de una segunda base `engine`.  
D. Cambio de Q35 a i440fx.

## 29. ¿Qué criterio correcto se aplica a los requisitos exactos de versión y repositorio?

A. Verificarlos en la matriz y guía de la versión que se va a desplegar.  
B. Copiarlos siempre de una instalación antigua.  
C. Elegir cualquier repositorio que contenga QEMU.  
D. Habilitar repositorios de desarrollo para obtener paquetes más nuevos.

## 30. ¿Qué expresa mejor la diferencia entre requisito mínimo y recomendado?

A. El mínimo puede permitir instalar; el recomendado aporta margen para carga, crecimiento y operación.  
B. Son siempre idénticos.  
C. El recomendado solo cambia el idioma del portal.  
D. El mínimo garantiza capacidad N+1.

---

# Firewall y Engine Setup

## 31. ¿Qué cinco datos definen correctamente un flujo de firewall?

A. Origen, destino, protocolo, puerto y finalidad.  
B. Nombre de VM, snapshot, template, pool y cuota.  
C. CPU, RAM, disco, BIOS y consola.  
D. Usuario, rol, tag, icono y comentario.

## 32. ¿Qué tráfico es esencial entre hosts para una migración en vivo?

A. Comunicación host a host mediante libvirt/VDSM en los puertos definidos para migración.  
B. Solo HTTPS del navegador a Engine.  
C. Solo SMTP desde Notifier.  
D. Solo PostgreSQL desde DWH.

## 33. ¿Qué servicio utiliza normalmente TCP 54321 en OLVM?

A. VDSM.  
B. PostgreSQL.  
C. NFS.  
D. Grafana.

## 34. ¿Cuándo es necesario permitir el acceso a TCP 5432 a través de la red?

A. Cuando Engine o DWH deben conectarse a un servidor PostgreSQL remoto.  
B. Para cada vNIC de una VM.  
C. Para migrar memoria entre hosts.  
D. Para conectar un bridge Linux a una NIC.

## 35. ¿Qué alcance tiene la configuración automática de firewall de OLVM?

A. Puede configurar el firewall local de Engine o del host, pero no sustituye las ACL y firewalls corporativos externos.  
B. Configura todos los switches y routers de la empresa.  
C. Elimina la necesidad de una matriz de flujos.  
D. Solo cambia el firewall dentro de las VMs.

## 36. ¿Qué diferencia existe entre instalar `ovirt-engine` y ejecutar `engine-setup`?

A. El paquete instala software; `engine-setup` configura servicios, bases, certificados y opciones de la plataforma.  
B. Son exactamente la misma operación.  
C. `engine-setup` solo crea una VM de prueba.  
D. El paquete configura automáticamente todos los hosts KVM.

## 37. ¿Qué decisión se toma durante `engine-setup` respecto a Data Warehouse?

A. Si se configura, dónde reside su base y qué escala o retención utilizar.  
B. Qué VM será SPM.  
C. Qué MAC tendrá cada invitado futuro.  
D. Qué snapshots se fusionarán.

## 38. ¿Qué implica elegir una base PostgreSQL remota para Engine?

A. Añadir dependencias de red, firewall, credenciales, latencia, backup y soporte que deben diseñarse.  
B. Eliminar la necesidad de DNS.  
C. Convertir Engine en un host KVM.  
D. Desactivar DWH automáticamente en todos los casos.

---

# Alta de hosts y validación

## 39. Al añadir un host, su estado aparece `Installing`. ¿Qué está haciendo Engine?

A. Conectando, desplegando VDSM y dependencias, estableciendo confianza e inventariando el host.  
B. Clonando todos los discos de las VMs al host.  
C. Convirtiendo NFS en almacenamiento local.  
D. Ejecutando una segunda copia de Engine.

## 40. Después de añadir un host administrado, ¿dónde deben realizarse los cambios persistentes de su red OLVM?

A. Desde Administration Portal o API, para mantener Engine como fuente de verdad.  
B. Siempre con `nmcli` directamente en el host, sin informar a Engine.  
C. Dentro de una VM cualquiera.  
D. En la base `ovirt_engine_history`.

## 41. Un host queda en `Installing` y Engine no puede autenticar por SSH. ¿Qué debe investigarse primero?

A. Conectividad, puerto y método de autenticación SSH.  
B. La retención diaria de DWH.  
C. Los snapshots de una VM.  
D. La política de afinidad VM–VM.

## 42. Un host tiene VDSM activo, pero queda `Non Operational` porque no dispone de una red requerida. ¿Qué describe mejor la situación?

A. Engine se comunica con el host, pero su configuración no cumple las condiciones del Cluster.  
B. El host está apagado de forma verificada.  
C. PostgreSQL ha perdido todas las bases.  
D. La VM de Engine se ha convertido en Hosted Engine.

## 43. ¿Qué prueba valida mejor que el camino de migración funciona?

A. Migrar de forma controlada una VM de prueba entre hosts compatibles y observar eventos y métricas.  
B. Abrir únicamente la pantalla de login.  
C. Crear un usuario sin permisos.  
D. Consultar solo el tamaño de la base `engine`.

## 44. ¿Qué afirmación describe una aceptación operativa completa?

A. Se validan portal, hosts, storage, redes, consola, migración, monitorización, notificaciones y backup.  
B. Basta con que `engine-setup` termine sin error.  
C. Basta con que el servicio PostgreSQL esté activo.  
D. Basta con crear el Data Center `Default`.

## 45. ¿Dónde debe guardarse el backup inicial de Engine?

A. En una ubicación independiente del propio host o VM de Engine y bajo una política de custodia.  
B. Únicamente en el filesystem local de Engine.  
C. Dentro del disco temporal de una VM de prueba.  
D. En la memoria de un host KVM.

---

# Casos integrados

## 46. Grafana muestra datos hasta las 11:00, pero son las 12:00. Engine administra las VMs con normalidad. ¿Qué componente conviene revisar primero?

A. El servicio DWH y su carga hacia `ovirt_engine_history`.  
B. El watchdog de todas las VMs.  
C. El bridge de una única vNIC.  
D. La plantilla Blank.

## 47. Se diseña una Self-Hosted Engine con un único host y storage local. ¿Cuál es el problema principal?

A. No proporciona la arquitectura compartida y redundante necesaria para proteger la VM de Engine entre hosts.  
B. El nombre de la VM contiene la palabra Engine.  
C. Las VMs no pueden usar VirtIO.  
D. Grafana requiere siempre un disco RAW.

## 48. Un equipo abre 443 desde los administradores a Engine y afirma que el firewall está completo. ¿Qué falta en el razonamiento?

A. Considerar los demás flujos: Engine–host, host–host, consolas, storage y componentes opcionales.  
B. Cambiar todos los discos a QCOW2.  
C. Crear una etiqueta de afinidad.  
D. Deshabilitar IPv6 en Engine.

## 49. La base `ovirt_engine_history` crece rápidamente. ¿Cuál es la primera actuación correcta?

A. Revisar tendencia, retención DWH, tamaño del entorno y espacio disponible antes de aplicar mantenimiento.  
B. Borrar filas manualmente al azar.  
C. Detener permanentemente Engine.  
D. Desconectar el Storage Domain NFS.

## 50. Una plataforma se instala correctamente, pero no se ha probado fencing ni el backup. ¿Puede darse por aceptada para HA?

A. No; existen componentes instalados, pero faltan pruebas esenciales de aislamiento y recuperación.  
B. Sí, porque el portal abre.  
C. Sí, porque DWH recoge muestras.  
D. Sí, siempre que todas las VMs estén en un único host.
