# Implementación de Monitoreo con Nagios, MySQL y NGINX usando Docker

## Estructura del Proyecto

/opt/monitoring/
├── docker-compose.yml
├── nagios/
│   ├── etc/
│   │   ├── nagios.cfg
│   │   ├── commands.cfg
│   │   ├── contacts.cfg
│   │   ├── hosts.cfg
│   │   ├── localhost.cfg
│   │   ├── services.cfg
│   │   └── resource.cfg
│   └── var/
└── nginx/
    ├── conf/
    └── html/


## 1. Configuración Inicial

### 1.1 Crear estructura de directorios
bash
sudo mkdir -p /opt/monitoring
cd /opt/monitoring
sudo mkdir -p nagios/{etc,var}
sudo mkdir -p nginx/{conf,html}


### 1.2 Docker Compose
yaml
version: '3'

services:
  mysql:
    image: mysql:5.7
    container_name: mysql_container
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_USER: user
      MYSQL_PASSWORD: userpassword123
      MYSQL_DATABASE: parcial
    ports:
      - "3306:3306"
    networks:
      - red_parcial_docker

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin_container
    environment:
      PMA_HOST: mysql 
      MYSQL_ROOT_PASSWORD: rootpassword
    ports:
      - "8081:80"
    networks:
      - red_parcial_docker

networks:
  :
    external: true


## 2. Archivos de Configuración de Nagios

### 2.1 nagios.cfgred_parcial_docker
Ubicación: /opt/monitoring/nagios/etc/nagios.cfg
ini
# Configuración básica
check_result_path=/opt/nagios/var/spool/checkresults
resource_file=/opt/nagios/etc/resource.cfg

# Archivos de configuración
cfg_file=/opt/nagios/etc/commands.cfg
cfg_file=/opt/nagios/etc/contacts.cfg
cfg_file=/opt/nagios/etc/timeperiods.cfg
cfg_file=/opt/nagios/etc/templates.cfg
cfg_file=/opt/nagios/etc/hosts.cfg
cfg_file=/opt/nagios/etc/services.cfg
cfg_file=/opt/nagios/etc/localhost.cfg


### 2.2 hosts.cfg
cfg
# MySQL Server
define host {
    use                     linux-server
    host_name               mysql_server
    alias                   MySQL Database Server
    address                 mysql
    check_command           check-host-alive
    max_check_attempts      5
    check_period           24x7
    notification_interval   30
    notification_period     24x7
}

# NGINX Server
define host {
    use                     linux-server
    host_name               nginx_server
    alias                   NGINX Web Server
    address                 nginx
    check_command           check-host-alive
    max_check_attempts      5
    check_period           24x7
    notification_interval   30
    notification_period     24x7
}

# Define hostgroups
define hostgroup {
    hostgroup_name          web-servers
    alias                   Web Servers
    members                 nginx_server
}

define hostgroup {
    hostgroup_name          db-servers
    alias                   Database Servers
    members                 mysql_server
}


### 2.3 services.cfg
cfg
# MySQL Services
define service {
    use                    generic-service
    host_name              mysql_server
    service_description    MySQL Connection Status
    check_command         check_tcp!3306
    check_interval        5
    retry_interval        1
    max_check_attempts    5
}

define service {
    use                    generic-service
    host_name              mysql_server
    service_description    MySQL Database
    check_command         check_mysql!nagios!nagios123!testdb
    check_interval        5
    retry_interval        1
    max_check_attempts    5
}

# NGINX Services
define service {
    use                    generic-service
    host_name              nginx_server
    service_description    NGINX HTTP
    check_command         check_http!-p 8081
    check_interval        5
    retry_interval        1
    max_check_attempts    5
}

define service {
    use                    generic-service
    host_name              nginx_server
    service_description    NGINX Process
    check_command         check_procs!1:!nginx
    check_interval        5
    retry_interval        1
    max_check_attempts    5
}


### 2.4 resource.cfg
cfg
$USER1$=/usr/lib/nagios/plugins


## 3. Configuración de MySQL
1. Acceder al contenedor MySQL:
bash
sudo docker-compose exec mysql bash
mysql -u root -prootpass


2. Configurar permisos:
sql
CREATE USER 'nagios'@'%' IDENTIFIED BY 'nagios123';
GRANT ALL PRIVILEGES ON *.* TO 'nagios'@'%';
FLUSH PRIVILEGES;


## 4. Instalación de Plugins en Nagios
bash
sudo docker-compose exec nagios bash
apt-get update
apt-get install -y nagios-plugins


## 5. Acceso Web
- URL: http://IP-DEL-SERVIDOR:8080/nagios
- Usuario: nagiosadmin
- Contraseña: nagios123

## 6. Estado Actual
- Monitoreo de MySQL: ✅ Funcionando
  * Connection Status: OK
  * Database: OK
- Monitoreo de NGINX: ⚠ Parcial
  * HTTP: En progreso
  * Process: En progreso
- Monitoreo de Host Local: ✅ Funcionando
  * Load: OK
  * Disk: OK
  * Memory: En ajuste

## 7. Solución de Problemas Comunes
1. Error de permisos en checkresults:
bash
mkdir -p /opt/monitoring/nagios/var/spool/checkresults
chmod -R 775 /opt/monitoring/nagios/var/spool


2. Error de acceso MySQL:
sql
GRANT ALL PRIVILEGES ON *.* TO 'nagios'@'%';
FLUSH PRIVILEGES;


## 8. Comandos Útiles
bash
# Verificar configuración de Nagios
sudo docker-compose exec nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg

# Reiniciar servicios
sudo docker-compose restart

# Ver logs
sudo docker-compose logs nagios
