# Manual: Cómo crear una distribución derivada de Arch Linux con tus paquetes y configuraciones

Este manual está dirigido a usuarios avanzados que desean crear una distribución Linux personalizada basada en Arch Linux. Arch, con su modelo *rolling release* y su herramienta `archiso`, permite construir ISOs live e instalables que incluyan exactamente los paquetes, configuraciones del sistema, aplicaciones de escritorio y preferencias de usuario que ya tienes. Cubriremos desde la instalación de `archiso` hasta la generación del ISO final, con énfasis en la personalización profunda.

A diferencia de Ubuntu (donde Cubic simplifica el proceso), en Arch el control total está en tus manos mediante perfiles de construcción. Este manual asume que ya tienes una instalación funcional de Arch Linux y deseas empaquetarla. Si partes desde cero, deberás primero instalar Arch y configurarlo a tu gusto.

## 1. ¿Qué es `archiso` y cómo funciona?

`archiso` es un conjunto de scripts que construyen imágenes ISO de Arch Linux. Utiliza **perfiles** (directorios con archivos de configuración) que definen:

- Qué paquetes incluir (lista en `packages.x86_64`).
- Cómo configurar el sistema en vivo (scripts en `airootfs/`).
- Personalizaciones de arranque (grub, syslinux, etc.).

El proceso completo:
1. Se crea un sistema raíz temporal usando `pacstrap` (similar a instalar Arch en un chroot).
2. Se aplican configuraciones personalizadas.
3. Se comprime ese sistema raíz en un archivo squashfs.
4. Se empaqueta junto con el cargador de arranque en un ISO.

## 2. Requisitos previos

- **Sistema base**: Arch Linux actualizado (cualquier versión, rolling).
- **Espacio en disco**: al menos 10 GB libres (para el perfil, el chroot temporal y el ISO final).
- **RAM**: 4 GB mínimo (8 GB recomendado).
- **Herramientas**: `archiso` (el paquete principal) y opcionalmente `git`, `vim`, `base-devel` para compilar desde AUR si es necesario.

Instalación de `archiso`:

```bash
sudo pacman -S archiso
```

## 3. Estructura de un perfil de construcción

Los perfiles oficiales de Archiso se encuentran en `/usr/share/archiso/configs/releng/` (para la ISO live estándar). Vamos a copiarlo como plantilla y modificarlo.

```bash
cp -r /usr/share/archiso/configs/releng/ ~/myarch-profile
cd ~/myarch-profile
```

La estructura principal:

```
myarch-profile/
├── airootfs/               # Sistema de archivos raíz adicional (copia sobre el sistema base)
├── efiboot/                # Contenido para arranque UEFI (opcional)
├── grub/                   # Configuración de GRUB
├── isolinux/               # Configuración de isolinux (BIOS)
├── syslinux/               # Configuración de syslinux (alternativa)
├── pacman.conf             # Configuración de pacman para el entorno de construcción
├── packages.x86_64         # Lista de paquetes a instalar (un paquete por línea)
├── packages.x86_64.sig     # Firma GPG (opcional)
├── profiledef.sh           # Configuración principal del perfil (nombre, versión, etc.)
└── mkarchiso               # Script de construcción (no modificar)
```

## 4. Personalización básica del perfil

### 4.1 Editar `profiledef.sh`

Define la identidad de tu distribución:

```bash
vim profiledef.sh
```

```bash
#!/usr/bin/env bash
# shellcheck disable=SC2034

iso_name="myarch"
iso_label="MYARCH_$(date +%Y%m)"
iso_publisher="Mi Distribución"
iso_application="My Arch Linux Custom Live/Rescue CD"
iso_version="$(date +%Y.%m.%d)"
install_dir="arch"
buildmodes=('iso')
bootmodes=('bios.syslinux.mbr' 'bios.syslinux.eltorito'
           'uefi-x64.systemd-boot.esp' 'uefi-x64.systemd-boot.eltorito')
arch="x86_64"
pacman_conf="pacman.conf"
airootfs_image_type="squashfs"
airootfs_image_tool_options=('-comp' 'xz' '-Xbcj' 'x86' '-b' '1M')
file_permissions=(
  ["/etc/shadow"]="0:0:400"
  ["/etc/gshadow"]="0:0:400"
  ["/root"]="0:0:750"
  ["/root/.automated_script.sh"]="0:0:755"
  ["/usr/local/bin/my-init"]="0:0:755"   # ejemplo
)
```

Cambia `iso_name`, `iso_label`, `iso_publisher`, `iso_application`, `iso_version` según prefieras. Los `bootmodes` garantizan compatibilidad con BIOS y UEFI.

### 4.2 Lista de paquetes (`packages.x86_64`)

Aquí pondrás los paquetes que quieres preinstalados. Para reflejar tu sistema actual, genera la lista:

```bash
pacman -Qqen > packages.x86_64   # paquetes explícitamente instalados (oficiales)
pacman -Qqem >> packages.x86_64  # paquetes del AUR (opcional, ver sección 5.3)
```

**Advertencia**: Los paquetes del AUR no se instalan directamente con `pacstrap`. Debes manejarlos aparte (ver más adelante). Por ahora, comenta las líneas correspondientes a paquetes AUR o elimínalas.

Asegúrate de que la lista incluya al menos:

- `base` (metapaquete)
- `base-devel` (si compilarás AUR dentro del ISO)
- `linux` (kernel)
- `linux-firmware` (firmware)
- `grub`, `efibootmgr` (si quieres instalador)
- `sudo`, `vim`, `git` (herramientas básicas)
- Tu escritorio (`gnome`, `plasma`, `xfce4`, etc.)
- tus aplicaciones favoritas

Puedes añadir repositorios adicionales en `pacman.conf` (ej. `[multilib]`, `[archlinuxcn]`).

### 4.3 Configuración de `pacman.conf` para la construcción

Copia tu `pacman.conf` del sistema o edita el que viene en la plantilla. Asegúrate de que los repositorios estén habilitados, especialmente `multilib` si lo necesitas.

```bash
# Habilitar multilib (descomentar)
[multilib]
Include = /etc/pacman.d/mirrorlist
```

## 5. Personalización del sistema en vivo (airootfs)

El directorio `airootfs/` se copia sobre el sistema raíz después de `pacstrap`. Puedes agregar archivos, scripts y configuraciones que se aplicarán al entorno live (y también estarán presentes si se instala).

### 5.1 Copiar configuraciones del sistema actual

Por ejemplo, para incluir tus configuraciones de red, usuarios, servicios y preferencias de escritorio:

```bash
# Crear directorios necesarios dentro de airootfs
mkdir -p airootfs/etc
mkdir -p airootfs/usr/local/bin
mkdir -p airootfs/home/liveuser   # usuario predeterminado

# Copiar archivos relevantes
sudo cp /etc/NetworkManager/system-connections/* airootfs/etc/NetworkManager/system-connections/ 2>/dev/null
sudo cp /etc/pacman.d/mirrorlist airootfs/etc/pacman.d/
sudo cp /etc/ssh/sshd_config airootfs/etc/ssh/ 2>/dev/null
```

**No copies claves SSH privadas ni datos sensibles** (a menos que sea intencional para tu distribución privada).

### 5.2 Configurar usuario por defecto

Por defecto, la ISO de releng crea un usuario `root` sin contraseña y un usuario `liveuser` (contraseña vacía). Puedes modificar los scripts de inicio para crear un usuario con contraseña personalizada.

Edita o crea `airootfs/etc/systemd/system/getty@tty1.service.d/autologin.conf` para autologin:

```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -f -- \\u' --noclear --autologin liveuser %I $TERM
```

Luego, en `airootfs/root/customize_airootfs.sh` (o un script propio) configura la contraseña:

```bash
#!/bin/bash
# Se ejecuta al final de la construcción del sistema raíz
useradd -m -G wheel -s /bin/bash liveuser
echo "liveuser:mi_contraseña" | chpasswd
# Copiar configuraciones de usuario si existen
cp -r /etc/skel/. /home/liveuser/
chown -R liveuser:liveuser /home/liveuser
```

No olvides dar permisos de ejecución a este script y referenciarlo en `profiledef.sh` dentro de `file_permissions`:

```
["/root/customize_airootfs.sh"]="0:0:755"
```

O puedes ubicarlo en `/usr/local/bin/` y ejecutarlo con un servicio systemd.

### 5.3 Incluir paquetes del AUR

Los paquetes del AUR no se pueden instalar directamente con `pacstrap` porque requieren compilación (o instalación desde binarios). Opciones:

1. **Precompilarlos en tu sistema y copiar los paquetes `.pkg.tar.zst`** a `airootfs/`. Luego, en `customize_airootfs.sh`, instálalos con `pacman -U`.

2. **Usar un repositorio local** (más avanzado): Crea un repositorio con tus paquetes AUR compilados y añádelo a `pacman.conf` durante la construcción.

3. **Incluir `yay`/`paru` y compilar al primer arranque** (no recomendado para ISO, pues alarga el tiempo de inicio y requiere red).

**Ejemplo práctico (opción 1)**:

```bash
# En tu sistema actual, compila un paquete AUR (ej. google-chrome)
cd /tmp
git clone https://aur.archlinux.org/google-chrome.git
cd google-chrome
makepkg -si   # lo instala en el sistema; también produce .pkg.tar.zst
# Copia el archivo generado a tu perfil
cp *.pkg.tar.zst ~/myarch-profile/airootfs/root/packages-aur/
```

Luego, dentro de `customize_airootfs.sh`:

```bash
cd /root/packages-aur
for pkg in *.pkg.tar.zst; do
    pacman -U --noconfirm "$pkg"
done
```

### 5.4 Configurar servicios y personalizaciones

- **Habilitar servicios**: Puedes crear un script que ejecute `systemctl enable NetworkManager`, `sshd`, etc.
- **Tema, iconos, fondo de pantalla**: Copia temas a `/usr/share/themes/`, iconos a `/usr/share/icons/`, fondos a `/usr/share/backgrounds/` y configura los defaults en `/etc/skel/.config/`.
- **Firewall**: `systemctl enable ufw` y configura reglas.

### 5.5 Script de personalización automatizado

Crea un script `airootfs/root/customize_airootfs.sh` con todas las configuraciones. Ten en cuenta que se ejecuta **durante la construcción del ISO**, no en cada arranque. Si necesitas configurar algo al primer arranque del live, usa systemd oneshots.

Ejemplo completo:

```bash
#!/bin/bash
# customize_airootfs.sh

# 1. Configurar locale y teclado
echo "es_ES.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=es_ES.UTF-8" > /etc/locale.conf
echo "KEYMAP=es" > /etc/vconsole.conf

# 2. Crear usuario personalizado con contraseña
useradd -m -G wheel,storage,power -s /bin/bash miusuario
echo "miusuario:miclave123" | chpasswd
# Copiar skel
cp -r /etc/skel/. /home/miusuario/
chown -R miusuario:miusuario /home/miusuario

# 3. Instalar paquetes adicionales (locales)
for pkg in /root/pkgs/*.pkg.tar.zst; do
    pacman -U --noconfirm "$pkg"
done

# 4. Habilitar servicios
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable cups

# 5. Eliminar paquetes basura (si quieres)
pacman -Rns --noconfirm xscreensaver  # ejemplo

# 6. Limpiar
rm -rf /root/pkgs
```

## 6. Generar el ISO

Una vez que el perfil esté listo, ejecuta:

```bash
sudo mkarchiso -v -w /tmp/archiso-tmp -o ~/myarch-profile/out ~/myarch-profile/
```

- `-w` especifica el directorio de trabajo (puede ser en `/tmp` o en otro lugar con suficiente espacio).
- `-o` directorio de salida para el ISO.
- `-v` verbose.

El proceso toma entre 5 y 15 minutos, dependiendo de la cantidad de paquetes. Al final, tendrás un archivo como `~/myarch-profile/out/myarch-2025.04.15-x86_64.iso`.

## 7. Prueba del ISO

1. Arranca el ISO en una máquina virtual (VirtualBox, QEMU, etc.) o en un equipo real.
2. Verifica que los paquetes estén presentes (`pacman -Q`).
3. Prueba el acceso a red, sonido, dispositivos.
4. Si tienes un instalador (`calamares` o script propio), pruébalo.

## 8. Opciones avanzadas

### 8.1 Incluir un instalador gráfico (Calamares)

Puedes agregar el instalador `calamares` (común en distribuciones derivadas de Arch). Incluye el paquete `calamares` y sus módulos de configuración. Luego, crea un archivo de configuración en `/etc/calamares/` para personalizar las páginas.

### 8.2 Firmar el ISO (verificación de integridad)

```bash
gpg --detach-sign --armor myarch-*.iso
# También genera un checksum
sha256sum myarch-*.iso > myarch-*.iso.sha256
```

### 8.3 Construcción automatizada con CI/CD

Puedes usar GitHub Actions o GitLab CI para reconstruir automáticamente el ISO tras actualizar la lista de paquetes. Ejemplo básico:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build ISO
        run: |
          docker run --privileged -v $PWD:/work archlinux:latest bash -c "
            pacman -Syu --noconfirm archiso
            cd /work
            mkarchiso -v -w /tmp/archiso-tmp -o out .
          "
```

(Requiere configurar Docker y permisos.)

## 9. Consideraciones legales y de marca

- **Nombre**: Al ser una distribución derivada de Arch, puedes usar "Arch Linux" en la descripción, pero evita nombres que puedan confundir con la oficial (por ejemplo, no llames a tu ISO "Arch Linux" a secas).
- **Logos**: El logo de Arch Linux (el arco) está bajo licencia MIT; puedes usarlo si lo deseas.
- **Licencias**: Respetar las licencias de los paquetes incluidos (GPL, MIT, etc.). No redistribuyas software propietario sin permiso (como algunos controladores NVIDIA privativos; mejor usar `nvidia-dkms` de AUR).
- **Atribuciones**: Incluye un archivo `/etc/arch-release` personalizado (opcional) y créditos en el menú de arranque.

## 10. Solución de problemas comunes

| Problema | Causa posible | Solución |
|----------|---------------|----------|
| `mkarchiso` falla con "could not find any database" | No se encontró pacman.conf o falta repo | Verifica que `pacman.conf` en el perfil tenga repositorios habilitados y mirrorlist válido. |
| El sistema en vivo no tiene red | Faltan firmwares o NetworkManager no inició | Asegúrate de incluir `linux-firmware`, `iwlwifi` y `NetworkManager` en packages. |
| El usuario live no puede hacer sudo | No está en grupo wheel | En `customize_airootfs.sh` añade `usermod -aG wheel liveuser` y descomenta `%wheel ALL=(ALL) ALL` en `/etc/sudoers`. |
| El ISO es demasiado grande | Muchos paquetes o squashfs sin comprimir | Usa `-comp xz` o `-comp zstd` en `profiledef.sh`. También puedes eliminar paquetes innecesarios. |
| Error de arranque UEFI | Falta `systemd-boot` o archivos de firmware | Asegura que `bootmodes` incluya `uefi-x64.systemd-boot.esp`. |

## 11. Consejos finales

- **Mantén un repositorio Git** para tu perfil (`git init`). Así puedes versionar los cambios.
- **Actualiza periódicamente**: Dado que Arch es rolling release, tu ISO caduca rápidamente. Reconstruye cada 1-2 meses para tener paquetes recientes.
- **Usa `overlayroot`** si necesitas que los cambios realizados en la sesión live se guarden (opcional).
- **Para distribuciones empresariales**: Considera usar `archiso` combinado con `docker` para construir en contenedores y garantizar reproducibilidad.

---

Con este manual, tienes una guía completa para crear tu propia distribución Arch Linux personalizada. Ya sea para tener un "sistema portátil" en un USB, para distribuir a colegas o para aprender, el proceso te da control total sobre cada aspecto. 
