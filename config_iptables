# Playbook: Firewall con iptables en Debian

![Debian](https://img.shields.io/badge/Debian-12%2B-A81D33?logo=debian&logoColor=white)
![iptables](https://img.shields.io/badge/iptables-nf__tables-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Guía paso a paso para implementar un firewall con **política restrictiva** (deny-by-default) usando `iptables` en Debian, con persistencia tras reinicio, verificación y resolución de problemas.

---

##  Índice

1. [Objetivo](#-objetivo)
2. [Requisitos](#-requisitos)
3. [Conceptos clave](#-conceptos-clave)
4. [Instalación](#1-instalación)
5. [Script del firewall](#2-script-del-firewall)
6. [Aplicación segura (anti-bloqueo SSH)](#3-aplicación-segura-anti-bloqueo-ssh)
7. [Verificación](#4-verificación)
8. [Persistencia](#5-persistencia)
9. [IPv6](#6-ipv6)
10. [Opcional: Debian como router (NAT)](#7-opcional-debian-como-router-nat)
11. [Comandos de referencia](#-comandos-de-referencia)
12. [Troubleshooting](#-troubleshooting)
13. [Rollback](#-rollback)

---

## Objetivo

| Tráfico | Acción |
|---|---|
| Entrante (INPUT) | **Denegado** salvo servicios permitidos explícitamente |
| Reenviado (FORWARD) | **Denegado** |
| Saliente (OUTPUT) | Permitido |
| Conexiones ya establecidas | Permitidas |
| SSH (22/tcp) | Permitido |
| ICMP echo-request (ping) | Permitido |
| Paquetes descartados | Registrados en el log (con límite) |

## Requisitos

- Debian 11/12 o superior
- Usuario con privilegios `sudo`
- Acceso a consola local **o** SSH (ver [aplicación segura](#3-aplicación-segura-anti-bloqueo-ssh))

## Conceptos clave

- **Tablas**: `filter` (por defecto, filtrado), `nat` (traducción de direcciones), `mangle` (modificación de paquetes).
- **Cadenas**: `INPUT` (hacia el equipo), `OUTPUT` (desde el equipo), `FORWARD` (atraviesa el equipo).
- **Política**: acción por defecto si ninguna regla coincide (`ACCEPT` / `DROP`).
- **Orden**: las reglas se evalúan de arriba abajo; la primera que coincide decide.
- **conntrack**: seguimiento de estado de conexiones (`NEW`, `ESTABLISHED`, `RELATED`, `INVALID`).
- Desde Debian 10, `iptables` usa el backend **nftables** (`iptables-nft`) como capa de compatibilidad.

---

## 1. Instalación

```bash
sudo apt update
sudo apt install -y iptables iptables-persistent
```

> Durante la instalación, `iptables-persistent` pregunta si guardar las reglas actuales. Se puede responder **Sí**; se sobrescribirán más adelante.

Comprobar estado inicial:

```bash
sudo iptables -L -v -n --line-numbers
```

## 2. Script del firewall

Crear `/usr/local/sbin/firewall.sh`:

```bash
#!/bin/bash
# =============================================
#  Firewall iptables - Debian
#  Política restrictiva en INPUT y FORWARD
# =============================================

# --- Limpiar reglas, cadenas y contadores ---
iptables -F
iptables -X
iptables -Z

# --- Políticas por defecto ---
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# --- Loopback ---
iptables -A INPUT -i lo -j ACCEPT

# --- Estado de conexiones ---
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# --- SSH ---
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
# Restringido a una red concreta (alternativa más segura):
# iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# --- Servicios web (descomentar si aplica) ---
# iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT

# --- ICMP (ping entrante) ---
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# --- Log de paquetes descartados (limitado) ---
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPT-DROP: " --log-level 4
```

Dar permisos de ejecución:

```bash
sudo chmod +x /usr/local/sbin/firewall.sh
```

## 3. Aplicación segura (anti-bloqueo SSH)

> Si se trabaja **en remoto**, un error en las reglas puede dejar el servidor inaccesible.

Programar una apertura automática a los 2 minutos **antes** de aplicar:

```bash
sudo bash -c 'sleep 120 && iptables -P INPUT ACCEPT && iptables -F' &
```

Aplicar el firewall:

```bash
sudo /usr/local/sbin/firewall.sh
```

Si la sesión SSH sigue funcionando, cancelar el rollback automático:

```bash
jobs          # localizar el trabajo en segundo plano
kill %1       # o el número que corresponda
```

## 4. Verificación

Listar reglas con contadores:

```bash
sudo iptables -L INPUT -v -n --line-numbers
```

Resultado esperado (resumen):

```
Chain INPUT (policy DROP)
num  target  prot  in   source     destination
1    ACCEPT  all   lo   0.0.0.0/0  0.0.0.0/0
2    ACCEPT  all   *    0.0.0.0/0  0.0.0.0/0   ctstate RELATED,ESTABLISHED
3    DROP    all   *    0.0.0.0/0  0.0.0.0/0   ctstate INVALID
4    ACCEPT  tcp   *    0.0.0.0/0  0.0.0.0/0   tcp dpt:22 ctstate NEW
5    ACCEPT  icmp  *    0.0.0.0/0  0.0.0.0/0   icmptype 8
6    LOG     all   *    0.0.0.0/0  0.0.0.0/0   limit: avg 5/min ... LOG prefix "IPT-DROP: "
```

Pruebas funcionales:

| Prueba | Comando | Resultado esperado |
|---|---|---|
| Salida a Internet | `ping -c 3 8.8.8.8` | ✅ Responde |
| Resolución DNS | `ping -c 3 debian.org` | ✅ Responde |
| Repositorios | `sudo apt update` | ✅ Funciona |
| Ping entrante | `ping <IP_debian>` desde otro equipo | ✅ Responde |
| SSH entrante | `ssh usuario@<IP_debian>` | ✅ Conecta |
| Puerto cerrado | `nc -zv <IP_debian> 80` desde otro equipo | ❌ Timeout |
| Logs de descartes | `sudo journalctl -k \| grep IPT-DROP` | Entradas registradas |

> **Tip**: lanzar una prueba y volver a listar las reglas; el contador `pkts` de la regla que deja pasar el tráfico aumentará.

## 5. Persistencia

Guardar las reglas activas:

```bash
sudo netfilter-persistent save
```

Ficheros generados:

| Fichero | Contenido |
|---|---|
| `/etc/iptables/rules.v4` | Reglas IPv4 |
| `/etc/iptables/rules.v6` | Reglas IPv6 |

Se cargan automáticamente al arrancar mediante el servicio `netfilter-persistent`:

```bash
systemctl status netfilter-persistent
```

Comprobar tras reinicio:

```bash
sudo reboot
sudo iptables -L -v -n
```

## 6. IPv6

> `iptables` **solo** filtra IPv4. Si el equipo tiene IPv6, todo queda abierto por esa vía salvo que se configure `ip6tables`.

```bash
ip6tables -F
ip6tables -X
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT ACCEPT

ip6tables -A INPUT -i lo -j ACCEPT
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
ip6tables -A INPUT -m conntrack --ctstate INVALID -j DROP
ip6tables -A INPUT -p ipv6-icmp -j ACCEPT        # Imprescindible (NDP, router advertisements)
ip6tables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

Después: `sudo netfilter-persistent save`.

## 7. Opcional: Debian como router (NAT)

Escenario: `eth0` = WAN, `eth1` = LAN.

Activar el reenvío de paquetes:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```

Reglas de NAT y forwarding:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Port forwarding (DNAT), ejemplo: puerto 8080 de la WAN → servidor web interno `192.168.1.10:80`:

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80
iptables -A FORWARD -i eth0 -o eth1 -p tcp -d 192.168.1.10 --dport 80 -m conntrack --ctstate NEW -j ACCEPT
```

---

## 📚 Comandos de referencia

| Acción | Comando |
|---|---|
| Listar reglas con números | `iptables -L -v -n --line-numbers` |
| Listar tabla NAT | `iptables -t nat -L -v -n` |
| Añadir regla al final | `iptables -A INPUT ...` |
| Insertar regla en posición N | `iptables -I INPUT N ...` |
| Borrar regla N | `iptables -D INPUT N` |
| Bloquear una IP | `iptables -I INPUT 1 -s 1.2.3.4 -j DROP` |
| Vaciar reglas | `iptables -F` |
| Volcar reglas | `iptables-save` |
| Restaurar reglas | `iptables-restore < /etc/iptables/rules.v4` |
| Guardar persistente | `netfilter-persistent save` |
| Ver logs | `journalctl -k \| grep IPT-DROP` |

## 🛠 Troubleshooting

### `unknown option "--ctsate"`

```
iptables v1.8.11 (nf_tables): unknown option "--ctsate"
```

**Causa**: errata al escribir el parámetro. Lo correcto es `--ctstate`.

### `Couldn't load match 'imit': No such file or directory`

**Causa**: errata en el módulo. Lo correcto es `-m limit`.

**Solución para ambas**:

```bash
sudo sed -i 's/--ctsate/--ctstate/g; s/-m imit/-m limit/g' /usr/local/sbin/firewall.sh
grep -n -E 'ctstate|limit' /usr/local/sbin/firewall.sh
sudo /usr/local/sbin/firewall.sh
```

> Un script de bash **no se detiene** ante un error: las políticas `DROP` se aplican aunque falle la regla `ESTABLISHED,RELATED`. Resultado: se bloquean las respuestas a conexiones salientes (DNS, `apt`, `ping` hacia fuera) y puede cortarse la sesión SSH. Para que el script aborte al primer fallo, añadir `set -e` tras el shebang.

### El ping funciona aunque la regla `ESTABLISHED` falló

Depende del sentido del ping:

| Origen → Destino | ¿Funciona? | Motivo |
|---|---|---|
| Otro equipo → Debian | ✅ | Regla `echo-request` + OUTPUT en ACCEPT |
| Debian → `localhost` | ✅ | Regla de loopback |
| Debian → Internet | ❌ | El `echo-reply` entra por INPUT sin regla que lo acepte |

### Las reglas desaparecen al reiniciar

No se guardaron: ejecutar `sudo netfilter-persistent save` y verificar `/etc/iptables/rules.v4`.

## Rollback

Abrir todo el tráfico de inmediato:

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -F
sudo iptables -t nat -F
sudo netfilter-persistent save   # solo si se quiere dejar así de forma permanente
```

---

## 👤 Autor

**Dragos** — Estudiante de CFGS ASIR
