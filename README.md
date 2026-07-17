 RDS RemoteApp · RD Web Client · NPS-RADIUS en Router Cisco

Práctica P5 — Implementación de una infraestructura de publicación de aplicaciones remotas (RemoteApp y RD Web Client) sobre Windows Server, integrada con un servidor NPS (RADIUS) para la autenticación centralizada y diferenciada por niveles de privilegio del acceso administrativo a un router Cisco.

**Autor** Henry Vicente Quezada **Matrícula** 2025-1332 PrácticaP5 EntornoPNETLab + VMware Workstation


🎥 Video demostrativo

Mostrar imagen

▶️ Haz clic aquí para ver el video completo


1. Objetivo de la red

El presente proyecto tiene como objetivo diseñar e implementar una infraestructura de red que integra publicación de aplicaciones remotas, control de identidad centralizado y administración segura de dispositivos de red. Concretamente, se busca:


Publicar una página web corporativa alojada en un servidor IIS a través de dos mecanismos de acceso remoto: RD Web Access (RemoteApp clásico) y RD Web Client (acceso vía navegador HTML5).
Centralizar la autenticación del acceso administrativo al router mediante un servidor RADIUS (NPS), eliminando la dependencia de credenciales locales únicas.
Implementar dos niveles de autorización (privilegio 15 y privilegio 1) aplicados dinámicamente según el grupo de Active Directory del usuario.
Validar el funcionamiento íntegro de la solución desde la perspectiva de un cliente final.



2. Topología de red

Mostrar imagen
Figura 1. Diagrama de topología, con nombre, matrícula, dispositivos e interfaces.

2.1 Direccionamiento IP

DispositivoInterfazIPMáscaraRolRouter R1Ethernet0/010.13.1.1/24Gateway red ClienteRouter R1Ethernet0/110.32.1.1/24Gateway red ServerWindows ClientEthernet010.13.1.10/24Estación clienteWindows ServerEthernet0 (VMnet7)10.32.1.10/24IIS / RDS / NPS / AD DS


Direccionamiento basado en la matrícula del estudiante (1332). No se emplearon VLANs; cada host conecta directamente a su interfaz correspondiente en el router.




3. Configuración del Windows Server

Servidor SRV-2025-1332, promovido a Controlador de Dominio (henryvicente.local), requisito necesario para completar la instalación de Servicios de Escritorio remoto.

3.1 Servicio de RDP RemoteApp

Se instalaron los roles RD Session Host, RD Connection Broker y RD Web Access, y se publicó Microsoft Edge como RemoteApp.

Mostrar imagen
Figura 2. Colección RemoteApp con Microsoft Edge publicado.

Mostrar imagen
Figura 3. Formulario de autenticación de RD Web Access.

Mostrar imagen
Figura 4. Portal RD Web Access autenticado.

3.2 Servicio de RDP RemoteApp Web Client

Componente HTML5 instalado vía PowerShell (RDWebClientManagement), disponible en /RDWeb/webclient.

powershellInstall-Module -Name RDWebClientManagement -Force
Import-Module RDWebClientManagement
Install-RDWebClientPackage
Import-RDWebClientBrokerCert -Path "C:\BrokerCert.cer"
Publish-RDWebClientPackage -Type Production -Latest

Mostrar imagen
Figura 5. Formulario de autenticación de RD Web Client (HTML5).

Mostrar imagen
Figura 6. Portal RD Web Client autenticado.

3.3 Página personalizada de IIS

Mostrar imagen
Figura 7. Página personalizada de IIS, accesible en http://10.32.1.10.

3.4 Publicación de la página IIS en los RemoteApp

El RemoteApp de Edge se configuró con el argumento http://10.32.1.10, cargando automáticamente la página al iniciarse desde cualquiera de los dos portales.

Mostrar imagen
Figura 8. RemoteApp lanzado desde RD Web Access, mostrando la página IIS.

Mostrar imagen
Figura 9. RemoteApp lanzado desde RD Web Client, mostrando la página IIS.

3.5 Servicio NPS (RADIUS Server)

Se registró el router R1 como cliente RADIUS (IP 10.32.1.1) y se crearon dos grupos de seguridad en AD, cada uno con su política de red y atributo Cisco-AV-Pair.

3.5.1 Grupo de nivel de acceso 15

Mostrar imagen
Figura 10. Grupo de seguridad NivelAcceso15 en Active Directory.

Mostrar imagen
Figura 11. Atributo Cisco-AV-Pair (shell:priv-lvl=15).

3.5.2 Grupo de nivel de acceso 1

Mostrar imagen
Figura 12. Grupo de seguridad NivelAcceso1 en Active Directory.

Mostrar imagen
Figura 13. Atributo Cisco-AV-Pair (shell:priv-lvl=1).


4. Configuración del Router

4.1 Usuario local

username admin privilege 15 secret Cisco123!

Mostrar imagen
Figura 14. Usuario local (show run | include username).

4.2 Contraseña para el modo de configuración

enable secret Cisco123!

4.3 AAA configurado para autenticar vía RADIUS

aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local

radius server NPS-SERVER
 address ipv4 10.32.1.10 auth-port 1812 acct-port 1813
 key RadiusKey123!

Mostrar imagen
Figura 15. Configuración AAA completa (show run | section aaa).

4.4 Logs de autenticación, autorización y RADIUS

terminal monitor
debug aaa authentication
debug aaa authorization
debug radius

Mostrar imagen
Figura 16. Salida combinada de los tres debugs durante una autenticación SSH exitosa.

show aaa servers

Mostrar imagen
Figura 17. Servidor RADIUS activo (state: current UP).

show aaa sessions

Mostrar imagen
Figura 18. Sesión SSH activa autenticada vía RADIUS.


5. Pruebas desde el Cliente

5.1 Consulta de la página vía los dos RemoteApp

Ya documentado en la sección 3.4 (Figuras 8 y 9).

5.2 Prueba de conexión SSH al router vía RADIUS

Usuario nivel de acceso 15 (adminradius)

ssh -l adminradius 10.13.1.1

La sesión inicia directamente en modo privilegiado (RTR-CORE-LAB#).

Mostrar imagen
Figura 19. Sesión SSH con privilegio 15 (show privilege).

Usuario nivel de acceso 1 (userradius)

ssh -l userradius 10.13.1.1

La sesión queda en modo restringido (RTR-CORE-LAB>).

Mostrar imagen
Figura 20. Sesión SSH con privilegio 1 (show privilege).

5.3 Conectividad de red

Mostrar imagen
Figura 21. Ping entre Windows Client y Windows Server, enrutado a través de R1.


6. Conclusiones

La implementación permitió integrar la publicación de aplicaciones remotas (RemoteApp y RD Web Client), un servidor web con contenido personalizado (IIS), y un esquema de autenticación centralizada para el acceso administrativo a un router Cisco mediante NPS/RADIUS.

Se comprobó que el atributo Cisco-AV-Pair, configurado en las políticas de red de NPS, permite asignar de forma dinámica el nivel de privilegio de un usuario administrativo sin mantener múltiples cuentas locales en el router, centralizando dicha gestión en Active Directory.

El proyecto cumple con la totalidad de los requisitos: instalación y publicación de los servicios RDS, configuración del NPS con los dos niveles de acceso solicitados, y configuración del router con AAA vía RADIUS, usuario local de respaldo, y los logs de verificación correspondientes.


📂 Estructura del repositorio

.
├── README.md
├── HenryVicenteQuezada_2025-1332_P5.zip
├── docs/
│   └── HenryVicenteQuezada_2025-1332_Informe_P5.pdf
├── configs/
│   └── router_config.txt
└── capturas/
    ├── 01_RemoteAppPublicando.png
    ├── 02_RDWebAccess-Login.png
    ├── ...
    └── 21_Ping_de_windowsClient.png


⚠️ Nota de integridad académica

Este proyecto fue desarrollado de forma individual. Cualquier indicio de fraude o copia equivale a la descalificación total de la asignación.
