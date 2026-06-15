# Manual: Cómo crear una distribución derivada de Ubuntu con tus paquetes y configuraciones

Este manual está dirigido a usuarios avanzados que desean crear una distribución Linux personalizada basada en Ubuntu, incluyendo todos los paquetes instalados, configuraciones del sistema, aplicaciones de escritorio y preferencias de usuario. Cubriremos dos métodos principales: **Cubic** (recomendado para simplicidad) y **personalización manual con chroot** (mayor control). Al final, tendrás un ISO instalable listo para usar en otros equipos o para compartir.

## 1. ¿Qué es una distribución derivada y por qué crearla?

- **Distribución derivada**: una versión modificada de Ubuntu que conserva su base (repositorios, núcleo, estructura) pero incorpora paquetes, temas, scripts y configuraciones propias.
- **Motivos comunes**:
  - Preinstalar software específico para un proyecto o empresa.
  - Tener un sistema "clon" portable con todas tus herramientas.
  - Ofrecer una experiencia preconfigurada a usuarios finales (kioscos, laboratorios, estaciones de trabajo).
  - Aprender sobre el proceso de construcción de distribuciones.

## 2. Requisitos previos

- **Sistema base**: Ubuntu 22.04 LTS o 24.04 LTS (recomendado por estabilidad).
- **Espacio en disco**: al menos 25 GB libres (para la imagen de trabajo y el ISO final).
- **RAM**: 4 GB mínimo (8 GB recomendado).
- **Herramientas básicas**: `git`, `make`, `wget`, `curl`, `p7zip-full`, `squashfs-tools`, `xorriso`, `gpg`.
- **Opcional**: máquina virtual para pruebas (VirtualBox/VMware).

## 3. Método A: Usando Cubic (Personal Graphical ISO Creator)

**Cubic** es una herramienta gráfica que simplifica la remasterización de Ubuntu. Permite modificar el sistema en vivo mediante una shell chroot.

### 3.1 Instalación de Cubic

```bash
sudo add-apt-repository ppa:cubic-wizard/release
sudo apt update
sudo apt install cubic
```

### 3.2 Preparación del proyecto

1. Ejecuta Cubic desde el menú o terminal (`sudo cubic`).
2. Selecciona "New Project". Escoge un directorio de trabajo (ej. `/home/usuario/custom-ubuntu`).
3. **Archivo ISO de origen**: proporciona la ISO oficial de Ubuntu (descárgala previamente).
4. **Nombre de la distribución**: el que quieras (ej. "MyUbuntu").
5. **Versión**: mantén la misma que la ISO base.
6. **Tipo**: "Desktop" o "Server" según tu caso.

Cubic extraerá el contenido de la ISO en el directorio de trabajo y preparará el entorno chroot.

### 3.3 Personalización en chroot

Al finalizar la extracción, Cubic abrirá una terminal en **modo chroot** simulando el sistema de destino. Aquí puedes:

- **Instalar/eliminar paquetes** (usando `apt`).
  ```bash
  apt update
  apt install mi-paquete-favorito
  apt remove firefox  # si no lo quieres
  ```

- **Copiar configuraciones del sistema host** (desde otra terminal).
  - Copia archivos de `/etc` relevantes (network, repositorios, users).
  - Si quieres incluir tu usuario por defecto, crea uno:
    ```bash
    useradd -m -G sudo -s /bin/bash miusuario
    passwd miusuario
    ```

- **Importar paquetes de PPAs o fuentes externas**:
  ```bash
  add-apt-repository ppa:mi-ppa
  apt update
  apt install paquete-desde-ppa
  ```

- **Configurar entornos de escritorio** (GNOME, KDE, XFCE, etc.):
  ```bash
  apt install ubuntu-desktop  # para GNOME completo
  ```

- **Aplicar personalizaciones visuales** (temas, iconos, fondo de pantalla).
  Copia archivos a `/usr/share/backgrounds` y ajusta `/usr/share/glib-2.0/schemas/`.

- **Configurar servicios** (NetworkManager, cups, firewall).
  ```bash
  systemctl enable cups
  ```

### 3.4 Configuración adicional fuera del chroot

Después de salir del chroot (escribe `exit`), Cubic te presentará opciones para:

- **Nombre del ISO**, **ID de distribución**, **versión**, **descripción**.
- **Idioma**, **zona horaria**, **teclado**.
- **Contraseña de root** (dejar en blanco para usar sudo).
- **Usuario por defecto** que se creará en la instalación (opcional).
- **Particionado automático** (LVM, cifrado, etc.).

### 3.5 Generación del ISO

1. Haz clic en "Generate". Cubic comprimirá el sistema en un squashfs, creará la estructura de arranque y generará el ISO.
2. El proceso puede tardar entre 10 y 30 minutos.
3. Al finalizar, el ISO se guardará en el directorio de proyecto con un nombre como `MyUbuntu-24.04-desktop-amd64.iso`.

## 4. Método B: Personalización manual con chroot

Este método ofrece control total y no depende de Cubic. Es ideal para scripting o integración en CI/CD.

### 4.1 Preparación del entorno de trabajo

```bash
mkdir ~/ubuntu-custom && cd ~/ubuntu-custom
# Descargar ISO oficial (ej. Ubuntu 24.04)
wget https://releases.ubuntu.com/24.04/ubuntu-24.04-desktop-amd64.iso
# Extraer contenido
mkdir iso && sudo mount -o loop ubuntu-24.04-desktop-amd64.iso mnt
sudo rsync -a mnt/ iso/
sudo umount mnt
```

### 4.2 Extraer y modificar el sistema squashfs

```bash
# El sistema en vivo está comprimido en squashfs
cd iso/casper
sudo unsquashfs filesystem.squashfs
# El directorio squashfs-root contiene el sistema completo
```

### 4.3 Personalización mediante chroot

```bash
sudo mount --bind /dev squashfs-root/dev
sudo mount --bind /proc squashfs-root/proc
sudo mount --bind /sys squashfs-root/sys
sudo cp /etc/resolv.conf squashfs-root/etc/resolv.conf
sudo chroot squashfs-root
```

Dentro del chroot:

```bash
export HOME=/root
export LC_ALL=C
apt update
# Ahora instala, elimina, configura igual que en Cubic
# ...
```

Al salir:

```bash
exit
sudo umount squashfs-root/dev squashfs-root/proc squashfs-root/sys
```

### 4.4 Reconstruir squashfs y regenerar ISO

```bash
# Eliminar el squashfs original
sudo rm iso/casper/filesystem.squashfs
# Crear el nuevo
sudo mksquashfs squashfs-root iso/casper/filesystem.squashfs -comp xz -b 1M
# Actualizar manifest (lista de paquetes)
chmod +w iso/casper/filesystem.manifest
sudo chroot squashfs-root dpkg-query -W --showformat='${Package} ${Version}\n' > iso/casper/filesystem.manifest
cp iso/casper/filesystem.manifest iso/casper/filesystem.manifest-desktop
```

### 4.5 Generar el ISO arrancable

```bash
cd ~/ubuntu-custom/iso
# Generar archivos de arranque (GRUB)
sudo grub-mkrescue -o ~/ubuntu-custom/my-ubuntu-custom.iso .
```

Alternativamente, usando `xorriso` directamente (más control):

```bash
xorriso -as mkisofs -r -V "My Ubuntu" -J -l -b isolinux/isolinux.bin \
  -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table \
  -isohybrid-mbr /usr/lib/syslinux/isohdpfx.bin -eltorito-alt-boot \
  -e boot/grub/efi.img -no-emul-boot -isohybrid-gpt-basdat \
  -o ~/ubuntu-custom/custom.iso .
```

## 5. Cómo incluir tus configuraciones y paquetes existentes

### 5.1 Listar paquetes instalados en tu sistema actual

```bash
dpkg -l | grep "^ii" | awk '{print $2}' > ~/paquetes-lista.txt
```

Luego, dentro del chroot, puedes instalar todos con:

```bash
xargs -a ~/paquetes-lista.txt apt install -y
```

**Atención**: algunos paquetes pueden no estar disponibles en los repos por defecto; deberás añadir PPAs manualmente.

### 5.2 Copiar configuraciones del sistema

Copia los archivos de configuración relevantes desde tu sistema host al entorno chroot. Por ejemplo:

- **Archivos de red**: `/etc/NetworkManager/system-connections/` (si quieres que las conexiones WiFi se hereden).
- **Repositorios**: `/etc/apt/sources.list` y `/etc/apt/sources.list.d/*`.
- **Configuraciones de usuario**: `/home/tu-usuario/.config`, `.bashrc`, `.profile`.
- **Servicios personalizados**: systemd units en `/etc/systemd/system/`.
- **Scripts de inicio**: `/usr/local/bin/`, `/etc/init.d/`.

**Importante**: para mantener la privacidad, evita incluir claves SSH, tokens o datos sensibles.

### 5.3 Sincronizar PPAs, snaps y flatpaks

- **PPAs**: exportar lista con `apt-cache policy | grep http | awk '{print $2}' | sort -u > ppas.txt`. Luego, en el chroot, añadir cada uno con `add-apt-repository`.
- **Snaps**: si usas snaps, puedes precargarlas mediante `snap download` (pero no se instalará automáticamente; deberías empaquetar los snaps como archivos `.snap` y dejarlos en el sistema).
- **Flatpaks**: `flatpak list --app` y luego `flatpak install` dentro del chroot (requiere Flatpak habilitado).

### 5.4 Crear un snapshot de la configuración de usuario

Si deseas que el sistema instalado ya tenga un usuario preconfigurado (con sus archivos, temas, etc.), puedes copiar `/etc/skel` desde tu sistema y añadir personalizaciones. O bien, durante la instalación, Cubic permite crear un usuario por defecto; luego puedes incluir scripts de postinstalación que copien archivos.

## 6. Prueba del ISO

Antes de distribuir, prueba el ISO en una máquina virtual:

- Arranca desde el ISO en VirtualBox/Virt-Manager.
- Verifica que el instalador funcione correctamente (Ubiquity o Subiquity).
- Comprueba que los paquetes y configuraciones se desplieguen adecuadamente.
- Prueba la conectividad, impresoras, dispositivos de almacenamiento.

## 7. Consideraciones legales y de marca

- **Nombre**: no uses "Ubuntu" en el nombre de tu distribución si no superas las pruebas oficiales de Canonical (puedes usar "basado en Ubuntu" en la descripción).
- **Licencias**: al incluir paquetes de terceros, respeta sus licencias (GPL, MIT, etc.). No redistribuyas software propietario sin permiso.
- **Logos**: evita usar el logo de Ubuntu a menos que indiques claramente que es una versión modificada no oficial.
- **Atribuciones**: incluye un archivo `README` o `LICENSE` en el ISO que reconozca a Canonical y a los autores de los paquetes incluidos.

## 8. Publicación y distribución

- Puedes subir el ISO a servicios como **SourceForge**, **GitHub Releases**, o **Google Drive** (con enlace público).
- Para equipos internos, usa un servidor local (Nginx, Apache) con un directorio de descargas.
- Genera una suma de verificación (SHA256) para que los usuarios validen la integridad:
  ```bash
  sha256sum my-ubuntu-custom.iso > my-ubuntu-custom.iso.sha256
  ```

## 9. Solución de problemas comunes

| Problema | Causa posible | Solución |
|----------|---------------|----------|
| El ISO no arranca | Imagen mal generada | Rehacer con `grub-mkrescue` o verificar que los binarios de arranque estén presentes. |
| El sistema en vivo no tiene red | Faltan controladores WiFi | Incluir paquetes `linux-firmware` y `iwlwifi`. |
| El instalador se cuelga | Problema de memoria o espacio | Aumentar el tamaño del squashfs (no más de 4 GB para compatibilidad). |
| Paquetes faltan al instalar | Dependencias no resueltas | Dentro del chroot, ejecutar `apt-get install -f` y volver a regenerar manifest. |

## 10. Consejos finales

- Mantén un registro de los cambios que realizas (usa un script de shell o un Dockerfile).
- Si necesitas actualizaciones periódicas, automatiza el proceso con un script CI/CD (GitHub Actions, por ejemplo).
- Para distribuciones más ligeras, considera usar **Ubuntu Server** como base y añadir solo el escritorio que necesites.
- Si tu distribución está destinada a un público amplio, incluye un tema visual propio y un instalador personalizado (como Ubiquity con skel).

---

Con este manual, tienes todas las herramientas para empaquetar tu sistema Ubuntu personalizado. Ya sea para uso personal, empresarial o educativo, una distribución derivada te permite transportar tu entorno informático completo a cualquier máquina. 
