# Playbook: Firewall con nftables en Debian

![Debian](https://img.shields.io/badge/Debian-12%2B-A81D33?logo=debian&logoColor=white)
![nftables](https://img.shields.io/badge/nftables-1.0%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Guía paso a paso para **migrar de iptables a nftables** e implementar un firewall con política restrictiva (deny-by-default) en Debian, filtrando IPv4 e IPv6 con un único fichero de configuración. Incluye reglas para los servicios DHCP y DNS de la LAN, NAT opcional, verificación y resolución de problemas.

---

## Índice

1. [Escenario](#-escenario)
2. [Requisitos](#-requisitos)
3. [Conceptos clave](#-conceptos-clave)
4. [Medida anti-bloqueo SSH](#️-medida-anti-bloqueo-ssh)
5. [Fase 1 — Eliminar iptables](#fase-1--eliminar-la-configuración-de-iptables)
6. [Fase 2 — Configurar nftables](#fase-2--configurar-nftables)
7. [Verificación](#verificación)
8. [Opcional: Debian como router (NAT)](#opcional-debian-como-router-nat)
9. [Cambios en caliente](#-cambios-en-caliente)
10. [Equivalencias iptables → nftables](#-equivalencias-iptables--nftables)
11. [Comandos de referencia](#-comandos-de-referencia)
12. [Troubleshooting](#-troubleshooting)
13. [Rollback](#-rollback)

---

## Escenario

```
                 ┌──────────────────────────────┐
   Internet ──── │ enp0s3 (WAN)        Debian   │
                 │   nftables + DHCP + DNS      │
                 │ enp0s8 (LAN) 192.168.10.1/24 │
                 └──────────────┬───────────────┘
                                │  LAN 192.168.10.0/24
                          ┌─────┴─────┐
                          │ Clientes  │
                          └───────────┘
```

| Tráfico | Acción |
|---|---|
| Entrante (input) | **Denegado** salvo lo permitido explícitamente |
| Reenviado (forward) | **Denegado** (salvo NAT opcional) |
| Saliente (output) | Permitido |
| Conexiones establecidas / relacionadas | Permitidas |
| SSH `22/tcp` | Permitido desde cualquier interfaz |
| Ping (ICMP echo-request) e ICMPv6 | Permitidos |
| DHCP `67/udp` | Permitido solo desde la LAN |
| DNS `53/udp` y `53/tcp` | Permitido solo desde la LAN |
| Paquetes descartados | Registrados en el log (con límite) |

## Requisitos

- Debian 11/12 o superior (nftables es el framework de filtrado por defecto desde Debian 10)
- Usuario con privilegios `sudo`
- Acceso a consola local **o** SSH (ver [medida anti-bloqueo](#️-medida-anti-bloqueo-ssh))

## Conceptos clave

- **nftables** sustituye a `iptables`, `ip6tables`, `arptables` y `ebtables` con una sola herramienta: `nft`.
- **Familias**: `ip` (IPv4), `ip6` (IPv6), **`inet` (IPv4 + IPv6 a la vez)**, `arp`, `bridge`, `netdev`.
- **Tablas**: contenedores de cadenas. A diferencia de iptables, **no existen tablas predefinidas**; se crean con el nombre que se quiera.
- **Cadenas base**: se enganchan a un *hook* del kernel (`input`, `output`, `forward`, `prerouting`, `postrouting`) y tienen una **política** por defecto.
- **Reglas**: se evalúan de arriba abajo; la primera que emite un veredicto (`accept`, `drop`) decide.
- **Variables**: `define NOMBRE = valor` permite reutilizar valores (interfaces, puertos, redes).
- **Configuración declarativa**: todo el firewall vive en `/etc/nftables.conf`, que se carga de forma atómica (todo o nada).

---

## Medida anti-bloqueo SSH

Si se trabaja en remoto, programar una limpieza automática **antes** de tocar el firewall:

```bash
sudo bash -c 'sleep 120 && nft flush ruleset' &
```

Si se pierde el acceso, a los 2 minutos se eliminan todas las reglas. Si todo funciona, cancelarla:

```bash
jobs
kill %1
```

---

## Fase 1 — Eliminar la configuración de iptables

### 1. Abrir el tráfico y vaciar reglas

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -F
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo ip6tables -F
sudo ip6tables -X
```

### 2. Eliminar la persistencia de iptables

Si se mantiene, al reiniciar se volverían a cargar las reglas antiguas en conflicto con nftables.

```bash
sudo apt purge -y iptables-persistent netfilter-persistent
sudo rm -f /etc/iptables/rules.v4 /etc/iptables/rules.v6
```

### 3. Archivar el script antiguo

```bash
sudo mv /usr/local/sbin/firewall.sh /usr/local/sbin/firewall.sh.old
```

### 4. Limpiar tablas residuales

En Debian, `iptables` funciona sobre nftables (`iptables-nft`), por lo que quedan tablas creadas por él:

```bash
sudo nft flush ruleset
sudo nft list ruleset        # no debe mostrar nada
```

---

## Fase 2 — Configurar nftables

### 5. Instalar y habilitar

```bash
sudo apt install -y nftables
sudo systemctl enable nftables
```

### 6. Escribir la configuración

```bash
sudo cp /etc/nftables.conf /etc/nftables.conf.bak
sudo nano /etc/nftables.conf
```

> En nano: `Alt+A` + `Ctrl+K` borra todo el contenido · `Ctrl+Shift+V` pega · `Ctrl+O` + `Enter` guarda · `Ctrl+X` sale.

Contenido de `/etc/nftables.conf`:

```nft
#!/usr/sbin/nft -f

flush ruleset

define LAN_IF = "enp0s8"

table inet filter {

    chain input {
        type filter hook input priority filter; policy drop;

        # Loopback
        iif "lo" accept

        # Estado de conexiones
        ct state established,related accept
        ct state invalid drop

        # SSH
        tcp dport 22 ct state new accept

        # ICMP: ping IPv4 + ICMPv6 (imprescindible para IPv6: NDP, RA)
        icmp type echo-request accept
        meta l4proto ipv6-icmp accept

        # DHCP en la LAN
        iifname $LAN_IF udp dport 67 accept

        # DNS en la LAN
        iifname $LAN_IF udp dport 53 accept
        iifname $LAN_IF tcp dport 53 accept

        # Registrar lo descartado
        limit rate 5/minute log prefix "NFT-DROP: " level warn
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

> En una máquina **sin** servicios DHCP/DNS (p. ej. un cliente), eliminar esos bloques. Ajustar `LAN_IF` al nombre real de la interfaz (`ip a`).

### 7. Validar y aplicar

```bash
sudo nft -c -f /etc/nftables.conf      # comprueba la sintaxis sin aplicar (sin salida = correcto)
sudo systemctl restart nftables
sudo systemctl status nftables
```

---

## Verificación

Ver el conjunto de reglas cargado:

```bash
sudo nft list ruleset
```

Pruebas funcionales:

| Prueba | Comando | Resultado esperado |
|---|---|---|
| Servicio activo | `systemctl status nftables` | `active (exited)` ✅ (normal: carga las reglas y termina) |
| Arranque automático | `systemctl is-enabled nftables` | `enabled` |
| Salida a Internet | `ping -c 3 8.8.8.8` | ✅ Responde |
| DNS local | `dig @127.0.0.1 servidor.dragos.lan` | `status: NOERROR` |
| DHCP activo | `systemctl status isc-dhcp-server` | `active (running)` |
| DHCP a cliente | `sudo dhclient -r && sudo dhclient -v` (cliente) | `DHCPOFFER ... from 192.168.10.1` |
| SSH entrante | `ssh usuario@<IP>` desde otro equipo | ✅ Conecta |
| Puerto cerrado | `nc -zv <IP> 80` desde otro equipo | ❌ Timeout |
| Logs de descartes | `sudo journalctl -k \| grep NFT-DROP` | Entradas registradas |
| Persistencia | `sudo reboot` → `sudo nft list ruleset` | Reglas cargadas |

Ver contadores de tráfico por regla (útil para depurar qué regla deja pasar un paquete):

```bash
sudo nft add rule inet filter input counter        # contador al final de la cadena
sudo nft list chain inet filter input
```

> Para contadores permanentes, añadir la palabra `counter` en las reglas del fichero, p. ej. `tcp dport 22 ct state new counter accept`.

---

## Opcional: Debian como router (NAT)

Escenario: `enp0s3` = WAN, `enp0s8` = LAN.

Activar el reenvío de paquetes:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```

En `/etc/nftables.conf`, añadir la variable de la WAN junto a la de la LAN:

```nft
define WAN_IF = "enp0s3"
```

Sustituir la cadena `forward` de la tabla `inet filter` por:

```nft
    chain forward {
        type filter hook forward priority filter; policy drop;

        ct state established,related accept
        ct state invalid drop

        # LAN → Internet
        iifname $LAN_IF oifname $WAN_IF accept

        # Tráfico redirigido por DNAT (port forwarding)
        ct status dnat accept
    }
```

Y añadir al final del fichero una tabla NAT:

```nft
table ip nat {

    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # Port forwarding (ejemplo): WAN:8080 → servidor web interno 192.168.10.10:80
        # iifname $WAN_IF tcp dport 8080 dnat to 192.168.10.10:80
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # Enmascarar la LAN tras la IP de la WAN
        oifname $WAN_IF masquerade
    }
}
```

Aplicar:

```bash
sudo nft -c -f /etc/nftables.conf
sudo systemctl restart nftables
```

Comprobar desde un cliente de la LAN: `ping -c 3 8.8.8.8` y `ping -c 3 debian.org`.

---

## Cambios en caliente

Los cambios con `nft add` / `nft delete` se aplican al momento **pero se pierden al reiniciar**. Sirven para probar; lo definitivo va en `/etc/nftables.conf`.

Añadir una regla (al final de la cadena):

```bash
sudo nft add rule inet filter input tcp dport 80 accept
```

Insertar una regla al principio:

```bash
sudo nft insert rule inet filter input ip saddr 1.2.3.4 drop
```

Borrar una regla por su *handle*:

```bash
sudo nft -a list chain inet filter input       # muestra "# handle N" en cada regla
sudo nft delete rule inet filter input handle N
```

Volver a la configuración del fichero:

```bash
sudo systemctl restart nftables
```

---

## Equivalencias iptables → nftables

| iptables | nftables |
|---|---|
| `iptables -P INPUT DROP` | `policy drop;` en la cadena |
| `-i lo -j ACCEPT` | `iif "lo" accept` |
| `-i enp0s8` | `iifname "enp0s8"` |
| `-m conntrack --ctstate ESTABLISHED,RELATED` | `ct state established,related` |
| `-p tcp --dport 22 -j ACCEPT` | `tcp dport 22 accept` |
| `-m multiport --dports 80,443` | `tcp dport { 80, 443 }` |
| `-s 192.168.10.0/24` | `ip saddr 192.168.10.0/24` |
| `-p icmp --icmp-type echo-request` | `icmp type echo-request` |
| `-m limit --limit 5/min -j LOG --log-prefix "X"` | `limit rate 5/minute log prefix "X"` |
| `-t nat -A POSTROUTING -o eth0 -j MASQUERADE` | `oifname "eth0" masquerade` (cadena `postrouting` tipo `nat`) |
| `-j DNAT --to-destination IP:PUERTO` | `dnat to IP:PUERTO` |
| `iptables` + `ip6tables` por separado | Una tabla `inet` para ambos |
| `iptables -L -v -n` | `nft list ruleset` |
| `netfilter-persistent save` | Editar `/etc/nftables.conf` + `systemctl restart nftables` |

> 💡 `iptables-translate` convierte reglas sueltas a sintaxis nft:
> ```bash
> iptables-translate -A INPUT -p tcp --dport 22 -j ACCEPT
> # nft 'add rule ip filter INPUT tcp dport 22 counter accept'
> ```

## Comandos de referencia

| Acción | Comando |
|---|---|
| Ver todas las reglas | `nft list ruleset` |
| Ver reglas con handles | `nft -a list ruleset` |
| Ver una cadena | `nft list chain inet filter input` |
| Validar fichero | `nft -c -f /etc/nftables.conf` |
| Cargar fichero | `nft -f /etc/nftables.conf` |
| Aplicar configuración | `systemctl restart nftables` |
| Vaciar todas las reglas | `nft flush ruleset` |
| Añadir regla en caliente | `nft add rule inet filter input <regla>` |
| Borrar regla | `nft delete rule inet filter input handle N` |
| Volcar reglas a fichero | `nft list ruleset > reglas.nft` |
| Ver logs | `journalctl -k \| grep NFT-DROP` |
| Puertos en escucha | `ss -tulpn` |

## Troubleshooting

### Error de sintaxis al validar

```
/etc/nftables.conf:20:9-11: Error: syntax error, unexpected ...
```

El formato es `fichero:línea:columna`. `nft` marca con `^^^` la posición. Causas típicas:

- Falta el `;` tras `type filter hook input priority filter` o tras `policy drop`.
- Llaves `{ }` sin cerrar.
- Palabra mal escrita (`establised`, `acept`, `dpot`...).

### `Error: No such file or directory` al añadir una regla en caliente

La tabla o cadena no existe o tiene otro nombre/familia. Comprobar con `nft list ruleset` y usar exactamente `inet filter input`.

### Las reglas desaparecen al reiniciar

| Causa | Solución |
|---|---|
| El servicio no está habilitado | `sudo systemctl enable nftables` |
| Los cambios se hicieron con `nft add` | Pasarlos a `/etc/nftables.conf` |
| Sigue instalado `iptables-persistent` y sobrescribe | `sudo apt purge iptables-persistent netfilter-persistent` |

### `systemctl status nftables` muestra `active (exited)`

**Es correcto.** El servicio carga las reglas en el kernel y termina; las reglas siguen activas. Verificar con `nft list ruleset`.

### El cliente no recibe IP por DHCP

```bash
sudo nft list ruleset | grep 67
systemctl status isc-dhcp-server
```

Comprobar que `LAN_IF` coincide con la interfaz real y que la regla `udp dport 67` existe.

### El DNS no responde desde la LAN

```bash
sudo nft list ruleset | grep 53
sudo ss -tulpn | grep :53
```

Deben existir las reglas UDP **y** TCP del puerto 53, y `named` debe escuchar en `192.168.10.1`.

### Sin salida a Internet tras aplicar el firewall

Falta o está mal la regla `ct state established,related accept`: las respuestas a las conexiones salientes se descartan por la política `drop`.

### Los clientes de la LAN no navegan (modo router)

| Comprobación | Comando |
|---|---|
| Reenvío activo | `sysctl net.ipv4.ip_forward` → `= 1` |
| Tabla NAT cargada | `nft list table ip nat` |
| Regla de forward LAN → WAN | `nft list chain inet filter forward` |
| Gateway del cliente | `ip route` en el cliente → `default via 192.168.10.1` |

## Rollback

Abrir todo el tráfico de inmediato:

```bash
sudo nft flush ruleset
```

Volver a la configuración original de Debian:

```bash
sudo cp /etc/nftables.conf.bak /etc/nftables.conf
sudo systemctl restart nftables
```

Desactivar nftables por completo:

```bash
sudo systemctl disable --now nftables
sudo nft flush ruleset
```

Recuperar el firewall de iptables anterior:

```bash
sudo mv /usr/local/sbin/firewall.sh.old /usr/local/sbin/firewall.sh
sudo systemctl disable --now nftables
sudo /usr/local/sbin/firewall.sh
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

## 🔗 Playbooks relacionados

- [Firewall con iptables](./playbook-iptables-debian.md)
- [Servidor DHCP (isc-dhcp-server)](./playbook-dhcp-debian.md)
- [Servidor DNS (BIND9)](./playbook-dns-debian.md)

---

## 👤 Autor

**Dragos** — Estudiante de CFGS ASIR
