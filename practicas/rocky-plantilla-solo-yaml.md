# Práctica · De imagen de nube a plantilla, con Rocky Linux 9

**Variante:** toda la personalización se hace escribiendo **un único fichero
YAML** en el script personalizado. No se rellena ni un solo campo del
formulario de Ejecución inicial.

**Bloque de examen:** administración de máquinas virtuales (10%) y plantillas.
Roza almacenamiento (19%) al ver la provisión delgada y las cadenas de discos.

**Duración:** 60–75 minutos.

**Punto de partida:** en *Almacenamiento → Discos* tienes un disco llamado
`rocky9-alumnoN`, con N tu número. Es una imagen de nube de Rocky Linux 9 sin
tocar. Sustituye `N` por tu número en todo el guion.

---

## Paso 1 · Mirar la materia prima

Ve a **Almacenamiento → Discos** y busca `rocky9-alumnoN`.

Anota estos cuatro datos de la fila:

| Campo | Valor esperado |
|---|---|
| Tamaño virtual | 10 GiB |
| Tamaño real | ~1,2 GiB |
| Política de asignación | Provisión delgada |
| Adjunto a | *(vacío)* |

**Antes de seguir, piensa:** ¿esto es un instalador?

No lo es. Es un disco con Rocky Linux **ya instalado**, con sus particiones EFI,
boot y raíz. No hay Anaconda ni preguntas. Arranca al sistema en segundos.
Una imagen de nube no se instala: se clona.

Y ocupa 1,2 GiB de los 10 que dice tener porque es **delgada**: el fichero
crece a medida que se escribe en él.

---

## Paso 2 · Crear la máquina

**Compute → Máquinas virtuales → Nueva.**

### Pestaña General

| Campo | Valor |
|---|---|
| Clúster | `Curso` |
| Plantilla | `Blank` — **no** partes de plantilla |
| Sistema operativo | `Red Hat Enterprise Linux 9.x x64` |
| Optimizado para | `Servidor` |
| Nombre | `rocky-alumnoN` |

Baja hasta **Imágenes de instancia** y pulsa **Adjuntar**. Ojo: *Adjuntar*, no
*Crear* — el disco ya existe, no vas a fabricar uno vacío.

En la lista que se abre, busca `rocky9-alumnoN`, marca su casilla y **marca
también la casilla de la columna de arranque** (aparece como *SO* o
*Arrancable* según la versión). Acepta.

> Si te olvidas de la casilla de arranque, la máquina encenderá y se quedará en
> el firmware diciendo que no encuentra sistema operativo. Es el fallo más
> común de esta práctica.

Sigue bajando, y en **Instanciar imagen de VM de la red** elige el perfil
`alumnos` para `nic1`.

### Pestaña Sistema

| Campo | Valor |
|---|---|
| Tamaño de memoria | `1024 MB` |
| Memoria física garantizada | `1024 MB` |
| CPU virtuales totales | `1` |

Pulsa **Aceptar**.

**Comprobación:** selecciona la máquina y mira la pestaña **Discos** del panel
inferior. Debe aparecer `rocky9-alumnoN` con la marca de arranque.

---

## Paso 3 · Arrancarla mal, a propósito

Pulsa **Ejecutar** y abre la **Consola**.

Verás un arranque limpio, sin un solo error, hasta un `localhost login:`.

Prueba a entrar. `root`, `rocky`, lo que se te ocurra. **No vas a poder.**

**Por qué:** en la imagen sólo existe la cuenta `root`, y está bloqueada, sin
contraseña. El usuario `rocky` **ni siquiera existe todavía**. Lo único que hay
es una receta en `/etc/cloud/cloud.cfg`:

```yaml
system_info:
  distro: rocky
  default_user:
    name: rocky
    lock_passwd: True
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
```

Ese usuario lo crearía cloud-init en el primer arranque, y fíjate en
`lock_passwd: True`: nacería **sin contraseña**, pensado para entrar con clave
SSH, no por consola.

Una imagen de nube sin metadatos es una máquina que arranca perfectamente y en
la que no puede entrar nadie.

**Apaga la máquina** antes de seguir.

---

## Paso 4 · Arrancarla bien, con un solo YAML

Con la máquina apagada y seleccionada, **no** pulses *Ejecutar*. A su lado hay
una flecha con más opciones: elige **Ejecutar una vez**.

En el diálogo, despliega **Ejecución inicial** y marca la casilla
**Cloud-Init**. Hasta que no la marques no aparece el resto.

**Deja TODOS los campos vacíos.** Nombre de host, autenticación, redes, zona
horaria: nada. Todo va a ir en el script.

Despliega **Script personalizado** y pega este fichero **completo**, cambiando
`N` por tu número:

```yaml
# ─── Identidad de la máquina ────────────────────────────────────────────
hostname: rocky-alumnoN
fqdn: rocky-alumnoN.curso.local
manage_etc_hosts: true
timezone: Europe/Madrid

# ─── Usuario ────────────────────────────────────────────────────────────
# Esta lista SUSTITUYE al usuario por defecto de la imagen: el usuario
# "rocky" no se creará. Si lo quisieras además del tuyo, la lista sería:
#   users:
#     - default
#     - name: alumno
#       ...
users:
  - name: alumno
    gecos: Alumno del curso OLVM
    groups: [wheel, adm, systemd-journal]
    shell: /bin/bash
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    lock_passwd: false

# ─── Contraseñas ────────────────────────────────────────────────────────
# expire: false es imprescindible. Sin eso la contraseña nace caducada y
# el primer inicio de sesión te obliga a cambiarla.
chpasswd:
  expire: false
  list: |
    root:Alumno.2026
    alumno:Alumno.2026

# Permite entrar por SSH con contraseña. Las imágenes de nube lo traen
# desactivado de fábrica: sólo admiten clave.
ssh_pwauth: true

# ─── Paquetes ───────────────────────────────────────────────────────────
# El agente de invitado es lo que hace que el portal muestre sistema
# operativo, nombre de host e IP en la pestaña Información del huésped.
packages:
  - qemu-guest-agent

# ─── Ficheros ───────────────────────────────────────────────────────────
# Repone lo que el engine hacía por su cuenta y que nuestro runcmd pisa.
# Ver la nota de más abajo.
write_files:
  - path: /etc/cloud/cloud.cfg.d/99-datasource.cfg
    permissions: '0644'
    content: |
      datasource_list: [ NoCloud, ConfigDrive ]

# ─── Órdenes finales ────────────────────────────────────────────────────
runcmd:
  - [ systemctl, enable, --now, qemu-guest-agent ]

final_message: "cloud-init ha terminado tras $UPTIME segundos."
```

Pulsa **Aceptar**, abre la **Consola** y observa: ahora verás pasar líneas de
`cloud-init` que en el paso 3 no salían.

Cuando llegue al `login:`, entra con `alumno` / `Alumno.2026`.

> El primer arranque tarda más: está creando el usuario, poniendo contraseñas y
> descargando el agente. Si el SSH te rechaza la contraseña en los primeros
> segundos, espera medio minuto: la máquina aún se está asentando.

### Tres cosas que hay que entender de este paso

**1. Qué acaba de pasar.** Al marcar Cloud-Init, OLVM fabrica un
**ConfigDrive**: un CD-ROM virtual con dos ficheros, `user_data` (tu YAML) y
`meta_data.json`, etiquetado `config-2`, y se lo engancha a la máquina antes de
encenderla. cloud-init, que ya venía instalado en la imagen, lo encuentra al
arrancar y obedece.

Con un **Ejecutar** normal ese CD-ROM **no se monta**. Por eso el paso 3 falló:
cloud-init no estaba roto, es que no tenía nada que leer.

**2. Tu YAML no es lo único que va dentro.** El engine escribe primero una
cabecera suya y **pega tu script detrás, tal cual**. El fichero que acaba
dentro de la máquina es este:

```yaml
#cloud-config
output:
  all: '>> /var/log/cloud-init-output.log'
disable_root: 0
runcmd:
- 'sed -i ... datasource_list: ["NoCloud", "ConfigDrive"] >> /etc/cloud/cloud.cfg'
ssh_deletekeys: 'false'
ssh_pwauth: true
chpasswd:
  expire: false
        ← a partir de aquí empieza tu script, pegado sin más
hostname: rocky-alumnoN
...
```

**Por eso no escribes `#cloud-config` al principio de tu script**: el engine ya
lo ha puesto. Si lo pones tú, queda en mitad del fichero como un comentario.
No rompe nada, pero confunde.

**3. Las claves repetidas: gana la última.** No hay fusión inteligente, hay
concatenación. Como tu `chpasswd` y tu `runcmd` van después, **sustituyen** a
los del engine. Eso es bueno para `chpasswd` —quieres el tuyo, que trae la
lista de contraseñas— pero significa que el `runcmd` del engine, que ajustaba
el `datasource_list`, se pierde. De ahí el bloque `write_files` de arriba: lo
repone de una forma más limpia, con un fichero en `/etc/cloud/cloud.cfg.d/` en
vez de un `sed`.

---

## Paso 5 · Comprobar que quedó bien

Dentro de la máquina:

```bash
hostname                              # rocky-alumnoN
id                                    # uid=1000(alumno)
sudo -n id                            # uid=0(root), sin pedir contraseña
cloud-init status                     # status: done
systemctl is-active qemu-guest-agent  # active
cat /etc/cloud/cloud.cfg.d/99-datasource.cfg
```

Si algo no se aplicó, el sitio donde mirar es `/var/log/cloud-init.log` y
`/var/log/cloud-init-output.log` dentro del invitado. Un error de indentación
en tu YAML hace que cloud-init **descarte el fichero entero**, no sólo tu
parte, y la máquina arranca sin personalizar.

En el portal, pestaña **Información del huésped** de la máquina: ya deben salir
sistema operativo, nombre de host e IP. Eso lo reporta el agente que acabas de
instalar. Compara con una VM CirrOS, que no lo tiene, y verás la pestaña vacía.

---

## Paso 6 · Quitarle la identidad a la máquina

Este paso separa una plantilla buena de una que dará guerra, y es el que más se
salta la gente.

Tu máquina tiene ahora cosas que la identifican **a ella y sólo a ella**: un
`machine-id`, unas claves de host SSH, un arrendamiento DHCP, unos logs. Si
plantillas así, todos los clones serán gemelos: presentarán la misma huella SSH
y, al derivar del `machine-id` su identificador de DHCP, pedirán el mismo
arrendamiento y se pelearán por la IP.

Dentro de la máquina, ejecuta esto en orden:

```bash
sudo cloud-init clean --logs --seed     # borra el ESTADO, no el programa
sudo rm -f /etc/ssh/ssh_host_*          # claves de host: se regeneran solas
sudo truncate -s 0 /etc/machine-id      # identidad de systemd
sudo rm -f /var/lib/dbus/machine-id
sudo rm -f /etc/hostname                # que cada clon ponga el suyo
sudo rm -rf /var/lib/NetworkManager/*   # arrendamientos DHCP
sudo rm -rf /var/log/journal/*
sudo rm -f  /var/log/messages /var/log/secure
sudo rm -rf /root/.ssh ~/.ssh
history -c
sudo poweroff
```

Fíjate en el primero: `cloud-init clean` borra el **estado**, no el paquete. La
plantilla conserva cloud-init utilizable, y quien clone podrá volver a
personalizar con "Ejecutar una vez".

---

## Paso 7 · Convertirla en plantilla

Con la máquina **apagada** (si no, la opción sale deshabilitada), selecciónala,
abre el menú de tres puntos o **Más acciones**, y elige **Crear plantilla**.

| Campo | Valor |
|---|---|
| Nombre | `rocky9-alumnoN-plantilla` |
| Clúster | `Curso` |
| **Sellar plantilla** | **NO marcar** |

Tarda un rato: está copiando el disco.

### ¿Qué hace exactamente "Sellar plantilla"? (medido en este laboratorio)

Sellar ejecuta `virt-sysprep` sobre el disco de la plantilla, con la máquina
apagada y desde fuera. OLVM lo lanza así, sin añadir operaciones:

```
virt-sysprep --hostname localhost --selinux-relabel -a <disco>
```

Se personalizó una VM, se comprobó por SSH que el usuario existía, se selló, y
se leyó el disco de la plantilla resultante. Esto es lo que pasó:

| Cosa | ¿Sobrevive al sellado? |
|---|---|
| Usuario `alumno` | **Sí** |
| Su contraseña (hash en `/etc/shadow`) | **Sí** |
| `/etc/sudoers.d/90-cloud-init-users` | **Sí** |
| Claves de host SSH | **No**, se borran |
| `/var/log/messages` y `secure` | **No**, se borran |
| Nombre de host | se fija a `localhost` |
| `machine-id` | se sustituye por **uno nuevo, pero fijo** |
| Estado de cloud-init (`/var/lib/cloud`) | **Sí**, se queda |
| Arrendamientos DHCP de NetworkManager | **Sí**, se quedan |

**Conclusión, y es la parte importante:** sellar **no borra usuarios** —la
operación `user-account` de `virt-sysprep` existe, pero no está entre las que
se ejecutan por defecto y OLVM no la pide—. Pero **sellar por sí solo no
basta**:

- Deja el estado de cloud-init, así que los clones creerán que ya se
  personalizaron y **no volverán a ejecutar cloud-init**.
- Deja los arrendamientos DHCP.
- Deja un `machine-id` **fijo** en la plantilla, con lo que todos los clones
  nacerían compartiéndolo, que es justo lo que se quiere evitar.

Por eso en esta práctica se limpia a mano (paso 6) y **no** se sella: la
limpieza manual cubre todo lo anterior. Sellar además no hace daño, pero no te
ahorra el paso 6.

---

## Paso 8 · Demostrar que los clones son independientes

Crea **dos** máquinas: **Compute → Máquinas virtuales → Nueva**, y esta vez en
**Plantilla** elige `rocky9-alumnoN-plantilla`. Llámalas `rocky-alumnoN-a` y
`rocky-alumnoN-b`, con la red `alumnos`.

Arráncalas con **Ejecutar** normal. Ahora sí funciona, porque el usuario viene
de fábrica dentro de la plantilla.

Entra en las dos y compara:

```bash
cat /etc/machine-id
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
hostname
ip -4 addr show eth0
```

Lo que debes ver:

| Dato | Esperado |
|---|---|
| `machine-id` | **distinto** en cada una: se genera al arrancar porque lo dejaste vacío |
| Huella SSH | **distinta**: las claves se regeneran porque las borraste |
| `hostname` | `localhost.localdomain` en **las dos** — correcto: quitaste `/etc/hostname` y sin cloud-init nadie se lo pone |
| IP | **distintas**, consecuencia de tener machine-id distinto |

Si algo sale **igual** en las dos máquinas, es que te saltaste parte del paso 6.
Vuelve atrás y busca qué falta.

---

## Paso 9 · Mirar lo que ocupa

**Almacenamiento → Discos**, busca los discos de tus dos clones: 10 GiB de
tamaño virtual y unos pocos MiB de tamaño real.

Los clones finos **no copian el disco**: crean una capa nueva encima que sólo
guarda las diferencias, y usan el disco de la plantilla como respaldo.

De ahí una consecuencia práctica: **no puedes borrar una plantilla mientras
existan clones finos suyos**, porque les quitarías el suelo.

---

## Extra · El experimento del sellado

Si te sobra tiempo, comprueba lo del paso 7 en vez de creértelo:

1. Crea otra VM desde tu `rocky9-alumnoN` y personalízala como en el paso 4.
2. Apágala y créale una plantilla, esta vez **marcando Sellar plantilla**.
3. Crea una VM desde esa plantilla y arráncala.
4. Intenta entrar con `alumno` / `Alumno.2026`.

Sí podrás entrar: el usuario sigue ahí. Lo que verás es que la máquina se
llama `localhost` y que cloud-init no se vuelve a ejecutar, porque el sellado
no borró su estado.

---

## Preguntas para pensar

1. ¿Por qué una imagen de nube no trae usuario ni contraseña? ¿Qué problema
   evita?
2. ¿Qué hace exactamente "Ejecutar una vez" que no hace "Ejecutar"?
3. Si diez máquinas comparten `machine-id`, ¿por qué acaban peleándose por la
   misma IP? ¿Qué tiene que ver systemd con el DHCP?
4. En tu YAML, ¿qué habría pasado si hubieras puesto `chpasswd` **antes** que
   el resto en lugar de después? ¿Y si el engine hubiera pegado su cabecera al
   final en vez de al principio?
5. Tienes una plantilla con veinte clones finos y necesitas borrarla. ¿Qué
   opciones tienes?
6. ¿En qué caso sí querrías sellar una plantilla?
