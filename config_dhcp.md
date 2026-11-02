# Playbook: Servidor DHCP en Debian (isc-dhcp-server)

![Debian](https://img.shields.io/badge/Debian-12%2B-A81D33?logo=debian&logoColor=white)
![ISC DHCP](https://img.shields.io/badge/isc--dhcp--server-4.4-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Guía paso a paso para montar un servidor DHCPv4 en Debian con `isc-dhcp-server`: IP estática, rango de direcciones, reservas por MAC, integración con el firewall `iptables`, verificación y resolución de problemas.

---

## Índice

1. [Escenario](#-escenario)
2. [Requisitos](#-requisitos)
3. [Conceptos clave](#-conceptos-clave)
4. [IP estática en la interfaz LAN](#1-ip-estática-en-la-interfaz-lan)
5. [Instalación](#2-instalación)
6. [Interfaz de escucha](#3-interfaz-de-escucha)
7. [Configuración del servidor](#4-configuración-del-servidor)
8. [Validación y arranque](#5-validación-y-arranque)
9. [Integración con iptables](#6-integración-con-iptables)
10. [Prueba desde un cliente](#7-prueba-desde-un-cliente)
11. [Comandos de referencia](#-comandos-de-referencia)
12. [Troubleshooting](#-troubleshooting)
13. [Rollback](#-rollback)

---

## Escenario

```
                 ┌──────────────────────────┐
   Internet ──── │ enp0s3          Debian   │
     (WAN)       │            servidor DHCP │
                 │ enp0s8  192.168.10.1/24  │
                 └────────────┬─────────────┘
                              │  LAN 192.168.10.0/24
                 ┌────────────┴─────────────┐
                 │ Clientes                  │
                 │ Rango: .100 – .200        │
                 │ Reserva: cliente1 → .50   │
                 └───────────────────────────┘
```

| Parámetro | Valor |
|---|---|
| Interfaz LAN (servidor) | `enp0s8` |
| IP del servidor | `192.168.10.1/24` |
| Red | `192.168.10.0/24` |
| Rango dinámico | `192.168.10.100` – `192.168.10.200` |
| Puerta de enlace | `192.168.10.1` |
| DNS | `8.8.8.8`, `1.1.1.1` |
| Reserva | `cliente1` → `192.168.10.50` |

> Adaptar los nombres de interfaz y direcciones al entorno real (`ip a` para ver las interfaces).

## Requisitos

- Debian 11/12 o superior
- Usuario con privilegios `sudo`
- Al menos una interfaz conectada a la red de los clientes (en VirtualBox: **Red interna** con el mismo nombre en servidor y clientes)
- **Ningún otro servidor DHCP** activo en esa red

## Conceptos clave

Proceso **DORA** entre cliente y servidor (UDP, cliente `68` ↔ servidor `67`):

| Paso | Mensaje | Sentido |
|---|---|---|
| D | **Discover** | Cliente → broadcast: "¿hay algún servidor DHCP?" |
| O | **Offer** | Servidor → cliente: "te ofrezco esta IP" |
| R | **Request** | Cliente → broadcast: "acepto esa IP" |
| A | **Ack** | Servidor → cliente: "confirmado, es tuya durante X segundos" |

- **Lease (concesión)**: tiempo durante el que el cliente puede usar la IP.
- **authoritative**: el servidor se declara responsable de la red y rechaza (NAK) peticiones de IPs que no le corresponden.
- **Reserva**: asignación fija de una IP a una dirección MAC concreta.

---

## 1. IP estática en la interfaz LAN

El servidor DHCP necesita una IP fija dentro de la subred que reparte.

Editar `/etc/network/interfaces`:

```bash
auto enp0s8
iface enp0s8 inet static
    address 192.168.10.1
    netmask 255.255.255.0
```

Aplicar y comprobar:

```bash
sudo systemctl restart networking
ip a show enp0s8
```

## 2. Instalación

```bash
sudo apt update
sudo apt install -y isc-dhcp-server
```

> Es **normal** que el servicio falle al terminar la instalación: aún no está configurado.

## 3. Interfaz de escucha

Editar `/etc/default/isc-dhcp-server`:

```bash
INTERFACESv4="enp0s8"
INTERFACESv6=""
```

## 4. Configuración del servidor

Copia de seguridad del fichero original:

```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bak
```

Contenido de `/etc/dhcp/dhcpd.conf`:

```conf
# ---------- Opciones globales ----------
authoritative;
default-lease-time 600;        # 10 minutos
max-lease-time 7200;           # 2 horas

# ---------- Subred LAN ----------
subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.10.255;
    option domain-name-servers 8.8.8.8, 1.1.1.1;
}

# ---------- Reservas por MAC ----------
host cliente1 {
    hardware ethernet 08:00:27:aa:bb:cc;
    fixed-address 192.168.10.50;
}
```

> La IP reservada (`.50`) queda **fuera** del rango dinámico (`.100–.200`) para evitar conflictos.

## 5. Validación y arranque

Comprobar sintaxis (sin salida de error = correcto):

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Arrancar y habilitar en el arranque:

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

## 6. Integración con iptables

Si el servidor tiene un firewall con política `INPUT DROP` (ver [playbook de iptables](./playbook-iptables-debian.md)), añadir a `/usr/local/sbin/firewall.sh` **antes de la regla de LOG**:

```bash
# --- DHCP en la LAN ---
iptables -A INPUT -i enp0s8 -p udp --sport 68 --dport 67 -j ACCEPT
```

Aplicar y guardar:

```bash
sudo /usr/local/sbin/firewall.sh
sudo netfilter-persistent save
```

> `dhcpd` recibe los paquetes mediante sockets raw y en la práctica suele funcionar sin esta regla, pero dejarla explícita documenta la política del firewall y evita sorpresas.

Si además el Debian hace de **router con NAT** para la LAN, los clientes necesitan salida a Internet a través de él (sección NAT del playbook de iptables).

## 7. Prueba desde un cliente

En el cliente (Debian/Ubuntu):

```bash
sudo dhclient -r          # liberar la IP actual
sudo dhclient -v          # pedir una nueva mostrando el proceso DORA
ip a
ip route                  # la ruta por defecto debe apuntar a 192.168.10.1
```

En el servidor:

```bash
cat /var/lib/dhcp/dhcpd.leases          # concesiones entregadas
journalctl -u isc-dhcp-server -f        # logs en tiempo real
```

Resultado esperado en el log del servidor:

```
DHCPDISCOVER from 08:00:27:xx:xx:xx via enp0s8
DHCPOFFER on 192.168.10.100 to 08:00:27:xx:xx:xx via enp0s8
DHCPREQUEST for 192.168.10.100 from 08:00:27:xx:xx:xx via enp0s8
DHCPACK on 192.168.10.100 to 08:00:27:xx:xx:xx via enp0s8
```

| Prueba | Comando | Resultado esperado |
|---|---|---|
| Cliente obtiene IP | `ip a` (cliente) | IP en `192.168.10.100–200` |
| Gateway correcto | `ip route` (cliente) | `default via 192.168.10.1` |
| DNS asignado | `cat /etc/resolv.conf` (cliente) | `8.8.8.8`, `1.1.1.1` |
| Reserva | `ip a` en `cliente1` | `192.168.10.50` |
| Concesión registrada | `cat /var/lib/dhcp/dhcpd.leases` | Entrada con la MAC del cliente |

---

## Comandos de referencia

| Acción | Comando |
|---|---|
| Comprobar sintaxis | `dhcpd -t -cf /etc/dhcp/dhcpd.conf` |
| Ejecutar en primer plano (depuración) | `dhcpd -4 -f -d -cf /etc/dhcp/dhcpd.conf enp0s8` |
| Reiniciar servicio | `systemctl restart isc-dhcp-server` |
| Estado del servicio | `systemctl status isc-dhcp-server` |
| Logs | `journalctl -u isc-dhcp-server -f` |
| Ver concesiones | `cat /var/lib/dhcp/dhcpd.leases` |
| Liberar IP (cliente) | `dhclient -r` |
| Pedir IP (cliente) | `dhclient -v` |
| Capturar tráfico DHCP | `tcpdump -i enp0s8 -n port 67 or port 68` |

## 🛠 Troubleshooting

### El servicio falla sin mostrar la causa

`systemctl` y `journalctl` muestran sobre todo el texto genérico de ISC ("If you think you have received this message due to a bug…"). Para ver el error real, ejecutar `dhcpd` en primer plano:

```bash
sudo dhcpd -4 -f -d -cf /etc/dhcp/dhcpd.conf enp0s8
```

El error aparece justo **antes** del bloque genérico. Si se queda esperando sin errores, la configuración es válida (`Ctrl+C` y reiniciar el servicio).

### `line 3: semicolon expected` — `default-least-time 600;`

**Causa**: errata en la directiva. Lo correcto es `default-lease-time`.

### `line 6: Expecting netmask` — `subnet 192.168.10.0 netmasl`

**Causa**: errata. Lo correcto es `netmask`.

**Solución para ambas**:

```bash
sudo sed -i 's/default-least-time/default-lease-time/; s/netmasl/netmask/' /etc/dhcp/dhcpd.conf
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
```

> `dhcpd` indica la **línea** y marca con `^` la posición aproximada del error. Si el mensaje apunta a una línea correcta, revisar la anterior (suele faltar un `;` o una `}`).

### Otros errores frecuentes

| Mensaje / síntoma | Causa | Solución |
|---|---|---|
| `Not configured to listen on any interfaces!` | `INTERFACESv4` vacío o con nombre incorrecto | Ajustar `/etc/default/isc-dhcp-server` con el nombre de `ip a` |
| `No subnet declaration for enp0s8` | La IP de la interfaz no pertenece a ninguna `subnet` declarada | Revisar `/etc/network/interfaces` e `ip a show enp0s8` |
| `Can't open lease database` | Falta el fichero de concesiones o permisos | `sudo touch /var/lib/dhcp/dhcpd.leases` |
| El cliente no recibe IP | Distinta red (VirtualBox: distinto nombre de red interna), interfaz `DOWN` u otro DHCP en la red | Verificar red interna, `ip link set enp0s8 up`, `tcpdump` |
| El cliente recibe IP de otro rango | Otro servidor DHCP en la red (p. ej. NAT de VirtualBox) | Desactivar el otro DHCP o aislar la red |

## Rollback

Detener el servicio y restaurar la configuración original:

```bash
sudo systemctl stop isc-dhcp-server
sudo systemctl disable isc-dhcp-server
sudo cp /etc/dhcp/dhcpd.conf.bak /etc/dhcp/dhcpd.conf
```

Desinstalar por completo:

```bash
sudo apt purge -y isc-dhcp-server
```

---

## 📝 Nota

`isc-dhcp-server` dejó de recibir mantenimiento por parte de ISC en 2022. Sigue disponible en los repositorios de Debian y es habitual en entornos formativos, pero para producción el sucesor recomendado es **ISC Kea** (`kea-dhcp4-server`).

---

## 👤 Autor

**Dragos** — Estudiante de CFGS ASIR
