# Replicar en una máquina nueva

Orden recomendado. Cada paso se puede verificar antes de seguir.

## 1. Paquetes

En Arch/CachyOS:

```bash
sudo pacman -S btrfs-progs btrbk snapper smartmontools pacman-contrib \
               restic rclone nextcloud-client python jq socat \
               libimobiledevice android-tools dysk gdu btop compsize
```

En otra distribución, instala los equivalentes con tu gestor de paquetes:

- Imprescindibles: `smartmontools`, `python3`, `jq`, `socat`, `restic`, `rclone`,
  `nextcloud-client` (o el cliente de escritorio), `libimobiledevice` (`idevicebackup2`),
  `android-tools` (`adb`).
- Solo si usas btrfs (ver [arquitectura.md](arquitectura.md)): `btrfs-progs`, `btrbk`.
- `snapper` y `pacman-contrib` (paquete de `paccache`) son de Arch/openSUSE: omítelos
  fuera de esas distribuciones; la capa de snapshots locales del sistema (paso 4) no
  tiene equivalente directo en otras, revisa [decisiones.md](decisiones.md).
- `dysk`, `gdu`, `btop`, `compsize` son utilidades opcionales, no dependencias.

## 2. Discos y montajes

1. Decide el rol de cada disco ([arquitectura.md](arquitectura.md)).
2. Crea los sistemas de archivos. **Esto borra todo lo que haya en el disco.** Usa siempre
   `/dev/disk/by-id/...` en lugar de `/dev/sdX`, porque los nombres cambian:
   ```bash
   ls -l /dev/disk/by-id/
   sudo wipefs -a /dev/disk/by-id/<disco>
   sudo parted -s /dev/disk/by-id/<disco> mklabel gpt mkpart datos btrfs 1MiB 100%
   sudo mkfs.btrfs -L datos /dev/disk/by-id/<disco>-part1
   ```
3. Para el disco de VMs, crea el subvolumen `@vm` y móntalo con `nodatacow`.
4. Agrega las líneas de [`etc/fstab.ejemplo`](../etc/fstab.ejemplo) con tus UUID
   (`lsblk -o NAME,UUID,FSTYPE`).
5. Comprueba antes de reiniciar: `sudo findmnt --verify` → `Success`, y luego
   `sudo mount -a`.

## 3. Herramientas

```bash
git clone https://github.com/BrahiamSSS/almacenamiento-linux && cd almacenamiento-linux
./install.sh                # revisa la lista
./install.sh --apply
sudoedit /etc/almacenamiento.env
```

## 4. Configuración de sistema (manual, depende de la máquina)

```bash
# btrbk
sudo cp etc/btrbk.conf /etc/btrbk/btrbk.conf && sudoedit /etc/btrbk/btrbk.conf   # ajusta target
sudo mkdir -p /mnt/btr_pool/@btrbk_snaps /mnt/datos/btrbk
sudo btrbk dryrun

# smartd: reemplaza la línea DEVICESCAN por la de etc/smartd.conf.linea
sudoedit /etc/smartd.conf && sudo systemctl enable --now smartd

# snapper y pacman
sudo snapper -c root set-config NUMBER_LIMIT=10
sudo systemctl enable --now paccache.timer
```

## 5. Secretos

Ver [nube.md](nube.md#configurar): contraseña de aplicación de Nextcloud (`~/.netrc` y
rclone), carpeta y contraseña de restic, llave SSH. Ninguno va en el repo.

## 6. Activar timers

```bash
systemctl --user enable --now disk-alert.timer nube-docs.timer nube-datos.timer
sudo systemctl enable --now disk-health-dump.timer btrbk.timer restic-nube.timer
```

La primera vez, lanza las tareas a mano y revisa que terminen bien:

```bash
sudo systemctl start disk-health-dump && discos
sudo btrbk run --progress
sudo systemctl start --no-block restic-nube && sudo journalctl -fu restic-nube
nube-sync docs
```

## 7. Verificación final

| Prueba | Esperado |
|---|---|
| `discos` | todos los discos, veredicto general en la primera línea |
| `disk-alert 1` | una notificación por cada montaje |
| `sudo env SMARTD_SUBJECT=Prueba /usr/local/bin/smartd-notify` | una notificación |
| `sudo btrbk list backups` | snapshots en el disco de datos |
| `sudo restic-nube cmd snapshots` | al menos 1 snapshot |
| Restaurar un archivo con restic y compararlo (`cmp`) | idéntico |
| `./install.sh` | "Todo al día." |
