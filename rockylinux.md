# Manual: Cómo crear una distribución derivada de Rocky Linux con tus paquetes y configuraciones

Este manual está diseñado para administradores y entusiastas que desean construir una distribución Linux personalizada basada en Rocky Linux (o cualquier otro clon de RHEL). A diferencia de Ubuntu (Cubic) o Arch (archiso), Rocky Linux ofrece herramientas oficiales como **livecd-tools** y **lorax** para crear ISOs live e instalables, además de la flexibilidad del sistema de kickstart. Cubriremos dos enfoques principales:

- **Método A**: Usar `livecd-creator` (livecd-tools) para generar un ISO live personalizado.
- **Método B**: Personalización manual mediante chroot + `mkksiso` y `xorriso` para un control máximo.
- **Bonus**: Integración de Ansible para reproducir configuraciones completas.

Al final, tendrás un ISO arrancable que incluye tus paquetes, ajustes del sistema, usuarios y aplicaciones, listo para desplegar en múltiples equipos.

## 1. ¿Qué es una distribución derivada de Rocky Linux?

Rocky Linux es un clon binario de RHEL, con el mismo kernel, paquetes y compatibilidad. Una "derivada" es una imagen ISO personalizada que contiene:

- Paquetes adicionales (repositorios EPEL, RPM Fusion, tus propios RPMs).
- Configuraciones predefinidas (fstab, redes, servicios, SELinux).
- Scripts de postinstalación.
- Usuarios y contraseñas preconfigurados.
- Personalizaciones visuales (temas, fondos, panel).

Se usa comúnmente para estaciones de trabajo preconfiguradas en entornos corporativos, laboratorios educativos o distribuciones especializadas (ej. para científico de datos, servidores de monitoreo, etc.).

## 2. Requisitos previos

- **Sistema base**: Rocky Linux 9.x (mínimo) instalado y actualizado (`dnf update`).
- **Espacio en disco**: 25 GB libres (para la imagen de trabajo y el ISO final).
- **RAM**: 4 GB mínimo (8 GB recomendado).
- **Herramientas**:
  - `livecd-tools` (para método A)
  - `lorax`, `mkisofs`, `xorriso`, `squashfs-tools`, `genisoimage` (método B)
  - `createrepo` si necesitas crear un repositorio local de RPMs.
  - `git`, `vim`, `ansible` (opcional, para automatización).

Instalación inicial:

```bash
sudo dnf install epel-release  # EPEL es esencial para herramientas adicionales
sudo dnf install livecd-tools lorax xorriso squashfs-tools genisoimage
```

## 3. Método A: Usando `livecd-creator` (livecd-tools)

**`livecd-creator`** es la herramienta oficial para construir ISOs live en Rocky/RHEL. Utiliza un archivo de configuración (`.ks` kickstart) para definir qué paquetes incluir y cómo configurar el sistema.

### 3.1 Crear el archivo kickstart (`myrocky.ks`)

El kickstart define la instalación. Comienza copiando un ejemplo:

```bash
cp /usr/share/doc/livecd-tools/livecd-fedora-minimal.ks ~/myrocky.ks
vim ~/myrocky.ks
```

Un kickstart típico para una ISO personalizada:

```kickstart
# myrocky.ks
# Configuración básica
lang en_US.UTF-8
keyboard us
timezone America/New_York
rootpw --iscrypted $6$...   # Contraseña encriptada (genera con: grub-crypt o python -c ...)
user --name=customuser --password=mipassword --groups=wheel
selinux --enforcing
services --enabled=NetworkManager,sshd

# Repositorios
repo --name=base --baseurl=https://mirror.rockylinux.org/rocky/9/BaseOS/x86_64/os/
repo --name=appstream --baseurl=https://mirror.rockylinux.org/rocky/9/AppStream/x86_64/os/
repo --name=epel --baseurl=https://dl.fedoraproject.org/pub/epel/9/Everything/x86_64/

# Selección de paquetes
%packages
@^minimal-environment   # Grupo mínimo (equivalente a "Minimal Install" en el instalador)
@^standard              # Herramientas estándar
epel-release
vim
git
cockpit                # Administración web
firefox
# Añade aquí todos tus paquetes (puedes copiar la lista desde tu sistema actual)
%end

# Personalización del sistema en vivo
%post
# Crear directorios, copiar configuraciones, habilitar servicios
mkdir -p /etc/systemd/system/getty@tty1.service.d/
cat > /etc/systemd/system/getty@tty1.service.d/override.conf << EOF
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin customuser --noclear %I \$TERM
EOF
systemctl enable NetworkManager
# Ejemplo: copiar archivos de configuración desde un directorio local (se debe colocar en el mismo directorio del .ks)
# cp /tmp/my-config/*.conf /etc/
%end
```

**Notas importantes**:
- Reemplaza las URLs de repositorios con las oficiales o mirrors locales.
- En `%packages`, puedes listar paquetes individuales (uno por línea) o grupos (`@^nombre`).
- La sección `%post` se ejecuta dentro del chroot durante la construcción de la ISO.
- Si necesitas incluir muchos archivos personalizados, puedes usar `%post` para descargarlos (tarball) o montar un directorio compartido.

### 3.2 Construir la imagen

```bash
sudo livecd-creator --verbose \
    --config=myrocky.ks \
    --fslabel=MYROCKY \
    --cache=/var/cache/live \
    --tmpdir=/tmp/livebuild \
    --title="My Rocky Linux 9"
```

- `--fslabel` es la etiqueta del volumen en el ISO.
- `--cache` reutiliza paquetes descargados (opcional).
- `--tmpdir` espacio de trabajo temporal (debe tener al menos 10 GB).

El proceso descarga todos los paquetes, los instala en un chroot temporal, comprime el sistema en squashfs y genera el ISO. Puede tardar 20-40 minutos dependiendo de la cantidad de paquetes y velocidad de red.

Al finalizar, el ISO se crea en el directorio actual con nombre `livecd-MYROCKY.iso` (o similar).

## 4. Método B: Personalización manual (máximo control)

Este método te permite modificar una ISO existente de Rocky Linux o construir desde cero usando `lorax` y `xorriso`. Es más complejo pero ofrece control granular.

### 4.1 Extraer una ISO oficial de Rocky

```bash
mkdir ~/rocky-iso && cd ~/rocky-iso
wget https://download.rockylinux.org/pub/rocky/9/isos/x86_64/Rocky-9.4-x86_64-minimal.iso   # o la versión que uses
# Montar y extraer
sudo mount -o loop Rocky-9.4-x86_64-minimal.iso /mnt
sudo rsync -a /mnt/ ./
sudo umount /mnt
```

### 4.2 Personalizar el contenido

El sistema en vivo de Rocky (si es una ISO live) está en un archivo squashfs dentro de `LiveOS/squashfs.img` (o `rootfs.img`). Pero para una ISO de instalación (como la minimal), el contenido es diferente (contiene una imagen de instalación, no un sistema live).

**Para una ISO live**: el directorio `LiveOS` contiene `squashfs.img`. Debes extraerlo:

```bash
mkdir squashfs-root
sudo unsquashfs LiveOS/squashfs.img
# Esto crea squashfs-root/ con el sistema raíz
```

Luego puedes modificar el sistema chroot:

```bash
sudo mount --bind /dev squashfs-root/dev
sudo mount --bind /proc squashfs-root/proc
sudo mount --bind /sys squashfs-root/sys
sudo cp /etc/resolv.conf squashfs-root/etc/resolv.conf
sudo chroot squashfs-root /bin/bash
```

Dentro del chroot:

```bash
export HOME=/root
export LC_ALL=C
# Configurar repositorios (si es necesario)
# Instalar/eliminar paquetes
dnf install -y paquete1 paquete2
dnf remove -y paquete3
# Aplicar configuraciones, crear usuarios, etc.
# ...
exit
```

Luego, reempaquetar el squashfs:

```bash
sudo rm LiveOS/squashfs.img
sudo mksquashfs squashfs-root LiveOS/squashfs.img -comp xz -b 1M
# Opcional: regenerar archivos de metadatos (md5sum.txt, etc.)
```

Finalmente, regenerar el ISO:

```bash
sudo genisoimage -r -T -J -l -d -N \
    -b isolinux/isolinux.bin \
    -c isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    -eltorito-alt-boot -e images/efiboot.img -no-emul-boot \
    -isohybrid-mbr /usr/lib/syslinux/isohdpfx.bin \
    -V "MYROCKY" \
    -o ~/my-rocky-custom.iso \
    .
```

**Para una ISO de instalación** (no live): La personalización se hace a través del archivo `kickstart` incrustado en la imagen, y luego se regenera la ISO usando `mkksiso` (parte de lorax).

### 4.3 Personalización mediante Kickstart + `mkksiso`

`mkksiso` inyecta un nuevo archivo kickstart en una ISO de instalación oficial (como la DVD o Minimal). Es el método más simple si solo necesitas cambiar paquetes y configuraciones de instalación (no modificar el sistema live).

```bash
# Descargar ISO oficial
wget https://download.rockylinux.org/pub/rocky/9/isos/x86_64/Rocky-9.4-x86_64-minimal.iso

# Crear un kickstart personalizado (como en 3.1)
vim my-kickstart.ks

# Inyectar y generar nueva ISO
mkksiso my-kickstart.ks Rocky-9.4-x86_64-minimal.iso ~/rocky-custom.iso
```

**Ventaja**: No necesitas extraer ni re-comprimir nada; `mkksiso` genera una nueva ISO lista para arrancar y que usará tu kickstart.

## 5. Cómo incluir tus paquetes y configuraciones existentes

### 5.1 Listar paquetes instalados en tu sistema actual

```bash
rpm -qa --queryformat '%{NAME}\n' | sort > ~/lista-paquetes.txt
```

Luego, en tu kickstart, puedes incluir esa lista. Si usas `%packages`, simplemente copia el contenido entre las marcas. Si usas el método de chroot, ejecuta `dnf install` desde el archivo.

### 5.2 Copiar configuraciones del sistema

- **Archivos del sistema**: `/etc/hosts`, `/etc/fstab`, `/etc/sysconfig/network-scripts/`, `/etc/ssh/sshd_config`.
- **Configuraciones de usuario**: `/home/tu-usuario/.bashrc`, `.config`, `.local` (si quieres que el usuario predefinido las herede).
- **Servicios**: units de systemd en `/etc/systemd/system/`.
- **Repositorios adicionales**: `/etc/yum.repos.d/` (EPEL, RPM Fusion, etc.).

En un kickstart, puedes usar `%post` para copiar archivos desde un directorio `%include` o descargarlos. Por ejemplo:

```bash
%post --log=/root/install.log
# Copiar configuraciones desde un repositorio git
dnf install -y git
git clone https://mi-repo/configs.git /tmp/configs
cp -r /tmp/configs/etc/* /etc/
%end
```

### 5.3 Sincronizar repositorios locales

Si tienes RPMs propietarios o compilados, crea un repositorio local:

```bash
mkdir -p ~/myrpms
# Copia tus RPMs allí
createrepo ~/myrpms
```

Luego, en el kickstart añade:

```kickstart
repo --name=myrepo --baseurl=file:///root/myrpms   # o en servidor http
```

Y en `%packages` lista el nombre de tu RPM.

### 5.4 Incluir configuraciones de usuario predefinidas

Puedes crear un archivo `/etc/skel` con los dotfiles deseados (`.bashrc`, `.config`, etc.). También puedes, en el `%post`, copiar todo el directorio `/home/tu-usuario` a `/home/customuser` y luego arreglar permisos.

## 6. Prueba del ISO

Arranca el ISO en una máquina virtual (VirtualBox, KVM) o en un equipo real:

```bash
# Usando qemu
qemu-system-x86_64 -m 2048 -cdrom ~/rocky-custom.iso -boot d
```

Verifica:

- Arranque correcto (BIOS y UEFI).
- Los paquetes preinstalados están presentes.
- Las configuraciones (red, teclado, usuario) funcionan.
- Si es ISO de instalación (no live), prueba el proceso de instalación automática con el kickstart.

## 7. Consideraciones legales y de marca

- **Rocky Linux** es software libre (GPL). Puedes redistribuirlo sin restricciones, pero no uses el nombre "Rocky Linux" como marca de tu distribución (debes indicar que está basado en Rocky Linux).
- **Marcas registradas**: El logo de Rocky Linux (la piedra) puede usarse con fines informativos, pero no como parte de tu nombre o logo principal sin permiso.
- **Licencias de paquetes**: Respetar licencias (GPL, MIT, etc.). No incluyas controladores propietarios (como NVIDIA) sin autorización; mejor incluye el RPM `nvidia-detect` o guía al usuario.
- **Atribuciones**: Incluye un archivo `/etc/product-id` o similar con créditos.

## 8. Solución de problemas comunes

| Problema | Causa posible | Solución |
|----------|---------------|----------|
| `livecd-creator` falla con "cannot find repo" | Kickstart mal escrito o URLs incorrectas | Verifica las URLs de los repositorios; usa https://mirror.rockylinux.org/rocky/$releasever/... |
| El ISO live no inicia sesión automática | Falta configuración de agetty | En `%post`, crea el override para autologin. |
| La ISO es demasiado grande (>2 GB) | Demasiados paquetes o archivos innecesarios | Reduce el grupo de paquetes (usa solo `core`), elimina documentación y locales con `--excludedocs`. |
| Error de arranque UEFI | Falta imagen EFI en el ISO | Al regenerar con `genisoimage`, incluye `-eltorito-alt-boot -e images/efiboot.img`. |
| Los paquetes personalizados no se instalan | Dependencias no resueltas | Asegúrate de que los RPMs estén firmados y tengan las dependencias presentes en los repos incluidos. |

## 9. Consejos finales

- **Automatiza con Ansible**: Define un playbook que configure todo el sistema (instalación de paquetes, ajustes de seguridad, usuarios) y úsalo dentro del `%post` (instalando Ansible en el chroot).
- **Usa contenedores**: Para entornos CI/CD, puedes construir la ISO dentro de un contenedor Docker con Rocky Linux como base, evitando modificar tu sistema host.
- **Mantén un repositorio Git** de tu kickstart y scripts de configuración; así puedes versionar y compartir tu distribución.
- **Actualiza periódicamente**: Rocky Linux recibe actualizaciones de seguridad. Reconstruye tu ISO cada 1-2 meses para incluir los últimos parches.
- **Prueba en hardware real**: La virtualización puede ocultar problemas de controladores. Prueba en un equipo comparable a tu objetivo de despliegue.

---

Con este manual, tienes las herramientas para crear tu propia distribución basada en Rocky Linux, empaquetando todo tu ecosistema en un solo ISO. Ya sea para desplegar estaciones de trabajo estandarizadas, servidores preconfigurados o distribuciones educativas, el proceso te da control total y reproducibilidad. 
