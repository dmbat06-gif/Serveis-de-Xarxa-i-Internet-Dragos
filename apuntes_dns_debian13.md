# MIS APUNTES: CONFIGURAR DNS BIND9 EN DEBIAN 13 (TRIXIE)


*   **Red en VirtualBox:** Red NAT
*   **Hardware:** 25GB de disco y 2GB de RAM.
*   **IP del Servidor:** Tiene que ser **Estática (Fija)** porque un servidor no puede estar cambiando de IP.

### Los dos fallos del principio 
1.  **Error de `sudo`:** Mi usuario `dragos` no tenía permisos de administrador. 
    *   *Cómo lo arreglé:* Entré como root con `su -`, metí a mi usuario en el grupo con `usermod -aG sudo dragos` y **reinicié la máquina** (`reboot`) para que se aplicaran los cambios.
2.  **Error de `Destination Host Unreachable` (No hacía ping):** No tenía internet en la máquina virtual.
    *   *Cómo lo arreglé:* Cambié el modo de red en VirtualBox a **Red NAT** y la máquina ya pudo salir a internet.

---

## 2. Instalar el Servidor DNS (Bind9)
**¿Qué es BIND9?** Es el programa clásico que se usa en Linux para que la máquina funcione como un servidor DNS. Lleva usándose un montón de años en internet.

*   *El comando correcto para instalarlo es:*
```bash
sudo apt install bind9 bind9-utils
```
*(Si ya lo habías instalado antes, el sistema te dirá "already the newest version" y no pasa nada).*

---

## 3. Poner la IP Fija
Editamos el archivo `/etc/network/interfaces` para quitar el DHCP automático y dejar una IP fija para nuestra práctica:

```ini
# Archivo: /etc/network/interfaces

auto lo
iface lo inet loopback

# Mi IP estática
auto enp0s3
iface enp0s3 inet static
    address 192.168.6.123
    netmask 255.255.255.0
    gateway 192.168.6.1      # El router de mi Red NAT
```

---

## 4. Configurar Bind9 (Paso a paso)
Casi toda la configuración se hace dentro de la carpeta `/etc/bind/`. Tenemos que tocar tres archivos y crear la carpeta `zones` (`sudo mkdir /etc/bind/zones`).

### Paso A: Decirle al DNS qué zonas va a controlar (`named.conf.local`)
Aquí configuramos la **zona directa** (para pasar de nombre a IP) y la **zona inversa** (para pasar de IP a nombre).

```named
// Archivo: /etc/bind/named.conf.local

zone "haven.local" {
    type master;
    file "/etc/bind/zones/db.haven.local";
};

zone "6.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.6.168.192";
};
```
---

### Paso B: Crear los mapas de la red (Ficheros de Zona)
Como en Debian 13 venimos sin plantillas, hay que escribir la estructura a mano.

#### 1. Fichero de Zona Directa: `/etc/bind/zones/db.haven.local`
```text
$TTL    86400
@       IN      SOA     ns1.haven.local. hostmaster.haven.local. (
                     2026092801         ; Serial (Versión del archivo, cámbialo si editas)
                         604800         ; Refresh (Cada quién mira el esclavo)
                          86400         ; Retry (Si falla, cuánto tarda en reintentar)
                        2419200         ; Expire (Tiempo límite del esclavo)
                          86400 )       ; Minimum TTL (Caché de fallos)
;
@       IN      NS      ns1.haven.local.
ns1     IN      A       192.168.6.123
```
*   **¿Qué significa cada cosa en el examen?**
    *   **TTL:** Lo que dura la información en la caché de otros PCs (86400 segundos = 1 día).
    *   **@:** Significa "el dominio actual" (`haven.local`).
    *   **ns1.haven.local.:** El nombre de este servidor DNS.
    *   **hostmaster...:** El correo del administrador (se pone un punto en vez de un `@`).
    *   **A:** El registro básico. Vincula un nombre con una IP IPv4.

#### 2. Fichero de Zona Inversa: `/etc/bind/zones/db.6.168.192`
```text
$TTL    86400
@       IN      SOA     ns1.haven.local. hostmaster.haven.local. (
                     2026092801         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Minimum TTL
;
@       IN      NS      ns1.haven.local.
123     IN      PTR     ns1.haven.local.
```
*   **¿Qué cambia aquí?**
    *   **in-addr.arpa:** Es la coletilla obligatoria para hacer DNS inverso en IPv4.
    *   **PTR:** Es el registro "puntero". Hace lo contrario que el registro **A**: vincula el último número de la IP (`123`) con el nombre del servidor.

#### Comandos para comprobar que las zonas no tengan fallos:
Antes de reiniciar el servicio, ejecuto esto para asegurarme el 10 en la práctica:
```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192
```

---

### Paso C: Configuración Global (`named.conf.options`)
Aquí decidimos las reglas del juego de nuestro servidor: quién puede preguntar y a dónde va la máquina si no sabe una respuesta.

```named
// Archivo: /etc/bind/named.conf.options

options {
        directory "/var/cache/bind";

        // Escucha peticiones en mi IP local y en mi IP fija
        listen-on port 53 { 127.0.0.1; 192.168.6.123; };
        
        // Activo la recursividad para que busque webs de fuera
        recursion yes;
        // Solo dejo hacer preguntas a mi propia red local
        allow-recursion { 127.0.0.1; 192.168.6.0/24; };

        // Si no sabe una IP (ej: google.com), se lo pregunta a estos DNS públicos
        forwarders {
                8.8.8.8;
                1.1.1.1;
        };

        dnssec-validation auto;
};
```

---

### Paso D: Desactivar IPv6 (`/etc/default/named`)
Para evitar que la terminal se llene de errores feos de IPv6 porque nuestra máquina virtual solo usa IPv4, editamos este archivo:
```bash
sudo nano /etc/default/named
```
Y dejamos la línea de opciones así, añadiendo el `-4`:
```ini
OPTIONS="-u bind -4"
```

---

## 5. Reiniciar y Probar que Funciona (El momento de la verdad)

1.  **Reinicio el servicio** para aplicar todo lo que he configurado:
    ```bash
    sudo systemctl restart bind9
    ```
2.  **Miro si está en verde** (running):
    ```bash
    sudo systemctl status bind9
    ```

3.  **Auditoría de puertos abiertos:**
    Para asegurarme de que el servidor está escuchando en el puerto que le toca (el **53** para DNS), ejecuto este comando para listar las conexiones activas:
    ```bash
    sudo ss -lntup
    ```
    *Aquí revisamos que la IP `192.168.6.123` y `127.0.0.1` estén asociadas al puerto `:53` bajo el proceso de `named`.*

4.  **Configurar y PROTEGER el archivo del cliente (`/etc/resolv.conf`):** 
    Tuve que ir a editar el archivo `/etc/resolv.conf` (`sudo nano /etc/resolv.conf`) para **cambiar la IP antigua por la mía fija** y añadir el dominio interno de la máquina virtual:
    ```text
    nameserver 192.168.6.123
    search myguest.virtualbox.org haven.local
    ```
    **¡Paso clave!** Como el NetworkManager o el DHCP de VirtualBox a veces machacan este archivo al reiniciar la red y te borran la configuración, **lo bloqueamos poniéndole el atributo de inmutable**:
    ```bash
    sudo chattr +i /etc/resolv.conf
    ```
    *(Si en el futuro necesito volver a editarlo, tendré que quitarle el bloqueo con `sudo chattr -i /etc/resolv.conf`).*

5.  **Hago los test finales con `nslookup`:**
    *   Si pongo `nslookup ns1.haven.local`, me tiene que escupir mi IP `192.168.6.123`.
    *   Si pongo `nslookup 192.168.6.123`, me tiene que escupir el nombre `ns1.haven.local`.
