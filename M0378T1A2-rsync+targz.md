
# Guía Práctica & Playbook: Automatización de Backup Remoto con Rsync e IP Dinámica/Estática en Linux

Este documento resume la implementación, solución de errores y el procedimiento paso a paso para configurar copias de seguridad remotas entre máquinas virtuales Linux (Debian/Ubuntu) utilizando **Rsync** sobre **SSH**.

---

## Resumen de la Solución
El objetivo de la práctica consistió en automatizar la transferencia y sincronización de datos desde una máquina origen local hacia un servidor remoto mediante un script en Bash (`backup_remoto.sh`).

* **Máquina Origen (rsync1):** Genera la estructura de archivos y ejecuta el script de sincronización.
* **Máquina Destino (rsync2 / debian):** Recibe las copias de seguridad en un directorio dedicado.
* **Protocolo de red:** SSH (Puerto 22) y Rsync.

---

## 🛠️ Errores Frecuentes Encontrados y Soluciones

Durante el proceso se presentaron diversos errores comunes en la administración de sistemas. A continuación se detallan sus causas y resoluciones:

### 1. `rsync: [sender] change_dir "/home/.../origen" failed: No such file or directory`
* **Causa:** La ruta especificada en la máquina de origen no existía físicamente o el script se estaba ejecutando dentro de un directorio donde no se encontraba la ruta relativa indicada.
* **Solución:** Se creó la estructura de origen con `mkdir -p ~/origen` y se aseguraron rutas absolutas dentro del script Bash.

### 2. `rsync: --mkdir: unknown option`
* **Causa:** La versión instalada de `rsync` en Debian/Ubuntu (v3.5.0 o anterior) no soportaba el flag `--mkdir` en línea de comandos.
* **Solución:** Se eliminó el parámetro del script y se garantizó la existencia del directorio destino de forma explícita antes de ejecutar el comando de sincronización.

### 3. Conflicto de IP duplicada en VirtualBox (Misma IP en ambas máquinas)
* **Causa:** Al clonar la máquina virtual en modo *Adaptador Solo Anfitrión*, ambas máquinas conservaron el mismo identificador de concesión DHCP o archivo de red.
* **Solución:** Se regeneraron las direcciones MAC en las propiedades de red de VirtualBox y se forzó la renovación de concesión de red mediante `dhclient`.

### 4. `bash: ./backup_remoto.sh: No such file or directory`
* **Causa:** El usuario intentaba ejecutar el script estando posicionado dentro del directorio `~/origen/` en lugar de estar en la raíz de su usuario (`~`).
* **Solución:** Regresar al directorio personal mediante el comando `cd ~`.

---

## Playbook Paso a Paso (Guía de Despliegue)

### Paso 1: Configurar la Red en VirtualBox
1. Apaga ambas Máquinas Virtuales.
2. En la ventana principal de VirtualBox, dirígete a: **Configuración > Red**.
3. Selecciona **Adaptador Solo-Anfitrión** (Host-Only Adapter).
4. En la sección **Avanzadas**, pulsa el botón de actualización para asignar **MACs distintas** a cada máquina.
5. Enciende ambas máquinas y verifica las direcciones IP:
   ```bash
   ip a
   ```

---

### Paso 2: Preparar la Estructura de Archivos en el Origen (`rsync1`)
En la máquina origen, crea el directorio de trabajo y genera los 10 archivos de prueba solicitados:

```bash
# Ir al directorio home
cd ~

# Crear el directorio origen si no existe
mkdir -p ~/origen

# Crear los 10 archivos de prueba
touch ~/origen/file{1..10}

# Comprobar la creación de los archivos
ls -la ~/origen/
```

---

### Paso 3: Preparar el Directorio Destino en la Máquina Remota (`rsync2`)
Desde la máquina origen, conéctate vía SSH o ejecuta el comando remoto para verificar/crear el directorio de destino:

```bash
ssh vboxuser@192.168.111.59 "mkdir -p ~/destino"
```

---

### Paso 4: Crear y Configurar el Script de Backup (`backup_remoto.sh`)
Crea el script de automatización en la raíz del usuario en la máquina origen:

```bash
nano ~/backup_remoto.sh
```

Añade el siguiente código adaptando el usuario y la IP remota:

```bash
#!/bin/bash

# ==============================================================================
# Script de Backup Remoto con Rsync
# ==============================================================================

ORIGEN="/home/vboxuser/origen/"
DESTINO_USUARIO="vboxuser"
DESTINO_IP="192.168.111.59"
DESTINO_RUTA="/home/vboxuser/destino/"

echo "Iniciando respaldo de $ORIGEN a $DESTINO_USUARIO@$DESTINO_IP:$DESTINO_RUTA..."

# Ejecución de Rsync
rsync -avzh "$ORIGEN" "$DESTINO_USUARIO@$DESTINO_IP:$DESTINO_RUTA"

if [ $? -eq 0 ]; then
    echo "¡Backup completado con éxito!"
else
    echo "Error en la transferencia de archivos."
fi
```

Guarda con `Ctrl + O`, confirma con `Enter` y sal con `Ctrl + X`.

---

### Paso 5: Conceder Permisos y Ejecutar el Script
Otorga permisos de ejecución al script y pruébalo:

```bash
chmod +x ~/backup_remoto.sh
./backup_remoto.sh
```

---

### Paso 6 (Opcional): Autenticación sin Contraseña mediante Claves SSH
Para evitar que `rsync` pida la contraseña en cada ejecución:

1. Genera un par de claves SSH en la máquina origen:
   ```bash
   ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
   ```
2. Copia la clave pública al servidor remoto:
   ```bash
   ssh-copy-id vboxuser@192.168.111.59
   ```
3. Vuelve a ejecutar `./backup_remoto.sh` para comprobar que la copia se realiza de forma automática sin pedir credenciales.

---

## Verificación Final
Para comprobar que los archivos llegaron correctamente a la máquina remota, ejecuta:

```bash
ssh vboxuser@192.168.111.59 "ls -l ~/destino"
```

Deberías observar el listado con los archivos `file1` a `file10`.
