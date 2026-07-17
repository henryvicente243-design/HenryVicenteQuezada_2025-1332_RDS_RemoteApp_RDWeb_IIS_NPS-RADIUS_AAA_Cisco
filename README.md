# RDS RemoteApp · RD Web Client · NPS-RADIUS en Router Cisco

Implementación de una infraestructura de publicación de aplicaciones remotas (RemoteApp y RD Web Client) sobre Windows Server, integrada con un servidor NPS (RADIUS) para la autenticación centralizada y diferenciada por niveles de privilegio del acceso administrativo a un router Cisco.

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

 <img width="934" height="655" alt="image" src="https://github.com/user-attachments/assets/6dd5d5c7-ea94-475a-b68e-6ec4ee443d91" />
 

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

 <img width="829" height="550" alt="image" src="https://github.com/user-attachments/assets/1d910475-5411-45c2-8717-ab0fecddbafa" />
 

*Figura 2. Colección RemoteApp con Microsoft Edge publicado.*

 <img width="1016" height="732" alt="image" src="https://github.com/user-attachments/assets/4926f8fc-2ac6-4e6a-a9c7-acb5a5d9fd55" />
 

*Figura 3. Formulario de autenticación de RD Web Access.*

 <img width="1011" height="775" alt="image" src="https://github.com/user-attachments/assets/660a8b64-ae43-426e-b0f6-a758bc185701" />
 

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

 <img width="1051" height="873" alt="image" src="https://github.com/user-attachments/assets/88458805-8167-4c2a-8c9f-410b53b2be00" />
 

*Figura 5. Formulario de autenticación de RD Web Client (HTML5).*

 <img width="1001" height="728" alt="image" src="https://github.com/user-attachments/assets/6c66c1e3-663b-4ba8-a724-3248f0cc34de" />
 

*Figura 6. Portal RD Web Client autenticado.*

### 3.3 Página personalizada de IIS

 <img width="938" height="693" alt="image" src="https://github.com/user-attachments/assets/2e7ebe95-9c4a-402b-ac65-4462b9c56db2" />
 

*Figura 7. Página personalizada de IIS, accesible en http://10.32.1.10.*

### 3.4 Publicación de la página IIS en los RemoteApp

El RemoteApp de Edge se configuró con el argumento `http://10.32.1.10`, cargando automáticamente la página al iniciarse desde cualquiera de los dos portales.

 <img width="1162" height="399" alt="image" src="https://github.com/user-attachments/assets/a420a58d-a2fe-4a5b-964e-88043026f9ea" />
 

 <img width="536" height="336" alt="image" src="https://github.com/user-attachments/assets/2e93018d-4ff3-4e00-9d79-b063e8e4bfaf" />
 

<img width="749" height="555" alt="image" src="https://github.com/user-attachments/assets/364f4c8d-9ed8-4509-b94a-aa1a348f3349" />


*Figura 8. RemoteApp lanzado desde RD Web Access, mostrando la página IIS.*

 <img width="871" height="615" alt="image" src="https://github.com/user-attachments/assets/54d71d8e-b727-41b2-94ed-40e08d857374" />
 

<img width="877" height="601" alt="image" src="https://github.com/user-attachments/assets/07b82273-5189-4e3e-b13c-78ef3740e0ee" />


<img width="974" height="702" alt="image" src="https://github.com/user-attachments/assets/f071c915-408f-4dc5-9e76-b1c0ae0ade9c" />


*Figura 9. RemoteApp lanzado desde RD Web Client, mostrando la página IIS.*

### 3.5 Servicio NPS (RADIUS Server)

Se registró el router R1 como cliente RADIUS (IP `10.32.1.1`) y se crearon dos grupos de seguridad en AD, cada uno con su política de red y atributo Cisco-AV-Pair.

#### 3.5.1 Grupo de nivel de acceso 15

 <img width="476" height="517" alt="image" src="https://github.com/user-attachments/assets/5bb2fbff-745a-4339-a569-b80181f1a238" />
 

*Figura 10. Grupo de seguridad NivelAcceso15 en Active Directory.*

 <img width="863" height="531" alt="image" src="https://github.com/user-attachments/assets/e9bd5c25-41f9-4450-a02d-9ab708425372" />
 

*Figura 11. Atributo Cisco-AV-Pair (`shell:priv-lvl=15`).*

#### 3.5.2 Grupo de nivel de acceso 1

 <img width="403" height="459" alt="image" src="https://github.com/user-attachments/assets/44db3d09-9b58-42a5-954e-1edf6f730e66" />
 

*Figura 12. Grupo de seguridad NivelAcceso1 en Active Directory.*

 <img width="802" height="584" alt="image" src="https://github.com/user-attachments/assets/d2989987-1c4f-4d92-b403-c97d9fff6b04" />
 

*Figura 13. Atributo Cisco-AV-Pair (`shell:priv-lvl=1`).*

---

## 4. Configuración del Router

### 4.1 Usuario local

```
username admin privilege 15 secret Cisco123!
```

 <img width="781" height="395" alt="image" src="https://github.com/user-attachments/assets/ff5eb8f9-3ecf-4331-9594-6b51ba7caef3" />
 

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

 <img width="639" height="300" alt="image" src="https://github.com/user-attachments/assets/0b336417-984e-4542-8a41-616454eea15e" />
 

*Figura 15. Configuración AAA completa (`show run | section aaa`).*

### 4.4 Logs de autenticación, autorización y RADIUS

```
terminal monitor
debug aaa authentication
debug aaa authorization
debug radius
```

 <img width="706" height="398" alt="image" src="https://github.com/user-attachments/assets/c241feab-1fe0-4c6a-ae6e-4ca09588c058" />
 

*Figura 16. Salida combinada de los tres debugs durante una autenticación SSH exitosa.*

#### show aaa servers

 <img width="974" height="697" alt="image" src="https://github.com/user-attachments/assets/93832a71-fd03-4ae2-907b-b0449a5fa882" />
 

*Figura 17. Servidor RADIUS activo (`state: current UP`).*

#### show aaa sessions

 <img width="653" height="219" alt="image" src="https://github.com/user-attachments/assets/fa15b865-02d3-4806-952f-ccda7c355608" />
 

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

 <img width="656" height="288" alt="image" src="https://github.com/user-attachments/assets/5aceb42c-3f40-4c99-beae-374cd951f96a" />
 

*Figura 19. Sesión SSH con privilegio 15 (`show privilege`).*

#### Usuario nivel de acceso 1 (`userradius`)

```
ssh -l userradius 10.13.1.1
```

La sesión queda en modo restringido (`RTR-CORE-LAB>`).

 <img width="745" height="250" alt="image" src="https://github.com/user-attachments/assets/df5024c8-a3ee-4024-906e-757c30da3d6f" />

*Figura 20. Sesión SSH con privilegio 1 (`show privilege`).*

### 5.3 Conectividad de red

 <img width="603" height="565" alt="image" src="https://github.com/user-attachments/assets/a4f980c6-334b-4aae-a4db-b20508b2c714" />

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
