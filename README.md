CIS Network Hardening — GNS3

Jennifer Lopez Paulino
Matrícula: 20240860
13–14 de noviembre de 2026

enlace video:


<img width="1500" height="850" alt="image" src="https://github.com/user-attachments/assets/4792ed50-937b-4ed2-8f7c-0a5476a6c4e3" />

Diagrama lógico de la infraestructura. La conexión IP directa de ZBX1 a R1 no se considera validada.

1. Propósito del laboratorio

El propósito del laboratorio fue aplicar controles de endurecimiento de seguridad a una infraestructura de red virtualizada en GNS3, documentando la configuración y las verificaciones de un router Cisco, un switch Open vSwitch, dos terminales VPCS y un servidor Ubuntu. El trabajo se enfocó en restringir el acceso administrativo, proteger el plano de acceso de capa 2, controlar servicios expuestos y comprobar la conectividad de los equipos de la LAN.

2. Arquitectura y direccionamiento

Equipo

Dirección / función

R1 — Cisco 7200

IOS 15.2(4)S6; FastEthernet0/0 192.168.60.1/24; gateway de la LAN.

SW1 — Open vSwitch

Bridge br0; OpenFlow 1.3; reglas de filtrado por MAC; RSTP habilitado.

PC1 — VPCS

192.168.60.21/24; conectado a SW1; ping al gateway validado.

PC2 — VPCS

192.168.60.22/24; conectado a SW1; ping al gateway validado.

ZBX1 — Ubuntu Server

Ubuntu Server 24.04.5; interfaz NAT 10.0.2.15; UFW y SSH hardening. La conectividad directa a R1 no quedó validada.

1. Preparación y arquitectura

Se preparó el escenario en GNS3 con R1, SW1, PC1, PC2 y ZBX1. R1 se utilizó como gateway de la red 192.168.60.0/24, con dirección 192.168.60.1/24 en FastEthernet0/0. Los clientes VPCS utilizaron 192.168.60.21/24 y 192.168.60.22/24.

El diagrama de esta documentación resume la arquitectura observada. ZBX1 se documenta con su interfaz NAT 10.0.2.15; no se declara como verificada una ruta IP directa desde ZBX1 hacia R1.

2. Endurecimiento del router R1

Se revisaron las interfaces con show ip interface brief y se confirmó FastEthernet0/0 con 192.168.60.1 y estado up/up. Se habilitó SSH versión 2, se generó una clave RSA de 2048 bits y se ajustaron los parámetros de sesión: timeout de autenticación de 60 segundos y dos intentos de autenticación.

Se habilitó AAA con aaa new-model y autenticación local predeterminada. Se configuró el usuario administrativo local con privilegio 15 y se establecieron secretos locales y de enable. Las líneas VTY 0–4 se limitaron a SSH y se configuró un tiempo de inactividad de cinco minutos. La consola también recibió timeout de cinco minutos, privilegio 15 y logging synchronous.

Se deshabilitaron los servicios HTTP y HTTPS del router. Finalmente, se guardó la configuración con copy running-config startup-config y el equipo devolvió [OK]. La captura de sesión SSH demuestra un inicio de sesión remoto en R1 y la comprobación del nivel de privilegio 15.

3. Endurecimiento del switch SW1

SW1 se implementó con Open vSwitch 2.17.12, no como un switch Cisco IOS. Por ello, los controles se aplicaron con ovs-vsctl y ovs-ofctl y se documentan como una adaptación para Open vSwitch, no como port-security de Cisco.

Se configuró el bridge br0 para utilizar OpenFlow 1.3. Se establecieron reglas de prioridad alta que permiten las direcciones MAC autorizadas observadas en los puertos de acceso y reglas de prioridad inferior que descartan el tráfico restante en esos puertos. Se conservó la acción NORMAL de prioridad 0 para el tráfico no cubierto por las reglas específicas.

Se habilitó RSTP en br0. La salida de ovs-appctl rstp/show mostró el bridge como root y los puertos presentados en estado Designated Forwarding. También se comprobó que rstp_enable devolviera true.

4. Pruebas de conectividad

Desde PC1 se ejecutó ping 192.168.60.1 y se recibieron cinco respuestas. Desde PC2 se ejecutó la misma prueba y se recibieron cinco respuestas. Las capturas finales muestran conectividad de ambos clientes VPCS con el gateway R1.

Los contadores de OpenFlow se revisaron con ovs-ofctl -O OpenFlow13 dump-flows br0. La existencia de reglas y contadores respalda que el flujo de paquetes atravesó las reglas instaladas. No se realizó una prueba controlada final con una MAC no autorizada, por lo que no se afirma que el descarte de una MAC desconocida haya sido validado experimentalmente.

5. Endurecimiento de Ubuntu ZBX1

En Ubuntu Server 24.04.5 se mantuvo activo UFW con política predeterminada de denegar conexiones entrantes, permitir salientes y enrutamiento deshabilitado. Se documentaron reglas que permiten TCP 22 y TCP 80 desde 10.0.2.0/24.

Se creó el archivo /etc/ssh/sshd_config.d/99-cis-hardening.conf con PermitRootLogin no, MaxAuthTries 4, X11Forwarding no, ClientAliveInterval 300 y ClientAliveCountMax 2. Se validó la configuración y el servicio SSH permaneció activo y habilitado.

Se ejecutó el script de auditoría ~/CIS-Network-Hardening/scripts/ubuntu_cis.py en modo audit. La captura final presenta cinco comprobaciones PASS y cero FAIL: prohibición de login root, máximo de intentos SSH, X11Forwarding deshabilitado, UFW activo y política de entrada DENY. El script generó reportes TXT y JSON dentro de la carpeta reports.

6. Alcance, limitaciones y honestidad técnica

El acceso SSH remoto a R1 se comprobó desde un cliente remoto y se verificó el nivel de privilegio. No se considera demostrada una sesión SSH desde ZBX1 porque su interfaz NAT y su ruta por defecto no pertenecían a la LAN 192.168.60.0/24 durante las pruebas descritas.

El switch utilizado es Open vSwitch y no dispone de la CLI IOS de un switch Cisco administrable. Por tanto, el documento identifica los controles equivalentes implementados con OpenFlow y evita atribuirle funciones IOS no disponibles.

RSTP quedó habilitado y su estado fue inspeccionado. La persistencia de las reglas OpenFlow después de reiniciar el nodo no se probó. Tampoco se ejecutó una prueba final controlada de MAC no autorizada. Estas limitaciones se conservan explícitas en el informe.

7. Matriz de verificación

Componente

Verificación documentada

Resultado

R1

FastEthernet0/0 192.168.60.1/24; SSHv2; AAA local; VTY solo SSH; configuración guardada

Configuración y guardado evidenciados

R1

Sesión SSH remota y show privilege

Nivel de privilegio 15 evidenciado

SW1

OpenFlow 1.3; reglas MAC; contadores

Configuración de flujos evidenciada

SW1

RSTP habilitado; bridge root; puertos forwarding

Estado mostrado en captura

PC1

ping 192.168.60.1

5 respuestas

PC2

ping 192.168.60.1

5 respuestas

ZBX1

UFW activo; incoming DENY; allow 22/80 desde 10.0.2.0/24

Evidenciado

ZBX1

SSH hardening y servicio activo

Evidenciado

ZBX1

Script de auditoría

5 PASS / 0 FAIL

Pendiente

Prueba controlada de MAC desconocida y persistencia OpenFlow tras reinicio

No verificado

Pendiente

Conectividad SSH directa desde ZBX1 a R1

No verificada

8. Conclusión

La práctica permitió aplicar y documentar controles de hardening en los componentes disponibles del laboratorio. Se verificó la administración segura de R1 mediante SSH y AAA local, el filtrado de capa 2 y RSTP en Open vSwitch, la conectividad de PC1 y PC2 hacia el gateway, y los controles de UFW y SSH en Ubuntu. Los resultados se presentan diferenciando las comprobaciones satisfactorias de las pruebas que quedaron pendientes, de modo que la documentación refleje fielmente el trabajo realizado.

9. Archivos y seguridad

Antes de subir configuraciones al repositorio, eliminar contraseñas, hashes, claves privadas, secretos compartidos y cualquier credencial. El running-config de R1 debe ser la salida real del equipo; SW1 se documenta mediante comandos propios de Open vSwitch y PC1/PC2 mediante show ip.
