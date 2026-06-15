# Manuales para crear distribuciones derivadas de Linux

Este repositorio contiene guías detalladas para construir distribuciones Linux personalizadas basadas en tres de las familias más populares:

- **Ubuntu** (basado en Debian)
- **Arch Linux** (rolling release, DIY)
- **Rocky Linux** (clon de RHEL)

Cada manual cubre métodos prácticos (herramientas gráficas y manuales) para empaquetar tu sistema actual con todos sus paquetes, configuraciones y preferencias, generando un ISO instalable y portable.

## 📁 Contenido

| Archivo | Sistema base | Herramientas principales | Enfoque |
|---------|--------------|--------------------------|---------|
| [`ubuntu.md`](manuales/ubuntu.md) | Ubuntu 22.04/24.04 LTS | Cubic, chroot manual | Gráfico (Cubic) y manual con chroot |
| [`archlinux.md`](manuales/archlinux.md) | Arch Linux | `archiso`, perfil personalizado | Control total mediante perfil de construcción |
| [`rockylinux.md`](manuales/rockylinux.md) | Rocky Linux 9 | `livecd-creator`, `mkksiso`, chroot | Kickstart y manipulación de ISO oficial |

Cada guía incluye:
- Requisitos previos.
- Explicación del proceso paso a paso.
- Ejemplos de personalización (paquetes, usuarios, configuraciones de red, servicios).
- Solución de problemas comunes.
- Consideraciones legales y de marca.

## 🎯 ¿Para qué sirve este repositorio?

- **Empaquetar tu entorno de trabajo** en un solo ISO portable (útil para desarrolladores, sysadmins, laboratorios).
- **Crear distribuciones preconfiguradas** para empresas, escuelas o equipos específicos.
- **Automatizar despliegues** de estaciones de trabajo o servidores.
- **Preservar configuraciones** complejas (ej. desarrollo con múltiples herramientas, entornos de ciencia de datos).
- **Aprender** sobre el proceso de construcción de distribuciones Linux.

## 🖥️ Requisitos generales

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| Disco libre | 15 GB | 30 GB |
| RAM | 4 GB | 8 GB |
| Sistema base | La distribución que desees derivar | Actualizada |
| Red | Conexión estable (descarga de paquetes) | Banda ancha |

## 🚀 Uso rápido

1. Clona el repositorio:
   ```bash
   git clone https://github.com/perseoq/distribuciones_derivadas.git
   cd distribuciones_derivadas
   ```

2. Elige tu distribución base:
   - Para **Ubuntu**: abre `ubuntu.md` y sigue el método Cubic (más sencillo).
   - Para **Arch Linux**: sigue `archlinux.md` con el perfil `releng` modificado.
   - Para **Rocky Linux**: usa `rockylinux.md` y el método `mkksiso` para inyectar un kickstart.

3. Personaliza los archivos según tus necesidades (listas de paquetes, configuraciones, scripts `%post`).

4. Genera tu ISO y pruébalo en una máquina virtual.

## 📚 Referencias adicionales

- [Cubic (Ubuntu)](https://launchpad.net/cubic)
- [Archiso (Arch Linux)](https://wiki.archlinux.org/title/Archiso)
- [Lorax / livemedia-creator (Rocky/RHEL)](https://github.com/weldr/lorax)
- [Guía de kickstart de Red Hat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/performing_an_advanced_rhel_9_installation/creating-kickstart-files_installing-rhel-as-an-experienced-user)

## 📝 Contribuciones

Las contribuciones son bienvenidas. Si mejoras algún manual o agregas soporte para otra distribución (Debian, Fedora, openSUSE), por favor abre un Pull Request.

## ⚖️ Licencia

Este trabajo se proporciona bajo la licencia **MIT**. Puedes usar, modificar y distribuir el contenido libremente, siempre que mantengas la atribución original.

**Nota**: Los manuales hacen referencia a marcas registradas (Ubuntu®, Arch Linux®, Rocky Linux®). Estas marcas pertenecen a sus respectivos propietarios. Este repositorio no está afiliado oficialmente con Canonical, Arch Linux o Rocky Enterprise Software Foundation.

