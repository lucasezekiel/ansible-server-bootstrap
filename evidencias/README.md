# Evidencias

Capturas y salidas de una ejecución real del playbook. Cada `.png` tiene al lado un `.txt` con la misma salida en texto plano, para poder buscar en ella o compararla.

## Entorno (2026-11-19)

| Componente | Versión |
|---|---|
| Nodo de control | Debian GNU/Linux 13 (trixie), Ansible Core 2.22.0, ansible-lint 26.11.0 |
| Virtualización | QEMU/KVM + libvirt 11.3.0, red NAT `default` |
| Destino | Ubuntu 24.04.5 LTS (kernel 6.8.0-147), 2 vCPU, 2 GB RAM, restaurable por snapshot |
| Instalado por el playbook | Docker 29.10.0, Node Exporter 1.12.1, Prometheus 3.16.0, Grafana 13.3.1 |

El inventario `inventory/lab.yml` que aparece en las capturas es local y no se versiona; apunta a la VM con el usuario `ubuntu` y su clave.

El destino partió de un estado sin UFW, Docker ni monitoreo, con acceso inicial como `ubuntu` por clave SSH. Al terminar, la VM se restauró al snapshot previo.

## Capturas

| # | Archivo | Qué demuestra |
|---|---|---|
| 01 | [01-syntax-lint](01-syntax-lint.png) | Sintaxis válida y `ansible-lint` sin fallas (perfil *production*) |
| 02 | [02-ping](02-ping.png) | Conectividad Ansible → destino |
| 03 | [03-primera-ejecucion](03-primera-ejecucion.png) | Primera ejecución: `ok=26 changed=12 failed=0` |
| 04 | [04-segunda-ejecucion-idempotente](04-segunda-ejecucion-idempotente.png) | Segunda ejecución: `ok=25 changed=0 failed=0`, es decir, idempotente |
| 05 | [05-estado-servidor](05-estado-servidor.png) | `sshd -T` efectivo, UFW activo (solo 22/tcp), Docker y unattended-upgrades activos, zona horaria y NTP |
| 06 | [06-pruebas-acceso-ssh](06-pruebas-acceso-ssh.png) | El administrador entra con su clave; se rechazan la contraseña y el login como root |
| 07 | [07-monitoreo-opcional](07-monitoreo-opcional.png) | Rol de monitoreo con `-e enable_monitoring=true`: `ok=36 changed=11 failed=0` |
| 08 | [08-monitoreo-solo-loopback](08-monitoreo-solo-loopback.png) | Grafana, Prometheus y Node Exporter escuchan solo en `127.0.0.1` y no son accesibles desde la red |
| 09 | [09-grafana-por-tunel-ssh](09-grafana-por-tunel-ssh.png) | Grafana accesible solo por túnel SSH (`ssh -L 13000:127.0.0.1:3000`) |
| 10 | [10-prometheus-targets](10-prometheus-targets.png) | Prometheus recolecta métricas de Node Exporter (target `UP`) |

## Observaciones

- La primera ejecución reporta `changed=12` en 24.04, frente a `changed=10` en la prueba anterior con 22.04. Esta vez el usuario administrador y su clave no existían de antemano, así que se crearon.
- En el primer arranque, Grafana tardó unos 6 minutos en escuchar (migraciones de su base interna con 2 GB de RAM). Durante ese tiempo `curl` devuelve código 000; no es un error de configuración.
- La prueba 08 verifica el *bind* a loopback con `ss` en el propio servidor, independientemente de UFW. La prueba desde el nodo de control confirma que tampoco hay exposición efectiva.
