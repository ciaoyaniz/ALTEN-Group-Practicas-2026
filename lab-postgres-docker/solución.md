# Solución

## PASO 1: Configuraciones generales

### Instalar Ubuntu 24 y preparar el entorno

> [!WARNING] Cambio de entorno
> Al comienzo de este laboratorio cree VMs con Vagrant y boxes `bento/ubuntu-24.04`, pero, debido a problemas de compatibilidad al instalar PostgreSQL, decidí rehacer las VMs con sus ISO oficiales de Ubuntu 24.x.

![](capturas/Pasted%20image%2020260921234441.png)

Creación de dos VM con Ubuntu 24.04 LTS (el usuario de trabajo no root que voy a usar lo he creado durante la instalación, llamado `ciaoyaniz`). 

Además he habilitado los servicios SSH para poder conectarme de forma remota a ellas.

> **Red**: en modo Bridged (Adaptador Puente), la VM se comporta como una máquina más de la red local (192.168.1.0), para poder ver a la otra VM por IP.

### Actualización del sistema

![](capturas/Pasted%20image%2020260921223035.png)

![](capturas/Pasted%20image%2020260921223051.png)

Inicié sesión desde mi host anfitrión mediante el servicio SSH con el comando `ssh ciaoyaniz@192.168.1.x`. Previamente obtuve la IP asignada a cada VM con el comando `ip a`. 

Una vez dentro de cada sistema, los actualicé:
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### Asignar permisos `sudo` a mi usuario

![](capturas/Pasted%20image%2020260921224242.png)

`usermod -aG sudo` agrega mi usuario al grupo `sudo`, habilitándome a ejecutar comandos administrativos con `sudo` sin ser `root`.

El proceso lo he repetido en las dos VM.

## PASO 2: Configurar VM con Docker
### Instalar Docker

He creado un script con ayuda de IA para poder instalar de forma rápida el servicio de Docker con sus dependencias y claves GPG ya que este es un paso crucial y no quería detenerme en ello.

Para crearlo y ejecutarlo utilice los comandos de edición, permisos de ejecución y al final ejecución del script:

```bash
nano install-docker.sh
chmod +x install-docker.sh
./install-docker.sh
```

```bash
#!/bin/bash

# Script para limpiar configuración anterior y instalar Docker correctamente
set -e

echo "========================================"
echo "Limpiando e instalando Docker"
echo "========================================"

# 1. Limpiar configuraciones anteriores
echo "[1/5] Limpiando configuraciones anteriores..."
sudo rm -f /etc/apt/sources.list.d/docker.list
sudo rm -f /etc/apt/keyrings/docker.asc
echo "✓ Configuraciones previas eliminadas"

# 2. Actualizar repositorios ANTES de agregar Docker
echo "[2/5] Actualizando repositorios..."
sudo apt clean
sudo apt update
sudo apt upgrade -y

# 3. Instalar dependencias
echo "[3/5] Instalando dependencias..."
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https

# 4. Crear directorio y descargar clave GPG
echo "[4/5] Agregando repositorio de Docker..."
sudo mkdir -p /etc/apt/keyrings

# Descargar la clave GPG
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Agregar repositorio
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Actualizar repositorios CON la clave ya instalada
sudo apt update

# 5. Instalar Docker
echo "[5/5] Instalando Docker..."
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Verificar instalación
echo ""
echo "========================================"
echo "✓ Docker instalado exitosamente"
echo "========================================"
docker --version
docker compose version

# Configurar usuario
echo ""
echo "⚙️  Configurando permisos..."
sudo usermod -aG docker $USER

# Iniciar servicio
sudo systemctl start docker
sudo systemctl enable docker

echo ""
echo "✅ ¡Instalación completada!"
echo ""
echo "IMPORTANTE: Debes cerrar sesión y volver a conectarte"
echo "Ejecuta: exit"
echo "Luego reconéctate y prueba: docker run hello-world"
```

![](capturas/Pasted%20image%2020260921234941.png)

El proceso de instalación ha sido un poco lento pero el script ha funcionado correctamente:


## PASO 3: Configurar VM con PostgreSQL

### Instalar PostgreSQL sobre Ubuntu

