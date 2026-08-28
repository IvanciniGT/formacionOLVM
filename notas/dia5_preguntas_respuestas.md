# Día 5 · Preguntas y respuestas razonadas

Estas preguntas acompañan a `dia5_conceptos.md`. Cada una tiene una única respuesta correcta. Las explicaciones relacionan la opción con el funcionamiento real de OLVM y señalan por qué las alternativas mezclan componentes o responsabilidades.

---

# Monitorización, DWH y Grafana

## 1. ¿Qué diferencia principal existe entre un estado y un evento?

A. El estado describe cómo está el objeto ahora; el evento registra un hecho en un momento determinado.  
B. El estado solo existe en PostgreSQL y el evento solo en el navegador.  
C. Un evento siempre contiene una serie temporal.  
D. No existe ninguna diferencia operativa.

**Respuesta correcta: A.**

El estado es la fotografía que Engine mantiene del objeto, por ejemplo un host `Up` o una VM `Down`. Un evento conserva un hecho fechado, como el comienzo de una migración o el fallo al activar un disco. Una métrica es la que aporta una serie temporal; estados y eventos pueden consultarse a través de varias interfaces y no pertenecen exclusivamente a una pantalla.

## 2. Una VM está `Up`, pero el usuario afirma que tardó diez minutos en arrancar. ¿Qué enfoque es el más adecuado?

A. Concluir que no hubo ningún problema porque el estado actual es correcto.  
B. Correlacionar eventos, tareas, logs y métricas en el intervalo del arranque.  
C. Reiniciar inmediatamente todos los hosts.  
D. Modificar directamente la base `engine`.

**Respuesta correcta: B.**

El estado `Up` solo confirma el resultado final. Para saber dónde se consumieron los diez minutos hay que reconstruir la secuencia: decisión de Engine, activación del disco, ejecución en VDSM, acceso a storage y arranque del guest. Reiniciar o modificar la base sin diagnóstico añade riesgo y puede eliminar evidencia.

## 3. ¿Qué almacena principalmente la base PostgreSQL `engine`?

A. El estado persistente, la configuración y la información operativa necesaria para administrar OLVM.  
B. Únicamente copias completas de los discos virtuales.  
C. Solo métricas diarias de hace varios años.  
D. Los archivos ISO cargados por los usuarios.

**Respuesta correcta: A.**

La base `engine` representa el estado persistente del plano de control: inventario, relaciones, configuración, permisos, eventos y datos operativos. Los discos e imágenes se guardan en Storage Domains, no dentro de PostgreSQL. El histórico de largo plazo corresponde a DWH y `ovirt_engine_history`.

## 4. ¿Cuál es la finalidad principal de `ovirt_engine_history`?

A. Guardar el histórico de configuración y estadísticas preparado por Data Warehouse.  
B. Sustituir los discos de las VMs.  
C. Ejecutar VDSM.  
D. Mantener las reglas del firewall de los hosts.

**Respuesta correcta: A.**

`ovirt_engine_history` está diseñada para análisis histórico e informes. DWH carga allí cambios de entidades y muestras estadísticas, que después pueden consolidarse. VDSM se ejecuta en cada host y el firewall pertenece al sistema operativo; ninguno de ellos utiliza la base histórica como sustituto de su función.

## 5. ¿Qué describe mejor el trabajo de DWH?

A. Extraer datos de `engine`, transformarlos y cargarlos en `ovirt_engine_history`.  
B. Migrar en vivo las VMs entre hosts.  
C. Convertir una red lógica en una VLAN física.  
D. Realizar fencing del host que no responde.

**Respuesta correcta: A.**

DWH ejecuta un proceso ETL: lee información de la base operativa, la adapta al modelo histórico y la inserta en la base de historia. La migración corresponde a Engine, VDSM, libvirt y los hosts; networking y fencing utilizan otros componentes. El error típico es atribuir a DWH cualquier operación que implique movimiento de datos.

## 6. El servicio DWH se detiene, pero Engine y los hosts continúan funcionando. ¿Qué efecto es más probable?

A. Todas las VMs se apagan inmediatamente.  
B. Engine puede seguir administrando, pero el histórico deja de actualizarse o queda retrasado.  
C. El Storage Domain se convierte en local.  
D. El host SPM pierde necesariamente la alimentación.

**Respuesta correcta: B.**

DWH pertenece al camino de observabilidad histórica, no al proceso QEMU que ejecuta las VMs. Su caída no obliga a detener cargas, pero interrumpe o retrasa el ETL; Grafana puede mostrar huecos o datos antiguos. Storage y SPM no cambian de naturaleza por este fallo.

## 7. ¿De qué fuente obtiene los datos la integración nativa de Grafana de OLVM?

A. De la base histórica de Data Warehouse.  
B. Directamente de los discos QCOW2.  
C. Del BMC de cada servidor exclusivamente.  
D. De las reglas de afinidad del scheduler, sin base de datos.

**Respuesta correcta: A.**

Los dashboards nativos consultan la información preparada en `ovirt_engine_history`. Grafana es una capa de visualización y necesita un datasource; no interpreta discos virtuales ni utiliza el BMC como fuente exclusiva. Las decisiones del scheduler pueden reflejarse en eventos y estados, pero no constituyen por sí solas el datasource de Grafana.

## 8. En el aula existe Grafana, pero no está instalada la integración nativa de OLVM. ¿Cómo es posible?

A. Grafana externo consulta Prometheus, alimentado por métricas expuestas mediante `collectd`.  
B. Toda instalación de VDSM incluye necesariamente Grafana.  
C. NFS genera automáticamente el Monitoring Portal.  
D. QEMU dibuja los paneles en el navegador.

**Respuesta correcta: A.**

Grafana es una herramienta general y puede consultar muchas fuentes. En el aula, el camino externo usa `collectd`, Prometheus y Grafana; por eso existen paneles aunque la integración nativa con DWH no esté instalada. Para interpretar un panel siempre hay que identificar su datasource.

## 9. ¿Qué ventaja proporcionan las agregaciones horarias y diarias de DWH?

A. Permiten conservar tendencias largas con menos detalle y menor coste que todas las muestras finas.  
B. Garantizan que nunca crezca la base histórica.  
C. Sustituyen la sincronización NTP.  
D. Eliminan la necesidad de monitorizar el almacenamiento.

**Respuesta correcta: A.**

La consolidación reduce resolución temporal para ampliar la retención de forma razonable. La base seguirá creciendo y necesita capacidad y mantenimiento; las muestras agregadas tampoco sustituyen el reloj coherente ni la observación del storage. Es un compromiso entre detalle, periodo y coste.

## 10. ¿Cuál es la fuente más adecuada para saber en qué host se ejecuta una VM ahora mismo?

A. El estado actual de Engine.  
B. Una agregación diaria de DWH.  
C. Un backup antiguo de Engine.  
D. El log de instalación de Oracle Linux.

**Respuesta correcta: A.**

Engine mantiene el inventario y la colocación operativa vigente. DWH sirve para histórico y puede tener retraso o datos consolidados; un backup representa un instante anterior. El log de instalación del sistema no conoce la colocación actual de la VM.

---

# Eventos, logs y diagnóstico

## 11. ¿Dónde buscarías primero el detalle de una operación de storage ordenada a un host que ha fallado?

A. En `vdsm.log` del host implicado, correlacionado con el evento de Engine.  
B. En el historial del navegador del alumno.  
C. En el disco virtual de otra VM.  
D. En el BMC del servidor NFS exclusivamente.

**Respuesta correcta: A.**

Engine registra la intención y el resultado general; VDSM ejecuta en el host la operación sobre storage y deja el detalle técnico. El BMC informa sobre hardware y energía, no sobre la llamada de VDSM. Después puede ser necesario ampliar la investigación a los logs del servidor NFS, pero primero se identifica el host, la operación y la hora.

## 12. ¿Qué función cumple `ovirt-engine-notifier`?

A. Procesar suscripciones de eventos y enviar notificaciones, por ejemplo mediante correo o SNMP.  
B. Ejecutar las VMs en lugar de QEMU.  
C. Crear Storage Domains NFS.  
D. Sustituir la base `engine`.

**Respuesta correcta: A.**

Notifier toma los eventos que cumplen las suscripciones y los entrega por los canales configurados. Si Notifier falla, el evento puede seguir existiendo en Engine aunque no llegue el mensaje. QEMU, VDSM y los servicios de storage conservan sus propias responsabilidades.

## 13. El evento aparece en Engine, pero no llega el correo. ¿Qué comprobación es la más razonable?

A. Revisar suscripción, filtro, servicio Notifier y conectividad con SMTP.  
B. Borrar el evento de la base.  
C. Reiniciar todas las VMs del Cluster.  
D. Cambiar el formato de sus discos a RAW.

**Respuesta correcta: A.**

La existencia del evento demuestra que el primer tramo funciona. La investigación continúa por la suscripción, severidad, servicio, resolución y conexión SMTP, aceptación del servidor y destinatario. Discos, VMs y formato RAW no intervienen en este flujo de entrega.

## 14. ¿Para qué se utiliza `ovirt-log-collector`?

A. Para recopilar de forma coordinada información de Engine, hosts y bases y preparar un archivo de diagnóstico.  
B. Para aplicar parches dentro de todos los invitados.  
C. Para convertir Standalone Engine en Self-Hosted Engine.  
D. Para fusionar snapshots.

**Respuesta correcta: A.**

El recolector prepara un paquete coherente de evidencias que puede acompañar un caso de soporte. No ejecuta cambios del ciclo de vida, no instala agentes en invitados y no transforma la topología de Engine. Antes de compartir el archivo hay que revisar su tamaño, alcance y sensibilidad.

## 15. ¿Qué hace la opción `--no-postgresql` de `ovirt-log-collector`?

A. Excluye la recogida de información de PostgreSQL.  
B. Desinstala PostgreSQL del Engine.  
C. Crea una base PostgreSQL remota.  
D. Detiene permanentemente DWH.

**Respuesta correcta: A.**

La opción limita el alcance del archivo de diagnóstico. No cambia la instalación ni el estado de la base. Puede utilizarse por tamaño, privacidad o porque el problema no requiere esa información, aunque el alcance final debe acordarse con soporte.

## 16. ¿Qué aporta Performance Co-Pilot —PCP—?

A. Métricas detalladas del sistema operativo sobre CPU, memoria, disco y red.  
B. Un hipervisor alternativo a KVM.  
C. Una copia activa de la VM de Engine.  
D. La configuración de etiquetas de afinidad.

**Respuesta correcta: A.**

PCP permite investigar el comportamiento del sistema cuando las vistas de OLVM no ofrecen suficiente detalle. No conoce por sí solo toda la semántica de la plataforma, por lo que se combina con objetos, eventos y logs de OLVM. KVM sigue siendo el hipervisor y la afinidad sigue siendo una función del scheduler.

## 17. En `pg_stat_activity`, ¿qué situación merece investigación si persiste durante mucho tiempo?

A. Una sesión `idle in transaction`.  
B. La existencia de la base `postgres`.  
C. Una conexión cerrada correctamente.  
D. Que el nombre de la base sea `engine`.

**Respuesta correcta: A.**

`idle in transaction` significa que la sesión inició una transacción y permanece inactiva sin cerrarla. Si persiste puede retener recursos o bloqueos y debe correlacionarse con la aplicación y la operación. La existencia de bases normales y nombres esperados no representa un fallo.

## 18. Una consulta a `pg_locks` muestra una fila con `granted = false`. ¿Qué significa?

A. Existe una solicitud de bloqueo pendiente; debe investigarse su duración y bloqueador antes de actuar.  
B. PostgreSQL ha borrado todos los datos.  
C. La VM de Engine está necesariamente apagada.  
D. El host ha pasado a ser SPM.

**Respuesta correcta: A.**

PostgreSQL todavía no ha concedido ese lock. El dato no indica por sí solo una avería: puede ser una espera breve y normal. Antes de cancelar nada se identifica quién espera, qué sesión bloquea, desde cuándo, qué consulta ejecuta y qué operación OLVM representa.

## 19. ¿Por qué no debe activarse indefinidamente el logging SQL detallado de PostgreSQL?

A. Puede generar carga, mucho volumen y registrar información sensible.  
B. Desactiva la virtualización de CPU.  
C. Convierte las VMs en templates.  
D. Impide utilizar NFSv4 por diseño.

**Respuesta correcta: A.**

El logging detallado se activa para una ventana de diagnóstico controlada y se revierte al terminar. Puede llenar el filesystem, aumentar E/S y registrar consultas o parámetros sensibles. No modifica KVM, templates ni el protocolo NFS.

## 20. ¿Cuál es la conducta correcta al consultar la base de Engine para diagnóstico?

A. Utilizar consultas de solo lectura y la API para modificar la plataforma.  
B. Actualizar tablas manualmente cuando la GUI no muestre lo esperado.  
C. Compartir las contraseñas en el ticket.  
D. Eliminar sesiones sin identificar su operación.

**Respuesta correcta: A.**

Las tablas internas no constituyen una interfaz administrativa soportada. Las modificaciones se realizan mediante portal o API para que Engine aplique validaciones y mantenga coherencia. Las credenciales se protegen y cualquier acción sobre sesiones necesita identificar primero impacto y causa.

---

# Topologías y requisitos de instalación

## 21. Engine se ejecuta como VM en una plataforma externa y administra hosts OLVM. ¿Qué topología describe?

A. Standalone Engine.  
B. Self-Hosted Engine necesariamente.  
C. Un host SPM.  
D. Un VM Pool.

**Respuesta correcta: A.**

Standalone describe la independencia del ciclo de vida de Engine respecto de los hosts OLVM administrados, no si el sistema es físico o virtual. La plataforma externa arranca y protege esa VM. SPM coordina operaciones de storage y un VM Pool agrupa VMs para usuarios; no son topologías de Engine.

## 22. ¿Qué convierte realmente una VM de Engine en Self-Hosted Engine?

A. Haber sido desplegada mediante Hosted Engine y estar gestionada por sus hosts y agentes específicos.  
B. Tener un disco QCOW2.  
C. Ejecutarse sobre cualquier hipervisor.  
D. Utilizar una dirección IP estática.

**Respuesta correcta: A.**

Self-Hosted Engine incluye metadatos, agentes, storage y un procedimiento de bootstrap que permite a los hosts controlar la VM especial de Engine. QCOW2 o una IP estable pueden aparecer en muchas VMs ordinarias y no crean ese mecanismo. La diferencia está en quién gestiona su ciclo de vida.

## 23. En la guía actual de OLVM 4.5, ¿cuántos hosts Hosted Engine se contemplan para el despliegue?

A. Un mínimo de dos y un máximo de siete hosts con ese rol.  
B. Exactamente un host.  
C. Un mínimo de ocho y sin máximo.  
D. El número de hosts no influye en la alta disponibilidad de Engine.

**Respuesta correcta: A.**

Se necesitan al menos dos hosts para que la VM de Engine disponga de otro destino. La guía de OLVM 4.5 establece un máximo de siete hosts desplegados con el rol Hosted Engine; además pueden incorporarse hosts regulares que ejecuten VMs pero no la VM de Engine. Como es un límite de versión, debe comprobarse de nuevo en una implantación futura.

## 24. ¿Puede un Standalone Engine configurarse además como host KVM administrado por ese Engine?

A. No; deben mantenerse separados esos roles.  
B. Sí, es la configuración recomendada para producción.  
C. Solo si todas las VMs usan RAW.  
D. Solo mientras DWH esté detenido.

**Respuesta correcta: A.**

La guía separa el host de Standalone Engine y los hosts KVM administrados. Mezclarlos introduce conflictos de paquetes, dependencias y ciclo de vida. El formato de discos o el estado de DWH no corrige esa incompatibilidad de roles.

## 25. ¿Por qué se recomienda `Minimal Install` para Engine y hosts?

A. Para partir de una base soportada y evitar paquetes o versiones que entren en conflicto con OLVM, QEMU y libvirt.  
B. Porque OLVM no utiliza red.  
C. Porque instala automáticamente todas las VMs.  
D. Porque deshabilita KVM.

**Respuesta correcta: A.**

OLVM necesita versiones coordinadas de sus componentes. Entornos gráficos, repositorios extra o software añadido antes pueden introducir dependencias incompatibles. Minimal Install no elimina la red ni KVM; crea una base controlada sobre la que se despliegan los paquetes soportados.

## 26. ¿Qué debe comprobarse respecto a la CPU de un host KVM?

A. Que las extensiones de virtualización y NX estén soportadas y habilitadas.  
B. Que tenga el mismo número de serie que Engine.  
C. Que no admita 64 bits.  
D. Que todos los cores estén dedicados a una sola VM.

**Respuesta correcta: A.**

Intel VT-x o AMD-V permiten la virtualización asistida por hardware y deben estar habilitadas en firmware; NX aporta la protección requerida. Además se verifican familia y compatibilidad del Cluster. Ni el número de serie de Engine ni dedicar toda la CPU a una VM son requisitos generales.

## 27. ¿Por qué es crítico decidir el FQDN de Engine antes de instalar?

A. Porque participa en certificados, SSO, URLs y configuración distribuida.  
B. Porque determina el formato de los discos virtuales.  
C. Porque selecciona el host SPM.  
D. Porque reemplaza la dirección MAC de las VMs.

**Respuesta correcta: A.**

`engine-setup` incorpora el FQDN en certificados y referencias utilizadas por clientes y componentes. Cambiarlo después requiere un procedimiento específico, no una simple edición local. Discos, SPM y MAC se gestionan mediante mecanismos diferentes.

## 28. ¿Qué problemas puede causar una hora incoherente entre Engine y hosts?

A. Fallos de certificados o autenticación y dificultad para correlacionar logs.  
B. Conversión automática de NFS en iSCSI.  
C. Creación de una segunda base `engine`.  
D. Cambio de Q35 a i440fx.

**Respuesta correcta: A.**

TLS y los tokens de autenticación tienen periodos de validez. Además, un incidente distribuido solo puede reconstruirse si los timestamps son comparables. El reloj no cambia protocolos de storage, bases o chipset virtual.

## 29. ¿Qué criterio correcto se aplica a los requisitos exactos de versión y repositorio?

A. Verificarlos en la matriz y guía de la versión que se va a desplegar.  
B. Copiarlos siempre de una instalación antigua.  
C. Elegir cualquier repositorio que contenga QEMU.  
D. Habilitar repositorios de desarrollo para obtener paquetes más nuevos.

**Respuesta correcta: A.**

Oracle actualiza soporte de sistemas, kernels, repositorios y compatibilidad. Mezclar instrucciones de otra versión puede instalar un conjunto no probado. Los repositorios de desarrollo o paquetes más nuevos no equivalen a una plataforma soportada y pueden romper dependencias.

## 30. ¿Qué expresa mejor la diferencia entre requisito mínimo y recomendado?

A. El mínimo puede permitir instalar; el recomendado aporta margen para carga, crecimiento y operación.  
B. Son siempre idénticos.  
C. El recomendado solo cambia el idioma del portal.  
D. El mínimo garantiza capacidad N+1.

**Respuesta correcta: A.**

Los mínimos no representan necesariamente una producción con DWH, concurrencia, históricos y crecimiento. Dimensionar también exige considerar indisponibilidad de un host y cargas futuras. N+1 se demuestra con capacidad y restricciones del conjunto, no por cumplir el mínimo de Engine.

---

# Firewall y Engine Setup

## 31. ¿Qué cinco datos definen correctamente un flujo de firewall?

A. Origen, destino, protocolo, puerto y finalidad.  
B. Nombre de VM, snapshot, template, pool y cuota.  
C. CPU, RAM, disco, BIOS y consola.  
D. Usuario, rol, tag, icono y comentario.

**Respuesta correcta: A.**

El mismo puerto puede tener sentidos diferentes según origen y destino. Documentar finalidad permite revisar si el flujo sigue siendo necesario. Las demás opciones enumeran objetos reales, pero no expresan una conversación de red.

## 32. ¿Qué tráfico es esencial entre hosts para una migración en vivo?

A. Comunicación host a host mediante libvirt/VDSM en los puertos definidos para migración.  
B. Solo HTTPS del navegador a Engine.  
C. Solo SMTP desde Notifier.  
D. Solo PostgreSQL desde DWH.

**Respuesta correcta: A.**

Engine coordina la decisión, pero la transferencia del estado ocurre entre los hosts. Por eso una política que permite únicamente cliente–Engine y Engine–host puede administrar sin permitir migrar. SMTP y PostgreSQL pertenecen a notificaciones y bases, no al plano de migración.

## 33. ¿Qué servicio utiliza normalmente TCP 54321 en OLVM?

A. VDSM.  
B. PostgreSQL.  
C. NFS.  
D. Grafana.

**Respuesta correcta: A.**

54321 corresponde a la comunicación de VDSM en los hosts. PostgreSQL utiliza habitualmente 5432, una cifra parecida que el examen puede utilizar para confundir. NFS y Grafana tienen flujos diferentes y dependientes de su despliegue.

## 34. ¿Cuándo es necesario permitir el acceso a TCP 5432 a través de la red?

A. Cuando Engine o DWH deben conectarse a un servidor PostgreSQL remoto.  
B. Para cada vNIC de una VM.  
C. Para migrar memoria entre hosts.  
D. Para conectar un bridge Linux a una NIC.

**Respuesta correcta: A.**

Una base local puede utilizar conexiones locales sin exponer PostgreSQL a otras zonas. Si se separa, hay que permitir el flujo exacto desde Engine o DWH hacia el servidor, además de diseñar cifrado y credenciales. vNIC, migración y bridge no utilizan PostgreSQL.

## 35. ¿Qué alcance tiene la configuración automática de firewall de OLVM?

A. Puede configurar el firewall local de Engine o del host, pero no sustituye las ACL y firewalls corporativos externos.  
B. Configura todos los switches y routers de la empresa.  
C. Elimina la necesidad de una matriz de flujos.  
D. Solo cambia el firewall dentro de las VMs.

**Respuesta correcta: A.**

`engine-setup` y el alta de hosts pueden gestionar `firewalld` local si se autoriza. Los firewalls entre redes, ACL, cabinas, switches y BMC siguen perteneciendo a sus equipos y deben configurarse mediante el proceso corporativo. La automatización local tampoco elimina la necesidad de documentar flujos.

## 36. ¿Qué diferencia existe entre instalar `ovirt-engine` y ejecutar `engine-setup`?

A. El paquete instala software; `engine-setup` configura servicios, bases, certificados y opciones de la plataforma.  
B. Son exactamente la misma operación.  
C. `engine-setup` solo crea una VM de prueba.  
D. El paquete configura automáticamente todos los hosts KVM.

**Respuesta correcta: A.**

La instalación coloca binarios y dependencias. Setup pregunta por FQDN, bases, DWH, identidad, proxies, Grafana, firewall y otras opciones, revisa el resumen y aplica la configuración. Los hosts se incorporan posteriormente mediante su propio flujo.

## 37. ¿Qué decisión se toma durante `engine-setup` respecto a Data Warehouse?

A. Si se configura, dónde reside su base y qué escala o retención utilizar.  
B. Qué VM será SPM.  
C. Qué MAC tendrá cada invitado futuro.  
D. Qué snapshots se fusionarán.

**Respuesta correcta: A.**

La configuración de DWH determina el servicio histórico, la base `ovirt_engine_history` y la cantidad de información conservada. SPM se elige entre hosts para storage y las MAC o snapshots se gestionan en el ciclo de vida de las VMs.

## 38. ¿Qué implica elegir una base PostgreSQL remota para Engine?

A. Añadir dependencias de red, firewall, credenciales, latencia, backup y soporte que deben diseñarse.  
B. Eliminar la necesidad de DNS.  
C. Convertir Engine en un host KVM.  
D. Desactivar DWH automáticamente en todos los casos.

**Respuesta correcta: A.**

Separar la base puede aislar carga, pero convierte su ruta de red en parte crítica del plano de control. Hay que verificar que esa modalidad esté soportada en la versión concreta, proteger las credenciales, decidir TLS y coordinar backup y recuperación. DNS y DWH siguen requiriendo decisiones independientes.

---

# Alta de hosts y validación

## 39. Al añadir un host, su estado aparece `Installing`. ¿Qué está haciendo Engine?

A. Conectando, desplegando VDSM y dependencias, estableciendo confianza e inventariando el host.  
B. Clonando todos los discos de las VMs al host.  
C. Convirtiendo NFS en almacenamiento local.  
D. Ejecutando una segunda copia de Engine.

**Respuesta correcta: A.**

El alta es un bootstrap real. Engine utiliza la conectividad y autenticación configuradas, instala el agente del host, crea confianza mediante certificados y obtiene sus capacidades. Los discos permanecen en el Storage Domain y no se duplica Engine.

## 40. Después de añadir un host administrado, ¿dónde deben realizarse los cambios persistentes de su red OLVM?

A. Desde Administration Portal o API, para mantener Engine como fuente de verdad.  
B. Siempre con `nmcli` directamente en el host, sin informar a Engine.  
C. Dentro de una VM cualquiera.  
D. En la base `ovirt_engine_history`.

**Respuesta correcta: A.**

Engine mantiene el estado deseado y VDSM aplica la configuración. Un cambio manual puede funcionar temporalmente en Linux y quedar fuera de sincronía, desaparecer o impedir que Engine valide el host. La base histórica no es una interfaz de configuración.

## 41. Un host queda en `Installing` y Engine no puede autenticar por SSH. ¿Qué debe investigarse primero?

A. Conectividad, puerto y método de autenticación SSH.  
B. La retención diaria de DWH.  
C. Los snapshots de una VM.  
D. La política de afinidad VM–VM.

**Respuesta correcta: A.**

El flujo aún no ha superado el acceso de bootstrap, por lo que no tiene sentido comenzar por capas posteriores. Se revisan FQDN, ruta, firewall, puerto, usuario, clave pública o contraseña y logs de SSH. DWH y afinidad no participan en la autenticación inicial.

## 42. Un host tiene VDSM activo, pero queda `Non Operational` porque no dispone de una red requerida. ¿Qué describe mejor la situación?

A. Engine se comunica con el host, pero su configuración no cumple las condiciones del Cluster.  
B. El host está apagado de forma verificada.  
C. PostgreSQL ha perdido todas las bases.  
D. La VM de Engine se ha convertido en Hosted Engine.

**Respuesta correcta: A.**

`Non Operational` puede aparecer cuando existe comunicación, pero el host no cumple red, storage u otra condición obligatoria. No equivale a `Non Responsive` ni confirma que el servidor esté apagado. La resolución consiste en corregir la condición desde el mecanismo soportado.

## 43. ¿Qué prueba valida mejor que el camino de migración funciona?

A. Migrar de forma controlada una VM de prueba entre hosts compatibles y observar eventos y métricas.  
B. Abrir únicamente la pantalla de login.  
C. Crear un usuario sin permisos.  
D. Consultar solo el tamaño de la base `engine`.

**Respuesta correcta: A.**

La prueba recorre scheduler, redes de migración, libvirt/VDSM, acceso compartido al storage y destino. Debe utilizar una carga prescindible y observar el resultado. Portal, usuario y tamaño de base prueban otros componentes, no la transferencia host a host.

## 44. ¿Qué afirmación describe una aceptación operativa completa?

A. Se validan portal, hosts, storage, redes, consola, migración, monitorización, notificaciones y backup.  
B. Basta con que `engine-setup` termine sin error.  
C. Basta con que el servicio PostgreSQL esté activo.  
D. Basta con crear el Data Center `Default`.

**Respuesta correcta: A.**

Setup demuestra que la configuración local terminó, pero una plataforma depende de caminos distribuidos y procedimientos operativos. Las pruebas funcionales descubren errores de firewall, DNS, storage, consola o migración que no aparecen al instalar Engine. Backup y observabilidad forman parte de la aceptación, no tareas opcionales sin fecha.

## 45. ¿Dónde debe guardarse el backup inicial de Engine?

A. En una ubicación independiente del propio host o VM de Engine y bajo una política de custodia.  
B. Únicamente en el filesystem local de Engine.  
C. Dentro del disco temporal de una VM de prueba.  
D. En la memoria de un host KVM.

**Respuesta correcta: A.**

Un backup que desaparece con el sistema que debe recuperar no cubre el fallo principal. Debe extraerse, protegerse y verificarse conforme a la política de la empresa. Guardarlo localmente puede servir como paso intermedio, pero no como destino único.

---

# Casos integrados

## 46. Grafana muestra datos hasta las 11:00, pero son las 12:00. Engine administra las VMs con normalidad. ¿Qué componente conviene revisar primero?

A. El servicio DWH y su carga hacia `ovirt_engine_history`.  
B. El watchdog de todas las VMs.  
C. El bridge de una única vNIC.  
D. La plantilla Blank.

**Respuesta correcta: A.**

El síntoma afecta al camino histórico, mientras el plano operativo sigue funcionando. Se comprueba servicio DWH, logs, conexión a ambas bases, última carga y espacio. Reiniciar VMs o tocar la red de una vNIC no explica que todo el histórico se haya detenido a una hora común.

## 47. Se diseña una Self-Hosted Engine con un único host y storage local. ¿Cuál es el problema principal?

A. No proporciona la arquitectura compartida y redundante necesaria para proteger la VM de Engine entre hosts.  
B. El nombre de la VM contiene la palabra Engine.  
C. Las VMs no pueden usar VirtIO.  
D. Grafana requiere siempre un disco RAW.

**Respuesta correcta: A.**

Con un único host y storage local no existe otro destino capaz de acceder a la VM de Engine cuando el servidor falla. Se conserva la complejidad de Hosted Engine sin obtener HA real. El diseño debe incluir varios hosts compatibles, storage compartido, red y fencing.

## 48. Un equipo abre 443 desde los administradores a Engine y afirma que el firewall está completo. ¿Qué falta en el razonamiento?

A. Considerar los demás flujos: Engine–host, host–host, consolas, storage y componentes opcionales.  
B. Cambiar todos los discos a QCOW2.  
C. Crear una etiqueta de afinidad.  
D. Deshabilitar IPv6 en Engine.

**Respuesta correcta: A.**

443 permite acceder a servicios web, pero la plataforma es distribuida. VDSM, bootstrap, migración, consolas, NFS y una base remota tienen otros orígenes y destinos. Además, IPv6 no debe deshabilitarse en Engine según los requisitos; hacerlo no arreglaría la matriz incompleta.

## 49. La base `ovirt_engine_history` crece rápidamente. ¿Cuál es la primera actuación correcta?

A. Revisar tendencia, retención DWH, tamaño del entorno y espacio disponible antes de aplicar mantenimiento.  
B. Borrar filas manualmente al azar.  
C. Detener permanentemente Engine.  
D. Desconectar el Storage Domain NFS.

**Respuesta correcta: A.**

El crecimiento puede ser coherente con una escala Full, más VMs o mayor retención. Primero se mide y compara con el diseño. Cualquier vacuum, ajuste o migración de DWH sigue el procedimiento soportado y se respalda previamente; borrar tablas manualmente puede corromper el histórico.

## 50. Una plataforma se instala correctamente, pero no se ha probado fencing ni el backup. ¿Puede darse por aceptada para HA?

A. No; existen componentes instalados, pero faltan pruebas esenciales de aislamiento y recuperación.  
B. Sí, porque el portal abre.  
C. Sí, porque DWH recoge muestras.  
D. Sí, siempre que todas las VMs estén en un único host.

**Respuesta correcta: A.**

HA depende de eliminar una ejecución dudosa, encontrar capacidad y recuperar el plano de control cuando sea necesario. Una configuración no probada es solo una intención. Abrir el portal o recibir métricas no demuestra fencing ni restauración, y concentrar todas las VMs aumenta el dominio de fallo.
