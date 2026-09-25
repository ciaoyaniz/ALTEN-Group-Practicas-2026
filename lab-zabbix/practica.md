# lab-zabbix

**Objetivo**
Llevar Zabbix 7.0 LTS "de cero a experto" montando un laboratorio real de 11 hosts, y familiarizarse con:
- Monitorización de infraestructura con Zabbix (server, frontend, agentes, proxy)
- Docker Compose (despliegue de un servicio con varios contenedores)
- VirtualBox + Vagrant (VM como código)
- Zabbix Agent 2 (Linux y Windows) y agente clásico (Solaris)
- Monitorización de servicios: MySQL, PostgreSQL, Redis, Nginx, Docker, RabbitMQ y SNMP
- Alertas, dashboards, mapas, templates propios, backups y mantenimiento
---
Tareas
1. Preparar el anfitrión
- Habilitar la virtualización por hardware (VT-x / AMD-V) y WSL2
- Instalar Docker Desktop, VirtualBox 7.x y Vagrant
- Fijar la IP del PC (`192.168.1.100`) y reservar el rango `.100`–`.110` fuera del DHCP del router
---
2. Diseño de la infraestructura
- Documentar la tabla de los 11 hosts: nombre, rol, IP y cómo se crea
---
3. Zabbix Server en Docker
- Levantar con Docker Compose MySQL 8 + Zabbix Server 7.0 + frontend web
- Comprobar que el puerto `10051` es accesible desde la LAN y abrirlo en el firewall de Windows
- Primer login, cambiar la contraseña de `Admin`, idioma y zona horaria
- Saber recuperar la contraseña de `Admin` desde MySQL
---
4. Agentes Linux con Vagrant
- Crear un único `Vagrantfile` que levante las 8 VM Ubuntu 24.04, cada una con su IP fija
- Provisioning: un script común que instala Zabbix Agent 2 en modo activo y un script por rol con el servicio de cada VM
---
5. Alta de hosts
- Crear los host groups y dar de alta los hosts con el template `Linux by Zabbix agent active`
- Activar el host `Zabbix server` y comprobar que todo está en verde y llegan datos
---
6. Hosts especiales
- `zbx-agent-5`: OpenIndiana (Solaris libre) instalado a mano, con el agente clásico compilado desde el código fuente
- `zbx-agent-6`: Windows Server levantado con Vagrant (WinRM), con Zabbix Agent 2, promovido a Domain Controller e integrado por LDAP con el frontend
---
7. Seguridad
- Cifrar con PSK la comunicación de un agente Linux y del agente Windows
---
8. Visualización y alertas
- Crear un dashboard con widgets (Problems, Host availability, Data overview, Graph, Top hosts)
- Generar un problema real, reconocerlo (Ack) y crear una ventana de mantenimiento
- Enviar alertas por email (Gmail) y por Telegram
---
9. Descubrimiento automático
- Autorregistro de hosts con `HostMetadata`
- Autodiscovery por red
---
10. Monitorización de servicios
- SNMP (`zbx-agent-10`)
- MySQL, PostgreSQL y Redis con los plugins nativos de Agent 2
- Nginx con UserParameters
- Contenedores Docker con el plugin de Agent 2 (discovery LLD)
- RabbitMQ sin agente, con items HTTP Agent y preprocessing JSONPath
---
11. Avanzado
- Mapa de red de los 11 hosts
- Template propio desde cero (items, item calculado, triggers, gráfico) y exportarlo en YAML
- Zabbix Proxy simulando una sede remota
- Backup y restauración de la base de datos
- Integración con Grafana
- Procedimiento de actualización de versión con rollback
- Housekeeper y rutina de mantenimiento
---
12. Documentación
Documentar en un archivo:
- pasos realizados
- problemas encontrados
- cómo los resolvieron
---
Requisito deseable:
- Subir la documentación al repo git con sus respectivos commits.
- No copiar/pegar sin entender — la idea es investigar y probar.
---
El objetivo final es tener un Zabbix Server en Docker monitorizando 10 agentes (Linux, Windows y Solaris), con servicios monitorizados por tres métodos distintos (plugin nativo, UserParameters y HTTP Agent), alertas funcionando, proxy, backups probados y un mapa de todo el laboratorio.
