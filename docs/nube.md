# Nube: Nextcloud + restic

## Objetivo

- En el servidor, todo guardado **comprimido sin pérdida** donde la compresión rinda.
- Al traer archivos al PC, que lleguen **normales, sin comprimir**.
- Elegir qué traer **por carpetas**, con el cliente gráfico de Nextcloud.

## Hecho clave: Nextcloud no comprime

Nextcloud guarda los archivos tal cual. Hay dos formas de comprimir:

| Dónde | Cómo | Ventaja | Requisito |
|---|---|---|---|
| **En el disco del servidor** | btrfs `compress-force=zstd` o ZFS `compression=zstd` | Transparente: se guarda comprimido y se entrega normal | El disco de datos de Nextcloud debe ser btrfs o ZFS |
| **En el cliente** | restic `--compression max` | Comprime también en tránsito; cifra; deduplica | Los datos no se ven como archivos en Nextcloud |

Antes de decidir, **mide cuánto se comprimen tus datos**. Fotos, videos y archivos ya
empaquetados no ganan casi nada; sistema, código y documentos sí:

```bash
# muestra aproximada: bytes originales vs comprimidos con zstd -3
find /ruta -type f -size +1M | shuf -n 40 | while read -r f; do head -c 8M "$f"; done \
  | tee >(wc -c >&2) | zstd -3 -c | wc -c
```

Si el servidor usa ext4 y lo grande es incompresible, convertirlo no vale la pena.
Comprime en el cliente lo que sí rinde.

## Tres canales

| Datos | Canal | Sentido | Comprimido | Visible en Nextcloud |
|---|---|---|---|---|
| Carpeta de trabajo (documentos) | `nube-sync docs` → `nextcloudcmd` | ambos | si el servidor lo hace | sí |
| Archivo, fotos de teléfonos | `nube-sync datos` → `rclone copy` | solo subida, nunca borra | si el servidor lo hace | sí |
| Sistema y backups crudos de teléfonos | `restic-nube` → SFTP | solo subida | **sí, en cliente** | no (repositorio aparte) |

Traer al PC: cliente gráfico de Nextcloud, **sincronización selectiva por carpetas**.
Desmarca **todo lo que suben `nube-sync docs` y `nube-sync datos`**: ya está en el PC.
Si lo dejas marcado, el cliente lo vuelve a bajar a su carpeta (por defecto `~/Nextcloud`,
en el disco del sistema). Eso duplica los datos, carga al servidor con descargas y
llena el disco. Con `docs`, además, quedan dos clientes bidireccionales sobre la misma
carpeta. Al desmarcar, el cliente borra solo su copia local; el servidor no cambia.

**No sincronices repositorios git.** Ya tienen su copia en el servidor git; en Nextcloud solo
suman miles de archivos que cambian a cada rato (y una sincronización bidireccional sobre
un repo vivo puede dejar copias de conflicto). Exclúyelos (ver abajo) o guárdalos fuera de
la carpeta sincronizada.

## Cuidar el servidor

Pensado para un servidor compartido con disco mecánico: cada archivo que sube o se compara
cuesta varias consultas a la base de datos de Nextcloud.

| Medida | Dónde | Valor por defecto |
|---|---|---|
| `docs` cada 3 h, no cada hora | `nube-docs.timer` | arranque + 15 min, luego cada 3 h ± 10 min |
| `docs` despacio | `nube-sync` | 1 petición a la vez (`OWNCLOUD_MAX_PARALLEL=1`), `NUBE_UPLIMIT_KB=2048` |
| `datos` en **tandas** | `nube-sync` | `NUBE_TANDA_MIN=30` por tanda; la siguiente sigue donde quedó |
| Tandas espaciadas | `nube-datos.timer` | arranque + 30 min, luego cada 2 h ± 10 min (sirve en un portátil apagado de noche) |
| `datos` despacio | `nube-sync` | 1 subida y 2 comparaciones a la vez, `NUBE_TPS=4` peticiones/s |
| Más lento de día | `nube-sync` | `NUBE_BWLIMIT="08:00,1M 22:00,off"` (formato de horario de rclone) |
| Sin reintentos en bloque | `nube-sync` | rclone `--retries 1 --low-level-retries 2`, nextcloudcmd `--max-sync-retries 1`: lo que falle se reintenta en la próxima corrida, no repite todo. Sin `--low-level-retries`, rclone repite 10 veces cada archivo que el servidor rechaza con un 500 |
| Nunca dos a la vez | `nube-sync` | `docs` y `datos` comparten un candado; el segundo espera |
| restic sin reescribir cada noche | `restic-nube` | `forget` diario; `prune --max-unused 10%` solo los domingos |

Los valores se cambian en `/etc/almacenamiento.env` (ver `config.env.example`).

**Avisos**: un fallo suelto (un archivo que desapareció durante la sincronización, un 502
pasajero) no avisa. Avisa al **segundo fallo seguido** del mismo modo y luego cada 12;
el contador está en `~/.local/state/nube/`.

**Registro**: `nextcloudcmd` va con `--silent` y rclone con nivel `NOTICE`. Sin eso, una sola
sincronización puede escribir decenas de MB y, con un diario limitado (p. ej. 50 MB),
borrar el historial de todos los servicios en pocas horas.

### Excluir carpetas solo en esta máquina

`~/.config/nube/exclude.lst` trae las exclusiones genéricas (carpetas de compilación) y lo
instala `install.sh`. Lo propio de tu máquina va en `~/.config/nube/exclude-local.lst`
(no está en el repo): una ruta por línea, relativa a la carpeta sincronizada, por ejemplo
`Trabajo/MiProyecto`. Evita nombres sueltos si se repiten en otras carpetas.

- **Sin `/` al inicio.** `/Trabajo/MiProyecto` no excluye nada.
- `nube-sync` junta las dos listas en un archivo temporal llamado **`sync-exclude.lst`**, y el
  nombre importa. `nextcloudcmd --exclude` ancla las rutas a la carpeta donde está el archivo
  de exclusión, salvo que se llame así: entonces las ancla a la carpeta sincronizada. Con
  cualquier otro nombre no aplica **ninguna** exclusión, ni las genéricas.
  Comprobado con `nextcloudcmd` 34.

### Quitar algo del servidor sin borrarlo en tu PC

`docs` es bidireccional: si borras en el servidor una carpeta que se sigue sincronizando,
**la próxima corrida la borra también en tu PC**. Orden seguro:

1. Detén la sincronización: `systemctl --user stop nube-docs.timer`.
2. Agrega la ruta a `exclude-local.lst`.
3. Comprueba en una carpeta de prueba que la exclusión funciona y que borrar en el servidor
   una carpeta excluida no toca la local. **No te saltes este paso**: con una exclusión que
   no aplica, el borrado del servidor se propaga a tu PC sin aviso.
   Prueba: sincroniza una carpeta de prueba entera, agrega la exclusión, sincroniza,
   bórrala en el servidor, sincroniza otra vez y mira si sigue en tu PC.
4. Bórrala en el servidor, de a una carpeta si son muchas: `rclone purge nube:Documentos/<ruta>`.
   Queda en la papelera de Nextcloud unos 30 días.
5. Corre `nube-sync docs` y comprueba que la copia local sigue igual. Reactiva el timer.

## Configurar

```bash
sudo pacman -S nextcloud-client rclone restic   # Arch/CachyOS; en otras distros, instala
                                                 # los equivalentes con tu gestor de paquetes
```

1. **Contraseña de aplicación**: Nextcloud → Configuración personal → Seguridad →
   "Crear nueva contraseña de aplicación". Nunca uses la contraseña principal.
2. **Guardarla sin que quede en el historial** (fish):
   ```fish
   read -s -P 'Contraseña de aplicación: ' pw
   printf 'machine nube.ejemplo.com login USUARIO password %s\n' $pw > ~/.netrc; chmod 600 ~/.netrc
   rclone config create nube webdav url=https://nube.ejemplo.com/remote.php/dav/files/USUARIO \
       vendor=nextcloud user=USUARIO pass=$pw --obscure >/dev/null
   set -e pw
   ```
3. **Servidor para restic**: una carpeta fuera de los datos de Nextcloud, dueña de tu
   usuario SSH: `sudo install -d -o USUARIO -g USUARIO /srv/restic`.
4. **Contraseña de restic**:
   ```bash
   sudo install -d -m 700 /etc/restic
   openssl rand -base64 32 | sudo tee /etc/restic/password >/dev/null; sudo chmod 600 /etc/restic/password
   sudo cat /etc/restic/password     # cópiala a tu gestor de contraseñas
   sudo restic-nube init
   ```
5. **Cuota**: lo que subas con `nube-sync` cuenta para tu cuota de Nextcloud; restic no.
   Con la imagen Docker de linuxserver.io, `occ` vuelve a separar los argumentos, así que
   escribe el valor sin espacios:
   `docker exec nextcloud occ user:setting USUARIO files quota 250GB`.

## Cliente gráfico en modo oscuro (escritorios sin GNOME/KDE)

En gestores de ventanas como niri o sway, el cliente Qt de Nextcloud puede salir en
claro aunque tu sistema prefiera oscuro. Nextcloud trae su propio modo oscuro: solo
necesita leer la preferencia del **portal** (`org.freedesktop.appearance color-scheme`).
Comprueba que tu portal la reporta:

```bash
gdbus call --session --dest org.freedesktop.portal.Desktop \
  --object-path /org/freedesktop/portal/desktop \
  --method org.freedesktop.portal.Settings.ReadOne org.freedesktop.appearance color-scheme
# (<uint32 1>,) = oscuro
```

Nextcloud arranca por dos vías (inicio de sesión y activación D-Bus), y las dos deben
llevar `QT_QPA_PLATFORMTHEME=xdgdesktopportal`. Para eso están
[`config/nextcloud-modo-oscuro/`](../config/nextcloud-modo-oscuro/), que `install.sh` copia
a `~/.config/autostart/` y `~/.local/share/dbus-1/services/`. Así solo cambia Nextcloud y
no el resto de apps Qt.

No sirve forzar `qt6ct` con la plantilla Qt de Noctalia u otras parecidas: suelen fijar el
rol `light` en blanco y Nextcloud lo usa como fondo de sus tarjetas, así que quedan
ilegibles.

## Problemas típicos

| Síntoma | Causa | Solución |
|---|---|---|
| Subidas lentísimas con miles de archivos pequeños | Cada archivo son varias consultas a la base de datos; servidor con HDD o compartido | Ver [Cuidar el servidor](#cuidar-el-servidor); excluir repos y carpetas que cambian sin parar |
| Notificaciones de error constantes del cliente gráfico; su registro dice "The download would reduce free local disk space below the limit" | El cliente intenta bajar al disco del sistema carpetas grandes que ya están en el PC | Desmarcarlas en "Elegir qué sincronizar" (ver [Tres canales](#tres-canales)). Sus registros: `~/.config/Nextcloud/logs/` |
| restic falla solo cuando lo lanza el timer: `HOME: unbound variable` | Los servicios de root no traen `HOME` y la configuración lo usa | `restic-nube` pone `HOME=/root` si falta (ya incluido) |
| `nextcloudcmd --exclude` no excluye nada | El archivo no se llama `sync-exclude.lst`, o la ruta empieza con `/` | Ver [Excluir carpetas solo en esta máquina](#excluir-carpetas-solo-en-esta-máquina) |
| `502 Bad Gateway` | Nextcloud detenido o saturado detrás del proxy | Comprobar el contenedor; las tandas siguientes reintentan solas |
| El diario solo guarda las últimas horas | Un proceso muy verboso llena el límite del diario | `--silent` / nivel `NOTICE` (ya incluido); ver `journalctl --disk-usage` |
| `Class "WoltLab\WebpExif\ChunkType" not found` (500) al subir `.webp` | Instalación del servidor incompleta tras una actualización | `occ maintenance:repair`; si no, actualizar el contenedor |
| `nextcloudcmd` deja `.sync_*.db` en la carpeta | Es su registro de estado | Normal; no se sube |
| `occ` dice "Too many arguments" | El atajo `occ` de linuxserver separa los argumentos con espacios | Valores sin espacios (`250GB`) |
| restic: "unable to locate cache directory" | Los servicios systemd no tienen `$HOME` | `RESTIC_CACHE_DIR=/var/cache/restic` (ya incluido) |
