# Capturas de pantalla — Monitoreo de Red con Zabbix + SNMP

| # | Archivo | Descripción |
|---|---|---|
| 01 | [`01_router_interfaces.png`](/screenshots/01_router_interfaces.png) | `show ip interface brief` en R1 mostrando `e0/0: 10.7.37.2` en estado `up/up`. |
| 02 | [`02_router_dhcp.png`](/screenshots/02_router_dhcp.png) | `show ip dhcp binding` en el Router mostrando al menos un lease asignado al PC Cliente y al servidor Ubuntu, con sus IPs del rango `.10–.200`. |
| 03 | [`03_router_snmp.png`](/screenshots/03_router_snmp.png) | `show snmp community` en el Router confirmando la comunidad `public_ro` con acceso `NOAUTHNOPRIV` y permisos `ro`. |
| 04 | [`04_switch_ip.png`](/screenshots/04_switch_ip.png) | `show interface vlan 1` en el Switch mostrando la IP `10.7.37.3/24` en estado `up/up`. |
| 05 | [`05_switch_snmp.png`](/screenshots/05_switch_snmp.png) | `show snmp community` en el Switch confirmando la comunidad `public_ro` con permisos `ro`. |
| 06 | [`06_ubuntu_ip.png`](/screenshots/06_ubuntu_ip.png) | Terminal Ubuntu mostrando `ip addr show ens34` con la IP estática `10.7.37.253/24` configurada y activa. |
| 07 | [`07_zabbix_instalado.png`](/screenshots/07_zabbix_instalado.png) | Terminal Ubuntu mostrando `systemctl status zabbix-server` con estado `active (running)`. |
| 08 | [`08_zabbix_gui_login.png`](/screenshots/08_zabbix_gui_login.png) | Navegador del host Windows mostrando la pantalla de login de Zabbix en `http://10.7.37.253/zabbix`. |
| 09 | [`09_zabbix_setup_ok.png`](/screenshots/09_zabbix_setup_ok.png) | Asistente de instalación web de Zabbix en el paso "Comprobación de requisitos previos" con todo en verde. |
| 10 | [`10_zabbix_host_router.png`](/screenshots/10_zabbix_host_router.png) | Formulario `Crear equipo` en Zabbix mostrando R1 configurado con interfaz SNMP `10.7.37.1:161`, plantilla `Cisco IOS SNMP` y macro `{$SNMP_COMMUNITY}=public_ro`. |
| 11 | [`11_zabbix_router_verde.png`](/screenshots/11_zabbix_router_verde.png) | Lista de equipos en Zabbix mostrando `R1` con ícono verde — confirmando que Zabbix recibe datos SNMP del Router correctamente. |
| 12 | [`12_zabbix_host_switch.png`](/screenshots/12_zabbix_host_switch.png) | Formulario `Crear equipo` mostrando el Switch configurado con interfaz SNMP `10.7.37.2:161` y plantilla `Cisco IOS SNMP`. |
| 13 | [`13_zabbix_switch_verde.png`](/screenshots/13_zabbix_switch_verde.png) | Lista de equipos mostrando `Switch-Lab` con ícono verde — Zabbix recibiendo datos SNMP del Switch. |
| 14 | [`14_zabbix_latest_data_router.png`](/screenshots/14_zabbix_latest_data_router.png) | `Monitorización → Últimos datos` filtrado por `R1` mostrando métricas SNMP activas: uptime, descripción del sistema, tráfico de interfaces. |
| 15 | [`15_zabbix_events.png`](/screenshots/15_zabbix_events.png) | `Monitorización → Problemas` mostrando los eventos generados por las plantillas de Cisco IOS SNMP. |
| 16 | [`16_zabbix_graphs_switch.png`](/screenshots/16_zabbix_graphs_switch.png) | `Monitorización → Últimos datos` filtrado por `Switch-Lab` mostrando al menos una gráfica de tráfico o CPU activa con datos. |
| 17 | [`17_cliente_dhcp.png`](/screenshots/17_cliente_dhcp.png) | `ipconfig /all` en el host Windows mostrando IP `10.7.37.X` asignada por DHCP con gateway `10.7.37.1` y servidor DHCP `10.7.37.1`. |
| 18 | [`18_cliente_snmpwalk_router.png`](/screenshots/18_cliente_snmpwalk_router.png) | Terminal del host Windows mostrando la salida de `snmpwalk` hacia `10.7.37.1` con los campos `sysDescr`, `sysUpTime`, `sysContact` y `sysLocation` del Router. |
| 19 | [`19_cliente_snmpwalk_switch.png`](/screenshots/19_cliente_snmpwalk_switch.png) | Terminal del host Windows mostrando la salida de `snmpwalk` hacia `10.7.37.2` con los datos SNMP del Switch. |
