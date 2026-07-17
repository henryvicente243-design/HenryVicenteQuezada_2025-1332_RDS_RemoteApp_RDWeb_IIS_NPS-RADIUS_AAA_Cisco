# RDS RemoteApp · RD Web Client · NPS-RADIUS en Router Cisco

**Práctica P5** — Implementación de una infraestructura de publicación de aplicaciones remotas (RemoteApp y RD Web Client) sobre Windows Server, integrada con un servidor NPS (RADIUS) para la autenticación centralizada y diferenciada por niveles de privilegio del acceso administrativo a un router Cisco.

| | |
|---|---|
| **Autor** | Henry Vicente Quezada |
| **Matrícula** | 2025-1332 |
| **Práctica** | P5 |
| **Entorno** | PNETLab + VMware Workstation |

---

## 🎥 Video demostrativo


▶️ **[Haz clic aquí para ver el video completo](https://youtu.be/ahc-m3bMVb0 )**

---

## 1. Objetivo de la red

El presente proyecto tiene como objetivo diseñar e implementar una infraestructura de red que integra publicación de aplicaciones remotas, control de identidad centralizado y administración segura de dispositivos de red. Concretamente, se busca:

- Publicar una página web corporativa alojada en un servidor IIS a través de dos mecanismos de acceso remoto: RD Web Access (RemoteApp clásico) y RD Web Client (acceso vía navegador HTML5).
- Centralizar la autenticación del acceso administrativo al router mediante un servidor RADIUS (NPS), eliminando la dependencia de credenciales locales únicas.
- Implementar dos niveles de autorización (privilegio 15 y privilegio 1) aplicados dinámicamente según el grupo de Active Directory del usuario.
- Validar el funcionamiento íntegro de la solución desde la perspectiva de un cliente final.

---

## 2. Topología de red

![Topología de red](./capturas/20_Topologia.png)
*Figura 1. Diagrama de topología, con nombre, matrícula, dispositivos e interfaces.*

### 2.1 Direccionamiento IP

| Dispositivo | Interfaz | IP | Máscara | Rol |
|---|---|---|---|---|
| Router R1 | Ethernet0/0 | 10.13.1.1 | /24 | Gateway red Cliente |
| Router R1 | Ethernet0/1 | 10.32.1.1 | /24 | Gateway red Server |
| Windows Client | Ethernet0 | 10.13.1.10 | /24 | Estación cliente |
| Windows Server | Ethernet0 (VMnet7) | 10.32.1.10 | /24 | IIS / RDS / NPS / AD DS |

> Direccionamiento basado en la matrícula del estudiante (1332). No se emplearon VLANs; cada host conecta directamente a su interfaz correspondiente en el router.

---

## 3. Configuración del Windows Server

Servidor `SRV-2025-1332`, promovido a Controlador de Dominio (`henryvicente.local`), requisito necesario para completar la instalación de Servicios de Escritorio remoto.

### 3.1 Servicio de RDP RemoteApp

Se instalaron los roles RD Session Host, RD Connection Broker y RD Web Access, y se publicó Microsoft Edge como RemoteApp.

![RemoteApp publicado](./capturas/01_RemoteAppPublicando.png)
*Figura 2. Colección RemoteApp con Microsoft Edge publicado.*

![Login RD Web Access](./capturas/02_RDWebAccess-Login.png)
*Figura 3. Formulario de autenticación de RD Web Access.*

![RD Web Access logueado](./capturas/03_RDWebAccess-Logueado.png)
*Figura 4. Portal RD Web Access autenticado.*

### 3.2 Servicio de RDP RemoteApp Web Client

Componente HTML5 instalado vía PowerShell (`RDWebClientManagement`), disponible en `/RDWeb/webclient`.

```powershell
Install-Module -Name RDWebClientManagement -Force
Import-Module RDWebClientManagement
Install-RDWebClientPackage
Import-RDWebClientBrokerCert -Path "C:\BrokerCert.cer"
Publish-RDWebClientPackage -Type Production -Latest
```

![Login RD Web Client](./capturas/04_RDWebClient-Login.png)
*Figura 5. Formulario de autenticación de RD Web Client (HTML5).*

![RD Web Client logueado](./capturas/05_RDWebClient-Logueado.png)
*Figura 6. Portal RD Web Client autenticado.*

### 3.3 Página personalizada de IIS

![Página IIS](./capturas/06_PaginaIIS.png)
*Figura 7. Página personalizada de IIS, accesible en http://10.32.1.10.*

### 3.4 Publicación de la página IIS en los RemoteApp

El RemoteApp de Edge se configuró con el argumento `http://10.32.1.10`, cargando automáticamente la página al iniciarse desde cualquiera de los dos portales.

![RemoteApp vía Web Access](./capturas/07_RemoteApp-MostrandoPaginaIIS_WebAccess.png)
*Figura 8. RemoteApp lanzado desde RD Web Access, mostrando la página IIS.*

![RemoteApp vía Web Client](./capturas/08_RemoteApp-MostrandoPaginaIIS_WebClient.png)
*Figura 9. RemoteApp lanzado desde RD Web Client, mostrando la página IIS.*

### 3.5 Servicio NPS (RADIUS Server)

Se registró el router R1 como cliente RADIUS (IP `10.32.1.1`) y se crearon dos grupos de seguridad en AD, cada uno con su política de red y atributo Cisco-AV-Pair.

#### 3.5.1 Grupo de nivel de acceso 15

![Grupo Nivel 15](./capturas/09_GrupoNivel15.png)
*Figura 10. Grupo de seguridad NivelAcceso15 en Active Directory.*

![Política Nivel 15](./capturas/10_PoliticaNivel15-AVPair.png)
*Figura 11. Atributo Cisco-AV-Pair (`shell:priv-lvl=15`).*

#### 3.5.2 Grupo de nivel de acceso 1

![Grupo Nivel 1](./capturas/11_GrupoNivel1.png)
*Figura 12. Grupo de seguridad NivelAcceso1 en Active Directory.*

![Política Nivel 1](./capturas/12_PoliticaNivel1-AVPair.png)
*Figura 13. Atributo Cisco-AV-Pair (`shell:priv-lvl=1`).*

---

## 4. Configuración del Router

### 4.1 Usuario local

```
username admin privilege 15 secret Cisco123!
```

![Usuario local](./capturas/13_UsuarioLocal.png)
*Figura 14. Usuario local (`show run | include username`).*

### 4.2 Contraseña para el modo de configuración

```
enable secret Cisco123!
```

### 4.3 AAA configurado para autenticar vía RADIUS

```
aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local

radius server NPS-SERVER
 address ipv4 10.32.1.10 auth-port 1812 acct-port 1813
 key RadiusKey123!
```

![AAA config](./capturas/14_AAA-Config.png)
*Figura 15. Configuración AAA completa (`show run | section aaa`).*

### 4.4 Logs de autenticación, autorización y RADIUS

```
terminal monitor
debug aaa authentication
debug aaa authorization
debug radius
```

![Debugs](./capturas/15_Debugs-Authentication-Authorization-Radius.png)
*Figura 16. Salida combinada de los tres debugs durante una autenticación SSH exitosa.*

#### show aaa servers

![show aaa servers](./capturas/16_ShowAAAServers.png)
*Figura 17. Servidor RADIUS activo (`state: current UP`).*

#### show aaa sessions

![show aaa sessions](./capturas/17_ShowAAASessions.png)
*Figura 18. Sesión SSH activa autenticada vía RADIUS.*

---

## 5. Pruebas desde el Cliente

### 5.1 Consulta de la página vía los dos RemoteApp

Ya documentado en la sección 3.4 (Figuras 8 y 9).

### 5.2 Prueba de conexión SSH al router vía RADIUS

#### Usuario nivel de acceso 15 (`adminradius`)

```
ssh -l adminradius 10.13.1.1
```

La sesión inicia directamente en modo privilegiado (`RTR-CORE-LAB#`).

![SSH nivel 15](./capturas/18_SSH-Nivel15.png)
*Figura 19. Sesión SSH con privilegio 15 (`show privilege`).*

#### Usuario nivel de acceso 1 (`userradius`)

```
ssh -l userradius 10.13.1.1
```

La sesión queda en modo restringido (`RTR-CORE-LAB>`).

![SSH nivel 1](./capturas/19_SSH-Nivel1.png)
*Figura 20. Sesión SSH con privilegio 1 (`show privilege`).*

### 5.3 Conectividad de red

![Ping](./capturas/21_Ping_de_windowsClient.png)
*Figura 21. Ping entre Windows Client y Windows Server, enrutado a través de R1.*

---

## 6. Conclusiones

La implementación permitió integrar la publicación de aplicaciones remotas (RemoteApp y RD Web Client), un servidor web con contenido personalizado (IIS), y un esquema de autenticación centralizada para el acceso administrativo a un router Cisco mediante NPS/RADIUS.

Se comprobó que el atributo Cisco-AV-Pair, configurado en las políticas de red de NPS, permite asignar de forma dinámica el nivel de privilegio de un usuario administrativo sin mantener múltiples cuentas locales en el router, centralizando dicha gestión en Active Directory.

El proyecto cumple con la totalidad de los requisitos: instalación y publicación de los servicios RDS, configuración del NPS con los dos niveles de acceso solicitados, y configuración del router con AAA vía RADIUS, usuario local de respaldo, y los logs de verificación correspondientes.

---

## 📂 Estructura del repositorio

```
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
```

---

## ⚠️ Nota de integridad académica

Este proyecto fue desarrollado de forma **individual**. Cualquier indicio de fraude o copia equivale a la descalificación total de la asignación.
