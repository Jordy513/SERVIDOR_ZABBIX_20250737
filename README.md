# Monitoreo de Red con Zabbix + SNMP

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo de la Red](#1-objetivo-de-la-red)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
   - [Diagrama de Topología](#21-diagrama-de-topología)
   - [Tabla de Dispositivos](#22-tabla-de-dispositivos)
3. [Parte 1 — Router Cisco](#3-parte-1--router-cisco)
   - [3.1 Configuración de Interfaces](#31-configuración-de-interfaces)
   - [3.2 Servidor DHCP](#32-servidor-dhcp)
   - [3.3 Comunidad SNMP](#33-comunidad-snmp)
4. [Parte 2 — Switch Cisco](#4-parte-2--switch-cisco)
   - [4.1 IP de Gestión](#41-ip-de-gestión)
   - [4.2 Comunidad SNMP](#42-comunidad-snmp)
5. [Parte 3 — Servidor Zabbix (Ubuntu)](#5-parte-3--servidor-zabbix-ubuntu)
   - [5.1 Configuración de IP Estática](#51-configuración-de-ip-estática)
   - [5.2 Instalación de Zabbix](#52-instalación-de-zabbix)
   - [5.3 Acceso a la GUI de Zabbix](#53-acceso-a-la-gui-de-zabbix)
   - [5.4 Agregar el Router como Equipo](#54-agregar-el-router-como-equipo)
   - [5.5 Agregar el Switch como Equipo](#55-agregar-el-switch-como-equipo)
   - [5.6 Verificar Eventos y Datos SNMP](#56-verificar-eventos-y-datos-snmp)
6. [Parte 4 — PC Cliente](#6-parte-4--pc-cliente)
   - [6.1 Verificar IP por DHCP](#61-verificar-ip-por-dhcp)
   - [6.2 Verificar Datos SNMP desde el Cliente](#62-verificar-datos-snmp-desde-el-cliente)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo de la Red

Esta topología implementa un sistema de monitoreo centralizado mediante **Zabbix** sobre una red LAN corporativa. Los objetivos específicos son:

* Configurar un Router Cisco como gateway de la red LAN `10.7.37.0/24`, con servidor DHCP para los clientes y comunidad SNMP de solo lectura para permitir el monitoreo.
* Configurar un Switch Cisco con IP de gestión y comunidad SNMP de solo lectura para ser monitoreado por Zabbix.
* Desplegar un servidor **Zabbix en Ubuntu** con IP estática dentro de la LAN, accesible desde el host Windows mediante la GUI web.
* Verificar desde el Zabbix los eventos, interfaces, carga de CPU y otros datos de telemetría del Router y el Switch mediante SNMP v2c.
* Verificar desde el PC Cliente (host Windows) la asignación DHCP y los datos SNMP de los dispositivos de red.

El direccionamiento IP está derivado de la matrícula `20250737` — últimos 4 dígitos `0737` → `XX.XX = 07.37`. La red utilizada es `10.7.37.0/24` para toda la topología — R1 actúa como gateway, DHCP server y agente SNMP sobre su única interfaz `e0/0`.

---

## 2. Topología y Direccionamiento

### 2.1 Diagrama de Topología

```
                         10.7.37.0/24
                               │
                        e0/0: 10.7.37.1
                       ┌───────┴───────┐
                       │    Router R1  │
                       │  Cisco IOS    │
                       └───────┬───────┘
                          e0/0 │
                       ┌───────┴───────┐
                       │    Switch     │
                       │  Cisco IOS    │
                       └──┬────────┬───┘
                        e1│        │e2
               ┌──────────┘        └──────────┐
               │ e0                        e0 │
       ┌───────┴───────┐           ┌──────────┴───────┐
       │    W10        │           │     ZABBIX       │
       │  (Host Win.)  │           │   10.7.37.253    │
       │  DHCP:        │           │   (Estática)     │
       │  10.7.37.X    │           └──────────────────┘
       └───────────────┘

  Monitoreo SNMP:
  Zabbix (10.7.37.253) ──SNMP v2c──► R1 (10.7.37.1)
  Zabbix (10.7.37.253) ──SNMP v2c──► Switch (10.7.37.2)

  Acceso GUI Zabbix desde W10:
  W10 ──HTTP──► http://10.7.37.253/zabbix
```

### 2.2 Tabla de Dispositivos

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway | Método | Rol |
|---|---|---|---|---|---|---|
| **R1** | e0/0 | 10.7.37.1 | /24 | — | Estática | Gateway LAN + DHCP + SNMP |
| **Switch** | VLAN 1 (SVI) | 10.7.37.2 | /24 | 10.7.37.1 | Estática | Conmutación + SNMP |
| **Ubuntu (Zabbix)** | ens34 | 10.7.37.253 | /24 | 10.7.37.1 | Estática | Servidor de monitoreo |
| **W10 (Host)** | e0 | 10.7.37.X | /24 | 10.7.37.1 | **DHCP** | Cliente + acceso GUI Zabbix |

**Parámetros SNMP:**

| Parámetro | Valor |
|---|---|
| Versión | SNMPv2c |
| Comunidad (Router) | `public_ro` |
| Comunidad (Switch) | `public_ro` |
| Puerto | UDP 161 |
| Tipo de acceso | Solo lectura (read-only) |

**Parámetros DHCP:**

| Parámetro | Valor |
|---|---|
| Red | `10.7.37.0/24` |
| Gateway | `10.7.37.1` |
| Rango | `10.7.37.10 – 10.7.37.200` |
| DNS | `8.8.8.8`, `8.8.4.4` |
| Excluidos | `10.7.37.1 – 10.7.37.9` (infraestructura) |

---

## 3. Parte 1 — Router R1

### 3.1 Configuración de Interfaces

```cisco
hostname R1

interface ethernet0/0
 description RED-LAN
 ip address 10.7.37.1 255.255.255.0
 no shutdown
```

Verificar:

```cisco
show ip interface brief
```

> Ver evidencia: [01_router_interfaces.png](#01_router_interfacespng)

---

### 3.2 Servidor DHCP

```cisco
! Excluir IPs de infraestructura del pool
ip dhcp excluded-address 10.7.37.1 10.7.37.9

! Pool DHCP para la LAN
ip dhcp pool LAN-POOL
 network 10.7.37.0 255.255.255.0
 default-router 10.7.37.1
 dns-server 8.8.8.8 8.8.4.4
 lease 1
```

Verificar:

```cisco
show ip dhcp pool
show ip dhcp binding
```

> Ver evidencia: [02_router_dhcp.png](#02_router_dhcppng)

---

### 3.3 Comunidad SNMP

```cisco
! Comunidad de solo lectura para Zabbix
snmp-server community public_ro ro

! Información del sistema (visible en Zabbix)
snmp-server location Lab-ITLA-20250737
snmp-server contact Jordy.Rosario@itla.edu.do

! Habilitar traps hacia Zabbix
snmp-server enable traps
snmp-server host 10.7.37.253 version 2c public_ro
```

Verificar:

```cisco
show snmp
show snmp community
```

> Ver evidencia: [03_router_snmp.png](#03_router_snmppng)
>
> Nota: puede aparecer también una comunidad `ILMI` de solo lectura creada automáticamente por el subsistema ATM del IOS. Es un artefacto normal de la imagen de IOS usada en el simulador, no algo configurado — Zabbix no la utiliza.

---

## 4. Parte 2 — Switch Cisco

### 4.1 IP de Gestión

```cisco
hostname Switch-Lab

! IP de gestión en VLAN 1
interface vlan 1
 ip address 10.7.37.2 255.255.255.0
 no shutdown

! Gateway para alcanzar Zabbix
ip default-gateway 10.7.37.1
```

> Ver evidencia: [04_switch_ip.png](#04_switch_ippng)

---

### 4.2 Comunidad SNMP

```cisco
! Comunidad de solo lectura
snmp-server community public_ro ro

! Información del sistema
snmp-server location Lab-ITLA-20250737
snmp-server contact Jordy.Rosario@itla.edu.do

! Traps hacia Zabbix
snmp-server enable traps
snmp-server host 10.7.37.253 version 2c public_ro
```

Verificar:

```cisco
show snmp
show snmp community
```

> Ver evidencia: [05_switch_snmp.png](#05_switch_snmppng)

---

## 5. Parte 3 — Servidor Zabbix (Ubuntu)

### 5.1 Configuración de IP Estática

El servidor Zabbix corre sobre **Ubuntu 24.04 LTS**, con `NetworkManager` como renderer de Netplan (por eso las interfaces se declaran de forma mínima).

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Contenido:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
      dhcp4: true
    ens34:
      dhcp4: false
      addresses:
        - 10.7.37.253/24
      routes:
        - to: default
          via: 10.7.37.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

`ens33` mantiene DHCP para salida a internet (descarga de paquetes); `ens34` es la interfaz estática hacia la LAN del laboratorio.

Aplicar:

```bash
sudo netplan apply
ip addr show ens34
```

> Ver evidencia: [06_ubuntu_ip.png](#06_ubuntu_ippng)

---

### 5.2 Instalación de Zabbix

Instalado: **Zabbix 7.0 LTS sobre Ubuntu 24.04 LTS** (Ubuntu 20.04 no publica paquetes de `zabbix-server-mysql` ni `zabbix-frontend-php` en el repositorio oficial de Zabbix — solo agente y sql-scripts — por lo que el servidor requiere 22.04 o superior).

```bash
# Repositorio oficial de Zabbix 7.0 LTS para Ubuntu 24.04
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb
sudo apt update

# Instalar Zabbix server, frontend, apache-conf, sql-scripts y agente
sudo apt install zabbix-server-mysql zabbix-frontend-php \
    zabbix-apache-conf zabbix-sql-scripts zabbix-agent -y

# Instalar MariaDB
sudo apt install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Crear base de datos y usuario
sudo mysql -uroot -e "
  CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
  CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'ZabbixPass123!';
  GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
  SET GLOBAL log_bin_trust_function_creators = 1;
  FLUSH PRIVILEGES;"

# Importar esquema de base de datos
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
    mysql --default-character-set=utf8mb4 -uzabbix -pZabbixPass123! zabbix

# Desactivar el flag una vez importado el schema
sudo mysql -uroot -e "SET GLOBAL log_bin_trust_function_creators = 0;"

# Configurar contraseña de BD en Zabbix
sudo sed -i 's/# DBPassword=/DBPassword=ZabbixPass123!/' \
    /etc/zabbix/zabbix_server.conf

# Iniciar servicios
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

Verificar que el servicio esté corriendo:

```bash
sudo systemctl status zabbix-server
```

> Ver evidencia: [07_zabbix_instalado.png](#07_zabbix_instaladopng)

---

### 5.3 Acceso a la GUI de Zabbix

Desde el **host Windows**, abrir el navegador y navegar a:

```
http://10.7.37.253/zabbix
```

Seguir el asistente de instalación web (`setup.php`):

1. **Bienvenido**
2. **Comprobación de requisitos previos** — todo debe quedar en verde (versión de PHP, memory_limit, extensiones, etc.)
3. **Configurar la conexión de BD** — tipo `MySQL`, servidor `localhost`, puerto `0` (por defecto), nombre de BD `zabbix`, usuario `zabbix`, contraseña `ZabbixPass123!`
4. **Ajustes** — nombre del servidor Zabbix, zona horaria (`America/Santo_Domingo`), tema
5. **Resumen de preinstalación**
6. **Instalar** → Finalizar

> Ver evidencia: [08_zabbix_gui_login.png](#08_zabbix_gui_loginpng) y [09_zabbix_setup_ok.png](#09_zabbix_setup_okpng)

**Credenciales por defecto (después de finalizar el asistente):**

| Campo | Valor |
|---|---|
| Usuario | `Admin` (con A mayúscula) |
| Contraseña | `zabbix` |

> Cambiar la contraseña al primer ingreso desde `Configuración de usuario → Contraseña`.

---

### 5.4 Agregar el Router como Equipo

> ⚠️ En Zabbix 6.4+ el antiguo menú **"Configuration"** se renombró a **"Data collection"** (`Recopilación de datos` en español). Ya no existe un menú llamado "Configuration".

**Ruta GUI:** `Recopilación de datos → Equipos → Crear equipo` (botón arriba a la derecha)

**Pestaña Equipo:**

| Campo | Valor |
|---|---|
| Nombre de host | `R1` |
| Nombre visible | `Router R1 — 10.7.37.1` |
| Grupos | `Network devices` |
| Interfaces → Agregar → SNMP | |
| Dirección IP | `10.7.37.1` |
| Puerto | `161` |

**Pestaña Plantillas:**

| Campo | Valor |
|---|---|
| Plantillas | `Cisco IOS SNMP` (buscar y seleccionar) |

**Pestaña Macros:**

| Macro | Valor |
|---|---|
| `{$SNMP_COMMUNITY}` | `public_ro` |
| `{$SNMP.TIMEOUT}` | `5s` |

Clic en **Agregar** para guardar.

> Después de unos minutos el ícono del equipo cambiará de gris a verde indicando que Zabbix recibió datos SNMP correctamente.

> Ver evidencia: [10_zabbix_host_router.png](#10_zabbix_host_routerpng) y [11_zabbix_router_verde.png](#11_zabbix_router_verdepng)

---

### 5.5 Agregar el Switch como Equipo

**Ruta GUI:** `Recopilación de datos → Equipos → Crear equipo`

**Pestaña Equipo:**

| Campo | Valor |
|---|---|
| Nombre de host | `Switch-Lab` |
| Nombre visible | `Switch Cisco — 10.7.37.2` |
| Grupos | `Network devices` |
| Interfaces → Agregar → SNMP | |
| Dirección IP | `10.7.37.2` |
| Puerto | `161` |

**Pestaña Plantillas:**

| Campo | Valor |
|---|---|
| Plantillas | `Cisco IOS SNMP` |

**Pestaña Macros:**

| Macro | Valor |
|---|---|
| `{$SNMP_COMMUNITY}` | `public_ro` |

Clic en **Agregar** para guardar.

> Ver evidencia: [12_zabbix_host_switch.png](#12_zabbix_host_switchpng) y [13_zabbix_switch_verde.png](#13_zabbix_switch_verdepng)

---

### 5.6 Verificar Eventos y Datos SNMP

**Verificar datos de monitoreo del Router:**

**Ruta:** `Monitorización → Últimos datos` → filtrar por equipo `R1`

Debe mostrar métricas SNMP como:

| Métrica | OID SNMP |
|---|---|
| CPU utilization | `1.3.6.1.4.1.9.2.1.58.0` |
| Uptime | `1.3.6.1.2.1.1.3.0` |
| Interface traffic (in/out) | `1.3.6.1.2.1.2.2.1.10/16` |
| System description | `1.3.6.1.2.1.1.1.0` |

> Ver evidencia: [14_zabbix_latest_data_router.png](#14_zabbix_latest_data_routerpng)

**Verificar eventos:**

**Ruta:** `Monitorización → Problemas`

Aquí aparecen las alertas generadas automáticamente por los templates — por ejemplo si una interfaz del router baja, Zabbix genera un evento de problema.

> Ver evidencia: [15_zabbix_events.png](#15_zabbix_eventspng)

**Verificar gráficas del Switch:**

**Ruta:** `Monitorización → Últimos datos` → filtrar por equipo `Switch-Lab` → clic en el ítem deseado → ícono de gráfico

> Ver evidencia: [16_zabbix_graphs_switch.png](#16_zabbix_graphs_switchpng)

---

## 6. Parte 4 — PC Cliente

### 6.1 Verificar IP por DHCP

Desde el **host Windows**, abrir `cmd` o `PowerShell`:

```cmd
ipconfig /all
```

Verificar que el adaptador de red muestra:

| Campo | Valor esperado |
|---|---|
| Dirección IPv4 | `10.7.37.X` (dentro del rango `.10–.200`) |
| Máscara | `255.255.255.0` |
| Puerta de enlace | `10.7.37.1` |
| Servidor DHCP | `10.7.37.1` |
| DNS | `8.8.8.8` |

> Ver evidencia: [17_cliente_dhcp.png](#17_cliente_dhcppng)

---

### 6.2 Verificar Datos SNMP desde el Cliente

Instalar **SNMP Tester** o usar `snmpwalk` desde el host Windows para verificar que el Router y el Switch responden a consultas SNMP.

**Opción A — snmpwalk en Windows (requiere Net-SNMP instalado):**

```cmd
snmpwalk -v2c -c public_ro 10.7.37.1 system
snmpwalk -v2c -c public_ro 10.7.37.2 system
```

*Salida esperada:*

```
SNMPv2-MIB::sysDescr.0 = STRING: Cisco IOS Software...
SNMPv2-MIB::sysUpTime.0 = Timeticks: (XXXXXX) X days, ...
SNMPv2-MIB::sysContact.0 = STRING: Jordy.Rosario@itla.edu.do
SNMPv2-MIB::sysLocation.0 = STRING: Lab-ITLA-20250737
```

**Opción B — GUI con iReasoning MIB Browser (más visual):**

1. Descargar e instalar [iReasoning MIB Browser](https://ireasoning.com/mibbrowser.shtml)
2. Address: `10.7.37.1` → Community: `public_ro` → Version: `v2c`
3. Expandir el árbol MIB y consultar el nodo `system`

> Ver evidencia: [18_cliente_snmpwalk_router.png](#18_cliente_snmpwalk_routerpng) y [19_cliente_snmpwalk_switch.png](#19_cliente_snmpwalk_switchpng)

---

## 7. Capturas de Pantalla

Todas las capturas están en la carpeta [`screenshots/`](screenshots/).

| # | Archivo | Descripción |
|---|---|---|
| 01 | [`01_router_interfaces.png`](screenshots/01_router_interfaces.png) | `show ip interface brief` en R1 mostrando `e0/0: 10.7.37.1` en estado `up/up`. |
| 02 | [`02_router_dhcp.png`](screenshots/02_router_dhcp.png) | `show ip dhcp binding` en el Router mostrando al menos un lease asignado al PC Cliente y al servidor Ubuntu, con sus IPs del rango `.10–.200`. |
| 03 | [`03_router_snmp.png`](screenshots/03_router_snmp.png) | `show snmp community` en el Router confirmando la comunidad `public_ro` con acceso `NOAUTHNOPRIV` y permisos `ro`. |
| 04 | [`04_switch_ip.png`](screenshots/04_switch_ip.png) | `show interface vlan 1` en el Switch mostrando la IP `10.7.37.2/24` en estado `up/up`. |
| 05 | [`05_switch_snmp.png`](screenshots/05_switch_snmp.png) | `show snmp community` en el Switch confirmando la comunidad `public_ro` con permisos `ro`. |
| 06 | [`06_ubuntu_ip.png`](screenshots/06_ubuntu_ip.png) | Terminal Ubuntu mostrando `ip addr show ens34` con la IP estática `10.7.37.253/24` configurada y activa. |
| 07 | [`07_zabbix_instalado.png`](screenshots/07_zabbix_instalado.png) | Terminal Ubuntu mostrando `systemctl status zabbix-server` con estado `active (running)`. |
| 08 | [`08_zabbix_gui_login.png`](screenshots/08_zabbix_gui_login.png) | Navegador del host Windows mostrando la pantalla de login de Zabbix en `http://10.7.37.253/zabbix`. |
| 09 | [`09_zabbix_setup_ok.png`](screenshots/09_zabbix_setup_ok.png) | Asistente de instalación web de Zabbix en el paso "Comprobación de requisitos previos" con todo en verde. |
| 10 | [`10_zabbix_host_router.png`](screenshots/10_zabbix_host_router.png) | Formulario `Crear equipo` en Zabbix mostrando R1 configurado con interfaz SNMP `10.7.37.1:161`, plantilla `Cisco IOS SNMP` y macro `{$SNMP_COMMUNITY}=public_ro`. |
| 11 | [`11_zabbix_router_verde.png`](screenshots/11_zabbix_router_verde.png) | Lista de equipos en Zabbix mostrando `R1` con ícono verde — confirmando que Zabbix recibe datos SNMP del Router correctamente. |
| 12 | [`12_zabbix_host_switch.png`](screenshots/12_zabbix_host_switch.png) | Formulario `Crear equipo` mostrando el Switch configurado con interfaz SNMP `10.7.37.2:161` y plantilla `Cisco IOS SNMP`. |
| 13 | [`13_zabbix_switch_verde.png`](screenshots/13_zabbix_switch_verde.png) | Lista de equipos mostrando `Switch-Lab` con ícono verde — Zabbix recibiendo datos SNMP del Switch. |
| 14 | [`14_zabbix_latest_data_router.png`](screenshots/14_zabbix_latest_data_router.png) | `Monitorización → Últimos datos` filtrado por `R1` mostrando métricas SNMP activas: uptime, descripción del sistema, tráfico de interfaces. |
| 15 | [`15_zabbix_events.png`](screenshots/15_zabbix_events.png) | `Monitorización → Problemas` mostrando los eventos generados por las plantillas de Cisco IOS SNMP. |
| 16 | [`16_zabbix_graphs_switch.png`](screenshots/16_zabbix_graphs_switch.png) | `Monitorización → Últimos datos` filtrado por `Switch-Lab` mostrando al menos una gráfica de tráfico o CPU activa con datos. |
| 17 | [`17_cliente_dhcp.png`](screenshots/17_cliente_dhcp.png) | `ipconfig /all` en el host Windows mostrando IP `10.7.37.X` asignada por DHCP con gateway `10.7.37.1` y servidor DHCP `10.7.37.1`. |
| 18 | [`18_cliente_snmpwalk_router.png`](screenshots/18_cliente_snmpwalk_router.png) | Terminal del host Windows mostrando la salida de `snmpwalk` hacia `10.7.37.1` con los campos `sysDescr`, `sysUpTime`, `sysContact` y `sysLocation` del Router. |
| 19 | [`19_cliente_snmpwalk_switch.png`](screenshots/19_cliente_snmpwalk_switch.png) | Terminal del host Windows mostrando la salida de `snmpwalk` hacia `10.7.37.2` con los datos SNMP del Switch. |

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

> *(Enlace disponible en `videos.txt` en la raíz del repositorio)*

**Contenido del video:**

* ✅ Topología en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica.
* ✅ `show ip interface brief` en el Router — interfaces up/up con IPs correctas.
* ✅ `show ip dhcp binding` en el Router — PC Cliente con IP asignada por DHCP.
* ✅ `show snmp community` en el Router y el Switch — comunidad `public_ro`.
* ✅ `ipconfig /all` en el host Windows — IP DHCP del rango `10.7.37.10–.200`.
* ✅ Login en la GUI de Zabbix desde el host Windows (`http://10.7.37.253/zabbix`).
* ✅ Lista de equipos en Zabbix — Router y Switch con ícono verde.
* ✅ `Monitorización → Últimos datos` del Router mostrando métricas SNMP activas.
* ✅ `Monitorización → Problemas` mostrando eventos de los dispositivos monitoreados.
* ✅ `snmpwalk` desde el host Windows hacia el Router y el Switch.

---

## 9. Referencias

* Zabbix LLC. (2026). *Zabbix 7.0 LTS Documentation — Installation on Ubuntu*. zabbix.com.
* Zabbix LLC. (2026). *Zabbix 7.0 LTS Documentation — SNMP Monitoring*.
* Cisco Systems. (2024). *Cisco IOS Network Management Command Reference — SNMP*.
* Case, J. et al. (1990). *RFC 1157 — A Simple Network Management Protocol (SNMP)*. IETF.
* Stallings, W. (2022). *Cryptography and Network Security: Principles and Practice (8th Ed.)*. Pearson.
