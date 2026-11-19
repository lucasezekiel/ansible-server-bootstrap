🇪🇸 Español | [🇬🇧 English](README.en.md)

# Ansible Server Bootstrap

[![Ansible checks](https://github.com/lucasezekiel/ansible-server-bootstrap/actions/workflows/ci.yml/badge.svg)](https://github.com/lucasezekiel/ansible-server-bootstrap/actions/workflows/ci.yml)

Playbook de Ansible que automatiza la puesta en marcha de un servidor Ubuntu recién instalado. Incluye:

- configuración base;
- endurecimiento de seguridad (hardening);
- instalación de Docker;
- un laboratorio de monitoreo opcional.

> **Solo para laboratorio: revisalo antes de ejecutarlo.** El playbook modifica usuarios, SSH, firewall y Docker en el destino. No es una línea base de seguridad lista para producción. Usá un servidor descartable con acceso por consola; ejecutarlo contra el equipo equivocado puede dejarte sin acceso o exponer servicios.

## Qué hace

**Rol `common`**

- Actualiza el índice de apt (con una ventana de caché de 1 hora).
- Instala paquetes base: `git`, `curl`, `wget`, `htop`, `unzip`, `ca-certificates`.
- Configura la zona horaria y verifica la sincronización de hora.
- Activa las actualizaciones de seguridad automáticas con `unattended-upgrades`.

**Rol `security`**

- Crea un usuario administrador con sudo y le copia la clave SSH pública del operador.
- Agrega un drop-in de SSH temprano que desactiva la autenticación por contraseña y keyboard-interactive. Antes de reiniciar SSH valida la configuración **efectiva** (`sshd -T`).
- Deshabilita el login remoto de root.
- Configura UFW: deniega todo lo entrante salvo SSH y permite lo saliente.

**Rol `docker`**

- Agrega la clave GPG y el repositorio oficial de Docker (formato deb822).
- Instala Docker Engine, CLI, containerd, Buildx y Compose.
- Habilita e inicia el servicio.
- Agrega el administrador al grupo `docker` **solo** si `docker_grant_admin_group` está activado.

**Rol `monitoring`** (desactivado por defecto)

- Instala Node Exporter desde el archivo oficial y lo ejecuta como servicio systemd.
- Despliega Prometheus y Grafana con Docker Compose en red de host.
- Escucha solo en loopback: Grafana (`3000/tcp`), Prometheus (`9090/tcp`) y Node Exporter (`9100/tcp`). No abre reglas de firewall para ellos.

```text
Nodo de control (Ansible) ──SSH──▶ Ubuntu Server
                                     ├─ common      paquetes, zona horaria, unattended-upgrades
                                     ├─ security    admin + clave, SSH endurecido, UFW
                                     ├─ docker      repo oficial, Engine + Compose
                                     └─ monitoring  Node Exporter, Prometheus, Grafana (opcional)
```

## Estructura

```text
├── inventory/hosts.yml       # servidores destino
├── group_vars/all.yml        # variables (admin, paquetes, zona horaria, clave SSH)
├── roles/
│   ├── common/
│   ├── security/             # handler: reinicia SSH solo ante cambios reales
│   ├── docker/
│   └── monitoring/           # Node Exporter, Prometheus, Grafana
├── evidencias/               # capturas y salidas de ejecuciones reales
├── site.yml                  # playbook de entrada
├── requirements.yml          # colecciones de Ansible con versión fija
└── .github/workflows/ci.yml  # chequeos estáticos
```

## Requisitos

- Colecciones `ansible.posix`, `community.general` y `community.docker` en el nodo de control (versiones en `requirements.yml`).
- Chequeos estáticos configurados para Ansible Core 2.22.0 y ansible-lint 26.11.0.
- Un servidor de pruebas descartable accesible por SSH. Se probó con Ubuntu 22.04.5 LTS en QEMU/KVM; una versión anterior, con Multipass.
- Un par de claves SSH propio (por defecto `~/.ssh/id_ed25519`).

## Uso

**Antes de ejecutar:** `inventory/hosts.yml` apunta a `192.0.2.10`, una dirección reservada para documentación. Prepará tu propio inventario y revisá estas variables:

- `ansible_user`: la cuenta de la conexión SSH inicial;
- `admin_user`: el administrador que gestionará el playbook;
- `ssh_pubkey_path`: la clave pública con la que vas a volver a conectarte.

Los valores por defecto son ejemplos. Tené a mano acceso por consola y un punto de restauración (snapshot).

```bash
ansible-galaxy collection install -r requirements.yml

# Conectividad
ansible -i inventory/hosts.yml all -m ping

# Qué hosts se seleccionarían
ansible-playbook -i inventory/hosts.yml site.yml --list-hosts

# Simulación (el modo check no garantiza cero efectos secundarios)
ansible-playbook -i inventory/hosts.yml site.yml --check --ask-become-pass

# Aplicar
ansible-playbook -i inventory/hosts.yml site.yml --ask-become-pass
```

La membresía al grupo `docker` y el monitoreo están desactivados en `group_vars/all.yml`. Para activarlos en un laboratorio descartable, poné `docker_grant_admin_group` o `enable_monitoring` en `true` después de leer las limitaciones de seguridad. El cambio de grupo requiere una sesión SSH nueva.

## Validación estática

GitHub Actions verifica la sintaxis del playbook y corre ansible-lint en cada push y pull request (el badge muestra el último resultado). Estos chequeos no se conectan a ningún destino ni verifican el comportamiento del despliegue. Localmente:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --syntax-check
ansible-lint --offline --nocolor site.yml
```

## Prueba en laboratorio e idempotencia

La revisión actual (repositorio deb822) se aplicó sobre una VM Ubuntu Server 22.04.5 en QEMU/KVM, restaurada a un estado previo al playbook, con Ansible Core 2.22.0. La VM ya tenía su cuenta de administrador y su clave SSH.

| Ejecución | Resultado |
|---|---|
| Primera | `ok=26 changed=10 unreachable=0 failed=0 skipped=10` |
| Repetición (dentro de la ventana de caché de apt) | `ok=25 changed=0 unreachable=0 failed=0 skipped=10` |
| Revisión anterior, sobre una VM ya aprovisionada | `ok=22 changed=0 unreachable=0 failed=0 skipped=10` |

Después de la primera ejecución:

- existía el repositorio de Docker en formato deb822 y Docker estaba activo;
- una conexión SSH nueva con la clave del laboratorio funcionó;
- para el usuario del laboratorio el servidor ofrecía solo autenticación por clave pública;
- el monitoreo y el grupo `docker` quedaron desactivados (tareas omitidas).

Las capturas y salidas están en [`evidencias/`](evidencias/). Estos resultados valen para ese estado y esa configuración. No prueban el comportamiento en cualquier host, la migración de un repositorio de Docker existente ni el rol de monitoreo.

## Limitaciones de seguridad

- **Grupo `docker`:** equivale a acceso root y por eso es opcional. Sin él, los comandos administrativos normalmente requieren `sudo`.
- **Sin contraseña de administrador:** el playbook no la define. Si sudo la pide, configurala por un canal seguro y conservá el acceso original hasta verificar la cuenta nueva.
- **Monitoreo:** usa red de host e imágenes `latest`. El bind a loopback limita la exposición, pero no está verificado con una prueba de integración. Revisá las credenciales iniciales de Grafana y no lo expongas a redes no confiables.
- **Docker y UFW:** Docker puede saltear el filtrado de UFW para los puertos publicados; verificá la exposición real, no solo las reglas.
- **Endurecimiento SSH:** un error en la clave o en la configuración del destino puede dejarte sin acceso. Mantené otra sesión abierta y acceso por consola.
- **Validación SSH global:** se comprueba la configuración global efectiva; las reglas `Match` específicas por host o usuario hay que revisarlas aparte.
- **Include de `sshd_config.d`:** el drop-in depende de que `/etc/ssh/sshd_config` incluya `/etc/ssh/sshd_config.d/*.conf`. Si un ajuste anterior tiene prioridad, la validación falla antes del reinicio.
- **Node Exporter:** se descarga sin verificar el checksum.
- **Desactivar no revierte:** apagar una opción o quitar una tarea no deshace lo aplicado antes. Revisá aparte reglas `NOPASSWD`, grupo `docker`, servicios de monitoreo y reglas de UFW previas.
- **Fuentes APT de Docker:** antes de ejecutar los roles, el playbook se detiene si existe `/etc/apt/sources.list.d/docker.list` o `docker.sources`. Esas fuentes se migran a mano; no se borra ninguna automáticamente.

## Decisiones de diseño

- **UFW en lugar de iptables directo:** prioriza legibilidad y mantenimiento sobre el control granular, adecuado para el alcance del proyecto.
- **Handler para reiniciar SSH:** solo reinicia cuando cambia el drop-in y después de validar la configuración efectiva.
- **Caché de apt acotada (`cache_valid_time: 3600`):** idempotencia con ventana de tiempo. Actualizar en cada ejecución reportaría `changed` sin aportar nada.
- **Docker desde el repositorio oficial:** sigue las versiones de upstream en lugar de los paquetes de la distribución.

## Limitaciones conocidas

- **Clave SSH bajo `--check`:** la tarea depende de que el usuario exista. En simulación nunca se crea, así que falla en `--check` aunque la ejecución real funciona.
- **Rol Docker bajo `--check`:** el repositorio depende de `python3-debian` y de la clave, y los paquetes dependen del repositorio. En un destino limpio la simulación puede fallar. La versión deb822 no se probó en modo check sobre una VM limpia.

En ambos casos el patrón es el mismo: **las tareas encadenadas que dependen de un efecto real de una tarea anterior dan falsos fallos en `--check`.** Es un comportamiento documentado de Ansible, no un bug del playbook.

## Próximas mejoras

- Ansible Vault para variables sensibles.
- Verificar con una prueba de integración que el monitoreo escucha solo en loopback; fijar versiones de imágenes y verificar artefactos.
- Procedimiento de migración explícito para hosts ya configurados.
- Pruebas automatizadas con Molecule.
- Ejemplo de inventario multi-host (staging/producción).

## Licencia

MIT. Ver [LICENSE](LICENSE).
