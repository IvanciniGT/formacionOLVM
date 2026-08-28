# Práctica · De una imagen de nube a una plantilla propia

**Bloque de examen:** administración de máquinas virtuales (10%) y plantillas.
Toca de paso almacenamiento (19%) al ver la provisión delgada y las cadenas de
discos.

**Duración estimada:** 60–75 minutos.

**Punto de partida:** cada alumno tiene en *Almacenamiento → Discos* un disco
llamado `rocky9-alumnoN`, con N su número. Es una imagen de nube de Rocky
Linux 9 sin tocar.

**Objetivo:** convertir esa imagen cruda en una plantilla propia y demostrar
que los clones que salgan de ella son máquinas independientes.

---

## Paso 1 · Mirar la materia prima

*Almacenamiento → Discos*, busca `rocky9-alumnoN`.

Anota lo que ves:

| Campo | Valor |
|---|---|
| Tamaño virtual | 10 GiB |
| Tamaño real | ~1,2 GiB |
| Política de asignación | Provisión delgada |
| Adjunto a | *(vacío)* |

**Piensa antes de seguir:** ¿esto es un instalador?

No. Es un disco con Rocky Linux **ya instalado**: tiene sus particiones EFI,
boot y raíz. No hay Anaconda ni preguntas de instalación. Arranca directo al
sistema en segundos. Eso es una imagen de nube: no se instala, se clona.

Y ocupa 1,2 GiB de los 10 que dice tener porque es **delgada**: el fichero
crece según se escribe.

---

## Paso 2 · Crear la máquina

En el menú de la izquierda: **Compute → Máquinas virtuales**. Botón **Nueva**,
arriba a la derecha.

Se abre un diálogo con pestañas a la izquierda. Vas a tocar tres.

### Pestaña General

| Campo | Qué pones |
|---|---|
| Clúster | `Curso` |
| Plantilla | déjalo en `Blank` — **no** vas a partir de plantilla |
| Sistema operativo | `Red Hat Enterprise Linux 9.x x64` |
| Optimizado para | `Servidor` |
| Nombre | `rocky-alumnoN` |

Más abajo, en **Imágenes de instancia**, pulsa **Adjuntar** (no *Crear*: el
disco ya existe, no vas a fabricar uno vacío).

Se abre una lista de discos sueltos. Busca `rocky9-alumnoN`, **marca su
casilla**, y **marca también la casilla de la columna de arranque** — según la
versión aparece como *SO* o como *Arrancable*. Acepta.

> Si te dejas la casilla de arranque, la máquina encenderá y se quedará en el
> firmware diciendo que no encuentra sistema operativo. Es el fallo más común
> de esta práctica.

Aún en General, abajo del todo, en **Instanciar imagen de VM de la red**, elige
el perfil `alumnos` para `nic1`.

### Pestaña Sistema

| Campo | Qué pones |
|---|---|
| Tamaño de memoria | `1024 MB` |
| Memoria física garantizada | `1024 MB` |
| CPU virtuales totales | `1` |

### Aceptar

Pulsa **Aceptar**. La máquina aparece en la lista con estado **Apagada**.

**Comprobación:** selecciona la máquina, pestaña **Discos** en el panel de
abajo. Debe salir `rocky9-alumnoN` con la marca de arranque puesta.

## Paso 3 · Arrancarla a lo tonto (y que falle)

Pulsa **Ejecutar** a secas y abre la **Consola**.

Verás el arranque completo, sin errores, hasta un `localhost login:`.

Prueba `root`, prueba `rocky`, prueba lo que quieras. **No vas a entrar.**

**Por qué:** en la imagen sólo existe la cuenta `root`, y está bloqueada — no
tiene contraseña. El usuario `rocky` **ni siquiera existe todavía**. Lo que hay
en `/etc/cloud/cloud.cfg` es una receta:

```yaml
system_info:
  distro: rocky
  default_user:
    name: rocky
    lock_passwd: True
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
```

Ese usuario lo crea **cloud-init en el primer arranque**, y fíjate en
`lock_passwd: True`: nace sin contraseña, pensado para entrar con clave SSH.

Una imagen de nube sin metadatos es una máquina que arranca perfectamente y en
la que no puede entrar nadie.

**Apaga la máquina.**

---

## Paso 4 · Arrancarla como se debe

Con la máquina **apagada**, selecciónala en la lista.

Arriba está el botón **Ejecutar**. **No lo pulses.** A su lado hay una flecha
que despliega más opciones: ahí eliges **Ejecutar una vez**.

Se abre un diálogo con varias secciones plegadas. Localiza **Ejecución
inicial** y despliégala.

Dentro verás una casilla **Cloud-Init**. **Márcala.** Hasta que no la marques,
el resto de campos ni aparecen.

Ahora rellena:

| Campo | Qué pones |
|---|---|
| Nombre de host de la VM | `rocky-alumnoN` |

Despliega **Autenticación**:

| Campo | Qué pones |
|---|---|
| Nombre de usuario | `alumno` |
| Configurar contraseña | **marcar la casilla** |
| Contraseña | `Alumno.2026` |
| Verificar contraseña | `Alumno.2026` |

Despliega **Script personalizado** y pega esto tal cual, respetando la
indentación, que es YAML:

```yaml
ssh_pwauth: true
chpasswd:
  expire: false
packages:
  - qemu-guest-agent
runcmd:
  - [ systemctl, enable, --now, qemu-guest-agent ]
```

Pulsa **Aceptar**.

La máquina arranca. Ábrele la **Consola** y observa: verás pasar líneas de
`cloud-init` que no salían en el intento anterior.

Espera a que llegue al `login:` y entra con `alumno` / `Alumno.2026`.

> El primer arranque tarda más de lo normal: cloud-init está creando el
> usuario, poniendo la contraseña y descargando el agente. Si el SSH te
> rechaza la contraseña en los primeros segundos, espera medio minuto. La
> máquina todavía se está asentando y no es un error tuyo.

### Qué acaba de pasar, que es el corazón de la práctica

Al marcar Cloud-Init, OLVM ha fabricado un **ConfigDrive**: un CD-ROM virtual
diminuto con los datos que has escrito, y se lo ha enganchado a la máquina
antes de encenderla. cloud-init, que ya venía instalado en la imagen, lo ha
encontrado al arrancar y ha hecho lo que decía.

Con un **Ejecutar** normal ese CD-ROM **no se monta**. Por eso el paso 3 falló.
No es que cloud-init estuviera roto: es que no tenía nada que leer.

## Paso 5 · Comprobar que quedó bien

Dentro de la máquina:

```bash
hostname                 # rocky-alumnoN
id                       # uid=1000(alumno)
sudo -n id               # uid=0(root), sin pedir contraseña
cloud-init status        # status: done
systemctl is-active qemu-guest-agent
```

Y en el portal, en la pestaña **Información del huésped** de la máquina,
deberías ver ya el sistema operativo, el nombre de host y la IP. Eso lo reporta
el agente que acabas de instalar. Sin agente, esa pestaña está vacía.

---

## Paso 6 · Quitarle la identidad a la máquina

Este paso es el que separa una plantilla buena de una que dará problemas.

Ahora mismo tu máquina tiene cosas que la identifican **a ella**: un
`machine-id`, unas claves de host SSH, un arrendamiento DHCP, unos logs. Si
plantillas así, los diez clones que salgan serán gemelos: presentarán la misma
huella SSH y pedirán el mismo arrendamiento al DHCP, con lo que se pelearán por
la IP.

Dentro de la máquina, ejecuta:

```bash
sudo cloud-init clean --logs --seed        # que pueda volver a ejecutarse
sudo rm -f /etc/ssh/ssh_host_*             # claves de host: se regeneran solas
sudo truncate -s 0 /etc/machine-id         # identidad de systemd
sudo rm -f /var/lib/dbus/machine-id
sudo rm -f /etc/hostname                   # que cada clon ponga el suyo
sudo rm -rf /var/lib/NetworkManager/*      # arrendamientos DHCP
sudo rm -rf /var/log/journal/*
sudo rm -f  /var/log/messages /var/log/secure
sudo rm -rf /root/.ssh ~/.ssh
history -c
sudo poweroff
```

Fíjate en el primero: `cloud-init clean` borra el **estado**, no el programa.
Así la plantilla conserva cloud-init utilizable y quien clone podrá volver a
personalizar con "Ejecutar una vez".

---

## Paso 7 · Convertirla en plantilla

Con la máquina **apagada**: *Compute → Máquinas virtuales*, seleccionala,
despliega los tres puntos y elige **Crear plantilla**.

| Campo | Valor |
|---|---|
| Nombre | `rocky9-alumnoN-plantilla` |
| Clúster | `Curso` |
| **Sellar plantilla** | **NO marcar** |

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

Crea **dos** máquinas desde tu plantilla:

*Compute → Máquinas virtuales → Nueva*, en **Basada en plantilla** elige
`rocky9-alumnoN-plantilla`, nómbralas `rocky-alumnoN-a` y `rocky-alumnoN-b`,
red `alumnos`, y arráncalas con **Ejecutar** normal — ahora sí funciona, porque
el usuario viene de fábrica dentro de la plantilla.

Entra en las dos y compara:

```bash
cat /etc/machine-id
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
hostname
ip -4 addr show eth0
```

Deberías ver:

- **machine-id distinto** en cada una — se genera en el primer arranque porque
  lo dejaste vacío
- **huella SSH distinta** — las claves se regeneran porque las borraste
- **hostname `localhost.localdomain`** en las dos — quitaste `/etc/hostname` y
  nadie se lo ha puesto; es lo esperado, y se arregla con "Ejecutar una vez"
- **IPs distintas** — consecuencia de tener machine-id distinto

Si alguna de estas cosas sale igual en las dos máquinas, algo del paso 6 se
quedó sin hacer.

---

## Paso 9 · Mirar lo que ocupa

*Almacenamiento → Discos.* Busca los discos de tus dos clones.

Verás 10 GiB de tamaño virtual y unos pocos MiB de tamaño real. **Los clones
finos no copian el disco**: crean una capa nueva encima que sólo guarda las
diferencias, y usan el disco de la plantilla como respaldo.

De ahí una consecuencia práctica: **no puedes borrar la plantilla mientras
existan clones finos suyos**, porque les quitarías el suelo.

---

## Extra · El experimento del sellado

Si te sobra tiempo, comprueba lo del paso 7 en vez de creértelo.

1. Vuelve a crear una VM desde tu `rocky9-alumnoN` original y personalízala con
   "Ejecutar una vez", como en el paso 4.
2. Apágala y créale una plantilla, esta vez **marcando Sellar plantilla**.
3. Crea una VM desde esa plantilla y arráncala.
4. Intenta entrar con `alumno` / `Alumno.2026`.

Sí podrás entrar: el usuario sigue ahí. Lo que verás es que la
máquina se llama `localhost` y que cloud-init NO se vuelve a ejecutar,
porque el sellado no borró su estado.

---

## Preguntas para pensar

1. ¿Por qué una imagen de nube no trae usuario ni contraseña? ¿Qué problema de
   seguridad evita?
2. ¿Qué hace exactamente "Ejecutar una vez" que no hace "Ejecutar"?
3. Si diez máquinas comparten `machine-id`, ¿por qué acaban peleándose por la
   misma IP? ¿Qué tiene que ver systemd con el DHCP?
4. Tienes una plantilla con veinte clones finos y necesitas borrarla. ¿Qué
   opciones tienes?
5. ¿En qué caso sí querrías sellar una plantilla?
