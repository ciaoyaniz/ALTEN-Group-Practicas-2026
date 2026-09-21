## Paso 1: crear las dos VMs

Cada VM contiene:
- VM 1 (Docker + app)
- VM 2 (PostgreSQL)

![VMs creadas en VirtualBox: vm-docker-app y vm-postgres](./capturas/01-virtualbox-vms-creadas.png)

**Red**: en modo Bridged, la VM se comporta como una máquina más de la red local, para poder ver a la otra VM por IP.

Instalación de las dos VMs:

![Instalación de Ubuntu corriendo en ambas VMs en paralelo](./capturas/02-instalacion-ubuntu-ambas-vms.png)
