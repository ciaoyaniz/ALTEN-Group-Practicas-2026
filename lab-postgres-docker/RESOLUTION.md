## Paso 1: crear las dos VMs

Cada VM contiene:
- VM 1 (Docker + app)
- VM 2 (PostgreSQL)

![VMs creadas en VirtualBox: vm-docker-app y vm-postgres](./capturas/01-virtualbox-vms-creadas.png)

**Red**: en modo Bridged, la VM se comporta como una máquina más de la red local, para poder ver a la otra VM por IP.

Instalación de las dos VMs:

![Instalación de Ubuntu corriendo en ambas VMs en paralelo](./capturas/02-instalacion-ubuntu-ambas-vms.png)


PROBLEMA: la intalación se estaba demorando muchisimo y decidi hacerlo con BOXES de VAGRANT,

```Vagrantfile
Vagrant.configure("2") do |config|

  config.vm.define "docker-app" do |app|
    app.vm.box = "ubuntu/focal64"         
    app.vm.hostname = "vm-docker-app"

    app.vm.network "public_network"

    app.vm.provider "virtualbox" do |vb|
      vb.name = "vm-docker-app"
      vb.memory = 2048
      vb.cpus = 2
    end
  end

  config.vm.define "postgres" do |db|
    db.vm.box = "ubuntu/focal64"           
    db.vm.hostname = "vm-postgres"

    db.vm.network "public_network"

    db.vm.provider "virtualbox" do |vb|
      vb.name = "vm-postgres"
      vb.memory = 2048
      vb.cpus = 2
    end
    
end
```
---

Actualización del sistema:

![Actualización del sistema en vm-docker-app (apt update && apt upgrade)](./capturas/03-actualizacion-sistema-docker-app.png)
![Actualización del sistema en vm-postgres (apt update && apt upgrade)](./capturas/04-actualizacion-sistema-postgres.png)

---

creación de users:

vagrant ya viene con un user por defecto llamado vagrant pero a terminos practicos voy a crear otro nuevo

![Creación del usuario de trabajo en vm-docker-app](./capturas/05-creacion-usuario-docker-app.png)

en ambas vms he creado mi user

![Creación del usuario de trabajo en vm-postgres](./capturas/06-creacion-usuario-postgres.png)

----
