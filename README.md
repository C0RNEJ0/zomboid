# 🧟 Guía Completa: Servidor Dedicado de Project Zomboid en la Nube

> **Autor:** C0RNEJ0  
> **Última actualización:** Junio 2026  
> **Plataforma:** Ubuntu 24.04 LTS en DigitalOcean  

Esta guía documenta **paso a paso** cómo montamos un servidor dedicado de Project Zomboid en la nube desde cero, incluyendo todos los errores que cometimos y cómo los solucionamos para que tú no los repitas.

---

## 📑 Índice

1. [¿Qué necesitas?](#-qué-necesitas)
2. [Opción A: Servidor en la nube (recomendado)](#-opción-a-servidor-en-la-nube-recomendado)
3. [Opción B: Servidor local (tu PC)](#-opción-b-servidor-local-tu-pc)
4. [Paso 1: Crear el Droplet en DigitalOcean](#paso-1-crear-el-droplet-en-digitalocean)
5. [Paso 2: Configurar puertos de red](#paso-2-configurar-puertos-de-red)
6. [Paso 3: Conectarte al servidor por SSH](#paso-3-conectarte-al-servidor-por-ssh)
7. [Paso 4: Instalar el servidor de Project Zomboid](#paso-4-instalar-el-servidor-de-project-zomboid)
8. [Paso 5: Configurar la memoria RAM correctamente](#paso-5-configurar-la-memoria-ram-correctamente)
9. [Paso 6: Crear Swap (MUY IMPORTANTE)](#paso-6-crear-swap-muy-importante)
10. [Paso 7: Configurar el servicio systemd](#paso-7-configurar-el-servicio-systemd)
11. [Paso 8: Subir tu mapa y configuración](#paso-8-subir-tu-mapa-y-configuración)
12. [Paso 9: Configurar backups automáticos](#paso-9-configurar-backups-automáticos)
13. [Paso 10: Administrar el servidor](#paso-10-administrar-el-servidor)
14. [Errores comunes y soluciones](#-errores-comunes-y-soluciones)
15. [Configuraciones recomendadas del juego](#-configuraciones-recomendadas-del-juego)
16. [Comandos útiles de referencia rápida](#-comandos-útiles-de-referencia-rápida)

---

## 🛒 ¿Qué necesitas?

| Requisito | Detalle |
|---|---|
| **Cuenta de DigitalOcean** | O cualquier proveedor de VPS (Vultr, Linode, etc.) |
| **Droplet/VPS** | Mínimo **4GB RAM** (recomendado **8GB** para 8+ jugadores) |
| **Sistema operativo** | Ubuntu 22.04 o 24.04 LTS |
| **Cuenta de GitHub** | Para guardar tus backups (gratis) |
| **Terminal SSH** | PowerShell (Windows), Terminal (Mac/Linux) |

### 💰 Costo aproximado
- **Droplet 4GB RAM**: ~$24 USD/mes en DigitalOcean
- **Droplet 8GB RAM**: ~$48 USD/mes (recomendado si tienes 6+ jugadores)

---

## ☁️ Opción A: Servidor en la nube (recomendado)

**Ventajas:**
- ✅ Está encendido 24/7, tus amigos pueden jugar cuando quieran
- ✅ No consume recursos de tu PC
- ✅ Mejor conexión y ping más estable
- ✅ No necesitas abrir puertos en tu router
- ✅ Se reinicia automáticamente si se cae

**Desventajas:**
- ❌ Cuesta dinero mensual (~$24-48 USD)
- ❌ Necesitas conocimientos básicos de terminal/SSH

**Esta es la opción que usamos en esta guía.**

---

## 🖥️ Opción B: Servidor local (tu PC)

Si no quieres pagar por un servidor en la nube, puedes correr el servidor directamente en tu computadora.

### Requisitos mínimos de tu PC:
- **RAM**: 8GB mínimo (el servidor usa ~3GB + tu juego usa ~4GB)
- **CPU**: 4 núcleos mínimo
- **Internet**: Conexión estable, velocidad de subida de al menos 5 Mbps

### Pasos para servidor local (Windows):

1. **Instalar el servidor desde Steam:**
   - Abre Steam → Biblioteca → Herramientas
   - Busca **"Project Zomboid Dedicated Server"**
   - Instálalo (ocupa ~2GB)

2. **Abrir puertos en tu router:**
   - Accede a la configuración de tu router (generalmente `192.168.1.1` o `192.168.0.1`)
   - Busca la sección de **"Port Forwarding"** o **"Reenvío de puertos"**
   - Agrega estas reglas:

   | Puerto | Protocolo | IP destino |
   |---|---|---|
   | 16261 | UDP | La IP local de tu PC (ej: `192.168.1.100`) |
   | 16262 | UDP | La IP local de tu PC |

   > ⚠️ **Nota:** Cada router es diferente. Busca en YouTube "abrir puertos [marca de tu router]"

3. **Iniciar el servidor:**
   - Ve a la carpeta de instalación (generalmente `C:\Program Files (x86)\Steam\steamapps\common\Project Zomboid Dedicated Server`)
   - Ejecuta `StartServer64.bat`
   - La primera vez te pedirá crear una contraseña de administrador

4. **Tus amigos se conectan con:**
   - Tu IP pública (búscala en [whatismyip.com](https://whatismyip.com))

### ⚠️ Desventajas del servidor local:
- ❌ Tu PC debe estar **encendida siempre** que quieran jugar
- ❌ Consume recursos de tu PC (puede causar lag si juegas al mismo tiempo)
- ❌ Si tu internet se cae, todos se desconectan
- ❌ Necesitas abrir puertos en tu router (puede ser complicado)
- ❌ Tu IP puede cambiar (IP dinámica)

---

## Paso 1: Crear el Droplet en DigitalOcean

1. Crea una cuenta en [digitalocean.com](https://www.digitalocean.com)
2. Click en **"Create"** → **"Droplets"**
3. Configura así:

| Opción | Selección |
|---|---|
| **Región** | La más cercana a ti y tus amigos (ej: `New York` o `San Francisco` para México) |
| **Image** | Ubuntu 24.04 LTS |
| **Size** | **Regular** → **$24/mo (4GB RAM, 2 vCPUs)** |
| **Authentication** | Password (pon una contraseña segura) |
| **Hostname** | `zomboid` |

4. Click en **"Create Droplet"**
5. Anota la **IP pública** que te asignan (ej: `68.183.119.214`)

> 💡 **Tip:** Si tienes presupuesto, elige el de **8GB RAM ($48/mo)**. Con 4GB funciona pero necesitas configurar swap (lo veremos después).

---

## Paso 2: Configurar puertos de red

En DigitalOcean, ve a **Networking** → **Firewalls** y crea reglas de entrada:

| Puerto | Protocolo | Descripción |
|---|---|---|
| **16261** | **UDP** | Puerto principal de juego (Steam) |
| **16262** | **UDP** | Puerto directo / consulta |
| **22** | **TCP** | SSH (ya viene por defecto) |

> ⚠️ **Sin estos puertos abiertos, nadie podrá conectarse al servidor.**

---

## Paso 3: Conectarte al servidor por SSH

Desde **PowerShell** en Windows (o Terminal en Mac):

```bash
ssh root@TU_IP_DEL_DROPLET
```

Te pedirá la contraseña que configuraste. Escríbela (no se muestra mientras escribes, es normal).

---

## Paso 4: Instalar el servidor de Project Zomboid

Una vez conectado por SSH, ejecuta estos comandos uno por uno:

### 4.1. Actualizar el sistema e instalar dependencias
```bash
dpkg --add-architecture i386
apt-get update && apt-get upgrade -y
apt-get install -y git curl tar screen lib32gcc-s1 lib32stdc++6 ufw
```

### 4.2. Crear usuario dedicado "steam"
```bash
useradd -m -s /bin/bash steam
```

> 🔒 **¿Por qué un usuario separado?** Por seguridad. Si alguien hackea el servidor de juego, solo tiene acceso al usuario `steam`, no a `root`.

### 4.3. Instalar SteamCMD
```bash
sudo -u steam -i bash -c "mkdir -p /home/steam/steamcmd"
sudo -u steam -i bash -c "cd /home/steam/steamcmd && curl -sqL 'https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz' | tar zxvf -"
```

### 4.4. Descargar Project Zomboid Dedicated Server
```bash
sudo -u steam -i bash -c "/home/steam/steamcmd/steamcmd.sh +force_install_dir /home/steam/pzserver +login anonymous +app_update 380870 validate +quit"
```

> ⏳ Esto tarda **5-15 minutos** dependiendo de la velocidad del servidor. El juego pesa ~2GB.

---

## Paso 5: Configurar la memoria RAM correctamente

> 🔴 **ESTE PASO ES CRÍTICO.** Si lo haces mal, el servidor se va a crashear y va a sacar a todos los jugadores.

El archivo de configuración de memoria está en:
```
/home/steam/pzserver/ProjectZomboid64.json
```

### Tabla de configuración recomendada:

| RAM del Droplet | -Xms (mínima) | -Xmx (máxima) | Jugadores |
|---|---|---|---|
| **4GB** | `2560m` | `2560m` | 4-6 jugadores |
| **8GB** | `4096m` | `4096m` | 8-16 jugadores |
| **16GB** | `8192m` | `8192m` | 16-32 jugadores |

### Cómo modificarlo:

```bash
# Editar el archivo
sudo -u steam nano /home/steam/pzserver/ProjectZomboid64.json
```

Busca las líneas con `-Xms` y `-Xmx` dentro de `vmArgs` y cámbialas. Ejemplo para 4GB de RAM:

```json
{
    "vmArgs": [
        "-Xms2560m",
        "-Xmx2560m",
        ...
    ]
}
```

> ⚠️ **Error que cometimos:** Pusimos `-Xms3072m` y `-Xmx3072m` (3GB) en un Droplet de 4GB. El servidor usaba 3.5GB con memoria nativa de Java, dejando solo 500MB para el sistema operativo. Linux mataba el proceso por falta de memoria (**OOM Kill**) y todos los jugadores se desconectaban.
>
> **Regla de oro:** Deja siempre **al menos 1.5GB libres** para el sistema operativo.

---

## Paso 6: Crear Swap (MUY IMPORTANTE)

> 🔴 **NO TE SALTES ESTE PASO.** El swap es tu red de seguridad contra crashes por falta de memoria.

El **swap** es un espacio en disco que actúa como memoria RAM de emergencia. Si la RAM se llena, el sistema usa el swap en vez de matar el proceso del servidor.

```bash
# Crear archivo swap de 2GB
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Hacerlo permanente (sobrevive reinicios)
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Configurar que solo se use en emergencia (swappiness baja)
sysctl vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

### Verificar que funciona:
```bash
free -h
```

Deberías ver algo como:
```
              total        used        free      shared  buff/cache   available
Mem:          3.8Gi       3.1Gi       793Mi       2.5Gi       2.7Gi       776Mi
Swap:         2.0Gi       5.9Mi       2.0Gi    <-- ¡Esto es el swap!
```

> 💡 **¿Por qué swappiness=10?** El valor va de 0 a 100. Con 10, Linux solo usará el swap cuando realmente lo necesite, evitando que el juego se ponga lento por usar disco en vez de RAM.

---

## Paso 7: Configurar el servicio systemd

El servicio systemd permite que el servidor:
- Se inicie automáticamente cuando el Droplet arranca
- Se reinicie solo si se crashea
- Se administre con comandos simples (`start`, `stop`, `restart`)

### Crear el archivo de servicio:

```bash
nano /etc/systemd/system/pzserver.service
```

Pega este contenido (cambia `68.183.119.214` por tu IP):

```ini
[Unit]
Description=Project Zomboid Dedicated Server
After=network.target

[Service]
Type=forking
User=steam
Group=steam
WorkingDirectory=/home/steam/pzserver
ExecStart=/usr/bin/screen -S pzserver -d -m ./start-server.sh -servername servertest -ip TU_IP_AQUI
ExecStop=/bin/bash -c '/usr/bin/screen -S pzserver -p 0 -X stuff "quit$(printf \\r)"'
Nice=-10
IOSchedulingClass=best-effort
IOSchedulingPriority=2
TimeoutStopSec=60
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Explicación de cada línea importante:

| Línea | Qué hace |
|---|---|
| `User=steam` | Corre el servidor como usuario `steam` (seguridad) |
| `ExecStart=...screen...` | Inicia el servidor dentro de `screen` para poder ver la consola |
| `ExecStop=...quit...` | Envía el comando `/quit` para que guarde antes de apagar |
| `Nice=-10` | Le da prioridad de CPU al servidor sobre otros procesos |
| `Restart=on-failure` | Si el servidor crashea, se reinicia automáticamente |
| `RestartSec=10` | Espera 10 segundos antes de reiniciar |
| `TimeoutStopSec=60` | Espera hasta 60 segundos para que guarde al apagar |

### Activar el servicio:
```bash
systemctl daemon-reload
systemctl enable pzserver.service
systemctl start pzserver
```

---

## Paso 8: Subir tu mapa y configuración

Los archivos del servidor se guardan en `/home/steam/Zomboid/`. Tiene esta estructura:

```
/home/steam/Zomboid/
├── Saves/
│   └── Multiplayer/
│       └── servertest/     <-- Tu mundo guardado (mapa, jugadores, zombies)
├── Server/
│   ├── servertest.ini      <-- Configuración principal del servidor
│   └── servertest_SandboxVars.lua  <-- Reglas del juego (dificultad, zombies, etc.)
└── db/
    └── servertest.db       <-- Base de datos de jugadores y contraseñas
```

### Si tienes un backup en GitHub:
```bash
git clone https://github.com/TU_USUARIO/TU_REPO.git /tmp/zomboid_config
cp -r /tmp/zomboid_config/* /home/steam/Zomboid/
chown -R steam:steam /home/steam/Zomboid
rm -rf /tmp/zomboid_config
systemctl restart pzserver
```

---

## Paso 9: Configurar backups automáticos

> 💾 **Esto salva vidas.** Si el servidor se corrompe, puedes restaurar el último backup.

### 9.1. Crear un GitHub Personal Access Token

1. Ve a [github.com/settings/tokens/new](https://github.com/settings/tokens/new)
2. **Note**: `zomboid-backup`
3. **Expiration**: `No expiration`
4. **Marca**: ✅ `repo`
5. Click **"Generate token"**
6. Copia el token (empieza con `ghp_`)

> ⚠️ **Usa un token clásico** (que empiece con `ghp_`). Los tokens fine-grained (`github_pat_`) pueden dar problemas de permisos.

### 9.2. Clonar tu repositorio en el servidor

```bash
sudo -u steam git clone https://TU_TOKEN@github.com/TU_USUARIO/TU_REPO.git /home/steam/zomboid_backup_repo
sudo -u steam git -C /home/steam/zomboid_backup_repo config user.email "backup@zomboid-server"
sudo -u steam git -C /home/steam/zomboid_backup_repo config user.name "Zomboid Backup Bot"
```

### 9.3. Crear el script de backup

```bash
nano /home/steam/backup_zomboid.sh
```

Pega esto:

```bash
#!/bin/bash
# Backup Automático de Project Zomboid

ZOMBOID_DIR="/home/steam/Zomboid"
REPO_DIR="/home/steam/zomboid_backup_repo"
LOG_FILE="/home/steam/backup.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "" >> "$LOG_FILE"
echo "===== Backup iniciado: $TIMESTAMP =====" >> "$LOG_FILE"

if [ ! -d "$REPO_DIR/.git" ]; then
    echo "ERROR: Repositorio no encontrado" >> "$LOG_FILE"
    exit 1
fi

cd "$REPO_DIR"
git pull origin main >> "$LOG_FILE" 2>&1 || git pull origin master >> "$LOG_FILE" 2>&1

# Copiar las 3 carpetas
rm -rf "$REPO_DIR/Saves/Multiplayer"
mkdir -p "$REPO_DIR/Saves"
cp -r "$ZOMBOID_DIR/Saves/Multiplayer" "$REPO_DIR/Saves/Multiplayer" 2>> "$LOG_FILE"

rm -rf "$REPO_DIR/Server"
cp -r "$ZOMBOID_DIR/Server" "$REPO_DIR/Server" 2>> "$LOG_FILE"

rm -rf "$REPO_DIR/db"
cp -r "$ZOMBOID_DIR/db" "$REPO_DIR/db" 2>> "$LOG_FILE"

git add -A >> "$LOG_FILE" 2>&1

if git diff --cached --quiet; then
    echo "Sin cambios. No se hace commit." >> "$LOG_FILE"
    exit 0
fi

git commit -m "Backup automático: $TIMESTAMP" >> "$LOG_FILE" 2>&1
git push origin main >> "$LOG_FILE" 2>&1

echo "===== Backup finalizado: $(date '+%Y-%m-%d %H:%M:%S') =====" >> "$LOG_FILE"
```

### 9.4. Dar permisos y configurar cron

```bash
chmod +x /home/steam/backup_zomboid.sh
chown steam:steam /home/steam/backup_zomboid.sh

# Agregar cron job cada 12 horas
(sudo -u steam crontab -l 2>/dev/null; echo "0 */12 * * * /home/steam/backup_zomboid.sh >> /home/steam/backup.log 2>&1") | sudo -u steam crontab -
```

### 9.5. Probar que funciona

```bash
sudo -u steam /home/steam/backup_zomboid.sh
cat /home/steam/backup.log
```

Si ves "Backup subido exitosamente", ¡está funcionando!

---

## Paso 10: Administrar el servidor

### Comandos esenciales:

| Acción | Comando |
|---|---|
| **Iniciar servidor** | `sudo systemctl start pzserver` |
| **Detener servidor** | `sudo systemctl stop pzserver` |
| **Reiniciar servidor** | `sudo systemctl restart pzserver` |
| **Ver estado** | `sudo systemctl status pzserver` |
| **Ver consola en vivo** | `sudo -u steam screen -r pzserver` |
| **Salir de la consola** | `Ctrl + A` luego `D` |
| **Ver uso de RAM** | `free -h` |
| **Ver logs de backup** | `tail -f /home/steam/backup.log` |

### Actualizar el juego cuando sale una nueva versión:

```bash
sudo systemctl stop pzserver
sudo -u steam -i bash -c "/home/steam/steamcmd/steamcmd.sh +force_install_dir /home/steam/pzserver +login anonymous +app_update 380870 validate +quit"
sudo systemctl start pzserver
```

---

## ❌ Errores comunes y soluciones

### 1. "Mis amigos se desconectan después de unas horas"

**Causa:** El servidor se queda sin RAM y Linux lo mata (OOM Kill).

**Diagnóstico:**
```bash
journalctl -u pzserver --no-pager -n 50
```
Si ves `killed by the OOM killer` o `Failed with result 'oom-kill'`, es esto.

**Solución:**
- Reduce el heap JVM (ver [Paso 5](#paso-5-configurar-la-memoria-ram-correctamente))
- Crea swap (ver [Paso 6](#paso-6-crear-swap-muy-importante))

---

### 2. "No puedo conectarme al servidor"

**Checklist:**
- [ ] ¿Están abiertos los puertos UDP 16261 y 16262?
- [ ] ¿El servicio está corriendo? (`systemctl status pzserver`)
- [ ] ¿Estás usando la IP correcta?
- [ ] ¿El firewall interno permite los puertos? (`ufw allow 16261/udp && ufw allow 16262/udp`)

---

### 3. "El servidor tarda mucho en iniciar"

Es **normal**. La primera vez el servidor tarda 2-5 minutos en generar el mapa. Las siguientes veces tarda 1-2 minutos.

Para ver el progreso:
```bash
sudo -u steam screen -r pzserver
```

---

### 4. "El backup no sube a GitHub (error 403)"

**Causa:** El token de GitHub no tiene permisos de escritura.

**Solución:** 
- Usa un **token clásico** (empieza con `ghp_`), NO fine-grained
- Asegúrate de marcar el permiso `repo` al crearlo
- Verifica que el token no haya expirado

---

### 5. "El servidor usa mucha CPU (100% en un núcleo)"

Project Zomboid usa **un solo núcleo** para el hilo principal. Si está al 100%, las opciones son:
- Reducir el número de zombies en `SandboxVars.lua`
- Reducir la distancia de carga del mapa
- Migrar a un Droplet con CPU de mayor frecuencia

---

## ⚙️ Configuraciones recomendadas del juego

### servertest.ini (configuración del servidor)

| Parámetro | Valor recomendado | Descripción |
|---|---|---|
| `MaxPlayers` | `8` | Máximo de jugadores simultáneos |
| `PingLimit` | `400` | Ping máximo antes de desconectar (ms) |
| `PauseEmpty` | `true` | Pausar el mundo si no hay jugadores (ahorra CPU) |
| `UPnP` | `false` | Desactivar en servidor de nube (no es necesario) |
| `BloodSplatLifespanDays` | `3` | Limpiar sangre después de 3 días (reduce carga) |

### SandboxVars.lua (reglas del juego)

| Parámetro | Valor recomendado | Descripción |
|---|---|---|
| `HoursForCorpseRemoval` | `72-100` | Horas para eliminar cadáveres (reduce carga del mapa) |
| `HoursForWorldItemRemoval` | `24` | Horas para eliminar items del suelo |
| `RespawnHours` | `72` | Horas para que reaparezcan zombies en una zona |
| `RespawnUnseenHours` | `16` | Horas sin ver una zona para que reaparezcan zombies |

---

## 📋 Comandos útiles de referencia rápida

```bash
# === SERVIDOR ===
sudo systemctl start pzserver       # Iniciar
sudo systemctl stop pzserver        # Detener (guarda primero)
sudo systemctl restart pzserver     # Reiniciar
sudo systemctl status pzserver      # Ver estado

# === CONSOLA DEL JUEGO ===
sudo -u steam screen -r pzserver    # Entrar a la consola
# Ctrl+A luego D                    # Salir sin apagar

# === MONITOREO ===
free -h                              # Ver RAM y Swap
htop                                 # Monitor de CPU/RAM en tiempo real
journalctl -u pzserver -n 50        # Ver últimos 50 logs del servicio
tail -f /home/steam/backup.log      # Ver logs de backup en vivo

# === BACKUP MANUAL ===
sudo -u steam /home/steam/backup_zomboid.sh  # Forzar backup ahora

# === ACTUALIZAR JUEGO ===
sudo systemctl stop pzserver
sudo -u steam -i bash -c "/home/steam/steamcmd/steamcmd.sh +force_install_dir /home/steam/pzserver +login anonymous +app_update 380870 validate +quit"
sudo systemctl start pzserver

# === ARCHIVOS IMPORTANTES ===
# Configuración del servidor:
#   /home/steam/Zomboid/Server/servertest.ini
# Reglas del juego:
#   /home/steam/Zomboid/Server/servertest_SandboxVars.lua
# Memoria JVM:
#   /home/steam/pzserver/ProjectZomboid64.json
# Log de backups:
#   /home/steam/backup.log
```

---

## 🏗️ Estructura del repositorio

```
zomboid/
├── README.md           <-- Esta guía
├── Saves/
│   └── Multiplayer/
│       └── servertest/ <-- Mundo guardado (backup automático cada 12h)
├── Server/
│   ├── servertest.ini              <-- Configuración del servidor
│   └── servertest_SandboxVars.lua  <-- Reglas del juego
└── db/
    └── servertest.db   <-- Base de datos de jugadores
```

---

> 🎮 **¡Listo!** Si seguiste todos los pasos, tu servidor debería estar funcionando correctamente con backups automáticos y sin crashes por memoria. ¡A matar zombies! 🧟‍♂️
